# RipSafe

**PDF sanitization, flattening, rasterization & minimal PDF reconstruction for enterprise printing pipelines.**

[![License](https://img.shields.io/badge/license-MIT%20OR%20Apache--2.0-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.85%2B-orange.svg)](https://www.rust-lang.org)

---

## The Problem

Enterprise printing systems and Raster Image Processors (RIPs) like Xerox FreeFlow, EFI Fiery, and HP Indigo frequently hang, run out of memory, or crash when processing complex enterprise PDFs.

These problematic files typically feature:
- 40,000+ pages and gigabytes in size
- Deeply nested Form XObjects
- Complex PDF transparency groups and soft masks
- Optional Content Groups (OCG / layers)
- Complex AcroForm annotations and interactive fields
- Broken xref tables and incremental revisions
- Obscure, unhinted, or corrupted embedded fonts

## The RipSafe Solution

RipSafe does not try to "repair" broken PDF object graphs.

> **Fundamental Principle**: *Do not repair complicated page structures when simply replacing them is safer.*

RipSafe flattens visible appearances into a pristine raster image and reconstructs a completely new, intentionally boring, 100% print-compatible PDF from scratch:

```text
INPUT PDF
   ↓
OPEN & VALIDATE
   ↓
MULTI-PROCESS RENDER & FLATTEN
   ↓
COMPRESS DIRECTLY TO ARTIFACT
   ↓
STREAMING INCREMENTAL PDF ASSEMBLER
   ↓
DELETE TEMPORARY ARTIFACTS
   ↓
STRICT POST-ASSEMBLY VALIDATION
   ↓
OUTPUT PDF
```

The reconstructed PDF contains **only**:
- A minimal Catalog
- A balanced `/Type /Pages` tree
- Clean `Page` objects with `/MediaBox`, `/Resources` (containing only the raster `/XObject`), and `/Contents` (calling `/Im0 Do`).
- Zero original annotations, AcroForms, transparency groups, or font structures survive.

---

## Quick Start

### Basic Conversion
```bash
ripsafe input.pdf -o output.pdf
```
*Defaults to print-safe settings: 300 DPI, RGB, JPEG quality 92, and automatically scales workers to your CPU count.*

### High-Resolution Print Prep (600 DPI, 8 Workers)
```bash
ripsafe input.pdf -o output.pdf --dpi 600 --workers 8
```

### Grayscale / Monochrome
```bash
ripsafe input.pdf -o output.pdf --color gray --dpi 300
```

### Lossless Compression Mode
```bash
ripsafe input.pdf -o output.pdf --compression lossless
```

### Resume an Interrupted Job
```bash
ripsafe input.pdf -o output.pdf --resume
```

---

## Key Features

- **Bounded $O(1)$ Memory Usage**: Page images are streamed directly from disk into the output file and unlinked immediately. RipSafe never accumulates pages in RAM.
- **Isolated Multi-Process Worker Pool**: Child worker processes isolate PDFium native rendering. If a corrupt page causes an engine crash, the parent supervisor catches it, retries, and protects the pipeline.
- **Hierarchical Page Tree**: Automatically structures `/Type /Pages` into a balanced B-tree (branching factor 64) for large documents ($> 64$ pages), preventing RIP parser stack overflows.
- **Stateful Resumability**: Tracks completed pages in an atomic state file (`<output>.ripsafe-state.json`) with sampled and full SHA-256 fingerprint validation.
- **Strict Structural Validator**: Validates the generated PDF structure post-assembly, guaranteeing that no `/Annots`, `/Group`, or `/AcroForm` elements survive.
- **Environment Diagnostics (`doctor`)**: Inspects PDFium dynamic libraries, write permissions, and CPU recommendations.
- **Built-in Concurrency Benchmark**: Empirically measures throughput across worker counts.

---

## CLI Reference

```text
Usage: ripsafe.exe [OPTIONS] [INPUT] [COMMAND]

Commands:
  doctor     Perform environment diagnostic checks for PDFium, CPUs, and disk access
  benchmark  Benchmark rendering performance across varying worker counts
  help       Print this message or the help of the given subcommand(s)

Arguments:
  [INPUT]  Path to input PDF file

Options:
  -o, --output <OUTPUT>              Path to output sanitized PDF file
      --dpi <DPI>                    Rendering DPI (dots per inch) [default: 300]
  -w, --workers <WORKERS>            Number of concurrent rendering worker processes
      --color <COLOR>                Color mode: rgb or gray [default: rgb]
      --jpeg-quality <JPEG_QUALITY>  JPEG compression quality (1-100) [default: 92]
      --compression <COMPRESSION>    Compression format: jpeg or lossless [default: jpeg]
      --resume                       Resume an interrupted job using existing state file
      --retry <RETRY>                Number of retries per page if rendering fails [default: 0]
  -q, --quiet                        Suppress progress display
      --progress <PROGRESS>          Progress reporting mode: text or json [default: text]
      --keep-temp                    Keep temporary image artifacts after assembly
      --temp-dir <TEMP_DIR>          Custom temporary directory path
      --validate <VALIDATE>          Validation level: standard, strict, or skip [default: standard]
      --full-hash                    Compute full cryptographic SHA-256 for input fingerprinting
      --pdfium-path <PDFIUM_PATH>    Explicit path to PDFium native library
  -v, --verbose...                   Verbose logging level (-v, -vv, -vvv)
  -h, --help                         Print help
  -V, --version                      Print version
```

---

## Diagnostic Check (`doctor`)

To verify your environment and renderer status:

```bash
ripsafe doctor
```

Example output:
```text
=== RipSafe System Diagnostics (Doctor) ===
RipSafe Version:        0.1.0
Target Architecture:    windows-x86_64
Detected CPU Cores:     12
Recommended Workers:    12
Expected Library Name:  pdfium.dll
PDFium Discovery:       FOUND
PDFium Library Path:    "C:\Users\...\.cache\pdfium-bundled\pdfium-7881\pdfium.dll"
System Temp Directory:  "C:\Users\...\AppData\Local\Temp\"
Temp Dir Write Access:  OK (verified)
Max Page Dimensions:   20000x20000 pixels
Max Total Pixels:       100000000 pixels (~400 MiB uncompressed buffer)
===========================================
```

---

## Concurrency Benchmark

```bash
ripsafe benchmark large_document.pdf --workers 1,2,4,8 --dpi 150
```

Example output:
```text
=== RipSafe Concurrency Benchmark ===
Input Document: "large_document.pdf"
Target DPI:     150
Max Pages:      20
Testing Workers: [1, 2, 4]

| Workers | Pages | Time (s) | Pages/sec | Scaling | Output Size |
|---------|-------|----------|-----------|---------|-------------|
|       1 |    12 |     3.33 |      3.60 | 1.00x   |      0.71 MiB |
|       2 |    12 |     1.89 |      6.34 | 1.76x   |      0.71 MiB |
|       4 |    12 |     1.14 |     10.52 | 2.92x   |      0.71 MiB |
=====================================
```

---

## Licensing & Native Dependencies

- **RipSafe**: MIT OR Apache-2.0.
- **PDFium**: Apache 2.0 / BSD 3-Clause (Google Chromium project). Safe for commercial deployment.
- **No AGPL Dependencies**: RipSafe intentionally avoids AGPL rendering backends (such as MuPDF) in its default distribution.
