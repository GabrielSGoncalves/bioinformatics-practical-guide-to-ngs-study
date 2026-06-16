# bioinformatics-practical-guide-to-ngs-study

Study notes and exercises following **_Bioinformatics: A Practical Guide to Next Generation Sequencing Data Analysis_** by Hamid D. Ismail.

## About

This repo tracks my progress through the book, chapter by chapter — notes, commands, and small scripts as I work through the material.

## Progress

| Chapter | Topic | Status |
|---------|-------|--------|
| 1 | Introduction to NGS & public sequence databases (NCBI SRA) | 🚧 In progress |

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

## Repo layout

```
.
├── README.md
├── docker-compose.yml   # sra-tools + fastqc containers, output -> ./fastqs
└── fastqs/              # FASTQ / .sra output + QC reports (gitignored, mounted into the containers)
```

## References

- Book: *Bioinformatics: A Practical Guide to Next Generation Sequencing Data Analysis* — Hamid D. Ismail
- [NCBI SRA](https://www.ncbi.nlm.nih.gov/sra)
- [SRA Toolkit documentation](https://github.com/ncbi/sra-tools/wiki)
- [FastQC](https://www.bioinformatics.babraham.ac.uk/projects/fastqc/) · [staphb/fastqc image](https://hub.docker.com/r/staphb/fastqc)
