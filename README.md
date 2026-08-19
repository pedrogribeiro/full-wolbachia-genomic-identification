# Wolbachia Genomic Identification Pipeline

Scripts accompanying the manuscript:

> **"Genomic identification of Wolbachia reveals widespread signals of codivergence in the butterfly Eudaminae subfamily"**

All shell scripts are PBS-scheduled HPC jobs developed and tested on the CESNET MetaCentrum cluster. Paths must be adapted to your local system. A `samples.txt` file listing one sample ID per line is required by all array jobs (see `resources/`).

---

## Repository structure

```
.
├── shell_scripts/     # PBS shell scripts — numbered in execution order
├── r_scripts/         # R scripts for visualisation and post-processing
├── resources/         # Static input files (samples list, reference genomes, loci)
└── docs/              # Extended notes and dependency information
```

---

## Pipeline overview

```
Raw reads (FASTQ)
      │
      ▼
[01] FastQC (raw)  ──►  [02] MultiQC (raw)
      │
      ▼
[03] Fastp (trimming)
      │
      ▼
[04] FastQC (trimmed)
      │
      ▼
[05] Bowtie2 — map to combined Wolbachia reference (supergroup A + B)
      │
      ▼
[06] Samtools — SAM→sorted BAM → split by supergroup (A / B)
      │
      ▼
[07] Bedtools — BAM→FASTQ per supergroup          (SUPERGROUP = A or B)
      │
      ▼
[08] SPAdes — assemble Wolbachia contigs           (SUPERGROUP = A or B)
      │
      ▼
[08b] collect SPAdes contigs → flat directory for SECAPR
      │
      ├──────────────────────────────────────────────────────────────────┐
      │  Curated loci database construction                              │
      │  [09] BLAST — blastn 210-loci reference vs supergroup genome     │
      │  [10] AWK  — filter by bitscore/pident → curated loci FASTA     │
      └──────────────────────────────────────────────────────────────────┘
      │
      ▼
      Validation
      [11] BLASTX  — assembled contigs vs Wolbachia protein database
      [12] AWK     — filter BLASTX results
      [13] SECAPR find_target_contigs — extract loci from assemblies
      [14] SECAPR align_sequences     — align extracted contigs per locus
      [15] MAFFT --addfull            — add samples to curated reference alignment (trim introns)
      [16] BUSCO                      — assembly completeness check
      │
      ▼
      Coverage / cleaning
      [17] Bowtie2 remap + Samtools coverage
      [R1] R — coverage plots
      │
      ▼
      Phylogenetics
      [18] FASTA preparation (linearise, gap→N, fill missing)
      [19] catfasta2phyml — concatenate loci → phylip + partitions
      [20] PartitionFinder — best partitioning scheme
      [21] IQ-TREE 2 — maximum-likelihood phylogeny
```

---

## Script reference

| Script | Stage | Type | Notes |
|---|---|---|---|
| `01_fastqc_raw.pbs` | QC | Array | Per sample |
| `02_multiqc_raw.pbs` | QC | Single | After all FastQC jobs |
| `03_fastp_trim.pbs` | QC | Array | Per sample |
| `04_fastqc_trimmed.pbs` | QC | Array | Per sample |
| `05_bowtie2_map_wolbachia.pbs` | Identification | Array | Index built once; per sample mapping |
| `06_samtools_sort_split.pbs` | Identification | Array | SAM→BAM→sorted→split by supergroup |
| `07_bedtools_bam2fastq.pbs` | Identification | Array | Toggle SUPERGROUP=A/B; submit twice |
| `08_spades_assembly.pbs` | Identification | Array | Toggle SUPERGROUP=A/B; submit twice |
| `08b_collect_spades_contigs.pbs` | Identification | Single | Collect all contigs into flat dir; toggle SUPERGROUP=A/B; submit twice |
| `09_blast_curated_loci.pbs` | Loci database | Single | Toggle SUPERGROUP=A/B; submit twice |
| `10_filter_blast_curated_loci.pbs` | Loci database | Single | Toggle SUPERGROUP=A/B; submit twice |
| `11_blastx_validation.pbs` | Validation | Array | Toggle SUPERGROUP=A/B; submit twice |
| `12_filter_blastx.pbs` | Validation | Array | Toggle SUPERGROUP=A/B; submit twice |
| `13_secapr_find_targets.pbs` | Validation | Single | Toggle SUPERGROUP=A/B; submit twice |
| `14_secapr_align.pbs` | Validation | Single | Toggle SUPERGROUP=A/B; submit twice |
| `15_mafft_addfull_loci.pbs` | Validation | Single | Toggle SUPERGROUP=A/B; submit twice |
| `16_busco_assembly.pbs` | Validation | Array | Toggle SUPERGROUP=A/B; submit twice |
| `17_coverage_remap.pbs` | Coverage | Array | Toggle SUPERGROUP=A/B; submit twice |
| `18_prep_concat_fastas.pbs` | Phylogenetics | Single | Toggle SUPERGROUP=A/B; submit twice |
| `19_catfasta2phyml_concat.pbs` | Phylogenetics | Single | Toggle SUPERGROUP=A/B; submit twice |
| `20_partition_finder.pbs` | Phylogenetics | Single | Requires manual cfg file |
| `21_iqtree2_phylogeny.pbs` | Phylogenetics | Single | Toggle SUPERGROUP=A/B; submit twice |
| `r_scripts/01_coverage_plots.R` | Coverage | R | Toggle SUPERGROUP=A/B |

---

## Dependencies

All tools were run inside conda/mamba environments. Exact versions used are reported in the Methods section of the paper.

| Tool | Used in scripts | Recommended environment |
|---|---|---|
| FastQC | 01, 04 | `qc_env` |
| MultiQC | 02 | `qc_env` |
| Fastp | 03 | `qc_env` |
| Bowtie2 | 05, 17 | `mapping_env` |
| Samtools | 06, 17 | `mapping_env` |
| Bedtools | 07 | `mapping_env` |
| SPAdes | 08 | `spades_env` |
| BLAST+ (blastn) | 09, 10 | `blast_env` |
| BLAST+ (blastx) | 11, 12 | `blast_env` |
| SECAPR | 13, 14, 18 | `secapr_env` |
| MAFFT | 15 | `mafft_env` |
| BUSCO | 16 | `busco_env` |
| catfasta2phyml | 19 | `phylo_env` |
| PartitionFinder2 | 20 | `partitionfinder_env` |
| IQ-TREE 2 | 21 | `iqtree_env` |
| R (ggplot2, dplyr, purrr) | 01_coverage_plots.R | local R installation |

---

## Resources

Place the following files in `resources/` before running the pipeline:

| File | Description |
|---|---|
| `resources/samples.txt` | One sample ID per line (e.g. `EMW001`). Used by all array jobs via `$PBS_ARRAY_INDEX`. |
| `resources/references/superA.fasta` | Wolbachia supergroup A reference genome (e.g. wMel, NC_002978.6) |
| `resources/references/superB.fasta` | Wolbachia supergroup B reference genome (e.g. wPip, NC_010981.1) |
| `resources/references/wolbGenomeA.fasta` | Same as superA.fasta — used separately for remapping and BLAST db |
| `resources/references/wolbGenomeB.fasta` | Same as superB.fasta — used separately for remapping and BLAST db |
| `resources/references/210ConsensusWolbLoci.fasta` | 210-loci reference from Wang et al. 2020 |
| `resources/references/wolbachiaProteinConcat.fasta` | Concatenated annotated proteins from both reference genomes |
| `resources/allStrains_ref_supergroupA/` | Per-locus curated multi-strain alignments for supergroup A (one FASTA per locus) |
| `resources/allStrains_ref_supergroupB/` | Per-locus curated multi-strain alignments for supergroup B (one FASTA per locus) |

---

## Usage notes

### Adapting paths
All scripts use a `PROJECT_DIR` variable at the top. Replace the CESNET path with your own project root. All other paths are derived from it.

### Supergroup toggle
Scripts that process supergroups independently include a `SUPERGROUP` variable at the top of the `##### define variables #####` block. Submit each script twice — once with `SUPERGROUP="A"` and once with `SUPERGROUP="B"`.

### Array job range
Adjust the `#PBS -J 1-N` directive to match the number of lines in your `samples.txt`.

### Conda environments
Scripts assume mamba/conda environments with the relevant tools installed. Environment names are defined per script in the `CONDA_ENV` variable and can be changed freely to match your setup.

### PartitionFinder config
Script `20_partition_finder.pbs` requires a manually prepared `partition_finder.cfg` inside `in_partition_finder_supergroupA/` or `in_partition_finder_supergroupB/`. Generate the config from the `partitions.txt` output of script 19 following the PartitionFinder documentation.

---

## Citation

If you use these scripts, please cite the associated manuscript (citation details to be added upon publication).

Reference for the 210-loci dataset:
> Wang, X., Xiong, X., Cao, W., Zhang, C., Werren, J. H., & Wang, X. (2020). Phylogenomic Analysis of Wolbachia Strains Reveals Patterns of Genome Evolution and Recombination. *Genome Biology and Evolution*, 12(12), 2508–2520. https://doi.org/10.1093/gbe/evaa219
