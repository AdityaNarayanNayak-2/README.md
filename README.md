# NISAR Pro (`sar_science_processor`) 🛰️⚡
**Memory-Safe, Sub-Second L/C/X-Band SAR Flood Inundation & 13-Stage PS-InSAR Engine in Pure Rust**

[![Rust](https://img.shields.io/badge/Rust-1.82%2B-orange?logo=rust)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-MIT%2FApache--2.0-blue)](#)
[![RAM](https://img.shields.io/badge/Peak_RAM-95.3_MB-brightgreen)](#12-part-scientific--systems-benchmark)
[![Parity](https://img.shields.io/badge/NASA%2FISRO_Parity-1.000000_IoU-00e5ff)](#12-part-scientific--systems-benchmark)

**NISAR Pro** is a high-performance, edge-deployable Synthetic Aperture Radar (SAR) processing engine and geospatial command workstation designed for **NASA-ISRO NISAR (L-Band)**, **Sentinel-1 (C-Band)**, and **ICEYE/TerraSAR-X (X-Band)** products.

Standard Python/GDAL SAR workflows require 4+ GB container images, heavy RAM footprints, and tens of seconds per tile. `sar_science_processor` compiles into a **single 6.5 MB native Rust binary** with **zero C/C++ GDAL dependencies**, executing full dual-date L2 GCOV flood mapping in **0.77 seconds** inside **95.3 MB of RAM**.

---

## 🏗️ Key Architectural Highlights

1. **HDF5 Hyperslab Crop-Before-Load & Lattice Phase Alignment:**
   Slices target AOIs (`1x1km` to `20x20km`) directly from 8+ GB NISAR HDF5 granules without loading full swaths into memory. Automatically detects resolution mismatches (e.g., 10 m DHDH active vs. 20 m QPDH baseline), verifies UTM lattice nesting, and applies phase-aligned complex multilooking.
2. **7-Stage Calibrated Flood & Inundation Engine (`src/flood_detect.rs`):**
   - **NESZ Swath-Boundary Filtering:** Rejects out-of-swath feathering floats (`< -40.0 dB` / $10^{-4}$ linear power) prior to water classification.
   - **Speckle Reduction:** Valid-mask-aware $3\times3$ median and Lee Sigma ($7\times7$) + Frost ($5\times5$) edge-preserving filter chains.
   - **Permanent Water Exclusion & Bounded Otsu:** Masks persistent water bodies (`< -13.0 dB` on both passes) and sensor dropouts (`< -25.0 dB`) from the 256-bin log-ratio change histogram so bimodal splits lock onto genuine inundation events.
   - **8-Connected Hysteresis Region Growing & Morphology:** Dual-threshold BFS queue traversal (`seed_thresh`, `growth_thresh`) paired with connected-component filtering and optional GUNW interferometric coherence fusion.
3. **Zero-Dependency Tiled GeoTIFF & Scanline-Merged GeoJSON Encoder (`src/io.rs`):**
   - Writes cloud-ready **256×256 tiled GeoTIFFs (`EPSG:4326`)** directly to raw little-endian byte buffers (`ModelPixelScaleTag 33550`, `ModelTiepointTag 33922`, `GeoKeyDirectoryTag 34735`) without linking `libgdal`.
   - Vectorizes flood clusters using 8-connected BFS + horizontal scanline run-length merging, compressing vector outputs from **244 MB (321,226 pixel boxes) $\rightarrow$ 37 KB (30 MultiPolygons)**.
4. **13-Stage Memory-Safe PS-InSAR Displacement Pipeline:**
   - Patch-based complex coregistration $\rightarrow$ Early multilooking $\rightarrow$ Interferogram formation $\rightarrow$ $O(N^2)$ Summed-Area-Table (SAT) coherence $\rightarrow$ Goldstein adaptive phase filter $\rightarrow$ Quality-guided flood-fill unwrapping $\rightarrow$ Topographic phase & 2D orbital ramp removal $\rightarrow$ Millimeter-scale LOS persistent scatterer health scoring.

---

## 📊 12-Part Scientific & Systems Benchmark
**Test Scene:** NISAR L2 GCOV L-Band HH Pair over Brahmaputra Valley, Assam, India (`27.2055°N, 94.8948°E`, `20×20 km` / `1000×1000 px`, Baseline: `2026-07-12`, Active: `2026-08-17`).  
**Hardware:** Red Hat Enterprise Linux 10.2 (Kernel 6.12.0) | Intel Core i5-6500T (4C/4T @ 2.50 GHz) | 15 GiB RAM.  
**Reference Stack:** NASA OPERA / ASF HyP3 / ISRO NRSC equivalent in Python (`h5py`, `NumPy`, `SciPy.ndimage`, `scikit-image`, `Rasterio`, `Shapely`).

| # | Benchmark Dimension | Rust Engine (`sar_science_processor`) | NASA/ISRO Python Reference | Speedup / Improvement |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **Cold-Start & Import Latency** | **1.83 ms** (± 0.10) | 646.04 ms (± 55.67) | **352.3× faster startup** |
| **2** | **HDF5 Hyperslab I/O + 2×2 Multilook** | **135.00 ms** (± 1.13) | 104.23 ms (± 2.38) | Includes UTM lattice alignment |
| **3** | **Linear Power to Decibel (dB)** | **23.34 ms** (± 0.61) | 25.21 ms (± 0.29) | **1.08× faster** |
| **4** | **3×3 Speckle Filtering** | **279.09 ms** (3.58 MPx/s) | 330.84 ms (3.02 MPx/s) | **1.19× higher throughput** |
| **5** | **Permanent Water + 256-Bin Otsu** | **16.77 ms** (± 0.13) | 9.28 ms (± 0.41) | Sub-17ms histogram split |
| **6** | **Dual-Threshold 8-Conn Region Grow** | **2.92 ms** (± 0.10) | 13.33 ms (± 1.79) | **4.6× faster BFS queue** |
| **7** | **Morphology & Component Filter** | **8.08 ms** (± 0.66) | 40.56 ms (± 11.40) | **5.0× faster cleanup** |
| **8** | **Geospatial Export (TIF+PNG+GeoJSON)** | **39.04 ms** (± 0.78) | 253.64 ms (± 25.04) | **6.5× faster zero-GDAL I/O** |
| **9** | **End-to-End Wall-Clock Time** | **0.77 s** | 2.22 s | **2.88× faster end-to-end** |
| **10** | **Peak Physical Memory (RSS)** | **95.3 MB** | 200.1 MB | **2.1× lower RAM** (32 streams/8GB) |
| **11** | **Deployment Footprint** | **6.5 MB binary** | 445.0 MB venv | **68.4× smaller artifact** |
| **12** | **Scientific & Numerical Parity** | **IoU = 1.000000, F1 = 1.000000** | $\Delta T_{\text{Otsu}} = \mathbf{0.000001\text{ dB}}$ | **100.00% exact flood match** |

### Reproducing the Benchmarks
```bash
cargo build --release
./benchmarks/run_12_benchmarks.sh
