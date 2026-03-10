# MetalLB FRR BGP Troubleshooting - Full Report

## Environment

| Component | Details |
|-----------|---------|
| Platform | OpenShift 4.20 (OCP) |
| Kubernetes | v1.33.6 |
| OS | Red Hat Enterprise Linux CoreOS 9.6 |
| MetalLB | Operator-managed, namespace `metallb-system` |
| FRR-K8s | DaemonSet in `openshift-frr-k8s` namespace, FRR 8.5.3 |
| External Router | RHEL 9 VM at `172.16.28.41`, FRR 8.5.3, AS 65000 |
| Cluster AS | 64512 |
| UDN Subnet | `172.16.230.0/24` (Layer2 ClusterUserDefinedNetwork) |

### Cluster Nodes

| Node | IP | Roles | BGP Peer on External Router |
|------|----|-------|-----------------------------|
| master-1 | 172.16.28.161 | control-plane, master | No |
| master-2 | 172.16.28.162 | control-plane, master | No |
| master-3 | 172.16.28.163 | control-plane, master | Yes |
| worker-1 | 172.16.28.164 | worker | Yes |
| worker-2 | 172.16.28.165 | worker | Yes |
| worker-3 | 172.16.28.166 | worker | Yes |
| infra-1  | 172.16.28.167 | infra, worker | No |
| infra-2  | 172.16.28.168 | infra, worker | No |

---

## Goal

Advertise a UserDefinedNetwork (UDN) subnet `172.16.230.0/24` from the OpenShift
cluster to an external FRR BGP router at `172.16.28.41` (AS 65000), and receive
external routes back into the cluster.

---

## Issues Found and Fixes Applied

### Issue 1: FRRConfiguration Created in Wrong Namespace

**Symptom:** FRR prefix-lists on all OCP nodes denied all routes:
```
ip prefix-list 172.16.28.41-allowed-ipv4 seq 1 deny any
```
No routes would ever be advertised even if BGP sessions came up.

**Root Cause:** The user-created FRRConfiguration `peer-external-router` was placed
in the `metallb-system` namespace. However, FRR-K8s only watches the
`openshift-frr-k8s` namespace (configured via its `--namespace` flag). The
FRRConfiguration was silently ignored.

The only active FRRConfigurations were the MetalLB-generated per-node ones
(e.g., `metallb-worker-1.lab.rhdemo.local`) in `openshift-frr-k8s`, which used
`toAdvertise.allowed.mode: filtered` with no allowed prefixes — resulting in
"deny any".

**Fix:** Recreate the FRRConfiguration in the correct namespace:
```bash
kubectl delete frrconfiguration peer-external-router -n metallb-system

kubectl apply -f - <<'EOF'
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  name: peer-external-router-2
  namespace: openshift-frr-k8s
  labels:
    bgp-advertise: udn
spec:
  bgp:
    routers:
    - asn: 64512
      neighbors:
      - address: 172.16.28.41
        asn: 65000
        disableMP: true
        toAdvertise:
          allowed:
            mode: all
        toReceive:
          allowed:
            mode: all
      prefixes:
      - 172.16.230.0/24
EOF
```

**Result:** Prefix-lists changed to permit the UDN subnet:
```
ip prefix-list 172.16.28.41-allowed-ipv4 seq 1 permit 172.16.230.0/24
```

---

### Issue 2: External Router FRR Config Had Inline Comments

**Symptom:** FRR service was running on `172.16.28.41` but bgpd was NOT listening
on TCP port 179. From OCP nodes:
```
bash: connect: Connection refused
bash: line 1: /dev/tcp/172.16.28.41/179: Connection refused
```

All BGP sessions stuck in "Active" state (never Established).

**Root Cause:** The `/etc/frr/frr.conf` file contained inline `#` comments:
```
router bgp 65000                        # external AS
  bgp router-id 172.16.28.41        # VM IP
  neighbor 172.16.28.164 remote-as 64512  # OCP node IP
```

FRR does NOT support inline `#` comments. It parsed the entire line including
the `#` text as a single command, which failed. The systemd journal showed:
```
line 20: % Unknown command[4]:   exit-address-family
Configuration file[/etc/frr/frr.conf] processing failure: 2
restart all process 26130 exited with non-zero status 2
```

bgpd started but failed to load the BGP config, so it never bound to port 179.

Also, the config declared `frr version 9.1.0` but the installed version was 8.5.3.

**Fix:** Rewrote `/etc/frr/frr.conf` with clean syntax on the external router:
```
frr version 8.5.3
frr defaults traditional
hostname external-bgp-router
!
router bgp 65000
 bgp router-id 172.16.28.41
 no bgp ebgp-requires-policy
 neighbor 172.16.28.163 remote-as 64512
 neighbor 172.16.28.164 remote-as 64512
 neighbor 172.16.28.165 remote-as 64512
 neighbor 172.16.28.166 remote-as 64512
 !
 address-family ipv4 unicast
  neighbor 172.16.28.163 activate
  neighbor 172.16.28.164 activate
  neighbor 172.16.28.165 activate
  neighbor 172.16.28.166 activate
  network 10.123.123.0/24
 exit-address-family
!
```

Then restarted FRR:
```bash
systemctl restart frr
```

**Result:** bgpd started listening on port 179:
```
LISTEN 0 4096 0.0.0.0:179 0.0.0.0:* users:(("bgpd",pid=28557,fd=22))
```

---

### Issue 3: ebgp-requires-policy Blocking Route Exchange

**Symptom:** After BGP sessions became Established, the external router showed
`(Policy)` instead of prefix counts:
```
Neighbor        V    AS   MsgRcvd MsgSent  Up/Down  State/PfxRcd  PfxSnt
172.16.28.164   4  64512      5       4   00:01:27     (Policy)  (Policy)
```
No routes were being sent or received.

**Root Cause:** FRR 8.5.3 enforces `ebgp-requires-policy` by default. Without
explicit route-maps or policy configuration, FRR refuses to send or accept
any eBGP routes. The OCP side had this disabled (`no bgp ebgp-requires-policy`
is set by FRR-K8s), but the external router did not.

**Fix:** Added `no bgp ebgp-requires-policy` to the external router via vtysh,
then soft-reset sessions:
```bash
vtysh -c "configure terminal" \
      -c "router bgp 65000" \
      -c "no bgp ebgp-requires-policy" \
      -c "exit" -c "exit" \
      -c "write memory"

vtysh -c "clear bgp * soft"
```

**Result:** Routes began flowing. External router received `172.16.230.0/24`
from all 4 OCP peers with ECMP:
```
B>* 172.16.230.0/24 [20/0] via 172.16.28.163, ens33, weight 1
  *                        via 172.16.28.164, ens33, weight 1
  *                        via 172.16.28.165, ens33, weight 1
  *                        via 172.16.28.166, ens33, weight 1
```

---

### Issue 4: RouteAdvertisements Rejected Due to Empty FRRConfigurations

**Symptom:**
```
$ oc get routeadvertisements default
NAME      STATUS
default   Not Accepted: configuration error: FRRConfiguration
          "metallb-master-3.lab.rhdemo.local" selected for node
          "master-3.lab.rhdemo.local" has no VRF matching the
          RouteAdvertisements target VRF or any selected network
```

**Root Cause:** The RouteAdvertisements resource had `frrConfigurationSelector: {}`
(empty selector = match everything). This matched ALL 7 FRRConfigurations in
`openshift-frr-k8s`, including the 6 MetalLB-generated per-node ones that had
`routers: []` (empty, no BGP config, no VRF).

The OVN-K RouteAdvertisements controller validates every matched FRRConfiguration.
When it found `metallb-master-3.lab.rhdemo.local` with empty routers and no VRF
matching the UDN, it rejected the entire RouteAdvertisement.

**Fix:**

Step 1 - Add a label to the working FRRConfiguration:
```bash
kubectl label frrconfiguration peer-external-router-2 \
  -n openshift-frr-k8s bgp-advertise=udn
```

Step 2 - Update RouteAdvertisements to use the label selector:
```bash
kubectl apply -f - <<'EOF'
apiVersion: k8s.ovn.org/v1
kind: RouteAdvertisements
metadata:
  name: default
spec:
  advertisements:
  - PodNetwork
  frrConfigurationSelector:
    matchLabels:
      bgp-advertise: udn
  networkSelectors:
  - clusterUserDefinedNetworkSelector:
      networkSelector:
        matchLabels:
          advertise: "true"
    networkSelectionType: ClusterUserDefinedNetworks
  nodeSelector: {}
EOF
```

**Result:** RouteAdvertisements became Accepted. The OVN-K controller automatically
injected VRF configuration into FRR for route leaking between the UDN VRF and
the default VRF:
```
router bgp 64512
 address-family ipv4 unicast
  network 172.16.230.0/24
  import vrf mp2-udn-vrf           <-- imports UDN routes into default VRF
 exit-address-family

router bgp 64512 vrf mp2-udn-vrf   <-- auto-created for the UDN
 address-family ipv4 unicast
  import vrf default                <-- imports default VRF routes into UDN
 exit-address-family
```

---

### Issue 5: Neighbor Mismatch (Minor, Not Fixed)

**Observation:** The `peer-external-router-2` FRRConfiguration has no `nodeSelector`,
so all 8 nodes (including master-1, master-2, infra-1, infra-2) attempt to BGP peer
with `172.16.28.41`. But the external router only has 4 neighbors configured
(master-3, worker-1, worker-2, worker-3). The extra 4 nodes remain in "Active"
state permanently.

**Optional Fix (pick one):**

Option A - Add `nodeSelector` to FRRConfiguration to limit BGP to specific nodes:
```yaml
spec:
  nodeSelector:
    matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - master-3.lab.rhdemo.local
      - worker-1.lab.rhdemo.local
      - worker-2.lab.rhdemo.local
      - worker-3.lab.rhdemo.local
```

Option B - Add the missing nodes to the external router's FRR config:
```
neighbor 172.16.28.161 remote-as 64512
neighbor 172.16.28.162 remote-as 64512
neighbor 172.16.28.167 remote-as 64512
neighbor 172.16.28.168 remote-as 64512
```

---

## Final Working State

### BGP Sessions
```
External Router (172.16.28.41, AS 65000)
    │
    ├── Established ── master-3  (172.16.28.163, AS 64512)
    ├── Established ── worker-1  (172.16.28.164, AS 64512)
    ├── Established ── worker-2  (172.16.28.165, AS 64512)
    └── Established ── worker-3  (172.16.28.166, AS 64512)
```

### Routes
- OCP → External: `172.16.230.0/24` (UDN subnet, ECMP via 4 next-hops)
- External → OCP: `10.123.123.0/24` (not active — needs a matching kernel route
  on the external router for the `network` statement to work)

### RouteAdvertisements
- Status: **Accepted**
- VRF route leaking between `mp2-udn-vrf` and default VRF: **configured automatically**

---

## All Working Configurations (Apply in Order)

### 1. Namespace: demo-udn

The namespace must have the label `k8s.ovn.org/primary-user-defined-network` to
enable primary UDN attachment for pods.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo-udn
  labels:
    k8s.ovn.org/primary-user-defined-network: ""
```

### 2. ClusterUserDefinedNetwork: cluster-udn-prod

Defines the Layer2 UDN with subnet `172.16.230.0/24`. The label `advertise: "true"`
is used by RouteAdvertisements to select this network for BGP advertisement.

```yaml
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: cluster-udn-prod
  labels:
    advertise: "true"
spec:
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: demo-udn
  network:
    layer2:
      ipam:
        lifecycle: Persistent
      role: Primary
      subnets:
      - 172.16.230.0/24
    topology: Layer2
```

### 3. MetalLB Instance

The MetalLB operator instance in `metallb-system`. Default spec is sufficient --
it deploys the controller, speakers, and integrates with FRR-K8s.

```yaml
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
spec: {}
```

### 4. MetalLB IPAddressPool (existing, for LoadBalancer services)

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: ip-addresspool-machine
  namespace: metallb-system
spec:
  addresses:
  - 172.16.28.180-172.16.28.184
  autoAssign: true
```

### 5. FRRConfiguration: peer-external-router-2

This is the core BGP configuration. MUST be in the `openshift-frr-k8s` namespace
(FRR-K8s only watches its own namespace). The label `bgp-advertise: udn` is used
by RouteAdvertisements to select this config.

```yaml
apiVersion: frrk8s.metallb.io/v1beta1
kind: FRRConfiguration
metadata:
  name: peer-external-router-2
  namespace: openshift-frr-k8s
  labels:
    bgp-advertise: udn
spec:
  bgp:
    routers:
    - asn: 64512
      neighbors:
      - address: 172.16.28.41
        asn: 65000
        disableMP: true
        toAdvertise:
          allowed:
            mode: all
        toReceive:
          allowed:
            mode: all
      prefixes:
      - 172.16.230.0/24
```

### 6. RouteAdvertisements: default

Tells OVN-Kubernetes to advertise UDN pod network routes via FRR-K8s BGP.
The `frrConfigurationSelector` MUST use a specific label (not empty `{}`) to avoid
selecting the empty MetalLB-generated FRRConfigurations.

```yaml
apiVersion: k8s.ovn.org/v1
kind: RouteAdvertisements
metadata:
  name: default
spec:
  advertisements:
  - PodNetwork
  frrConfigurationSelector:
    matchLabels:
      bgp-advertise: udn
  networkSelectors:
  - clusterUserDefinedNetworkSelector:
      networkSelector:
        matchLabels:
          advertise: "true"
    networkSelectionType: ClusterUserDefinedNetworks
  nodeSelector: {}
```

### 7. External Router FRR Config (/etc/frr/frr.conf on 172.16.28.41)

RHEL 9 host running FRR 8.5.3. Requires `bgpd=yes` and `zebra=yes` in
`/etc/frr/daemons`. No inline `#` comments allowed.

```
frr version 8.5.3
frr defaults traditional
hostname external-bgp-router
!
router bgp 65000
 bgp router-id 172.16.28.41
 no bgp ebgp-requires-policy
 neighbor 172.16.28.163 remote-as 64512
 neighbor 172.16.28.164 remote-as 64512
 neighbor 172.16.28.165 remote-as 64512
 neighbor 172.16.28.166 remote-as 64512
 !
 address-family ipv4 unicast
  network 10.123.123.0/24
 exit-address-family
!
```

External router `/etc/frr/daemons` must have:
```
zebra=yes
bgpd=yes
```

After writing configs:
```bash
systemctl restart frr
systemctl enable frr
```

### 8. Demo Application (test workload in demo-udn)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-web
  namespace: demo-udn
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-web
  template:
    metadata:
      labels:
        app: demo-web
    spec:
      containers:
      - name: web
        image: quay.io/openshifttest/hello-openshift:1.2.0
        ports:
        - containerPort: 8080
        - containerPort: 8888
```

Pods get UDN IPs from `172.16.230.0/24` on the `ovn-udn1` interface (primary),
and cluster network IPs on `eth0` (infrastructure-locked).

### Verification from External Router

```bash
# Check BGP sessions
vtysh -c "show bgp summary"

# Check received routes
vtysh -c "show ip bgp"

# Check kernel route
ip route show 172.16.230.0/24

# Test pod reachability
curl http://172.16.230.4:8080    # → "Hello OpenShift!"
curl http://172.16.230.7:8080    # → "Hello OpenShift!"
```

---

## Key Lessons Learned

1. **FRRConfiguration namespace matters** — FRR-K8s only watches its own namespace
   (`openshift-frr-k8s`), not `metallb-system`.

2. **FRR does not support inline `#` comments** — use `!` on separate lines only.

3. **Match `frr version` to what's installed** — declaring `frr version 9.1.0` when
   8.5.3 is installed can cause parsing issues.

4. **`ebgp-requires-policy` is on by default in FRR 8.5.3** — either disable it or
   create explicit route-maps on both sides.

5. **Empty selector `{}` matches everything** — use specific `matchLabels` in
   `frrConfigurationSelector` to avoid selecting MetalLB's auto-generated empty
   FRRConfigurations.

6. **OCP FRR-K8s runs bgpd with `-p 0`** — it never listens on port 179. It only
   makes outbound connections. The external peer must accept incoming connections.

7. **BGP `network` statement requires a matching RIB route** — declaring
   `network 10.123.123.0/24` does nothing unless a route for that prefix exists
   in the kernel routing table.
