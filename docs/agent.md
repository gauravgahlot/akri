# Akri Agent

The Agent is a DaemonSet component that runs on each node and serves as the core device plugin manager for Akri.

## Overview

**Location**: [`agent/`](../agent/)
**Binary**: `akri-agent`
**Deployment**: Kubernetes DaemonSet (one per node)
**Purpose**: Orchestrate device discovery, manage device plugins, and create Instance CRDs

## Key Responsibilities

1. **Configuration Management**: Watch Configuration CRDs and trigger discovery
2. **Discovery Handler Registry**: Maintain registry of available Discovery Handlers
3. **Device Discovery Coordination**: Route discovery requests to appropriate handlers
4. **Device Plugin Implementation**: Implement Kubernetes Device Plugin API
5. **Instance Management**: Create, update, and delete Instance CRDs
6. **Device Slot Management**: Track device allocation and reclaim unused slots

## Architecture

See [Architecture Overview](architecture.md#1-akri-agent) for detailed diagrams.

## Core Components

### 1. Discovery Configuration Controller
[`agent/src/util/discovery_configuration_controller.rs`](../agent/src/util/discovery_configuration_controller.rs)

- Watches Configuration CRDs from Kubernetes API
- Starts/stops discovery when Configurations are added/modified/deleted
- Manages error backoff for failed discoveries

### 2. Discovery Handler Manager
[`agent/src/discovery_handler_manager/`](../agent/src/discovery_handler_manager/)

- Provides gRPC registration service for Discovery Handlers
- Maintains registry: handler name → endpoint mapping
- Routes discovery requests to registered handlers
- Manages handler lifecycle and re-registration

### 3. Device Manager
[`agent/src/device_manager/`](../agent/src/device_manager/)

- In-memory store of discovered devices
- Structure: `Configuration → Vec<Device>`
- Notifies Device Plugin Manager when devices change
- Thread-safe with Arc<Mutex>

### 4. Device Plugin Manager
[`agent/src/plugin_manager/device_plugin_instance_controller.rs`](../agent/src/plugin_manager/device_plugin_instance_controller.rs)

- Creates device plugin per Configuration
- Implements Kubernetes Device Plugin API (v1 and v1beta1)
- Manages Unix domain sockets in `/var/lib/kubelet/device-plugins/`
- Creates and updates Instance CRDs
- Handles device allocation requests from kubelet

### 5. Device Plugin Slot Reclaimer
[`agent/src/plugin_manager/device_plugin_slot_reclaimer.rs`](../agent/src/plugin_manager/device_plugin_slot_reclaimer.rs)

- Periodically checks for disappeared devices
- Reclaims device slots from unavailable devices
- Updates Instance CRDs to remove nodes
- Deletes Instances when no nodes remain

## Startup Sequence

```
1. Agent starts
   ├─> Read AGENT_NODE_NAME environment variable
   ├─> Initialize Kubernetes client
   └─> Start metrics server

2. Initialize Discovery Handler Registry
   ├─> Start gRPC registration server
   └─> Listen on Unix domain socket

3. Initialize Device Manager
   └─> Create in-memory device store

4. Start Device Plugin Manager
   ├─> Create instances cache
   └─> Start device plugin controller

5. Start Device Plugin Slot Reclaimer
   └─> Periodic check for disappeared devices

6. Start Configuration Controller
   ├─> Watch Configuration CRDs
   └─> Trigger discovery for existing Configurations

7. Wait for all tasks
   └─> Run until terminated
```

## Discovery Flow

```
Configuration Created
        ↓
Configuration Controller detects
        ↓
Look up Discovery Handler in registry
        ↓
Send gRPC Discover(discovery_details, discovery_properties)
        ↓
Discovery Handler streams devices back
        ↓
Device Manager updates in-memory store
        ↓
Device Plugin Manager notified
        ↓
Create/Update Instance CRD
        ↓
Register device plugin with kubelet
        ↓
Kubelet advertises device resource
```

## Device Plugin API Implementation

The Agent implements both v1 and v1beta1 of the Kubernetes Device Plugin API.

### Registration

```
Agent → Kubelet:
  Register via /var/lib/kubelet/device-plugins/kubelet.sock
  Provide: Unix socket path, resource name (akri.sh/<config-name>)
```

### ListAndWatch

```
Kubelet → Agent:
  Call ListAndWatch()

Agent → Kubelet:
  Stream Device list with health status
  Update when devices appear/disappear
```

### Allocate

```
Kubelet → Agent:
  Allocate(device_ids) when Pod scheduled

Agent:
  1. Update Instance.deviceUsage
  2. Return AllocateResponse:
     - device_specs: Device files to mount
     - mounts: Volume mounts
     - envs: Environment variables
```

## Instance CRD Management

### Creation

When a device is discovered for the first time on a node:

```rust
create_instance(
    instance_spec,    // Device info from Discovery Handler
    name,            // Generated from config + device ID
    namespace,       // From Configuration
    owner_config_name,
    owner_config_uid,
    kube_client
)
```

### Update

When device is discovered on additional nodes (shared devices):

```rust
// Add node to Instance.spec.nodes
// Add slot to Instance.spec.deviceUsage
update_instance(instance_spec, name, namespace, kube_client)
```

### Deletion

When device disappears from all nodes:

```rust
delete_instance(name, namespace, kube_client)
```

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `AGENT_NODE_NAME` | Yes | Name of the node this Agent is running on |

### Volume Mounts

| Host Path | Container Path | Purpose |
|-----------|----------------|---------|
| `/var/lib/kubelet/device-plugins` | `/var/lib/kubelet/device-plugins` | Device plugin socket directory |
| `/var/lib/kubelet/pod-resources` | `/var/lib/kubelet/pod-resources` | Pod resource information |

### Required Permissions

**RBAC**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: akri-agent
rules:
- apiGroups: ["akri.sh"]
  resources: ["configurations"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["akri.sh"]
  resources: ["instances"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

**Security Context**:
- May need `privileged: true` for device plugin registration
- Needs access to kubelet socket directories

## Metrics

Prometheus metrics endpoint on port `8080` (default):

- Device discovery metrics
- Device plugin registration status
- Instance creation/update/delete counters

## Logging

Log levels controlled by `RUST_LOG` environment variable:

```bash
RUST_LOG=info        # Standard logging
RUST_LOG=debug       # Detailed logging
RUST_LOG=trace       # Very verbose logging
RUST_LOG=akri=trace  # Trace only Akri components
```

## Troubleshooting

### No Devices Discovered

1. Check Agent logs:
   ```bash
   kubectl logs -n akri daemonset/akri-agent -f
   ```

2. Verify Discovery Handler is registered:
   ```bash
   # Look for "RegisterDiscoveryHandler" in logs
   ```

3. Check Configuration is valid:
   ```bash
   kubectl get configuration <name> -o yaml
   ```

### Device Plugin Not Registered

1. Check kubelet socket access:
   ```bash
   kubectl logs -n akri daemonset/akri-agent | grep "device plugin"
   ```

2. Verify volume mounts:
   ```bash
   kubectl get ds akri-agent -n akri -o yaml | grep -A 5 volumeMounts
   ```

3. Check permissions:
   ```bash
   kubectl exec -n akri <agent-pod> -- ls -la /var/lib/kubelet/device-plugins/
   ```

### Instances Not Created

1. Check RBAC permissions:
   ```bash
   kubectl auth can-i create instances --as=system:serviceaccount:akri:akri-agent -n default
   ```

2. Check API server connectivity:
   ```bash
   kubectl logs -n akri daemonset/akri-agent | grep "create_instance"
   ```

## Code References

| Component | File |
|-----------|------|
| Main entry point | [`agent/src/main.rs`](../agent/src/main.rs:23) |
| Configuration controller | [`agent/src/util/discovery_configuration_controller.rs`](../agent/src/util/discovery_configuration_controller.rs) |
| Discovery handler manager | [`agent/src/discovery_handler_manager/`](../agent/src/discovery_handler_manager/) |
| Device manager | [`agent/src/device_manager/`](../agent/src/device_manager/) |
| Device plugin manager | [`agent/src/plugin_manager/device_plugin_instance_controller.rs`](../agent/src/plugin_manager/device_plugin_instance_controller.rs) |
| Slot reclaimer | [`agent/src/plugin_manager/device_plugin_slot_reclaimer.rs`](../agent/src/plugin_manager/device_plugin_slot_reclaimer.rs) |
| Device plugin v1 | [`agent/src/plugin_manager/v1.rs`](../agent/src/plugin_manager/v1.rs) |
| Device plugin v1beta1 | [`agent/src/plugin_manager/v1beta1.rs`](../agent/src/plugin_manager/v1beta1.rs) |

## See Also

- [Agent vs Discovery Handlers](agent-vs-handlers.md) - Critical distinction
- [Architecture Overview](architecture.md)
- [Discovery Handlers](discovery-handlers.md)
- [Custom Resources](custom-resources.md)
