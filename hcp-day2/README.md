# HCP Day2 Operations

Day 2 configuration for OpenShift Hosted Control Planes (HCP / Hypershift).

## Directory Structure

```
hcp-day2/
├── nodepools/          # NodePool CRs — applied to the management cluster
├── authentication/     # OAuth and identity provider config
├── autoscaling/        # ClusterAutoscaler and MachineAutoscaler
├── imageregistry/      # Internal image registry configuration and PVC
├── alerting/           # Custom Prometheus alerting rules
├── networkpolicy/      # Network policy baseline templates
├── resourcequota/      # LimitRange and ResourceQuota templates
└── operators/          # Operator installations for the hosted cluster
    ├── cert-manager/
    └── openshift-logging/
```

## Where Resources Are Applied

HCP resources belong to two different clusters:

| Directory | Target Cluster | ArgoCD App |
|---|---|---|
| `nodepools/` | **Management cluster** | `hcp-day2-nodepools` |
| All others | **Hosted cluster** | `hcp-day2-hosted` |

The ArgoCD Application definitions are in `/argocd/`.

---

## Prerequisites

### 1. Register the Hosted Cluster in ArgoCD

Before the `hcp-day2-hosted` ArgoCD Application can sync, register the hosted cluster as an ArgoCD destination:

```bash
# Extract the hosted cluster kubeconfig
oc extract -n clusters secret/<CLUSTER_NAME>-admin-kubeconfig --to=- > /tmp/hosted-kubeconfig

# Add to ArgoCD
argocd cluster add <KUBECONFIG_CONTEXT> \
  --name <CLUSTER_NAME> \
  --kubeconfig /tmp/hosted-kubeconfig
```

Then update `argocd/hcp-day2-hosted-app.yaml` with the hosted cluster's API server URL.

### 2. NodePool Namespace

NodePool and HostedCluster CRs live in a namespace on the management cluster. The default is `clusters`. Ensure the namespace exists before applying:

```bash
oc get namespace clusters
```

---

## Placeholder Values

Files in this directory use placeholders that must be replaced before applying:

| Placeholder | Description |
|---|---|
| `<CLUSTER_NAME>` | Name of the HostedCluster (e.g., `my-hcp-cluster`) |
| `<OCP_VERSION>` | OpenShift release version (e.g., `4.16.10`) |
| `<NAMESPACE>` | Target namespace for resource/quota templates |
| `<HOSTED_CLUSTER_API_URL>` | API server URL of the hosted cluster |
| `<STORAGE_CLASS_NAME>` | StorageClass for the image registry PVC |
| `<BASE64_ENCODED_HTPASSWD_FILE>` | Base64-encoded htpasswd file content |

---

## NodePools

Two NodePool templates are provided:

- **`worker-nodepool.yaml`** — Standard compute nodes (3 replicas, `Agent` platform type)
- **`infra-nodepool.yaml`** — Infrastructure nodes (2 replicas, tainted with `node-role.kubernetes.io/infra:NoSchedule`)

The `Agent` platform type is used for bare-metal deployments via the Assisted Installer. Change `spec.platform.type` to `AWS`, `Azure`, or `KubeVirt` as needed.

`autoRepair: true` enables automatic node replacement on failure. `upgradeType: Replace` ensures nodes are replaced during upgrades rather than in-place updated.

---

## Authentication

`authentication/oauth.yaml` configures HTPasswd as the identity provider.

To create the required secret:
```bash
htpasswd -c -B -b users.htpasswd <username> <password>
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n openshift-config
```

**Do not commit real credentials.** The `htpasswd-secret.yaml` is a template only.

---

## Autoscaling

- **`cluster-autoscaler.yaml`** — Cluster-wide autoscaler limits (max 20 nodes, 128 cores, 256 GiB memory).
- **`worker-machine-autoscaler.yaml`** — Scales the worker MachineDeployment between 2–10 nodes.

Adjust `minReplicas` / `maxReplicas` and `resourceLimits` to match your environment.

---

## Image Registry

`imageregistry/config.yaml` enables the internal registry with a PVC-backed storage.

Apply the PVC first:
```bash
oc apply -f imageregistry/pvc.yaml
oc apply -f imageregistry/config.yaml
```

Update `<STORAGE_CLASS_NAME>` in `pvc.yaml` to a `ReadWriteOnce`-capable StorageClass.

---

## Alerting

Three alert groups are provided:

| File | Alerts |
|---|---|
| `alert-node-not-ready.yaml` | NodeNotReady, NodeMemoryPressure, NodeDiskPressure |
| `alert-etcd.yaml` | EtcdHighLeaderChanges, EtcdHighRoundTripTime |
| `alert-api-availability.yaml` | APIServerHighErrorRate, APIServerHighLatency |

---

## Network Policy Templates

The files in `networkpolicy/` are **namespace-scoped templates**. Replace `<NAMESPACE>` and apply per-namespace as needed. Recommended baseline for each application namespace:

1. `np-default-deny.yaml` — deny all ingress and egress by default
2. `np-allow-same-namespace.yaml` — allow intra-namespace traffic
3. `np-allow-ingress.yaml` — allow traffic from OCP router
4. `np-allow-monitoring.yaml` — allow Prometheus scraping

---

## Resource Quota Templates

Similarly, `resourcequota/` files are templates to apply per namespace. Replace `<NAMESPACE>` with the target project name.

---

## Operators

Follow the same three-file pattern as the rest of this repository (`namespace` → `operator-group` → `subscription`). All subscriptions use `installPlanApproval: Manual`.

Additional operator configurations:
- `operators/openshift-logging/cluster-logging.yaml` — configures log collection via Vector and forwarding to LokiStack.
