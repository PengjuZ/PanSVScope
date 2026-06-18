# PanSVScope: Comprehensive Structural Variant Analysis Toolkit

**PanSVScope** is an integrated, end‑to‑end C‑based pipeline for comprehensive structural variant (SV) analysis across multiple samples. It covers the full lifecycle of SV analysis from raw read alignment to population‑level genotyping:

- **mapr** – Align paired‑end reads, sort, mark duplicates and index BAM files (optimised for SV discovery)
- **svcall** – Run six complementary SV callers (Manta, Delly, WHAM, Smoove, Dysgu, Matchclips) in parallel and consolidate results
- **pangra** – Integrate WGS‑based SVs, existing pan‑genome VCFs and known SV databases to build a unified variation graph
- **svgeno** – Perform graph‑based genotyping using VG‑Giraffe, BayesTyper, GraphTyper2 and PanGenie
- **svmer** – Merge individual VCFs into a multi‑sample population‑level VCF

## Table of Contents

- [Installation](#installation)
- [Quick Start Examples](#quick-start-examples)
- [Command Reference](#command-reference)
- [Output File Descriptions](#output-file-descriptions)
- [Important Notes](#important-notes)
- [Citation](#citation)
- [License](#license)

## Installation

### Dependencies

- C compiler (gcc / clang) with C99 support
- Standard Unix utilities: `grep`, `awk`, `sort`, `zcat`, `mkdir`, `rm`, `find`

#### Bundled Tools (included with PanSVScope)

The following tools are packaged together with PanSVScope – no separate installation is required:

| Tool | Version |
|------|---------|
| PanSVScope | 1.0.0 |
| CheckSV | 1.0.0 (bundled) |
| Markdup | 1.0.0 (bundled) |
| Bwa | 0.7.17-r1188 (bundled) |
| Sambamba | 1.0.1 (bundled) |
| Delly | 1.7.3 (bundled) |
| matchclips | 2 (bundled) |
| SURVIVOR | 1.0.7 (bundled) |
| minimap2 | 2.30-r1290-dirty (bundled) |
| Graphvcf | 1.2.2 (bundled) |
| Vg | 1.72.0 (bundled) |
| PanGenie | 4.2.1 (bundled) |
| Parallel | 20170422 (bundled) |
| graphtyper | 2.7.7 (bundled) |
| bayesTyper | 1.5 (bundled) |
| bayesTyperTools | 1.5 (bundled) |
| Kmc | 3.2.4 (bundled) |

> **Note:** These bundled tools have been tested with PanSVScope and are expected to work out‑of‑the‑box. If you encounter environment‑specific issues (e.g., library conflicts), you may choose to install your own versions of these tools and provide their paths to the respective command‑line options. However, we recommend using the bundled versions for full compatibility.

#### External Tools (must be provided as paths)

The following tools are **not** included and must be installed separately by the user. Provide their full paths when running PanSVScope commands.

| Tool | Tested Version | Notes |
|------|----------------|-------|
| bedtools | 2.31.1 | Install via conda (Python 3.11 environment) |
| vcfwave | 1.0.14 (vcflib) | Install via conda (Python 3.11 environment) |
| wham | 1.7.0 | Install via conda (Python 3.11 environment) |
| samtools | 1.23 | Install via conda (Python 3.11 environment) |
| bgzip / tabix | 1.23 (htslib) | Install via conda (Python 3.11 environment) |
| bcftools | 1.23.1 | Install via conda (Python 3.11 environment) |
| Manta | 1.6.0 | Requires Python 2.7 environment (separate conda env) |
| smoove | 0.2.8 | Requires Python 2.7 environment (separate conda env) |
| dysgu | 1.40 / 1.70 / 1.87 | Requires its own isolated conda environment (see below) |

### Download precompiled binary

You can also run it directly:

```bash
./PanSVScope --help
```

### Conda environment setup

We strongly recommend using [**mamba**](https://github.com/mamba-org/mamba) instead of `conda` for faster and more reliable dependency resolution. If you do not have mamba installed, you can install it via conda first:

```bash
conda install -n base -c conda-forge mamba
```

Then create and configure the required environments as follows.

#### Main environment (Python 3.11) – for most external tools

```bash
mamba create -n PanSVScope python=3.11 -y
mamba activate PanSVScope
mamba install bioconda::wham bioconda::bedtools bioconda::bcftools bioconda::samtools bioconda::htslib bioconda::vcflib
```

#### Environment for Manta and Smoove (Python 2.7)

```bash
mamba create -n SV_check python=2.7 -y
mamba activate SV_check
mamba install bioconda::manta bioconda::smoove
```

#### Isolated environment for Dysgu (Python 3.11)

Dysgu must be installed in its own isolated environment to avoid conflicts with other packages. Two installation methods are provided:

**Option 1 – Install via bioconda (simple):**

```bash
mamba create -n dysgu-env python=3.11 -y
mamba activate dysgu-env
mamba install bioconda::dysgu
mamba install --force-reinstall numpy=1.26.4 pandas=1.5.3 -y
```

**Option 2 – Install from source (if the bioconda package fails):**

```bash
# Create the environment with build dependencies
mamba create -n dysgu-env \
  python=3.11 \
  "pip>=23.1" \
  "cython>=3" \
  "meson-python>=0.14" \
  compilers \
  numpy \
  pandas \
  scipy \
  scikit-learn \
  networkx \
  sortedcontainers \
  "click>=8.0" \
  "pysam=0.23.3" \
  "superintervals>=0.3.0" \
  lightgbm \
  "htslib=1.23" \
  -c conda-forge -c bioconda --strict-channel-priority -y

# Activate the environment
mamba activate dysgu-env

# Clone the Dysgu source code (version 1.8.8)
git clone --branch v1.8.8 --depth 1 https://github.com/kcleal/dysgu.git
cd dysgu

# Install Dysgu using pip (linking to the htslib from the conda environment)
python -m pip install . --no-build-isolation \
  -Csetup-args="-Dhtslib_prefix=$CONDA_PREFIX"
```

> **Note:** If you prefer to use `conda` instead of `mamba`, simply replace `mamba` with `conda` in the commands above. However, `mamba` is significantly faster and less prone to dependency conflicts.

## Quick Start Examples

### 1. Align reads and produce a sorted, duplicate‑marked BAM (mapr)

```bash
PanSVScope mapr \
  --ID sample01 \
  --GENOME hg38.fa \
  --R1 sample01_R1.fastq.gz \
  --R2 sample01_R2.fastq.gz \
  --OUTDIR ./bams \
  --TMPDIR ./tmp \
  --THREADS 32 \
  --BWA /usr/bin/bwa \
  --SAMBAMBA /usr/bin/sambamba
```

Output: `./bams/sample01.bam` and `./bams/sample01.bam.bai`.

### 2. Run multiple SV callers and consolidate results (svcall)

```bash
PanSVScope svcall \
  --id sample01 \
  --bam ./bams/sample01.bam \
  --genome hg38.fa \
  --outdir ./sv_calls \
  --threads 16 \
  --clean-temp \
  --manta /usr/bin/configManta.py \
  --Pythonformanta /usr/bin/python2 \
  --delly /usr/bin/delly \
  --wham /usr/bin/whamg \
  --smoove /usr/bin/smoove \
  --dysgu /usr/bin/dysgu \
  --matchclips /usr/local/bin/matchclips \
  --survivor /usr/local/bin/SURVIVOR \
  --bcftools /usr/bin/bcftools \
  --samtools-bindir /usr/bin
```

Output: `./sv_calls/sample01.SVout.txt` (consolidated SV calls) and `./sv_calls/sample01.SVcall.log`.

### 3. Build a pan‑genome graph from multiple SV sources (pangra)

First create a file `wgs_sv_list.txt` listing all `.SVout.txt` files (one per sample):

```
./sv_calls/sample01.SVout.txt
./sv_calls/sample02.SVout.txt
./sv_calls/sample03.SVout.txt
```

Then run:

```bash
PanSVScope pangra \
  --workdir ./pangraph_results \
  --genome hg38.fa \
  --sv-out-list wgs_sv_list.txt \
  --known-sv gnomad_sv.txt \
  --pgenome 1kGP.pangenome.vcf.gz \
  --threads 32 \
  --clean \
  --bedtools /usr/bin/bedtools \
  --vcfwave /usr/bin/vcfwave \
  --check-sv-tools /usr/local/bin/CheckSV \
  --markdup /usr/local/bin/markdup \
  --minimap2 /usr/bin/minimap2 \
  --graphvcf /usr/local/bin/graphvcf \
  --vg /usr/local/bin/vg \
  --pangenie-index /usr/local/bin/PanGenie-index \
  --parallel /usr/bin/parallel \
  --bgzip /usr/bin/bgzip \
  --tabix /usr/bin/tabix
```

Outputs: `./pangraph_results/SVsets.Merge.vcf.gz` (integrated SV set), `./pangraph_results/Index/` (graph indexes for VG‑Giraffe and PanGenie).

### 4. Genotype a sample using the built graph (svgeno)

```bash
PanSVScope svgeno \
  --id sample01 \
  --workdir ./svgeno_out \
  --index ./pangraph_results/Index \
  --genome hg38.fa \
  --bam ./bams/sample01.bam \
  --r1 sample01_R1.fastq.gz \
  --r2 sample01_R2.fastq.gz \
  --enable-all \
  --threads 24 \
  --vg /usr/bin/vg \
  --bayestyper-tools /usr/bin/bayesTyperTools \
  --bayestyper /usr/bin/bayesTyper \
  --kmc /usr/bin/kmc \
  --graphtyper /usr/bin/graphtyper \
  --sambamba /usr/bin/sambamba \
  --pangenie /usr/bin/PanGenie \
  --graphvcf /usr/local/bin/graphvcf \
  --bgzip /usr/bin/bgzip
```

Outputs: `./svgeno_out/sample01/sample01.vcf.gz` (merged genotypes from all enabled tools), plus per‑tool VCFs (`VG-Giraffe.vcf.gz`, `BayesTyper.vcf.gz`, etc.).

### 5. Merge multiple sample VCFs into a population matrix (svmer)

Create `vcf_list.txt` with paths to each sample’s final VCF (from svgeno):

```
./svgeno_out/sample01/sample01.vcf.gz
./svgeno_out/sample02/sample02.vcf.gz
./svgeno_out/sample03/sample03.vcf.gz
```

Then run:

```bash
PanSVScope svmer \
  --workdir ./svmerge_out \
  --SVindex ./pangraph_results/Index/Index.vcf.gz \
  --Vcflist vcf_list.txt \
  --BGZIP /usr/bin/bgzip
```

Output: `./svmerge_out/SVgeno.vcf.gz` – a multi‑sample VCF with genotype columns for every sample.

## Command Reference

### `PanSVScope mapr`

Align paired‑end reads, sort, mark duplicates and index. Supports two alignment tools: **BWA‑MEM** (classic) and **minibwa** (faster). If both `--BWA` and `--Minibwa` are provided, **minibwa** takes precedence.

| Option | Description |
|--------|-------------|
| `--ID` | Sample identifier (used in read groups and output names) (required) |
| `--GENOME` | Reference genome FASTA file (for BWA) or index prefix (for minibwa) (required) |
| `--R1` | Forward FASTQ file (may be gzipped) (required) |
| `--R2` | Reverse FASTQ file (required) |
| `--OUTDIR` | Output directory for final BAM and index (required) |
| `--TMPDIR` | Temporary directory for sorting (required) |
| `--THREADS` | Number of CPU threads (required) |
| `--BWA` | Path to BWA‑MEM executable (optional, but at least one aligner is required) |
| `--Minibwa` | Path to minibwa executable (optional, takes precedence if both provided) |
| `--SAMBAMBA` | Path to Sambamba executable (required) |
| `-h, --help` | Show help message |

**Pipeline steps:** Alignment (BWA or minibwa) → SAM to BAM conversion → sorting → duplicate marking → indexing → cleanup of intermediate files.

**Indexing your reference:**
- For **BWA**: `bwa index -p <reference.fa> <reference.fa>` (the `-p` option sets the prefix; if omitted, the output files use the reference name). The `--GENOME` must point to the same prefix file (or the reference file if no prefix was used).
- For **minibwa**: `minibwa index <reference.fa> <reference.fa>` (the second argument is the output prefix; if omitted, it uses the input name). The `--GENOME` must be the prefix you used.

Example commands:
```bash
# BWA index (outputs hg38.fa.amb, hg38.fa.ann, ...)
bwa index -p hg38.fa hg38.fa

# minibwa index (outputs hg38.fa.amb, hg38.fa.ann, ...)
minibwa index hg38.fa hg38.fa
```

**Examples:**
- Using BWA:
```bash
PanSVScope mapr --ID=sample01 --GENOME=hg38.fa --R1=sample01_R1.fq.gz --R2=sample01_R2.fq.gz --OUTDIR=./bams --TMPDIR=./tmp --THREADS=32 --BWA=/usr/bin/bwa --SAMBAMBA=/usr/bin/sambamba
```
- Using minibwa (the `--GENOME` is the index prefix):
```bash
PanSVScope mapr --ID=sample01 --GENOME=hg38.fa --R1=sample01_R1.fq.gz --R2=sample01_R2.fq.gz --OUTDIR=./bams --TMPDIR=./tmp --THREADS=32 --Minibwa=/usr/bin/minibwa --SAMBAMBA=/usr/bin/sambamba
```

### `PanSVScope svcall`

Ensemble SV calling using multiple tools.

| Option | Description |
|--------|-------------|
| `--id` | Sample identifier (required) |
| `--bam` | Input BAM file (sorted and indexed) (required) |
| `--genome` | Reference genome FASTA (required) |
| `--outdir` | Output directory (required) |
| `--threads` | Number of CPU threads [default: 1] |
| `--clean-temp` | Remove tool‑specific temporary directories after completion |
| `--manta` | Path to Manta configuration script (e.g., `configManta.py`) |
| `--Pythonformanta` | Python interpreter for Manta (e.g., `python2`) |
| `--delly` | Path to Delly |
| `--wham` | Path to WHAM |
| `--smoove` | Path to Smoove |
| `--dysgu` | Path to Dysgu |
| `--matchclips` | Path to Matchclips |
| `--survivor` | Path to SURVIVOR (required) |
| `--bcftools` | Path to bcftools (required if Delly used) |
| `--samtools-bindir` | Directory containing samtools (added to PATH) |
| `--help` | Show help message |

At least one SV caller must be provided. Output is a unified `{id}.SVout.txt` file with columns: `chrom start end ref alt type tool`.

### `PanSVScope pangra`

Integrate SVs from multiple sources and build a variation graph.

| Option | Description |
|--------|-------------|
| `--workdir` | Working directory (required) |
| `--genome` | Reference genome FASTA (required) |
| `--sv-out-list` | File listing `.SVout.txt` files (WGS calls) |
| `--known-sv` | Known SV database (tab‑delimited) |
| `--pgenome` | Existing pan‑genome VCF (gzipped, tabix‑indexed) |
| `--threads` | Number of CPU threads [default: 1] |
| `--clean` | Remove temporary files after completion |
| `--phase` | Run specific phases: `1,2,3,4,5` or `all` [default: all] |
| `--bedtools` | Path to bedtools (required) |
| `--vcfwave` | Path to vcfwave (required) |
| `--check-sv-tools` | Path to CheckSV (required) |
| `--markdup` | Path to markdup (required) |
| `--minimap2` | Path to minimap2 (required) |
| `--graphvcf` | Path to graphvcf (required) |
| `--vg` | Path to vg (required) |
| `--pangenie-index` | Path to PanGenie-index (required) |
| `--parallel` | Path to GNU parallel (required) |
| `--bgzip` | Path to bgzip (required) |
| `--tabix` | Path to tabix (required) |
| `--help` | Show help message |

At least one SV source (`--sv-out-list`, `--known-sv` or `--pgenome`) must be provided. The five phases are:

1. **WGS Processing** – cluster and filter WGS SV calls  
2. **Pan‑genome Processing** – normalise and split existing VCF  
3. **Known SV Processing** – load and validate known database  
4. **Merge and Validation** – deduplicate and merge all sources  
5. **Index Construction** – build VG Giraffe and PanGenie indexes

### `PanSVScope svgeno`

Graph‑based genotyping using multiple methods.

| Option | Description |
|--------|-------------|
| `--id` | Sample ID (required) |
| `--workdir` | Working directory (required) |
| `--index` | Directory containing pre‑built graph indexes (required) |
| `--genome` | Reference genome FASTA (required) |
| `--bam` | Input BAM file (required for BayesTyper / GraphTyper2) |
| `--r1`, `--r2` | FASTQ files (required for VG‑Giraffe / PanGenie) |
| `--threads` | Number of CPU threads [default: 1] |
| `--enable-vg-giraffe` | Enable VG‑Giraffe |
| `--enable-bayestyper` | Enable BayesTyper |
| `--enable-graphtyper2` | Enable GraphTyper2 |
| `--enable-pangenie` | Enable PanGenie |
| `--enable-all` | Enable all four tools |
| `--vg` | Path to vg (required if VG‑Giraffe enabled) |
| `--bayestyper-tools` | Path to BayesTyperTools (required if BayesTyper enabled) |
| `--bayestyper` | Path to BayesTyper (required if BayesTyper enabled) |
| `--kmc` | Path to KMC (required if BayesTyper enabled) |
| `--graphtyper` | Path to GraphTyper2 (required if GraphTyper2 enabled) |
| `--sambamba` | Path to Sambamba (required if GraphTyper2 enabled) |
| `--pangenie` | Path to PanGenie (required if PanGenie enabled) |
| `--graphvcf` | Path to graphvcf (required for merging) |
| `--bgzip` | Path to bgzip (required) |
| `--help` | Show help message |

At least one genotyping tool must be enabled. The final merged VCF is written as `{workdir}/{id}/{id}.vcf.gz`.

### `PanSVScope svmer`

Merge multiple VCFs into a multi‑sample VCF.

| Option | Description |
|--------|-------------|
| `--workdir` | Working directory (required) |
| `--SVindex` | Index VCF (from `pangra`, e.g., `Index.vcf.gz`) (required) |
| `--Vcflist` | File listing input VCFs (one per line) (required) |
| `--BGZIP` | Path to bgzip (required) |
| `--help` | Show help message |

Output: `{workdir}/SVgeno.vcf.gz` – a VCF with genotype columns for all samples listed in `Vcflist`.

## Output File Descriptions

### mapr outputs
- `<OUTDIR>/<ID>.bam` – final sorted, duplicate‑marked BAM
- `<OUTDIR>/<ID>.bam.bai` – BAM index

### svcall outputs
- `<outdir>/<id>.SVout.txt` – consolidated SV calls (tab‑separated: `chrom, start, end, ref, alt, type, tool`)
- `<outdir>/<id>.SVcall.log` – detailed log with timestamps and per‑tool status
- Tool‑specific directories (`<id>_Manta/`, `<id>_Delly/`, …) – kept unless `--clean-temp` is used

### pangra outputs
- `<workdir>/SVsets.Merge.vcf.gz` – final integrated SV set (compressed and indexed)
- `<workdir>/Index/` – directory containing:
  - `Index.vcf.gz` – VCF representation of the graph
  - `Index.vg.giraffe.gbz` – VG Giraffe graph index
  - `Index.vg.dist`, `Index.vg.snarls` – auxiliary VG indexes
  - `PanGenie/` – PanGenie index directory
- `<workdir>/tmp/` – intermediate files (removed if `--clean` used)

### svgeno outputs
- `<workdir>/<id>/<id>.vcf.gz` – merged genotypes (consensus of enabled tools)
- `<workdir>/<id>/VG-Giraffe.vcf.gz` – VG‑Giraffe results (if enabled)
- `<workdir>/<id>/BayesTyper.vcf.gz` – BayesTyper results (if enabled)
- `<workdir>/<id>/GraphTyper2.vcf.gz` – GraphTyper2 results (if enabled)
- `<workdir>/<id>/PanGenie.vcf.gz` – PanGenie results (if enabled)

### svmer outputs
- `<workdir>/SVgeno.vcf.gz` – multi‑sample VCF with one column per input sample
- `<workdir>/TMP/` and `<workdir>/TMPfile/` – temporary files (automatically cleaned)

## Important Notes

1. **External dependencies**  
   All required tools must be installed and provided as **full paths** to the respective `--tool` arguments. The software bundle includes only PanSVScope itself; external binaries (BWA, Sambamba, SV callers, etc.) must be obtained separately.

2. **Python environments**  
   Manta and Smoove require **Python 2.7** – use a separate conda environment for them (e.g., `SV_check`). Dysgu requires its **own isolated environment** (e.g., `dysgu-env`) due to dependency conflicts with other packages.

3. **BAM indexing**  
   Input BAM files for `svcall` must be coordinate‑sorted and indexed (`.bam.bai`). The `mapr` module produces such BAMs. If you have your own generated bam, as long as it meets the requirements, it is acceptable.

4. **Reference genome indexing**  
   The reference FASTA must be indexed with `bwa index` (for `mapr`), `samtools faidx` (for many tools), and for some SV callers additional indexes may be required.

5. **Chromosome naming**  
   All inputs (BAM, reference, VCFs, index files) must use consistent chromosome names (e.g., `chr1` vs `1`). Inconsistent naming will cause errors.

6. **Memory and disk space**  
   - `mapr` sorting uses 16 GB per thread (configurable via Sambamba’s `-m` parameter in the code).  
   - `pangra` phases may create large intermediate files; ensure `TMPDIR` or the working directory has sufficient free space (at least 2× the size of input FASTQ files).  
   - `svgeno` runs multiple tools sequentially; each tool may use substantial memory (e.g., VG‑Giraffe >32 GB for human genomes).

7. **Running phases independently**  
   `pangra` allows re‑running individual phases with the `--phase` option, which is useful for debugging or when only part of the pipeline needs to be updated.

8. **Clean‑up options**  
   Use `--clean-temp` (svcall) or `--clean` (pangra) to remove intermediate directories. Without these flags, temporary files are kept for inspection.

9. **Selective SV caller execution (`svcall`)**  
   `svcall` supports running any subset of the six SV callers (Manta, Delly, WHAM, Smoove, Dysgu, Matchclips). Simply provide the paths for the tools you wish to use; any omitted tool is automatically skipped. This allows you to tailor the run to your needs – for example, run only one or two callers for a quick test, or enable multiple callers to maximise recall. **Note:** If a tool is not installed or its path is not provided, PanSVScope ignores it and continues without error.

10. **Flexible genotyper selection (`svgeno`)**  
    `svgeno` requires at least one genotyping method to be enabled (`--enable-vg-giraffe`, `--enable-bayestyper`, `--enable-graphtyper2`, `--enable-pangenie`, or `--enable-all`). Depending on your goals:
    - **For speed**: Enable only `--enable-pangenie` – its k‑mer based approach is computationally efficient and suitable for large cohorts.
    - **For accuracy**: Enable `--enable-all` to run all four tools and automatically merge their results into a consensus genotype, giving the most reliable calls.
    - **Custom combinations**: Enable any subset (e.g., only VG‑Giraffe and GraphTyper2).  
    If no genotyper is enabled, the program will exit with an error.

11. **Optional external tools**  
    Consistent with the above, you only need to install and provide paths for the tools you actually intend to use. For example, if you run `svcall` with only Delly and WHAM, you do not need to install Manta, Smoove, or other callers. Likewise, if `svgeno` is run with only PanGenie, you do not need BayesTyper, GraphTyper2, or their dependencies. This greatly simplifies environment setup.

12. **Pre‑processing raw WGS FASTQ data**  
    We strongly recommend running **fastp** (or a similar quality control tool) on your raw FASTQ files **before** feeding them into the PanSVScope pipeline. Quality trimming, adapter removal, and read filtering significantly improve mapping accuracy and SV calling sensitivity.

13. **Pre‑built `--known-sv` datasets**  
    For several domestic animals (e.g., cattle, pig, chicken) and human, we provide ready‑to‑use known SV databases that can be directly supplied to the `--known-sv` parameter of `pangra`. These datasets have been curated from public resources (e.g., dbVar, gnomAD‑SV) and are compatible with PanSVScope.  
    ➤ **Download links:** [See our data repository – XXXXA]  

14. **Building and using `--pgenome` files**  
    If you wish to construct a custom pan‑genome VCF for your own set of genomes, we recommend using **cactus-pangenome** with the `--vcf` flag to produce a `pangenome.vcf.gz` file.  
    The resulting `pangenome.vcf.gz` can be used directly with `pangra --pgenome`.  
    Additionally, we provide pre‑built pan‑genome VCF files for several species (human, cattle, pig, etc.) to save you computation time.  
    ➤ **Download links:** [See our data repository – XXXXB]  

15. **Quick SV genotyping for a few samples**  
    If you have only a handful of samples and want to quickly genotype them without building a full pan‑genome graph from scratch, you can use a streamlined workflow:
    - **Step 1:** Run `pangra` with **only** `--known-sv` (and optionally `--pgenome`, but at least `--known-sv`). This will create a merged VCF and indexes using the known SV database.
    - **Step 2:** Run `svgeno` with **only** PanGenie (`--enable-pangenie`) using the indexes produced in Step 1.  
    This two‑step approach is fast and does not require WGS‑based SV calls (i.e., you can skip `svcall`).

## Citation

If you use PanSVScope in a publication, please cite:

> PanSVScope development team, 2026.

## License

MIT License

Copyright (c) 2026 Institute of Genomics
