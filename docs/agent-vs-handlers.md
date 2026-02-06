# Agent vs Discovery Handlers: A Critical Distinction

**This is one of the most commonly misunderstood aspects of Akri.** This document clarifies the difference between the Akri Agent and Discovery Handlers.

## TL;DR

| Aspect | Akri Agent | Discovery Handler |
|--------|------------|-------------------|
| **What is it?** | Platform/Framework | Protocol Plugin |
| **Quantity** | One per node | Multiple per node |
| **Deployment** | DaemonSet | DaemonSet or embedded |
| **Purpose** | Manages device plugins and orchestrates discovery | Discovers specific devices using a protocol |
| **Protocol Knowledge** | Protocol-agnostic | Protocol-specific (ONVIF, udev, OPC UA, etc.) |
| **Kubernetes Integration** | Direct (device plugins, CRDs, API server) | Indirect (through Agent) |
| **Code Location** | `agent/` directory | `discovery-handlers/` and `discovery-handler-modules/` |
| **Binary** | `akri-agent` | `akri-onvif`, `akri-udev`, etc. |

## The Key Analogy

Think of the relationship like this:

```
Web Browser (Agent)           vs.    Browser Extensions (Discovery Handlers)
    │                                      │
    ├─ One application                     ├─ Multiple plugins
    ├─ Manages tabs, windows               ├─ Each adds specific functionality
    ├─ Provides plugin framework           ├─ Uses browser APIs
    └─ Can work without plugins            └─ Cannot work without browser

Kubernetes Cluster (Agent)    vs.    Operators (Discovery Handlers)
    │                                      │
    ├─ Core platform                       ├─ Specialized controllers
    ├─ Manages core resources              ├─ Manage specific resources
    ├─ Provides extension points           ├─ Use Kubernetes APIs
    └─ Platform for applications           └─ Run on the platform
```

**In Akri's case:**
- **Agent** = The platform that provides device plugin management
- **Discovery Handlers** = Plugins that implement specific discovery protocols

## Detailed Comparison

### Akri Agent

**Location**: [`agent/`](../agent/)

**What it does:**
1. **Runs the Device Plugin Framework**
   - Implements Kubernetes Device Plugin API
   - Creates Unix domain sockets for kubelet communication
   - Advertises discovered devices to kubelet as allocatable resources

2. **Manages Discovery Handler Registry**
   - Provides a registration service for Discovery Handlers to connect
   - Maintains a registry of available Discovery Handlers
   - Routes discovery requests to appropriate handlers

3. **Watches Configurations**
   - Monitors Akri Configuration CRDs
   - Determines which Discovery Handlers to invoke
   - Triggers discovery based on Configuration changes

4. **Creates and Manages Instances**
   - Creates Instance CRDs for discovered devices
   - Updates Instance CRDs with device availability
   - Manages device usage tracking (which nodes are using which devices)

5. **Manages Device Plugins**
   - Creates device plugins for each Configuration
   - Implements device allocation and deallocation
   - Handles slot reclamation when devices disappear

**Key Characteristics:**
- **Single binary**: `akri-agent`
- **One per node**: Deployed as a DaemonSet
- **Protocol-agnostic**: Doesn't know about ONVIF, udev, or OPC UA
- **Framework provider**: Provides infrastructure for Discovery Handlers
- **Kubernetes-native**: Direct integration with kubelet and API server

**Agent Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                        Akri Agent                                │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Configuration Controller                       │ │
│  │  - Watches Configuration CRDs                              │ │
│  │  - Triggers discovery when Configurations change           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         Discovery Handler Registry & Manager               │ │
│  │  - Registration service (gRPC server)                      │ │
│  │  - Maintains list of available Discovery Handlers          │ │
│  │  - Routes discovery requests to handlers                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Device Manager                             │ │
│  │  - Tracks discovered devices in memory                     │ │
│  │  - Notifies Device Plugin Manager of changes               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            Device Plugin Manager                            │ │
│  │  - Creates device plugins for each Configuration           │ │
│  │  - Implements Kubernetes Device Plugin API                 │ │
│  │  - Manages Unix domain sockets                             │ │
│  │  - Creates/updates Instance CRDs                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Device Plugin Slot Reclaimer                   │ │
│  │  - Monitors for disappeared devices                        │ │
│  │  - Reclaims slots when devices become unavailable          │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Code Structure:**
```
agent/
├── src/
│   ├── main.rs                                    # Entry point
│   ├── device_manager/                            # Tracks devices
│   ├── discovery_handler_manager/                 # DH registry & registration
│   ├── plugin_manager/                            # Device plugin implementation
│   │   ├── device_plugin_instance_controller.rs   # Creates plugins
│   │   ├── device_plugin_slot_reclaimer.rs        # Reclaims slots
│   │   ├── v1.rs                                  # Device Plugin API v1
│   │   └── v1beta1.rs                             # Device Plugin API v1beta1
│   └── util/
│       └── discovery_configuration_controller.rs  # Watches Configurations
```

### Discovery Handlers

**Locations**:
- [`discovery-handlers/`](../discovery-handlers/) - Embedded implementations
- [`discovery-handler-modules/`](../discovery-handler-modules/) - Standalone implementations

**What they do:**
1. **Implement the Discovery Protocol**
   - ONVIF: Discovers IP cameras using WS-Discovery
   - udev: Queries Linux kernel for USB/local devices
   - OPC UA: Discovers industrial automation servers
   - Debug Echo: Returns mock devices for testing

2. **Register with Agent**
   - Connect to Agent's registration service
   - Provide Discovery Handler metadata (name, endpoint, shared capability)

3. **Respond to Discovery Requests**
   - Receive discovery requests from Agent (via gRPC)
   - Parse discovery details from Configuration
   - Perform protocol-specific discovery
   - Return discovered devices with properties

4. **Continuously Monitor Devices**
   - Periodically re-scan for devices
   - Detect device appearance/disappearance
   - Stream updated device lists back to Agent

**Key Characteristics:**
- **Multiple binaries**: `akri-onvif`, `akri-udev`, `akri-opcua`, `akri-debug-echo`
- **Multiple per node**: Each protocol has its own handler
- **Protocol-specific**: Deep knowledge of one protocol
- **Plugin architecture**: Uses Agent's framework
- **gRPC communication**: Talks to Agent via gRPC

**Discovery Handler Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Discovery Handler (e.g., ONVIF)               │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Registration Logic                         │ │
│  │  - Connects to Agent's registration service                │ │
│  │  - Registers as "onvif" handler                            │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │             gRPC Discovery Service                          │ │
│  │  - Implements DiscoveryHandler gRPC service                │ │
│  │  - Receives Discover() requests from Agent                 │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           Protocol-Specific Discovery Logic                 │ │
│  │  ONVIF:  - WS-Discovery multicast                          │ │
│  │          - ONVIF device probing                            │ │
│  │          - Credential management                           │ │
│  │  udev:   - Enumerate devices via udev library              │ │
│  │          - Parse udev rules                                │ │
│  │          - Build device tree with relatives                │ │
│  │  OPC UA: - OPC UA server discovery                         │ │
│  │          - Certificate management                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Discovery Loop                                 │ │
│  │  - Periodically scans for devices (every 10 seconds)       │ │
│  │  - Applies filters from Configuration                      │ │
│  │  - Detects changes in device availability                  │ │
│  │  - Streams results back to Agent                           │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Code Structure (ONVIF example):**
```
discovery-handlers/onvif/
├── src/
│   ├── lib.rs                      # Library entry point
│   ├── discovery_handler.rs        # Implements DiscoveryHandler trait
│   ├── discovery_impl.rs           # ONVIF-specific discovery logic
│   ├── discovery_utils.rs          # ONVIF utilities
│   ├── credential_store.rs         # Credential management
│   └── username_token.rs           # WS-Security username token

discovery-handler-modules/onvif-discovery-handler/
├── src/
│   └── main.rs                     # Standalone binary entry point
```

## Communication Flow

Here's how Agent and Discovery Handlers work together:

```
┌──────────────┐                                  ┌──────────────────┐
│ Kubernetes   │                                  │   Discovery      │
│ API Server   │                                  │   Handler        │
└──────┬───────┘                                  │   (ONVIF)        │
       │                                          └────────┬─────────┘
       │ 1. User creates                                   │
       │    Configuration CRD                              │
       │    (name: "onvif")                                │
       ▼                                                   │
┌─────────────────────────────────────────────┐           │
│            Akri Agent                        │           │
│                                              │           │
│  ┌────────────────────────────────────────┐ │           │
│  │  Configuration Controller              │ │           │
│  │  2. Detects new Configuration          │ │           │
│  └───────────┬────────────────────────────┘ │           │
│              │                                │           │
│              ▼                                │           │
│  ┌────────────────────────────────────────┐ │           │
│  │  Discovery Handler Registry            │ │           │
│  │  3. Looks up "onvif" handler           │ │           │
│  │     endpoint                           │ │◄──────────┤
│  └───────────┬────────────────────────────┘ │   0. Registers
│              │                                │      at startup
│              ▼                                │
│  ┌────────────────────────────────────────┐ │           │
│  │  4. Sends gRPC Discover() request      │ ├──────────►│
│  │     with discovery_details             │ │   Request │
│  └───────────┬────────────────────────────┘ │           │
│              │                                │           ▼
│              │                                │  ┌─────────────────┐
│              │                                │  │ 5. Parses       │
│              │                                │  │    discovery    │
│              │                                │  │    details      │
│              │                                │  └────────┬────────┘
│              │                                │           │
│              │                                │           ▼
│              │                                │  ┌─────────────────┐
│              │                                │  │ 6. Performs     │
│              │                                │  │    ONVIF        │
│              │                                │  │    discovery    │
│              │                                │  └────────┬────────┘
│              │                                │           │
│              │◄───────────────────────────────┼───────────┘
│              │       7. Streams Device list   │   Response
│              │          continuously          │
│              ▼                                │
│  ┌────────────────────────────────────────┐ │
│  │  Device Manager                        │ │
│  │  8. Updates in-memory device list      │ │
│  └───────────┬────────────────────────────┘ │
│              │                                │
│              ▼                                │
│  ┌────────────────────────────────────────┐ │
│  │  Device Plugin Manager                 │ │
│  │  9. Creates/updates Instance CRDs      │ │
│  └───────────┬────────────────────────────┘ │
└──────────────┼────────────────────────────────┘
               │
               ▼
       ┌──────────────┐
       │  Instance    │
       │  CRD         │
       │  (Camera-1)  │
       └──────────────┘
```

## Common Misconceptions

### ❌ Misconception 1: "The Agent discovers devices"
**Reality**: The Agent does NOT discover devices. Discovery Handlers discover devices. The Agent orchestrates the discovery process and manages the results.

### ❌ Misconception 2: "Discovery Handlers create Instances"
**Reality**: Discovery Handlers do NOT create Instances. They return device information to the Agent, which then creates Instance CRDs.

### ❌ Misconception 3: "Discovery Handlers talk directly to kubelet"
**Reality**: Discovery Handlers do NOT talk to kubelet or Kubernetes APIs. The Agent handles all Kubernetes integration.

### ❌ Misconception 4: "I need to modify the Agent to add a new protocol"
**Reality**: You do NOT modify the Agent. You create a new Discovery Handler that implements the DiscoveryHandler gRPC service.

### ❌ Misconception 5: "Discovery Handlers and Agent are the same thing"
**Reality**: They are separate components with different responsibilities, running in different processes.

## When to Modify Each Component

### Modify the Agent when:
- Changing how device plugins are managed
- Modifying Instance creation logic
- Changing device slot allocation/reclamation
- Adding new device plugin framework features
- Modifying Configuration watching logic

### Create/Modify a Discovery Handler when:
- Adding support for a new protocol (e.g., Bluetooth, Zigbee)
- Changing how a specific protocol discovers devices
- Adding protocol-specific filtering
- Modifying device properties for a protocol
- Fixing bugs in protocol-specific code

## Example: Adding Bluetooth Support

**Wrong approach** ❌:
```
1. Modify agent/src/main.rs to add Bluetooth discovery
2. Add Bluetooth libraries to Agent dependencies
```

**Correct approach** ✅:
```
1. Create discovery-handlers/bluetooth/
2. Implement DiscoveryHandler gRPC service
3. Add Bluetooth-specific discovery logic
4. Register with Agent at startup
5. Agent automatically uses it when Configuration has discoveryHandler.name = "bluetooth"
```

## API Contract: The Proto File

Both Agent and Discovery Handlers communicate using the protocol defined in [`discovery-utils/proto/discovery.proto`](../discovery-utils/proto/discovery.proto):

```protobuf
service Registration {
    // Discovery Handler calls this to register with Agent
    rpc RegisterDiscoveryHandler(RegisterDiscoveryHandlerRequest) returns (Empty);
}

service DiscoveryHandler {
    // Agent calls this to request device discovery
    rpc Discover(DiscoverRequest) returns (stream DiscoverResponse);
}

message DiscoverRequest {
    string discovery_details = 1;           // From Configuration.spec.discoveryHandler.discoveryDetails
    map<string, ByteData> discovery_properties = 2;
}

message DiscoverResponse {
    repeated Device devices = 1;            // List of discovered devices
}

message Device {
    string id = 1;                          // Unique device ID
    map<string, string> properties = 2;     // Device properties (env vars for brokers)
    repeated Mount mounts = 3;              // Volume mounts for broker Pods
    repeated DeviceSpec device_specs = 4;   // Device files to expose
}
```

## Deployment Models

### Embedded Discovery Handlers (Simpler)
```yaml
# Agent includes Discovery Handlers in the same container
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-agent
spec:
  template:
    spec:
      containers:
      - name: akri-agent
        image: akri-agent:latest
        # Discovery Handlers run as part of the Agent binary
        # or as separate processes in the same Pod
```

### Standalone Discovery Handlers (More Flexible)
```yaml
# Discovery Handlers run in separate DaemonSets
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-onvif-discovery-handler
spec:
  template:
    spec:
      containers:
      - name: onvif-dh
        image: akri-onvif:latest
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-udev-discovery-handler
spec:
  template:
    spec:
      containers:
      - name: udev-dh
        image: akri-udev:latest
```

## Summary

**Think of it this way:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Akri System                               │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              Akri Agent (The Platform)               │   │
│  │  • One per node                                      │   │
│  │  • Manages everything                                │   │
│  │  • Protocol-agnostic                                 │   │
│  │  • Talks to Kubernetes                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                          │                                    │
│                          │ Uses                               │
│                          ▼                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         Discovery Handlers (Protocol Plugins)        │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌────────┐ │   │
│  │  │  ONVIF  │  │  udev   │  │ OPC UA  │  │ Custom │ │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └────────┘ │   │
│  │  • Multiple per node                                │   │
│  │  • Protocol-specific                                │   │
│  │  • Just discover devices                            │   │
│  │  • Return results to Agent                          │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Agent = Operating System** (provides infrastructure)
**Discovery Handlers = Applications** (use the infrastructure)

The Agent provides the **framework**, Discovery Handlers provide the **functionality**.

## Decision Matrix: When Working With Each Component

Use this matrix to determine which component you need to work with:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DECISION MATRIX                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  I want to...                    Agent    Discovery Handler              │
│  ═══════════════════════════════════════════════════════════════════    │
│                                                                           │
│  Add support for Bluetooth       ✗         ✓ Create bluetooth-dh        │
│  Add support for Zigbee          ✗         ✓ Create zigbee-dh           │
│  Add support for Modbus          ✗         ✓ Create modbus-dh           │
│                                                                           │
│  Change how ONVIF discovers      ✗         ✓ Modify onvif-dh            │
│  Fix udev device filtering       ✗         ✓ Modify udev-dh             │
│  Add OPC UA authentication       ✗         ✓ Modify opcua-dh            │
│                                                                           │
│  Modify Instance CRD creation    ✓ Agent   ✗                            │
│  Change device plugin behavior   ✓ Agent   ✗                            │
│  Modify slot allocation logic    ✓ Agent   ✗                            │
│  Change registration protocol    ✓ Agent   ✗ (but DHs must adapt)       │
│                                                                           │
│  Debug why devices not found     Both      ✓ Check DH logs first         │
│  Debug why Instances not created ✓ Agent   ✗                            │
│  Debug device properties         ✗         ✓ Check DH code              │
│                                                                           │
│  Return device properties        ✗         ✓ In Device.properties       │
│  Set device mounts               ✗         ✓ In Device.mounts           │
│  Set device specs                ✗         ✓ In Device.device_specs     │
│                                                                           │
│  Change discovery interval       ✗         ✓ In DH implementation       │
│  Change gRPC protocol            ✓ Both    ✓ Breaking change!           │
│  Add new filter type             ✗         ✓ In DH only                 │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Responsibilities Matrix

A detailed breakdown of who does what:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        RESPONSIBILITIES MATRIX                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Responsibility                           Agent    Discovery Handler      │
│  ═════════════════════════════════════════════════════════════════════   │
│                                                                            │
│  KUBERNETES INTEGRATION                                                   │
│  ─────────────────────────                                                │
│  Watch Configuration CRDs                 ✓        ✗                     │
│  Create Instance CRDs                     ✓        ✗                     │
│  Update Instance CRDs                     ✓        ✗                     │
│  Delete Instance CRDs                     ✓        ✗                     │
│  Talk to Kubernetes API Server            ✓        ✗                     │
│  Implement Device Plugin API              ✓        ✗                     │
│  Register with Kubelet                    ✓        ✗                     │
│  Create Unix domain sockets               ✓        ✗                     │
│  Respond to Allocate() calls              ✓        ✗                     │
│                                                                            │
│  DISCOVERY MANAGEMENT                                                     │
│  ───────────────────                                                      │
│  Provide registration service             ✓        ✗                     │
│  Maintain DH registry                     ✓        ✗                     │
│  Route discovery requests                 ✓        ✗                     │
│  Register with Agent                      ✗        ✓                     │
│  Implement discovery protocol             ✗        ✓                     │
│  Scan for devices                         ✗        ✓                     │
│  Apply filters                            ✗        ✓                     │
│  Return device list                       ✗        ✓                     │
│  Set device properties                    ✗        ✓                     │
│  Set device mounts                        ✗        ✓                     │
│  Set device specs                         ✗        ✓                     │
│                                                                            │
│  DEVICE MANAGEMENT                                                        │
│  ────────────────                                                         │
│  Track devices in memory                  ✓        ✗                     │
│  Notify on device changes                 ✓        ✗                     │
│  Create device plugins                    ✓        ✗                     │
│  Manage device slots                      ✓        ✗                     │
│  Reclaim slots                            ✓        ✗                     │
│  Update deviceUsage map                   ✓        ✗                     │
│                                                                            │
│  PROTOCOL-SPECIFIC                                                        │
│  ────────────────                                                         │
│  Know about ONVIF                         ✗        ✓ (onvif-dh)          │
│  Know about udev                          ✗        ✓ (udev-dh)           │
│  Know about OPC UA                        ✗        ✓ (opcua-dh)          │
│  Parse discovery details                  ✗        ✓                     │
│  Handle credentials                       ✗        ✓                     │
│  Protocol-specific libraries              ✗        ✓                     │
│                                                                            │
│  LIFECYCLE                                                                │
│  ────────                                                                 │
│  Start when node starts                   ✓        ✓                     │
│  Run continuously                         ✓        ✓                     │
│  Handle re-registration                   ✓        ✓                     │
│  Survive handler crashes                  ✓        N/A                   │
│  Survive agent crash                      N/A      ✓ (must re-register)  │
│                                                                            │
└──────────────────────────────────────────────────────────────────────────┘
```

## Code Change Impact Analysis

Understanding what changes when you modify each component:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CODE CHANGE IMPACT ANALYSIS                        │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  If you change the AGENT:                                            │
│  ══════════════════════════                                           │
│                                                                        │
│    ┌─────────────────────────────────────────────────────────┐      │
│    │  Changed Component: Agent                                │      │
│    └────────────────────┬────────────────────────────────────┘      │
│                         │                                             │
│         ┌───────────────┼───────────────┐                            │
│         ▼               ▼               ▼                            │
│    ┌─────────┐    ┌─────────┐    ┌──────────┐                       │
│    │All DHs  │    │Instance │    │Kubelet   │                       │
│    │may need │    │CRD may  │    │interaction                       │
│    │updates  │    │change   │    │changes   │                       │
│    └─────────┘    └─────────┘    └──────────┘                       │
│         │              │               │                              │
│         └──────────────┴───────────────┘                             │
│                        │                                              │
│                        ▼                                              │
│           ┌──────────────────────────┐                               │
│           │  HIGH IMPACT CHANGE      │                               │
│           │  Affects entire system   │                               │
│           └──────────────────────────┘                               │
│                                                                        │
│  ─────────────────────────────────────────────────────────────────   │
│                                                                        │
│  If you change a DISCOVERY HANDLER:                                  │
│  ════════════════════════════════════                                 │
│                                                                        │
│    ┌─────────────────────────────────────────────────────────┐      │
│    │  Changed Component: ONVIF Discovery Handler             │      │
│    └────────────────────┬────────────────────────────────────┘      │
│                         │                                             │
│                         ▼                                             │
│                   ┌──────────┐                                        │
│                   │Only ONVIF│                                        │
│                   │discovery │                                        │
│                   │affected  │                                        │
│                   └──────────┘                                        │
│                         │                                             │
│                         ▼                                             │
│           ┌──────────────────────────┐                               │
│           │  LOW IMPACT CHANGE       │                               │
│           │  Only affects one protocol                               │
│           └──────────────────────────┘                               │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

**Agent = Operating System** (provides infrastructure)
**Discovery Handlers = Applications** (use the infrastructure)

The Agent provides the **framework**, Discovery Handlers provide the **functionality**.
