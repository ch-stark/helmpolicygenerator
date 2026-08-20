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
- Example repo: [ch-stark/helmpolicygenerator](https://github.com/ch-stark/helmpolicygenerator)

---

## Benefits of this approach

Compared with `helm install` per cluster, Argo CD Helm Applications, or hand-written `Policy` YAML, inflating the chart and wrapping it with PolicyGenerator gives you Helm’s packaging **and** ACM’s fleet governance without running Helm on the spokes.

**Keep the Helm chart. Stop rewriting it as policy.** App teams keep `Chart.yaml`, templates, and `values.yaml`. Platform teams point PolicyGenerator at the Kustomize directory that inflates the chart. You do not transcribe Deployments and Services into `ConfigurationPolicy` by hand, and you do not fork the chart to make it “ACM-native.”

**One hub object, many clusters.** GitOps applies Policy / Placement / PlacementBinding on the hub once. Placement label selectors and ClusterSets fan the same desired state to tens or hundreds of managed clusters. You do not maintain a Helm release or an Argo Application per cluster for that configuration.

**Deploy and prove compliance.** Helm and Argo apply objects. ACM also reports whether they stayed that way. `inform` gives fleet-wide drift in the Governance console. `enforce` self-heals. Standards annotations, PolicySets, and PolicyAutomation (Ansible on violation) sit on the same objects. That is the gap a Helm release secret on each spoke does not fill.

**The rendered desired state is reviewable.** Inflation happens at `kustomize build` time. A PR can show the exact Policy YAML that will land on the hub, including chart version bumps. There is no surprise `helm template` on the managed cluster at reconcile time.

**No Helm runtime on managed clusters.** Spokes do not need the Helm CLI, release history Secrets, or chart pulls. That is smaller operational surface area, friendlier for air-gapped and GitOps ZTP / edge clusters, and it matches ACM’s model: the hub owns desired state, the config-policy controller compares or enforces.

**Patch charts without forking them.** `valuesInline` / `valuesFile` plus Kustomize patches on the inflated manifests let you overlay image, replica, or label changes while still consuming upstream charts. PolicyGenerator patches apply *before* wrapping, so the Policy contains the already-customized objects.

**Less policy boilerplate.** PolicyGenerator emits Placement, PlacementBinding, wrapping `ConfigurationPolicy`, consolidation vs per-manifest policies, install order (`orderManifests` / `dependencies`), and Kyverno/Gatekeeper expanders. The example in [helmpolicygenerator](https://github.com/ch-stark/helmpolicygenerator) is a normal Helm chart whose `templates/` already carry `kustomization.yaml` + `policyGenerator.yaml` + `input/` — the chart stays a chart, the generator produces the ACM envelope.

**Placement is the targeting API you already use.** Cluster labels, ClusterSets, and the same binding model as every other ACM policy. Mixing this Helm-wrapped policy with OperatorPolicy, CertificatePolicy, or Gatekeeper in one PolicySet is straightforward.

**Safer default semantics for third-party YAML.** `musthave` requires the chart’s fields without fighting controller-added labels, defaults, or status. That is often a better fleet contract than Helm’s three-way merge on every spoke.

**Fits the hub GitOps path you already run.** OpenShift GitOps or ACM subscriptions on the hub already speak Kustomize. Enable the PolicyGenerator plugin and `POLICY_GEN_ENABLE_HELM`; you do not stand up a second CD product just to get Helm apps under governance.

**What you trade for those benefits.** Helm hooks do not run. Chart upgrades are GitOps rebuilds, not in-place Helm upgrades. Secrets from the chart land in Policy YAML on the hub unless you template them out. Start with `inform` on large charts. For OLM operators, `OperatorPolicy` is still the better primitive than inflating an operator chart.

---

## What problems this approach solves (and what it does not)

Internal Helm-vs-GitOps research often concludes that Helm-heavy teams prefer **Flux** (`helm-controller` + Helm SDK) over **Argo CD** (`helm template` + kubectl apply). That comparison is real — and it is the **wrong yardstick** for HelmChartInflationGenerator + PolicyGenerator + RHACM Policies.

This pipeline is a **`helm template` path**, like Argo CD, not a Helm-release path, like Flux. It will not make `helm list` work on the spoke, will not run vendor `pre-install` hooks, and will not call `helm rollback`. Judged as a Helm runtime, Flux wins. Judged as **fleet configuration governance**, this approach solves a different set of problems that Flux HelmReleases do not.

### How the Flux claims map

| Helm-heavy claim | Flux v2 | Argo CD | This ACM approach |
| --- | --- | --- | --- |
| Helm hooks (migrations, tests) | Native Helm SDK | Ignored; rewrite as Argo hooks | **Not solved.** Hooks do not run. Use `skipHooks` or split jobs into their own Policies. |
| `helm list` / `helm history` | Release Secrets on the cluster | Nothing | **Not solved, by design.** No Helm release on the spoke. Visibility is ACM Policy status. |
| Automatic Helm rollback | Helm SDK on failed install/upgrade | Git revert / Argo sync | **Not solved as Helm rollback.** Fix Git and rebuild; or `enforce` reconverges objects. |
| OCI chart registries | First-class `OCIRepository` | Supported, more setup | **Partially.** Kustomize helm inflation is a limited Helm subset; private OCI auth is a known gap. Prefer vendoring the chart in Git. |
| Kustomize over vendor Helm | HelmRelease → Kustomization | Plugins / post-renderer | **Solved at build time.** Inflation + Kustomize patches + PolicyGenerator patches, no cluster plugin. |
| Subcharts / dependencies | Helm SDK | Rendered by `helm template` | **Solved if `helm dependency build` ran.** Output is still flattened YAML in the Policy, not a live subchart tree. |

The research is directionally right about Flux vs Argo **as Helm installers**. It overstates a few points (Argo does support OCI; `helm template` still renders subcharts after `helm dependency build`; Argo has post-renderers, not only `argocd-cm` plugins). None of that changes the ACM conclusion: **do not use this approach to replace Flux if the requirement is a real Helm release on every cluster.**

### Problems it does solve

**1. Helm-per-cluster does not scale as a fleet control plane.**  
A Prometheus or cert-manager HelmRelease (or Argo Application) per managed cluster means N sources, N reconciles, N sets of registry credentials. One Policy + Placement on the hub is the targeting model ACM already uses. That is the problem GitOps ZTP and large hub-spoke estates actually have.

**2. Spokes should not pull charts or store Helm release Secrets.**  
Flux’s “you can `helm list`” is also a cost: full release state (often including values) on every cluster, `helm-controller` + `source-controller` (or equivalent) everywhere, and chart/OCI pulls from the edge. Air-gapped, disconnected, and small clusters cannot or should not do that. Inflation on the hub, Policy on the hub, config-policy-controller on the spoke.

**3. “Installed” is not “compliant.”**  
`helm list` and Flux HelmRelease status tell you the installer succeeded. They do not tell you a Deployment still matches the chart 12 hours later, or map to NIST controls. `inform` / `enforce`, hub status rollup, PolicySets, and PolicyAutomation are the missing layer after any Helm-based CD.

**4. Vendor charts you must govern, not “Helm-install.”**  
Many platform charts (ingress, logging, compliance operators’ supporting YAML) are configuration you want identical across a ClusterSet. You need dry-run-like **inform first**, then enforce. Flux helm-controller is an installer. ConfigurationPolicy is a continuous contract.

**5. Hooks are a fleet hazard, not only a missing feature.**  
The same Prometheus/cert-manager hooks Flux runs natively are Jobs, migrations, and deletes on **every** selected cluster. On 200 edge nodes that is often the bug. Skipping hooks and encoding order with `orderManifests` / Policy `dependencies` is a solution to “vendor chart side effects at fleet scale,” not a regression.

**6. Two sources of truth (Git vs Helm rollback).**  
Flux automatic rollback can move the cluster off Git. Argo/ACM GitOps people treat that as a defect. This approach has one source: the rendered Policy in Git. Failed enforce shows NonCompliant; you fix Git. That solves the GitOps split-brain Helm rollback creates.

**7. Kustomize post-render without a cluster-side Helm pipeline.**  
The research’s valid Argo complaint is “wrapping Helm then Kustomize is awkward.” HelmChartInflationGenerator exists for that: render, patch, wrap. No Flux HelmRelease→Kustomization chain on the spoke, no Argo config-management plugin.

**8. App teams keep charts; platform teams keep Placement.**  
The [helmpolicygenerator](https://github.com/ch-stark/helmpolicygenerator) layout (Helm chart + `policyGenerator.yaml` + `input/`) solves the org problem: do not rewrite charts as Policy YAML, and do not give every app team a Flux HelmRelease API on production spokes.

**9. Mix Helm-originated objects with the rest of ACM governance.**  
One PolicySet can include the inflated chart policy, `OperatorPolicy`, CertificatePolicy, and Gatekeeper. Flux HelmRelease does not bind those together with Placement and compliance aggregation.

**10. Pin and review the exact YAML.**  
Helm SDK reconcilers can re-template. This approach freezes `helm template` output into the Policy. Chart bumps are diffs in Git. That solves supply-chain and change-control problems that “latest chart from OCI on each cluster” creates.

### Decision rule

- Need **Helm lifecycle on the cluster** (hooks, tests, `helm rollback`, `helm list`): Flux (or Helm itself). This approach will not get you there.
- Need **one Git-reviewed desired state, Placement to a fleet, compliance and optional self-heal, no Helm on the spoke**: HelmChartInflationGenerator + PolicyGenerator + RHACM Policies.

They are complementary. A hub can still use OpenShift GitOps to apply generated Policies while a few clusters keep Flux HelmReleases for the rare chart that truly requires hooks. Do not pick Flux because an installer comparison said Argo is bad at Helm — pick this ACM path when the problem is **fleet governance of chart-shaped YAML**, not **Helm as the runtime.**
