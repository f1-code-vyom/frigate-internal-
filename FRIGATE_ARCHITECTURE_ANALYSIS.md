# Frigate NVR Architecture Analysis

**A comprehensive technical reference for understanding Frigate's video processing pipeline**

This document provides detailed analysis of how Frigate NVR handles video streams, FFmpeg commands, motion detection, recording, and scaling. Designed to serve as a foundation for building similar systems.

---

## Table of Contents
1. [FFmpeg Commands Reference](#1-ffmpeg-commands-reference)
2. [Hardware Acceleration Presets](#2-hardware-acceleration-presets)
3. [Python Process Management](#3-python-process-management)
4. [Image Snapshots and Video Recording](#4-image-snapshots-and-video-recording)
5. [Motion Detection and Frame Selection](#5-motion-detection-and-frame-selection)
6. [Video Storage Architecture](#6-video-storage-architecture)
7. [Scaling Architecture](#7-scaling-architecture)
8. [Key Takeaways for System Design](#8-key-takeaways-for-system-design)

---

## 1. FFmpeg Commands Reference

### 1.1 Complete FFmpeg Command Structure

Frigate builds FFmpeg commands in a specific order. Here's the complete structure:

```
[ffmpeg_path] + [global_args] + [hwaccel_args] + [input_args] + [-i input_path] + [output_args]
```

**Source File:** `frigate/config/camera/camera.py:264-273`

```python
cmd = (
    [self.ffmpeg.ffmpeg_path]           # /usr/lib/ffmpeg/*/bin/ffmpeg
    + global_args                        # Global FFmpeg settings
    + (hwaccel_args if "detect" in roles else [])  # HW accel for detect only
    + input_args                         # Input preset args
    + ["-i", escape_special_characters(ffmpeg_input.path)]  # Input source
    + ffmpeg_output_args                 # Output configuration
)
```

### 1.2 Global Arguments

**Source File:** `frigate/ffmpeg_presets.py`

```bash
# Default global arguments applied to ALL FFmpeg commands
-hide_banner          # Suppress FFmpeg banner
-loglevel warning     # Only show warnings and errors
-threads 2            # Limit thread count for stability
```

### 1.3 Input Presets (RTSP/HTTP/RTMP)

**Source File:** `frigate/ffmpeg_presets.py:295-428`

#### RTSP Generic (Default - Most Common)
```bash
-user_agent "FFmpeg Frigate/0.x"
-avoid_negative_ts make_zero      # Handle negative timestamps
-fflags +genpts+discardcorrupt    # Generate PTS, discard corrupt frames
-rtsp_transport tcp               # Use TCP (more reliable than UDP)
-timeout 10000000                 # 10 second timeout (microseconds)
-use_wallclock_as_timestamps 1    # Use system clock for timestamps
```

#### RTSP UDP (Lower Latency)
```bash
-user_agent "FFmpeg Frigate/0.x"
-avoid_negative_ts make_zero
-fflags +genpts+discardcorrupt
-rtsp_transport udp               # UDP for lower latency
-timeout 10000000
-use_wallclock_as_timestamps 1
```

#### RTSP Restream (For Go2RTC/Frigate Restreams)
```bash
-user_agent "FFmpeg Frigate/0.x"
-rtsp_transport tcp
-timeout 10000000
# Note: No genpts/discardcorrupt - already processed
```

#### RTSP Restream Low Latency
```bash
-user_agent "FFmpeg Frigate/0.x"
-rtsp_transport tcp
-timeout 10000000
-fflags nobuffer                  # Disable buffering
-flags low_delay                  # Minimize latency
```

#### RTSP Blue Iris
```bash
-user_agent "FFmpeg Frigate/0.x"
-avoid_negative_ts make_zero
-flags low_delay
-strict experimental
-fflags +genpts+discardcorrupt
-rtsp_transport tcp
-timeout 10000000
-use_wallclock_as_timestamps 1
```

#### HTTP MJPEG Generic
```bash
-user_agent "FFmpeg Frigate/0.x"
-avoid_negative_ts make_zero
-fflags nobuffer
-flags low_delay
-strict experimental
-fflags +genpts+discardcorrupt
-use_wallclock_as_timestamps 1
```

#### HTTP JPEG (Single Frame Loop)
```bash
-r {fps}                          # Frame rate
-stream_loop -1                   # Loop indefinitely
-f image2                         # Image input format
-avoid_negative_ts make_zero
-fflags nobuffer
-flags low_delay
-strict experimental
-fflags +genpts+discardcorrupt
-use_wallclock_as_timestamps 1
```

#### HTTP Reolink
```bash
-user_agent "FFmpeg Frigate/0.x"
-avoid_negative_ts make_zero
-fflags +genpts+discardcorrupt
-flags low_delay
-strict experimental
-analyzeduration 1000M            # Extended analysis duration
-probesize 1000M                  # Extended probe size
-rw_timeout 10000000              # Read/write timeout
```

#### RTMP Generic
```bash
-avoid_negative_ts make_zero
-fflags nobuffer
-flags low_delay
-strict experimental
-fflags +genpts+discardcorrupt
-rw_timeout 10000000
-use_wallclock_as_timestamps 1
-f live_flv                       # Live FLV format
```

### 1.4 Output Arguments for Detection

**Source File:** `frigate/ffmpeg_presets.py` and config

**Detect Output (Raw Video to Pipe):**
```bash
-threads 2
-f rawvideo                       # Raw video format (uncompressed)
-pix_fmt yuv420p                  # YUV 4:2:0 pixel format
pipe:                             # Output to stdout pipe
```

**Scaling (Software - Default):**
```bash
-r {fps}                          # Target frame rate
-vf fps={fps},scale={width}:{height}   # FPS filter + scaling
```

### 1.5 Output Arguments for Recording

**Source File:** `frigate/ffmpeg_presets.py:445-539`

#### Generic Recording (No Audio)
```bash
-f segment                        # FFmpeg segment muxer
-segment_time 10                  # 10-second segments
-segment_format mp4               # MP4 container
-reset_timestamps 1               # Reset timestamps each segment
-strftime 1                       # Enable strftime for filenames
-c copy                           # Copy codec (no re-encoding)
-an                               # No audio
# Output: /tmp/cache/{camera}@%Y%m%d%H%M%S%z.mp4
```

#### Recording with AAC Audio
```bash
-f segment
-segment_time 10
-segment_format mp4
-reset_timestamps 1
-strftime 1
-c:v copy                         # Copy video codec
-c:a aac                          # Encode audio to AAC
```

#### Recording with Audio Copy
```bash
-f segment
-segment_time 10
-segment_format mp4
-reset_timestamps 1
-strftime 1
-c copy                           # Copy both video and audio
```

#### MJPEG Recording (Re-encode Required)
```bash
-f segment
-segment_time 10
-segment_format mp4
-reset_timestamps 1
-strftime 1
-c:v libx264                      # Re-encode MJPEG to H.264
-an
```

#### Ubiquiti Recording
```bash
-f segment
-segment_time 10
-segment_format mp4
-reset_timestamps 1
-strftime 1
-c:v copy
-ar 44100                         # Resample audio to 44.1kHz
-c:a aac
```

### 1.6 Complete Example Commands

**Detection Stream (NVIDIA GPU):**
```bash
/usr/lib/ffmpeg/7.1/bin/ffmpeg \
  -hide_banner -loglevel warning -threads 2 \
  -hwaccel_device 0 -hwaccel cuda -hwaccel_output_format cuda \
  -user_agent "FFmpeg Frigate/0.15" \
  -avoid_negative_ts make_zero \
  -fflags +genpts+discardcorrupt \
  -rtsp_transport tcp \
  -timeout 10000000 \
  -use_wallclock_as_timestamps 1 \
  -i rtsp://user:pass@192.168.1.100:554/stream1 \
  -r 5 -vf fps=5,scale_cuda=w=1280:h=720,hwdownload,format=nv12 \
  -threads 2 -f rawvideo -pix_fmt yuv420p \
  pipe:
```

**Recording Stream (Copy Codec):**
```bash
/usr/lib/ffmpeg/7.1/bin/ffmpeg \
  -hide_banner -loglevel warning -threads 2 \
  -user_agent "FFmpeg Frigate/0.15" \
  -avoid_negative_ts make_zero \
  -fflags +genpts+discardcorrupt \
  -rtsp_transport tcp \
  -timeout 10000000 \
  -use_wallclock_as_timestamps 1 \
  -i rtsp://user:pass@192.168.1.100:554/stream0 \
  -f segment -segment_time 10 -segment_format mp4 \
  -reset_timestamps 1 -strftime 1 \
  -c copy -an \
  /tmp/cache/front_door@%Y%m%d%H%M%S%z.mp4
```

**Snapshot Extraction (From Recording):**
```bash
/usr/lib/ffmpeg/7.1/bin/ffmpeg \
  -hide_banner -loglevel warning \
  -ss 00:00:5.5 \                    # Seek to timestamp
  -i /path/to/recording.mp4 \
  -frames:v 1 \                      # Extract single frame
  -c:v mjpeg \                       # JPEG codec
  -f image2pipe \                    # Output to pipe
  -vf scale=-1:180 \                 # Scale to 180px height
  -
```

---

## 2. Hardware Acceleration Presets

### 2.1 Decode Presets (Input Processing)

**Source File:** `frigate/ffmpeg_presets.py:84-117`

| Preset | Arguments | Platform |
|--------|-----------|----------|
| **preset-vaapi** | `-hwaccel_flags allow_profile_mismatch -hwaccel vaapi -hwaccel_device {gpu_device} -hwaccel_output_format vaapi` | Intel/AMD |
| **preset-nvidia** | `-hwaccel_device {gpu_index} -hwaccel cuda -hwaccel_output_format cuda` | NVIDIA |
| **preset-nvidia-h264** | Same as preset-nvidia | NVIDIA (H.264) |
| **preset-nvidia-h265** | Same as preset-nvidia | NVIDIA (H.265) |
| **preset-nvidia-mjpeg** | Same as preset-nvidia | NVIDIA (MJPEG) |
| **preset-intel-qsv-h264** | `-hwaccel qsv -qsv_device {gpu_device} -hwaccel_output_format qsv -c:v h264_qsv` | Intel QSV |
| **preset-intel-qsv-h265** | `-load_plugin hevc_hw -hwaccel qsv -qsv_device {gpu_device} -hwaccel_output_format qsv` | Intel QSV |
| **preset-jetson-h264** | `-c:v h264_nvmpi -resize {width}x{height}` | NVIDIA Jetson |
| **preset-jetson-h265** | `-c:v hevc_nvmpi -resize {width}x{height}` | NVIDIA Jetson |
| **preset-rpi-64-h264** | `-c:v:1 h264_v4l2m2m` | Raspberry Pi 64-bit |
| **preset-rpi-64-h265** | `-c:v:1 hevc_v4l2m2m` | Raspberry Pi 64-bit |
| **preset-rkmpp** | `-hwaccel rkmpp -hwaccel_output_format drm_prime` | Rockchip |
| **preset-rk-h264** | Same as preset-rkmpp | Rockchip (H.264) |
| **preset-rk-h265** | Same as preset-rkmpp | Rockchip (H.265) |
| **preset-vulkan** | `-hwaccel vulkan -init_hw_device vulkan=gpu:0 -filter_hw_device gpu -hwaccel_output_format vulkan` | Vulkan (experimental) |
| **preset-amd-amf** | `-hwaccel amf -init_hw_device amf=gpu:0 -filter_hw_device gpu -hwaccel_output_format amf` | AMD AMF (experimental) |

### 2.2 Scale Presets (Frame Scaling)

**Source File:** `frigate/ffmpeg_presets.py:120-134`

| Preset | Arguments | Notes |
|--------|-----------|-------|
| **default** | `-r {fps} -vf fps={fps},scale={w}:{h}` | Software scaling |
| **preset-vaapi** | `-r {fps} -vf fps={fps},scale_vaapi=w={w}:h={h},hwdownload,format=nv12,eq=gamma=1.4:gamma_weight=0.5` | Hardware scale + gamma |
| **preset-nvidia** | `-r {fps} -vf fps={fps},scale_cuda=w={w}:h={h},hwdownload,format=nv12,eq=gamma=1.4:gamma_weight=0.5` | CUDA scaling + gamma |
| **preset-intel-qsv-h264** | `-r {fps} -vf vpp_qsv=framerate={fps}:w={w}:h={h}:format=nv12,hwdownload,format=nv12,format=yuv420p` | QSV VPP |
| **preset-intel-qsv-h265** | Same as above | QSV VPP |
| **preset-jetson-h264** | `-r {fps}` | Scaling done in decoder |
| **preset-jetson-h265** | `-r {fps}` | Scaling done in decoder |
| **preset-rkmpp** | `-r {fps} -vf scale_rkrga=w={w}:h={h}:format=yuv420p:force_original_aspect_ratio=0,hwmap=mode=read,format=yuv420p` | RGA scaling |
| **preset-vulkan** | `-r {fps} -vf fps={fps},hwupload,scale_vulkan=w={w}:h={h},hwdownload` | Vulkan scaling |
| **preset-amd-amf** | `-r {fps} -vf fps={fps},hwupload,scale_amf=w={w}:h={h},hwdownload` | AMF scaling |

### 2.3 Encode Presets (Birdseye/Timelapse)

**Source File:** `frigate/ffmpeg_presets.py:150-205`

| Preset | Encoder | Arguments |
|--------|---------|-----------|
| **default** | libx264 | `-c:v libx264 -g 50 -profile:v high -level:v 4.1 -preset:v superfast -tune:v zerolatency` |
| **preset-vaapi** | h264_vaapi | `-c:v h264_vaapi -g 50 -bf 0 -profile:v high -level:v 4.1 -sei:v 0 -an -vf format=vaapi|nv12,hwupload` |
| **preset-nvidia** | h264_nvenc | `-c:v h264_nvenc -g 50 -profile:v high -level:v auto -preset:v p2 -tune:v ll` |
| **preset-intel-qsv-h264** | h264_qsv | `-c:v h264_qsv -g 50 -bf 0 -profile:v high -level:v 4.1 -async_depth:v 1` |
| **preset-jetson-h264** | h264_nvmpi | `-c:v h264_nvmpi -profile high` |
| **preset-rkmpp** | h264_rkmpp | `-c:v h264_rkmpp -profile:v high` |
| **preset-amd-amf** | h264_amf | `-c:v h264_amf -g 50 -profile:v high` |

### 2.4 GPU Device Selection

**Source File:** `frigate/ffmpeg_presets.py:23-70`

```python
class LibvaGpuSelector:
    """Automatically selects the correct GPU device path."""

    def get_gpu_arg(self, preset: str, gpu: int) -> str:
        # NVIDIA uses simple GPU index
        if "nvidia" in preset:
            return str(gpu)  # Returns "0", "1", etc.

        # VAAPI/others use DRI device paths
        # Maps gpu index to /dev/dri/renderD128, /dev/dri/renderD129, etc.
        valid_gpus = self._get_valid_gpus()
        if gpu <= len(valid_gpus):
            return valid_gpus[gpu]  # Returns "/dev/dri/renderD128", etc.
```

---

## 3. Python Process Management

### 3.1 Process Architecture Overview

**Source Files:** `frigate/video.py`, `frigate/util/process.py`

```
FrigateApp (Main Process)
│
├── Per-Camera Processes:
│   ├── CameraCapture (FrigateProcess)      # FFmpeg + frame capture
│   │   └── CameraWatchdog (Thread)         # Monitors FFmpeg health
│   │       └── CameraCaptureRunner (Thread) # Reads frames from pipe
│   └── CameraTracker (FrigateProcess)      # Motion + detection
│
├── Shared Processes:
│   ├── ObjectDetectProcess                 # Shared detector
│   ├── RecordProcess                       # Recording management
│   ├── ReviewProcess                       # Event review
│   └── StorageMaintainer                   # Storage cleanup
```

### 3.2 FFmpeg Process Spawning

**Source File:** `frigate/video.py:74-97`

```python
def start_or_restart_ffmpeg(
    ffmpeg_cmd,
    logger,
    logpipe: LogPipe,
    frame_size=None,          # None for record, size for detect
    ffmpeg_process=None       # Existing process to restart
) -> sp.Popen[Any]:

    # Stop existing process if restarting
    if ffmpeg_process is not None:
        stop_ffmpeg(ffmpeg_process, logger)

    # For recording (no frame output needed)
    if frame_size is None:
        process = sp.Popen(
            ffmpeg_cmd,
            stdout=sp.DEVNULL,           # Discard stdout
            stderr=logpipe,              # Capture errors via pipe
            stdin=sp.DEVNULL,            # No stdin
            start_new_session=True,      # Isolate from parent signals
        )
    # For detection (raw frames to pipe)
    else:
        process = sp.Popen(
            ffmpeg_cmd,
            stdout=sp.PIPE,              # Read raw frames
            stderr=logpipe,              # Capture errors
            stdin=sp.DEVNULL,
            bufsize=frame_size * 10,     # Buffer 10 frames
            start_new_session=True,
        )
    return process
```

### 3.3 FFmpeg Process Termination

**Source File:** `frigate/video.py:61-71`

```python
def stop_ffmpeg(ffmpeg_process: sp.Popen[Any], logger: logging.Logger):
    logger.info("Terminating the existing ffmpeg process...")
    ffmpeg_process.terminate()       # Send SIGTERM first

    try:
        logger.info("Waiting for ffmpeg to exit gracefully...")
        ffmpeg_process.communicate(timeout=30)  # Wait up to 30 seconds
    except sp.TimeoutExpired:
        logger.info("FFmpeg didn't exit. Force killing...")
        ffmpeg_process.kill()        # Send SIGKILL
        ffmpeg_process.communicate()
```

### 3.4 LogPipe - Capturing FFmpeg Stderr

**Source File:** `frigate/log.py:108-141`

FFmpeg's stderr output is captured via a pipe-based logging thread:

```python
class LogPipe(threading.Thread):
    def __init__(self, log_name: str, level: int = logging.ERROR):
        super().__init__(daemon=False)
        self.logger = logging.getLogger(log_name)
        self.level = level
        self.deque: Deque[str] = deque(maxlen=100)  # Keep last 100 lines

        # Create a pipe for FFmpeg stderr
        self.fdRead, self.fdWrite = os.pipe()
        self.pipeReader = os.fdopen(self.fdRead)
        self.start()

    def fileno(self) -> int:
        """Return write FD - passed to subprocess stderr."""
        return self.fdWrite

    def run(self) -> None:
        """Continuously read from pipe and store in deque."""
        for line in iter(self.pipeReader.readline, ""):
            self.deque.append(self.cleanup_log(line))
        self.pipeReader.close()

    def dump(self) -> None:
        """Log all buffered lines (called on error/restart)."""
        while len(self.deque) > 0:
            self.logger.log(self.level, self.deque.popleft())

    def close(self) -> None:
        """Close the write end of the pipe."""
        os.close(self.fdWrite)
```

**Usage Pattern:**
```python
# Create log pipe for this camera's FFmpeg
logpipe = LogPipe(f"ffmpeg.{camera_name}.detect")

# Start FFmpeg with stderr going to the pipe
process = sp.Popen(ffmpeg_cmd, stderr=logpipe, ...)

# On error, dump the last 100 lines
logpipe.dump()

# On shutdown
logpipe.close()
```

### 3.5 CameraWatchdog - FFmpeg Health Monitoring

**Source File:** `frigate/video.py:168-430`

The watchdog thread monitors FFmpeg and restarts it on failure:

```python
class CameraWatchdog(threading.Thread):
    def __init__(self, config, ...):
        self.sleeptime = config.ffmpeg.retry_interval  # Check interval

    def run(self) -> None:
        if self._update_enabled_state():
            self.start_all_ffmpeg()

        time.sleep(self.sleeptime)
        while not self.stop_event.wait(self.sleeptime):
            # Check if capture thread died
            if not self.capture_thread.is_alive():
                self.logger.error("Ffmpeg process crashed unexpectedly")
                self.reset_capture_thread(terminate=False)

            # Check for FPS overflow (stream sending too fast)
            elif self.camera_fps.value >= (self.config.detect.fps + 10):
                self.fps_overflow_count += 1
                if self.fps_overflow_count == 3:
                    self.logger.info("Exceeded fps limit. Exiting ffmpeg...")
                    self.reset_capture_thread(drain_output=False)

            # Check for stalled frames (no frames in 20 seconds)
            elif now - self.capture_thread.current_frame.value > 20:
                self.logger.info("No frames received in 20 seconds")
                self.reset_capture_thread()

            # Check recording health (no valid segments in 120 seconds)
            for p in self.ffmpeg_other_processes:
                if "record" in p["roles"]:
                    if cache_stale or valid_stale or invalid_stale:
                        p["process"] = start_or_restart_ffmpeg(...)
```

**Health Checks Performed:**
| Check | Threshold | Action |
|-------|-----------|--------|
| Capture thread dead | Immediate | Restart FFmpeg |
| FPS overflow | 3 consecutive checks | Restart FFmpeg |
| No frames | 20 seconds | Restart FFmpeg |
| No recording segments | 120 seconds | Restart recording FFmpeg |
| Process poll() not None | Immediate | Restart that process |

### 3.6 Frame Capture from FFmpeg Pipe

**Source File:** `frigate/video.py:100-165`

```python
def capture_frames(
    ffmpeg_process: sp.Popen[Any],
    config: CameraConfig,
    shm_frame_count: int,
    frame_index: int,
    frame_shape: tuple[int, int],
    frame_manager: FrameManager,
    frame_queue,
    fps: Value,
    skipped_fps: Value,
    current_frame: Value,
    stop_event: MpEvent,
) -> None:
    frame_size = frame_shape[0] * frame_shape[1]  # YUV420p size
    frame_rate = EventsPerSecond()

    while not stop_event.is_set():
        # Update FPS metrics
        fps.value = frame_rate.eps()
        current_frame.value = datetime.now().timestamp()

        # Get shared memory buffer for this frame slot
        frame_name = f"{config.name}_frame{frame_index}"
        frame_buffer = frame_manager.write(frame_name)

        try:
            # Read raw frame from FFmpeg stdout
            frame_buffer[:] = ffmpeg_process.stdout.read(frame_size)
        except Exception:
            if stop_event.is_set():
                break
            if ffmpeg_process.poll() is not None:
                break  # FFmpeg died
            continue

        frame_rate.update()

        # Add to processing queue (non-blocking)
        try:
            frame_queue.put((frame_name, current_frame.value), False)
            frame_manager.close(frame_name)
        except queue.Full:
            skipped_fps.update()  # Track dropped frames

        # Cycle through frame slots
        frame_index = 0 if frame_index == shm_frame_count - 1 else frame_index + 1
```

### 3.7 Shared Memory Frame Management

**Source File:** `frigate/util/image.py:838-909`

Frames are stored in POSIX shared memory for zero-copy IPC:

```python
class SharedMemoryFrameManager(FrameManager):
    def __init__(self):
        self.shm_store: dict[str, UntrackedSharedMemory] = {}

    def create(self, name: str, size) -> memoryview:
        """Create a new shared memory buffer."""
        shm = UntrackedSharedMemory(name=name, create=True, size=size)
        self.shm_store[name] = shm
        return shm.buf

    def write(self, name: str) -> Optional[memoryview]:
        """Get writable buffer (create if needed)."""
        if name in self.shm_store:
            shm = self.shm_store[name]
        else:
            shm = UntrackedSharedMemory(name=name)
            self.shm_store[name] = shm
        return shm.buf

    def get(self, name: str, shape) -> Optional[np.ndarray]:
        """Get numpy array view of shared memory."""
        shm = self.shm_store.get(name) or UntrackedSharedMemory(name=name)
        return np.ndarray(shape, dtype=np.uint8, buffer=shm.buf)

    def delete(self, name: str):
        """Unlink shared memory (cleanup)."""
        if name in self.shm_store:
            self.shm_store[name].close()
            self.shm_store[name].unlink()
            del self.shm_store[name]
```

**Frame Naming Convention:**
```
{camera_name}_frame{index}

Example: front_door_frame0, front_door_frame1, front_door_frame2, ...
```

### 3.8 FrigateProcess Base Class

**Source File:** `frigate/util/process.py:49-118`

```python
class FrigateProcess(BaseProcess):
    def pre_run_setup(self, logConfig: LoggerConfig | None = None) -> None:
        # Set process priority (nice value)
        os.nice(self.priority)

        # Set descriptive process name (visible in ps/top)
        setproctitle(self.name)  # e.g., "frigate.capture:front_door"

        # Name the main thread
        threading.current_thread().name = f"process:{self.name}"

        # Enable crash dumps
        faulthandler.enable()

        # Setup logging via queue handler
        self.logger = logging.getLogger(self.name)
        logging.getLogger().addHandler(QueueHandler(self.__log_queue))

        # Optional memray profiling
        self._setup_memray()
```

**Process Priority Levels:**
```python
PROCESS_PRIORITY_HIGH = 0    # Camera capture/tracking
PROCESS_PRIORITY_MED = 10    # Event processing
PROCESS_PRIORITY_LOW = 19    # Storage cleanup
```

---

## 4. Image Snapshots and Video Recording

### 4.1 Live Snapshot Creation (In-Memory)

**Source File:** `frigate/track/tracked_object.py:565-618`

Snapshots are created from in-memory frames using OpenCV, **NOT** FFmpeg:

```python
def write_snapshot_to_disk(self) -> None:
    snapshot_config = self.camera_config.snapshots

    # Get the best frame for this tracked object
    jpg_bytes = self.get_img_bytes(
        ext="jpg",
        timestamp=snapshot_config.timestamp,      # Add timestamp overlay
        bounding_box=snapshot_config.bounding_box, # Add bounding box
        crop=snapshot_config.crop,                # Crop to object
        height=snapshot_config.height,            # Resize height
        quality=snapshot_config.quality,          # JPEG quality (0-100)
    )

    # Save JPEG with overlays
    with open(f"{CLIPS_DIR}/{camera}-{object_id}.jpg", "wb") as f:
        f.write(jpg_bytes)
```

**Frame Processing Pipeline:**
```
1. FFmpeg outputs raw YUV420p frame
2. Frame stored in shared memory
3. TrackedObject selects "best frame" (highest confidence)
4. Convert: cv2.cvtColor(frame, cv2.COLOR_YUV2BGR_I420)
5. Draw overlays: cv2.rectangle(), cv2.putText()
6. Encode: cv2.imencode(".jpg", frame, [IMWRITE_JPEG_QUALITY, 70])
7. Save: {camera}-{id}.jpg + {camera}-{id}-clean.webp
```

### 4.2 On-Demand Snapshot from Recording

**Source File:** `frigate/util/image.py:946-996`

For extracting frames from saved recordings:

```python
def run_ffmpeg_snapshot(
    ffmpeg,
    input_path: str,
    codec: str,           # "mjpeg" or "png"
    seek_time: Optional[float] = None,
    height: Optional[int] = None,
    timeout: Optional[int] = None,
) -> tuple[Optional[bytes], str]:
    ffmpeg_cmd = [
        ffmpeg.ffmpeg_path,
        "-hide_banner",
        "-loglevel", "warning",
    ]

    # Seek to specific time (fast seek before input)
    if seek_time is not None:
        ffmpeg_cmd.extend(["-ss", f"00:00:{seek_time}"])

    ffmpeg_cmd.extend([
        "-i", input_path,
        "-frames:v", "1",      # Extract single frame
        "-c:v", codec,         # JPEG or PNG
        "-f", "image2pipe",    # Output to pipe
        "-",
    ])

    # Optional scaling
    if height is not None:
        ffmpeg_cmd.insert(-3, "-vf")
        ffmpeg_cmd.insert(-3, f"scale=-1:{height}")

    process = sp.run(ffmpeg_cmd, capture_output=True, timeout=timeout)
    return process.stdout, ""  # Returns JPEG/PNG bytes
```

### 4.3 Recording Segment Pipeline

**Recording Output Path:**
```python
# From frigate/config/camera/camera.py:223
output_path = f"{CACHE_DIR}/{camera_name}@{CACHE_SEGMENT_FORMAT}.mp4"
# Example: /tmp/cache/front_door@20251129143000+0000.mp4
```

**Segment Movement to Permanent Storage:**
```python
# From frigate/record/maintainer.py
# FFmpeg adds faststart for better seeking:
ffmpeg -y -i {cache_path} -c copy -movflags +faststart {final_path}

# Final path: /media/frigate/recordings/{YYYY-MM-DD}/{HH}/{camera}/{MM.SS}.mp4
```

---

## 5. Motion Detection and Frame Selection

### 5.1 Motion Detection Algorithm

**Source File:** `frigate/motion/improved_motion.py`

Frigate uses **Exponential Weighted Moving Average (EWMA) Background Subtraction**:

```python
def detect(self, frame) -> list[tuple[int, int, int, int]]:
    # 1. Resize to small size (default 100px height)
    resized = cv2.resize(frame, (self.frame_width, self.frame_height))

    # 2. Improve contrast (percentile clipping 4th-96th)
    if self.config.improve_contrast:
        resized = self.contrast_improve(resized)

    # 3. Apply Gaussian blur
    resized = cv2.GaussianBlur(resized, (0, 0), sigma=1)

    # 4. Compute absolute difference from background
    frame_delta = cv2.absdiff(resized, self.avg_frame.astype(np.uint8))

    # 5. Binary threshold (default: 30)
    thresh = cv2.threshold(frame_delta, self.config.threshold, 255, cv2.THRESH_BINARY)[1]

    # 6. Dilate to fill gaps
    thresh = cv2.dilate(thresh, None, iterations=1)

    # 7. Find contours
    contours = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)

    # 8. Filter by area and create motion boxes
    motion_boxes = []
    for contour in contours:
        if cv2.contourArea(contour) > self.config.contour_area:
            motion_boxes.append(cv2.boundingRect(contour))

    # 9. Update background model (slow adaptation)
    cv2.accumulateWeighted(resized, self.avg_frame, self.config.frame_alpha)

    return motion_boxes
```

### 5.2 Motion Configuration Parameters

**Source File:** `frigate/config/camera/motion.py`

| Parameter | Default | Description |
|-----------|---------|-------------|
| `threshold` | 30 | Pixel intensity change threshold (1-255) |
| `contour_area` | 10 | Minimum motion area in resized frame |
| `delta_alpha` | 0.2 | Weight for motion delta accumulation |
| `frame_alpha` | 0.01 | Background model update rate (lower = slower) |
| `frame_height` | 100 | Frame height for motion detection |
| `lightning_threshold` | 0.8 | Recalibrate if >80% of frame changes |
| `improve_contrast` | True | Apply contrast enhancement |

### 5.3 Frame Processing Decision Logic

**Source File:** `frigate/video.py:815-952`

This is **the critical decision point** for whether to process a frame:

```python
# Step 1: ALWAYS run motion detection (cheap operation)
motion_boxes = motion_detector.detect(frame)

# Step 2: Get regions around EXISTING tracked objects
object_regions = [get_cluster_region(...) for obj in tracked_objects]

# Step 3: Find NEW motion not covered by tracked objects
standalone_motion = [b for b in motion_boxes if not inside_any(b, object_regions)]

# Step 4: Cluster standalone motion into detection regions
if standalone_motion:
    motion_regions = cluster_and_create_regions(standalone_motion)
    regions = object_regions + motion_regions
else:
    regions = object_regions

# Step 5: THE KEY DECISION
if len(regions) > 0:
    # Run object detection ONLY on regions with motion
    for region in regions:
        detections.extend(detect(object_detector, frame, region))
    object_tracker.match_and_update(frame_time, detections)
else:
    # NO MOTION + NO TRACKED OBJECTS = SKIP EXPENSIVE DETECTION
    object_tracker.update_frame_times(frame_time)  # Just update timestamps
```

### 5.4 Practical Example: Parking Lot at Night

**Scenario:** Static parking lot, no car movement, no people

**What happens:**
1. Motion detector computes frame difference: **zero or near-zero**
2. `motion_boxes = []` (empty)
3. No existing tracked objects → `object_regions = []`
4. `regions = []` (empty)
5. **Object detection is COMPLETELY SKIPPED**
6. Only `update_frame_times()` is called (nearly free)
7. If recording is continuous, segment is still saved
8. If recording is motion-only, segment may be discarded

**Cost savings:**
- Object detection: **SKIPPED** (most expensive operation)
- API calls: **SKIPPED** (in your case)
- Only motion detection runs (~1ms per frame)

---

## 6. Video Storage Architecture

### 6.1 Directory Structure

```
/media/frigate/                    # BASE_DIR
├── recordings/                    # RECORD_DIR
│   └── {YYYY-MM-DD}/              # Date (UTC)
│       └── {HH}/                  # Hour (UTC)
│           └── {camera_name}/     # Camera
│               └── {MM.SS}.mp4    # Minute.Second start time
├── clips/                         # CLIPS_DIR
│   ├── {camera}-{id}.jpg          # Event snapshots
│   ├── {camera}-{id}-clean.webp   # Clean copies
│   ├── thumbs/                    # Thumbnails
│   └── previews/                  # Preview clips
└── exports/                       # User exports

/tmp/cache/                        # CACHE_DIR (temporary)
├── {camera}@{timestamp}.mp4       # Live recording segments
└── preview_frames/                # Preview frame cache
```

### 6.2 Three-Stage Recording Pipeline

```
Stage 1: Live Cache
    FFmpeg writes → /tmp/cache/{camera}@%Y%m%d%H%M%S%z.mp4
    Max segments: 6 (MAX_SEGMENTS_IN_CACHE)
    Segment duration: up to 10 minutes (MAX_SEGMENT_DURATION=600s)

         ↓ RecordingMaintainer (every 5 seconds)

Stage 2: Validation & Decision
    - Validate video with ffprobe
    - Calculate: motion_count, object_count, dBFS
    - Apply retention policy
    - Decision: KEEP or DISCARD

         ↓ If KEEP

Stage 3: Permanent Storage
    ffmpeg -i {cache} -c copy -movflags +faststart {final}
    → /media/frigate/recordings/{date}/{hour}/{camera}/{MM.SS}.mp4
    → Database: INSERT INTO Recordings (...)
```

### 6.3 Retention Logic

**Source File:** `frigate/record/maintainer.py:59-78`

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

    return True  # Discard this segment
```

---

## 7. Scaling Architecture

### 7.1 Process Model

**Per Camera:**
- `CameraCapture` (Process) - FFmpeg frame capture
- `CameraTracker` (Process) - Motion detection, detection requests

**Shared Across All Cameras:**
- `ObjectDetectProcess` (Process) - Single detector for ALL cameras
- `RecordProcess` (Process) - Recording management
- `ReviewProcess` (Process) - Event review
- `StorageMaintainer` (Process) - Storage cleanup

### 7.2 Camera Scaling Summary

| Cameras | Camera Processes | Detector Processes | FFmpeg Processes |
|---------|------------------|-------------------|------------------|
| 1 | 2 | 1 | 1-2 per camera |
| 16 | 32 | 1 | 16-32 |
| 200 | 400 | 1+ (multi-detector) | 200-400 |

### 7.3 Detection Queue Architecture

```
Camera 1     Camera 2     Camera 3     ...     Camera N
    ↓            ↓            ↓                    ↓
RemoteObjectDetector (per camera, sends to shared queue)
    └────────────┴────────────┴────────────────────┘
                              ↓
              detection_queue (multiprocessing.Queue)
                              ↓
               ObjectDetectProcess (SHARED)
                              ↓
              ZMQ Pub/Sub (detection results)
                              ↓
    ┌────────────┬────────────┬────────────────────┐
    ↓            ↓            ↓                    ↓
Camera 1     Camera 2     Camera 3     ...     Camera N
```

### 7.4 Multi-Detector Configuration

For high camera counts, multiple detectors can be configured:

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

---

## 8. Key Takeaways for System Design

### 8.1 FFmpeg Best Practices

1. **Use TCP for RTSP** - More reliable than UDP for most networks
2. **Separate streams by role** - Sub-stream for detection, main for recording
3. **Hardware acceleration** - Essential for scaling beyond 4-5 cameras
4. **Segment muxer** - Creates atomic, recoverable recording segments
5. **faststart flag** - Add when moving to permanent storage for seeking

### 8.2 Process Management Patterns

1. **Watchdog threads** - Monitor FFmpeg health, auto-restart on failure
2. **LogPipe for stderr** - Capture FFmpeg logs without blocking
3. **Shared memory for frames** - Zero-copy between processes
4. **Process isolation** - `start_new_session=True` for signal isolation
5. **Graceful shutdown** - SIGTERM first, SIGKILL after timeout

### 8.3 Frame Processing Optimization

1. **Motion-first filtering** - Skip expensive operations on static frames
2. **Region-based detection** - Only process areas with motion
3. **Background subtraction** - EWMA is efficient and adaptive
4. **Downsampled motion** - 100px height is sufficient for motion
5. **Stationary object caching** - Don't re-detect objects that haven't moved

### 8.4 For Your API-Based System

```
Recommended Architecture:

┌─────────────────────────────────────────────────────────────┐
│ Per-Camera Process                                          │
│   FFmpeg → Shared Memory → Motion Detection                 │
│                               ↓                             │
│                    motion_boxes empty? ──YES──→ SKIP        │
│                               │                             │
│                              NO                             │
│                               ↓                             │
│                    Queue frame for API                      │
└─────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────┐
│ API Request Queue (Shared)                                  │
│   - Rate limiting (respect API limits)                      │
│   - Batching (group frames if API supports)                 │
│   - Priority (favor high-motion frames)                     │
└─────────────────────────────────────────────────────────────┘
                               ↓
┌─────────────────────────────────────────────────────────────┐
│ API Worker Process(es)                                      │
│   - Fetch frame from shared memory                          │
│   - Convert YUV420p → JPEG/PNG                              │
│   - Call external API                                       │
│   - Return results via ZMQ                                  │
└─────────────────────────────────────────────────────────────┘
```

**Key decisions for your system:**
- Motion threshold for API calls (adjust `threshold` and `contour_area`)
- Batch size for API requests (if API supports batching)
- Rate limiting strategy (token bucket, sliding window)
- Result caching for repeated detections
- Fallback behavior when API is unavailable

---

## Appendix: File Reference

| Component | File Path |
|-----------|-----------|
| FFmpeg Presets | `frigate/ffmpeg_presets.py` |
| FFmpeg Command Building | `frigate/config/camera/camera.py:186-273` |
| FFmpeg Process Management | `frigate/video.py:61-97` |
| CameraWatchdog | `frigate/video.py:168-488` |
| Frame Capture | `frigate/video.py:100-165` |
| LogPipe | `frigate/log.py:108-141` |
| Shared Memory | `frigate/util/image.py:838-909` |
| Motion Detection | `frigate/motion/improved_motion.py` |
| Frame Processing | `frigate/video.py:690-1077` |
| Recording Maintainer | `frigate/record/maintainer.py` |
| Storage Cleanup | `frigate/record/cleanup.py` |
| Process Base Class | `frigate/util/process.py` |
| Constants | `frigate/const.py` |
