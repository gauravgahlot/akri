# Akri Documentation

This directory contains comprehensive documentation for the Akri project - a Kubernetes Resource Interface for the Edge.

## Table of Contents

### Core Concepts
- [Architecture Overview](architecture.md) - High-level architecture and system components
- [Custom Resources](custom-resources.md) - Configuration and Instance CRDs
- [Agent vs Discovery Handlers](agent-vs-handlers.md) - **Critical distinction between these components**

### Component Documentation
- [Agent Component](agent.md) - Device plugin implementation that runs on each node
- [Controller Component](controller.md) - Kubernetes controller managing broker deployments
- [Discovery Handlers](discovery-handlers.md) - Protocol-specific device discovery implementations
- [Broker Pods](brokers.md) - Workload pods that utilize discovered devices

### Protocols and Discovery
- [ONVIF Discovery Handler](discovery-handlers/onvif.md) - IP camera discovery
- [udev Discovery Handler](discovery-handlers/udev.md) - USB and local device discovery
- [OPC UA Discovery Handler](discovery-handlers/opcua.md) - Industrial automation protocol
- [Debug Echo Discovery Handler](discovery-handlers/debug-echo.md) - Testing and development

### Deployment and Operations
- [Deployment Guide](deployment.md) - Installing and configuring Akri
- [Configuration Examples](configuration-examples.md) - Sample configurations for different use cases

### Development
- [Development Guide](development.md) - Building, testing, and contributing
- [Creating Custom Discovery Handlers](creating-discovery-handlers.md) - Extending Akri with new protocols
- [Broker Development](broker-development.md) - Building custom broker applications

## Quick Overview

### What is Akri?

Akri is a Kubernetes project that makes it easy to expose heterogeneous leaf devices (cameras, sensors, USB devices, GPUs, etc.) as Kubernetes resources. It continuously monitors device availability and schedules workloads accordingly.

**Key Features:**
- **Dynamic Device Discovery**: Automatically finds and tracks devices
- **Protocol Extensibility**: Support for ONVIF, udev, OPC UA, and custom protocols
- **High Availability**: Multiple nodes can share network-accessible devices
- **Automatic Service Creation**: Kubernetes services created for discovered devices
- **Resource Management**: Integrates with Kubernetes device plugin framework

### Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                 Akri Controller                             │ │
│  │  - Watches Instances                                        │ │
│  │  - Deploys Broker Pods                                      │ │
│  │  - Creates Services                                         │ │
│  │  - Monitors Nodes                                           │ │
│  └────────────────────────────────────────────────────────────┘ │
│                            │                                      │
│                            ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Configuration CRD (User-defined)                ││
│  │  - Which protocol to discover (ONVIF, udev, OPC UA)         ││
│  │  - Discovery filters/parameters                             ││
│  │  - Broker Pod specification                                 ││
│  │  - Capacity (how many nodes can share)                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                            │                                      │
│         ┌──────────────────┴──────────────────┐                  │
│         ▼                                      ▼                  │
│  ┌─────────────┐                       ┌─────────────┐           │
│  │   Node A    │                       │   Node B    │           │
│  │             │                       │             │           │
│  │  ┌────────────────────┐            │  ┌────────────────────┐ │
│  │  │   Akri Agent       │            │  │   Akri Agent       │ │
│  │  │  - Device Manager  │            │  │  - Device Manager  │ │
│  │  │  - DH Registry     │            │  │  - DH Registry     │ │
│  │  │  - Device Plugins  │            │  │  - Device Plugins  │ │
│  │  └────────────────────┘            │  └────────────────────┘ │
│  │           │                         │           │             │
│  │           ▼                         │           ▼             │
│  │  ┌────────────────────┐            │  ┌────────────────────┐ │
│  │  │ Discovery Handlers │            │  │ Discovery Handlers │ │
│  │  │  - ONVIF DH        │            │  │  - ONVIF DH        │ │
│  │  │  - udev DH         │            │  │  - udev DH         │ │
│  │  │  - OPC UA DH       │            │  │  - OPC UA DH       │ │
│  │  └────────────────────┘            │  └────────────────────┘ │
│  │           │                         │           │             │
│  │           ▼                         │           ▼             │
│  │  ┌────────────────────┐            │  ┌────────────────────┐ │
│  │  │ Creates Instances  │            │  │ Creates Instances  │ │
│  │  └────────────────────┘            │  └────────────────────┘ │
│  │           │                         │           │             │
│  │           ▼                         │           ▼             │
│  │  ┌────────────────────┐            │  ┌────────────────────┐ │
│  │  │   Broker Pods      │            │  │   Broker Pods      │ │
│  │  │ (Use the devices)  │            │  │ (Use the devices)  │ │
│  │  └────────────────────┘            │  └────────────────────┘ │
│  └─────────────┘                       └─────────────┘           │
└───────────────────────────────────────────────────────────────────┘
          │                                       │
          ▼                                       ▼
    [USB Camera]                           [IP Camera]
    [Sensors]                              [Network Devices]
```

### How It Works: The Complete Flow

```
                    ┌─────────────────────────────────────────────┐
                    │  1. User Creates Configuration              │
                    │     kubectl apply -f config.yaml            │
                    └──────────────────┬──────────────────────────┘
                                       │
                                       ▼
                    ┌─────────────────────────────────────────────┐
                    │  2. Configuration CRD Created in K8s        │
                    │     Stored in etcd, watched by Agents       │
                    └──────────┬──────────────────┬───────────────┘
                               │                  │
            ┌──────────────────┘                  └──────────────────┐
            ▼                                                         ▼
┌───────────────────────────┐                        ┌───────────────────────────┐
│  3a. Agent on Node A      │                        │  3b. Agent on Node B      │
│      Detects Config       │                        │      Detects Config       │
└───────────┬───────────────┘                        └───────────┬───────────────┘
            │                                                     │
            ▼                                                     ▼
┌───────────────────────────┐                        ┌───────────────────────────┐
│  4a. Calls ONVIF          │                        │  4b. Calls ONVIF          │
│      Discovery Handler    │                        │      Discovery Handler    │
└───────────┬───────────────┘                        └───────────┬───────────────┘
            │                                                     │
            ▼                                                     ▼
┌───────────────────────────┐                        ┌───────────────────────────┐
│  5a. Discovers:           │                        │  5b. Discovers:           │
│      • IP Camera A        │                        │      • IP Camera A (same!)│
│      • IP Camera B        │                        │      • IP Camera C        │
└───────────┬───────────────┘                        └───────────┬───────────────┘
            │                                                     │
            ▼                                                     ▼
┌───────────────────────────┐                        ┌───────────────────────────┐
│  6a. Creates Instances:   │                        │  6b. Updates/Creates:     │
│      • camera-a (new)     │                        │      • camera-a (update)  │
│        nodes: [node-a]    │                        │        nodes: [a, b]      │
│      • camera-b (new)     │                        │      • camera-c (new)     │
│        nodes: [node-a]    │                        │        nodes: [node-b]    │
└───────────┬───────────────┘                        └───────────┬───────────────┘
            │                                                     │
            └────────────────┬────────────────────────────────────┘
                             │
                             ▼
            ┌─────────────────────────────────────────────┐
            │  7. Controller Watches Instance CRDs        │
            │     Sees 3 Instances (camera-a, b, c)       │
            └──────────────────┬──────────────────────────┘
                               │
                               ▼
            ┌─────────────────────────────────────────────┐
            │  8. Controller Deploys Broker Pods:         │
            │     • camera-a-broker on node-a             │
            │     • camera-a-broker on node-b (HA!)       │
            │     • camera-b-broker on node-a             │
            │     • camera-c-broker on node-b             │
            └──────────────────┬──────────────────────────┘
                               │
                               ▼
            ┌─────────────────────────────────────────────┐
            │  9. Brokers Request Device Resources        │
            │     resources.requests:                     │
            │       akri.sh/onvif-cameras: "1"            │
            └──────────────────┬──────────────────────────┘
                               │
                               ▼
            ┌─────────────────────────────────────────────┐
            │ 10. Agent's Device Plugin Allocates         │
            │     Updates Instance.deviceUsage            │
            │     Returns device info to kubelet          │
            └──────────────────┬──────────────────────────┘
                               │
                               ▼
            ┌─────────────────────────────────────────────┐
            │ 11. Brokers Start with Device Access        │
            │     • Environment variables from properties │
            │     • Device mounts (if specified)          │
            │     • Can now use the cameras!              │
            └──────────────────┬──────────────────────────┘
                               │
                               ▼
            ┌─────────────────────────────────────────────┐
            │ 12. Services Created                        │
            │     • Instance services (per camera)        │
            │     • Configuration service (all cameras)   │
            │     Applications can now access brokers!    │
            └─────────────────────────────────────────────┘
```

**Summary**:
1. **User Creates Configuration** → Deploy Configuration CRD
2. **Agents Discover Devices** → Each Agent uses Discovery Handlers
3. **Instances Created** → Per discovered device (shared across nodes if network device)
4. **Controller Deploys Brokers** → Sees Instances and creates Pods
5. **Brokers Access Devices** → Via device plugins and resource allocation
6. **Services Created** → Stable endpoints to access brokers

### Critical Distinction: Agent vs Discovery Handlers

**This is a common point of confusion!** See [Agent vs Discovery Handlers](agent-vs-handlers.md) for a detailed explanation.

**Quick Summary:**
- **Agent**: Kubernetes device plugin that runs as a DaemonSet on each node. It manages the lifecycle of device plugins, registers Discovery Handlers, and creates Instance CRDs.
- **Discovery Handlers**: Protocol-specific plugins that run as separate processes/containers. They implement the actual device discovery logic (e.g., scanning for ONVIF cameras, querying udev for USB devices).

Think of it this way:
- **Agent** = Platform/Framework (one per node, manages everything)
- **Discovery Handlers** = Protocol Plugins (multiple, each handles a specific protocol)

## Getting Started

1. Start with [Architecture Overview](architecture.md) to understand the system
2. Read [Agent vs Discovery Handlers](agent-vs-handlers.md) to clarify the key distinction
3. Follow the [Deployment Guide](deployment.md) to install Akri
4. Try the [Configuration Examples](configuration-examples.md) for your use case
5. For development, see [Development Guide](development.md)

## Key Terminology

| Term | Description |
|------|-------------|
| **Configuration** | Custom resource defining what devices to discover and how to use them |
| **Instance** | Custom resource representing a specific discovered device |
| **Agent** | DaemonSet component managing device plugins on each node |
| **Discovery Handler** | Protocol-specific plugin that discovers devices |
| **Controller** | Cluster-wide component that deploys brokers for discovered devices |
| **Broker** | User-defined workload Pod that utilizes a discovered device |
| **Capacity** | Number of nodes that can simultaneously use a discovered device |

## Documentation Structure

```
docs/
├── README.md                           # This file - main entry point
├── architecture.md                     # High-level architecture
├── agent-vs-handlers.md               # Agent vs Discovery Handlers clarification
├── custom-resources.md                # Configuration and Instance CRDs
├── agent.md                           # Agent component details
├── controller.md                      # Controller component details
├── discovery-handlers.md              # Discovery Handlers overview
├── brokers.md                         # Broker Pods documentation
├── deployment.md                      # Deployment guide
├── configuration-examples.md          # Example configurations
├── development.md                     # Development guide
├── creating-discovery-handlers.md     # Creating custom handlers
├── broker-development.md              # Creating custom brokers
└── discovery-handlers/                # Protocol-specific docs
    ├── onvif.md
    ├── udev.md
    ├── opcua.md
    └── debug-echo.md
```

## Additional Resources

- [Official Akri Documentation](https://docs.akri.sh/)
- [Akri GitHub Repository](https://github.com/project-akri/akri)
- [CNCF Sandbox Project Page](https://www.cncf.io/sandbox-projects/)
- [Kubernetes Device Plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/)
