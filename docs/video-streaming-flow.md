# Video Streaming Flow: Webcam to Browser

This document provides a comprehensive walkthrough of how video data flows from a USB webcam through Akri's udev discovery, broker, and streaming application to finally be rendered in a web browser.

## Table of Contents
- [Overview](#overview)
- [Components Involved](#components-involved)
- [Complete Flow Diagram](#complete-flow-diagram)
- [Detailed Step-by-Step Flow](#detailed-step-by-step-flow)
- [Component Deep Dive](#component-deep-dive)
- [Data Flow Analysis](#data-flow-analysis)
- [Sequence Diagram](#sequence-diagram)

## Overview

The Akri video streaming example demonstrates end-to-end device discovery, broker deployment, and application integration. It shows how a USB webcam's video frames are:
1. Discovered by the udev Discovery Handler
2. Exposed as a Kubernetes device resource
3. Accessed by a broker Pod that captures frames
4. Served via gRPC to a streaming application
5. Displayed in a web browser via Flask/MJPEG streaming

**Technologies Used**:
- **udev** - Linux device manager for USB device discovery
- **rscam** - Rust V4L2 (Video4Linux2) wrapper for camera access
- **gRPC** - Remote procedure call for broker-to-app communication
- **Flask** - Python web framework for HTTP streaming
- **MJPEG** - Motion JPEG format for browser-compatible streaming

## Components Involved

```
┌──────────────────────────────────────────────────────────────────────┐
│                        COMPONENT ARCHITECTURE                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  [Physical Layer]                                                     │
│  ┌─────────────────┐                                                  │
│  │  USB Webcam     │  <-- Physical device                             │
│  │  /dev/video0    │                                                  │
│  └────────┬────────┘                                                  │
│           │ USB connection                                            │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Linux Kernel                            │                         │
│  │  • V4L2 (Video4Linux2) subsystem        │                         │
│  │  • udev device manager                  │                         │
│  │  • Creates /dev/video0 device node      │                         │
│  └────────┬────────────────────────────────┘                         │
│           │                                                           │
│  ═══════════════════════════════════════════════════════════════    │
│                                                                        │
│  [Akri Layer]                                                         │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  udev Discovery Handler                 │                         │
│  │  • Scans for USB video devices          │                         │
│  │  • Matches udev rules                   │                         │
│  │  • Returns device properties            │                         │
│  │    - UDEV_DEVNODE=/dev/video0           │                         │
│  └────────┬────────────────────────────────┘                         │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Akri Agent                             │                         │
│  │  • Receives discovered devices          │                         │
│  │  • Creates Instance CRD                 │                         │
│  │  • Exposes as device plugin resource    │                         │
│  │    akri.sh/udev-video: "1"              │                         │
│  └────────┬────────────────────────────────┘                         │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Akri Controller                        │                         │
│  │  • Watches Instance CRDs                │                         │
│  │  • Deploys udev-video-broker Pod        │                         │
│  │  • Creates Services                     │                         │
│  └────────┬────────────────────────────────┘                         │
│           │                                                           │
│  ═══════════════════════════════════════════════════════════════    │
│                                                                        │
│  [Broker Layer]                                                       │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  udev-video-broker Pod                  │                         │
│  │  Components:                            │                         │
│  │  ┌──────────────────────────────────┐  │                         │
│  │  │ 1. Camera Capturer               │  │                         │
│  │  │    • Opens /dev/video0           │  │                         │
│  │  │    • Configures format (MJPG)    │  │                         │
│  │  │    • Sets resolution (640x480)   │  │                         │
│  │  │    • Sets FPS (10)               │  │                         │
│  │  │    • Uses rscam (V4L2 wrapper)   │  │                         │
│  │  └──────────────────────────────────┘  │                         │
│  │  ┌──────────────────────────────────┐  │                         │
│  │  │ 2. gRPC Camera Service           │  │                         │
│  │  │    • Listens on port 8083        │  │                         │
│  │  │    • Implements GetFrame() RPC   │  │                         │
│  │  │    • Returns JPEG frames         │  │                         │
│  │  └──────────────────────────────────┘  │                         │
│  └────────┬────────────────────────────────┘                         │
│           │ gRPC (port 8083)                                         │
│           │                                                           │
│  ═══════════════════════════════════════════════════════════════    │
│                                                                        │
│  [Application Layer]                                                  │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Video Streaming App                    │                         │
│  │  Components:                            │                         │
│  │  ┌──────────────────────────────────┐  │                         │
│  │  │ 1. Service Discovery             │  │                         │
│  │  │    • Queries K8s for services    │  │                         │
│  │  │    • Finds broker services       │  │                         │
│  │  └──────────────────────────────────┘  │                         │
│  │  ┌──────────────────────────────────┐  │                         │
│  │  │ 2. gRPC Client (CameraFeed)      │  │                         │
│  │  │    • Connects to broker          │  │                         │
│  │  │    • Calls GetFrame() every 1s   │  │                         │
│  │  │    • Queues frames               │  │                         │
│  │  └──────────────────────────────────┘  │                         │
│  │  ┌──────────────────────────────────┐  │                         │
│  │  │ 3. Flask Web Server              │  │                         │
│  │  │    • Serves on port 5000         │  │                         │
│  │  │    • Streams MJPEG               │  │                         │
│  │  │    • Routes: /, /camera_frame_feed                         │  │
│  │  └──────────────────────────────────┘  │                         │
│  └────────┬────────────────────────────────┘                         │
│           │ HTTP (MJPEG stream)                                      │
│           │                                                           │
│  ═══════════════════════════════════════════════════════════════    │
│                                                                        │
│  [Client Layer]                                                       │
│           │                                                           │
│           ▼                                                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  Web Browser                            │                         │
│  │  • Loads HTML page                      │                         │
│  │  • <img> tag with multipart/x-mixed-replace                      │
│  │  • Continuously receives JPEG frames    │                         │
│  │  • Native MJPEG decoding                │                         │
│  │  • Renders video stream                 │                         │
│  └─────────────────────────────────────────┘                         │
│                                                                        │
└──────────────────────────────────────────────────────────────────────┘
```

## Complete Flow Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│                     VIDEO STREAMING COMPLETE FLOW                   │
└────────────────────────────────────────────────────────────────────┘

PHASE 1: DISCOVERY & SETUP
═══════════════════════════

[USB Webcam] ──USB──► [Linux Kernel]
                           │
                           │ udev events
                           ▼
                    [udev Discovery Handler]
                           │
                           │ Device found: /dev/video0
                           │ Properties: UDEV_DEVNODE=/dev/video0
                           ▼
                      [Akri Agent]
                           │
                           │ Creates Instance CRD
                           │ Advertises: akri.sh/udev-video: "1"
                           ▼
                   [Akri Controller]
                           │
                           │ Deploys broker Pod
                           ▼
              [udev-video-broker Pod STARTING]


PHASE 2: BROKER INITIALIZATION
═══════════════════════════════

[udev-video-broker Pod]
     │
     │ 1. Read environment variable
     │    UDEV_DEVNODE_123456=/dev/video0
     │
     ▼
[Camera Capturer Initialization]
     │
     ├──► Open device: /dev/video0
     │
     ├──► Query supported formats
     │    Response: [MJPG, YUYV, ...]
     │
     ├──► Select format: MJPG
     │    (from env var FORMAT or default)
     │
     ├──► Query supported resolutions
     │    Response: [(640,480), (1280,720), ...]
     │
     ├──► Select resolution: 640x480
     │    (from env vars RESOLUTION_WIDTH/HEIGHT or default)
     │
     ├──► Query supported frame rates
     │    Response: [1fps, 5fps, 10fps, 30fps]
     │
     ├──► Select FPS: 10
     │    (from env var FRAMES_PER_SECOND or default)
     │
     ├──► Configure V4L2 with rscam
     │    camera.start(Config {
     │      format: b"MJPG",
     │      resolution: (640, 480),
     │      interval: (1, 10), // 10 fps
     │    })
     │
     └──► Camera ready to capture!

[gRPC Server Initialization]
     │
     ├──► Create CameraService
     │    • Holds RsCamera instance
     │    • Implements Camera trait
     │
     ├──► Start gRPC server on 0.0.0.0:8083
     │
     └──► Server listening...


PHASE 3: APPLICATION INITIALIZATION
════════════════════════════════════

[Video Streaming App STARTING]
     │
     │ Read env: CONFIGURATION_NAME=akri-udev
     │
     ▼
[Service Discovery]
     │
     ├──► Query Kubernetes API for services
     │    coreV1Api.list_service_for_all_namespaces()
     │
     ├──► Find Configuration service:
     │    • akri-udev-svc (ClusterIP)
     │    • Port: grpc/8083
     │    • URL: 10.96.1.100:8083
     │
     ├──► Find Instance services:
     │    • akri-udev-a1b2c3-svc
     │    • akri-udev-d4e5f6-svc
     │    • Each with grpc/8083 port
     │
     └──► Create CameraFeed objects for each

[CameraFeed Threads Start]
     │
     └──► For each camera:
          • Spawn thread
          • Create gRPC client
          • Start get_frames() loop

[Flask Web Server]
     │
     └──► Listen on 0.0.0.0:5000
          Routes:
          • GET /  → index.html
          • GET /camera_list → JSON camera IDs
          • GET /camera_frame_feed/<id> → MJPEG stream


PHASE 4: FRAME CAPTURE & STREAMING (Continuous Loop)
═════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────┐
│  FRAME FLOW (Repeats every ~1 second per camera)                │
└─────────────────────────────────────────────────────────────────┘

Thread in Streaming App:
│
│ [1] Create gRPC channel
│     channel = grpc.insecure_channel(broker_url)
│
│ [2] Create stub
│     stub = camera_pb2_grpc.CameraStub(channel)
│
│ [3] Call GetFrame RPC
│     stub.GetFrame(NotifyRequest())
│          │
│          │ Network (gRPC over TCP)
│          ▼
│     ┌──────────────────────────────────┐
│     │  udev-video-broker Pod           │
│     │  gRPC Server receives request    │
│     │                                   │
│     │  [4] CameraService.get_frame()   │
│     │       │                          │
│     │       │ [5] camera.capture()     │
│     │       │     (rscam/V4L2)         │
│     │       │          │               │
│     │       │          ▼               │
│     │       │   ┌──────────────────┐  │
│     │       │   │ V4L2 ioctl calls │  │
│     │       │   │ to /dev/video0   │  │
│     │       │   └────────┬─────────┘  │
│     │       │            │            │
│     │       │            ▼            │
│     │       │   [Physical webcam     │
│     │       │    captures frame]     │
│     │       │            │            │
│     │       │            ▼            │
│     │       │   [MJPEG frame data    │
│     │       │    in kernel buffer]   │
│     │       │            │            │
│     │       │   [6] Return frame[]   │
│     │       │            │            │
│     │       ▼            ▼            │
│     │  [7] Create NotifyResponse {   │
│     │       frame: Vec<u8>,          │
│     │       camera: "/dev/video0"    │
│     │      }                          │
│     │                                 │
│     │  [8] Return response           │
│     └───────────┬──────────────────────┘
│                 │
│                 │ gRPC response
│                 ▼
│ [9] Receive frame (bytes)
│     frame = response.frame
│
│ [10] Put frame in queue (size 1)
│      self.queue.put(frame)
│      (Replaces old frame if queue full)
│
│ [11] Close channel
│      channel.close()
│
│ [12] Sleep 1 second
│      sleep(1)
│
└──► Repeat from [1]


PHASE 5: WEB BROWSER STREAMING (Continuous)
════════════════════════════════════════════

[Browser]
│
│ [1] User opens http://streaming-app:5000/
│
│ [2] Flask renders index.html
│     <img src="/camera_frame_feed/0">
│
│ [3] Browser requests MJPEG stream
│     GET /camera_frame_feed/0
│          │
│          ▼
│     ┌─────────────────────────────┐
│     │  Flask Route Handler        │
│     │  camera_frame_feed()        │
│     │       │                     │
│     │       ▼                     │
│     │  selected_camera.          │
│     │    generator_func()         │
│     │       │                     │
│     │       ▼                     │
│     │  while not stopped:         │
│     │    [4] frame = queue.get()  │
│     │        (blocks until frame  │
│     │         available)           │
│     │       │                     │
│     │       ▼                     │
│     │    [5] yield:               │
│     │      '--frame\r\n'          │
│     │      'Content-Type: '       │
│     │       'image/jpeg\r\n\r\n'  │
│     │      + frame_bytes          │
│     │      + '\r\n'                │
│     │       │                     │
│     └───────┼─────────────────────┘
│             │
│             │ HTTP response chunk
│             ▼
│ [6] Browser receives JPEG frame
│
│ [7] Browser decodes JPEG
│
│ [8] Browser renders image in <img> tag
│
│ [9] Browser expects next frame
│     (multipart/x-mixed-replace)
│
└──► Repeat from [4]


SUMMARY OF DATA FLOW:
══════════════════════

Physical Device → V4L2 Kernel → rscam → gRPC → Flask → HTTP → Browser

Frame Size: ~5-50 KB (MJPEG compressed)
Latency: ~1-2 seconds (depending on FPS setting)
Format: JPEG frames in MJPEG container
Frame Rate: 10 FPS (default, configurable)
```

## Detailed Step-by-Step Flow

### Step 1: Device Discovery (udev Discovery Handler)

**Code**: [`discovery-handlers/udev/`](../discovery-handlers/udev/)

1. udev Discovery Handler scans for devices matching udev rules
2. For video devices, it looks for `KERNEL=="video[0-9]*"`
3. Finds `/dev/video0` (USB webcam)
4. Creates Device message with:
   ```protobuf
   Device {
     id: "video0-usb-path",
     properties: {
       "UDEV_DEVNODE": "/dev/video0",
       "UDEV_DEVPATH": "/sys/devices/...",
     },
     device_specs: [{
       container_path: "/dev/video0",
       host_path: "/dev/video0",
       permissions: "rwm"
     }]
   }
   ```
5. Streams device to Agent

### Step 2: Instance Creation (Akri Agent)

**Code**: [`agent/src/`](../agent/src/)

1. Agent receives Device from Discovery Handler
2. Creates Instance CRD:
   ```yaml
   apiVersion: akri.sh/v0
   kind: Instance
   metadata:
     name: udev-video-abc123
   spec:
     configurationName: udev-video
     nodes: [worker-1]
     brokerProperties:
       UDEV_DEVNODE_ABC123: "/dev/video0"
   ```
3. Creates device plugin for `akri.sh/udev-video`
4. Advertises to kubelet: `akri.sh/udev-video: "1"`

### Step 3: Broker Deployment (Akri Controller)

**Code**: [`controller/src/`](../controller/src/)

1. Controller watches Instance CRD
2. Reads Configuration's brokerSpec
3. Deploys Pod:
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: udev-video-abc123-broker
   spec:
     containers:
     - name: broker
       image: udev-video-broker:latest
       env:
       - name: UDEV_DEVNODE_ABC123
         value: "/dev/video0"
       resources:
         requests:
           akri.sh/udev-video: "1"
         limits:
           akri.sh/udev-video: "1"
   ```

### Step 4: Broker Initialization

**Code**: [`samples/brokers/udev-video-broker/`](../samples/brokers/udev-video-broker/)

**main.rs**:
```rust
#[tokio::main]
async fn main() {
    // 1. Read device node from environment
    let devnode = get_video_devnode(&env_var_query);
    // Returns: "/dev/video0"

    // 2. Build and start camera capturer
    let camera_capturer = camera_capturer::build_and_start_camera_capturer(&devnode);

    // 3. Start gRPC service
    camera_service::serve(&devnode, camera_capturer).await;
}
```

**camera_capturer.rs**:
```rust
pub fn build_and_start_camera_capturer(devnode: &str) -> RsCamera {
    let mut camera = RsCamera::new(devnode).unwrap(); // Open /dev/video0

    // Get supported formats
    let formats = camera.formats().collect();
    let format = get_format(&env_query, formats); // "MJPG"

    // Get supported resolutions
    let resolutions = camera.resolutions(&format);
    let resolution = get_resolution(&env_query, resolutions); // (640, 480)

    // Get supported frame rates
    let intervals = camera.intervals(&format, resolution);
    let interval = get_interval(&env_query, intervals); // (1, 10) = 10 FPS

    // Start camera with config
    camera.start(&Config {
        interval: interval,
        resolution: resolution,
        format: format.as_bytes(),
        ..Default::default()
    }).unwrap();

    camera
}
```

**camera_service.rs**:
```rust
#[tonic::async_trait]
impl Camera for CameraService {
    async fn get_frame(&self, _request: Request<NotifyRequest>)
        -> Result<Response<NotifyResponse>, Status> {

        // Capture frame from camera (V4L2 ioctl)
        let frame = self.camera_capturer.capture().unwrap();

        // Return as gRPC response
        Ok(Response::new(NotifyResponse {
            frame: frame[..].to_vec(),  // JPEG bytes
            camera: self.devnode.clone(),
        }))
    }
}

pub async fn serve(devnode: &str, camera_capturer: RsCamera) {
    let service = CameraServer::new(CameraService {
        camera_capturer,
        devnode: devnode.to_string(),
    });

    let addr = "0.0.0.0:8083".parse().unwrap();
    Server::builder()
        .add_service(service)
        .serve(addr)
        .await
        .unwrap();
}
```

### Step 5: Streaming App Initialization

**Code**: [`samples/apps/video-streaming-app/app.py`](../samples/apps/video-streaming-app/app.py)

**Service Discovery**:
```python
def get_camera_display(configuration_name):
    config.load_incluster_config()
    api = client.CoreV1Api()

    camera_display = CameraDisplay()

    # List all services
    services = api.list_service_for_all_namespaces(watch=False)

    for svc in services.items:
        # Find configuration service (all cameras)
        if svc.metadata.name == configuration_name + "-svc":
            url = f"{svc.spec.cluster_ip}:8083"
            camera_display.main_camera = CameraFeed(url)

        # Find instance services (individual cameras)
        elif re.match(configuration_name + "-[\\da-f]{6}-svc", svc.metadata.name):
            url = f"{svc.spec.cluster_ip}:8083"
            camera_display.small_cameras.append(CameraFeed(url))

    return camera_display
```

**Frame Capture Thread**:
```python
class CameraFeed:
    def get_frames(self):
        while not self.stop_event.wait(0.01):
            try:
                # Create gRPC client
                channel = grpc.insecure_channel(self.url)
                stub = camera_pb2_grpc.CameraStub(channel)

                # Call GetFrame RPC
                response = stub.GetFrame(camera_pb2.NotifyRequest())
                frame = response.frame  # JPEG bytes

                channel.close()

                # Put frame in queue (size 1, replace old)
                if len(frame) > 0:
                    if self.queue.full():
                        self.queue.get(False)  # Remove old
                    self.queue.put(frame, False)

                sleep(1)  # Wait before next capture

            except Exception as e:
                logging.info(f"Exception: {e}")
                sleep(1)
```

**Flask Routes**:
```python
@app.route('/')
def index():
    return render_template('index.html',
                         camera_count=global_camera_display.count())

@app.route('/camera_frame_feed/<camera_id>')
def camera_frame_feed(camera_id=0):
    selected_camera = get_camera_by_id(camera_id)

    # Return MJPEG stream
    return Response(
        selected_camera.generator_func(),
        mimetype='multipart/x-mixed-replace; boundary=frame'
    )

def generator_func(self):
    while not self.stop_event.wait(0.01):
        frame = self.queue.get(True, None)  # Block until frame available

        # Yield MJPEG frame
        yield (b'--frame\r\n'
               b'Content-Type: image/jpeg\r\n\r\n' +
               frame +
               b'\r\n')
```

### Step 6: Browser Rendering

**HTML**:
```html
<img src="/camera_frame_feed/0" />
```

**Browser Behavior**:
1. Makes GET request to `/camera_frame_feed/0`
2. Receives `Content-Type: multipart/x-mixed-replace; boundary=frame`
3. Reads each part separated by `--frame`
4. Each part is a JPEG image
5. Browser automatically decodes and renders each JPEG
6. Continuously replaces image creating video effect

## Component Deep Dive

### Camera Capturer (rscam / V4L2)

**Technology**: rscam is a Rust wrapper for V4L2 (Video4Linux2)

**What happens when camera.capture() is called**:
```
1. rscam calls ioctl(VIDIOC_DQBUF)
   ├─> Kernel: Dequeue buffer from V4L2 driver
   └─> Returns buffer with frame data

2. Kernel reads from USB webcam
   ├─> USB bulk transfer
   └─> MJPEG compressed frame in kernel memory

3. rscam copies frame to userspace
   └─> Returns Vec<u8> with JPEG bytes

4. Frame ready for gRPC transmission
```

**Configuration Parameters**:
- **Format**: MJPG (Motion JPEG) - Each frame is a JPEG image
- **Resolution**: 640x480 (default) or from `RESOLUTION_WIDTH`/`HEIGHT` env vars
- **FPS**: 10 (default) or from `FRAMES_PER_SECOND` env var
- **Permissions**: rwm (read, write, mknod)

### gRPC Communication

**Proto Definition** ([`camera.proto`](../samples/apps/video-streaming-app/camera.proto)):
```protobuf
service Camera {
  rpc GetFrame (NotifyRequest) returns (NotifyResponse);
}

message NotifyRequest {}

message NotifyResponse {
  bytes frame = 1;      // JPEG bytes
  string camera = 2;    // Device path (/dev/video0)
}
```

**Wire Protocol**:
- Transport: HTTP/2
- Serialization: Protocol Buffers
- Frame size: 5-50 KB (depends on scene complexity)
- No streaming: Request-response per frame

### MJPEG Streaming

**Format**:
```http
HTTP/1.1 200 OK
Content-Type: multipart/x-mixed-replace; boundary=frame

--frame
Content-Type: image/jpeg

<JPEG binary data>
--frame
Content-Type: image/jpeg

<JPEG binary data>
--frame
...
```

**Why MJPEG?**:
- ✅ Native browser support (no JavaScript needed)
- ✅ Simple implementation
- ✅ Frame-by-frame independence
- ❌ Lower compression than H.264
- ❌ Higher bandwidth usage

## Data Flow Analysis

### Frame Journey Time

```
┌─────────────────────────────────────────────────────────────┐
│                    FRAME LATENCY BREAKDOWN                   │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Webcam capture           : ~33ms  (at 30fps capability)  │
│  2. USB transfer             : ~5ms                          │
│  3. V4L2/rscam processing    : ~10ms                         │
│  4. gRPC serialization       : ~5ms                          │
│  5. Network (same node)      : ~1ms                          │
│  6. gRPC deserialization     : ~5ms                          │
│  7. Queue wait               : ~0-1000ms (depends on timing) │
│  8. HTTP chunk write         : ~5ms                          │
│  9. Network to browser       : ~10-50ms                      │
│  10. Browser JPEG decode     : ~10ms                         │
│  11. Browser render          : ~16ms (60Hz display)          │
│                                                               │
│  TOTAL LATENCY: ~100ms - 1.2s                                │
│  (Lower end if queue has frame ready, higher if waiting)     │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

### Bandwidth Usage

```
Frame size: ~20 KB (MJPEG at 640x480)
FPS: 10 (as configured)
Bandwidth per camera: 20 KB * 10 fps = 200 KB/s = 1.6 Mbps

For multiple cameras:
- 1 camera:  1.6 Mbps
- 4 cameras: 6.4 Mbps
- 10 cameras: 16 Mbps
```

### Resource Usage

**Broker Pod**:
- CPU: ~10-50m (varies with FPS and resolution)
- Memory: ~50-100 MB
- Devices: 1 device file mount (`/dev/video0`)

**Streaming App**:
- CPU: ~100-200m (for 4 cameras)
- Memory: ~200-400 MB
- Network: 1.6 Mbps per camera

## Sequence Diagram

```
┌────────┐  ┌──────┐  ┌───────┐  ┌────────┐  ┌────────┐  ┌─────────┐  ┌─────────┐
│Webcam  │  │Kernel│  │udev DH│  │ Agent  │  │Ctrl    │  │ Broker  │  │  App    │
└───┬────┘  └───┬──┘  └───┬───┘  └───┬────┘  └───┬────┘  └────┬────┘  └────┬────┘
    │           │         │          │           │            │          │
    │USB plug   │         │          │           │            │          │
    ├──────────►│         │          │           │            │          │
    │           │         │          │           │            │          │
    │       udev event    │          │           │            │          │
    │           ├────────►│          │           │            │          │
    │           │         │          │           │            │          │
    │           │    Discover        │           │            │          │
    │           │         ├─────────►│           │            │          │
    │           │         │          │           │            │          │
    │           │         │  Device  │           │            │          │
    │           │         │◄─────────┤           │            │          │
    │           │         │/dev/video0           │            │          │
    │           │         │          │           │            │          │
    │           │         │    Create Instance   │            │          │
    │           │         │          ├──────────►│            │          │
    │           │         │          │           │            │          │
    │           │         │          │      Deploy Broker     │          │
    │           │         │          │           ├───────────►│          │
    │           │         │          │           │            │          │
    │           │         │          │           │    Start   │          │
    │           │         │          │           │            │          │
    │           │         │          │           │  Open /dev/video0     │
    │           │◄───────────────────────────────────────────┤          │
    │           │         │          │           │            │          │
    │           │         │          │           │  Start gRPC│          │
    │           │         │          │           │  :8083     │          │
    │           │         │          │           │◄───────────┤          │
    │           │         │          │           │            │          │
    │           │         │          │           │            │  Discover│
    │           │         │          │           │            │  services│
    │           │         │          │           │            │◄─────────┤
    │           │         │          │           │            │          │
    │           │         │          │           │     GetFrame() ───────►│
    │           │         │          │           │            │◄─────────┤
    │           │         │          │           │            │          │
    │           │         │          │           │  capture() │          │
    │           │◄───────────────────────────────────────────┤          │
    │           │ V4L2 ioctl        │           │            │          │
    │           │         │          │           │            │          │
    │  Frame    │         │          │           │            │          │
    ├──────────►│         │          │           │            │          │
    │           │         │          │           │            │          │
    │           │ Return frame[]    │           │            │          │
    │           ├────────────────────────────────────────────►│          │
    │           │         │          │           │            │          │
    │           │         │          │           │  Return    │          │
    │           │         │          │           │  NotifyResponse       │
    │           │         │          │           │            ├─────────►│
    │           │         │          │           │            │          │
    │           │         │          │           │            │  Queue   │
    │           │         │          │           │            │  frame   │
    │           │         │          │           │            │          │
    │           │         │          │           │            │  Browser │
    │           │         │          │           │            │  request │
    │           │         │          │           │            │◄─────────┤
    │           │         │          │           │            │  HTTP GET│
    │           │         │          │           │            │  /camera_│
    │           │         │          │           │            │  frame_  │
    │           │         │          │           │            │  feed/0  │
    │           │         │          │           │            │          │
    │           │         │          │           │            │  Stream  │
    │           │         │          │           │            │  MJPEG   │
    │           │         │          │           │            │  frame   │
    │           │         │          │           │            ├─────────►│
    │           │         │          │           │            │          │
    │           │         │          │           │            │          Browser
    │           │         │          │           │            │          renders
    │           │         │          │           │            │          video
    │           │         │          │           │            │          │
    └───┬───────┴─────────┴──────────┴───────────┴────────────┴──────────┴────┘
        │                                                                │
        │  Loop: Continuous frame capture, streaming, and rendering     │
        └───────────────────────────────────────────────────────────────┘
```

## Key Takeaways

1. **Device Plugin Pattern**: Akri uses Kubernetes device plugins to make physical devices schedulable resources
2. **Separation of Concerns**:
   - Discovery Handler: Finds devices
   - Agent: Manages device plugins and Instances
   - Broker: Accesses and serves device data
   - Application: Consumes device data
3. **gRPC for Internal Communication**: Broker-to-App uses gRPC for efficient binary transfer
4. **MJPEG for Browser**: HTTP multipart streaming for universal browser support
5. **V4L2 for Camera Access**: Standard Linux interface for video devices
6. **Dynamic Discovery**: Devices automatically discovered and brokers deployed

## Configuration Options

### Broker Configuration (Environment Variables)

| Variable | Default | Purpose |
|----------|---------|---------|
| `UDEV_DEVNODE_<hash>` | (required) | Device node path |
| `FORMAT` | `MJPG` | Image format (MJPG, YUYV, etc.) |
| `RESOLUTION_WIDTH` | `640` | Frame width in pixels |
| `RESOLUTION_HEIGHT` | `480` | Frame height in pixels |
| `FRAMES_PER_SECOND` | `10` | Capture frame rate |

### Streaming App Configuration

| Variable | Purpose |
|----------|---------|
| `CONFIGURATION_NAME` | Akri Configuration name to discover |
| `CAMERA_COUNT` | Number of cameras (manual mode) |
| `CAMERAS_SOURCE_SVC` | Main camera service URL |
| `CAMERA1_SOURCE_SVC` | Individual camera service URLs |

## Troubleshooting

### No Video Stream

1. **Check webcam is detected**:
   ```bash
   ls -la /dev/video*
   v4l2-ctl --list-devices
   ```

2. **Check Instance created**:
   ```bash
   kubectl get instances
   ```

3. **Check broker Pod running**:
   ```bash
   kubectl get pods -l akri.sh/configuration=udev-video
   kubectl logs <broker-pod>
   ```

4. **Check gRPC service**:
   ```bash
   kubectl get svc -l akri.sh/configuration=udev-video
   ```

5. **Check streaming app**:
   ```bash
   kubectl logs <streaming-app-pod>
   ```

### Poor Video Quality

- Increase resolution: Set `RESOLUTION_WIDTH`/`RESOLUTION_HEIGHT`
- Increase FPS: Set `FRAMES_PER_SECOND`
- Note: Higher settings = more CPU/bandwidth

### High Latency

- Reduce FPS in broker (less frequent captures)
- Optimize network between broker and app
- Use faster image format if camera supports

## References

- [V4L2 API Documentation](https://www.kernel.org/doc/html/latest/userspace-api/media/v4l/v4l2.html)
- [rscam Rust Library](https://crates.io/crates/rscam)
- [gRPC Protocol](https://grpc.io/docs/)
- [MJPEG Streaming](https://en.wikipedia.org/wiki/Motion_JPEG)
- [Akri USB Camera Demo](https://docs.akri.sh/demos/usb-camera-demo)
