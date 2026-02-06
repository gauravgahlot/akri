# Akri Architecture

This document provides a comprehensive overview of Akri's architecture, components, and how they work together.

## Table of Contents
- [High-Level Architecture](#high-level-architecture)
- [Core Components](#core-components)
- [Custom Resources](#custom-resources)
- [Data Flow](#data-flow)
- [Component Interactions](#component-interactions)
- [Deployment Architecture](#deployment-architecture)

## High-Level Architecture

Akri extends Kubernetes to support edge devices by introducing a device discovery and management layer. It consists of five key components:

```
┌───────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                             │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐│
│  │                       Akri Controller                              ││
│  │  (Cluster-wide, watches Instances, deploys Brokers)               ││
│  └───────────────────────────────────────────────────────────────────┘│
│                                  │                                      │
│                                  ▼                                      │
│  ┌───────────────────────────────────────────────────────────────────┐│
│  │                    Configuration CRD                               ││
│  │  (User-defined: what to discover, how to use it)                  ││
│  └───────────────────────────────────────────────────────────────────┘│
│            │                            │                               │
│            ▼                            ▼                               │
│  ┌──────────────────┐        ┌──────────────────┐                     │
│  │  Worker Node 1   │        │  Worker Node 2   │                     │
│  │                  │        │                  │                     │
│  │  ┌────────────────────┐  │  ┌────────────────────┐                │
│  │  │  Akri Agent        │  │  │  Akri Agent        │                │
│  │  │  (DaemonSet)       │  │  │  (DaemonSet)       │                │
│  │  │                    │  │  │                    │                │
│  │  │  • Device Mgr      │  │  │  • Device Mgr      │                │
│  │  │  • DH Registry     │  │  │  • DH Registry     │                │
│  │  │  • Device Plugins  │  │  │  • Device Plugins  │                │
│  │  └──────┬─────────────┘  │  └──────┬─────────────┘                │
│  │         │                 │         │                               │
│  │         ▼                 │         ▼                               │
│  │  ┌────────────────────┐  │  ┌────────────────────┐                │
│  │  │ Discovery Handlers │  │  │ Discovery Handlers │                │
│  │  │ • ONVIF            │  │  │ • ONVIF            │                │
│  │  │ • udev             │  │  │ • udev             │                │
│  │  │ • OPC UA           │  │  │ • OPC UA           │                │
│  │  └──────┬─────────────┘  │  └──────┬─────────────┘                │
│  │         │                 │         │                               │
│  │         ▼                 │         ▼                               │
│  │    Discovers devices      │    Discovers devices                   │
│  │         │                 │         │                               │
│  │         ▼                 │         ▼                               │
│  │  ┌────────────────────┐  │  ┌────────────────────┐                │
│  │  │ Creates Instances  │  │  │ Creates Instances  │                │
│  │  └────────────────────┘  │  └────────────────────┘                │
│  │         │                 │         │                               │
│  │         ▼                 │         ▼                               │
│  │  ┌────────────────────┐  │  ┌────────────────────┐                │
│  │  │  Broker Pods       │  │  │  Broker Pods       │                │
│  │  │  (use devices)     │  │  │  (use devices)     │                │
│  │  └────────────────────┘  │  └────────────────────┘                │
│  └──────────────────────────┘  └──────────────────────────────────────┘
│         │                              │                                │
│         ▼                              ▼                                │
│  [USB Camera]                    [Network Device]                      │
│  [Sensors]                       [IP Camera]                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Akri Agent

**Purpose**: Device plugin manager running on each node

**Key Responsibilities**:
- Watches Configuration CRDs
- Manages Discovery Handler registry
- Coordinates device discovery
- Implements Kubernetes Device Plugin API
- Creates and updates Instance CRDs
- Exposes devices to kubelet

**Deployment**: DaemonSet (one per node)

**See**: [Agent Documentation](agent.md)

```
┌─────────────────────────────────────────────────────────────────┐
│                         Akri Agent                               │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           Discovery Configuration Controller               │ │
│  │  • Watches Configuration CRDs                              │ │
│  │  • Starts/stops discovery based on Configurations          │ │
│  │  • Manages backoff on errors                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         Discovery Handler Manager & Registry              │ │
│  │  • gRPC server for DH registration                         │ │
│  │  • Maintains map: handler name → endpoint                  │ │
│  │  • Routes discovery requests to handlers                   │ │
│  │  • Handles DH lifecycle                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Device Manager                             │ │
│  │  • In-memory device store                                  │ │
│  │  • Tracks: Configuration → Devices                         │ │
│  │  • Notifies plugin manager of changes                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            Device Plugin Instance Controller               │ │
│  │  • Creates device plugin per Configuration                 │ │
│  │  • Implements Device Plugin API (v1/v1beta1)               │ │
│  │  • Creates/updates Instance CRDs                           │ │
│  │  • Manages device allocation                               │ │
│  │  • Unix domain socket management                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │             Device Plugin Slot Reclaimer                    │ │
│  │  • Monitors device availability                            │ │
│  │  • Reclaims slots from disappeared devices                 │ │
│  │  • Updates Instance CRDs                                   │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Metrics Server                              │ │
│  │  • Prometheus metrics endpoint                             │ │
│  │  • Device availability metrics                             │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
           │                           │
           ▼                           ▼
    [Kubelet Device                [K8s API Server]
     Plugin Socket]                (Instance CRDs)
```

### 2. Discovery Handlers

**Purpose**: Protocol-specific device discovery plugins

**Key Responsibilities**:
- Register with Agent
- Implement protocol-specific discovery (ONVIF, udev, OPC UA, etc.)
- Continuously scan for devices
- Stream discovered devices to Agent
- Apply filters from Configuration

**Deployment**: Embedded in Agent or standalone DaemonSet

**See**: [Discovery Handlers Documentation](discovery-handlers.md)

```
┌─────────────────────────────────────────────────────────────────┐
│              Discovery Handler (Generic Structure)               │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Main Entry Point                          │ │
│  │  • Initializes handler                                     │ │
│  │  • Registers with Agent via gRPC                           │ │
│  │  • Starts gRPC server                                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            gRPC DiscoveryHandler Service                    │ │
│  │  • Implements Discover(DiscoverRequest) RPC                │ │
│  │  • Returns stream of DiscoverResponse                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │            Protocol-Specific Discovery                      │ │
│  │  ONVIF:  • WS-Discovery multicast probe                    │ │
│  │          • Device metadata retrieval                       │ │
│  │          • RTSP URL construction                           │ │
│  │          • Credential management                           │ │
│  │                                                             │ │
│  │  udev:   • Enumerate devices via libudev                   │ │
│  │          • Parse udev rules                                │ │
│  │          • Build device tree                               │ │
│  │          • Determine device paths                          │ │
│  │                                                             │ │
│  │  OPC UA: • OPC UA Discovery Server queries                 │ │
│  │          • Application discovery                           │ │
│  │          • Certificate management                          │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                  Discovery Loop                             │ │
│  │  • Runs every DISCOVERY_INTERVAL_SECS (10s default)        │ │
│  │  • Compares with previous discovery results               │ │
│  │  • Applies filters (IP, MAC, UUID, scopes, etc.)          │ │
│  │  • Sends updates only when changes detected                │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                Device List Builder                          │ │
│  │  • Creates Device proto messages                           │ │
│  │  • Sets device ID (unique identifier)                      │ │
│  │  • Populates properties (env vars for brokers)             │ │
│  │  • Specifies mounts (volume mounts)                        │ │
│  │  • Specifies device_specs (device files)                   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
                    [Streams to Agent via gRPC]
```

### 3. Akri Controller

**Purpose**: Cluster-wide controller managing broker deployments

**Key Responsibilities**:
- Watches Instance CRDs
- Deploys broker Pods/Jobs for discovered devices
- Creates Kubernetes Services (instance and configuration services)
- Monitors node availability
- Watches broker Pod state
- Cleans up resources

**Deployment**: Single replica Deployment

**See**: [Controller Documentation](controller.md)

```
┌─────────────────────────────────────────────────────────────────┐
│                      Akri Controller                             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              Instance Action Handler                        │ │
│  │  • Watches Instance CRDs (add/modify/delete)               │ │
│  │  • Determines broker deployment strategy                   │ │
│  │  • Coordinates with other handlers                         │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│  ┌────────────────────────┴───────────────────────────────────┐ │
│  │                                                             │ │
│  ▼                          ▼                          ▼       │ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │ │
│  │   Broker     │  │   Service    │  │   Resource   │        │ │
│  │ Pod/Job Mgmt │  │   Manager    │  │   Cleanup    │        │ │
│  │              │  │              │  │              │        │ │
│  │ • Create Pod │  │ • Instance   │  │ • Finalizers │        │ │
│  │ • Update Pod │  │   Services   │  │ • Owner refs │        │ │
│  │ • Delete Pod │  │ • Config     │  │ • GC         │        │ │
│  │ • Set owner  │  │   Services   │  │              │        │ │
│  │   references │  │              │  │              │        │ │
│  └──────────────┘  └──────────────┘  └──────────────┘        │ │
│                                                                 │ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Node Watcher                              │ │
│  │  • Watches Node resources                                  │ │
│  │  • Detects node disappearance                              │ │
│  │  • Cleans up orphaned Instances                            │ │
│  │  • Removes disappeared nodes from Instance.spec.nodes      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                Broker Pod Watcher                           │ │
│  │  • Watches broker Pod state changes                        │ │
│  │  • Updates metrics (broker_pod_count)                      │ │
│  │  • Detects Pod failures                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Metrics Server                              │ │
│  │  • Prometheus metrics endpoint                             │ │
│  │  • Broker Pod count by Configuration and Node              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Configuration CRD

**Purpose**: User-defined specification of what to discover and how to use it

**Key Fields**:
- `discoveryHandler`: Which handler to use and discovery parameters
- `capacity`: How many nodes can share a device
- `brokerSpec`: Pod/Job specification for broker workloads
- `instanceServiceSpec`: Service for individual devices
- `configurationServiceSpec`: Service for all devices of this type
- `brokerProperties`: Environment variables for broker Pods

**See**: [Custom Resources Documentation](custom-resources.md)

### 5. Instance CRD

**Purpose**: Represents a specific discovered device

**Key Fields**:
- `configurationName`: Parent Configuration
- `cdiName`: CDI (Container Device Interface) name
- `capacity`: Number of slots available
- `shared`: Whether multiple nodes can access
- `nodes`: List of nodes that can see this device
- `deviceUsage`: Map of slots to nodes
- `brokerProperties`: Device-specific properties + Configuration properties

**See**: [Custom Resources Documentation](custom-resources.md)

## Custom Resources

### Configuration

```yaml
apiVersion: akri.sh/v0
kind: Configuration
metadata:
  name: onvif-cameras
spec:
  discoveryHandler:
    name: onvif
    discoveryDetails: |
      ipAddresses:
        action: Exclude
        items:
        - 192.168.1.100  # Exclude specific camera
  capacity: 2  # Up to 2 nodes can share each camera
  brokerSpec:
    brokerPodSpec:
      containers:
      - name: camera-broker
        image: my-camera-broker:latest
        resources:
          requests:
            akri.sh/onvif-cameras: "1"
          limits:
            akri.sh/onvif-cameras: "1"
  instanceServiceSpec:
    type: ClusterIP
    ports:
    - port: 80
      targetPort: 8080
  configurationServiceSpec:
    type: ClusterIP
    ports:
    - port: 80
  brokerProperties:
    resolution: "1920x1080"
```

### Instance

```yaml
apiVersion: akri.sh/v0
kind: Instance
metadata:
  name: onvif-cameras-abc123
  ownerReferences:
  - apiVersion: akri.sh/v0
    kind: Configuration
    name: onvif-cameras
    uid: ...
spec:
  configurationName: onvif-cameras
  cdiName: akri.sh/onvif-cameras=abc123
  capacity: 2
  shared: true
  nodes:
  - worker-1
  - worker-2
  deviceUsage:
    "0": "worker-1"
    "1": ""  # Available slot
  brokerProperties:
    RESOLUTION: "1920x1080"  # From Configuration
    ONVIF_DEVICE_SERVICE_URL: "http://192.168.1.10/onvif/device_service"
    ONVIF_DEVICE_IP_ADDRESS: "192.168.1.10"
    ONVIF_DEVICE_MAC_ADDRESS: "aa:bb:cc:dd:ee:ff"
```

## Data Flow

### Complete Device Discovery to Broker Flow

```
1. User Action
   │
   └──> kubectl apply -f configuration.yaml
        │
        ▼
2. Configuration CRD Created
   │
   └──> Stored in etcd, watched by Agents on all nodes
        │
        ▼
3. Agent Detects Configuration
   │
   └──> Configuration Controller in Agent sees new Configuration
        │
        ▼
4. Agent Looks Up Discovery Handler
   │
   └──> Checks Discovery Handler Registry for "onvif" handler
        │
        ▼
5. Agent Calls Discovery Handler
   │
   └──> gRPC Discover(discovery_details, discovery_properties)
        │
        ▼
6. Discovery Handler Discovers Devices
   │
   └──> ONVIF: WS-Discovery multicast, device probes
        udev: Query libudev for matching devices
        OPC UA: Query OPC UA Discovery Server
        │
        ▼
7. Discovery Handler Streams Results
   │
   └──> Stream of DiscoverResponse messages to Agent
        │
        ▼
8. Agent Updates Device Manager
   │
   └──> In-memory map updated with discovered devices
        │
        ▼
9. Device Manager Notifies Device Plugin Manager
   │
   └──> "New device discovered for Configuration X"
        │
        ▼
10. Device Plugin Manager Creates/Updates Instance
    │
    └──> Creates Instance CRD via Kubernetes API
         Sets spec.nodes with current node
         Creates device plugin if needed
         │
         ▼
11. Device Plugin Advertises to Kubelet
    │
    └──> Unix domain socket: "I have 1 akri.sh/onvif-cameras"
         │
         ▼
12. Kubelet Updates Node Status
    │
    └──> Node allocatable resources updated
         │
         ▼
13. Controller Watches Instance
    │
    └──> Instance Action Handler sees new Instance
         │
         ▼
14. Controller Deploys Broker Pod
    │
    └──> Creates Pod from Configuration.brokerSpec
         Adds env vars from Instance.brokerProperties
         Sets requests/limits for device resource
         Sets owner reference to Instance
         │
         ▼
15. Scheduler Places Broker Pod
    │
    └──> Finds node with available akri.sh/onvif-cameras
         │
         ▼
16. Kubelet Calls Device Plugin Allocate
    │
    └──> Agent's device plugin allocates device slot
         Updates Instance.deviceUsage
         Returns device specs and mounts
         │
         ▼
17. Kubelet Starts Broker Pod
    │
    └──> Pod has access to device
         Environment variables set from Instance.brokerProperties
         Devices mounted per device_specs
         │
         ▼
18. Broker Application Runs
    │
    └──> Uses device (e.g., streams video from camera)
         Accessible via Instance Service
```

### Device Disappearance Flow

```
1. Device Disappears
   │
   └──> Physical disconnect, network issue, or power off
        │
        ▼
2. Discovery Handler Detects Absence
   │
   └──> Next discovery iteration (10s default) doesn't find device
        │
        ▼
3. Discovery Handler Streams Updated List
   │
   └──> Sends DiscoverResponse without the disappeared device
        │
        ▼
4. Agent Updates Device Manager
   │
   └──> Device removed from in-memory map
        │
        ▼
5. Device Plugin Slot Reclaimer Detects
   │
   └──> Polls for disappeared devices
        │
        ▼
6. Slot Reclaimer Updates Instance
   │
   └──> Removes node from Instance.spec.nodes
        If last node, deletes Instance CRD
        │
        ▼
7. Device Plugin Updates Kubelet
   │
   └──> Reports unhealthy device to kubelet
        │
        ▼
8. Kubelet Terminates Broker Pod
   │
   └──> Pod using the device is terminated
        │
        ▼
9. Controller Sees Instance Deletion
   │
   └──> Owner references trigger broker Pod deletion
        Services cleaned up
```

### Instance Lifecycle State Machine

The following diagram shows the complete lifecycle of an Instance from discovery to deletion:

```
                     ┌─────────────────────────┐
                     │  Configuration Created  │
                     └────────────┬────────────┘
                                  │
                                  ▼
                     ┌─────────────────────────┐
                     │   Device Discovered     │
                     │   (Discovery Handler)   │
                     └────────────┬────────────┘
                                  │
                                  ▼
         ┌────────────────────────────────────────────┐
         │         Instance State: CREATED            │
         │  • Instance CRD created                    │
         │  • spec.nodes: [node-a]                    │
         │  • deviceUsage: {"0": ""}  (free slot)     │
         └────────┬──────────────────┬────────────────┘
                  │                  │
                  │                  │ Device discovered on more nodes
                  │                  ▼
                  │        ┌─────────────────────────┐
                  │        │  Instance State: SHARED │
                  │        │  • spec.nodes: [a, b]   │
                  │        │  • capacity: 2          │
                  │        └──────────┬──────────────┘
                  │                   │
                  │ Controller watches Instance
                  ▼                   │
         ┌────────────────────────────┴───────────────┐
         │    Instance State: BROKER PENDING          │
         │  • Controller creates broker Pods          │
         │  • Pods: Pending (waiting for schedule)    │
         └────────────────┬───────────────────────────┘
                          │
                          │ Scheduler places Pods
                          ▼
         ┌─────────────────────────────────────────────┐
         │    Instance State: ALLOCATING               │
         │  • Kubelet calls Device Plugin Allocate()   │
         │  • Agent updates deviceUsage                │
         │    {"0": "node-a", "1": "node-b"}           │
         └────────────────┬────────────────────────────┘
                          │
                          │ Allocation successful
                          ▼
         ┌─────────────────────────────────────────────┐
         │    Instance State: IN USE                   │
         │  • Broker Pods: Running                     │
         │  • deviceUsage: all slots allocated         │
         │  • Services: Created                        │
         │  ✓ Fully operational                        │
         └─────┬──────────────────────┬────────────────┘
               │                      │
               │                      │ Broker Pod crashes
               │                      ▼
               │           ┌──────────────────────────┐
               │           │  Instance State: PARTIAL │
               │           │  • Some slots free       │
               │           │  • Controller recreates  │
               │           │    failed broker Pods    │
               │           └────────┬─────────────────┘
               │                    │
               │                    │ Broker restarts
               │                    └────────┐
               │                             │
               │ Device disappears           ▼
               │ from one node         ┌──────────┐
               ▼                       │   Back   │
    ┌──────────────────────┐          │    to    │
    │ Instance State:      │          │  IN USE  │
    │   NODE REMOVED       │◄─────────┴──────────┘
    │ • Node removed from  │
    │   spec.nodes         │
    │ • deviceUsage slot   │
    │   freed              │
    │ • Broker Pod deleted │
    └──────┬───────────────┘
           │
           │ Device disappears from all nodes
           ▼
    ┌──────────────────────┐
    │ Instance State:      │
    │   TERMINATING        │
    │ • Instance deleted   │
    │ • Owner refs trigger:│
    │   - Broker Pods →    │
    │   - Services →       │
    └──────┬───────────────┘
           │
           ▼
    ┌──────────────────────┐
    │   Instance DELETED   │
    │  (GC by Kubernetes)  │
    └──────────────────────┘


    State Transitions Summary:
    ══════════════════════════

    CREATED → SHARED          : More nodes discover device
    CREATED → BROKER_PENDING  : Controller sees Instance
    BROKER_PENDING → ALLOCATING : Scheduler places Pods
    ALLOCATING → IN_USE       : Allocation successful
    IN_USE → PARTIAL          : Broker Pod fails
    PARTIAL → IN_USE          : Broker Pod recovers
    IN_USE → NODE_REMOVED     : Device disappears from one node
    NODE_REMOVED → TERMINATING: Device gone from all nodes
    TERMINATING → DELETED     : Garbage collection
```

## Component Interactions

### Agent ↔ Discovery Handler

```
┌──────────────┐                           ┌───────────────────┐
│    Agent     │                           │ Discovery Handler │
└──────┬───────┘                           └─────────┬─────────┘
       │                                             │
       │  1. Discovery Handler Startup              │
       │◄────────────────────────────────────────────┤
       │     RegisterDiscoveryHandler(name, endpoint)
       │                                             │
       │  2. Registration Acknowledged              │
       ├────────────────────────────────────────────►│
       │     Empty response                          │
       │                                             │
       │  3. Configuration Created/Updated          │
       │                                             │
       │  4. Discover Request                       │
       ├────────────────────────────────────────────►│
       │     Discover(discovery_details, props)     │
       │                                             │
       │  5. Continuous Device Stream               │
       │◄────────────────────────────────────────────┤
       │     Stream<DiscoverResponse>               │
       │     devices: [Device{id, properties, ...}] │
       │                                             │
       │  6. Stream continues until...              │
       │     - Configuration deleted                │
       │     - Discovery Handler crashes            │
       │     - Agent stops                          │
       │                                             │
       │  7. If stream breaks...                    │
       │◄────────────────────────────────────────────┤
       │     RegisterDiscoveryHandler (re-register) │
       │                                             │
```

### Agent ↔ Kubelet

```
┌──────────────┐                           ┌───────────────┐
│    Agent     │                           │    Kubelet    │
└──────┬───────┘                           └───────┬───────┘
       │                                           │
       │  1. Device Plugin Registration           │
       ├──────────────────────────────────────────►│
       │     Register(socket, resource_name)      │
       │     e.g., akri.sh/onvif-cameras          │
       │                                           │
       │  2. List Devices                         │
       │◄──────────────────────────────────────────┤
       │     ListAndWatch()                       │
       │                                           │
       │  3. Device List Response                 │
       ├──────────────────────────────────────────►│
       │     Stream<ListAndWatchResponse>         │
       │     devices: [Device{ID, Health}]        │
       │                                           │
       │  4. Pod Scheduled                        │
       │◄──────────────────────────────────────────┤
       │     Allocate(device_ids)                 │
       │                                           │
       │  5. Allocation Response                  │
       ├──────────────────────────────────────────►│
       │     AllocateResponse{                    │
       │       devices: [DeviceSpec],             │
       │       mounts: [Mount],                   │
       │       envs: {...}                        │
       │     }                                     │
       │                                           │
       │  6. Device Health Updates                │
       │     (if device disappears)               │
       ├──────────────────────────────────────────►│
       │     ListAndWatchResponse{                │
       │       devices: [Device{ID, Unhealthy}]   │
       │     }                                     │
```

### Agent ↔ Kubernetes API

```
┌──────────────┐                           ┌───────────────┐
│    Agent     │                           │  API Server   │
└──────┬───────┘                           └───────┬───────┘
       │                                           │
       │  1. Watch Configurations                 │
       ├──────────────────────────────────────────►│
       │     GET /apis/akri.sh/v0/configurations  │
       │     ?watch=true                          │
       │                                           │
       │  2. Configuration Events                 │
       │◄──────────────────────────────────────────┤
       │     ADDED/MODIFIED/DELETED               │
       │                                           │
       │  3. Create Instance                      │
       ├──────────────────────────────────────────►│
       │     POST /apis/akri.sh/v0/namespaces/    │
       │          default/instances               │
       │     body: Instance spec                  │
       │                                           │
       │  4. Update Instance                      │
       ├──────────────────────────────────────────►│
       │     PATCH /apis/akri.sh/v0/namespaces/   │
       │           default/instances/{name}       │
       │     body: {spec: {deviceUsage, nodes}}   │
       │                                           │
       │  5. Delete Instance                      │
       ├──────────────────────────────────────────►│
       │     DELETE /apis/akri.sh/v0/namespaces/  │
       │            default/instances/{name}      │
```

### Controller ↔ Kubernetes API

```
┌──────────────┐                           ┌───────────────┐
│  Controller  │                           │  API Server   │
└──────┬───────┘                           └───────┬───────┘
       │                                           │
       │  1. Watch Instances                      │
       ├──────────────────────────────────────────►│
       │     GET /apis/akri.sh/v0/instances       │
       │     ?watch=true                          │
       │                                           │
       │  2. Instance Events                      │
       │◄──────────────────────────────────────────┤
       │     ADDED: new device discovered         │
       │                                           │
       │  3. Get Configuration                    │
       ├──────────────────────────────────────────►│
       │     GET /apis/akri.sh/v0/configurations/ │
       │         {instance.spec.configurationName}│
       │                                           │
       │  4. Create Broker Pod                    │
       ├──────────────────────────────────────────►│
       │     POST /api/v1/namespaces/default/pods │
       │     body: Pod spec from Configuration    │
       │     ownerReferences: [Instance]          │
       │                                           │
       │  5. Create Instance Service              │
       ├──────────────────────────────────────────►│
       │     POST /api/v1/namespaces/default/     │
       │          services                        │
       │     selector: instance={instance-name}   │
       │                                           │
       │  6. Watch Nodes                          │
       ├──────────────────────────────────────────►│
       │     GET /api/v1/nodes?watch=true         │
       │                                           │
       │  7. Node Deleted                         │
       │◄──────────────────────────────────────────┤
       │     DELETED event                        │
       │                                           │
       │  8. Clean Up Instances                   │
       ├──────────────────────────────────────────►│
       │     PATCH instances to remove node       │
```

## Deployment Architecture

### Typical Deployment

```yaml
# Akri Controller (Cluster-wide)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: akri-controller-deployment
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: controller
        image: ghcr.io/project-akri/akri/controller:latest

---
# Akri Agent (Per-node)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-agent-daemonset
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: agent
        image: ghcr.io/project-akri/akri/agent:latest
        env:
        - name: AGENT_NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
        - name: pod-resources
          mountPath: /var/lib/kubelet/pod-resources
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
      - name: pod-resources
        hostPath:
          path: /var/lib/kubelet/pod-resources

---
# Discovery Handler (Per-node, per-protocol)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: akri-onvif-discovery-handler
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: onvif-dh
        image: ghcr.io/project-akri/akri/onvif-discovery-handler:latest
```

### Resource Hierarchy

```
Cluster
│
├── Akri Controller (Deployment, 1 replica)
│   └── Watches all Instances across all namespaces
│
├── Node 1
│   ├── Akri Agent (DaemonSet Pod)
│   │   ├── Device Plugin for Configuration A
│   │   ├── Device Plugin for Configuration B
│   │   └── Device Plugin Sockets in /var/lib/kubelet/device-plugins/
│   │
│   ├── ONVIF Discovery Handler (DaemonSet Pod)
│   ├── udev Discovery Handler (DaemonSet Pod)
│   ├── OPC UA Discovery Handler (DaemonSet Pod)
│   │
│   ├── Broker Pod 1 (for Instance X)
│   └── Broker Pod 2 (for Instance Y)
│
└── Node 2
    ├── Akri Agent (DaemonSet Pod)
    ├── ONVIF Discovery Handler (DaemonSet Pod)
    ├── udev Discovery Handler (DaemonSet Pod)
    ├── OPC UA Discovery Handler (DaemonSet Pod)
    └── Broker Pod 3 (for Instance X) <- Shared instance
```

## Security Considerations

### Agent Permissions
- **RBAC**: Needs read/write access to Configuration and Instance CRDs
- **Device Access**: Runs privileged for device plugin registration
- **Host Network**: May need host network for discovery

### Discovery Handler Permissions
- **RBAC**: No direct Kubernetes API access needed
- **Network**: May need host network for device discovery (ONVIF, OPC UA)
- **Device Access**: udev handler needs access to /dev

### Controller Permissions
- **RBAC**: Needs read access to Instances, Configurations; write access to Pods, Services
- **Cluster-wide**: Watches resources across all namespaces

## Performance and Scalability

### Agent
- One per node (DaemonSet)
- Scales linearly with cluster size
- In-memory device tracking per node

### Discovery Handlers
- One per protocol per node
- Discovery interval: 10 seconds (configurable)
- Efficient streaming protocol

### Controller
- Single replica (leader election possible)
- Watches all Instances cluster-wide
- Scales to thousands of Instances

### Resource Estimates

| Component | CPU (requests) | Memory (requests) | Notes |
|-----------|----------------|-------------------|-------|
| Agent | 100m | 128Mi | Per node |
| Controller | 100m | 128Mi | Cluster-wide |
| ONVIF DH | 50m | 64Mi | Per node |
| udev DH | 50m | 64Mi | Per node |
| OPC UA DH | 50m | 64Mi | Per node |

## Next Steps

- [Agent Component Details](agent.md)
- [Controller Component Details](controller.md)
- [Discovery Handlers](discovery-handlers.md)
- [Custom Resources](custom-resources.md)
- [Agent vs Discovery Handlers](agent-vs-handlers.md)
