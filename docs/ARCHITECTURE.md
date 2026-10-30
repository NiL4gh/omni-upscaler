# 🔬 OmniUpscaler Technical Architecture

This document details the engineering decisions, PyTorch 2.6 compatibility layers, in-memory FFmpeg streaming mechanics, and segment chunking pipeline behind **OmniUpscaler**.

---

## 🏗️ Architectural Overview

```
                                      [User Video File]
                                              │
                                              ▼
                                 FFmpeg Normalization & Probe
                               (Streams extracted, audio isolated)
                                              │
                      ┌───────────────────────┴───────────────────────┐
                      ▼                                               ▼
         [If Duration <= 180s]                             [If Duration > 180s]
          Direct Single Stream                            Smart 120s Chunking
                      │                                               │
                      ▼                                               ▼
     FFmpeg Raw Video Pipe (bgr24)                  N Slices via Frame-Accurate Slicing
                      │                                               │
                      ▼                                               ▼
     PyTorch Real-ESRGAN Tensor                     Sequential In-Memory Pipe Upscaling
         (NVIDIA T4 GPU)                              (torch.cuda.empty_cache per slice)
                      │                                               │
                      ▼                                               ▼
   FFmpeg Encoder Pipe (libx264/5)                     Lossless Concatenation Demuxer
                      │                             (ffmpeg -f concat -safe 0 -c copy)
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                                 Final Master Muxing Engine
                             (Audio Track re-attached + Gain applied)
                                  -movflags +faststart
                                              │
                                              ▼
                                    Master 4K Video (.mp4)
```

---

## 1. Zero-Disk In-Memory Pipe Streaming

### The Problem with Traditional Implementations:
By default, Real-ESRGAN's CLI uses `--extract_frame_first` to prevent multiprocessing deadlocks. However, this extracts all input frames as uncompressed `.png` files to disk:
* A 5-minute 1080p video at 30 FPS consists of $5 \times 60 \times 30 = 9,000$ frames.
* 9,000 uncompressed 1080p PNG frames $\approx 22.5\text{ GB}$.
* 9,000 upscaled 4K PNG frames $\approx 110\text{ GB}$.
* Total storage: $\approx 132.5\text{ GB}$.
* **Google Colab Free Storage limit**: $\approx 25\text{–}35\text{ GB}$ available space.
* Consequently, any video exceeding 3–5 minutes crashed with `[Errno 28] No space left on device`.

### The In-Memory Pipe Solution:
OmniUpscaler eliminates the `--extract_frame_first` flag and uses direct streaming UNIX pipes:
1. **Input Reader**: FFmpeg reads the input `.mp4` and streams raw uncompressed `bgr24` pixel buffers directly into standard output (`stdout`).
2. **GPU Processor**: Python consumes the byte buffer in exact single-frame chunks (`width × height × 3`), converts it to a PyTorch tensor on CUDA memory, applies the model pass, and retrieves the upscaled tensor.
3. **Output Writer**: Python writes the processed raw byte buffer straight into FFmpeg's standard input (`stdin`) pipe for immediate H.264 / H.265 compression.
4. **Result**: Zero temporary image files on disk. 99% storage reduction.

---

## 2. Smart 120-Second Segment Chunking

For long-duration videos (10 minutes, 30 minutes, 60+ minutes), running a single continuous streaming pipe exposes the pipeline to two potential hazards:
* **Gradio WebSocket Dropouts**: Single HTTP/WS requests held open for 45+ minutes can drop.
* **CUDA VRAM Memory Fragmentation**: Continuous tiling over 10,000+ frames can cause driver-level memory leaks or CUDA memory fragmentation.

### Solution:
When the input duration exceeds 180 seconds, OmniUpscaler automatically segments the normalized video into 120-second segments:
```python
segment_sec = 120.0
num_segs = math.ceil(total_dur / segment_sec)
for s_idx in range(num_segs):
    s_start = s_idx * segment_sec
    s_len = min(segment_sec, total_dur - s_start)
    # Extract segment losslessly with audio stripped
    subprocess.run(["ffmpeg", "-y", "-ss", str(s_start), "-i", input_file, "-t", str(s_len), "-c:v", "libx264", "-crf", "14", "-an", s_file])
```
* **VRAM Reset**: Between segments, `torch.cuda.empty_cache()` is called, and subprocess teardown releases all CUDA buffers.
* **Lossless Reassembly**: When all segments finish, they are joined instantly without quality loss using FFmpeg's concat demuxer:
  ```bash
  ffmpeg -y -f concat -safe 0 -i concat_manifest.txt -c copy concat_master.mp4
  ```

---

## 3. PyTorch 2.6 Weights Unpickler Patch

In PyTorch 2.6+, `torch.load()` defaulted `weights_only=True` for security reasons. Legacy Real-ESRGAN and BasicSR checkpoints rely on serialized class metadata and fail to load with:
`_pickle.UnpicklingError: Weights only load failed.`

OmniUpscaler injects a compatibility wrapper:
```python
_orig_torch_load = torch.load
def _safe_torch_load(*args, **kwargs):
    if "weights_only" not in kwargs:
        kwargs["weights_only"] = False
    return _orig_torch_load(*args, **kwargs)
torch.load = _safe_torch_load
```
Additionally, `TORCH_FORCE_WEIGHTS_ONLY_LOAD=0` is exported in the subprocess environment.

---

## 4. Python 3.13 Type Union Safeguard

Modern Python runtimes (Python 3.10+) use PEP 604 type unions (`A | B`). In standard PyTorch, internal modules like `torch.utils._device` use:
```python
CURRENT_DEVICE: torch.device | None = None
```
Overriding `torch.device` with a wrapper function causes `TypeError: unsupported operand type(s) for |: 'function' and 'NoneType'`. OmniUpscaler eliminates function monkey-patching and preserves native class references while ensuring seamless CUDA $\rightarrow$ CPU fallbacks.
