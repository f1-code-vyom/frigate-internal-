# Frigate NVR Architecture Analysis

This document provides a comprehensive analysis of how Frigate NVR handles video streams, motion detection, recording, and scaling. This analysis serves as a foundation for building similar systems.

---

## Table of Contents
1. [FFmpeg and RTSP Stream Processing](#1-ffmpeg-and-rtsp-stream-processing)
2. [Image Snapshots and Video Recording](#2-image-snapshots-and-video-recording)
3. [Motion Detection and Frame Selection](#3-motion-detection-and-frame-selection)
4. [Video Storage Architecture](#4-video-storage-architecture)
5. [Scaling Architecture](#5-scaling-architecture)
6. [Key Takeaways for System Design](#6-key-takeaways-for-system-design)

---

## 1. FFmpeg and RTSP Stream Processing

### 1.1 FFmpeg Process Spawning

FFmpeg processes are spawned using Python's `subprocess.Popen()`:

**File:** `frigate/video.py:53-72`

```python
def start_or_restart_ffmpeg(ffmpeg_cmd, logger, logpipe, frame_size=None, ffmpeg_process=None):
    if ffmpeg_process is not None:
        stop_ffmpeg(ffmpeg_process, logger)

    if frame_size is None:
        process = sp.Popen(
            ffmpeg_cmd,
            stdout=sp.DEVNULL,
            stderr=logpipe,
            stdin=sp.DEVNULL,
            start_new_session=True,
        )
    else:
        process = sp.Popen(
            ffmpeg_cmd,
            stdout=sp.PIPE,          # Raw video output to pipe
            stderr=logpipe,
            stdin=sp.DEVNULL,
            bufsize=frame_size * 10,
            start_new_session=True,
        )
    return process
```

**Key Design Decisions:**
- **Separate session** (`start_new_session=True`): Isolates FFmpeg from parent process signals
- **Buffered stdout** for detect role: `bufsize=frame_size * 10` for 10 frames of buffer
- **Stderr logging**: Via LogPipe for debugging without blocking
- **Watchdog monitoring**: `CameraWatchdog` class monitors and auto-restarts crashed processes

### 1.2 RTSP Input Arguments

**File:** `frigate/ffmpeg_presets.py:366-428`

```python
# Default RTSP preset
"preset-rtsp-generic": [
    "-avoid_negative_ts", "make_zero",
    "-fflags", "+genpts+discardcorrupt",
    "-rtsp_transport", "tcp",           # TCP for reliability
    "-timeout", "10000000",             # 10 second timeout
    "-use_wallclock_as_timestamps", "1",
]
```

**Available RTSP Presets:**
| Preset | Use Case |
|--------|----------|
| `preset-rtsp-generic` | Default TCP transport with corruption handling |
| `preset-rtsp-udp` | UDP transport for lower latency |
| `preset-rtsp-restream` | For restreamed RTSP (no genpts/discardcorrupt) |
| `preset-rtsp-restream-low-latency` | Adds `low_delay` and `nobuffer` flags |

### 1.3 Multi-Stream Architecture (Main vs Sub Stream)

**File:** `frigate/config/camera/camera.py:186-273`

Frigate supports **separate streams per role**:

```python
class CameraInput(FrigateBaseModel):
    path: EnvString              # RTSP URL
    roles: list[CameraRoleEnum]  # detect, record, audio
    input_args: Union[str, list[str]]
    hwaccel_args: Union[str, list[str]]
```

**Role System:**
- **detect**: Lower resolution sub-stream for object detection (raw YUV420p output)
- **record**: Higher resolution main stream for recordings (codec copy)
- **audio**: Audio extraction stream

**Example Configuration:**
```yaml
cameras:
  front_door:
    ffmpeg:
      inputs:
        - path: rtsp://camera/stream/sub   # Sub stream
          roles: [detect]
        - path: rtsp://camera/stream/main  # Main stream
          roles: [record]
```

**Each input spawns a SEPARATE FFmpeg process**, allowing:
- Different quality streams for detection vs recording
- Independent restart/recovery per stream
- Optimized encoding settings per use case

### 1.4 Hardware Acceleration

**File:** `frigate/ffmpeg_presets.py:84-117`

Supported hardware acceleration:

| Preset | Decode Args | Platform |
|--------|-------------|----------|
| `preset-vaapi` | `-hwaccel vaapi -hwaccel_output_format vaapi` | Intel/AMD GPUs |
| `preset-nvidia` | `-hwaccel cuda -hwaccel_output_format cuda` | NVIDIA GPUs |
| `preset-intel-qsv-h264` | `-hwaccel qsv -c:v h264_qsv` | Intel QuickSync |
| `preset-jetson-h264` | `-c:v h264_nvmpi -resize {w}x{h}` | NVIDIA Jetson |
| `preset-rpi-64-h264` | `-c:v h264_v4l2m2m` | Raspberry Pi |
| `preset-rkmpp` | `-hwaccel rkmpp -hwaccel_output_format drm_prime` | Rockchip |

### 1.5 Output Consumption

**For Detection (Raw Frames via Pipe):**

```python
# frigate/ffmpeg_presets.py
DETECT_FFMPEG_OUTPUT_ARGS_DEFAULT = [
    "-threads", "2",
    "-f", "rawvideo",        # Raw video format
    "-pix_fmt", "yuv420p",   # YUV 4:2:0 pixel format
]
```

**Frame Capture Flow:**
```
FFmpeg stdout (pipe)
    ↓
CameraCaptureRunner.read_stdout()  # frigate/video.py:100-166
    ↓
SharedMemoryFrameManager.write()   # frigate/util/image.py:838-908
    ↓
Named shared memory buffer (e.g., "front_door_frame0")
    ↓
Available to detection/tracking processes
```

**For Recording (MP4 Segments):**
```python
record_args = [
    "-f", "segment",
    "-segment_time", "10",           # 10-second segments
    "-segment_format", "mp4",
    "-reset_timestamps", "1",
    "-strftime", "1",
    "-c", "copy",                    # No re-encoding
]
# Output: /tmp/cache/{camera}@%Y%m%d%H%M%S%z.mp4
```

---

## 2. Image Snapshots and Video Recording

### 2.1 Snapshot Creation

Snapshots are **NOT created by FFmpeg**. They are extracted from in-memory frames using OpenCV:

**File:** `frigate/track/tracked_object.py:565-618`

```python
def write_snapshot_to_disk(self) -> None:
    snapshot_config = self.camera_config.snapshots

    # Get best frame from in-memory cache
    jpg_bytes = self.get_img_bytes(
        ext="jpg",
        timestamp=snapshot_config.timestamp,
        bounding_box=snapshot_config.bounding_box,
        crop=snapshot_config.crop,
        height=snapshot_config.height,
        quality=snapshot_config.quality,
    )

    # Write JPEG with overlays
    with open(f"{CLIPS_DIR}/{camera}-{object_id}.jpg", "wb") as f:
        f.write(jpg_bytes)
```

**Snapshot Flow:**
```
1. Frame captured from FFmpeg into shared memory (YUV420p)
2. Object detected, TrackedObject created
3. Best frame selected (highest confidence, largest size)
4. Frame converted: cv2.cvtColor(frame, cv2.COLOR_YUV2BGR_I420)
5. Overlays rendered: bounding box, timestamp (via OpenCV)
6. Encoded: cv2.imencode(".jpg", frame, [IMWRITE_JPEG_QUALITY, 70])
7. Saved to /media/frigate/clips/{camera}-{id}.jpg
```

**Two formats saved:**
- `{camera}-{id}.jpg` - With bounding box overlays
- `{camera}-{id}-clean.webp` - Clean copy without overlays

### 2.2 Runtime Snapshot Extraction (From Recordings)

**File:** `frigate/util/image.py:946-996`

For extracting frames from recorded video on-demand:

```python
def run_ffmpeg_snapshot(ffmpeg, input_path, codec, seek_time=None, height=None):
    ffmpeg_cmd = [
        ffmpeg.ffmpeg_path,
        "-hide_banner", "-loglevel", "warning",
    ]
    if seek_time is not None:
        ffmpeg_cmd.extend(["-ss", f"00:00:{seek_time}"])

    ffmpeg_cmd.extend([
        "-i", input_path,
        "-frames:v", "1",
        "-c:v", codec,        # "mjpeg" or "png"
        "-f", "image2pipe",
        "-",
    ])

    process = sp.run(ffmpeg_cmd, capture_output=True, timeout=timeout)
    return process.stdout  # JPEG bytes
```

### 2.3 Video Recording Creation

**File:** `frigate/ffmpeg_presets.py:445-556`

FFmpeg's **segment muxer** creates continuous recordings:

```python
"preset-record-generic": [
    "-f", "segment",              # Segment muxer
    "-segment_time", "10",        # 10-second segments
    "-segment_format", "mp4",     # MP4 container
    "-reset_timestamps", "1",     # Reset per segment
    "-strftime", "1",             # Filename timestamp
    "-c", "copy",                 # No re-encoding
    "-an",                        # No audio (optional)
]
```

**Recording Output Path:**
```python
# frigate/config/camera/camera.py:218-222
output_path = f"{CACHE_DIR}/{camera_name}@{CACHE_SEGMENT_FORMAT}.mp4"
# Example: /tmp/cache/front_door@20251129143000+0000.mp4
```

### 2.4 Recording Types

| Type | Trigger | Retention |
|------|---------|-----------|
| **Continuous** | Always | `continuous.days` setting |
| **Motion** | Motion detected | `motion.days` setting |
| **Event-based** | Object detected (alert/detection) | `alerts.retain.days` / `detections.retain.days` |

**Configuration Example:**
```yaml
record:
  enabled: true
  continuous:
    days: 7
  detections:
    pre_capture: 5    # 5 seconds before
    post_capture: 5   # 5 seconds after
    retain:
      days: 10
      mode: motion    # Only keep if motion detected
```

---

## 3. Motion Detection and Frame Selection

### 3.1 Motion Detection Algorithm

**File:** `frigate/motion/improved_motion.py`

Frigate uses **exponential weighted moving average (EWMA) background subtraction**:

```python
class ImprovedMotionDetector:
    def detect(self, frame):
        # 1. Resize to small size (default 100px height)
        resized = cv2.resize(frame, (frame_width, frame_height))

        # 2. Improve contrast (percentile clipping 4th-96th)
        if self.improve_contrast:
            frame = self.contrast_improve(frame)

        # 3. Apply Gaussian blur
        frame = cv2.GaussianBlur(frame, (0,0), sigma)

        # 4. Compute absolute difference from background
        frame_delta = cv2.absdiff(frame, self.avg_frame.astype(np.uint8))

        # 5. Binary threshold
        thresh = cv2.threshold(frame_delta, threshold, 255, cv2.THRESH_BINARY)[1]

        # 6. Dilate to fill gaps
        thresh = cv2.dilate(thresh, None, iterations=1)

        # 7. Find contours
        contours = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

        # 8. Filter by area and create motion boxes
        motion_boxes = []
        for contour in contours:
            if cv2.contourArea(contour) > self.config.contour_area:
                motion_boxes.append(cv2.boundingRect(contour))

        # 9. Update background model
        cv2.accumulateWeighted(resized, self.avg_frame, self.config.frame_alpha)

        return motion_boxes
```

### 3.2 Motion Configuration Parameters

**File:** `frigate/config/camera/motion.py`

| Parameter | Default | Purpose |
|-----------|---------|---------|
| `threshold` | 30 | Pixel intensity change threshold (1-255) |
| `contour_area` | 10 | Minimum motion area in resized frame |
| `delta_alpha` | 0.2 | Weight for motion delta accumulation |
| `frame_alpha` | 0.01 | Background model update rate |
| `frame_height` | 100 | Frame height for motion detection |
| `lightning_threshold` | 0.8 | Recalibrate if >80% frame changes |
| `improve_contrast` | True | Apply contrast enhancement |

### 3.3 Motion → Object Detection Flow (Key Decision Point)

**File:** `frigate/video.py:815-952`

This is **how Frigate decides whether to process a frame**:

```python
# Step 1: Always run motion detection
motion_boxes = motion_detector.detect(frame)

# Step 2: Get regions around existing tracked objects
object_regions = get_cluster_region(tracked_objects)

# Step 3: Find NEW motion not covered by tracked objects
standalone_motion = [b for b in motion_boxes if not inside_any(b, object_regions)]

# Step 4: Cluster standalone motion into detection regions
if standalone_motion:
    motion_regions = cluster_and_create_regions(standalone_motion)
    regions = object_regions + motion_regions
else:
    regions = object_regions

# Step 5: CRITICAL DECISION - Skip detection if no regions
if len(regions) > 0:
    # Run object detection ONLY on motion regions
    for region in regions:
        detections.extend(detect(object_detector, frame, region))
else:
    # NO MOTION + NO TRACKED OBJECTS = SKIP FRAME
    object_tracker.update_frame_times(frame_time)  # Just update timestamps
```

### 3.4 Frame Skip Logic (Parking Lot at Night Example)

**Scenario:** Parking lot at night, no car movement

**What happens:**
1. Motion detector computes frame difference: **zero or near-zero motion**
2. `motion_boxes = []` (empty)
3. No existing tracked objects → `object_regions = []`
4. `regions = []` (empty)
5. **Object detection is SKIPPED entirely**
6. Frame is NOT saved (unless continuous recording enabled)

**The system automatically:**
- Skips expensive object detection on static frames
- Maintains background model updates
- Monitors for any changes (headlights, movement, etc.)

### 3.5 Lightning/Flash Handling

```python
# frigate/motion/improved_motion.py:180-185
if pct_motion > self.config.lightning_threshold:  # Default: 80%
    # Too much motion = probably lightning/IR flash
    self.calibrating = True  # Reset background model
    return []  # Don't report this as motion
```

### 3.6 Stationary Object Optimization

**File:** `frigate/config/camera/detect.py`

Objects that stop moving are handled differently:

```yaml
detect:
  stationary:
    interval: 50        # Check every 50 frames (not every frame)
    threshold: 50       # Frames without movement before marked stationary
    max_frames: 0       # Max frames to track (0 = unlimited)
```

---

## 4. Video Storage Architecture

### 4.1 Directory Structure

```
/media/frigate/                    # BASE_DIR
├── recordings/                    # RECORD_DIR
│   └── {YYYY-MM-DD}/
│       └── {HH}/
│           └── {camera_name}/
│               └── {MM.SS}.mp4    # Segment file
├── clips/                         # CLIPS_DIR
│   ├── {camera}-{id}.jpg          # Event snapshots
│   ├── {camera}-{id}-clean.webp   # Clean copies
│   ├── thumbs/                    # Thumbnails
│   └── previews/                  # Hour-long preview clips
└── exports/                       # User exports

/tmp/cache/                        # CACHE_DIR (temporary)
└── {camera}@{timestamp}.mp4       # Live segments
```

### 4.2 Three-Stage Recording Pipeline

**File:** `frigate/record/maintainer.py`

```
Stage 1: Live Cache
    FFmpeg writes → /tmp/cache/{camera}@{timestamp}.mp4
    Max 6 segments in cache (MAX_SEGMENTS_IN_CACHE)

    ↓ (RecordingMaintainer runs every 5 seconds)

Stage 2: Validation & Decision
    - Validate video (ffprobe)
    - Calculate metrics (motion_count, object_count, dBFS)
    - Check retention policy
    - Decision: KEEP or DISCARD

    ↓ (if KEEP)

Stage 3: Permanent Storage
    ffmpeg -i {cache} -c copy -movflags +faststart {final}
    → /media/frigate/recordings/{date}/{hour}/{camera}/{MM.SS}.mp4
    → Insert metadata into Recordings table
```

### 4.3 Database Schema

**File:** `frigate/models.py`

```python
class Recordings(Model):
    id = CharField(primary_key=True)       # "{timestamp}-{rand_id}"
    camera = CharField(index=True)
    path = CharField(unique=True)

    start_time = DateTimeField()
    end_time = DateTimeField()
    duration = FloatField()

    # Metrics for retention decisions
    motion = IntegerField()                # Motion frame count
    objects = IntegerField()               # Object count
    dBFS = IntegerField()                  # Audio level
    segment_size = FloatField()            # Size in MB
```

### 4.4 Retention Logic

**File:** `frigate/record/maintainer.py:59-78`

```python
def should_discard_segment(self, retain_mode: RetainModeEnum) -> bool:
    # Mode: all - keep everything
    if retain_mode == RetainModeEnum.all:
        return False

    # Mode: motion - keep if motion or audio detected
    if retain_mode == RetainModeEnum.motion:
        if self.motion_count > 0 or self.average_dBFS > 0:
            return False

    # Mode: active_objects - keep only if objects detected
    if self.active_object_count > 0:
        return False

    return True  # Discard
```

### 4.5 Cleanup Process

**File:** `frigate/record/cleanup.py`

Runs every `expire_interval` minutes (default: 60):

```python
def expire_recordings():
    for camera in cameras:
        # Calculate expiration dates
        continuous_expire = now - timedelta(days=continuous.days)
        motion_expire = now - timedelta(days=motion.days)

        for recording in expired_recordings:
            # Check if overlaps with any event (with pre/post capture)
            overlaps_event = check_review_segments(recording, events)

            if not overlaps_event and recording.end_time < expire_date:
                delete_file_and_db_entry(recording)
```

**Storage pressure cleanup** (`frigate/storage.py`):
- Runs every 5 minutes
- If remaining space < 1 hour of bandwidth → delete oldest segments
- Priority: non-retained segments first, then retained

---

## 5. Scaling Architecture

### 5.1 Process Architecture

**Frigate uses multi-process architecture with 2 processes per camera:**

```
FrigateApp (Main Process)
├── CameraMaintainer (Thread) - Manages all cameras
│
├── Camera "front_door"
│   ├── CameraCapture (Process) - FFmpeg frame capture
│   └── CameraTracker (Process) - Motion detection, detection requests
│
├── Camera "backyard"
│   ├── CameraCapture (Process)
│   └── CameraTracker (Process)
│
├── ObjectDetectProcess (Process) - SHARED detector
│   └── DetectorRunner - Handles ALL cameras
│
├── RecordProcess (Process)
├── ReviewProcess (Process)
├── EventProcessor (Process)
├── AudioProcessor (Process)
└── StorageMaintainer (Process)
```

**File:** `frigate/camera/maintainer.py:129-142`

```python
# Per-camera processes
camera_process = CameraTracker(
    config,
    self.detection_queue,      # Shared queue to detector
    self.detected_frames_queue,
    self.camera_metrics[name],
    ...
)
camera_process.start()
```

### 5.2 Shared Detection Architecture

**Critical Design:** ONE detector process serves ALL cameras

**File:** `frigate/app.py:340-377`

```python
def start_detectors(self):
    # Shared memory for ALL cameras
    for camera_name in self.config.cameras.keys():
        shm_in = UntrackedSharedMemory(name=camera_name, size=largest_frame)
        shm_out = UntrackedSharedMemory(name=f"out-{camera_name}", size=output_size)

    # ONE detector process
    for name, detector_config in self.config.detectors.items():
        self.detectors[name] = ObjectDetectProcess(
            name,
            self.detection_queue,           # Shared queue
            list(self.config.cameras.keys()), # ALL cameras
            detector_config,
        )
```

**Detection Flow:**
```
Camera 1     Camera 2     Camera 3
    ↓            ↓            ↓
RemoteObjectDetector (via shared queue)
    └────────────┴────────────┘
                 ↓
        detection_queue (mp.Queue)
                 ↓
        ObjectDetectProcess (ONE)
                 ↓
        ZMQ Pub/Sub (results)
                 ↓
    ┌────────────┴────────────┐
    ↓            ↓            ↓
Camera 1     Camera 2     Camera 3
```

### 5.3 Scaling Strategies

**For 16 cameras:**
- 32 camera processes (2 per camera)
- 1 detector process
- Uses multi-threading in detector for batch processing
- Shared memory prevents data copying

**For 200 cameras:**
- 400 camera processes
- Multiple detector configurations supported:
  ```yaml
  detectors:
    coral1:
      type: coral
      device: /dev/bus/usb/001/002
    coral2:
      type: coral
      device: /dev/bus/usb/001/003
    gpu:
      type: tensorrt
      device: 0
  ```
- Each detector is still ONE process, but multiple detectors share load

### 5.4 Resource Management

**Shared Memory:**
```python
# frigate/camera/maintainer.py:74-93
def calculate_shm_requirements():
    frame_size = width * height * 1.5  # YUV420p
    total_shm = get_available_shm()
    frames_per_camera = total_shm / (num_cameras * frame_size)
```

**Queue Management:**
```python
# frigate/app.py:148-160
detection_queue = mp.Queue()
detected_frames_queue = mp.Queue(
    maxsize=(enabled_camera_count + 2) * 2
)
```

**IPC Communication:**
- **ZMQ Pub/Sub**: Low-latency detection signaling (`ipc:///tmp/cache/detector_pub`)
- **Multiprocessing Queues**: Detection requests and results
- **Shared Memory**: Frame buffers (zero-copy between processes)

### 5.5 Process Spawn Optimization

**File:** `frigate/__main__.py:135`

```python
mp.set_start_method("forkserver", force=True)
mp.set_forkserver_preload([
    "sqlite3", "numpy", "cv2", "peewee", "zmq",
    "frigate.camera.maintainer",
])
```

Benefits:
- **forkserver**: Clean process spawning, avoids COW memory issues
- **preload**: Heavy libraries loaded once, inherited by all processes
- Reduces memory footprint with many cameras

---

## 6. Key Takeaways for System Design

### 6.1 FFmpeg Best Practices

1. **Use TCP for RTSP**: More reliable than UDP
2. **Separate streams by role**: Sub-stream for detection, main for recording
3. **Hardware acceleration**: Essential for scaling
4. **Segment muxer**: Creates atomic recording segments
5. **faststart flag**: Add when moving to permanent storage

### 6.2 Frame Processing Optimization

1. **Motion-first filtering**: Skip object detection on static frames
2. **Region-based detection**: Only process areas with motion
3. **Background subtraction**: Exponential weighted average is efficient
4. **Downsampled motion detection**: 100px height is sufficient
5. **Stationary object caching**: Don't re-detect objects that haven't moved

### 6.3 Storage Design

1. **Three-stage pipeline**: Cache → Validate → Permanent
2. **Segment-based storage**: Small atomic files (10 minutes)
3. **Metadata in database**: Enables flexible retention queries
4. **Date/hour/camera hierarchy**: Efficient cleanup by time
5. **Storage pressure handling**: Auto-cleanup when space low

### 6.4 Scaling Architecture

1. **Process-per-camera**: Isolation, crash recovery
2. **Shared detector**: Efficient hardware utilization
3. **Shared memory for frames**: Zero-copy between processes
4. **Queue-based detection**: Decouples cameras from detector
5. **forkserver + preload**: Efficient multi-process spawning

### 6.5 For Your API-Based System

Based on this analysis, for a system that applies an API to images:

```
Recommended Architecture:

1. Per-Camera Process
   - FFmpeg → Shared Memory
   - Motion Detection (local)
   - Decision: Send to API? (based on motion)

2. API Request Queue (Shared)
   - Rate limiting
   - Batching
   - Priority (motion score)

3. API Worker Process(es)
   - Fetch frame from shared memory
   - Call external API
   - Return results via ZMQ

4. Result Processing
   - Store metadata in database
   - Trigger recording/snapshots
```

**Key decisions to make:**
- Motion threshold for API calls
- Batch size for API requests
- Rate limiting strategy
- Result caching for repeated detections
