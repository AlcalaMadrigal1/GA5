
[![DOI](https://sandbox.zenodo.org/badge/DOI/10.5072/zenodo.636747.svg)](https://sandbox.zenodo.org/doi/10.5072/zenodo.636747)

# GA5 answers - <Antonio Alcala Madrigal>

------------------------------------------------------------------------

## Part A: Your project dir, Git repo, and protocol

### 1: Fill in the top of the protocol

<!-- No answer needed here -- we check `protocol.md` directly. -->

### 2: Markdown formatting

<!-- No answer needed here -- we check the formatting of both Markdown files. -->

### 3: Create subdirs, and copy the scripts

```bash
mkdir -p scripts data/ref results
```

### 4: Initialize a Git repository, and commit your first files

<!-- No answer needed here -- we check the repo directly. -->

### 5: Download the reference transcriptome


Code to count the number of transcripts:

```bash
zgrep -c '^>' data/ref/GCF_016801865.2_TS_CPP_V2_rna.fna.gz
```

The output of the command was:

```28624
```

*Answer*: The number of transcripts in the file: 28,624.

------------------------------------------------------------------------

## Part B: Read trimming

### 6: Run TrimGalore

*Answer*: All 22 Trim Galore batch jobs completed successfully. I verified this by checking the Slurm job queue using `squeue -u $USER` and examining the output logs. All 22 logs contained the message "Successfully finished script trimgalore.sh".

I also confirmed that 44 trimmed FASTQ files were generated, corresponding to two paired-end files (R1 and R2) for each of the 22 samples. Finally, I moved all 22 Slurm output logs to `results/trimgalore/logs/` and verified their number.

No errors preventing successful completion were observed. Some jobs initially remained pending in the Slurm queue, but they subsequently completed without requiring intervention.

------------------------------------------------------------------------

## Part C: Transcript quantification with Salmon

### 7: Create the Salmon index

Code to check the size of the index dir:

```bash
du -sh results/salmon_index/
```

The output of the command was:

```
1.1G    results/salmon_index/
```

*Answer*: How long it took to create the index, and how you found out:The Salmon index took approximately 22 seconds to create. I determined this by checking the start and end timestamps in the Slurm output log (`slurm-salmon_index-57112779.out`). The job started at 2:11:03 PM and finished at 2:11:25 PM on November 7, 2026.


*Answer*: The size of the index dir: The Salmon index directory occupies 1.1 GB, as determined using the command `du -sh results/salmon_index/`.

### 8: Run Salmon for all samples

<!-- No answer needed here -- we check `protocol.md` directly. -->

### 9: The transcripts with the most reads

*Answer*: What the `NumReads` column contains:
The `NumReads` column contains the estimated number of sequencing fragments assigned to each transcript by Salmon. These values can be fractional because Salmon uses probabilistic methods to estimate transcript abundance, particularly when fragments are compatible with multiple transcripts.

```bash
(head -n 1 results/salmon_quant/S63/quant.sf; \
tail -n +2 results/salmon_quant/S63/quant.sf | sort -k5,5nr | head -n 3)
```

The output of the command was:

```
Name            Length  EffectiveLength  TPM           NumReads
XM_039591534.2  6566    6315.086         14518.656821  24004.889
XM_039573735.1  6445    6194.086         5342.073324   8663.254
XM_039573733.1  6591    6340.086         2876.480244   4774.749
```

### 10: The gene of the top transcript

```bash
zgrep -m 1 '^>XM_039591534.2' data/ref/GCF_016801865.2_TS_CPP_V2_rna.fna.gz
```

The output of the command was:

```text
>XM_039591534.2 PREDICTED: Culex pipiens pallens vitellogenin-A1-like (LOC120426760), mRNA
```

*Answer*: The gene that the top transcript belongs to: The transcript XM_039591534.2 belongs to the predicted vitellogenin-A1-like gene (LOC120426760) in Culex pipiens pallens. This transcript had the highest estimated number of sequencing fragments (NumReads = 24004.889) in sample S63.

### 11: Mapping rates

```bash
for file in results/salmon_quant/S*/aux_info/meta_info.json; do
    sample=$(basename "$(dirname "$(dirname "$file")")")
    rate=$(grep '"percent_mapped"' "$file" | cut -d ':' -f 2 | tr -d ' ,')
    echo "$sample $rate"
done | sort -k2,2n
```

The output of the command was:

```text
S83 76.15040975718468
S85 76.9540290286533
S74 78.27531771788918
S82 78.72909822829908
S68 78.80701178209286
S65 79.10547001678594
S71 79.90465999047863
S75 80.39148943504188
S79 80.40608313734863
S64 81.91516220086585
S66 81.93504834505177
S80 82.17212303963116
S69 82.18239269874552
S70 82.3059863363086
S77 82.3281029418101
S76 82.33958793689634
S63 82.45598120021427
S67 83.04590854943645
S86 83.05318481885998
S84 83.22071223998448
S81 83.67951691407399
S78 84.24900290065264
```

*Answer*: The sample with the lowest mapping rate:
Sample S83 had the lowest Salmon mapping rate at approximately 76.15%, based on the `percent_mapped` values reported in the `aux_info/meta_info.json` files for all 22 samples.
------------------------------------------------------------------------

## Part D: Summarizing all steps with MultiQC

### 12: Update your MultiQC script

<!-- No need to paste the script: we'll look at scripts/multiqc.sh and its Git history directly. -->

### 13: Run MultiQC

*Answer*: For each program, the number of reports that MultiQC found,
and why it found this number:MultiQC detected 22 Salmon metadata reports, 22 fragment length distribution reports, and 22 library format counts reports. This is because Salmon quantified 22 paired-end RNA-seq samples, generating one set of quantification results per sample.

MultiQC also detected 44 Trim Galore reports and 44 FastQC reports. These correspond to the two FASTQ files (R1 and R2) associated with each of the 22 samples, resulting in 44 reports for each program.

The MultiQC analysis completed successfully and generated the report results/multiqc/multiqc_report.html.

### 14: The General Statistics table

*Answer*: The sample with the lowest Salmon mapping rate,
and whether this matches your answer to question 11:  According to the MultiQC General Statistics table, sample S83 had the lowest Salmon mapping rate at 76.15%. This agrees with the result obtained directly from Salmon in Question 11, confirming that MultiQC accurately summarized the mapping statistics.

### 15: Adapter lengths

*Answer*: The most common adapter lengths, whether this surprised you,
and why you think these lengths are the most common: The most commonly detected adapter lengths were 1, 2, and 3 nucleotides, with 1 nucleotide being the most frequent. This was surprising because adapter contamination is generally associated with longer sequences rather than extremely short fragments. One possible explanation is that Trim Galore detected short partial matches to adapter sequences at the ends of reads. Therefore, these short matches do not necessarily indicate the presence of complete adapter sequences.

### 16: Ask Claude about the MultiQC report

*Answer*: What you asked, and what you learned from Claude's answer (in your own words):**Answer:**

I asked Claude to help me understand the adapter-length distribution in the MultiQC report, particularly why 1–3 nucleotide adapter matches were much more frequent than longer matches.

Claude explained that very short matches can occur by chance at the ends of sequencing reads and do not necessarily indicate substantial adapter contamination. It suggested that longer adapter matches, especially those occurring more frequently than expected by chance, would provide stronger evidence of actual adapter contamination.

I learned that the most frequent adapter lengths do not necessarily represent complete adapter sequences. Therefore, the adapter-length distribution should be interpreted alongside other quality-control metrics before drawing conclusions about sequencing quality.

------------------------------------------------------------------------

## Part E: Improving the workflow with genAI

### 17: Claude's review of your protocol and scripts

*Answer*: At least three of Claude's suggestions,
and whether you agree or disagree with each one, and why:I asked Claude Code to review my protocol.md and four shell scripts (trimgalore.sh, salmon_index.sh, salmon_quant.sh, and multiqc.sh) to identify issues affecting reproducibility. Claude provided five recommendations, of which I selected three to discuss.

1. Make the reference transcriptome download reproducible. I agree with this recommendation because the protocol should contain an executable download command specifying the exact NCBI reference version. Recording a checksum would also help verify that future analyses use the same reference file.

2. Document the choice of a non-decoy-aware Salmon index. I agree that methodological decisions should be explained. Although a simpler transcriptome-only index was used for this assignment, it has limitations compared with a decoy-aware index. Documenting this choice would help other researchers understand and evaluate the workflow.

3. Clarify sample selection and Slurm job IDs. I agree that the protocol should explain why S72 and S73 are absent from the sample loops, after verifying the available input files. I also agree that Slurm job IDs should be identified as specific to the original execution rather than reusable identifiers.

Overall, Claude's review helped me recognize that reproducibility requires not only working scripts but also clear documentation of input data, methodological choices, and execution details.

### 18: Address a suggestion

*Answer*: Which suggestion(s) you addressed, and what you changed:I addressed Claude's recommendation to improve the reproducibility of the reference transcriptome download.

I updated protocol.md by replacing the placeholder download command with an executable wget command, documenting the exact NCBI RefSeq assembly version (GCF_016801865.2_TS_CPP_V2), and adding a SHA-256 checksum for the reference FASTA file.

I also included a command to verify the downloaded file against the recorded checksum. I tested this verification on OSC, and it returned OK, confirming that the reference file matches the documented checksum.

These changes improve reproducibility by allowing future users to retrieve the specified reference and verify its integrity without repeating the downstream analyses.

------------------------------------------------------------------------

## Part F: Publish your repo on GitHub

### 19: Create a GitHub repository and push

<!-- No answer needed here -- we check your GitHub repo directly. -->

### 20: Get a (test) DOI with Zenodo

*Answer*: Your Zenodo Sandbox DOI:10.5072/zenodo.636747
