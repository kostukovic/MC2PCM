# MC2PCM

**Multi-Camera-to-Point-Cloud Module**
*Calibrated, low-latency 3D capture from 4–8 mono cameras with optional depth; streaming sparse/dense point clouds and 3D keypoints in real time.*

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](#license) [![Status: Core MVP](https://img.shields.io/badge/status-Core_MVP_planned-yellow)](#roadmap)

---

## What is MC2PCM?

**MC2PCM** turns **synchronized multi-camera video** into **metric 3D** in real time:

* **Capture & sync:** 4–8 global-shutter mono cameras (GigE/USB3). Optional depth sensors. PTP/trigger sync.
* **Calibrate:** Intrinsics (ChArUco/Checkerboard), extrinsics (multi-pose grid), thermo-drift checks.
* **Reconstruct:** Multi-view matching + **triangulation** → **3D keypoints** and **sparse/dense point clouds**.
* **Stream & view:** Zero-copy GPU path (GStreamer → DMABUF → CUDA → Vulkan) + gRPC streaming to clients.
* **Integrate:** FlatBuffers messages, plug-in adapters; feeds **MotionCoder** or any 3D consumer.

Target domains: **CAD/PLM**, **VFX/animation**, **robotics/HMI**, **biomechanics**, **assistive tech**.

---

## Key features

* **Deterministic latency:** GPU pipeline with **zero-copy** handoffs (DMABUF).
* **Accurate 3D:** Calibrated multi-view **triangulation**; optional depth fusion (SGM / learned stereo / TSDF preview).
* **Flexible outputs:** **3D keypoints**, **ROI point patches**, preview **point clouds**, PLY/PCD/GLB export.
* **Robustness tools:** One-Euro/Kalman filters, RANSAC refine, bundle adjustment (Ceres).
* **Portable I/O:** gRPC/IPC, FlatBuffers schema, CLI and lightweight viewer (Vulkan).

---

## Architecture (high level)

```
Cameras (4–8) ──> Capture (GStreamer, PTP/Trigger) ──> GPU ingest (DMABUF)
          │                                              │
          └─> Depth (optional) ──────────────────────────┘
                │
Calib (OpenCV: intrinsics/extrinsics) ──> Multi-view matching (CUDA/LightGlue-like)
                                           │
                                       Triangulation ──> 3D keypoints
                                           │
                                 Sparse/Dense fusion (SGM/TSDF)
                                           │
                     Vulkan preview  ──>  gRPC/FlatBuffers  ──>  Clients (e.g. MotionCoder)
```

---

## Quick start (Linux)

> **Prereqs:** Ubuntu LTS, NVIDIA driver/CUDA 12+, CMake ≥3.20, GStreamer (with NV plugins), OpenCV (contrib), Vulkan SDK, Ceres, Protobuf/FlatBuffers. A recent NVIDIA GPU is recommended.

```bash
# 1) Clone
git clone https://github.com/<you>/mc2pcm.git
cd mc2pcm

# 2) Configure & build
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j

# 3) Run capture (example with 4 cameras via GStreamer)
./build/bin/mc2pcm_capture \
  --pipeline configs/gst_4cam_ptp.yaml \
  --sync ptp \
  --out /tmp/mc2pcm_stream

# 4) Calibrate (record, then solve intrinsics/extrinsics)
./build/bin/mc2pcm_calib \
  --images data/calib/*/*.png \
  --board chAruco_A5.yaml \
  --save out/calib.yaml

# 5) Triangulate & preview
./build/bin/mc2pcm_triangulate \
  --calib out/calib.yaml \
  --input /tmp/mc2pcm_stream \
  --preview
```

**gRPC stream to a client (e.g., MotionCoder):**

```bash
./build/bin/mc2pcm_server \
  --calib out/calib.yaml \
  --roi configs/roi_hands.yaml \
  --bind 0.0.0.0:50051
```

---

## Repository layout (suggested)

```
mc2pcm/
  cmake/
  configs/
    gst_*.yaml           # capture pipelines
    roi_*.yaml           # ROI patch definitions
    calib/*.yaml         # board presets (ChArUco, checker)
  data/
    calib/               # sample calibration sets (optional)
  include/
  src/
    capture/             # GStreamer ingest, PTP/trigger, timestamping
    calib/               # OpenCV intrinsics/extrinsics, drift
    match/               # feature extract/match (CUDA-accelerated), RANSAC
    triangulate/         # multi-view triangulation, confidence
    stereo/              # SGM / learned depth, TSDF preview
    filter/              # temporal filters, BA hooks (Ceres)
    io/                  # FlatBuffers schema, gRPC/IPC, PLY/PCD/GLB
    viewer/              # Vulkan previewer
  tools/
    record_calib.cpp     # record grids/boards
    export_cloud.cpp     # to PLY/PCD/GLB
  docs/
    data_flow.md
    calib_guide.md
    performance_tips.md
  tests/
  LICENSE
```

---

## Calibration guide (essentials)

1. **Intrinsics:**

   * Print **ChArUco** or checkerboard; cover the full FOV at multiple distances/angles.
   * Use `mc2pcm_calib` to solve fx/fy/cx/cy + distortion (radial/tangential).

2. **Extrinsics:**

   * Show a **multi-pose grid** (e.g., AprilTag/ChArUco wall) to all cameras.
   * Solve relative poses; verify **reprojection ≤ 0.3 px**.

3. **Sync & thermo drift:**

   * Prefer **hardware trigger** or **PTP** on a PoE/PTP switch.
   * Warm up sensors 10–15 min; run drift check (recalibrate if needed).

---

## I/O formats

* **Stream:** FlatBuffers over **gRPC** (point clouds, keypoints, timestamps, camera poses).
* **Files:** PLY/PCD/GLB (clouds), JSON/CSV (keypoints/skeletons), NPZ (test sets).
* **Viewer:** Vulkan preview (LOD, frustum culling), point color = view consistency or confidence.

---

## Performance targets (baseline)

* **Latency:** camera → cloud preview **≤ 90 ms** (P95) with 4 cams @ 720p.
* **Rate:** **15–30 FPS** depending on pipeline (matching/stereo density).
* **Accuracy:** depth σᶻ **≤ 5 mm @ 2 m** (well-lit scene, proper baseline).
* **Sync:** inter-camera skew **< ±1 ms** (PTP or trigger).

> Tips: use global-shutter lenses, consistent exposure, adequate baseline (scene-dependent), avoid rolling-shutter skew.

---

## Integration with MotionCoder

MC2PCM can feed **3D keypoints** and **ROI patches** directly to **MotionCoder**:

* **Why:** LLM semantics benefit from **true 3D** (metric pose, contact geometry).
* **How:** enable the **ROI extractor** to crop small local patches around hands/tools; publish via gRPC.
* **What:** MotionCoder consumes **keypoints + ROI** to emit **JSON/DSL commands** for CAD/DCC.

---

## Roadmap

* **MVP:** 4-cam capture, calib, triangulation, Vulkan preview, FlatBuffers I/O.
* **+Depth:** SGM / learned stereo, TSDF preview, ROI downsampling.
* **Perf:** full **zero-copy** path, quantization, multi-GPU scheduling.
* **Robustness:** auto-recalib hints, jitter mitigation, logging/telemetry.
* **UX:** GUI calib assistant, packaged viewer, Docker base images.

See GitHub **issues** for granular milestones.

---

## Build options (CMake)

```bash
cmake -B build \
  -DUSE_CUDA=ON \
  -DUSE_VULKAN=ON \
  -DUSE_GSTREAMER=ON \
  -DUSE_CERES=ON \
  -DUSE_SGM=ON
```

---

## Troubleshooting

* **Dropped frames:** check NIC/PCIe bandwidth, enable jumbo frames, pin CPU cores for capture.
* **Bad 3D accuracy:** verify **intrinsics** first, then **extrinsics**; increase grid poses; check lens focus.
* **High latency:** confirm zero-copy (DMABUF), disable needless color converts/scales; prefer FP16.
* **Jitter:** ensure **PTP lock**, stable exposure/gain; warm up sensors.
* **Sparse matches:** raise light, add texture, adjust matcher thresholds.

---

## Contributing

PRs welcome! Please include:

* clear description & perf numbers,
* tests for new modules,
* docs updates (calibration/perf notes).

---

## License

**Apache-2.0** — see [`LICENSE`](./LICENSE).

---

## Related

* **MotionCoder** — LLM-powered motion semantics from keypoints/ROI → JSON/DSL commands.
* **AnimalMotionCoder** — synthetic motion set for encoder bootstrapping.

---
