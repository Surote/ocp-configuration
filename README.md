# ocp-configuration

OpenShift 4.x cluster configuration managed via ArgoCD GitOps. All resources are plain YAML manifests — no Helm, no Kustomize at the cluster level.

---

## OpenShift AI — Model-as-a-Service (MaaS) Installation

This section documents the full GitOps-driven installation of **Red Hat OpenShift AI** with MaaS and MLflow capabilities. The setup spans the following ArgoCD-managed directories:

| ArgoCD Application | Path | Purpose |
|---|---|---|
| `openshift-ai-operator-set` | `operators/openshift-ai-operator-set/` | All prerequisite operators |
| `openshift-logging` | `operators/openshift-logging/` | Cluster Logging operator |
| `openshift-loki` | `operators/openshift-loki/` | Loki operator for log storage |
| `cloudnative-pg` | `operators/cloudnative-pg/` | PostgreSQL HA clusters (MaaS + MLflow) |
| `gatewayapi` | `config/gatewayapi/` | GatewayClass and ArgoCD RBAC for Gateway API |
| `openshift-ai-maas-config` | `config/openshift-ai-maas-config/` | MaaS runtime configuration |
| `openshift-ai-mlflow-pg` | `config/openshift-ai-mlflow-pg/` | PostgreSQL HA cluster for MLflow |
| `openshift-ai-mlflow-dev` | `config/openshift-ai-mlflow-dev/` | MLflow experiment tracking server |

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          ArgoCD (openshift-gitops)                           │
├──────────────────┬──────────────┬────────────────┬───────────────────────────┤
│  Operator Set    │  Logging     │  CloudNativePG │  Configuration            │
│                  │              │                │                           │
│  OpenShift AI    │  Logging     │  CNPG Operator │  Gateway API              │
│  NVIDIA GPU      │  Loki        │                │  ├─ GatewayClass          │
│  NFD             │              │  MaaS DB       │  └─ ArgoCD RBAC           │
│  Cert Manager    │              │  (3x HA)       │                           │
│  Observability   │              │                │  MaaS Config              │
│  Tempo           │              │  MLflow DB     │  ├─ Gateway (HTTPS)       │
│  OpenTelemetry   │              │  (3x HA)       │  ├─ TLS Cert Rotation     │
│  Connectivity    │              │                │  ├─ Authorino TLS Setup   │
│  Link / Kuadrant │              │                │  ├─ DB Secret Sync        │
│  JobSet          │              │                │  └─ Managed Namespaces    │
│  MCP Gateway     │              │                │                           │
│                  │              │                │  MLflow                   │
│                  │              │                │  └─ Experiment Tracking   │
└──────────────────┴──────────────┴────────────────┴───────────────────────────┘
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

### 2. Logging and Loki

Log collection and storage for the cluster, managed as separate ArgoCD applications.

| Operator | Path | Subscription Name | Namespace | Channel | Source |
|---|---|---|---|---|---|
| **Cluster Logging** | `operators/openshift-logging/` | `cluster-logging` | `openshift-logging` | `stable-6.4` | `redhat-operators` |
| **Loki Operator** | `operators/openshift-loki/` | `loki-operator` | `openshift-operators-redhat` | `stable-6.4` | `redhat-operators` |

Each follows the standard three-file pattern (`namespace.yaml`, `operator-group.yaml`, `subscription.yaml`).

---

### 3. PostgreSQL Databases (`operators/cloudnative-pg/`)

The **CloudNativePG** operator provides PostgreSQL HA clusters for both MaaS and MLflow.

**Operator (in `operators/cloudnative-pg/`):**

| Resource | File | Details |
|---|---|---|
| Operator namespace | `namespace.yaml` | `openshift-cnpg` |
| OperatorGroup | `operator-group.yaml` | Cluster-scoped |
| Subscription | `subscription.yaml` | Channel `stable-v1`, source `certified-operators` |

**MaaS database (in `operators/cloudnative-pg/`):**

| Resource | File | Details |
|---|---|---|
| Database namespace | `maas-postgres-namespace.yaml` | `maas-postgres` |
| Postgres cluster | `postgres-ha-app-cluster.yaml` | 3 instances, 10Gi, database `app`, owner `app` |

**MLflow database (in `config/openshift-ai-mlflow-pg/`):**

| Resource | File | Details |
|---|---|---|
| Database namespace | `mlflow-postgres-ns.yaml` | `mlflow-postgres` |
| Postgres cluster | `mlflow-postgres-cluster.yaml` | 3 instances, 10Gi, database `mlflow`, owner `mlflow` |

Both clusters share the same HA configuration:

```yaml
instances: 3
storage:
  size: 10Gi
postgresql:
  parameters:
    shared_buffers: 256MB
enableSuperuserAccess: true
affinity:
  enablePodAntiAffinity: true    # spread across nodes
```

---

### 4. Gateway API (`config/gatewayapi/`)

Before the MaaS gateway can be created, the cluster needs a **GatewayClass** and ArgoCD needs RBAC permissions to manage Gateway API resources. This directory provides both.

| Resource | File | Details |
|---|---|---|
| GatewayClass | `gatewayclass.yaml` | `openshift-default` using `openshift.io/gateway-controller/v1` |
| ClusterRole + ClusterRoleBinding | `argocd-gatewayapi-rbac.yaml` | Grants the ArgoCD application controller permissions to manage `gatewayclasses`, `gateways`, `httproutes`, and `referencegrants` |

The ClusterRole is bound to the `openshift-gitops-argocd-application-controller` ServiceAccount so ArgoCD can create and reconcile Gateway API resources across namespaces.

> **Note:** This must be synced before `openshift-ai-maas-config`, as the MaaS gateway references the `openshift-default` GatewayClass defined here.

---

### 5. MaaS Configuration (`config/openshift-ai-maas-config/`)

This directory uses **Kustomize** (the ArgoCD app points to a `kustomization.yaml`). It configures four components:

#### 5.1 Managed Namespaces

`managed-namespaces.yaml` — Ensures `redhat-ods-applications` and `openshift-ingress` namespaces exist with the `argocd.argoproj.io/managed-by: openshift-gitops` label.

#### 5.2 Gateway API (`gateway/`)

Deploys a Kubernetes Gateway API `Gateway` resource for MaaS HTTPS ingress:

- **Name:** `maas-default-gateway`
- **Namespace:** `openshift-ingress`
- **Gateway class:** `openshift-default`
- **Listener:** HTTPS on port 443 with TLS termination
- **Routes:** Allowed from all namespaces

#### 5.3 TLS Certificate Rotation (`tls-cert-rotation/`)

A **CronJob** (`tls-cert-rotator`) that runs on the 1st and 15th of each month at 03:00 UTC. It copies the cluster's default ingress TLS certificate into the `maas-gateway-tls` secret used by the Gateway.

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

#### 5.4 Authorino TLS Setup (`authorino-tls/`)

A **CronJob** (`authorino-tls-setup`) that runs weekly (Sunday 04:00 UTC) to configure Authorino for TLS-secured authentication:

1. Annotates the Authorino service for OpenShift serving cert
2. Patches the Authorino CR with TLS cert reference
3. Sets `SSL_CERT_FILE` and `REQUESTS_CA_BUNDLE` env vars on the Authorino deployment
4. Annotates the MaaS gateway for Authorino TLS bootstrap

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

#### 5.5 Database Secret Sync (`db-secret-sync/`)

A **CronJob** (`db-secret-sync`) that runs on the 1st and 15th of each month at 03:30 UTC. It reads the CloudNativePG app credentials from `maas-postgres` namespace and creates/updates the `maas-db-config` secret in `redhat-ods-applications` with the connection URL.

**Connection format:**
```
postgresql://<user>:<pass>@postgres-ha-app-rw.maas-postgres.svc.cluster.local:5432/app?sslmode=require
```

**Resources:** ServiceAccount, ClusterRole, ClusterRoleBinding, CronJob

---

### 6. MLflow Experiment Tracking (`config/openshift-ai-mlflow-dev/`)

**MLflow** provides experiment tracking, model registry, and artifact storage for AI/ML workflows within OpenShift AI.

| Resource | File | Details |
|---|---|---|
| MLflow CR | `config/openshift-ai-mlflow-dev/mlflow.yaml` | Deployed in `redhat-ods-applications` |

**Configuration:**

```yaml
apiVersion: mlflow.opendatahub.io/v1
kind: MLflow
metadata:
  name: mlflow
  namespace: redhat-ods-applications
spec:
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  backendStoreUri: "sqlite:////mlflow/mlflow.db"
  artifactsDestination: "file:///mlflow/artifacts"
```

The MLflow PostgreSQL cluster (section 3) is available at `postgres-ha-app-rw.mlflow-postgres.svc.cluster.local:5432` for future migration from SQLite to PostgreSQL-backed tracking.

---

### Deployment Order

All three ArgoCD applications sync automatically. However, the logical dependency order is:

1. **Operators** — `openshift-ai-operator-set`, `openshift-logging`, `openshift-loki` sync first; approve all InstallPlans
2. **Databases** — `cloudnative-pg` and `openshift-ai-mlflow-pg` sync; approve the CNPG InstallPlan, then wait for both Postgres clusters to become ready
3. **Gateway API** — `gatewayapi` syncs; creates the `openshift-default` GatewayClass and grants ArgoCD the RBAC to manage Gateway API resources
4. **MaaS config** — `openshift-ai-maas-config` syncs; CronJobs run on schedule or can be triggered manually:
   ```bash
   oc create job --from=cronjob/tls-cert-rotator tls-cert-rotator-manual -n openshift-ingress
   oc create job --from=cronjob/db-secret-sync db-secret-sync-manual -n redhat-ods-applications
   oc create job --from=cronjob/authorino-tls-setup authorino-tls-setup-manual -n kuadrant-system
   ```
5. **MLflow** — `openshift-ai-mlflow-dev` syncs; deploys the MLflow experiment tracking server

### Post-Installation

After all operators are installed and CronJobs have completed:

1. Verify both Postgres clusters are healthy:
   ```bash
   oc get cluster postgres-ha-app -n maas-postgres
   oc get cluster postgres-ha-app -n mlflow-postgres
   ```
2. Verify the `maas-db-config` secret exists:
   ```bash
   oc get secret maas-db-config -n redhat-ods-applications
   ```
3. Verify the gateway is ready:
   ```bash
   oc get gateway maas-default-gateway -n openshift-ingress
   ```
4. Verify MLflow is running:
   ```bash
   oc get mlflow mlflow -n redhat-ods-applications
   ```
5. Access MaaS via the configured hostname
