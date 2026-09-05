# OpenShift AI Operator Set

This directory contains all operators and supporting resources required for a **Red Hat OpenShift AI** deployment with Model-as-a-Service (MaaS) capabilities.

All subscriptions use `installPlanApproval: Manual` — new operator versions must be approved before they install.

## Operators

Each operator follows the three-file pattern: `<name>-namespace.yaml`, `<name>-operator-group.yaml`, `<name>-subscription.yaml`.

| Operator | Subscription Name | Namespace | Channel | Source | Purpose |
|---|---|---|---|---|---|
| **OpenShift AI** | `rhods-operator` | `redhat-ods-operator` | `stable-3.x` | `redhat-operators` | Core AI/ML platform — model serving, training, notebooks |
| **NVIDIA GPU** | `gpu-operator-certified` | `nvidia-gpu-operator` | `v26.3` | `certified-operators` | GPU device plugin, drivers, and monitoring for AI workloads |
| **Node Feature Discovery** | `nfd` | `openshift-nfd` | `stable` | `redhat-operators` | Detects hardware features (GPUs, NICs) and labels nodes — prerequisite for the GPU operator |
| **Cert Manager** | `openshift-cert-manager-operator` | `cert-manager-operator` | `stable-v1` | `redhat-operators` | Automated TLS certificate lifecycle management |
| **Cluster Observability** | `cluster-observability-operator` | `openshift-cluster-observability-operator` | `stable` | `redhat-operators` | Unified observability stack (metrics, dashboards) |
| **Tempo** | `tempo-product` | `openshift-tempo-operator` | `stable` | `redhat-operators` | Distributed tracing backend |
| **OpenTelemetry** | `opentelemetry-product` | `openshift-opentelemetry-operator` | `stable` | `redhat-operators` | Trace/metric collection and export via OpenTelemetry Collector |
| **Connectivity Link** | `rhcl-operator` | `kuadrant-system` | `stable` | `redhat-operators` | API gateway policies — rate limiting, auth, and DNS via Kuadrant |
| **JobSet** | `job-set` | `openshift-jobset-operator` | `stable-v1.0` | `redhat-operators` | Manages groups of Kubernetes Jobs — used for distributed training |
| **MCP Gateway** | `mcp-gateway` | `openshift-operators` | `preview` | `redhat-operators` | Model Context Protocol gateway for AI tool integration |

## Additional Resources

| File | Kind | Purpose |
|---|---|---|
| `user-workload-monitoring-configmap.yaml` | `ConfigMap` | Enables user workload monitoring in `openshift-monitoring` (`enableUserWorkload: true`) |
| `kuadrant-deployment.yaml` | `Kuadrant` | Deploys the Kuadrant CR in `kuadrant-system` with observability enabled |

## File Listing

```
openshift-ai-operator-set/
├── cert-manager-namespace.yaml
├── cert-manager-operator-group.yaml
├── cert-manager-subscription.yaml
├── cluster-observability-namespace.yaml
├── cluster-observability-operator-group.yaml
├── cluster-observability-subscription.yaml
├── connectivity-link-namespace.yaml
├── connectivity-link-operator-group.yaml
├── connectivity-link-subscription.yaml
├── jobset-namespace.yaml
├── jobset-operator-group.yaml
├── jobset-subscription.yaml
├── kuadrant-deployment.yaml
├── kuadrant-namespace.yaml
├── mcp-gateway-namespace.yaml
├── mcp-gateway-operator-group.yaml
├── mcp-gateway-subscription.yaml
├── nfd-namespace.yaml
├── nfd-operator-group.yaml
├── nfd-subscription.yaml
├── nvidia-gpu-namespace.yaml
├── nvidia-gpu-operator-group.yaml
├── nvidia-gpu-subscription.yaml
├── openshift-ai-namespace.yaml
├── openshift-ai-operator-group.yaml
├── openshift-ai-subscription.yaml
├── opentelemetry-namespace.yaml
├── opentelemetry-operator-group.yaml
├── opentelemetry-subscription.yaml
├── tempo-namespace.yaml
├── tempo-operator-group.yaml
├── tempo-subscription.yaml
└── user-workload-monitoring-configmap.yaml
```

## Post-Sync

After ArgoCD syncs this directory, approve each operator's InstallPlan:

```bash
oc get installplan -A | grep -v Complete
```
