# RNA-seq analysis of _Culex pipiens_

- Author: <Antonio Guadalupe Faustino Alcalá Madrigal>
- Date created: <11/07/2026>
- Date last modified: <11/07/2026>
- This was run at the <Pitzer> cluster of the Ohio Supercomputer Center (<https://www.osc.edu>)
- OSC working dir: </fs/ess/PAS3493/people/alcalamadrigal1/GA5>
- GitHub repository: https://github.com/AlcalaMadrigal1/GA5

## Prerequisites

- **Input files**:
  - Paired-end RNA-seq reads are in `../garrigos/fastq`

- **Software**:
  - The scripts run programs using Apptainer containers,
    with links contained inside the scripts.
  - If running at OSC, no software installations are needed.
  - If you're not at OSC, you need to have Apptainer and Slurm installed.

- Run all code from the working dir listed above.
  You don't need to create the output dirs, because the scripts will do so.

- Each script writes a Slurm log file (`slurm-<script>-<jobID>.out`) to the working dir.
  After the jobs of a step have finished, check these logs for errors,
  then move them into a `logs/` dir within that step's output dir.


## Set up: Download the reference transcriptome FASTA file from NCBI

The reference transcriptome is from NCBI RefSeq, assembly GCF_016801865.2_TS_CPP_V2 (*Culex pipiens*). The file was downloaded on November 7, 2026.

```bash
mkdir -p data/ref/

wget -P data/ref/ https://ftp.ncbi.nlm.nih.gov/genomes/refseq/invertebrate/Culex_pipiens/latest_assembly_versions/GCF_016801865.2_TS_CPP_V2/GCF_016801865.2_TS_CPP_V2_rna.fna.gz
```

### Reference file integrity verification

The SHA-256 checksum of the reference transcriptome used in this analysis is:

```text
c5e96fc85a19b572acd40af808cd239f7eef19d5578721e3b8811a3623284b24
```

To verify the integrity of the downloaded reference file, run:

```bash
echo "c5e96fc85a19b572acd40af808cd239f7eef19d5578721e3b8811a3623284b24  data/ref/GCF_016801865.2_TS_CPP_V2_rna.fna.gz" | sha256sum -c -
```

The expected result is `OK`, confirming that the reference file is identical to the one used in this analysis.


## Set up: Set file paths

```bash
# Inputs:
fastq_dir=../garrigos/fastq
ref_transcripts=data/ref/GCF_016801865.2_TS_CPP_V2_rna.fna.gz

# Base output dir:
outdir=results
```

## Step 1: Read trimming with TrimGalore


```bash
# Run Trim Galore for sample S63
sbatch scripts/trimgalore.sh ../garrigos/fastq/S63_R1.fastq.gz ../garrigos/fastq/S63_R2.fastq.gz results/trimgalore

# Check the job status
squeue -j 57111236

# Inspect the Slurm log
cat slurm-trimgalore-57111236.out

# Submit Trim Galore jobs for the remaining 21 samples
for sample in S{64..71} S{74..86}; do
    sbatch scripts/trimgalore.sh \
        ../garrigos/fastq/${sample}_R1.fastq.gz \
        ../garrigos/fastq/${sample}_R2.fastq.gz \
        results/trimgalore
done

# Check the status of submitted jobs
squeue -u $USER

# Count successfully completed jobs
grep -l "Successfully finished script trimgalore.sh" slurm-trimgalore-*.out | wc -l
# Result: 22

# Count trimmed FASTQ files
find results/trimgalore/ -maxdepth 1 -name '*.fastq.gz' | wc -l
# Result: 44

# Move Slurm logs to the logs directory
mv slurm-trimgalore-*.out results/trimgalore/logs/

# Verify the number of Slurm log files
ls results/trimgalore/logs/slurm-trimgalore-*.out | wc -l
# Result: 22
```


## Step 2: Creating a transcriptome index with Salmon


```bash
# Step 2: Build Salmon transcriptome index

sbatch scripts/salmon_index.sh \
    data/ref/GCF_016801865.2_TS_CPP_V2_rna.fna.gz \
    results/salmon_index

# Check job status
squeue -j 57112779

# Inspect the Slurm log
cat slurm-salmon_index-57112779.out

# Inspect the generated index files
ls -lh results/salmon_index/

# Move the Slurm log to the logs directory
mv slurm-salmon_index-57112779.out results/salmon_index/logs/

# Verify the archived log
ls -lh results/salmon_index/logs/
```

## Step 3: Pseudo-alignment and quantification with Salmon

#Quantify transcript abundance for all 22 paired-end RNA-seq samples using Salmon.

```bash
# Run Salmon for the first sample (S63)
sbatch scripts/salmon_quant.sh \
    results/trimgalore/S63_R1.fastq.gz \
    results/trimgalore/S63_R2.fastq.gz \
    results/salmon_index \
    results/salmon_quant

# Run Salmon for the remaining 21 samples
for sample in S{64..71} S{74..86}; do
    sbatch scripts/salmon_quant.sh \
        results/trimgalore/${sample}_R1.fastq.gz \
        results/trimgalore/${sample}_R2.fastq.gz \
        results/salmon_index \
        results/salmon_quant
done

# Check job status
squeue -u $USER

# Count successfully completed jobs
grep -l "Successfully finished script salmon_quant.sh" \
    slurm-salmon_quant-*.out | wc -l

# Count the generated quantification files
find results/salmon_quant/ -name 'quant.sf' | wc -l

# Archive Slurm output logs
mv slurm-salmon_quant-*.out results/salmon_quant/logs/

# Verify the number of archived logs
find results/salmon_quant/logs/ \
    -name 'slurm-salmon_quant-*.out' | wc -l
```

All 22 Salmon quantification jobs completed successfully. A total of 22 `quant.sf` files were generated, and all 22 Slurm logs were archived.


## Step 4: Summarizing the results of all steps with MultiQC
Summarize the results from FastQC, Trim Galore, and Salmon using MultiQC.

```bash
# Run MultiQC on all results
sbatch scripts/multiqc.sh results results/multiqc

# Check the Slurm job status
squeue -j 57113775

# Inspect the MultiQC output
cat slurm-multiqc-57113775.out

# Archive the Slurm log
mv slurm-multiqc-57113775.out results/multiqc/logs/
```

MultiQC v1.35 successfully generated the report at
`results/multiqc/multiqc_report.html`.

The analysis detected:
- 44 FastQC reports
- 44 Trim Galore reports
- 22 Salmon metadata reports
- 22 Salmon fragment length distribution reports
- 22 Salmon library format counts reports

The 44 FastQC and Trim Galore reports correspond to two paired-end read files (R1 and R2) for each of the 22 samples. Salmon generated one set of quantification reports per sample.
