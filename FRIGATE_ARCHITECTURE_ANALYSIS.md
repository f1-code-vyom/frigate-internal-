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

### 3.1 Process vs Thread Clarification

**IMPORTANT: These are REAL OS Processes, NOT Python Threads**

**Source Files:** `frigate/util/process.py`, `frigate/__main__.py`

`FrigateProcess` inherits from `multiprocessing.Process` - these are **real OS processes** with:
- Separate PID
- Separate memory space
- Separate Python interpreter
- True parallelism (bypasses Python GIL)

```python
# frigate/util/process.py:19-35
class BaseProcess(mp.Process):  # Inherits from multiprocessing.Process
    def __init__(self, stop_event: MpEvent, priority: int, ...):
        super().__init__(...)  # Real OS process
```

**Process Spawning Method:**
```python
# frigate/__main__.py:135
mp.set_start_method("forkserver", force=True)  # Not fork, not spawn
mp.set_forkserver_preload([
    "sqlite3", "numpy", "cv2", "peewee", "zmq",
    "frigate.camera.maintainer",
])
```

**Why forkserver?**
- A "fork server" process is created at startup
- New processes fork from the server (not the main process)
- Heavy libraries preloaded once, inherited by all children
- Better memory efficiency for many processes

### 3.2 Process Architecture Overview

```
FrigateApp (Main Process - PID 1)
│
│   ═══════════════════════════════════════════════════
│   REAL OS PROCESSES (multiprocessing.Process)
│   Each has its own PID, memory space, Python interpreter
│   ═══════════════════════════════════════════════════
│
├── CameraCapture (Process - PID 100)      # Separate OS process
│   │
│   │   ─────────────────────────────────────────────
│   │   THREADS (threading.Thread) - within the process
│   │   Share memory, same PID, lightweight
│   │   ─────────────────────────────────────────────
│   │
│   └── CameraWatchdog (Thread)            # Thread inside PID 100
│       └── CameraCaptureRunner (Thread)   # Thread inside PID 100
│
├── CameraTracker (Process - PID 101)      # Another separate process
│
├── ObjectDetectProcess (Process - PID 102)
├── RecordProcess (Process - PID 103)
├── ReviewProcess (Process - PID 104)
└── StorageMaintainer (Process - PID 105)
```

**NOT Uvicorn - Direct Python Execution:**
```python
# frigate/__main__.py
if __name__ == "__main__":
    frigate_app = FrigateApp()
    frigate_app.start()  # Starts all processes directly
```

The web API (Flask/FastAPI) runs inside the main process, but camera/detection work runs in separate processes.

**Why Real Processes (Not Threads)?**
1. **Python GIL bypass** - Threads can't run Python code in parallel; processes can
2. **Crash isolation** - If CameraCapture crashes, it doesn't take down the detector
3. **True parallelism** - Each process uses a different CPU core
4. **Memory isolation** - One camera's memory leak doesn't affect others

**You can see them in `ps`:**
```bash
$ ps aux | grep frigate
frigate    1  ...  frigate (main)
frigate  100  ...  frigate.capture:front_door
frigate  101  ...  frigate.process:front_door
frigate  102  ...  frigate.detect
frigate  103  ...  frigate.record
```

### 3.3 Inter-Process Communication (IPC)

Since they're real processes, they use IPC mechanisms:

| Mechanism | Used For | Location |
|-----------|----------|----------|
| `multiprocessing.Queue` | Detection requests, event queues | Between camera and detector |
| Shared Memory (`/dev/shm`) | Frame buffers (zero-copy) | `frigate/util/image.py` |
| ZMQ Pub/Sub | Detection results, config updates | `ipc:///tmp/cache/detector_pub` |
| `multiprocessing.Value` | Metrics (FPS, timestamps) | Shared counters |

### 3.4 FFmpeg Process Spawning

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

### 3.5 FFmpeg Process Termination

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

### 3.6 LogPipe - Capturing FFmpeg Stderr

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

### 3.7 CameraWatchdog - FFmpeg Health Monitoring

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

### 3.8 Frame Capture from FFmpeg Pipe

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

### 3.9 Shared Memory Frame Management

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

### 3.10 FrigateProcess Base Class

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

### 4.1 What is a Snapshot?

A **snapshot** is a single JPEG/WebP image captured when an object is detected. It represents the "best frame" of a tracked object event.

**Key characteristics:**
- Created from **in-memory frames**, NOT from disk
- Uses **OpenCV** for encoding, NOT FFmpeg
- Stored to **disk** after encoding
- Two versions saved: with overlays (JPG) and clean (WebP)

### 4.2 Snapshot Storage Locations

**Source File:** `frigate/const.py`

```
/media/frigate/clips/                      # CLIPS_DIR
├── {camera}-{object_id}.jpg               # Snapshot WITH overlays
├── {camera}-{object_id}-clean.webp        # Clean snapshot (no overlays)
├── thumbs/                                # Thumbnails
│   └── {camera}-{object_id}.webp          # 175px cropped thumbnail
└── export/                                # Export thumbnails
```

### 4.3 Snapshot Creation Flow (In-Memory → Disk)

**Source File:** `frigate/track/tracked_object.py:435-600`

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Frame in Shared Memory (YUV420p raw)               │
│   - FFmpeg outputs to stdout pipe                          │
│   - CameraCaptureRunner reads into SharedMemoryFrameManager│
│   - Frame buffer: /dev/shm/{camera}_frame{N}               │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Frame Cache (In-Memory Dict)                       │
│   - TrackedObject maintains frame_cache dict               │
│   - Stores recent frames with detection data               │
│   - Selects "best frame" (highest confidence + area)       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Color Conversion (OpenCV)                          │
│   cv2.cvtColor(frame, cv2.COLOR_YUV2BGR_I420)              │
│   - Converts YUV420p → BGR for processing                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Overlay Rendering (OpenCV)                         │
│   - Bounding boxes: cv2.rectangle()                        │
│   - Labels: draw_box_with_label()                          │
│   - Timestamps: draw_timestamp()                           │
│   - Crop to object region if configured                    │
│   - Resize to configured height                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ Step 5: Encode & Write to Disk                             │
│   cv2.imencode(".jpg", frame, [IMWRITE_JPEG_QUALITY, 70])  │
│   → /media/frigate/clips/{camera}-{id}.jpg                 │
│                                                            │
│   cv2.imencode(".webp", frame, [IMWRITE_WEBP_QUALITY, 60]) │
│   → /media/frigate/clips/{camera}-{id}-clean.webp          │
└─────────────────────────────────────────────────────────────┘
```

**Code for snapshot creation:**
```python
# frigate/track/tracked_object.py:565-600
def write_snapshot_to_disk(self) -> None:
    snapshot_config = self.camera_config.snapshots

    # Get best frame from IN-MEMORY cache (not disk!)
    jpg_bytes = self.get_img_bytes(
        ext="jpg",
        timestamp=snapshot_config.timestamp,
        bounding_box=snapshot_config.bounding_box,
        crop=snapshot_config.crop,
        height=snapshot_config.height,
        quality=snapshot_config.quality,
    )

    # Write to disk
    with open(f"{CLIPS_DIR}/{camera}-{object_id}.jpg", "wb") as f:
        f.write(jpg_bytes)

    # Also write clean WebP copy (no overlays)
    if snapshot_config.clean_copy:
        webp_bytes = self.get_clean_webp()
        with open(f"{CLIPS_DIR}/{camera}-{id}-clean.webp", "wb") as f:
            f.write(webp_bytes)
```

### 4.4 On-Demand Snapshot from Recording (FFmpeg)

For extracting frames from **saved recordings** (not live):

**Source File:** `frigate/util/image.py:946-996`

```python
def run_ffmpeg_snapshot(ffmpeg, input_path, codec, seek_time=None, height=None):
    ffmpeg_cmd = [
        ffmpeg.ffmpeg_path,
        "-hide_banner", "-loglevel", "warning",
        "-ss", f"00:00:{seek_time}",    # Seek to timestamp
        "-i", input_path,                # Recording file
        "-frames:v", "1",                # Single frame
        "-c:v", codec,                   # mjpeg or png
        "-f", "image2pipe",              # Output to pipe
        "-",
    ]
    process = sp.run(ffmpeg_cmd, capture_output=True)
    return process.stdout  # JPEG/PNG bytes
```

### 4.5 Recording: How It Works

**Recording is NOT snapshot-based** - it's continuous video segments.

**Two-Stage Process:**
1. **FFmpeg writes** 10-second MP4 segments to `/tmp/cache/`
2. **RecordingMaintainer** moves valid segments to permanent storage

### 4.6 Recording Storage & Seeking

**Recording Storage Structure:**
```
/media/frigate/recordings/
├── 2024-01-15/                    # Date (UTC)
│   ├── 00/                        # Hour (UTC)
│   │   ├── front_door/
│   │   │   ├── 00.00.mp4          # MM.SS = start time
│   │   │   ├── 00.10.mp4          # 10 seconds later
│   │   │   ├── 00.20.mp4
│   │   │   └── ...
│   │   └── backyard/
│   ├── 01/
│   └── ...
```

**Segment Filename Format:**
- `{MM.SS}.mp4` where MM = minute, SS = second of start time
- Each segment is ~10 seconds (configurable up to 600s max)

### 4.7 Recording Playback & Seeking (HLS/VOD)

**Source File:** `frigate/api/media.py:835-900`

Frigate serves recordings via **HLS (HTTP Live Streaming)**:

```
GET /vod/{camera}/start/{start_ts}/end/{end_ts}/index.m3u8
```

**How seeking works:**
1. API receives time range request
2. Query database for matching recording segments
3. Generate HLS playlist with segment references
4. Player fetches segments as needed

```python
# frigate/api/media.py:840-890
async def vod_ts(camera_name: str, start_ts: float, end_ts: float):
    # Query recordings in time range
    recordings = Recordings.select().where(
        Recordings.start_time.between(start_ts, end_ts)
    ).where(Recordings.camera == camera_name)

    # Build clip list with seek offsets
    clips = []
    for recording in recordings:
        clip = {"type": "source", "path": recording.path}

        # Adjust start offset if start_ts is after recording.start_time
        if start_ts > recording.start_time:
            clip["clipFrom"] = int((start_ts - recording.start_time) * 1000)

        clips.append(clip)

    return {"clips": clips, "durations": durations}
```

### 4.8 Recording Joining (Export/Clip)

**Source File:** `frigate/api/media.py:727-832`

When exporting a clip, segments are joined using FFmpeg concat:

```python
# frigate/api/media.py:782-827
# 1. Create concat playlist file
with open(file_path, "w") as file:
    for clip in recordings:
        file.write(f"file '{clip.path}'\n")
        if clip.start_time < start_ts:
            file.write(f"inpoint {int(start_ts - clip.start_time)}\n")
        if clip.end_time > end_ts:
            file.write(f"outpoint {int(end_ts - clip.start_time)}\n")

# 2. FFmpeg concat demuxer joins them
ffmpeg_cmd = [
    config.ffmpeg.ffmpeg_path,
    "-f", "concat",
    "-safe", "0",
    "-i", file_path,             # Playlist file
    "-c", "copy",                # No re-encoding
    "-movflags", "frag_keyframe+empty_moov",
    "-f", "mp4",
    "pipe:",                     # Stream output
]
```

**Export vs Streaming:**
| Method | Use Case | Command |
|--------|----------|---------|
| HLS/VOD | In-app playback | Returns playlist, player fetches segments |
| clip.mp4 | Download/export | FFmpeg concat → single MP4 file |

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

### 6.1 Storage Layer Overview

**Frigate uses a hybrid storage approach:**

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Database** | SQLite (with WAL mode) | Metadata, events, recording index |
| **File System** | Local disk | Video segments, snapshots |
| **Shared Memory** | POSIX `/dev/shm` | Live frame buffers |
| **Temp Cache** | `/tmp/cache/` (tmpfs) | Active recording segments |

**NOT used:** Redis, PostgreSQL, cloud storage

### 6.2 Database: SQLite

**Source Files:** `frigate/app.py:165-270`, `frigate/models.py`

```python
# frigate/app.py:262-270
db = SqliteVecQueueDatabase(
    self.config.database.path,  # /config/frigate.db
    pragmas={
        "auto_vacuum": "FULL",    # Automatic cleanup
        "cache_size": -512 * 1000, # 512MB cache
        "synchronous": "NORMAL",   # WAL-safe mode
    },
    timeout=max(60, 10 * num_cameras)
)
```

**Database Location:** `/config/frigate.db`

**SQLite Configuration:**
- **WAL mode** (Write-Ahead Logging) for concurrent reads/writes
- **512MB cache** for performance
- **Auto-vacuum FULL** for automatic space reclamation
- **SqliteQueueDatabase** - thread-safe wrapper from Peewee

**IMPORTANT: SQLite and Multi-Process Architecture**

SQLite is opened in the **main process only**. Child processes **cannot write directly** to SQLite because:
1. SQLite connection is not shared across `fork()`
2. SQLite doesn't handle multi-process writes well
3. Database file locks would cause conflicts

**Solution:** Child processes use **ZMQ REQ/REP** (InterProcessRequestor) to send write requests to the main process:

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MAIN PROCESS                                 │
│  SQLite DB ◄─── Dispatcher receives requests via ZMQ REP           │
│  (Peewee ORM)    and executes: Recordings.insert_many(...)          │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ ZMQ REQ/REP (ipc:///tmp/cache/comms)
                              │
      ┌───────────────────────┼───────────────────────┐
      │                       │                       │
      ▼                       ▼                       ▼
┌───────────┐          ┌───────────┐          ┌───────────┐
│ Recording │          │ Review    │          │ Event     │
│ Process   │          │ Process   │          │ Process   │
│           │          │           │          │           │
│ requestor │          │ requestor │          │ requestor │
│ .send_data│          │ .send_data│          │ .send_data│
│ (INSERT...)│         │ (UPSERT...)│         │ (UPDATE...)│
└───────────┘          └───────────┘          └───────────┘
```

### 6.3 Database Schema (Key Tables)

**Source File:** `frigate/models.py`

#### Recordings Table
```python
class Recordings(Model):
    id = CharField(primary_key=True)      # "{timestamp}-{rand_id}"
    camera = CharField(index=True)         # Camera name
    path = CharField(unique=True)          # Full file path

    start_time = DateTimeField()           # Segment start (UTC)
    end_time = DateTimeField()             # Segment end (UTC)
    duration = FloatField()                # Duration in seconds

    # Metrics for retention decisions
    motion = IntegerField()                # Motion frame count
    objects = IntegerField()               # Active object count
    regions = IntegerField()               # Detection region count
    dBFS = IntegerField()                  # Audio level (decibels)
    segment_size = FloatField()            # Size in MB
```

#### Event Table
```python
class Event(Model):
    id = CharField(primary_key=True)
    label = CharField(index=True)          # "person", "car", etc.
    camera = CharField(index=True)
    start_time = DateTimeField()
    end_time = DateTimeField()

    has_clip = BooleanField(default=True)  # Has associated recording
    has_snapshot = BooleanField(default=True)  # Has snapshot image
    thumbnail = TextField()                 # Base64 encoded thumbnail
    retain_indefinitely = BooleanField()   # Don't auto-delete
    data = JSONField()                      # Detection metadata
```

#### ReviewSegment Table
```python
class ReviewSegment(Model):
    id = CharField(primary_key=True)
    camera = CharField(index=True)
    start_time = DateTimeField()
    end_time = DateTimeField()
    severity = CharField()                 # "alert" or "detection"
    thumb_path = CharField(unique=True)    # Thumbnail file path
    data = JSONField()                     # Labels, zones, motion data
```

### 6.4 Directory Structure

```
/media/frigate/                    # BASE_DIR (persistent storage)
├── recordings/                    # RECORD_DIR
│   └── {YYYY-MM-DD}/              # Date (UTC)
│       └── {HH}/                  # Hour (UTC)
│           └── {camera_name}/     # Camera
│               └── {MM.SS}.mp4    # Minute.Second start time
├── clips/                         # CLIPS_DIR
│   ├── {camera}-{id}.jpg          # Event snapshots
│   ├── {camera}-{id}-clean.webp   # Clean copies
│   ├── thumbs/                    # Thumbnails
│   ├── faces/                     # Face recognition images
│   └── previews/                  # Hour-long preview clips
└── exports/                       # User exports

/config/                           # CONFIG_DIR
├── frigate.db                     # SQLite database
├── frigate.db-wal                 # WAL file
└── model_cache/                   # ML model cache

/tmp/cache/                        # CACHE_DIR (tmpfs - RAM)
├── {camera}@{timestamp}.mp4       # Active recording segments
├── preview_frames/                # Preview frame cache
└── birdseye                       # Birdseye pipe
```

### 6.5 Three-Stage Recording Pipeline

```
Stage 1: Live Cache (RAM - /tmp/cache/)
    FFmpeg writes → /tmp/cache/{camera}@%Y%m%d%H%M%S%z.mp4
    Max segments: 6 (MAX_SEGMENTS_IN_CACHE)
    Segment duration: up to 10 minutes (MAX_SEGMENT_DURATION=600s)

         ↓ RecordingMaintainer runs every 5 seconds

Stage 2: Validation & Decision
    - Check if file in use (psutil.process_iter)
    - Get video properties (ffprobe)
    - Calculate metrics:
      • motion_count (frames with motion)
      • active_object_count (detected objects)
      • region_count (detection regions)
      • average_dBFS (audio level)
    - Apply retention policy
    - Decision: KEEP or DISCARD

         ↓ If KEEP

Stage 3: Permanent Storage
    # Add faststart for better seeking
    ffmpeg -y -i {cache} -c copy -movflags +faststart {final}

    # File: /media/frigate/recordings/{YYYY-MM-DD}/{HH}/{camera}/{MM.SS}.mp4
    # Database: INSERT INTO Recordings (id, camera, path, start_time, ...)
```

**Source File:** `frigate/record/maintainer.py:489-577`

```python
async def move_segment(self, camera, start_time, end_time, duration, cache_path, store_mode):
    segment_info = self.segment_stats(camera, start_time, end_time)

    # Check if should discard
    if segment_info.should_discard_segment(store_mode):
        self.drop_segment(cache_path)
        return

    # Build directory path
    directory = os.path.join(RECORD_DIR, start_time.strftime("%Y-%m-%d/%H"), camera)
    os.makedirs(directory, exist_ok=True)
    file_path = os.path.join(directory, f"{start_time.strftime('%M.%S.mp4')}")

    # FFmpeg with faststart
    await asyncio.create_subprocess_exec(
        self.config.ffmpeg.ffmpeg_path,
        "-hide_banner", "-y",
        "-i", cache_path,
        "-c", "copy",
        "-movflags", "+faststart",
        file_path,
    )

    # Insert into database
    return {
        Recordings.id.name: f"{start_time.timestamp()}-{rand_id}",
        Recordings.camera.name: camera,
        Recordings.path.name: file_path,
        Recordings.start_time.name: start_time.timestamp(),
        Recordings.end_time.name: end_time.timestamp(),
        Recordings.duration.name: duration,
        Recordings.motion.name: segment_info.motion_count,
        Recordings.objects.name: segment_info.active_object_count,
        Recordings.segment_size.name: segment_size,
    }
```

### 6.6 Retention Logic

**Source File:** `frigate/record/maintainer.py:59-78`

```python
class SegmentInfo:
    def should_discard_segment(self, retain_mode: RetainModeEnum) -> bool:
        keep = False

        # Mode: all - keep everything
        if retain_mode == RetainModeEnum.all:
            keep = True

        # Mode: motion - keep if motion or audio detected
        if retain_mode == RetainModeEnum.motion:
            if self.motion_count > 0 or self.average_dBFS > 0:
                keep = True

        # Mode: active_objects - keep only if objects detected
        if self.active_object_count > 0:
            keep = True

        return not keep  # Return True to discard
```

### 6.7 Storage Cleanup

**Source File:** `frigate/record/cleanup.py`

Cleanup runs every `expire_interval` minutes (default: 60):

```python
def expire_recordings():
    for camera in cameras:
        # Calculate expiration dates based on config
        continuous_expire = now - timedelta(days=config.continuous.days)
        motion_expire = now - timedelta(days=config.motion.days)

        # Query expired recordings
        expired = Recordings.select().where(
            Recordings.camera == camera,
            Recordings.end_time < expire_date
        )

        for recording in expired:
            # Check if overlaps with any retained event
            if not overlaps_review_segment(recording):
                # Delete file and database entry
                os.remove(recording.path)
                recording.delete_instance()

    # Cleanup empty directories
    remove_empty_directories(RECORD_DIR)

    # Truncate WAL file if > 10MB
    if wal_size > MAX_WAL_SIZE:
        db.execute_sql("PRAGMA wal_checkpoint(TRUNCATE)")
```

### 6.8 Storage Summary

| What | Where | Format | Persistence |
|------|-------|--------|-------------|
| Live frames | `/dev/shm/{camera}_frame{N}` | YUV420p raw | Volatile (RAM) |
| Active recordings | `/tmp/cache/{camera}@{ts}.mp4` | MP4 | Volatile (tmpfs) |
| Permanent recordings | `/media/frigate/recordings/...` | MP4 | Persistent |
| Event snapshots | `/media/frigate/clips/{camera}-{id}.jpg` | JPEG/WebP | Persistent |
| Metadata | `/config/frigate.db` | SQLite | Persistent |

---

## 7. Inter-Process Communication (IPC) Patterns

### 7.1 Overview

Frigate uses **three main IPC patterns** for communication between processes:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           COMMUNICATION PATTERNS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. ZMQ Pub/Sub (Proxy)  → One-to-many broadcast (detections, events)        │
│ 2. ZMQ REQ/REP          → Request-response (database writes, queries)       │
│ 3. multiprocessing.Queue → Direct frame passing (detection queue)           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Pattern 1: ZMQ Pub/Sub Proxy (Detection Broadcasting)

**Purpose:** Broadcast detection results from one publisher to multiple subscribers

**Source Files:** `frigate/comms/zmq_proxy.py`, `frigate/comms/detections_updater.py`

```
                         ZMQ Proxy (XSUB/XPUB)
                      ipc:///tmp/cache/proxy_pub
                      ipc:///tmp/cache/proxy_sub
                               │
   PUBLISHERS                  │                    SUBSCRIBERS
   ──────────                  │                    ───────────
┌─────────────────┐            │            ┌─────────────────────┐
│ TrackedObject   │            │            │ RecordingMaintainer │
│ Processor       │───publish──┼──subscribe─│ (segment stats)     │
└─────────────────┘            │            └─────────────────────┘
                               │
┌─────────────────┐            │            ┌─────────────────────┐
│ AudioProcessor  │───publish──┼──subscribe─│ ReviewMaintainer    │
└─────────────────┘            │            └─────────────────────┘
                               │
                               │            ┌─────────────────────┐
                               ├──subscribe─│ EmbeddingMaintainer │
                               │            └─────────────────────┘
                               │
                               │            ┌─────────────────────┐
                               └──subscribe─│ OutputProcessor     │
                                            └─────────────────────┘
```

**Topic Structure:**
```
detection/video  → Video frame detections (camera, objects, motion, regions)
detection/audio  → Audio detections (camera, dBFS, audio_labels)
detection/api    → Manual API-triggered detections
detection/lpr    → License plate detections
event/update     → Object tracking start/update
event/finalized  → Object tracking ended
recordings/saved → Segment saved to permanent storage
```

**Publisher Example:**
```python
# frigate/track/object_processing.py:765-775
self.detection_publisher.publish(
    (
        camera,              # "front_door"
        frame_name,          # "front_door_frame3"
        frame_time,          # 1699876543.123
        tracked_objects,     # [{id, label, box, score, ...}]
        motion_boxes,        # [(x, y, w, h), ...]
        regions,             # [(x, y, w, h), ...]
    ),
    DetectionTypeEnum.video.value,  # Topic suffix
)
# Sends to: "detection/video {json_payload}"
```

**Subscriber Example:**
```python
# frigate/record/maintainer.py:605-630
(topic, data) = self.detection_subscriber.check_for_update(timeout=0.1)

if topic == DetectionTypeEnum.video.value:
    (camera, _, frame_time, objects, motion, regions) = data
    self.object_recordings_info[camera].append(
        (frame_time, objects, motion, regions)
    )
```

### 7.3 Pattern 2: ZMQ REQ/REP (Database Writes via InterProcessRequestor)

**Purpose:** Child processes request main process to write to SQLite (and wait for response)

**Source Files:** `frigate/comms/inter_process.py`, `frigate/comms/dispatcher.py`

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MAIN PROCESS                                 │
│                                                                      │
│  InterProcessCommunicator (zmq.REP)                                 │
│  ipc:///tmp/cache/comms                                             │
│         │                                                            │
│         ▼                                                            │
│  Dispatcher handles:                                                 │
│    INSERT_MANY_RECORDINGS  → Recordings.insert_many(payload)        │
│    UPSERT_REVIEW_SEGMENT   → ReviewSegment.insert(...).on_conflict()│
│    INSERT_PREVIEW          → Previews.insert(payload)               │
│    REQUEST_REGION_GRID     → Returns detection grid data            │
│         │                                                            │
│         ▼                                                            │
│  SQLite (Peewee ORM)                                                │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ ZMQ REQ/REP
                              │
      ┌───────────────────────┼───────────────────────┐
      ▼                       ▼                       ▼
┌───────────┐          ┌───────────┐          ┌───────────┐
│Recording  │          │Review     │          │Preview    │
│Maintainer │          │Maintainer │          │Process    │
└───────────┘          └───────────┘          └───────────┘
```

**InterProcessRequestor (Child Process Side):**
```python
# frigate/comms/inter_process.py:68-86
class InterProcessRequestor:
    def __init__(self):
        self.context = zmq.Context()
        self.socket = self.context.socket(zmq.REQ)  # Request socket
        self.socket.connect("ipc:///tmp/cache/comms")

    def send_data(self, topic: str, data: Any) -> Any:
        self.socket.send_json((topic, data))
        return self.socket.recv_json()  # BLOCKS until response
```

**Usage in RecordingMaintainer:**
```python
# Child process wants to insert recordings
recordings_to_insert = [
    {"id": "123", "camera": "front", "path": "/media/...", ...},
    {"id": "124", "camera": "front", "path": "/media/...", ...},
]

# Send to main process, wait for confirmation
self.requestor.send_data(INSERT_MANY_RECORDINGS, recordings_to_insert)
```

**Dispatcher (Main Process Side):**
```python
# frigate/comms/dispatcher.py:114-115
def _receive(self, topic: str, payload: Any):
    if topic == INSERT_MANY_RECORDINGS:
        Recordings.insert_many(payload).execute()
        return {"success": True}
```

### 7.4 Pattern 3: multiprocessing.Queue (Frame Passing)

**Purpose:** Pass detection requests from multiple cameras to shared detector

**Source File:** `frigate/app.py:148-160`

```python
# Created once in main process
self.detection_queue: Queue = mp.Queue()

# Passed to all camera processes AND detector process
# All cameras write to same queue, detector reads from it
```

```
┌─────────────────┐     detection_queue    ┌─────────────────┐
│ CameraTracker 1 │──────────┐             │                 │
│ (front_door)    │          │             │ ObjectDetect    │
└─────────────────┘          │             │ Process         │
                             ├──────.put()─▶│                 │
┌─────────────────┐          │             │ while True:     │
│ CameraTracker 2 │──────────┤             │   .get()        │
│ (backyard)      │          │             │   detect()      │
└─────────────────┘          │             └────────┬────────┘
                             │                      │
┌─────────────────┐          │                      │ Results via
│ CameraTracker N │──────────┘                      │ ZMQ Pub/Sub
└─────────────────┘                                 ▼
```

**Why mp.Queue vs ZMQ for this?**

| Aspect | mp.Queue | ZMQ Pub/Sub |
|--------|----------|-------------|
| **Pattern** | Many-to-one | One-to-many |
| **Guarantee** | Messages never lost | Fire-and-forget |
| **Backpressure** | Blocks when full | Drops if slow |
| **Use case** | Detection requests (must not lose) | Broadcasts (ok to miss) |

### 7.5 Why ZMQ Instead of Redis?

Frigate chose ZMQ over Redis for these reasons:

| Aspect | ZMQ (Frigate's choice) | Redis |
|--------|------------------------|-------|
| **Latency** | ~10-50μs (IPC sockets) | ~100-500μs (TCP overhead) |
| **Dependency** | Library (no server) | External server required |
| **Deployment** | Zero config | Redis to configure/maintain |
| **Memory** | In-process | Separate process |
| **Failure mode** | Process crash = local | Redis crash = all affected |

**When Redis would make sense:**
- Distributed across multiple machines
- Need message persistence/replay
- Already have Redis in your stack
- Need caching alongside pub/sub

### 7.6 IPC Data Flow Example

**Complete flow: Camera frame → Database recording entry**

```
1. FFmpeg captures frame
   └─▶ CameraCaptureRunner writes to SharedMemory
       └─▶ /dev/shm/front_door_frame0

2. CameraTracker detects motion
   └─▶ Sends to detection_queue (mp.Queue)
       └─▶ ObjectDetectProcess runs inference
           └─▶ Returns via shared memory + ZMQ signal

3. TrackedObjectProcessor publishes detection
   └─▶ DetectionPublisher (ZMQ Pub/Sub)
       Topic: "detection/video"
       Payload: (camera, frame_name, time, objects, motion, regions)

4. RecordingMaintainer subscribes & receives
   └─▶ DetectionSubscriber gets message
       └─▶ Stores in object_recordings_info[camera] (dict)

5. Every 5 seconds: RecordingMaintainer processes segments
   └─▶ Calculate segment stats from stored detection data
       └─▶ If keep segment:
           └─▶ FFmpeg moves cache → permanent
               └─▶ InterProcessRequestor.send_data(
                       INSERT_MANY_RECORDINGS,
                       [{camera, path, motion, objects, ...}]
                   )

6. Dispatcher (main process) receives
   └─▶ Recordings.insert_many(payload).execute()
       └─▶ SQLite: INSERT INTO recordings (...)
```

### 7.7 IPC Summary Table

| Component | Type | Direction | Use Case |
|-----------|------|-----------|----------|
| `mp.Queue` | multiprocessing.Queue | Many→One | Camera → Detector frames |
| ZMQ Pub/Sub | XPUB/XSUB | One→Many | Detection results broadcast |
| ZMQ REQ/REP | REQ/REP | Request-Response | Database writes |
| Shared Memory | `/dev/shm` | Shared buffer | Zero-copy frame data |
| `mp.Value` | multiprocessing.Value | Shared counter | FPS, timestamps |

---

## 8. Scaling Architecture

### 8.1 Process Model

**Per Camera:**
- `CameraCapture` (Process) - FFmpeg frame capture
- `CameraTracker` (Process) - Motion detection, detection requests

**Shared Across All Cameras:**
- `ObjectDetectProcess` (Process) - Single detector for ALL cameras
- `RecordProcess` (Process) - Recording management
- `ReviewProcess` (Process) - Event review
- `StorageMaintainer` (Process) - Storage cleanup

### 8.2 Camera Scaling Summary

| Cameras | Camera Processes | Detector Processes | FFmpeg Processes |
|---------|------------------|-------------------|------------------|
| 1 | 2 | 1 | 1-2 per camera |
| 16 | 32 | 1 | 16-32 |
| 200 | 400 | 1+ (multi-detector) | 200-400 |

### 8.3 Detection Queue Architecture

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

### 8.4 Multi-Detector Configuration

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

### 8.5 Single-Machine Architecture (Limitation)

**Frigate is designed for single-machine deployment only.** All IPC mechanisms are local:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SINGLE MACHINE                                  │
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  IPC Mechanisms (ALL LOCAL)                                      │   │
│   │                                                                  │   │
│   │  1. ZMQ IPC Sockets:                                            │   │
│   │     ipc:///tmp/cache/proxy_pub    ← Unix domain socket          │   │
│   │     ipc:///tmp/cache/proxy_sub    ← Unix domain socket          │   │
│   │     ipc:///tmp/cache/comms        ← Unix domain socket          │   │
│   │                                                                  │   │
│   │  2. Shared Memory:                                               │   │
│   │     /dev/shm/front_door_frame0    ← Local RAM                   │   │
│   │     /dev/shm/backyard_frame0      ← Local RAM                   │   │
│   │                                                                  │   │
│   │  3. multiprocessing.Queue:                                       │   │
│   │     Backed by pipes/shared memory ← Local IPC                   │   │
│   │                                                                  │   │
│   │  4. SQLite Database:                                             │   │
│   │     /config/frigate.db            ← Local file                  │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │
│   │ Camera 1 │ │ Camera 2 │ │ Camera 3 │ │   ...    │ │ Camera N │     │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘     │
│        │            │            │            │            │             │
│        └────────────┴────────────┴────────────┴────────────┘             │
│                          All on same host                                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Why Frigate cannot span multiple machines:**

| Component | Technology | Why Single-Machine Only |
|-----------|------------|------------------------|
| ZMQ Sockets | `ipc://` (Unix domain) | Unix sockets exist only within one host's filesystem |
| Shared Memory | `/dev/shm` | RAM-backed tmpfs, not accessible across network |
| mp.Queue | Unix pipes | Kernel pipes don't cross machine boundaries |
| SQLite | Local file | No network protocol, file locking is local |
| Process spawning | `multiprocessing.Process` | Spawns child processes on same machine |

**Consequence for 50+ cameras:**

```
❌ What you CANNOT do:

Machine A                          Machine B
┌────────────┐                    ┌────────────┐
│ Cameras    │                    │ Cameras    │
│ 1-25       │                    │ 26-50      │
│            │      ???           │            │
│ Frigate    │◄─────────────────► │ Frigate    │
│ Instance 1 │  No IPC possible   │ Instance 2 │
└────────────┘                    └────────────┘

✅ What you CAN do (workarounds):

Option 1: Run completely separate Frigate instances
┌────────────┐                    ┌────────────┐
│ Frigate A  │                    │ Frigate B  │
│ Cameras    │                    │ Cameras    │
│ 1-25       │                    │ 26-50      │
│            │                    │            │
│ Own DB     │                    │ Own DB     │
│ Own API    │                    │ Own API    │
└────────────┘                    └────────────┘
     │                                  │
     └──────────► Aggregation ◄─────────┘
                 Layer (your code)
                 - Combine APIs
                 - Unified dashboard

Option 2: RTSP proxy with single big machine
┌────────────┐     ┌─────────────────────────────┐
│ NVR/Camera │     │      POWERFUL SINGLE        │
│ Network    │     │      MACHINE                │
│            │     │                             │
│ 50 streams │────►│  Frigate                    │
│ RTSP       │     │  - 50 camera processes      │
└────────────┘     │  - Multiple detectors       │
                   │  - 128GB+ RAM               │
                   │  - Multi-GPU                │
                   └─────────────────────────────┘
```

**If you need true multi-machine distribution, you would need to:**

1. **Replace ZMQ IPC with TCP sockets:**
   ```python
   # Current (local only):
   SOCKET_PUB = "ipc:///tmp/cache/proxy_pub"

   # Distributed would need:
   SOCKET_PUB = "tcp://192.168.1.100:5555"
   ```

2. **Replace shared memory with network storage or Redis:**
   ```python
   # Current: /dev/shm/camera_frame (zero-copy, local)
   # Distributed: Network copy each frame (bandwidth intensive!)
   # A single 1080p YUV frame = ~3MB
   # At 5fps × 50 cameras = 750MB/sec network traffic
   ```

3. **Replace SQLite with PostgreSQL/MySQL:**
   - Network-accessible database
   - Connection pooling
   - Replication for high availability

4. **Replace mp.Queue with distributed queue (Redis, RabbitMQ):**
   ```python
   # Current
   detection_queue = mp.Queue()

   # Distributed
   redis_client.lpush("detection_queue", frame_data)
   ```

**For your API-based system, consider:**

If you need to scale beyond one machine:
- Use Redis for IPC (already network-capable)
- Use PostgreSQL for persistence
- Use S3/MinIO for video storage
- Each machine can run independent camera processes
- Centralized detection workers pull from Redis queue

---

## 9. Key Takeaways for System Design

### 9.1 FFmpeg Best Practices

1. **Use TCP for RTSP** - More reliable than UDP for most networks
2. **Separate streams by role** - Sub-stream for detection, main for recording
3. **Hardware acceleration** - Essential for scaling beyond 4-5 cameras
4. **Segment muxer** - Creates atomic, recoverable recording segments
5. **faststart flag** - Add when moving to permanent storage for seeking

### 9.2 Process Management Patterns

1. **Watchdog threads** - Monitor FFmpeg health, auto-restart on failure
2. **LogPipe for stderr** - Capture FFmpeg logs without blocking
3. **Shared memory for frames** - Zero-copy between processes
4. **Process isolation** - `start_new_session=True` for signal isolation
5. **Graceful shutdown** - SIGTERM first, SIGKILL after timeout

### 9.3 Frame Processing Optimization

1. **Motion-first filtering** - Skip expensive operations on static frames
2. **Region-based detection** - Only process areas with motion
3. **Background subtraction** - EWMA is efficient and adaptive
4. **Downsampled motion** - 100px height is sufficient for motion
5. **Stationary object caching** - Don't re-detect objects that haven't moved

### 9.4 For Your API-Based System

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
