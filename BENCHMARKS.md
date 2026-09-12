# RipSafe - Benchmark Suite & Performance History

This document records empirical performance benchmarks across hardware configurations, worker counts, and document types.

---

## Benchmark Methodology
- All tests are executed using release builds (`cargo build --release`).
- Benchmarks measure end-to-end performance including PDF reading, rasterization, JPEG/lossless compression, ordering, streaming PDF generation, and validation.
- Concurrency scaling is evaluated across worker configurations (1, 2, 4, 8, 12 workers).

---

## Historical Benchmark Log

### Benchmark Run #1: Multi-Worker Scaling Test
- **Date**: 2026-09-12
- **Version / Commit**: v0.1.0 (Initial Production Release)
- **Environment**:
  - **OS**: Windows 11 Pro x86_64
  - **CPU**: 12 Logical Cores (AMD/Intel 64-bit Native)
  - **Rust Version**: rustc 1.98.1
- **Renderer Backend**: Google PDFium (Chromium build 7881 via `pdfium-bundled`)
- **Input Characteristics**: Synthetic multi-page document with vector paths, coordinate transformations, and color fills
- **Page Count**: 12 pages
- **Resolution**: 150 DPI
- **Color Mode**: RGB (`/DeviceRGB`)
- **Compression**: JPEG (Quality 92, `/DCTDecode`)

#### Results Table
| Workers | Pages | Total Time (s) | Throughput (pages/sec) | Scaling Factor | Output Size | Status |
|:-------:|:-----:|:--------------:|:----------------------:|:--------------:|:-----------:|:------:|
| 1       | 12    | 3.33s          | 3.60 pages/s           | 1.00x          | 0.71 MiB    | Passed |
| 2       | 12    | 1.89s          | 6.34 pages/s           | 1.76x          | 0.71 MiB    | Passed |
| 4       | 12    | 1.14s          | 10.52 pages/s          | 2.92x          | 0.71 MiB    | Passed |

#### Analysis & Notes
- **Throughput Scaling**: Performance scales from 3.60 pages/sec on 1 worker to 10.52 pages/sec on 4 workers, demonstrating a **2.92x speedup**.
- **Deterministic Output**: Output size remains byte-identical (0.71 MiB) regardless of worker count and page completion order.
- **Memory Boundedness**: Memory usage per worker remained below 45 MiB throughout the test. Temporary image artifacts were deleted immediately upon assembly.

---

### Benchmark Run #2: Real-World Industry Standard Prepress Test (`eci_altona-test-suite.pdf`)
- **Date**: 2026-09-12
- **Version / Commit**: v0.1.0
- **Environment**: Windows 11 Pro x86_64, 12 Cores, Rust 1.98.1
- **Renderer Backend**: Google PDFium (Chromium build 7881)
- **Input File**: `eci_altona-test-suite.pdf` (Industry benchmark prepress test with heavy transparency, nested XObjects, spot colors, and overprint)
- **Source Size**: 121.8 MiB (127,724,771 bytes)
- **Page Count**: 17 pages
- **Resolution**: 300 DPI
- **Color Mode**: RGB (`/DeviceRGB`)
- **Compression**: JPEG (Quality 92, `/DCTDecode`)
- **Workers**: 12

#### Results
- **Processing Time**: **1.06 seconds**
- **Throughput**: **16.0 pages/sec**
- **Output Size**: **3.37 MiB** (3,535,808 bytes) — **97.2% reduction** from original bloated file size!
- **Validation**: **PASSED** (all 17 pages verified, strict simplicity validated, zero broken/bloated structures survive).

---

## How to Run Benchmarks

To benchmark your own PDF files on your hardware:

```bash
# Benchmark 1, 2, 4, and 8 workers at 300 DPI
ripsafe benchmark large_document.pdf --workers 1,2,4,8 --dpi 300

# Benchmark specific page range
ripsafe benchmark large_document.pdf --workers 2,4,8 --max-pages 50
```
