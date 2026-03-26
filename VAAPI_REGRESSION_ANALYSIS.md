# VAAPI Regression Analysis: v0.16.4 -> v0.17.x on Intel J4105

## Problem

After upgrading Frigate from 0.16.4 to 0.17.0/0.17.1, CPU usage jumps from ~50% to ~90-99% on Intel Celeron J4105 (Gemini Lake, Gen8 iGPU).
FFmpeg appears to fall back to software decoding instead of using VAAPI hardware acceleration.

Related discussion: https://github.com/blakeblackshear/frigate/discussions/22424

## Test Matrix

### v0.16.4

| ffmpeg | preset               | driver | result |
|--------|----------------------|--------|--------|
| 5.0    | preset-vaapi         | iHD    | bad    |
| 5.0    | preset-intel-qsv-h264| iHD    | bad    |
| 7.0    | preset-vaapi         | iHD    | bad    |
| 7.0    | preset-intel-qsv-h264| iHD    | bad    |
| 5.0    | preset-vaapi         | i965   | good   |
| 5.0    | preset-intel-qsv-h264| i965   | bad    |
| 7.0    | preset-vaapi         | i965   | good   |
| 7.0    | preset-intel-qsv-h264| i965   | bad    |

### v0.17.0 / v0.17.1

| ffmpeg | preset               | driver | result |
|--------|----------------------|--------|--------|
| 5.0    | preset-vaapi         | iHD    | bad    |
| 5.0    | preset-intel-qsv-h264| iHD    | bad    |
| 7.0    | preset-vaapi         | iHD    | bad    |
| 7.0    | preset-intel-qsv-h264| iHD    | bad    |
| 5.0    | preset-vaapi         | i965   | bad    |
| 5.0    | preset-intel-qsv-h264| i965   | bad    |
| 7.0    | preset-vaapi         | i965   | bad    |
| 7.0    | preset-intel-qsv-h264| i965   | bad    |

**The only working combination (`preset-vaapi + i965`) broke in v0.17.x.**

## Root Causes

### 1. Unpinned `intel-media-va-driver-non-free` (Primary)

In `docker/main/install_deps.sh`, the Intel VA-API driver package was unpinned:

- **v0.16.4**: `intel-media-va-driver-non-free=24.3.3-996~22.04` (pinned)
- **v0.17.x**: `intel-media-va-driver-non-free` (unpinned, pulls latest)

Even when using the `i965` driver (`LIBVA_DRIVER_NAME=i965`), the iHD package
installs shared VA-API infrastructure libraries. A newer version can change
shared library versions that the i965 driver also depends on. Additionally,
`libigdgmm12` is now explicitly installed at version `22.5.5` alongside both
legacy and standard Intel compute-runtime packages, which may introduce
library conflicts on older Gen8 hardware.

### 2. OpenVINO 2025.3 GPU Enumeration (Secondary)

A new `frigate/detectors/detection_runners.py` imports OpenVINO at module level
and calls `ov.Core()` to enumerate GPU/NPU devices -- even when using a Coral TPU.
OpenVINO was bumped from `2024.4` to `2025.3`. On the J4105's constrained UHD 600,
the GPU plugin initialization may interfere with ffmpeg's VAAPI initialization.

### 3. Increased Baseline CPU Load (Contributing)

v0.17.x adds several CPU-intensive features enabled by default:
- Stationary object classifier (`detect.stationary.classifier: true`)
- TensorFlow CPU bundled (`tensorflow-cpu==2.19`)
- New post-processors (audio transcription, object descriptions, etc.)
- MemryX runtime with additional shared libraries

## Fix

Re-pin `intel-media-va-driver-non-free` to the known-good version from v0.16.4
(`24.3.3-996~22.04`) to restore VAAPI functionality on legacy Intel GPUs.
