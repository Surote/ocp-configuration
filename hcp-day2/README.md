# HCP Day2 Operations — Bare Metal (Agent Platform)

Day 2 configuration for OpenShift Hosted Control Planes (HCP / Hypershift) on bare metal
using the Agent platform (Assisted Installer).

---

## Architecture — How the Hosted Cluster Connects to the Hosted Control Plane

In HCP, the control plane and the data plane run in **separate locations**:

```
Management Cluster
└── namespace: clusters
    ├── HostedCluster CR            ← defines cluster config
    ├── NodePool CR                 ← defines worker config
    └── hosted control plane pods
        ├── kube-apiserver
        ├── etcd
        ├── konnectivity-server     ← sidecar of kube-apiserver
        └── oauth-server

Bare Metal Workers (Agents)
└── each node runs:
    ├── kubelet                     → connects to kube-apiserver (port 6443)
    └── konnectivity-agent          → reverse tunnel to konnectivity-server
```

### Konnectivity Tunnel

Konnectivity provides bidirectional communication between the control plane and worker
nodes even though the control plane is in the management cluster.

- **konnectivity-server** runs as a sidecar next to kube-apiserver in the management cluster.
- **konnectivity-agent** runs as a DaemonSet on every worker node in the hosted cluster.
- At startup, each agent dials **outbound** to the konnectivity-server on port **6443**
  (same endpoint as kube-apiserver), establishing a reverse tunnel.
- The control plane uses this tunnel to reach pods and services in the hosted cluster.

**Required network access from workers to management cluster:**

| Destination | Port | Purpose |
|---|---|---|
| `api.<CLUSTER_NAME>.<BASE_DOMAIN>` | 6443 | kube-apiserver + konnectivity |
| `oauth.<CLUSTER_NAME>.<BASE_DOMAIN>` | 443 | OAuth server |
| Ignition endpoint (Route/NodePort) | 443 / NodePort | Initial node bootstrap |

### Ignition Bootstrap Flow

When a bare-metal agent is provisioned into the hosted cluster:

1. Agent contacts Assisted Service → gets ignition endpoint URL from the HostedCluster status.
2. Node downloads ignition config from the Ignition service (exposed via Route or NodePort).
3. Node boots, kubelet starts, connects to kube-apiserver.
4. konnectivity-agent starts, opens reverse tunnel to konnectivity-server.

### Exposed Services (spec.services — IMMUTABLE after cluster creation)

| Service | Recommended (bare metal) | Notes |
|---|---|---|
| `APIServer` | `LoadBalancer` with hostname | Or `NodePort` if no MetalLB |
| `OAuthServer` | `Route` | Uses management cluster ingress |
| `Konnectivity` | `Route` | Uses management cluster ingress |
| `Ignition` | `Route` | Uses management cluster ingress |
| `OVNSbDb` | `Route` | OVN southbound DB |

> **spec.services is immutable.** You cannot change the publishing strategy after the
> HostedCluster is created. Plan this before first deploy.

---

## Directory Structure

```
hcp-day2/
├── hostedcluster/
│   ├── hostedcluster-baremetal.yaml          # Reference CR — LoadBalancer API
│   ├── hostedcluster-nodeport-baremetal.yaml # Reference CR — NodePort API
│   ├── api-custom-cert-secret.yaml           # TLS secret template for custom API cert
│   └── api-custom-cert-patch.yaml            # Patch to add custom cert (mutable)
├── authentication/
│   ├── htpasswd-secret.yaml                  # HTPasswd secret template (clusters ns)
│   └── hostedcluster-oauth-patch.yaml        # Patch to add OAuth IDP (mutable)
└── nodepools/
    ├── worker-nodepool.yaml                  # Standard compute NodePool
    └── infra-nodepool.yaml                   # Infra NodePool with taint
```

All resources in `hostedcluster/` and `authentication/` are applied to the
**management cluster**, in the namespace where the HostedCluster lives (default: `clusters`).

---

## Day2 Operations

### 1. Replace / Add Custom API Server FQDN Certificate

The API server FQDN is set via `spec.services[APIServer].servicePublishingStrategy`
and is **immutable after creation**.

What IS mutable post-creation: the **TLS certificate** served for that FQDN.

**Steps:**

```bash
# 1. Generate a TLS cert with the correct SAN (do NOT include api-int.*)
openssl req -x509 -newkey rsa:4096 -keyout tls.key -out tls.crt \
  -days 365 -nodes \
  -subj "/CN=api.<CLUSTER_NAME>.<BASE_DOMAIN>" \
  -addext "subjectAltName=DNS:api.<CLUSTER_NAME>.<BASE_DOMAIN>"

# 2. Create the secret in the clusters namespace (NOT openshift-config)
oc create secret tls api-custom-cert \
  --cert=tls.crt --key=tls.key \
  -n clusters

# 3. Patch the HostedCluster
oc patch hostedcluster <CLUSTER_NAME> -n clusters --type=merge \
  --patch-file hostedcluster/api-custom-cert-patch.yaml
```

**Verify:**
```bash
oc get hostedcluster <CLUSTER_NAME> -n clusters -o jsonpath='{.spec.configuration.apiServer}'
```

> Note: `spec.configuration.apiServer.servingCerts.namedCertificates` is also used by
> the managed OAuth server. Any cert matching the OAuth hostname is automatically served there.

---

### 2. Add Authentication (HTPasswd Identity Provider)

In HCP, the OAuth configuration lives in **`spec.configuration.oauth`** of the
HostedCluster CR, not in a standalone OAuth CR inside the hosted cluster.

**Important differences from standard OCP:**
- HTPasswd secret must be in the **clusters namespace** (same as HostedCluster),
  not in `openshift-config`.
- Must also be listed in `spec.configuration.secretRefs`.
- Adding any IDP **disables the kubeadmin password**.

**Steps:**

```bash
# 1. Generate htpasswd file
htpasswd -c -B -b users.htpasswd admin <password>
htpasswd -B -b users.htpasswd developer <password>   # add more users

# 2. Create secret in the clusters namespace
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n clusters

# 3. Patch the HostedCluster to add the IDP
oc patch hostedcluster <CLUSTER_NAME> -n clusters --type=merge \
  --patch-file authentication/hostedcluster-oauth-patch.yaml
```

**Verify:**
```bash
# Check HostedCluster conditions
oc get hostedcluster <CLUSTER_NAME> -n clusters -o jsonpath='{.status.conditions}' | jq .

# Test login via the hosted cluster kubeconfig
export KUBECONFIG=/path/to/hosted-cluster-kubeconfig
oc login -u admin -p <password> https://api.<CLUSTER_NAME>.<BASE_DOMAIN>:6443
```

**Update htpasswd (add/remove users):**
```bash
# Edit the existing secret in place
htpasswd -B -b users.htpasswd newuser <password>
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=users.htpasswd \
  -n clusters \
  --dry-run=client -o yaml | oc apply -f -
```

---

### 3. Scale NodePool (add / remove workers)

```bash
# Scale up
oc scale nodepool <CLUSTER_NAME>-worker -n clusters --replicas=5

# Scale down
oc scale nodepool <CLUSTER_NAME>-worker -n clusters --replicas=3

# Check status
oc get nodepool -n clusters
```

---

### 4. Upgrade NodePool (rolling worker upgrade)

```bash
# Update the release image in the NodePool
oc patch nodepool <CLUSTER_NAME>-worker -n clusters --type=merge \
  -p '{"spec":{"release":{"image":"quay.io/openshift-release-dev/ocp-release:<NEW_VERSION>-x86_64"}}}'

# Monitor progress
oc get nodepool <CLUSTER_NAME>-worker -n clusters -w
```

---

### 5. Upgrade the Hosted Control Plane

```bash
# Update the release image in the HostedCluster
oc patch hostedcluster <CLUSTER_NAME> -n clusters --type=merge \
  -p '{"spec":{"release":{"image":"quay.io/openshift-release-dev/ocp-release:<NEW_VERSION>-x86_64"}}}'

# Monitor rollout
oc get hostedcluster <CLUSTER_NAME> -n clusters -w
```

> Upgrade the control plane before upgrading NodePools to maintain skew compatibility.

---

## Placeholder Reference

| Placeholder | Description |
|---|---|
| `<CLUSTER_NAME>` | Name of the HostedCluster resource |
| `<BASE_DOMAIN>` | DNS base domain (e.g., `example.com`) |
| `<OCP_VERSION>` | OpenShift release version (e.g., `4.20.0`) |
| `<AGENT_NAMESPACE>` | Namespace where Assisted Installer agents register |
| `<MANAGEMENT_NODE_IP>` | IP of a management cluster node (NodePort strategy) |
| `<PULL_SECRET_NAME>` | Name of the pull secret in the clusters namespace |
| `<SSH_KEY_SECRET_NAME>` | Name of the SSH key secret in the clusters namespace |
| `<INFRA_ID>` | Infrastructure ID (usually same as cluster name) |

---

## Key Differences from Standard OCP Day2

| Topic | Standard OCP | HCP |
|---|---|---|
| OAuth config | `OAuth` CR in hosted cluster | `spec.configuration.oauth` in HostedCluster |
| HTPasswd secret namespace | `openshift-config` | `clusters` (HostedCluster namespace) |
| API server cert | `APIServer` CR in hosted cluster | `spec.configuration.apiServer` in HostedCluster |
| Worker provisioning | MachineSet / BareMetalHost | NodePool + Agent |
| Control plane location | Same cluster | Management cluster namespace |
