# Discovery Handlers

Discovery Handlers are protocol-specific plugins that discover devices and report them to the Akri Agent.

## Overview

**Locations**:
- [`discovery-handlers/`](../discovery-handlers/) - Library implementations
- [`discovery-handler-modules/`](../discovery-handler-modules/) - Standalone binaries

**Purpose**: Implement device discovery for specific protocols (ONVIF, udev, OPC UA, etc.)

## Architecture

Each Discovery Handler:
1. Registers with Agent at startup
2. Implements gRPC `DiscoveryHandler` service
3. Continuously scans for devices
4. Streams discovered devices to Agent
5. Applies filters from Configuration

See [Agent vs Discovery Handlers](agent-vs-handlers.md) for the critical distinction.

## Available Discovery Handlers

| Handler | Protocol | Use Case | Shared |
|---------|----------|----------|--------|
| **ONVIF** | ONVIF (WS-Discovery) | IP cameras | Yes |
| **udev** | Linux udev | USB devices, local hardware | No |
| **OPC UA** | OPC UA | Industrial automation | Yes |
| **Debug Echo** | Mock | Testing/development | Yes |

## Common Structure

All Discovery Handlers follow this pattern:

```
┌───────────────────────────────────────────────────────┐
│            Discovery Handler                           │
│                                                         │
│  1. Main Entry Point                                   │
│     ├─> Register with Agent                           │
│     └─> Start gRPC server                             │
│                                                         │
│  2. gRPC DiscoveryHandler Service                      │
│     └─> Implement Discover(DiscoverRequest) RPC       │
│                                                         │
│  3. Discovery Loop                                     │
│     ├─> Parse discovery_details                       │
│     ├─> Perform protocol-specific discovery           │
│     ├─> Apply filters                                 │
│     ├─> Compare with previous results                 │
│     └─> Stream changes to Agent                       │
│                                                         │
│  4. Device Builder                                     │
│     └─> Create Device proto messages                  │
│                                                         │
└───────────────────────────────────────────────────────┘
```

## gRPC API

Discovery Handlers implement the `DiscoveryHandler` service defined in [`discovery-utils/proto/discovery.proto`](../discovery-utils/proto/discovery.proto):

```protobuf
service DiscoveryHandler {
  rpc Discover(DiscoverRequest) returns (stream DiscoverResponse);
}

message DiscoverRequest {
  string discovery_details = 1;
  map<string, ByteData> discovery_properties = 2;
}

message DiscoverResponse {
  repeated Device devices = 1;
}

message Device {
  string id = 1;
  map<string, string> properties = 2;
  repeated Mount mounts = 3;
  repeated DeviceSpec device_specs = 4;
}
```

## Registration Protocol

On startup, each Discovery Handler registers with the Agent:

```protobuf
service Registration {
  rpc RegisterDiscoveryHandler(RegisterDiscoveryHandlerRequest) returns (Empty);
}

message RegisterDiscoveryHandlerRequest {
  string name = 1;              // e.g., "onvif", "udev"
  string endpoint = 2;           // gRPC endpoint
  EndpointType endpoint_type = 3;  // UDS or NETWORK
  bool shared = 4;               // Can devices be shared across nodes?
}
```

## Discovery Handlers in Detail

### ONVIF Discovery Handler

**Purpose**: Discover IP cameras using ONVIF protocol (WS-Discovery)

**Files**:
- Library: [`discovery-handlers/onvif/`](../discovery-handlers/onvif/)
- Binary: [`discovery-handler-modules/onvif-discovery-handler/`](../discovery-handler-modules/onvif-discovery-handler/)

**Discovery Details Schema**:
```yaml
discoveryDetails: |
  ipAddresses:
    action: Include | Exclude
    items: [192.168.1.0/24, ...]
  macAddresses:
    action: Include | Exclude
    items: [aa:bb:cc:dd:ee:ff, ...]
  scopes:
    action: Include | Exclude
    items: [onvif://www.onvif.org/Profile/Streaming, ...]
  uuids:
    action: Include | Exclude
    items: [uuid:12345678-..., ...]
  discoveryTimeoutSeconds: 1
```

**Properties Returned**:
- `ONVIF_DEVICE_IP_ADDRESS`: Device IP
- `ONVIF_DEVICE_MAC_ADDRESS`: Device MAC
- `ONVIF_DEVICE_SERVICE_URL`: ONVIF service endpoint
- `ONVIF_DEVICE_UUID`: Device UUID

**See**: [ONVIF Handler Documentation](discovery-handlers/onvif.md)

### udev Discovery Handler

**Purpose**: Discover USB and local hardware devices using Linux udev

**Files**:
- Library: [`discovery-handlers/udev/`](../discovery-handlers/udev/)
- Binary: [`discovery-handler-modules/udev-discovery-handler/`](../discovery-handler-modules/udev-discovery-handler/)

**Discovery Details Schema**:
```yaml
discoveryDetails: |
  udevRules:
  - 'KERNEL=="video[0-9]*"'
  - 'SUBSYSTEM=="sound"'
  groupRecursive: true | false
  permissions: "rwm"  # cgroups permissions
```

**Properties Returned**:
- `UDEV_DEVNODE`: Device node path (e.g., `/dev/video0`)
- `UDEV_DEVPATH`: sysfs device path
- Additional udev properties

**Device Specs Returned**:
```yaml
device_specs:
- container_path: /dev/video0
  host_path: /dev/video0
  permissions: rwm
```

**See**: [udev Handler Documentation](discovery-handlers/udev.md)

### OPC UA Discovery Handler

**Purpose**: Discover OPC UA servers for industrial automation

**Files**:
- Library: [`discovery-handlers/opcua/`](../discovery-handlers/opcua/)
- Binary: [`discovery-handler-modules/opcua-discovery-handler/`](../discovery-handler-modules/opcua-discovery-handler/)

**Discovery Details Schema**:
```yaml
discoveryDetails: |
  discoveryUrls:
  - opc.tcp://opcua-server:4840
  applicationNames:
    action: Include | Exclude
    items: [MyOPCServer, ...]
```

**Properties Returned**:
- `OPCUA_DISCOVERY_URL`: Server discovery URL
- `OPCUA_APPLICATION_URI`: Application URI
- `OPCUA_APPLICATION_NAME`: Server name

**See**: [OPC UA Handler Documentation](discovery-handlers/opcua.md)

### Debug Echo Discovery Handler

**Purpose**: Testing and development (returns mock devices)

**Files**:
- Library: [`discovery-handlers/debug-echo/`](../discovery-handlers/debug-echo/)
- Binary: [`discovery-handler-modules/debug-echo-discovery-handler/`](../discovery-handler-modules/debug-echo-discovery-handler/)

**Discovery Details Schema**:
```yaml
discoveryDetails: |
  descriptions:
  - "mock-device-1"
  - "mock-device-2"
  shared: true
```

**Properties Returned**:
- `DEBUG_ECHO_DESCRIPTION`: Device description
- `DEBUG_ECHO_EXTRA_INFO`: Additional info

**See**: [Debug Echo Handler Documentation](discovery-handlers/debug-echo.md)

## Discovery Loop

All Discovery Handlers run a continuous discovery loop:

```rust
loop {
    // Check if Agent disconnected
    if discovered_devices_sender.is_closed() {
        // Re-register with Agent
        break;
    }

    // Perform discovery
    let latest_devices = discover_devices(&discovery_details).await;

    // Apply filters
    let filtered_devices = apply_filters(latest_devices, &filters);

    // Compare with previous discovery
    if devices_changed(&filtered_devices, &previous_devices) {
        // Send update to Agent
        discovered_devices_sender.send(DiscoverResponse {
            devices: filtered_devices.clone()
        }).await;
    }

    previous_devices = filtered_devices;

    // Wait before next discovery
    sleep(Duration::from_secs(DISCOVERY_INTERVAL_SECS)).await;
}
```

**Default Interval**: 10 seconds (configurable)

## Credential Management

Discovery Handlers can receive credentials via `discovery_properties`:

```yaml
spec:
  discoveryHandler:
    name: onvif
    discoveryProperties:
    - name: ONVIF_USERNAME
      valueFrom:
        secretKeyRef:
          name: onvif-credentials
          key: username
    - name: ONVIF_PASSWORD
      valueFrom:
        secretKeyRef:
          name: onvif-credentials
          key: password
```

Discovery Handlers access these in the `DiscoverRequest.discovery_properties` map.

## Filtering

Discovery Handlers support two types of filtering:

### Include Filtering
Only devices matching the filter are returned:
```yaml
ipAddresses:
  action: Include
  items:
  - 192.168.1.0/24
```

### Exclude Filtering
Devices matching the filter are excluded:
```yaml
ipAddresses:
  action: Exclude
  items:
  - 192.168.1.100
```

## Device Properties

Properties set by Discovery Handlers become environment variables in broker Pods:

```
Discovery Handler returns:
  properties: {
    "ONVIF_DEVICE_IP_ADDRESS": "192.168.1.10",
    "ONVIF_DEVICE_SERVICE_URL": "http://192.168.1.10/onvif/device_service"
  }

Broker Pod receives:
  env:
  - name: ONVIF_DEVICE_IP_ADDRESS
    value: "192.168.1.10"
  - name: ONVIF_DEVICE_SERVICE_URL
    value: "http://192.168.1.10/onvif/device_service"
```

## Device Mounts

Discovery Handlers can specify volume mounts for broker Pods:

```protobuf
mounts {
  container_path: "/usr/lib/libopencv.so"
  host_path: "/usr/lib/x86_64-linux-gnu/libopencv.so"
  read_only: true
}
```

## Device Specs

Discovery Handlers can specify device files to expose:

```protobuf
device_specs {
  container_path: "/dev/video0"
  host_path: "/dev/video0"
  permissions: "rwm"  // read, write, mknod
}
```

Used by udev handler to expose device nodes to broker Pods.

## Deployment Models

### Embedded (Default)

Discovery Handlers run as part of the Agent process or in the same Pod:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-agent
spec:
  template:
    spec:
      containers:
      - name: agent
        image: akri-agent:latest
        # Discovery Handlers embedded
```

### Standalone

Discovery Handlers run in separate DaemonSets:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-onvif-dh
spec:
  template:
    spec:
      containers:
      - name: onvif-dh
        image: akri-onvif-dh:latest
```

**Advantage**: Easier to add/remove handlers without restarting Agent

## Creating Custom Discovery Handlers

See [Creating Discovery Handlers](creating-discovery-handlers.md) for a complete guide.

**Quick Overview**:

1. Implement `DiscoveryHandler` trait:
   ```rust
   #[async_trait]
   impl DiscoveryHandler for MyDiscoveryHandlerImpl {
       type DiscoverStream = DiscoverStream;
       async fn discover(
           &self,
           request: tonic::Request<DiscoverRequest>,
       ) -> Result<Response<Self::DiscoverStream>, Status> {
           // Implementation
       }
   }
   ```

2. Register with Agent:
   ```rust
   register_discovery_handler(
       "my-protocol",
       endpoint,
       EndpointType::UDS,
       shared,
   ).await;
   ```

3. Implement discovery logic specific to your protocol

4. Return `Device` messages with properties and device specs

## Troubleshooting

### Discovery Handler Not Registered

1. Check Agent logs for registration:
   ```bash
   kubectl logs -n akri daemonset/akri-agent | grep RegisterDiscoveryHandler
   ```

2. Verify Discovery Handler is running:
   ```bash
   kubectl get pods -n akri -l app=<handler-name>
   ```

3. Check network connectivity (if standalone):
   ```bash
   kubectl logs -n akri <handler-pod>
   ```

### No Devices Discovered

1. Check Discovery Handler logs:
   ```bash
   kubectl logs -n akri <handler-pod>
   ```

2. Verify discovery details are valid:
   ```bash
   kubectl get configuration <name> -o jsonpath='{.spec.discoveryHandler.discoveryDetails}'
   ```

3. Test discovery manually (if possible):
   ```bash
   # For ONVIF
   kubectl exec -n akri <handler-pod> -- wsdd
   ```

### Devices Discovered but Wrong Properties

1. Check Discovery Handler implementation
2. Verify properties are set correctly in Device message
3. Check broker Pod environment variables:
   ```bash
   kubectl exec <broker-pod> -- env | grep <PROPERTY>
   ```

## Performance Considerations

- **Discovery Interval**: Balance between responsiveness and system load
- **Network Discovery**: ONVIF/OPC UA may cause network traffic
- **Filtering**: Apply filters early to reduce processing
- **Caching**: Compare with previous results to avoid unnecessary updates

## Code Structure

Each Discovery Handler typically has:

```
discovery-handlers/<protocol>/
├── src/
│   ├── lib.rs                   # Library entry
│   ├── discovery_handler.rs     # Implements DiscoveryHandler trait
│   ├── discovery_impl.rs        # Protocol-specific logic
│   └── discovery_utils.rs       # Utilities

discovery-handler-modules/<protocol>-discovery-handler/
├── src/
│   └── main.rs                  # Binary entry, registration
```

## See Also

- [Agent vs Discovery Handlers](agent-vs-handlers.md) - Critical distinction
- [ONVIF Handler](discovery-handlers/onvif.md)
- [udev Handler](discovery-handlers/udev.md)
- [OPC UA Handler](discovery-handlers/opcua.md)
- [Creating Custom Handlers](creating-discovery-handlers.md)
- [Architecture Overview](architecture.md)
