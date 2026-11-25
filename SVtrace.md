# SVtrace: Structural Variant Source Tracing Toolkit

**SVtrace** is a lightweight, C‑based companion utility for **PanSVScope** (`pangra` module) that traces the supporting origin(s) of every structural variant in the final merged SV set (`SVsets.Merge.txt`). It strictly reproduces `PanGra` Phase 4 merging, duplicate marking, and cluster selection rules so that each merged variant can be accurately attributed to one or more input sources (`Known`, `PGenome`, `WGS`):

- **Phase 1 (Source Merge & Markdup)** – Tag active SV sources (`W`, `K`, `P`), sort and deduplicate in exact `PanGra` order, and run `markdup` with `minimap2`
- **Phase 2 (Cluster Source Tracing)** – Reconstruct source support per cluster using the exact `PanGra` `similarity >= 50` merging threshold and representative selection rule
- **Phase 3 (Labeling & Statistics)** – Assign canonical source labels to every record in `SVsets.Merge.txt` and generate a summary report (`SVtrace.summary.txt`)

## Table of Contents

- [Installation](#installation)
- [Quick Start Examples](#quick-start-examples)
- [Command Reference](#command-reference)
- [Tracing & Labeling Rules](#tracing--labeling-rules)
- [Input & Output File Descriptions](#input--output-file-descriptions)
- [Important Notes](#important-notes)
- [Citation](#citation)
- [License](#license)

---

## ⚠️ Important: Use Validated `PanSVScope pangra` Outputs

**SVtrace must be run on the validated `SVsets.*.txt` files produced by `PanSVScope pangra` Phase 4** (`SVsets.Known.txt`, `SVsets.PGenome.txt`, `SVsets.WGS.txt`, and `SVsets.Merge.txt`), using the **same `markdup` and `minimap2` binaries** used during the `pangra` run.

Do **not** pass raw, unvalidated source files (such as raw `--known-sv`, raw `.SVout.txt`, or unsplit `--pgenome` VCFs) directly to SVtrace. Those files have not yet undergone `CheckSV` normalization and validation in `pangra`, which will lead to coordinate/allele mismatches and `Unknown` labels in the traced output.

---

## Installation

### Dependencies

- C compiler (`gcc` / `clang`) with C99 and POSIX (`_POSIX_C_SOURCE 200809L`) support
- Standard Unix utilities: `awk`, `sort`, `uniq`, `mkdir`, `rm`, `rmdir`

#### Required Tools (bundled with PanSVScope)

SVtrace reuses the exact same `markdup` and `minimap2` executables bundled with PanSVScope – no additional third‑party dependencies are required:

| Tool | Tested Version | Notes |
|------|----------------|-------|
| SVtrace | 1.0.0 | Compiled from `SVtrace.c` |
| markdup | 1.0.0 (bundled with PanSVScope) | Required for Phase 1 (`--markdup`) |
| minimap2 | 2.30-r1290-dirty (bundled with PanSVScope) | Required for Phase 1 (`--minimap2`) |

> **Note:** `--markdup` and `--minimap2` are only required when Phase 1 is executed (`--phase=all` or `--phase=1`). If you are re‑running only Phase 2 or Phase 3 on existing `tmp/` files, these two options may be omitted.

### Build from source

Compile `SVtrace.c` directly with `gcc` or `clang` (no external libraries needed):

```bash
gcc -O2 -Wall -o SVtrace SVtrace.c
```

Verify the binary:

```bash
./SVtrace --help
```

## Quick Start Examples

### 1. Trace all three SV sources (`Known`, `PGenome`, `WGS` → 7 combinations)

```bash
./SVtrace \
  --workdir=./trace_out \
  --known-sv=./pangraph_results/SVsets.Known.txt \
  --pgenome-sv=./pangraph_results/SVsets.PGenome.txt \
  --wgs-sv=./pangraph_results/SVsets.WGS.txt \
  --merge-sv=./pangraph_results/SVsets.Merge.txt \
  --markdup=/usr/local/bin/markdup \
  --minimap2=/usr/bin/minimap2
```

Outputs: `./trace_out/SVsets.Merge.trace.txt` (labeled SVs) and `./trace_out/SVtrace.summary.txt` (per‑category counts and percentages).

### 2. Trace with two sources only (`Known` + `PGenome` → 3 combinations)

When only a subset of sources was used in `pangra`, supply only the corresponding files. SVtrace automatically restricts the active trace categories to the valid combinations (`Known`, `PGenome`, `Known-PGenome`):

```bash
./SVtrace \
  --workdir=./trace_2src \
  --known-sv=./pangraph_results/SVsets.Known.txt \
  --pgenome-sv=./pangraph_results/SVsets.PGenome.txt \
  --merge-sv=./pangraph_results/SVsets.Merge.txt \
  --markdup=/usr/local/bin/markdup \
  --minimap2=/usr/bin/minimap2 \
  --clean
```

### 3. Run directly in a `pangra` working directory (auto‑detect `--merge-sv`)

If `--workdir` already contains `SVsets.Merge.txt`, `--merge-sv` can be omitted and will default to `<workdir>/SVsets.Merge.txt`:

```bash
./SVtrace \
  --workdir=./pangraph_results \
  --known-sv=./pangraph_results/SVsets.Known.txt \
  --pgenome-sv=./pangraph_results/SVsets.PGenome.txt \
  --wgs-sv=./pangraph_results/SVsets.WGS.txt \
  --output=./pangraph_results/SVsets.Merge.trace.txt \
  --markdup=/usr/local/bin/markdup \
  --minimap2=/usr/bin/minimap2
```

### 4. Re‑run specific phases without repeating `markdup`

If Phase 1 has already completed and `<workdir>/tmp/Trace.Merge.md.txt` is preserved (i.e., `--clean` was not used), you can re‑run Phase 2 and Phase 3 directly without invoking `markdup`/`minimap2` again:

```bash
./SVtrace \
  --workdir=./trace_out \
  --known-sv=./pangraph_results/SVsets.Known.txt \
  --pgenome-sv=./pangraph_results/SVsets.PGenome.txt \
  --wgs-sv=./pangraph_results/SVsets.WGS.txt \
  --merge-sv=./pangraph_results/SVsets.Merge.txt \
  --phase=2,3
```

## Command Reference

### `SVtrace`

Both `--option=value` and `--option value` command‑line styles are supported.

| Option | Description |
|--------|-------------|
| `--workdir` | Working directory for intermediate (`tmp/`) and output files (required) |
| `--markdup` | Path to `markdup` executable – use the same binary as `PanSVScope pangra` (required for Phase 1) |
| `--minimap2` | Path to `minimap2` executable – use the same binary as `PanSVScope pangra` (required for Phase 1) |
| `--merge-sv` | Path to `SVsets.Merge.txt` to be labeled [default: `<workdir>/SVsets.Merge.txt` if present] |
| `--known-sv` | Path to `SVsets.Known.txt` (source tag: `Known`) |
| `--pgenome-sv` | Path to `SVsets.PGenome.txt` (source tag: `PGenome`; alias: `--pgenome`) |
| `--wgs-sv` | Path to `SVsets.WGS.txt` (source tag: `WGS`; alias: `--wgs`) |
| `--output` | Output path for the labeled SV file [default: `<workdir>/SVsets.Merge.trace.txt`] |
| `--clean` | Remove intermediate files in `<workdir>/tmp/` after successful completion [default: disabled] |
| `--phase` | Run specific phase(s): comma‑separated values from `1,2,3` or `all` [default: `all`] |
| `-h, --help` | Show help message and exit |

At least one SV source (`--known-sv`, `--pgenome-sv`, or `--wgs-sv`) must be provided.

The three phases are:

1. **Source Merge & Markdup** – Tag each provided source (`W` for WGS, `K` for Known, `P` for PGenome), sort by `CHROM` and numeric `POS`, deduplicate with `uniq`, and run `markdup` with `minimap2`.
2. **Cluster Source Tracing** – Group `markdup` results by cluster ID and apply the exact `PanGra` `similarity >= 50` merge/keep logic to compute supporting source sets for every variant key.
3. **Labeling & Statistics** – Map canonical source labels onto `SVsets.Merge.txt` and write both the labeled SV file and the summary table.

## Tracing & Labeling Rules

### PanGra‑consistent cluster tracing

SVtrace mirrors `PanGra` Phase 4 (`final_cluster.awk`) verbatim:

1. **Singleton clusters (`count == 1`)** – The variant is retained independently; its supporting source is solely its own origin (`K`, `P`, or `W`).
2. **Multi‑record clusters (`count > 1`)**:
   - Records with **`similarity < 50`** are retained independently (not merged with other records in the cluster) and keep only their own source.
   - Records with **`similarity >= 50`** are grouped together as a merged variant cluster; `PanGra` selects the representative with the maximum `length(REF) + length(ALT)`, and SVtrace assigns the **union of all sources** across all `similarity >= 50` members in that group.
3. **Identical variant keys** – If the same `(CHROM, POS, REF, ALT)` key (case‑insensitive on `REF`/`ALT`) receives support from multiple records or clusters, its source flags are merged into their union.

### Canonical source labels

Labels always follow the fixed canonical order **`Known` → `PGenome` → `WGS`**:

| Support Type | Possible Labels |
|--------------|-----------------|
| Single source | `Known`, `PGenome`, `WGS` |
| Two sources | `Known-PGenome`, `Known-WGS`, `PGenome-WGS` |
| Three sources | `Known-PGenome-WGS` |
| Unmatched (fallback) | `Unknown` |

The set of active categories adapts automatically to the provided sources:

- **3 sources** (`Known`, `PGenome`, `WGS`) → 7 active categories
- **2 sources** (e.g., `Known`, `PGenome`) → 3 active categories (`Known`, `PGenome`, `Known-PGenome`)
- **1 source** (e.g., `WGS`) → 1 active category (`WGS`)

## Input & Output File Descriptions

### Input file format

`SVsets.Known.txt`, `SVsets.PGenome.txt`, `SVsets.WGS.txt`, and `SVsets.Merge.txt` are headerless, 4‑column tab‑delimited files generated by `PanSVScope pangra`:

| Column | Name | Description |
|--------|------|-------------|
| 1 | `CHROM` | Chromosome / contig name |
| 2 | `POS` | 1‑based start position |
| 3 | `REF` | Reference allele sequence |
| 4 | `ALT` | Alternate allele sequence |

Example:

```
chr1	112140	CTGATCATGCACTGATCACGTACATGATCATGCACTGATCATTTGGCTGATCATACACTG	C
chr1	113730	A	ACACACTGATCACTTGGATGATCATACCCCTATCACGTGACTGATCATGTTAATGCACTGATCACGTGGCTGATCATGCACTGATCACGTGGCTGATCATGCACTGAT
```

### Primary outputs

- **`<output>`** (default `<workdir>/SVsets.Merge.trace.txt`) – 5‑column headerless tab‑delimited file with the original `SVsets.Merge.txt` columns plus the canonical `SOURCE_LABEL`:

```
chr1	112140	CTGATCATGCACTGATCACGTACATGATCATGCACTGATCATTTGGCTGATCATACACTG	C	Known-PGenome-WGS
chr1	113730	A	ACACACTGATCACTTGGATGATCATACCCCTATCACGTGACTGATCATGTTAATGCACTGATCACGTGGCTGATCATGCACTGATCACGTGGCTGATCATGCACTGAT	PGenome-WGS
```

- **`<workdir>/SVtrace.summary.txt`** – Summary report containing metadata headers (input file paths and record counts) followed by counts and percentages for every active source combination:

```
# SVtrace Summary Report (PanGra Merge Standard)
# Merge_File	./pangraph_results/SVsets.Merge.txt
# Output_File	./trace_out/SVsets.Merge.trace.txt
# Input_Known	./pangraph_results/SVsets.Known.txt	52340
# Input_PGenome	./pangraph_results/SVsets.PGenome.txt	89120
# Input_WGS	./pangraph_results/SVsets.WGS.txt	41205
Source_Label	Count	Percentage
Known	12015	11.44%
PGenome	38420	36.59%
WGS	9850	9.38%
Known-PGenome	15630	14.89%
Known-WGS	4120	3.92%
PGenome-WGS	8940	8.51%
Known-PGenome-WGS	16025	15.26%
Total	105000	100.00%
```

> **Note:** An `Unknown` row is only appended to `SVtrace.summary.txt` (with a console warning) if one or more variants in `SVsets.Merge.txt` fail to match any provided source.

### Intermediate files (`<workdir>/tmp/`)

Kept for inspection unless `--clean` is specified:

- `<workdir>/tmp/Trace.Merge.txt` – Combined, source‑tagged (`W`/`K`/`P`), coordinate‑sorted, and `uniq`‑deduplicated source SVs
- `<workdir>/tmp/Trace.Merge.md.txt` – `markdup` output containing cluster IDs (`$6`) and sequence similarity scores (`$7`)
- `<workdir>/tmp/Trace.key_sources.txt` – Deduplicated variant‑to‑source lookup table (`CHROM  POS  REF  ALT  SOURCE_LABEL  CLUSTER_IDS`)
- `<workdir>/tmp/Trace.cluster.detail.txt` – Sorted per‑cluster audit log (`CHROM  POS  REF  ALT  SOURCE_LABEL  CLUSTER_ID  STATUS  MEMBER_COUNT`), where `STATUS` is one of `singleton`, `low_sim_kept`, or `merged_rep`

## Important Notes

1. **Exact reproducibility with `PanSVScope pangra`**  
   SVtrace reproduces the exact source concatenation order (`WGS` → `Known` → `PGenome`), tab‑delimited sorting, `markdup` invocation, and `similarity >= 50` cluster selection used in `PanGra.c` Phase 4. To guarantee 100% agreement with `SVsets.Merge.txt`, always pass the same `markdup` and `minimap2` versions used by `pangra`.

2. **Interpreting `Unknown` labels**  
   In a properly matched run, `Unknown` should be `0`. Non‑zero `Unknown` counts typically indicate that (a) one of the sources used to build `SVsets.Merge.txt` was omitted from the SVtrace command, (b) raw pre‑`CheckSV` files were passed instead of `SVsets.Known.txt` / `SVsets.PGenome.txt` / `SVsets.WGS.txt`, or (c) a different version of `markdup` / `minimap2` produced different clustering results.

3. **Running phases independently**  
   Use `--phase` to run or re‑run individual phases (`1`, `2`, `3`, or comma‑separated combinations like `2,3`):
   - Phase 2 requires `<workdir>/tmp/Trace.Merge.md.txt` from Phase 1.
   - Phase 3 requires `<workdir>/tmp/Trace.key_sources.txt` from Phase 2.
   - Running `--phase=2`, `--phase=3`, or `--phase=2,3` does not require `--markdup` or `--minimap2`.

4. **Cluster‑level diagnostics (`Trace.cluster.detail.txt`)**  
   If you want to inspect how individual clusters were resolved (whether a merged variant came from a `singleton`, a `low_sim_kept` entry with similarity `< 50`, or a `merged_rep` combining multiple records, along with the number of merged members), omit `--clean` and check `<workdir>/tmp/Trace.cluster.detail.txt`.

5. **Clean‑up behavior**  
   Use `--clean` to remove all intermediate files in `<workdir>/tmp/` upon successful completion. Without `--clean`, intermediate files are retained for inspection and phase re‑runs. Note that temporary internal `awk` scripts are always removed automatically at the end of Phase 2 and Phase 3.
