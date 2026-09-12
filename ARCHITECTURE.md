# RipSafe Architecture Specification

RipSafe is designed to solve a fundamental challenge in high-volume enterprise printing: processing massive (up to 40,000+ page), bloated, or malformed PDFs that choke printer Raster Image Processors (RIPs).

This document details the internal systems architecture, concurrency model, memory boundaries, and structural invariants.

---

## 1. High-Level Process Model

RipSafe employs a multi-process architecture to isolate CPU-intensive PDF rendering from document orchestration and assembly:

```text
                        ┌──────────────────────────────┐
                        │        RipSafe Parent        │
                        │    (Orchestrator & Assembler)│
                        └──────────────┬───────────────┘
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
   ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
   │ Worker Child 0  │        │ Worker Child 1  │        │ Worker Child N  │
   │ (ripsafe worker)│        │ (ripsafe worker)│        │ (ripsafe worker)│
   │  PDFium Native  │        │  PDFium Native  │        │  PDFium Native  │
   └────────┬────────┘        └────────┬────────┘        └────────┬────────┘
            │                          │                          │
            ▼                          ▼                          ▼
   ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
   │ page_000000.img │        │ page_000001.img │        │ page_000002.img │
   └────────┬────────┘        └────────┬────────┘        └────────┬────────┘
            │                          │                          │
            └──────────────────────────┼──────────────────────────┘
                                       ▼
                        ┌──────────────────────────────┐
                        │      Reorder Buffer &        │
                        │   StreamingPdfWriter         │
                        └──────────────┬───────────────┘
                                       ▼
                        ┌──────────────────────────────┐
                        │      output.pdf.partial      │
                        │              ↓               │
                        │          output.pdf          │
                        └──────────────────────────────┘
```

### Why Multi-Process?
1. **Engine Crash Isolation**: C++ native rendering engines (e.g. PDFium) can segfault on corrupted fonts, broken xref tables, or decompression bombs. Running in child processes ensures a worker crash never brings down the parent orchestrator.
2. **True Memory Reclamation**: Operating systems reclaim 100% of memory, open file handles, and native allocator pools upon process termination.
3. **Thread-Safety Guarantees**: PDFium contains internal static state that limits concurrent multi-threaded execution within a single address space. Multi-process execution eliminates thread lock contention.

---

## 2. Parent / Worker IPC Protocol

Communication between parent and worker child processes occurs over standard I/O streams (`stdin` and `stdout`) using Newline-Delimited JSON (NDJSON):

### Parent -> Worker (`stdin`)
- `Init`: Transmits input file path, target DPI, color mode, compression mode, and optional custom PDFium path.
- `RenderPage`: Requests rendering of a 0-based page index with 1-based human page number, job ID, and destination disk path for the compressed raster artifact.
- `Shutdown`: Commands the worker to terminate gracefully.

### Worker -> Parent (`stdout`)
- `Ready`: Informs the parent that the document is open and reports total page count.
- `PageSuccess`: Reports page index, physical dimensions in points (`width_pt`, `height_pt`), rendered pixel dimensions, bytes written to disk, and rendering duration.
- `PageFailure`: Reports page index, human page number, and error details.
- `FatalError`: Reports an unrecoverable initialization error.

---

## 3. Bounded-Memory Lifecycle & Artifact Management

RipSafe guarantees **$O(1)$ memory consumption** with respect to total document size and page count:

```text
[Page Job Dispatched]
         ↓
[Worker Renders Page to RawBitmap in RAM]
         ↓
[Worker Compresses Pixels Directly to Temp Disk Artifact]
         ↓
[RawBitmap Dropped & Freed in Worker RAM]
         ↓
[Parent Receives PageSuccess with File Reference]
         ↓
[Assembler Streams File Directly into output.pdf.partial]
         ↓
[Temporary Disk Artifact Deleted Immediately]
```

### Key Invariants:
1. **At most $2 \times W$ jobs in flight**: The orchestrator limits outstanding jobs to twice the worker count, preventing unbounded temporary disk or queue growth.
2. **Immediate Cleanup**: Once page $K$ is appended to the output PDF, its temporary artifact (`page_00000K.img`) is deleted immediately.
3. **No In-Memory PDF Graph**: Output pages and image streams are not accumulated in RAM. Only 8-byte object file offsets (`Vec<u64>`) are retained in memory.

---

## 4. Bounded Reorder Mechanism

Workers may finish rendering out of order (e.g. page 3 finishes before page 1).
The parent maintains a `ReorderBuffer`:
- Tracks `next_page_to_write` (starting at page 0).
- Incoming `PageSuccess` events are placed into a `BTreeMap<u32, CompletedPageInfo>`.
- While `reorder_buffer.contains_key(&next_page_to_write)`:
  - Pops the next sequential page.
  - Appends to `StreamingPdfWriter`.
  - Unlinks the temporary image artifact.
  - Increments `next_page_to_write`.

This ensures deterministic, sequential page ordering in the final output file without storing uncompressed bitmaps in memory.

---

## 5. Streaming PDF Assembly & Balanced Pages Tree

The generated PDF strictly follows the "intentionally boring" philosophy required for enterprise RIP stability.

### Output Page Object Structure
```text
Page Object
 ├── /Type /Page
 ├── /Parent <pages_id> 0 R
 ├── /MediaBox [0 0 width_pt height_pt]
 ├── /Resources
 │     └── /XObject
 │           └── /Im0 <image_id> 0 R
 └── /Contents <contents_id> 0 R
```

### Contents Stream
Contains only standard graphics state operations mapping the unit-square image to the page MediaBox:
```text
q {width_pt} 0 0 {height_pt} 0 0 cm /Im0 Do Q
```

### Balanced Pages Tree for Large Documents
Legacy RIPs often fail when parsing single arrays containing thousands of children. For documents exceeding 64 pages, RipSafe generates a hierarchical balanced tree with branching factor 64:
- Level 0: Individual `Page` objects (`3*i + 3`).
- Level 1: Intermediate `/Type /Pages` nodes, each indexing up to 64 pages.
- Level 2: Super-nodes indexing up to 64 Level 1 nodes.
- Root: Top-level `/Type /Pages` node referenced by `/Catalog`.

---

## 6. Resumability & Input Fingerprint

Long-running jobs are statefully resumable via `--resume`:
- **State File**: `<output>.ripsafe-state.json`.
- **Atomic State Persistence**: Saved periodically and upon interruption by writing to a `.tmp` file, syncing, and atomically replacing the state file.
- **Fingerprinting**: Uses file size, modification timestamp, and a sampled SHA-256 hash of the head and tail (or full SHA-256 with `--full-hash`).
- **Safety Checks**: Resume is rejected if file size, fingerprint, DPI, color mode, or compression settings mismatch.

---

## 7. Atomic Output & Interruption Handling

1. All assembly is written to `<output>.partial`.
2. When Ctrl+C is pressed, workers are stopped, partial progress is saved to the state file, and `<output>.partial` is preserved.
3. Only after the output document is finalized and passes post-assembly validation is `<output>.partial` atomically renamed to `<output>`.
4. Input and output paths are canonicalized; jobs targeting the input file path are rejected immediately.
