# Custom Resources

Akri extends Kubernetes with two Custom Resource Definitions (CRDs):
1. **Configuration** - Defines what devices to discover and how to use them
2. **Instance** - Represents a specific discovered device

## Quick Visual Reference

```
┌─────────────────────────────────────────────────────────────────────────┐
│                 CONFIGURATION vs INSTANCE CHEAT SHEET                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  CONFIGURATION                    │  INSTANCE                            │
│  ══════════════                   │  ════════                            │
│                                   │                                      │
│  Created by: User                 │  Created by: Agent                   │
│  Lifecycle: Manual                │  Lifecycle: Automatic                │
│  Scope: Cluster-wide intent       │  Scope: Specific device              │
│                                   │                                      │
│  ┌──────────────────────────┐    │  ┌──────────────────────────┐       │
│  │  Configuration            │    │  │  Instance                 │       │
│  │                          │    │  │                          │       │
│  │  "I want to discover     │    │  │  "Camera at 192.168.1.10 │       │
│  │   ONVIF cameras"         │────┼─►│   has been discovered"   │       │
│  │                          │    │  │                          │       │
│  │  • discoveryHandler      │    │  │  • configurationName     │       │
│  │  • capacity: 2           │    │  │  • nodes: [a, b]         │       │
│  │  • brokerSpec            │    │  │  • deviceUsage: {...}    │       │
│  │  • services              │    │  │  • properties: {...}     │       │
│  └──────────────────────────┘    │  └──────────────────────────┘       │
│           │                       │           │                          │
│           │ 1 Configuration       │           │ Multiple Instances       │
│           ▼                       │           ▼                          │
│  ┌──────────────────────────┐    │  ┌───────┬───────┬────────┐         │
│  │  Applies to ALL devices  │    │  │Inst 1 │Inst 2 │Inst 3  │         │
│  │  of this type            │    │  │Camera │Camera │Camera  │         │
│  └──────────────────────────┘    │  │   A   │   B   │   C    │         │
│                                   │  └───────┴───────┴────────┘         │
│                                   │                                      │
│  Think: "Recipe" or "Template"    │  Think: "Actual cake/result"        │
│                                   │                                      │
│  ─────────────────────────────────┼──────────────────────────────────   │
│                                   │                                      │
│  KEY FIELDS:                      │  KEY FIELDS:                         │
│                                   │                                      │
│  discoveryHandler                 │  configurationName ───┐              │
│    ├─ name: "onvif"               │  (points to parent) ──┘              │
│    └─ discoveryDetails: {...}     │                                      │
│                                   │  nodes: [node-a, node-b]             │
│  capacity: 2                      │  (who can see this device)           │
│  (max nodes sharing device)       │                                      │
│                                   │  deviceUsage:                        │
│  brokerSpec:                      │    "0": "node-a"  ← slot used        │
│    └─ PodSpec or JobSpec          │    "1": ""        ← slot free        │
│                                   │                                      │
│  instanceServiceSpec              │  brokerProperties:                   │
│  (per-device service)             │    DEVICE_IP: "192.168.1.10"         │
│                                   │    (merged from Config + DH)         │
│  configurationServiceSpec         │                                      │
│  (all-devices service)            │  cdiName:                            │
│                                   │    "akri.sh/onvif-cameras=abc123"    │
│  brokerProperties:                │  (for device plugin)                 │
│    RESOLUTION: "1920x1080"        │                                      │
│                                   │                                      │
│  ─────────────────────────────────┼──────────────────────────────────   │
│                                   │                                      │
│  RELATIONSHIP:                                                           │
│  ═════════════                                                           │
│                                                                           │
│       1 Configuration                                                    │
│             │                                                            │
│             ├──► N Instances (one per discovered device)                │
│             │                                                            │
│             └──► Each Instance has ownerReference to Configuration      │
│                  (deletion cascades!)                                   │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Configuration CRD

**API Group**: `akri.sh/v0`
**Kind**: `Configuration`
**Scope**: Namespaced

**Code**: [shared/src/akri/configuration.rs](../shared/src/akri/configuration.rs)

### Purpose

The Configuration CRD is the primary way users interact with Akri. It tells Akri:
- **What to discover**: Which discovery handler to use and how to filter devices
- **How to use it**: What workload (broker) to run for discovered devices
- **Where to run it**: Capacity controls device sharing across nodes
- **How to access it**: Service specifications for network access

### Spec Structure

```rust
pub struct ConfigurationSpec {
    pub discovery_handler: DiscoveryHandlerInfo,
    pub capacity: usize,  // default: 1
    pub broker_spec: Option<BrokerSpec>,
    pub instance_service_spec: Option<ServiceSpec>,
    pub configuration_service_spec: Option<ServiceSpec>,
    pub broker_properties: HashMap<String, String>,
}
```

### Fields

#### discoveryHandler (Required, Immutable)

Specifies which Discovery Handler to use and provides discovery parameters.

```yaml
discoveryHandler:
  name: onvif  # Which handler: "onvif", "udev", "opcua", "debug-echo"
  discoveryDetails: |  # Handler-specific JSON/YAML
    ipAddresses:
      action: Include
      items:
      - 192.168.1.0/24
  discoveryProperties:  # Optional: secrets/configmaps
  - name: ONVIF_USERNAME
    valueFrom:
      secretKeyRef:
        name: onvif-credentials
        key: username
```

**Note**: The `discoveryHandler` field is **immutable** after creation. To change it, you must delete and recreate the Configuration.

#### capacity (Optional, Default: 1)

Number of nodes that can simultaneously use each discovered device.

```yaml
capacity: 1   # Exclusive: only 1 node can use the device
capacity: 5   # Shared: up to 5 nodes can use the device
capacity: 100 # Highly shared (e.g., network cameras)
```

**Use Cases**:
- `capacity: 1` - USB devices, local sensors (non-shared)
- `capacity: 2-5` - IP cameras with limited bandwidth
- `capacity: 100` - Highly available network resources

#### brokerSpec (Optional)

Defines the workload that will run on nodes with access to discovered devices.

Two types:
1. **brokerPodSpec**: Long-running Pod (most common)
2. **brokerJobSpec**: One-time Job

```yaml
brokerSpec:
  brokerPodSpec:  # PodSpec from k8s.io/api/core/v1
    containers:
    - name: camera-broker
      image: my-camera-broker:v1
      resources:
        requests:
          akri.sh/onvif-cameras: "1"  # Request device resource
        limits:
          akri.sh/onvif-cameras: "1"
      env:
      - name: CUSTOM_VAR
        value: "custom-value"
    imagePullSecrets:
    - name: my-registry-secret
```

**Important**: Brokers must request the device resource:
```yaml
resources:
  requests:
    akri.sh/<configuration-name>: "1"
  limits:
    akri.sh/<configuration-name>: "1"
```

#### instanceServiceSpec (Optional)

Service created for **each discovered device**.

```yaml
instanceServiceSpec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
```

**Service Name**: `akri-<configuration-name>-<instance-id>-svc`
**Selector**: `akri.sh/instance=<instance-name>`

#### configurationServiceSpec (Optional)

Service created for **all devices** of this Configuration.

```yaml
configurationServiceSpec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 8080
```

**Service Name**: `akri-<configuration-name>-svc`
**Selector**: `akri.sh/configuration=<configuration-name>`

#### brokerProperties (Optional)

Key-value pairs set as environment variables in broker Pods.

```yaml
brokerProperties:
  RESOLUTION: "1920x1080"
  FRAME_RATE: "30"
  LOG_LEVEL: "info"
```

These are merged with device-specific properties from the Discovery Handler.

### Complete Example

```yaml
apiVersion: akri.sh/v0
kind: Configuration
metadata:
  name: onvif-cameras
  namespace: default
spec:
  discoveryHandler:
    name: onvif
    discoveryDetails: |
      ipAddresses:
        action: Exclude
        items:
        - 192.168.1.100
      scopes:
        action: Include
        items:
        - onvif://www.onvif.org/Profile/Streaming
    discoveryProperties:
    - name: ONVIF_USERNAME
      valueFrom:
        secretKeyRef:
          name: onvif-auth
          key: username
          namespace: default
    - name: ONVIF_PASSWORD
      valueFrom:
        secretKeyRef:
          name: onvif-auth
          key: password
          namespace: default

  capacity: 3

  brokerSpec:
    brokerPodSpec:
      containers:
      - name: camera-streamer
        image: ghcr.io/project-akri/akri/onvif-video-broker:latest
        resources:
          requests:
            akri.sh/onvif-cameras: "1"
            memory: "256Mi"
            cpu: "500m"
          limits:
            akri.sh/onvif-cameras: "1"
            memory: "512Mi"
            cpu: "1000m"
      imagePullSecrets:
      - name: ghcr-secret

  instanceServiceSpec:
    type: ClusterIP
    ports:
    - name: grpc
      port: 80
      targetPort: 8083
      protocol: TCP

  configurationServiceSpec:
    type: ClusterIP
    ports:
    - name: grpc
      port: 80
      targetPort: 8083

  brokerProperties:
    RESOLUTION_WIDTH: "1920"
    RESOLUTION_HEIGHT: "1080"
    FRAMES_PER_SECOND: "10"
```

### Discovery Handler Details

Each Discovery Handler has its own schema for `discoveryDetails`. See:
- [ONVIF Discovery Handler](discovery-handlers/onvif.md)
- [udev Discovery Handler](discovery-handlers/udev.md)
- [OPC UA Discovery Handler](discovery-handlers/opcua.md)

## Instance CRD

**API Group**: `akri.sh/v0`
**Kind**: `Instance`
**Scope**: Namespaced
**Short Name**: `akrii`

**Code**: [shared/src/akri/instance.rs](../shared/src/akri/instance.rs)

### Purpose

The Instance CRD represents a specific discovered device. Instances are:
- **Created by Agents** when devices are discovered
- **Owned by Configurations** (owner reference)
- **Watched by Controller** to deploy brokers
- **Updated by Agents** to track device availability and usage

### Spec Structure

```rust
pub struct InstanceSpec {
    pub configuration_name: String,
    pub cdi_name: String,
    pub capacity: usize,
    pub broker_properties: HashMap<String, String>,
    pub shared: bool,  // default: false
    pub nodes: Vec<String>,
    pub device_usage: HashMap<String, String>,
}
```

### Fields

#### configurationName (Required)

Name of the parent Configuration.

```yaml
configurationName: onvif-cameras
```

#### cdiName (Required)

CDI (Container Device Interface) fully qualified name.

Format: `akri.sh/<configuration-name>=<instance-id>`

```yaml
cdiName: akri.sh/onvif-cameras=camera-192-168-1-10-abc123
```

#### capacity (Required)

Number of slots available (copied from Configuration).

```yaml
capacity: 3
```

#### brokerProperties (Optional)

Merged properties from Configuration and Discovery Handler.

```yaml
brokerProperties:
  # From Configuration
  RESOLUTION_WIDTH: "1920"
  RESOLUTION_HEIGHT: "1080"
  # From Discovery Handler
  ONVIF_DEVICE_SERVICE_URL: "http://192.168.1.10/onvif/device_service"
  ONVIF_DEVICE_IP_ADDRESS: "192.168.1.10"
  ONVIF_DEVICE_MAC_ADDRESS: "aa:bb:cc:dd:ee:ff"
  ONVIF_DEVICE_UUID: "uuid:12345678-1234-1234-1234-123456789abc"
```

These become environment variables in broker Pods.

#### shared (Optional, Default: false)

Whether the device can be accessed by multiple nodes.

```yaml
shared: true   # Network device (IP camera, OPC UA server)
shared: false  # Local device (USB camera, local sensor)
```

Set automatically by Discovery Handler during registration.

#### nodes (Optional)

List of nodes that can currently access this device.

```yaml
nodes:
- worker-1
- worker-2
```

**For shared devices**: Multiple nodes
**For non-shared devices**: Single node

#### deviceUsage (Optional)

Map of slot index to node name, tracking which nodes are using which slots.

```yaml
deviceUsage:
  "0": "worker-1"  # Slot 0 used by worker-1
  "1": ""          # Slot 1 available
  "2": "worker-2"  # Slot 2 used by worker-2
```

**Slot States**:
- `""` - Free
- `"<node-name>"` - Used by Instance device plugin
- `"C:<vdev-id>:<node-name>"` - Used by Configuration device plugin

### Complete Example

```yaml
apiVersion: akri.sh/v0
kind: Instance
metadata:
  name: onvif-cameras-192-168-1-10-abc123
  namespace: default
  ownerReferences:
  - apiVersion: akri.sh/v0
    kind: Configuration
    name: onvif-cameras
    uid: 12345678-abcd-1234-abcd-123456789abc
    controller: true
    blockOwnerDeletion: true
spec:
  configurationName: onvif-cameras
  cdiName: akri.sh/onvif-cameras=192-168-1-10-abc123
  capacity: 3
  shared: true
  nodes:
  - worker-1
  - worker-2
  - worker-3
  deviceUsage:
    "0": "worker-1"
    "1": "worker-2"
    "2": ""
  brokerProperties:
    # Configuration properties
    RESOLUTION_WIDTH: "1920"
    RESOLUTION_HEIGHT: "1080"
    FRAMES_PER_SECOND: "10"
    # Device-specific properties (from ONVIF Discovery Handler)
    ONVIF_DEVICE_SERVICE_URL: "http://192.168.1.10/onvif/device_service"
    ONVIF_DEVICE_IP_ADDRESS: "192.168.1.10"
    ONVIF_DEVICE_MAC_ADDRESS: "aa:bb:cc:dd:ee:ff"
    ONVIF_DEVICE_UUID: "uuid:12345678-90ab-cdef-1234-567890abcdef"
status:
  # Status is not currently used
```

### kubectl Output

```bash
$ kubectl get akrii
NAME                                     CONFIG          SHARED   NODES                          AGE
onvif-cameras-192-168-1-10-abc123       onvif-cameras   true     ["worker-1","worker-2"]        5m
onvif-cameras-192-168-1-11-def456       onvif-cameras   true     ["worker-1","worker-3"]        5m
udev-video-dev-video0                   udev-video      false    ["worker-1"]                   10m
```

## Lifecycle

### Configuration → Instances Flow

```
1. User creates Configuration
   │
   ├─> Configuration CRD created in K8s
   │
   └─> Watched by all Agents
       │
       ├─> Agent on worker-1
       │   ├─> Looks up Discovery Handler
       │   ├─> Calls Discover()
       │   ├─> Device A found
       │   └─> Creates Instance A (nodes: [worker-1])
       │
       ├─> Agent on worker-2
       │   ├─> Looks up Discovery Handler
       │   ├─> Calls Discover()
       │   ├─> Device A found (shared)
       │   └─> Updates Instance A (nodes: [worker-1, worker-2])
       │
       └─> Agent on worker-3
           ├─> Looks up Discovery Handler
           ├─> Calls Discover()
           └─> Device B found
               └─> Creates Instance B (nodes: [worker-3])
```

### Instance → Broker Flow

```
1. Instance created/updated
   │
   └─> Watched by Controller
       │
       ├─> Get parent Configuration
       │
       ├─> Extract brokerSpec
       │
       ├─> For each node in Instance.spec.nodes:
       │   │
       │   └─> Create broker Pod
       │       ├─> Set nodeSelector or nodeName
       │       ├─> Add env vars from Instance.brokerProperties
       │       ├─> Set resource requests/limits
       │       └─> Set ownerReference to Instance
       │
       ├─> Create Instance Service (if instanceServiceSpec defined)
       │   ├─> Name: akri-<config>-<instance-id>-svc
       │   └─> Selector: akri.sh/instance=<instance-name>
       │
       └─> Create/update Configuration Service (if configurationServiceSpec defined)
           ├─> Name: akri-<config>-svc
           └─> Selector: akri.sh/configuration=<config-name>
```

### Device Disappearance Flow

```
1. Device physically removed/disconnected
   │
   └─> Discovery Handler detects absence (next scan)
       │
       └─> Agent updates deviceUsage
           │
           ├─> Remove node from Instance.spec.nodes
           │
           ├─> If last node removed:
           │   └─> Delete Instance CRD
           │       └─> Owner reference triggers broker Pod deletion
           │
           └─> If other nodes remain:
               └─> Broker on that node terminated
                   └─> Slot marked unhealthy in device plugin
```

## Labels and Selectors

### Broker Pods

Labels automatically added to broker Pods:

```yaml
labels:
  akri.sh/configuration: onvif-cameras
  akri.sh/instance: onvif-cameras-192-168-1-10-abc123
  akri.sh/target-node: worker-1
```

### Services

**Instance Service**:
```yaml
selector:
  akri.sh/instance: onvif-cameras-192-168-1-10-abc123
```

**Configuration Service**:
```yaml
selector:
  akri.sh/configuration: onvif-cameras
```

## Owner References

Instances have owner references to their parent Configuration:

```yaml
ownerReferences:
- apiVersion: akri.sh/v0
  kind: Configuration
  name: onvif-cameras
  uid: <configuration-uid>
  controller: true
  blockOwnerDeletion: true
```

**Result**: Deleting a Configuration cascades to delete all its Instances (and their broker Pods).

## Device Resources

### Resource Names

Format: `akri.sh/<configuration-name>`

Examples:
- `akri.sh/onvif-cameras`
- `akri.sh/udev-video`
- `akri.sh/opcua-servers`

### Advertised to Kubelet

```yaml
# Node allocatable resources
status:
  allocatable:
    cpu: "4"
    memory: "16Gi"
    akri.sh/onvif-cameras: "3"  # 3 devices/slots available
```

### Requested by Brokers

```yaml
resources:
  requests:
    akri.sh/onvif-cameras: "1"
  limits:
    akri.sh/onvif-cameras: "1"
```

## Validation

### Configuration Validation

- `discoveryHandler.name` is **required**
- `discoveryHandler.discoveryDetails` defaults to `""`
- `capacity` defaults to `1`
- `discoveryHandler` is **immutable** (enforced by CRD validation)

### Instance Validation

- `configurationName` is **required**
- `cdiName` is **required**
- `capacity` is **required**
- `nodes` must be a set (no duplicates)
- `deviceUsage` uses granular merge strategy

## API Operations

### List Configurations

```bash
kubectl get configurations
kubectl get akric  # short name
```

### Get Configuration Details

```bash
kubectl describe configuration onvif-cameras
kubectl get configuration onvif-cameras -o yaml
```

### Create Configuration

```bash
kubectl apply -f my-configuration.yaml
```

### Delete Configuration

```bash
kubectl delete configuration onvif-cameras
# Cascades to delete all Instances and broker Pods
```

### List Instances

```bash
kubectl get instances
kubectl get akrii  # short name
```

### Get Instance Details

```bash
kubectl describe instance onvif-cameras-192-168-1-10-abc123
kubectl get instance onvif-cameras-192-168-1-10-abc123 -o yaml
```

### Watch Instances

```bash
kubectl get instances --watch
```

## Troubleshooting

### No Instances Created

1. Check Agent logs:
   ```bash
   kubectl logs -n akri daemonset/akri-agent
   ```

2. Verify Discovery Handler is registered:
   ```bash
   # Look for "RegisterDiscoveryHandler" in Agent logs
   ```

3. Check Configuration is valid:
   ```bash
   kubectl describe configuration <name>
   ```

### Instances Created but No Brokers

1. Check Controller logs:
   ```bash
   kubectl logs -n akri deployment/akri-controller
   ```

2. Verify brokerSpec is defined:
   ```bash
   kubectl get configuration <name> -o jsonpath='{.spec.brokerSpec}'
   ```

3. Check for Pod creation errors:
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   ```

### Brokers Not Getting Resources

1. Verify resource requests:
   ```bash
   kubectl get pod <broker-pod> -o jsonpath='{.spec.containers[0].resources}'
   ```

2. Check Node allocatable resources:
   ```bash
   kubectl describe node <node-name> | grep akri.sh
   ```

3. Verify device plugin is registered:
   ```bash
   # Agent logs should show "Device plugin registered"
   ```

## Best Practices

### Configuration

1. **Use meaningful names**: `onvif-cameras`, not `config1`
2. **Set appropriate capacity**: Consider bandwidth and device limitations
3. **Use secrets for credentials**: Never hardcode passwords in discoveryDetails
4. **Define services**: Makes brokers accessible
5. **Set resource limits**: Prevent broker resource exhaustion

### Instance

- **Don't create manually**: Let Agents create them
- **Don't delete manually** (unless troubleshooting): Can cause inconsistencies
- **Monitor `deviceUsage`**: Understand slot allocation

### Naming Conventions

- **Configuration**: `<protocol>-<device-type>` (e.g., `onvif-cameras`, `udev-video`)
- **Instance**: Auto-generated by Agent: `<config-name>-<device-id>`

## Examples

See [Configuration Examples](configuration-examples.md) for complete, runnable examples.

## Next Steps

- [ONVIF Configuration Guide](discovery-handlers/onvif.md)
- [udev Configuration Guide](discovery-handlers/udev.md)
- [OPC UA Configuration Guide](discovery-handlers/opcua.md)
- [Broker Development](broker-development.md)
- [Deployment Guide](deployment.md)
