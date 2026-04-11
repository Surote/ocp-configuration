# CLAUDE.md — AI Assistant Guide for ocp-configuration

This file provides context, conventions, and workflows for AI assistants working in this repository.

## Repository Overview

This is an **OpenShift Configuration-as-Code (GitOps) repository** that declaratively manages an OpenShift 4.x cluster. All resources are expressed as plain YAML manifests deployed via ArgoCD. There is no Helm, no Kustomize overlays at the cluster level, and no scripted imperative setup—everything is a Kubernetes/OpenShift CRD applied directly.

**Primary concerns:**
- Operator lifecycle management (OLM subscriptions)
- Cluster-level configuration (MachineConfig, KubeletConfig, network)
- GitOps delivery via ArgoCD
- CI/CD via Tekton Pipelines (OpenShift Pipelines)
- BGP/MetalLB networking
- KubeVirt virtualization
- Policy enforcement via Gatekeeper
- Monitoring via custom Prometheus alerting rules

---

## Directory Structure

```
ocp-configuration/
├── argocd/              # ArgoCD Application CRs (one per managed directory)
├── bgp/                 # BGP/FRR network configuration and NNCP
├── config/
│   ├── gatekeeper/      # Gatekeeper Assign mutation policies
│   ├── kubeletconfig/   # KubeletConfig CRs per MachineConfigPool
│   ├── machineconfig/   # MachineConfig CRs (ignition, chrony, timezone)
│   ├── machineconfigpool/ # MachineConfigPool definitions
│   ├── metallb-l2/      # MetalLB IPAddressPool and L2Advertisement
│   └── networkpolicy/   # NetworkPolicy resources
├── monitor/
│   └── custom-alert/    # Custom Prometheus AlertingRule CRs
├── operators/           # OLM operator installations (one subdir per operator)
│   ├── cert-manager/
│   ├── cluster-observability-operator/
│   ├── compliance-operator/
│   ├── connectivity-link/
│   ├── metallb/
│   ├── openshift-logging/
│   ├── openshift-loki/
│   └── openshift-pipelines/
├── pipelines/
│   └── build-and-update/ # Tekton Pipeline, Tasks, PVC, secret templates
├── virtualization/
│   └── vm/              # KubeVirt VirtualMachine and DataVolume CRs
└── README.md
```

---

## Key Conventions

### File Naming

- All filenames use **kebab-case** (e.g., `cert-manager-app.yaml`, `master-kubeletconfig.yaml`).
- Each YAML file contains a **single Kubernetes resource** (no multi-document `---` separators).
- ArgoCD application files are named `<component>-app.yaml`.

### YAML Style

- **No trailing whitespace**, standard 2-space indentation.
- Always include `apiVersion`, `kind`, `metadata.name`, and (where required) `metadata.namespace`.
- Labels and selectors must be consistent between related resources (e.g., MachineConfigPool selectors and MachineConfig `machineconfiguration.openshift.io/role` labels).

### Operator Installation Pattern

Every operator follows this three-file layout inside `operators/<name>/`:

```
namespace.yaml        # Namespace for the operator
operator-group.yaml   # OLM OperatorGroup
subscription.yaml     # OLM Subscription
```

Subscriptions always use `installPlanApproval: Manual` — **never change this to Automatic** without explicit intent, as automatic upgrades can break cluster stability.

### ArgoCD Application Pattern

Every `argocd/<name>-app.yaml` follows this structure:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: <name>
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/Surote/ocp-configuration.git
    targetRevision: main
    path: <relative-path-to-managed-dir>
  destination:
    server: https://kubernetes.default.svc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

When adding a new managed directory, always create a matching ArgoCD Application in `argocd/`.

### MachineConfig Guidelines

- MachineConfig files embed content as **gzip + base64** encoded strings in the Ignition `source` field.
- To encode a config file for embedding:
  ```bash
  cat <file> | gzip | base64 -w0
  ```
- Use Ignition spec version `3.2.0`.
- Apply the role label: `machineconfiguration.openshift.io/role: master` or `worker`.
- See `config/machineconfig/README.md` for the chrony config example.

### KubeletConfig Guidelines

- Create **one KubeletConfig CR per MachineConfigPool** containing all changes for that pool.
- Edit an existing CR to add settings rather than creating new CRs.
- Maximum **10 KubeletConfig CRs per cluster**.
- When deleting, delete in reverse numeric suffix order (e.g., `-3` before `-2`).
- See `config/kubeletconfig/README.md` for full guidance.

---

## GitOps Workflow

1. Make changes to YAML files in the appropriate directory.
2. Commit and push to `main` (or a feature branch via PR).
3. ArgoCD detects the change and syncs the target cluster automatically.
4. `prune: true` means resources removed from git are deleted from the cluster.
5. `selfHeal: true` means manual cluster changes are reverted by ArgoCD.

> **Do not apply resources manually with `oc apply` in a directory managed by ArgoCD**, as the next sync will overwrite or prune the changes.

---

## CI/CD — Tekton Pipelines

The `pipelines/build-and-update/` pipeline:
1. Clones a source repo (`git-clone` cluster task)
2. Extracts the git commit SHA (`get-commit-sha` custom task)
3. Builds and pushes a container image (`buildah` cluster task)
4. Updates a `kustomization.yaml` in a config repo with the new image tag

**Prerequisites before running:**
- Create secrets from templates: `secret-git.yaml` (git credentials) and `secret-registry.yaml` (registry credentials).
- Link secrets to the `pipeline` ServiceAccount:
  ```bash
  oc secrets link pipeline git-auth
  oc secrets link pipeline registry-auth --for=mount
  ```
- Apply the shared PVC: `oc apply -f pvc.yaml`
- Apply the custom task: `oc apply -f get-commit-sha-task.yaml`

**Running the pipeline:**
```bash
oc create -f pipelinerun.yaml
```

> For OpenShift 4.20+, enable the **Dynamic Plugin** for OpenShift Pipelines so cluster tasks in `openshift-pipelines` namespace are accessible via the Cluster Resolver.

---

## Networking

### BGP / FRR

- `bgp/frr.yaml` — FRRConfiguration CRs for multi-VRF BGP peering (VRFs: extranet, extranet2, primary).
- `bgp/nncp.yaml` — NodeNetworkConfigurationPolicy for advanced node-level networking.
- `bgp/cudn-*.yaml` — ClusterUserDefinedNetwork resources.
- `bgp/ra-extranet.yaml` — RouterAdvertisement config.

### MetalLB

- IP pool defined in `config/metallb-l2/` (range: `172.16.28.180–172.16.28.184`).
- The MetalLB operator itself is installed via `operators/metallb/`.

### Network Policies

- `config/networkpolicy/` contains policies for OCP ingress and monitoring namespaces.

---

## API Groups Reference

| Resource Type | API Group |
|---|---|
| ArgoCD Application | `argoproj.io/v1alpha1` |
| Subscription / OperatorGroup | `operators.coreos.com/v1alpha1` |
| MachineConfig / KubeletConfig | `machineconfiguration.openshift.io/v1` |
| MetalLB IPAddressPool | `metallb.io/v1beta1` |
| FRRConfiguration | `frrk8s.metallb.io/v1beta1` |
| Gatekeeper Assign | `mutations.gatekeeper.sh/v1` |
| AlertingRule | `monitoring.openshift.io/v1` |
| VirtualMachine | `kubevirt.io/v1` |
| Pipeline / Task / PipelineRun | `tekton.dev/v1` |
| NodeNetworkConfigurationPolicy | `nmstate.io/v1` |

---

## Git Conventions

### Branch Naming

- Feature branches: `<username>/<short-description>` or `claude/<short-description>-<id>`
- Target branch for merges: `main`

### Commit Messages

Follow **Conventional Commits**:
```
feat: add compliance operator subscription
fix: move openshift-pipeline to infra node
test: bgp frr configuration
```

Common types: `feat`, `fix`, `test`, `docs`, `chore`.

### What NOT to Commit

- Actual secret values — `secret-git.yaml` and `secret-registry.yaml` are templates; never fill in real credentials.
- Binary artifacts or images other than pipeline success screenshots.
- Kubernetes resources that belong to application namespaces (this repo is cluster config only).

---

## Adding a New Operator

1. Create `operators/<operator-name>/namespace.yaml`
2. Create `operators/<operator-name>/operator-group.yaml`
3. Create `operators/<operator-name>/subscription.yaml` with `installPlanApproval: Manual`
4. Create `argocd/<operator-name>-app.yaml` pointing to `operators/<operator-name>`
5. Commit and push — ArgoCD will reconcile the rest.

---

## Common Pitfalls

- **MachineConfig changes trigger node reboots** via the MCO — be careful with mass changes to master or worker pools.
- **KubeletConfig limit is 10 per cluster** — edit existing CRs rather than creating new ones.
- **ArgoCD prune is enabled** — removing a file from git removes the resource from the cluster.
- **`installPlanApproval: Manual`** — new operator versions require manual approval in the OCP console or via `oc` before they install.
- **Ignition content must be gzip+base64 encoded** — raw text in MachineConfig source fields will not work.
- **The `pipeline` ServiceAccount is auto-created** by the OpenShift Pipelines operator; do not define it manually.
