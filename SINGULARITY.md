# Running PanSVScope with Singularity

This document describes how to run **PanSVScope** using a pre‑built Singularity
image (`PanSVScope_SIF`). It covers all five modules (`mapr`, `svcall`, `pangra`,
`svgeno`, `svmer`) and shows how to integrate the pipeline into an HPC workflow
managed by SLURM. If you prefer to run PanSVScope from a conda environment
instead, please refer to the main [README](README.md).

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Bind Mounts – What You Need to Mount](#bind-mounts--what-you-need-to-mount)
- [Path Conventions Inside the Container](#path-conventions-inside-the-container)
- [Environment Variables – The Variables Used in Examples](#environment-variables--the-variables-used-in-examples)
- [Module 1 – mapr (read alignment)](#module-1--mapr-read-alignment)
- [Module 2 – svcall (ensemble SV calling)](#module-2--svcall-ensemble-sv-calling)
- [Module 3 – pangra (pan-genome graph construction)](#module-3--pangra-pan-genome-graph-construction)
- [Module 4 – svgeno (graph-based genotyping)](#module-4--svgeno-graph-based-genotyping)
- [Module 5 – svmer (population matrix)](#module-5--svmer-population-matrix)
- [Running on SLURM](#running-on-slurm)
- [Troubleshooting](#troubleshooting)

---

## Overview

The PanSVScope Singularity image bundles:

- The `PanSVScope` executable (in `/opt/PanSVScope/`)
- The **bundled tools** listed in the main README (`bwa`, `sambamba`, `delly`,
  `matchclips`, `SURVIVOR`, `minimap2`, `graphvcf`, `vg`, `PanGenie-index`,
  `parallel`, `graphtyper`, `bayesTyper`, `bayesTyperTools`, `kmc`, `CheckSV`,
  `Markdup`) under `/opt/PanSVScope/Bin/`
- Three conda environments used by the pipeline:
  - `/opt/envs/main/` – Python 3.11 environment (bedtools, vcfwave, wham,
    bcftools, samtools, bgzip, tabix, …)
  - `/opt/envs/legacy/` – Python 2.7 environment (Manta, Smoove)
  - `/opt/envs/dysgu/` – Isolated Python 3.11 environment for Dysgu

This means you only need to mount your **data directories** and pass the
appropriate `--bind` flags — no local installation of any of these tools is
required.

---

## Prerequisites

- Singularity/Apptainer ≥ 3.6 (or Apptainer ≥ 1.0)
- A copy of the PanSVScope Singularity image, e.g.:

  ```bash
  PanSVScope_SIF=/path/to/pansvscope_1.0.0.sif
  ```

  If you do not have the image yet, build it from the provided definition file
  (see the main README for details) or download it from the release page.

- Make sure the image is executable and readable by your user:

  ```bash
  chmod +x ${PanSVScope_SIF}
  ```

---

## Bind Mounts – What You Need to Mount

Singularity isolates the container from the host filesystem by default. You
must explicitly bind the host directories that contain your **inputs** and
**outputs**, otherwise PanSVScope will not see them.

For a typical full run you will need to mount the following directories. The
example uses shell variables (`${DIR_*}`, `${SampleID}`, …) that you define
once at the top of your script — see [Environment Variables](#environment-variables--the-variables-used-in-examples).

| Variable             | Purpose                                              |
|----------------------|------------------------------------------------------|
| `${DIR_fastq}`       | Raw FASTQ files (`R1`, `R2`)                         |
| `${DIR_Ref_Fasta}`   | Reference genome FASTA (and its `.fai`, `.bwt`, etc.)|
| `${DIR_bam}`         | Input/output BAM files                               |
| `${DIR_tmp}`         | Temporary directory for sorting / intermediate files |
| `${DIR_svcall}`      | Output of `svcall` (`.SVout.txt` files)              |
| `${DIR_known_sv}`    | Pre‑built known‑SV database directory                |
| `${DIR_pgenome}`     | Pre‑built pan‑genome VCF directory                   |
| `${DIR_Index}`       | Graph index directory from `pangra`                  |
| `${DIR_vcf_list}`    | Directory containing the VCF list for `svmer`        |
| `${DIR_svgeno}`      | Output directory of `svgeno` (per‑sample VCFs)       |
| `${DIR_out}`         | Generic output directory for the current module      |

Multiple directories can be passed in a single `--bind` argument separated by
commas:

```bash
--bind "${DIR_fastq}","${DIR_Ref_Fasta}","${DIR_bam}","${DIR_tmp}"
```

You do **not** need to mount `/opt/PanSVScope`, `/opt/envs/`, or any tool
under them — those are already inside the image.

> **Tip:** If most of your data lives under a single parent directory (e.g.,
> `/data`), you can bind just that parent once:
>
> ```bash
> --bind /data:/data
> ```
>
> This is simpler but gives the container access to the entire `/data` tree.
> Bind only what you actually need if you have strict access‑control
> requirements.

> **Reminder (see main README):** the reference genome must contain only
> uppercase letters (A/C/G/T/N). If it is soft‑masked, convert it first with
> `seqtk seq -A -U ref.fa -l 60 > ref.upper.fa` and re‑index it before running
> PanSVScope.

---

## Path Conventions Inside the Container

Because you bind your host directories to the **same absolute paths** inside
the container, the paths passed to PanSVScope are identical on both sides.
For example:

```bash
--bind "${DIR_bam}","${DIR_Ref_Fasta}"
```

If `${DIR_bam}` is `/data/bam`, then `/data/bam` is also visible as
`/data/bam` inside the container. Therefore you can safely use variables like
`${REF_FASTA}`, `${IN_Bam}`, etc., both in shell scripts outside the container
and in `PanSVScope` arguments inside it.

If you prefer to bind to a **different path** inside the container (e.g., to
avoid long host paths), use the `src:dst` syntax:

```bash
--bind /data:/mnt/data
```

and then pass `/mnt/data/...` to PanSVScope. **Do not do this** unless you have
a specific reason; keeping paths identical on both sides is simpler and less
error‑prone.

---

## Environment Variables – The Variables Used in Examples

The examples below use the following shell variables. They are expanded by the
host shell **before** `singularity exec` runs, so they are unaffected by the
container's `--cleanenv` flag.

| Variable                 | Meaning                                              |
|--------------------------|------------------------------------------------------|
| `PanSVScope_SIF`         | Absolute path to the Singularity image               |
| `SampleID`               | Sample identifier                                    |
| `REF_FASTA`              | Reference genome FASTA (uppercase, indexed)          |
| `IN_R1`, `IN_R2`         | Input FASTQ files                                    |
| `IN_Bam`                 | Input BAM file                                       |
| `SVOUT_LIST`             | File listing `.SVout.txt` files for `pangra`         |
| `KNOWN_SV`               | Known‑SV database file                               |
| `PGENOME_VCF`            | Pan‑genome VCF file                                  |
| `VCF_LIST`               | File listing per‑sample VCFs for `svmer`             |
| `SLURM_CPUS_PER_TASK`    | Number of CPUs allocated by SLURM (auto‑set by SLURM)|
| `DIR_*`                  | Host directories that must be bind‑mounted           |

You can define them once at the top of your script:

```bash
export PanSVScope_SIF=/path/to/pansvscope_1.0.0.sif

export DIR_fastq=/data/fastq
export DIR_Ref_Fasta=/data/ref
export DIR_bam=/data/bam
export DIR_tmp=/scratch/tmp
export DIR_svcall=/data/svcall
export DIR_known_sv=/data/known_sv
export DIR_pgenome=/data/pangenome
export DIR_Index=/data/pangraph_results/Index
export DIR_vcf_list=/data/vcf_lists
export DIR_svgeno=/data/svgeno
export DIR_out=/data/output

export SampleID=sample01
export REF_FASTA=${DIR_Ref_Fasta}/ref.upper.fa
export IN_R1=${DIR_fastq}/${SampleID}_R1.fastq.gz
export IN_R2=${DIR_fastq}/${SampleID}_R2.fastq.gz
export IN_Bam=${DIR_bam}/${SampleID}.bam
export SVOUT_LIST=${DIR_svcall}/svout_list.txt
export KNOWN_SV=${DIR_known_sv}/known_sv.txt
export PGENOME_VCF=${DIR_pgenome}/pangenome.vcf.gz
export VCF_LIST=${DIR_vcf_list}/vcf_list.txt
```

> **Note:** In a SLURM batch script, `SLURM_CPUS_PER_TASK` is set automatically
> by SLURM when you use `#SBATCH --cpus-per-task=N`. If you run interactively,
> replace it with a fixed number such as `16` or `32`.

---

## Module 1 – mapr (read alignment)

Aligns paired‑end reads, sorts, marks duplicates, and indexes the final BAM.

```bash
singularity exec --cleanenv \
  --bind "${DIR_fastq}","${DIR_Ref_Fasta}","${DIR_bam}","${DIR_tmp}" \
  "${PanSVScope_SIF}" \
  PanSVScope mapr \
  --ID="${SampleID}" \
  --GENOME="${REF_FASTA}" \
  --R1="${IN_R1}" \
  --R2="${IN_R2}" \
  --OUTDIR="${DIR_bam}" \
  --TMPDIR="${DIR_tmp}" \
  --THREADS="${SLURM_CPUS_PER_TASK}" \
  --BWA=/opt/PanSVScope/Bin/bwa \
  --SAMBAMBA=/opt/PanSVScope/Bin/sambamba
```

**Output:** `${DIR_bam}/${SampleID}.bam` and `${DIR_bam}/${SampleID}.bam.bai`.

---

## Module 2 – svcall (ensemble SV calling)

Runs six SV callers (Manta, Delly, WHAM, Smoove, Dysgu, Matchclips) and
consolidates the results into a single `.SVout.txt`.

```bash
singularity exec --cleanenv \
  --env PREPEND_PATH=/opt/envs/legacy/bin \
  --bind "${DIR_bam}","${DIR_Ref_Fasta}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope svcall \
  --id="${SampleID}" \
  --bam="${IN_Bam}" \
  --genome="${REF_FASTA}" \
  --outdir="${DIR_out}" \
  --threads="${SLURM_CPUS_PER_TASK}" \
  --clean-temp \
  --manta=/opt/envs/legacy/bin/configManta.py \
  --Pythonformanta=/opt/envs/legacy/bin/python \
  --delly=/opt/PanSVScope/Bin/delly \
  --wham=/opt/envs/main/bin/whamg \
  --smoove=/opt/envs/legacy/bin/smoove \
  --dysgu=/opt/envs/dysgu/bin/dysgu \
  --matchclips=/opt/PanSVScope/Bin/matchclips \
  --survivor=/opt/PanSVScope/Bin/SURVIVOR \
  --bcftools=/opt/envs/main/bin/bcftools \
  --samtools-bindir=/opt/envs/main/bin
```

**Output:** `${DIR_out}/${SampleID}.SVout.txt` and
`${DIR_out}/${SampleID}.SVcall.log`.

> The `--env PREPEND_PATH=/opt/envs/legacy/bin` flag makes the legacy Python
> 2.7 environment visible first on `PATH`, which is needed by Manta and
> Smoove.

---

## Module 3 – pangra (pan-genome graph construction)

Integrates WGS SVs, known‑SV databases, and existing pan‑genome VCFs into a
unified variation graph.

```bash
singularity exec --cleanenv \
  --bind "${DIR_svcall}","${DIR_Ref_Fasta}","${DIR_known_sv}","${DIR_pgenome}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope pangra \
  --workdir="${DIR_out}" \
  --genome="${REF_FASTA}" \
  --sv-out-list="${SVOUT_LIST}" \
  --known-sv="${KNOWN_SV}" \
  --pgenome="${PGENOME_VCF}" \
  --threads="${SLURM_CPUS_PER_TASK}" \
  --clean \
  --bedtools=/opt/envs/main/bin/bedtools \
  --vcfwave=/opt/envs/main/bin/vcfwave \
  --check-sv-tools=/opt/PanSVScope/Bin/CheckSV \
  --markdup=/opt/PanSVScope/Bin/Markdup \
  --minimap2=/opt/PanSVScope/Bin/minimap2 \
  --graphvcf=/opt/PanSVScope/Bin/graphvcf \
  --vg=/opt/PanSVScope/Bin/vg \
  --pangenie-index=/opt/PanSVScope/Bin/PanGenie-index \
  --parallel=/opt/PanSVScope/Bin/parallel \
  --bgzip=/opt/envs/main/bin/bgzip \
  --tabix=/opt/envs/main/bin/tabix
```

**Output:** `${DIR_out}/SVsets.Merge.vcf.gz` and `${DIR_out}/Index/` (graph
indexes for VG‑Giraffe and PanGenie).

---

## Module 4 – svgeno (graph-based genotyping)

Genotypes a sample using VG‑Giraffe, BayesTyper, GraphTyper2, and/or PanGenie,
then merges the results into a high‑confidence VCF.

```bash
singularity exec --cleanenv \
  --bind "${DIR_bam}","${DIR_fastq}","${DIR_Ref_Fasta}","${DIR_Index}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope svgeno \
  --id="${SampleID}" \
  --workdir="${DIR_out}" \
  --index="${DIR_Index}" \
  --genome="${REF_FASTA}" \
  --bam="${IN_Bam}" \
  --r1="${IN_R1}" \
  --r2="${IN_R2}" \
  --enable-vg-giraffe \
  --enable-bayestyper \
  --enable-graphtyper2 \
  --threads="${SLURM_CPUS_PER_TASK}" \
  --vg=/opt/PanSVScope/Bin/vg \
  --bayestyper-tools=/opt/PanSVScope/Bin/bayesTyperTools \
  --bayestyper=/opt/PanSVScope/Bin/bayesTyper \
  --kmc=/opt/PanSVScope/Bin/kmc \
  --graphtyper=/opt/PanSVScope/Bin/graphtyper \
  --sambamba=/opt/PanSVScope/Bin/sambamba \
  --graphvcf=/opt/PanSVScope/Bin/graphvcf \
  --bgzip=/opt/envs/main/bin/bgzip
```

**Output:** `${DIR_out}/${SampleID}/h.${SampleID}.vcf.gz` plus one VCF per
enabled tool.

> **Resume support:** if a tool's output VCF already exists and is valid, it is
> skipped on subsequent runs. Add `--force` to rerun every enabled tool from
> scratch. This is particularly useful on high‑depth data where a tool may be
> killed by the operating system's OOM killer — simply re‑invoke the same
> command and only the failed tool will run again.

---

## Module 5 – svmer (population matrix)

Merges per‑sample high‑confidence VCFs into a single multi‑sample VCF.

```bash
singularity exec --cleanenv \
  --bind "${DIR_Index}","${DIR_vcf_list}","${DIR_svgeno}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope svmer \
  --workdir="${DIR_out}" \
  --SVindex="${DIR_Index}" \
  --Vcflist="${VCF_LIST}" \
  --BGZIP=/opt/envs/main/bin/bgzip
```

**Output:** `${DIR_out}/SVgeno.vcf.gz` — a multi‑sample VCF with one genotype
column per sample.

---

## Running on SLURM

The following template is a minimal SLURM submission script that runs all five
modules on one sample. Save it as `run_pansvscope.slurm` and submit with
`sbatch run_pansvscope.slurm`.

```bash
#!/bin/bash
#SBATCH --job-name=pansvscope
#SBATCH --cpus-per-task=32
#SBATCH --mem=128G
#SBATCH --time=48:00:00
#SBATCH --output=pansvscope.%j.log

set -euo pipefail

# ---- Paths --------------------------------------------------------------
export PanSVScope_SIF=/path/to/pansvscope_1.0.0.sif

export DIR_fastq=/data/fastq
export DIR_Ref_Fasta=/data/ref
export DIR_bam=/data/bam
export DIR_tmp=/scratch/tmp
export DIR_svcall=/data/svcall
export DIR_known_sv=/data/known_sv
export DIR_pgenome=/data/pangenome
export DIR_Index=/data/pangraph_results/Index
export DIR_vcf_list=/data/vcf_lists
export DIR_svgeno=/data/svgeno
export DIR_out=/data/output

export SampleID=sample01
export REF_FASTA=${DIR_Ref_Fasta}/ref.upper.fa
export IN_R1=${DIR_fastq}/${SampleID}_R1.fastq.gz
export IN_R2=${DIR_fastq}/${SampleID}_R2.fastq.gz
export IN_Bam=${DIR_bam}/${SampleID}.bam
export SVOUT_LIST=${DIR_svcall}/svout_list.txt
export KNOWN_SV=${DIR_known_sv}/known_sv.txt
export PGENOME_VCF=${DIR_pgenome}/pangenome.vcf.gz
export VCF_LIST=${DIR_vcf_list}/vcf_list.txt

mkdir -p "${DIR_bam}" "${DIR_tmp}" "${DIR_svcall}" "${DIR_out}"

# ---- Module 1: mapr -----------------------------------------------------
singularity exec --cleanenv \
  --bind "${DIR_fastq}","${DIR_Ref_Fasta}","${DIR_bam}","${DIR_tmp}" \
  "${PanSVScope_SIF}" \
  PanSVScope mapr \
    --ID="${SampleID}" \
    --GENOME="${REF_FASTA}" \
    --R1="${IN_R1}" \
    --R2="${IN_R2}" \
    --OUTDIR="${DIR_bam}" \
    --TMPDIR="${DIR_tmp}" \
    --THREADS="${SLURM_CPUS_PER_TASK}" \
    --BWA=/opt/PanSVScope/Bin/bwa \
    --SAMBAMBA=/opt/PanSVScope/Bin/sambamba

# ---- Module 2: svcall ---------------------------------------------------
singularity exec --cleanenv \
  --env PREPEND_PATH=/opt/envs/legacy/bin \
  --bind "${DIR_bam}","${DIR_Ref_Fasta}","${DIR_svcall}" \
  "${PanSVScope_SIF}" \
  PanSVScope svcall \
    --id="${SampleID}" \
    --bam="${IN_Bam}" \
    --genome="${REF_FASTA}" \
    --outdir="${DIR_svcall}" \
    --threads="${SLURM_CPUS_PER_TASK}" \
    --clean-temp \
    --manta=/opt/envs/legacy/bin/configManta.py \
    --Pythonformanta=/opt/envs/legacy/bin/python \
    --delly=/opt/PanSVScope/Bin/delly \
    --wham=/opt/envs/main/bin/whamg \
    --smoove=/opt/envs/legacy/bin/smoove \
    --dysgu=/opt/envs/dysgu/bin/dysgu \
    --matchclips=/opt/PanSVScope/Bin/matchclips \
    --survivor=/opt/PanSVScope/Bin/SURVIVOR \
    --bcftools=/opt/envs/main/bin/bcftools \
    --samtools-bindir=/opt/envs/main/bin

# ---- Module 3: pangra ---------------------------------------------------
singularity exec --cleanenv \
  --bind "${DIR_svcall}","${DIR_Ref_Fasta}","${DIR_known_sv}","${DIR_pgenome}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope pangra \
    --workdir="${DIR_out}" \
    --genome="${REF_FASTA}" \
    --sv-out-list="${SVOUT_LIST}" \
    --known-sv="${KNOWN_SV}" \
    --pgenome="${PGENOME_VCF}" \
    --threads="${SLURM_CPUS_PER_TASK}" \
    --clean \
    --bedtools=/opt/envs/main/bin/bedtools \
    --vcfwave=/opt/envs/main/bin/vcfwave \
    --check-sv-tools=/opt/PanSVScope/Bin/CheckSV \
    --markdup=/opt/PanSVScope/Bin/Markdup \
    --minimap2=/opt/PanSVScope/Bin/minimap2 \
    --graphvcf=/opt/PanSVScope/Bin/graphvcf \
    --vg=/opt/PanSVScope/Bin/vg \
    --pangenie-index=/opt/PanSVScope/Bin/PanGenie-index \
    --parallel=/opt/PanSVScope/Bin/parallel \
    --bgzip=/opt/envs/main/bin/bgzip \
    --tabix=/opt/envs/main/bin/tabix

# ---- Module 4: svgeno ---------------------------------------------------
singularity exec --cleanenv \
  --bind "${DIR_bam}","${DIR_fastq}","${DIR_Ref_Fasta}","${DIR_Index}","${DIR_svgeno}" \
  "${PanSVScope_SIF}" \
  PanSVScope svgeno \
    --id="${SampleID}" \
    --workdir="${DIR_svgeno}" \
    --index="${DIR_Index}" \
    --genome="${REF_FASTA}" \
    --bam="${IN_Bam}" \
    --r1="${IN_R1}" \
    --r2="${IN_R2}" \
    --enable-vg-giraffe \
    --enable-bayestyper \
    --enable-graphtyper2 \
    --threads="${SLURM_CPUS_PER_TASK}" \
    --vg=/opt/PanSVScope/Bin/vg \
    --bayestyper-tools=/opt/PanSVScope/Bin/bayesTyperTools \
    --bayestyper=/opt/PanSVScope/Bin/bayesTyper \
    --kmc=/opt/PanSVScope/Bin/kmc \
    --graphtyper=/opt/PanSVScope/Bin/graphtyper \
    --sambamba=/opt/PanSVScope/Bin/sambamba \
    --graphvcf=/opt/PanSVScope/Bin/graphvcf \
    --bgzip=/opt/envs/main/bin/bgzip

# ---- Module 5: svmer ----------------------------------------------------
singularity exec --cleanenv \
  --bind "${DIR_Index}","${DIR_vcf_list}","${DIR_svgeno}","${DIR_out}" \
  "${PanSVScope_SIF}" \
  PanSVScope svmer \
    --workdir="${DIR_out}" \
    --SVindex="${DIR_Index}" \
    --Vcflist="${VCF_LIST}" \
    --BGZIP=/opt/envs/main/bin/bgzip

echo "All modules completed."
```

**Tips for SLURM:**

- `--cpus-per-task` determines the value of `${SLURM_CPUS_PER_TASK}`, which is
  passed to PanSVScope via `--THREADS`.
- If a module exhausts memory, either increase `--mem`, or enable PanSVScope's
  resume feature (`svgeno`) and rerun the job — completed tools will be
  skipped.
- Use `--cleanenv` **every time** to prevent host environment variables from
  leaking into the container and shadowing the bundled tools.
- If your cluster uses Apptainer instead of Singularity, replace `singularity`
  with `apptainer` — the command‑line interface is identical.

---

## Troubleshooting

### “Permission denied” when reading the SIF

Make the image executable:

```bash
chmod +x ${PanSVScope_SIF}
```

### PanSVScope cannot find an input file

Almost always a **missing bind mount**. Double‑check that every directory
containing an input or output path passed to PanSVScope is listed in
`--bind`.

Quick check:

```bash
singularity exec --bind /data:/data ${PanSVScope_SIF} ls /data
```

### “command not found” for a tool

All bundled tools live under `/opt/PanSVScope/Bin/` and the three conda
environments under `/opt/envs/{main,legacy,dysgu}/bin/`. Use the absolute
paths shown in the examples above; do **not** rely on `PATH` inside the
container.

### Manta or Smoove fails with a Python error

These tools require **Python 2.7**. Make sure you pass
`--env PREPEND_PATH=/opt/envs/legacy/bin` and use
`--Pythonformanta=/opt/envs/legacy/bin/python`.

### Out‑of‑memory during `svgeno`

Re‑invoke the same command. Tools that already succeeded will be skipped
automatically. Only the failed tool will be rerun. Add `--force` only if you
want to rerun **all** enabled tools.

### Reference genome has lowercase bases

Convert to uppercase before running PanSVScope:

```bash
seqtk seq -A -U ref.fa -l 60 > ref.upper.fa
```

Then re‑index (BWA, samtools faidx) and use `ref.upper.fa` as `--GENOME` /
`--genome`.

### `--cleanenv` removes my custom variables

`--cleanenv` only clears **environment variables** inside the container.
Your shell variables (e.g., `${SampleID}`, `${REF_FASTA}`) are expanded by the
host shell **before** `singularity exec` runs, so they are unaffected.
