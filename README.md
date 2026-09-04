# ocp-configuration

OpenShift 4.x cluster configuration managed via ArgoCD GitOps. All resources are plain YAML manifests — no Helm, no Kustomize at the cluster level.

---

## OpenShift AI — Model-as-a-Service (MaaS) Installation

This section documents the full GitOps-driven installation of **Red Hat OpenShift AI** with MaaS capabilities. The setup spans three ArgoCD-managed directories:

| ArgoCD Application | Path | Purpose |
|---|---|---|
| `openshift-ai-operator-set` | `operators/openshift-ai-operator-set/` | All prerequisite operators |
| `cloudnative-pg` | `operators/cloudnative-pg/` | PostgreSQL HA cluster for MaaS |
| `gatewayapi` | `config/gatewayapi/` | GatewayClass and ArgoCD RBAC for Gateway API |
| `openshift-ai-maas-config` | `config/openshift-ai-maas-config/` | MaaS runtime configuration |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      ArgoCD (openshift-gitops)                  │
├──────────────────┬──────────────────┬───────────────────────────┤
│  Operator Set    │  CloudNativePG   │  Gateway API   │  MaaS Config          │
│                  │                  │                │                       │
│  OpenShift AI    │  CNPG Operator   │  GatewayClass  │  Gateway (HTTPS)      │
│  NVIDIA GPU      │  Postgres HA     │  ArgoCD RBAC   │  TLS Cert Rotation    │
│  NFD             │  Cluster (3x)    │                │  Authorino TLS Setup  │
│  Cert Manager    │                  │                │  DB Secret Sync       │
│  Observability   │                  │                │  Managed Namespaces   │
│  Tempo           │                  │                │                       │
│  OpenTelemetry   │                  │                │                       │
│  Connectivity    │                  │                │                       │
│  Link / Kuadrant │                  │                │                       │
│  JobSet          │                  │                │                       │
│  MCP Gateway     │                  │                │                       │
└──────────────────┴──────────────────┴────────────────┴───────────────────────┘
```

---

### 1. Operators (`operators/openshift-ai-operator-set/`)

All operators use `installPlanApproval: Manual`. After ArgoCD syncs the subscriptions, approve the InstallPlan in the OCP console or via `oc` to complete installation.

| Operator | Namespace | Channel | Source |
|---|---|---|---|
| OpenShift AI (`rhods-operator`) | `redhat-ods-operator` | `stable-3.x` | `redhat-operators` |
| NVIDIA GPU (`gpu-operator-certified`) | `nvidia-gpu-operator` | `v26.3` | `certified-operators` |
| Node Feature Discovery (`nfd`) | `openshift-nfd` | `stable` | `redhat-operators` |
| Cert Manager (`openshift-cert-manager-operator`) | `cert-manager-operator` | `stable-v1` | `redhat-operators` |
| Cluster Observability (`cluster-observability-operator`) | `openshift-cluster-observability-operator` | `stable` | `redhat-operators` |
| Tempo (`tempo-product`) | `openshift-tempo-operator` | `stable` | `redhat-operators` |
| OpenTelemetry (`opentelemetry-product`) | `openshift-opentelemetry-operator` | `stable` | `redhat-operators` |
| Connectivity Link (`rhcl-operator`) | `kuadrant-system` | `stable` | `redhat-operators` |
| JobSet (`job-set`) | `openshift-jobset-operator` | `stable-v1.0` | `redhat-operators` |
| MCP Gateway (`mcp-gateway`) | `openshift-operators` | `preview` | `redhat-operators` |

**Additional resources in this directory:**

- `user-workload-monitoring-configmap.yaml` — Enables user workload monitoring (`enableUserWorkload: true`) in `openshift-monitoring`
- `kuadrant-deployment.yaml` — Deploys the `Kuadrant` CR in `kuadrant-system` with observability enabled

Each operator follows the standard three-file pattern: `<name>-namespace.yaml`, `<name>-operator-group.yaml`, `<name>-subscription.yaml`.

---

### 2. PostgreSQL Database (`operators/cloudnative-pg/`)

MaaS requires a PostgreSQL database. This is provided by the **CloudNativePG** operator running a 3-instance HA cluster.

| Resource | File | Details |
|---|---|---|
| Operator namespace | `namespace.yaml` | `openshift-cnpg` |
| OperatorGroup | `operator-group.yaml` | Cluster-scoped |
| Subscription | `subscription.yaml` | Channel `stable-v1`, source `certified-operators` |
| Database namespace | `maas-postgres-namespace.yaml` | `maas-postgres` |
| Postgres cluster | `postgres-ha-app-cluster.yaml` | 3 instances, 10Gi storage, database `app` |

**Cluster configuration highlights:**

```yaml
instances: 3
storage:
  size: 10Gi
postgresql:
  parameters:
    shared_buffers: 256MB
bootstrap:
  initdb:
    database: app
    owner: app
enableSuperuserAccess: true
affinity:
  enablePodAntiAffinity: true    # spread across nodes
```

---

### 3. Gateway API (`config/gatewayapi/`)

Before the MaaS gateway can be created, the cluster needs a **GatewayClass** and ArgoCD needs RBAC permissions to manage Gateway API resources. This directory provides both.

| Resource | File | Details |
|---|---|---|
| GatewayClass | `gatewayclass.yaml` | `openshift-default` using `openshift.io/gateway-controller/v1` |
| ClusterRole + ClusterRoleBinding | `argocd-gatewayapi-rbac.yaml` | Grants the ArgoCD application controller permissions to manage `gatewayclasses`, `gateways`, `httproutes`, and `referencegrants` |

The ClusterRole is bound to the `openshift-gitops-argocd-application-controller` ServiceAccount so ArgoCD can create and reconcile Gateway API resources across namespaces.

> **Note:** This must be synced before `openshift-ai-maas-config`, as the MaaS gateway references the `openshift-default` GatewayClass defined here.

---

### 4. MaaS Configuration (`config/openshift-ai-maas-config/`)

This directory uses **Kustomize** (the ArgoCD app points to a `kustomization.yaml`). It configures four components:

#### 4.1 Managed Namespaces

`managed-namespaces.yaml` — Ensures `redhat-ods-applications` and `openshift-ingress` namespaces exist with the `argocd.argoproj.io/managed-by: openshift-gitops` label.

#### 4.2 Gateway API (`gateway/`)

Deploys a Kubernetes Gateway API `Gateway` resource for MaaS HTTPS ingress:

- **Name:** `maas-default-gateway`
- **Namespace:** `openshift-ingress`
- **Gateway class:** `openshift-default`
- **Listener:** HTTPS on port 443 with TLS termination
- **Routes:** Allowed from all namespaces

#### 4.3 TLS Certificate Rotation (`tls-cert-rotation/`)

A **CronJob** (`tls-cert-rotator`) that runs on the 1st and 15th of each month at 03:00 UTC. It copies the cluster's default ingress TLS certificate into the `maas-gateway-tls` secret used by the Gateway.

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

#### 4.4 Authorino TLS Setup (`authorino-tls/`)

A **CronJob** (`authorino-tls-setup`) that runs weekly (Sunday 04:00 UTC) to configure Authorino for TLS-secured authentication:

1. Annotates the Authorino service for OpenShift serving cert
2. Patches the Authorino CR with TLS cert reference
3. Sets `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE` env vars on the Authorino deployment
4. Annotates the MaaS gateway for Authorino TLS bootstrap

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

#### 4.5 Database Secret Sync (`db-secret-sync/`)

A **CronJob** (`db-secret-sync`) that runs on the 1st and 15th of each month at 03:30 UTC. It reads the CloudNativePG app credentials from `maas-postgres` namespace and creates/updates the `maas-db-config` secret in `redhat-ods-applications` with the connection URL.

**Connection format:**
```
postgresql://<user>:<pass>@postgres-ha-app-rw.maas-postgres.svc.cluster.local:5432/app?sslmode=require
```

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

---

### Deployment Order

All three ArgoCD applications sync automatically. However, the logical dependency order is:

1. **Operators** — `openshift-ai-operator-set` syncs first; approve all InstallPlans
2. **Database** — `cloudnative-pg` syncs; approve the InstallPlan, then wait for the Postgres cluster to become ready
3. **Gateway API** — `gatewayapi` syncs; creates the `openshift-default` GatewayClass and grants ArgoCD the RBAC to manage Gateway API resources
4. **MaaS config** — `openshift-ai-maas-config` syncs; CronJobs run on schedule or can be triggered manually:
   ```bash
   oc create job --from=cronjob/tls-cert-rotator tls-cert-rotator-manual -n openshift-ingress
   oc create job --from=cronjob/db-secret-sync db-secret-sync-manual -n redhat-ods-applications
   oc create job --from=cronjob/authorino-tls-setup authorino-tls-setup-manual -n kuadrant-system
   ```

### Post-Installation

After all operators are installed and CronJobs have completed:

1. Verify the Postgres cluster is healthy:
   ```bash
   oc get cluster postgres-ha-app -n maas-postgres
   ```
2. Verify the `maas-db-config` secret exists:
   ```bash
   oc get secret maas-db-config -n redhat-ods-applications
   ```
3. Verify the gateway is ready:
   ```bash
   oc get gateway maas-default-gateway -n openshift-ingress
   ```
4. Access MaaS via the configured hostname
