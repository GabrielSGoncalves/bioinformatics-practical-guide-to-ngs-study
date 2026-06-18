# bioinformatics-practical-guide-to-ngs-study

Study notes and exercises following **_Bioinformatics: A Practical Guide to Next Generation Sequencing Data Analysis_** by Hamid D. Ismail.

## About

This repo tracks my progress through the book, chapter by chapter — notes, commands, and small scripts as I work through the material.

## Progress

| Chapter | Topic | Status |
|---------|-------|--------|
| 1 | Introduction to NGS, QC & preprocessing (SRA Toolkit, FastQC, FASTX-Toolkit, Trimmomatic) | ✅ Done |
| 2 | Mapping reads to a reference genome (reference genomes, BWA, samtools) | 🚧 In progress |

## Chapter 1 — NCBI SRA & the SRA Toolkit

The [Sequence Read Archive (SRA)](https://www.ncbi.nlm.nih.gov/sra) is NCBI's public repository of high-throughput sequencing reads. The **SRA Toolkit** provides command-line tools to download and convert that data.

### Key accession prefixes

| Prefix | Refers to |
|--------|-----------|
| `SRPxxxxxx` | Study / project |
| `SRSxxxxxx` | Sample |
| `SRXxxxxxx` | Experiment |
| `SRRxxxxxx` | Run (the actual read data you download) |

### Running the SRA Toolkit via Docker (preferred)

Bioinformatics tools in this repo run inside Docker containers — no local installs.
The official image is [`ncbi/sra-tools`](https://hub.docker.com/r/ncbi/sra-tools).

A [`docker-compose.yml`](./docker-compose.yml) wires it up so output lands in `./fastqs`
on the host and the download cache persists in a named volume between runs. This mirrors
the book's `mkdir fastqs && cd fastqs` workflow — instead of `cd`-ing in, the container's
working directory is mounted to `./fastqs`.

```bash
# Pull the image
docker compose pull sra-tools

# The tools are one-shot commands — run them through `docker compose run`:
docker compose run --rm sra-tools <tool> [args]
```

Behind the scenes the service mounts `./fastqs` → `/output` and sets `/output` as the
working directory, so every tool writes its results into `./fastqs/`.

#### Plain `docker run` (equivalent, no compose)

```bash
docker run -t --rm -v "$PWD/fastqs:/output:rw" -w /output ncbi/sra-tools fasterq-dump --verbose SRR030834
```

### Chapter 1 workflow (book example: run `SRR030834`, single-end)

```bash
# Download + convert the run to FASTQ, straight into ./fastqs
docker compose run --rm sra-tools fasterq-dump --verbose SRR030834
# -> ./fastqs/SRR030834.fastq
```

For larger or paired-end runs, the book also uses the prefetch-then-dump pattern:

```bash
docker compose run --rm sra-tools prefetch SRR030834
docker compose run --rm sra-tools fasterq-dump -e 2 -p --split-files SRR030834
```

### Handy commands

```bash
# Inspect a run's metadata without downloading the reads
docker compose run --rm sra-tools vdb-dump --info SRR030834

# Peek at the first few spots/reads
docker compose run --rm sra-tools fastq-dump --stdout -X 2 SRR030834
```

> **Note:** the container runs as `root`, so files written to `./fastqs` are owned by
> root. `sudo chown -R "$USER" fastqs` if you need to edit them from the host.

## Quality control — FastQC via Docker

[FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) runs basic QC checks
on raw sequencing reads. The [`fastqc`](https://hub.docker.com/r/staphb/fastqc) service in
`docker-compose.yml` uses the `staphb/fastqc:0.12.1` image and mounts `./fastqs` → `/data`,
so reports land right next to the FASTQ files.

```bash
# Run FastQC on a downloaded run; writes <name>_fastqc.html + .zip into ./fastqs
docker compose run --rm fastqc fastqc SRR030834.fastq
# -> ./fastqs/SRR030834_fastqc.html
#    ./fastqs/SRR030834_fastqc.zip
```

Open the `.html` report in a browser to review per-base quality, GC content, adapter
content, etc.

## Read preprocessing — FASTX-Toolkit via Docker

The [FASTX-Toolkit](http://hannonlab.cshl.edu/fastx_toolkit/) is a collection of
command-line tools for FASTA/FASTQ preprocessing — quality statistics, trimming,
filtering, clipping, etc. The book (section 1.6) installs it from the legacy
`fastx_toolkit_0.0.13_binaries_Linux_2.6_amd64.tar.bz2` tarball, but rather than
installing binaries locally this repo uses the maintained
[BioContainers image](https://quay.io/repository/biocontainers/fastx_toolkit),
pinned to the same `0.0.13` release: `quay.io/biocontainers/fastx_toolkit:0.0.13--0`.

The `fastx-toolkit` service in `docker-compose.yml` mounts `./fastqs` → `/data`, so
its tools read and write FASTQ files right alongside the downloaded reads.

```bash
# Per-base quality statistics for a run
docker compose run --rm fastx-toolkit \
  fastx_quality_stats -i SRR030834.fastq -o SRR030834_stats.txt

# Quality-filter reads (keep reads with >=90% of bases at quality >=20)
docker compose run --rm fastx-toolkit \
  fastq_quality_filter -q 20 -p 90 -i SRR030834.fastq -o SRR030834_filtered.fastq
```

Any FASTX tool works the same way — `fastx_trimmer`, `fastx_clipper`,
`fastq_quality_trimmer`, `fastx_collapser`, etc. Run a tool with `-h` to see its options:

```bash
docker compose run --rm fastx-toolkit fastx_trimmer -h
```

## Read trimming — Trimmomatic via Docker

[Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) is a flexible read
trimmer for Illumina data — adapter removal, quality trimming, and length filtering for
both single-end and paired-end reads. The book downloads
[`Trimmomatic-0.39.zip`](http://www.usadellab.org/cms/uploads/supplementary/Trimmomatic/Trimmomatic-0.39.zip)
and runs it as `java -jar trimmomatic-0.39.jar`, but this repo uses the maintained
[`staphb/trimmomatic`](https://hub.docker.com/r/staphb/trimmomatic) image pinned to the
same `0.39` release. The image ships a `trimmomatic` wrapper, so you don't have to call
`java -jar` yourself, and the bundled adapter FASTAs live at
`/Trimmomatic-0.39/adapters/` inside the container (`TruSeq3-SE.fa`, `TruSeq3-PE.fa`,
`NexteraPE-PE.fa`, etc.).

The `trimmomatic` service in `docker-compose.yml` mounts `./fastqs` → `/data`, so it reads
and writes FASTQ files alongside the downloaded reads.

```bash
# Sanity check — prints 0.39
docker compose run --rm trimmomatic trimmomatic -version

# Single-end: quality-trim with a sliding window and drop reads shorter than 36 bp
docker compose run --rm trimmomatic \
  trimmomatic SE -phred33 bad.fastq bad_trimmed.fastq SLIDINGWINDOW:4:20 MINLEN:36

# Paired-end: 4 outputs (paired + unpaired for each mate), with adapter clipping
docker compose run --rm trimmomatic \
  trimmomatic PE -phred33 \
    SRR957824_1.fastq SRR957824_2.fastq \
    out_1P.fastq out_1U.fastq out_2P.fastq out_2U.fastq \
    ILLUMINACLIP:/Trimmomatic-0.39/adapters/TruSeq3-PE.fa:2:30:10 \
    SLIDINGWINDOW:4:20 MINLEN:36
```

Common steps you can chain (order matters — they run left to right): `ILLUMINACLIP`
(adapter removal), `SLIDINGWINDOW:<size>:<quality>`, `LEADING:<q>`/`TRAILING:<q>`,
`CROP:<len>`/`HEADCROP:<len>`, and `MINLEN:<len>`.

## Chapter 1 preprocessing workflow (section 1.6)

The book walks through a small end-to-end example: download a **paired-end** run
(`SRR957824`), deliberately keep only one mate and rename it to `bad.fastq`, then run
FastQC on it to see what a low-quality read set looks like. The original commands assume
local installs and a GUI browser:

```bash
# Book version (local installs)
mkdir preprocessing
cd preprocessing
fasterq-dump --verbose SRR957824
rm SRR957824_1.fastq
mv SRR957824_2.fastq bad.fastq
fqfile=$(ls *.fastq)
fastqc $fqfile
htmlfile=$(ls *.html)
firefox $htmlfile
```

Adapted to this repo's Docker setup, working inside `./fastqs/preprocessing` (mounted as
`/output` in `sra-tools` and `/data` in `fastqc`):

```bash
# 1. Make the working subdir on the host
mkdir -p fastqs/preprocessing

# 2. Download the paired-end run -> SRR957824_1.fastq + SRR957824_2.fastq
docker compose run --rm --workdir /output/preprocessing \
  sra-tools fasterq-dump --verbose SRR957824

# 3. Drop mate 1, rename mate 2 to bad.fastq.
#    Run inside the container so you don't fight root-owned files with sudo.
docker compose run --rm --workdir /output/preprocessing \
  sra-tools bash -c 'rm SRR957824_1.fastq && mv SRR957824_2.fastq bad.fastq'

# 4. QC the remaining fastq, reusing the book's `ls *.fastq` idiom
docker compose run --rm --workdir /data/preprocessing \
  fastqc bash -c 'fqfile=$(ls *.fastq); fastqc "$fqfile"'
# -> fastqs/preprocessing/bad_fastqc.html + bad_fastqc.zip
```

Instead of `firefox $htmlfile`, just open the report on the host — e.g.
`xdg-open fastqs/preprocessing/bad_fastqc.html` (or double-click it in a file manager).

> **Heads up:** `SRR957824` is ~296 MB compressed and expands to two ~640 MB FASTQ
> files. The `./fastqs` directory is gitignored; delete `fastqs/preprocessing` when you're
> done to reclaim the space.

## Chapter 2 — Mapping reads to a reference genome

Once reads pass QC, Chapter 2 maps them onto a **reference genome**: download and index
the reference, align the reads with [BWA](https://github.com/lh3/bwa), then turn the
aligner's text output into a compact, sorted, indexed binary with
[samtools](https://www.htslib.org/). Two new Docker services back this chapter:

- **`bwa`** — `staphb/bwa:0.7.17` (indexing + BWA-MEM alignment)
- **`samtools`** — `staphb/samtools:1.21` (SAM ⇄ BAM, sort, index, stats)

Both mount the reference genome at `./refgenome` → `/ref` and the reads/alignment output
at `./fastqs` → `/data` (the working directory), so a single `docker compose run` can see
the reference and the reads at the same time.

```bash
docker compose pull bwa samtools
```

> **Note:** `./refgenome` is gitignored (reference FASTAs and their index files are large),
> as are `*.sam` / `*.bam` / `*.bai`. The full human reference is ~900 MB compressed and its
> BWA index expands to several GB — keep an eye on disk space.

### 2.1 The reference genome

The reference genome is the assembled sequence the reads get aligned against. The book uses
the human genome **GRCh38.p13**, distributed by NCBI as a gzipped FASTA (`.fna.gz`). The
key gotcha: you must point at the **actual file**, not the directory that contains it —
fetching the directory URL just downloads NCBI's HTML index page.

> ⚠️ The `refgenome/GRCh38.p13_ref.fna.gz` already in this repo is exactly that mistake —
> `file` reports it as `HTML document, ASCII text` (an "Index of /genomes/all/GCF/000/001/405"
> listing) whose first bytes are `<!DOCTYPE HTML`, not gzip. Re-download it from one of the
> sources below before indexing.

#### Download sources

All four URLs below were verified to return real gzip data (not an HTML page). The book
follows NCBI **RefSeq** (`GCF`), so prefer Option 1 — the README's `bwa index`/alignment
commands all reference `GCF_000001405.39_GRCh38.p13_genomic.fna`. The others are the same
GRCh38 assembly but use different chromosome naming, which ripples into any downstream
BED/VCF/annotation files.

```bash
cd refgenome

# Option 1 — NCBI RefSeq (GCF, matches the book) — ~964 MB, names like NC_000001.11
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/001/405/GCF_000001405.39_GRCh38.p13/GCF_000001405.39_GRCh38.p13_genomic.fna.gz

# Option 2 — NCBI GenBank (GCA) — ~965 MB. Note GRCh38.p13 = GCA_000001405.28 (not .24)
wget https://ftp.ncbi.nlm.nih.gov/genomes/all/GCA/000/001/405/GCA_000001405.28_GRCh38.p13/GCA_000001405.28_GRCh38.p13_genomic.fna.gz

# Option 3 — Ensembl primary assembly — ~882 MB, clean names (1, 2, ... X, Y, MT)
wget https://ftp.ensembl.org/pub/release-111/fasta/homo_sapiens/dna/Homo_sapiens.GRCh38.dna.primary_assembly.fa.gz

# Option 4 — UCSC hg38 — ~984 MB, chr-prefixed names (chr1, chr2, ...)
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz

cd ..
```

Alternatively, the [NCBI `datasets` CLI](https://www.ncbi.nlm.nih.gov/datasets/) resolves the
path for you so you never hand-build a URL: `datasets download genome accession GCF_000001405.39 --include genome`.

#### Validate the download, then unzip

Always confirm you got gzip and not an HTML error page — this is the check that catches the
broken-download mistake:

```bash
# Real FASTA gz reports "gzip compressed data"; a bad one reports "HTML document"
file refgenome/*.gz
# or check the magic bytes directly — must start with 1f8b
xxd refgenome/GCF_000001405.39_GRCh38.p13_genomic.fna.gz | head -1

# Once validated, decompress (BWA needs the reference uncompressed to build its index)
gunzip refgenome/GCF_000001405.39_GRCh38.p13_genomic.fna.gz
# -> refgenome/GCF_000001405.39_GRCh38.p13_genomic.fna

# Final sanity check — should print a FASTA header line (>...), not HTML
docker compose run --rm samtools head -1 /ref/GCF_000001405.39_GRCh38.p13_genomic.fna
```

### 2.2 Indexing the reference & aligning with BWA

Aligners can't scan a multi-gigabase genome linearly for every read, so the reference is
first turned into an **index** (a Burrows–Wheeler / FM-index). `bwa index` writes five
sidecar files next to the FASTA (`.amb`, `.ann`, `.bwt`, `.pac`, `.sa`); `samtools faidx`
adds a `.fai` that lets tools seek into the FASTA by coordinate.

```bash
# Build the BWA index (one-time; slow + memory-hungry for the full human genome)
docker compose run --rm bwa bwa index /ref/GCF_000001405.39_GRCh38.p13_genomic.fna

# Build the samtools FASTA index (.fai) — handy for later steps
docker compose run --rm samtools \
  samtools faidx /ref/GCF_000001405.39_GRCh38.p13_genomic.fna
```

With the index in place, align reads with **BWA-MEM** (the algorithm for ≥70 bp reads).
The working directory is `/data` (= `./fastqs`), so reads are referenced by bare filename
and the SAM lands back in `./fastqs`. BWA 0.7.17 supports `-o`, so no shell redirection is
needed:

```bash
# Single-end alignment -> ./fastqs/SRR030834.sam
docker compose run --rm bwa \
  bwa mem -o SRR030834.sam \
    /ref/GCF_000001405.39_GRCh38.p13_genomic.fna \
    SRR030834.fastq

# Paired-end: pass both mates
docker compose run --rm bwa \
  bwa mem -o SRR957824.sam \
    /ref/GCF_000001405.39_GRCh38.p13_genomic.fna \
    SRR957824_1.fastq SRR957824_2.fastq
```

SAM is a verbose text format — convert it to a **sorted, indexed BAM** with samtools so
downstream tools (and IGV) can use it efficiently:

```bash
# SAM -> coordinate-sorted BAM (samtools sort reads SAM and writes BAM directly)
docker compose run --rm samtools \
  samtools sort -o SRR030834.sorted.bam SRR030834.sam

# Index the sorted BAM -> SRR030834.sorted.bam.bai (enables random access by region)
docker compose run --rm samtools \
  samtools index SRR030834.sorted.bam

# Mapping summary — total reads, % mapped, properly paired, etc.
docker compose run --rm samtools samtools flagstat SRR030834.sorted.bam
```

## Repo layout

```
.
├── README.md
├── docker-compose.yml   # sra-tools + fastqc + fastx-toolkit + trimmomatic + bwa + samtools, mounts -> ./fastqs (+ ./refgenome)
├── refgenome/           # reference genome FASTA + aligner/samtools indexes (gitignored, mounted as /ref)
└── fastqs/              # FASTQ / .sra output + QC reports + SAM/BAM (gitignored, mounted as /data)
    └── preprocessing/   # section 1.6 workflow scratch space (also gitignored)
```

## References

- Book: *Bioinformatics: A Practical Guide to Next Generation Sequencing Data Analysis* — Hamid D. Ismail
- [NCBI SRA](https://www.ncbi.nlm.nih.gov/sra)
- [SRA Toolkit documentation](https://github.com/ncbi/sra-tools/wiki)
- [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) · [staphb/fastqc image](https://hub.docker.com/r/staphb/fastqc)
- [FASTX-Toolkit](http://hannonlab.cshl.edu/fastx_toolkit/) · [biocontainers/fastx_toolkit image](https://quay.io/repository/biocontainers/fastx_toolkit)
- [Trimmomatic](http://www.usadellab.org/cms/?page=trimmomatic) · [staphb/trimmomatic image](https://hub.docker.com/r/staphb/trimmomatic)
- [BWA](https://github.com/lh3/bwa) · [staphb/bwa image](https://hub.docker.com/r/staphb/bwa)
- [samtools / HTSlib](https://www.htslib.org/) · [staphb/samtools image](https://hub.docker.com/r/staphb/samtools)
- [NCBI GRCh38.p13 reference (GCF_000001405.39)](https://ftp.ncbi.nlm.nih.gov/genomes/all/GCF/000/001/405/GCF_000001405.39_GRCh38.p13/)
