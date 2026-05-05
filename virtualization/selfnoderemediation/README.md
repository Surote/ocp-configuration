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
