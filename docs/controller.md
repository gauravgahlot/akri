# Akri Controller

The Controller is a cluster-wide component that watches Instance CRDs and deploys broker workloads for discovered devices.

## Overview

**Location**: [`controller/`](../controller/)
**Binary**: `akri-controller`
**Deployment**: Kubernetes Deployment (single replica, can use leader election)
**Purpose**: Deploy broker Pods/Jobs and Services for discovered devices

## Key Responsibilities

1. **Instance Watching**: Monitor Instance CRDs across all namespaces
2. **Broker Deployment**: Create/update/delete broker Pods or Jobs
3. **Service Management**: Create instance and configuration services
4. **Node Monitoring**: Watch for node disappearance and clean up orphaned resources
5. **Pod State Tracking**: Monitor broker Pod health and update metrics

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Akri Controller                         │
│                                                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │        Instance Action Handler                   │   │
│  │  - Watches Instance ADDED/MODIFIED/DELETED       │   │
│  └─────────────┬───────────────────────────────────┘   │
│                │                                         │
│      ┌─────────┴─────────┬──────────────┐              │
│      ▼                    ▼              ▼              │
│  ┌─────────┐      ┌─────────────┐  ┌─────────────┐    │
│  │ Broker  │      │   Service   │  │  Resource   │    │
│  │ Manager │      │   Manager   │  │   Cleanup   │    │
│  └─────────┘      └─────────────┘  └─────────────┘    │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │            Node Watcher                          │   │
│  │  - Detects node disappearance                   │   │
│  │  - Cleans up orphaned Instances                 │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │         Broker Pod Watcher                       │   │
│  │  - Monitors broker Pod state changes            │   │
│  │  - Updates Prometheus metrics                   │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Instance Action Handler
[`controller/src/util/instance_action.rs`](../controller/src/util/instance_action.rs)

Watches Instance CRDs and coordinates broker deployment:
- Handles existing Instances on startup
- Processes Instance ADDED/MODIFIED/DELETED events
- Determines appropriate action based on Instance state
- Synchronizes operations to prevent race conditions

### 2. Broker Pod Manager

Creates and manages broker Pods or Jobs based on Configuration.brokerSpec:

**For BrokerPodSpec**:
- Creates long-running Pods
- Sets node selector/affinity
- Adds resource requests/limits
- Injects environment variables from Instance.brokerProperties
- Sets owner reference to Instance (for garbage collection)

**For BrokerJobSpec**:
- Creates one-time Jobs
- Similar configuration as Pods
- Jobs can have different restart policies

### 3. Service Manager

Creates two types of services:

**Instance Service** (one per Instance):
```yaml
name: akri-<config-name>-<instance-hash>-svc
selector:
  akri.sh/instance: <instance-name>
```

**Configuration Service** (one per Configuration):
```yaml
name: akri-<config-name>-svc
selector:
  akri.sh/configuration: <config-name>
```

### 4. Node Watcher
[`controller/src/util/node_watcher.rs`](../controller/src/util/node_watcher.rs)

- Watches Node resources
- Detects when nodes disappear (deleted or NotReady)
- Cleans up:
  - Removes disappeared nodes from Instance.spec.nodes
  - Deletes Instances if no nodes remain
  - Ensures orphaned broker Pods are cleaned up

### 5. Broker Pod Watcher
[`controller/src/util/pod_watcher.rs`](../controller/src/util/pod_watcher.rs)

- Watches broker Pods with `akri.sh/configuration` label
- Updates Prometheus metrics:
  - `akri_broker_pod_count` by Configuration and Node
- Tracks Pod state transitions

## Startup Sequence

```
1. Controller starts
   ├─> Initialize Kubernetes client
   └─> Start metrics server

2. Handle existing Instances
   └─> Process all Instances already in cluster

3. Start watchers (parallel tasks):
   ├─> Instance watcher (main reconciliation loop)
   ├─> Node watcher (detect node disappearance)
   └─> Broker Pod watcher (update metrics)

4. Continuous reconciliation
   └─> React to Instance/Node/Pod events
```

## Instance Event Handling

### Instance ADDED

```
1. New Instance detected
   ↓
2. Get parent Configuration
   ↓
3. Check if brokerSpec is defined
   ↓
4. For each node in Instance.spec.nodes:
   ├─> Create broker Pod with:
   │   ├─> Resource requests: akri.sh/<config-name>: "1"
   │   ├─> Node selector: kubernetes.io/hostname=<node>
   │   ├─> Env vars from Instance.brokerProperties
   │   └─> Owner reference to Instance
   ↓
5. Create Instance Service (if instanceServiceSpec defined)
   ↓
6. Create/Update Configuration Service (if configurationServiceSpec defined)
```

### Instance MODIFIED

```
1. Instance updated (e.g., nodes list changed)
   ↓
2. Reconcile broker Pods:
   ├─> Add Pods for new nodes
   └─> Remove Pods for removed nodes (via owner refs)
   ↓
3. Update Services if needed
```

### Instance DELETED

```
1. Instance deleted
   ↓
2. Owner references trigger:
   ├─> Broker Pods deleted
   └─> Instance Service deleted
   ↓
3. Configuration Service updated:
   └─> Selector no longer matches deleted Instance
```

## Broker Pod Specifications

### Environment Variables

Broker Pods receive environment variables from `Instance.brokerProperties`:

```yaml
env:
- name: RESOLUTION
  value: "1920x1080"  # From Configuration.brokerProperties
- name: ONVIF_DEVICE_IP_ADDRESS
  value: "192.168.1.10"  # From Discovery Handler
- name: ONVIF_DEVICE_SERVICE_URL
  value: "http://192.168.1.10/onvif/device_service"
```

### Resource Requests

```yaml
resources:
  requests:
    akri.sh/<configuration-name>: "1"
  limits:
    akri.sh/<configuration-name>: "1"
```

### Labels

```yaml
labels:
  akri.sh/configuration: <config-name>
  akri.sh/instance: <instance-name>
  akri.sh/target-node: <node-name>
```

### Owner References

```yaml
ownerReferences:
- apiVersion: akri.sh/v0
  kind: Instance
  name: <instance-name>
  uid: <instance-uid>
  controller: true
  blockOwnerDeletion: true
```

## Configuration

### Environment Variables

None required (Controller discovers API server automatically).

### Required Permissions

**RBAC**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: akri-controller
rules:
- apiGroups: ["akri.sh"]
  resources: ["configurations", "instances"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
```

## Metrics

Prometheus metrics exposed on port `8080`:

**`akri_broker_pod_count`** (IntGaugeVec):
- Labels: `configuration`, `node`
- Value: Number of broker Pods running

Example:
```
akri_broker_pod_count{configuration="onvif-cameras",node="worker-1"} 2
akri_broker_pod_count{configuration="onvif-cameras",node="worker-2"} 1
akri_broker_pod_count{configuration="udev-video",node="worker-1"} 1
```

## Troubleshooting

### Instances Created but No Broker Pods

1. Check Controller logs:
   ```bash
   kubectl logs -n akri deployment/akri-controller -f
   ```

2. Verify Configuration has brokerSpec:
   ```bash
   kubectl get configuration <name> -o jsonpath='{.spec.brokerSpec}'
   ```

3. Check RBAC permissions:
   ```bash
   kubectl auth can-i create pods --as=system:serviceaccount:akri:akri-controller
   ```

### Broker Pods Not Scheduled

1. Check Pod events:
   ```bash
   kubectl describe pod <broker-pod>
   ```

2. Verify node has device resource:
   ```bash
   kubectl describe node <node> | grep akri.sh
   ```

3. Check resource requests match available:
   ```bash
   kubectl get pod <broker-pod> -o jsonpath='{.spec.containers[0].resources}'
   ```

### Services Not Created

1. Verify serviceSpec in Configuration:
   ```bash
   kubectl get configuration <name> -o jsonpath='{.spec.instanceServiceSpec}'
   ```

2. Check Controller logs for service creation:
   ```bash
   kubectl logs -n akri deployment/akri-controller | grep "service"
   ```

3. List services:
   ```bash
   kubectl get services -l akri.sh/configuration=<config-name>
   ```

## Code References

| Component | File |
|-----------|------|
| Main entry point | [`controller/src/main.rs`](../controller/src/main.rs:22) |
| Instance action handler | [`controller/src/util/instance_action.rs`](../controller/src/util/instance_action.rs) |
| Node watcher | [`controller/src/util/node_watcher.rs`](../controller/src/util/node_watcher.rs) |
| Pod watcher | [`controller/src/util/pod_watcher.rs`](../controller/src/util/pod_watcher.rs) |
| Pod action utilities | [`controller/src/util/pod_action.rs`](../controller/src/util/pod_action.rs) |

## See Also

- [Architecture Overview](architecture.md)
- [Agent Component](agent.md)
- [Custom Resources](custom-resources.md)
- [Broker Development](broker-development.md)
