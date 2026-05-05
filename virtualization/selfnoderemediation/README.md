# Self Node Remediation

Automatic node remediation for unhealthy nodes. When a node becomes unresponsive, it fences itself and reboots to restore workloads.

## Prerequisites

Install the **Node Health Check Operator** from OperatorHub:

- **Namespace:** `openshift-workload-availability`
- **Channel:** `stable`
- **Install Plan Approval:** `Manual`

The Node Health Check Operator automatically installs the Self Node Remediation Operator as a dependency. On install, the operators auto-create:

- `SelfNodeRemediationConfig` — cluster-wide remediation settings
- `SelfNodeRemediationTemplate` — templates for both `Automatic` and `ResourceDeletion` strategies

No need to declare these manually — the operators manage their lifecycle.

## Files

| File | Description |
|------|-------------|
| `nodehealthcheck.yaml` | NodeHealthCheck CR — watches worker nodes and triggers remediation |

## How It Works

1. Install Node Health Check Operator → auto-creates SNR config and templates
2. Apply `NodeHealthCheck` CR → operator watches specified nodes
3. If node matches `unhealthyConditions` for the specified duration → remediation triggers
4. Self Node Remediation fences the node and reboots it

## Remediation Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRIGGER                                  │
│                                                                 │
│   Worker Node Fails (kernel panic, power loss, NIC failure)     │
│   Kubelet heartbeat stops                                       │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CORE                               │
│                                                                 │
│   kube-controller-manager marks node NotReady (~40s)            │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    NHC OPERATOR                                  │
│                                                                 │
│   1. NHC detects unhealthy condition                            │
│   2. Waits unhealthyConditions duration (300s)                  │
│   3. Checks minHealthy (51%) — won't remediate if too           │
│      many nodes are unhealthy                                   │
│   4. Creates SelfNodeRemediation CR from template               │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SNR OPERATOR                                  │
│                                                                 │
│   SNR Controller (on healthy nodes):                            │
│   1. Fences node — cordon + adds medik8s.io/remediation         │
│      NoExecute taint                                            │
│   2. Adds node.kubernetes.io/out-of-service:NoExecute taint     │
│                                                                 │
│   SNR Agent (DaemonSet on failing node):                        │
│   3. Attempts reboot via watchdog device or software reboot     │
│                                                                 │
│   SNR Controller:                                               │
│   4. Waits safeTimeToAssumeNodeRebootedSeconds (~185s)          │
│   5. Monitors pod deletion, removes taint when complete         │
└──────────┬──────────────────────────────────┬───────────────────┘
           │                                  │
           ▼                                  ▼
┌────────────────────────┐   ┌────────────────────────────────────┐
│   NODE REBOOTS         │   │   KUBERNETES CORE (KCM)            │
│                        │   │                                    │
│   Node restarts and    │   │   kube-controller-manager sees     │
│   rejoins cluster      │   │   out-of-service taint:            │
│                        │   │   • Force-deletes pods             │
│                        │   │   • Detaches PersistentVolumes     │
└────────────────────────┘   └─────────────────┬──────────────────┘
                                               │
                                               ▼
                             ┌─────────────────────────────────────┐
                             │   KUBEVIRT                           │
                             │                                     │
                             │   virt-controller detects VM has     │
                             │   no virt-launcher pod but           │
                             │   runStrategy requires it running    │
                             │                                     │
                             │   → Creates new virt-launcher pod    │
                             │   → Scheduler places on healthy node │
                             │   → VM COLD RESTARTS on new node     │
                             └─────────────────────────────────────┘
```

## Who Evicts the Pods and VMs?

> **Key Insight:** SNR does NOT directly delete pods (in the default OutOfServiceTaint strategy).
> It adds the `node.kubernetes.io/out-of-service` taint. The **kube-controller-manager** sees
> this taint and force-deletes pods + detaches volumes. For VMs, the KubeVirt **virt-controller**
> then detects the missing pod and cold-restarts the VM on a healthy node.

| Actor | Component | What It Does |
|-------|-----------|-------------|
| Node Controller | kube-controller-manager | Marks node `NotReady` when kubelet heartbeats stop (~40s) |
| NHC Operator | Node Health Check | Watches node conditions, waits configured duration, creates `SelfNodeRemediation` CR |
| SNR Controller | SNR manager (on healthy nodes) | Manages remediation phases: adds taints, monitors deletion, removes taints |
| SNR Agent | SNR DaemonSet (on failing node) | Performs actual reboot via watchdog device or software reboot |
| kube-controller-manager | Kubernetes core | Responds to `out-of-service` taint: **force-deletes pods, detaches PVs** |
| virt-controller | KubeVirt | Detects VM has no virt-launcher pod, **creates new pod for cold restart** |
| kube-scheduler | Kubernetes core | Schedules new virt-launcher pod on a healthy node |

### Remediation Strategies (from SNR source code)

| Strategy | Who Deletes Pods | Mechanism |
|----------|-----------------|-----------|
| **OutOfServiceTaint** (default) | kube-controller-manager | SNR adds `node.kubernetes.io/out-of-service:NoExecute` taint. KCM force-deletes pods lacking a matching toleration and detaches volumes. |
| **ResourceDeletion** | SNR controller directly | SNR calls `resources.DeletePods()` via Kubernetes API to explicitly delete each pod on the unhealthy node. |

## VM Behavior: Cold Restart, NOT Live Migration

> **Critical Distinction:** When a node is down, live migration is impossible because the source
> VM is unreachable. The VM undergoes a **cold restart** — it is stopped on the failed node and
> freshly booted on a new node. This is fundamentally different from live migration during
> planned maintenance.

### VM Recovery by runStrategy

| runStrategy | Recovery After Node Failure | Behavior |
|-------------|---------------------------|----------|
| `Always` | Auto-restart on healthy node | virt-controller creates new virt-launcher pod automatically |
| `RerunOnFailure` | Auto-restart on healthy node | VM is restarted because the previous instance failed |
| `Manual` | No auto-restart | Admin must manually start the VM with `virtctl start` |
| `Halted` | No restart | VM remains stopped |

### evictionStrategy Is for Planned Maintenance Only

| Scenario | `evictionStrategy: LiveMigrate` | `evictionStrategy: None` |
|----------|-------------------------------|------------------------|
| Planned drain (`oc adm drain`) | VM is live-migrated (no downtime) | VM is shut down |
| **Node failure (NHC/SNR)** | **NOT live-migrated** (node is dead). Cold restart via `runStrategy` | Cold restart via `runStrategy` (same behavior) |

## Timeline (Default Settings)

| Time | Duration | What Happens |
|------|----------|-------------|
| T+0 | 0s | Node fails (kernel panic, power loss, NIC failure) |
| T+40s | ~40s | kube-controller-manager marks node `NotReady` |
| T+340s | ~300s | NHC `unhealthyConditions` duration expires, creates `SelfNodeRemediation` CR |
| T+340s | Immediate | SNR fences node: cordon + `NoExecute` + `out-of-service` taints |
| T+340s | Immediate | SNR agent on failing node attempts watchdog/software reboot |
| T+340s | Seconds | KCM force-deletes pods and detaches PVs (triggered by `out-of-service` taint) |
| T+525s | ~185s | `safeTimeToAssumeNodeRebootedSeconds` passes, node assumed rebooted |
| T+525s | Seconds | virt-controller detects missing virt-launcher, creates new pod |
| T+530s | ~5s | VM boots fresh on new node |

> **Total estimated time from failure to VM recovery: ~8–9 minutes** with default settings.
> Reduce `unhealthyConditions` duration in NHC to shorten detection time.

## NodeHealthCheck Configuration

```yaml
selector:              # which nodes to monitor
  matchExpressions:
    - key: node-role.kubernetes.io/worker
      operator: Exists

remediationTemplate:   # which remediation strategy to use
  name: self-node-remediation-automatic-strategy-template

unhealthyConditions:   # when to trigger remediation
  - type: Ready
    status: "False"
    duration: 300s     # NotReady for 5 minutes
  - type: Ready
    status: Unknown
    duration: 300s     # Unknown for 5 minutes

minHealthy: 51%        # safety: don't remediate if >49% nodes unhealthy
```

### Remediation Strategies

| Template Name | Behavior |
|---------------|----------|
| `self-node-remediation-automatic-strategy-template` | Fence + reboot node |
| `self-node-remediation-resource-deletion-template` | Delete pods/VMs only, no reboot |

## Operator Installation

If not already installed, add to `operators/node-healthcheck-operator/`:

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-workload-availability
```

```yaml
# operator-group.yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: node-healthcheck-operator
  namespace: openshift-workload-availability
```

```yaml
# subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: node-healthcheck-operator
  namespace: openshift-workload-availability
spec:
  channel: stable
  name: node-healthcheck-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Manual
```
