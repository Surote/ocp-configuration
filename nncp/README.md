# NodeNetworkConfigurationPolicy (NNCP) Examples

Example [nmstate](https://nmstate.io/) `NodeNetworkConfigurationPolicy` manifests for common OpenShift secondary-networking patterns. Each file is a self-contained scenario and can be applied independently. Bond names (`bond1`–`bond4`) and NIC names (`ens5`–`ens12`) are placeholders — adjust to match your hardware before applying.

## Files

| File | Scenario |
|------|----------|
| [01-bond-ovs-bridge-virt.yaml](01-bond-ovs-bridge-virt.yaml) | LACP bond → OVS bridge with OVN `localnet` mapping, for KubeVirt secondary networks |
| [02-bond-linux-bridge.yaml](02-bond-linux-bridge.yaml) | LACP bond → Linux bridge |
| [03-bond-vlan-ipv4-dns.yaml](03-bond-vlan-ipv4-dns.yaml) | LACP bond → VLAN sub-interface with IPv4 address + DNS resolver |
| [04-bond-access-vlan-ipv4.yaml](04-bond-access-vlan-ipv4.yaml) | LACP bond with IPv4 directly on the bond (access / untagged) |

## Common Fields

### `spec.nodeSelector`

Which nodes the policy applies to. Examples use `node-role.kubernetes.io/worker: ""` (all workers). For a single node use `kubernetes.io/hostname: <node>`.

### Bond — `link-aggregation`

```yaml
mode: 802.3ad           # LACP — requires switch-side port-channel config
options:
  miimon: "100"         # link-monitor interval in ms
  lacp_rate: fast       # LACPDU every 1s (vs. slow = 30s)
  xmit_hash_policy: layer3+4   # hash on L3+L4 for better flow distribution
port:
  - ens5
  - ens6
```

| Option | Purpose |
|--------|---------|
| `mode: 802.3ad` | IEEE 802.1AX LACP. Alternatives: `active-backup`, `balance-tlb`, `balance-alb`, etc. |
| `miimon` | How often (ms) the kernel checks link state on each slave. `100` is standard. |
| `lacp_rate: fast` | Faster failure detection (~3s vs. ~90s with `slow`). Must match switch. |
| `xmit_hash_policy: layer3+4` | Distributes flows by src/dst IP + L4 port. `layer2` (default) hashes on MACs only and is poor for single-peer traffic. |
| `port` | Physical NICs enslaved to the bond. |

### `ipv4` / `ipv6`

Set `enabled: false` on the underlying bond when IP lives on a child interface (bridge, VLAN). Set `enabled: true` + `dhcp: false` + static `address` when the bond/VLAN itself owns an IP.

## Scenario-specific Notes

### 01 — OVS bridge for KubeVirt (`ovs-bridge`)

```yaml
- name: br-ex-secondary
  type: ovs-bridge
  bridge:
    allow-extra-patch-ports: true
    options:
      stp: false
    port:
      - name: bond1
ovn:
  bridge-mappings:
    - localnet: secondary-localnet
      bridge: br-ex-secondary
      state: present
```

| Option | Purpose |
|--------|---------|
| `type: ovs-bridge` | Open vSwitch bridge, required for OVN-Kubernetes `localnet` networks used by KubeVirt secondary NICs. |
| `allow-extra-patch-ports: true` | Lets OVN-K attach its own patch ports to the bridge without nmstate reverting them. **Required** when OVN manages this bridge; without it the CNO-added ports are removed on every reconcile. |
| `options.stp: false` | OVS bridge does not participate in STP (usual for fabric-facing ports). |
| `ovn.bridge-mappings[].localnet` | Name referenced by the `NetworkAttachmentDefinition` / `UserDefinedNetwork` that VMs attach to. |
| `ovn.bridge-mappings[].bridge` | Must match the OVS bridge name above. |

### 02 — Linux bridge

```yaml
- name: br1
  type: linux-bridge
  bridge:
    options:
      stp:
        enabled: false
    port:
      - name: bond2
```

Use when workloads need a plain kernel bridge (e.g., bridge CNI, non-OVN KubeVirt NADs). `stp.enabled: false` matches typical fabric-facing configs — enable STP only if the bridge could create L2 loops.

### 03 — VLAN sub-interface with IP + DNS

```yaml
- name: bond3.100
  type: vlan
  vlan:
    base-iface: bond3
    id: 100
  ipv4:
    enabled: true
    dhcp: false
    address:
      - ip: 192.168.100.10
        prefix-length: 24
dns-resolver:
  config:
    server:
      - 192.168.100.1
      - 8.8.8.8
    search:
      - lab.rhdemo.local
```

| Option | Purpose |
|--------|---------|
| `type: vlan` | Creates an 802.1Q tagged sub-interface. Naming convention `<parent>.<vlanid>` is recommended but not required. |
| `vlan.base-iface` | Parent interface (bond or ethernet). |
| `vlan.id` | 1–4094. |
| `dns-resolver` | **Cluster-wide change** — overrides `/etc/resolv.conf` on selected nodes. Use carefully: breaking DNS breaks kubelet. Prefer DHCP-provided DNS on the primary interface when possible. |

### 04 — Access VLAN, IP on the bond

```yaml
- name: bond4
  type: bond
  ...
  ipv4:
    enabled: true
    dhcp: false
    address:
      - ip: 10.10.20.10
        prefix-length: 24
```

IP configured directly on the bond — appropriate when the switch port is an access port (untagged) or the node should live in the native VLAN. No VLAN interface, no bridge.

## Validating

```bash
oc apply --dry-run=client -f nncp/<file>.yaml
yamllint nncp/<file>.yaml
```

After apply, check rollout status:

```bash
oc get nncp
oc get nnce       # per-node enactment
oc describe nncp <name>
```

## References

- [nmstate NNCP examples](https://nmstate.io/examples.html)
- [OpenShift — Updating node network configuration](https://docs.openshift.com/container-platform/latest/networking/k8s_nmstate/k8s-nmstate-updating-node-network-config.html)
- [OVN-K localnet networks for KubeVirt](https://docs.openshift.com/container-platform/latest/virt/vm_networking/virt-connecting-vm-to-ovn-secondary-network.html)
