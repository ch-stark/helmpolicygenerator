# HelmChartInflationGenerator, RHACM Policies, and PolicyGenerator

These three pieces form a **build-time pipeline**: Helm is rendered into YAML, that YAML is wrapped into ACM policies, and ACM ships those policies to managed clusters.

```
Helm chart + values
        │  HelmChartInflationGenerator
        │  (kustomize --enable-helm)
        ▼
Kubernetes manifests
        │  PolicyGenerator
        │  (kustomize --enable-alpha-plugins)
        ▼
Policy + Placement + PlacementBinding
        │  GitOps apply on the hub
        ▼
ACM governance framework
        │  replicate to selected clusters
        ▼
ConfigurationPolicy inform / enforce
```

Inflation and wrapping happen when Kustomize runs (CI, OpenShift GitOps, or ACM subscriptions). Managed clusters never see a Helm release. They see ACM `ConfigurationPolicy` objects that already contain the rendered YAML.

---

## 1. HelmChartInflationGenerator

`HelmChartInflationGenerator` is a **Kustomize built-in**, not an ACM object. It shells out to Helm v3 (`helm template` / `helm pull`) and turns a chart into Kubernetes YAML so the rest of the Kustomize pipeline can patch, label, and namespace those objects.

Two equivalent ways to invoke it:

**Convenience field in `kustomization.yaml`:**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
helmCharts:
  - name: cert-manager
    repo: https://charts.jetstack.io
    version: v1.14.4
    releaseName: cert-manager
    namespace: cert-manager
    valuesFile: values.yaml
    includeCRDs: true
```

**Explicit generator (more control, often more reliable for namespace / overlay merges):**

```yaml
apiVersion: builtin
kind: HelmChartInflationGenerator
metadata:
  name: cert-manager
name: cert-manager
repo: https://charts.jetstack.io
version: v1.14.4
releaseName: cert-manager
namespace: cert-manager
valuesFile: values.yaml
includeCRDs: true
```

```yaml
# kustomization.yaml
generators:
  - helm-chart.yaml
```

Useful fields:

| Field | Role |
| --- | --- |
| `name` / `repo` / `version` | Chart identity. Omit `repo` to use a local chart under `chartHome` (default `charts/`). |
| `releaseName` / `namespace` | Passed to `helm template`. |
| `valuesFile` / `additionalValuesFiles` | File-based values. |
| `valuesInline` + `valuesMerge` | Inline overrides (`merge`, `override`, `replace`). |
| `includeCRDs` | Include CRDs from the chart (default `false`). |
| `skipHooks` / `skipTests` | Keep output closer to `helm install` vs dumping every template. |
| `kubeVersion` / `apiVersions` | Helm `Capabilities`. |

Requirements:

- Helm v3 on the machine (or GitOps repo-server) that runs Kustomize
- `kustomize build --enable-helm`

This is a **limited** Helm subset. SIG Kustomize will not add private-registry auth or post-renderers to the builtin. Treat it as “render this public/local chart into YAML,” not as a full Helm runtime.

That last point is the whole reason this pairs with ACM: once inflated, the chart is just manifests. Policies can own them.

---

## 2. RHACM Policies

A `Policy` is the smallest deployable unit on the **hub**. It does not apply objects itself. It groups **policy templates** that run on managed clusters after Placement selects those clusters.

Typical hub objects (same namespace):

- `Policy` — wrapper + aggregated compliance
- `Placement` — which `ManagedCluster`s
- `PlacementBinding` — binds Placement to Policy or PolicySet
- `ManagedClusterSetBinding` — required so the Placement can see any clusters

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-prod-ns
  namespace: policies
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-2 Baseline Configuration
spec:
  disabled: false
  remediationAction: inform
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-prod-ns
        spec:
          remediationAction: inform
          severity: low
          object-templates:
            - complianceType: musthave
              objectDefinition:
                apiVersion: v1
                kind: Namespace
                metadata:
                  name: prod
```

What actually runs on the managed cluster is usually a **`ConfigurationPolicy`**. Other templates include `OperatorPolicy`, `CertificatePolicy`, and Gatekeeper constraints.

### Remediation

- `inform` — report drift, do not change the cluster
- `enforce` — make the cluster match the object definition
- Policy-level `remediationAction` overrides the child template

### `complianceType` (the important Helm-related knob)

- `musthave` — listed fields must be present; extra fields on the live object are OK
- `mustonlyhave` — exact match; extra fields are violations (and get stripped if enforcing)
- `mustnothave` — matching object must not exist

For inflated Helm output, **`musthave` is the default and the usual choice**. Charts emit dozens of defaulted fields. `mustonlyhave` often fights controllers that add status, defaulted spec, or generated names.

### How delivery works

1. Root `Policy` lives on the hub (not in a managed-cluster namespace).
2. Placement selects clusters.
3. Governance replicates the policy into each matching cluster namespace on the hub, then to the cluster.
4. The config-policy controller compares/enforces.
5. Compliance rolls up to the hub Policies table.

`PolicyGenerator` only applies to this **hub policy framework**. If you push `ConfigurationPolicy` objects straight to clusters with GitOps, ACM treats them as discovered policies and the generator path does not apply.

---

## 3. PolicyGenerator

PolicyGenerator is a **Kustomize exec plugin** (`policy.open-cluster-management.io/v1`, kind `PolicyGenerator`). You do not install a CRD and wait for a controller. Kustomize finds the YAML under `generators:` and runs the `PolicyGenerator` binary.

It turns ordinary manifests into:

- `Policy` (with `ConfigurationPolicy` templates wrapping your YAML)
- `Placement`
- `PlacementBinding`
- optionally `PolicySet`

```yaml
# kustomization.yaml (outer)
generators:
  - policy-generator.yaml
```

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: cert-manager-policies
policyDefaults:
  namespace: policies
  remediationAction: inform
  severity: medium
  complianceType: musthave
  placement:
    labelSelector:
      matchLabels:
        environment: prod
policies:
  - name: policy-cert-manager
    manifests:
      - path: cert-manager/   # file, dir, or Kustomize directory
```

`kustomize build --enable-alpha-plugins` emits the Policy/Placement/Binding YAML. GitOps applies **that** to the hub.

What the generator does with `manifests[].path`:

| Input | Result |
| --- | --- |
| Plain Kubernetes YAML | Wrapped in a `ConfigurationPolicy` `object-templates` entry |
| Already a `*Policy` kind (`ConfigurationPolicy`, `OperatorPolicy`, `CertificatePolicy`) | Inserted as a policy-template, not double-wrapped |
| Kustomize directory | Runs Kustomize first, then wraps the build output |
| `object-templates-raw` file | Used as raw ConfigurationPolicy content (Go templates / `lookup`) |

Other capabilities worth using:

- **Patches** on input manifests before wrapping
- **`consolidateManifests`** (default `true`) — one ConfigurationPolicy for all objects vs one per object
- **`orderPolicies` / `orderManifests`** — generated dependencies
- **Kyverno / Gatekeeper expanders** — extra inform policies so ACM shows those engines’ violations
- **PolicySets** — one Placement for a group of policies
- GitOps ZTP: PolicyGenerator is the supported replacement for `PolicyGenTemplate`

---

## How they fit together

PolicyGenerator can recurse into a Kustomize directory. If that directory uses `helmCharts` or `HelmChartInflationGenerator`, Helm inflation happens **inside** the generator run.

That Helm support is **off by default**. Enable it where PolicyGenerator runs:

```bash
export POLICY_GEN_ENABLE_HELM=true
kustomize build --enable-alpha-plugins
```

If the chart lives outside the Kustomize root:

```bash
export POLICY_GEN_DISABLE_LOAD_RESTRICTORS=true
```

Recommended layout — **do not** put `HelmChartInflationGenerator` as a sibling generator next to PolicyGenerator. Sibling generators would dump raw Helm YAML onto the hub as well as wrapping it in policies.

```
repo/
  kustomization.yaml              # generators: policy-generator.yaml
  policy-generator.yaml           # path: helm-app/
  helm-app/
    kustomization.yaml            # helmCharts / generators: helm-chart.yaml
    helm-chart.yaml               # HelmChartInflationGenerator
    values.yaml
    patches/                      # optional JSON/strategic merge on inflated objects
```

- **Outer** Kustomize: `--enable-alpha-plugins` so the PolicyGenerator binary runs.
- **Inner** Kustomize (spawned by PolicyGenerator): Helm via `POLICY_GEN_ENABLE_HELM=true`.

What you apply to the hub is only Policy / Placement / PlacementBinding. `kind: HelmChartInflationGenerator` never becomes a live cluster object.

### OpenShift GitOps on the hub

The PolicyGenerator binary is not in the GitOps image by default. Configure the Argo CD instance:

1. Init container copies the PolicyGenerator binary from the ACM subscription/CLI image into `KUSTOMIZE_PLUGIN_HOME`
2. `spec.kustomizeBuildOptions: --enable-alpha-plugins`
3. `POLICY_GEN_ENABLE_HELM: "true"` on the repo-server if nested dirs use Helm
4. Helm v3 available to that repo-server
5. RBAC so the GitOps application controller can manage Policy, Placement, PlacementBinding

Community example: [policy-openshift-gitops-policygenerator.yaml](https://github.com/open-cluster-management-io/policy-collection/blob/main/community/CM-Configuration-Management/policy-openshift-gitops-policygenerator.yaml).

---

## Practical choices when wrapping a Helm chart

**One policy vs many.** Default `consolidateManifests: true` puts every inflated object (Deployments, Services, CRDs, RBAC, …) in one `ConfigurationPolicy`. Compliance is all-or-nothing for the chart. Set `false` if you need per-object status; expect a lot of policies.

**inform first.** Rendered charts are large. Start with `remediationAction: inform`, confirm the object list and field-level diffs, then switch to `enforce`.

**CRDs and install order.** Charts that install CRDs plus CRs need CRDs first. Use `orderManifests: true` with `consolidateManifests: false`, or split CRDs and the rest into two Policies with `dependencies`.

**Hooks.** Helm install hooks do not run. Use `skipHooks: true` unless you intentionally want hook templates as standing objects.

**Secrets.** Chart-generated Secrets become part of the Policy on the hub. Prefer external secret management, or hub/managed templates (`{{hub … hub}}` / `protect`) instead of baking credentials into Git-rendered YAML.

**Chart upgrades.** The Policy stores a snapshot of `helm template` output. Bumping `version:` only takes effect on the next Kustomize/GitOps build. This is GitOps-friendly (review the inflated diff) and not a live Helm release.

**Operators.** Prefer `OperatorPolicy` (or an OperatorGroup + Subscription you wrote) over inflating an operator’s Helm chart, when the operator is OLM-based.

**`musthave` vs `mustonlyhave`.** Inflated Helm YAML is almost always `musthave`. `mustonlyhave` on a Deployment that a chart also labels/annotates at runtime will flap.

---

## Minimal end-to-end example

`helm-app/helm-chart.yaml`:

```yaml
apiVersion: builtin
kind: HelmChartInflationGenerator
metadata:
  name: nginx
name: nginx
repo: https://charts.bitnami.com/bitnami
version: 18.1.0
releaseName: nginx
namespace: nginx
includeCRDs: false
valuesInline:
  replicaCount: 2
```

`helm-app/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
  - helm-chart.yaml
```

`policy-generator.yaml`:

```yaml
apiVersion: policy.open-cluster-management.io/v1
kind: PolicyGenerator
metadata:
  name: nginx-chart-policies
policyDefaults:
  namespace: policies
  remediationAction: inform
  complianceType: musthave
  consolidateManifests: true
  placement:
    labelSelector:
      matchLabels:
        vendor: OpenShift
policies:
  - name: policy-nginx
    manifests:
      - path: helm-app/
```

`kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
generators:
  - policy-generator.yaml
```

```bash
POLICY_GEN_ENABLE_HELM=true kustomize build --enable-alpha-plugins
```

Apply the output to the hub in a namespace that has a `ManagedClusterSetBinding`. Placement then distributes `policy-nginx` to matching clusters; the config-policy controller reports (or later enforces) the inflated nginx objects.

---

## Mental model

| Piece | What it is | When it runs | Output |
| --- | --- | --- | --- |
| HelmChartInflationGenerator | Kustomize builtin | `kustomize build --enable-helm` | Raw Kubernetes YAML from a chart |
| PolicyGenerator | Kustomize exec plugin | `kustomize build --enable-alpha-plugins` | Policy + Placement + PlacementBinding |
| RHACM Policy | Hub governance API | Continuously on hub + managed clusters | Compliance, optional enforce |

HelmChartInflationGenerator answers “how do I get chart YAML into GitOps without a Helm release?”

PolicyGenerator answers “how do I wrap that YAML so ACM can place, inform, and enforce it across the fleet?”

Policies are the runtime contract on the hub.

---

## References

- [RHACM 2.16 Policy deployment / Policy Generator](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html/governance/policy-deployment)
- [policy-generator-plugin](https://github.com/open-cluster-management-io/policy-generator-plugin)
- [Kustomize HelmChartInflationGenerator](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/#_helmchartinflationgenerator_)
- [OCM Policy API concepts](https://open-cluster-management.io/docs/getting-started/integration/policy-controllers/policy/)
- [Community OpenShift GitOps PolicyGenerator policy](https://github.com/open-cluster-management-io/policy-collection/blob/main/community/CM-Configuration-Management/policy-openshift-gitops-policygenerator.yaml)
