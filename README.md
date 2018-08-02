taffeta
=======

Reproducible analysis and validation of RNA-Seq data

Authors: Maya Shumyatcher, Mengyuan Kan, Blanca Himes.

## Overview

The goal of taffeta is to preform reproducible analysis and validation of RNA-Seq data, as a part of [RAVED pipeline](https://github.com/HimesGroup/raved):
  * Download SRA .fastq data
  * Perform preliminary QC
  * Align reads to a reference genome
  * Perform QC on aligned files
  * Create a report that can be used to verify that sequencing was successful and/or identify sample outliers
  * Perform differential expression of reads aligned to transcripts according to a given reference genome
  * Create a report that summarizes the differential expression results

Generate LSF scripts in each step for HPC use.

## Bioinformatic Tools

* **Alignment and quantification:** STAR, HTSeq, kallisto (older versions used bowtie2, tophat, cufflinks, cummerbund, whose options are no longer available).
* **Differential expression (DE) analysis:** R packages DESeq2 and sleuth
* **QC:** fastqc, trimmomatic, samtools, bamtools, picardtools.
* **Spesis-specific genome reference files:** reference genome fasta, gtf, refFlat, and index files. As our example uses ERCC spike-ins, the reference files include ERCC mix transcripts. 
* **Adapter and primer sequences**: a list of is provided. For fastqc, replace . For trimming we provide adapter and primer sequences for the following types: Ilumina TruSeq single index, Illumina unique dual (UD) index adpter and PrepX. Users can tailor this file by adding sequences from other protocols.
* **Pipeline scripts:** the Python scripts make use of various modules including subprocess, os, argparse, sys.
* **R markdown scripts for summary reporot:** Require various R libraries such as DT, gplots, ggplot2, rmarkdown, RColorBrewer, plyr, dplyr, lattice, ginefilter, biomaRt. Note that the current RMD scripts require pandoc version 1.12.3 or higher to generate HTML report.

## Workflow

### Download data from GEO/SRA

Run **pipeline_scripts/rnaseq\_sra\_download.py** to download .fastq files from SRA. Read in **template_files/rnaseq_sra_download_Rmd_template.txt** from specified directory <i>template_dir</i> to create a RMD script. Ftp addresses for corresponding samples are obtained from SRA SQLite database using R package SRAdb.

> rnaseq\_sra\_download.py --geo\_id <i>GEO_Accession</i> --path\_start <i>output_path</i> --project\_name <i>output_prefix</i> --template\_dir <i>templete_file_directory</i> --fastqc

> bsub < <i>project_name</i>_download.lsf

The option *--pheno_info* refers to using user provided SRA ID for download which is included in the SRA_ID column in the provided phenotype file. If the phenotype file is not provided, use phenotype information from GEO. SRA_ID is retrieved from the field relation.1.

The option *--fastqc* refers to running FastQC for downloaded .fastq files.

**Output files:** 

Fastq files downloaded from SRA are saved in the following directory.

> <i>path_start</i>/<i>project_name</i>_SRAdownload/

Fastqc results of corresponding sample are saved in:

> <i>path_start</i>/<i>sample_name</i>/<i>fastq_name</i>_fastqc.zip

The RMD and corresponding HTML report files:

> <i>path_start</i>/<i>project_name</i>_SRAdownload/<i>project_name</i>__SRAdownload_RnaSeqReport.Rmd
> <i>path_start</i>/<i>project_name</i>_SRAdownload/<i>project_name</i>__SRAdownload\_RnaSeqReport.html

GEO phenotype file generated by RMD reports:

> <i>path_start</i>/<i>project_name</i>_SRAdownload/<i>GEO_Accession</i>_withoutQC.txt

SRA information file generated by RMD reports:

> <i>path_start</i>/<i>project_name</i>_SRAdownload/<i>project_name</i>_sraFile.info

### User-tailored phentoype file

The sample info file used in the following steps should be provided by users.

**Required columns:**

* 'Sample' column containing sample ID
* 'Status' column containing variables of comparison state
* 'R1' and/or 'R2' columns containing full paths of .fastq files

**Other columns:**

'Treatment', 'Disease', 'Donor' (i.e. cell line ID if <i>in vitro</i> treatment is used), 'Tissue', 'ERCC\_Mix' (i.e. ERCC mix ID if ERCC spike-in sample is used), 'protocol' designating sample preparation kit information.

**'Index' column** contains index sequence for each sample. If provided, trim raw .fastq files based on corresponding adapter sequences.

If use data from GEO, most GEO phenotype data do not have index information. However, FastQC is able to detect them as "Overrepresented sequences". Users can tailor the 'Index' column based on FastQC results. We provide a file with most updated adapter and primer sequences for FastQC detection.

An example phenotype file can be found here: **example_files/sample_info_file.txt**. Note that column naming is rigid for the following columns: 'Sample', 'Status', 'Index', 'R1', 'R2', 'ERCC\_Mix', 'Treatment', 'Disease', 'Donor', because pipeline scripts will recognize these name strings, but the column order can be changed.

### Alignment, quantification and QC

1) Run **pipeline_scripts/rnaseq_align_and_qc.py** to: 1) trim adapter and primer sequences if index information is available, 2) run FastQC for (un)trimmed .fastq files, 3) align reads and quantify reads mapped to genes/transcripts, and 5) obtain various QC metrics from .bam files.

Edit **pipeline_scripts/rnaseq_userdefine_variables.py** with a list user-defined variables (e.g. paths of genome reference file, paths of bioinformatics tools, versions of bioinformatics tools), and save the file under an executable search path.

If perform adapter trimming, read in **template_files/rnaseq_adapter_primer_sequences.txt** from specified directory <i>template_dir</i> used as a reference list of index and primer sequences for various library preparation kits.

> rnaseq\_align\_and\_qc.py --project\_name <i>output_prefix</i> --sample\_in <i>sample_info_file.txt</i> --aligner star --ref\_genome hg38 --librar\_type PE --index\_type truseq\_single\_index --strand nonstrand --path\_start <i>output_path</i> --template\_dir <i>templete_file_directory</i>

> for i in *.lsf; do bsub < $i; done

The "--aligner" option indicates aligner should be used (default: star).

The "--ref\_genome" option refers to using selected version of genome reference.

The "--library\_type" option refers to PE (paired-end) or SE (single-end) library.

The "--index\_type" option refers to index used in sample library preparation.

The index types provided in **template_files/rnaseq_adapter_primer_sequences.txt** are:

* truseq\_single\_index (TruSeq Single Indexes)
* illumina\_ud\_sys1 (Illumina UD indexes for NovaSeq, MiSeq, HiSeq 2000/2500)
* illumina\_ud\_sys2 (Illumina UD indexed for MiniSeq, NextSeq, HiSeq 3000/4000)
* prepX (PrepX for Apollo 324 NGS Library Prep System)

**template_files/rnaseq_adapter_primer_sequences.txt** contains four columns (i.e. Type, Index, Description, Sequence). Sequences in the Index column is used to match those in Index column in sample info file. This column naming is rigid.

The list is based on the following resources:

* [illumina adapter sequences](https://support.illumina.com/content/dam/illumina-support/documents/documentation/chemistry_documentation/experiment-design/illumina-adapter-sequences-1000000002694-07.pdf)
* [PrepX RNA-Seq Index Primers and Sequences](https://genome.med.harvard.edu/documents/illumina/IntegenX_Apollo324_mRNA_Seq_Protocol_10012012.pdf)

If users provide new sequences, add the new index type in the 1st column 'Type' and specify it in "--index\_type".

The "--strand" option refers to sequencing that captures both strands (nonstrand) or the 1st synthesized strand (reverse) or the 2nd synthesized strand (forward) of cDNA. If the 2nd strand is synthesized using dUTP, this strand will extinct during PCA amplification, thus only 1st (reverse) strand will be sequenced.

Read sample preparation protocal carefully. Reads not in the specified strand will be discarded. Double check proprotion of reads mapped to no feature category in QC report. If a lot of reads are mapped to 'no feature', the strand option setting is likely incorrect.

**Output files:**

Various output files will be written for each sample in directories structured as:

> <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R1_Trimmed.fastq <br>
> <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R2_Trimmed.fastq <br>
> <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R1_Trimmed\_fastqc.zip <br>
> <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_R2_Trimmed\_fastqc.zip <br>
> <i>path_start</i>/<i>sample_name</i>/<i>sample_name</i>_ReadCount <br>
> <i>path_start</i>/<i>sample_name</i>/<i>aligner</i>_out <br>
> <i>path_start</i>/<i>sample_name</i>/<i>quantification_tool</i>_out <br>

2) Run **pipeline_scripts/rnaseq\_align\_and\_qc\_report.py** to create an HTML report of QC and alignment summary statistics for RNA-seq samples. Read in **template\_files/rnaseq\_align\_and\_qc\_report\_Rmd\_template.txt** from specified directory <i>template_dir</i> to create a RMD script.

If ERCC_Mix column exists in phenotype file, it will report the concordance between ERCC spike-in transcript-level read counts and its molecular concentrations. Read in ERCC molecular concentration file **template_files/ERCC_SpikeIn_Controls_Analysis.txt** from specified directory <i>template_dir</i> which can be downloaded [here](https://assets.thermofisher.com/TFS-Assets/LSG/manuals/cms_095046.txt).

> python rnaseq\_align\_and\_qc\_report.py  --project\_name <i>output_prefix</i> --sample\_in <i>sample_info_file.txt</i> --aligner star --ref\_genome hg38 --library\_type PE --path\_start <i>output_path</i> --template\_dir <i>templete_file_directory</i>

> bsub < <i>project_name</i>_qc.lsf

**Output files:**

This script uses the many output files created in step 1), converts these sample-specific files into matrices that include data for all samples, and then creates an Rmd document.

The report and accompanying files are contained in:

> <i>path_start</i>/<i>project_name</i>_Alignment_QC_Report_<i>aligner</i>/

The RMD and corresponding HTML report files:

> <i>path_start</i>/<i>project_name</i>_Alignment_QC\_Report_<i>aligner</i>/<i>project_name</i>_QC_RnaSeqReport.Rmd
> <i>path_start</i>/<i>project_name</i>_Alignment_QC\_Report_<i>aligner</i>/<i>project_name</i>_QC_RnaSeqReport.html

### Gene-based differential expression analysis - htseq-count/DESeq2

Run **pipeline\_scripts/rnaseq\_de\_report.py** to perform DE analysis and create an HTML report of differential expression summary statistics.  Read in **template\_files/rnaseq\_de\_report\_Rmd\_template.txt** from specified directory <i>template_dir</i> to create a RMD script.

> rnaseq_de_report.py --project_name <i>output_prefix</i> --sample_in <i>sample_info_file_withQC.txt</i> --comp <i>sample_comp_file.txt</i> --de_package deseq2 --ref_genome hg38 --path_start <i>output_path</i> --template_dir <i>templete_file_directory</i>

> bsub < <i>project_name</i>_deseq2.lsf

The "--sample_in" option specifies user provided phenotype file for DE analysis. The columns are the same as **example\_files/sample\_info\_file.txt** but with an additional column "QC_Pass" designating samples to be included (QC_Pass=1) or excluded (QC_Pass=0) after QC. This column naming is rigid which will be recoganized in pipeline scripts, but column order can be changed.

The "--comp" option specifies comparisons of interest in a tab-delimited text file with one comparison per line with three columns (i.e. Condition1, Condition0, Design), designating Condition1 vs. Condition2. The DE analysis accommodates a "paired" or "unpaired" option specified in Design column. For paired design, specify the condition to correct for that should match the column name in the sample info file - e.g. paired:Donor. Note that if there are any samples without a pair in any given comparison, the script will automatically drop these samples from that comparison, which will be noted in the report.

Find the example comp file here **example\_files/SRP033351\_comp\_file.txt**.

**Output files:**

The pairwise DE results and normalized counts for all samples and samples from pairwise comparisons are contained in:

> <i>project_name</i>/<i>project_name</i>_deseq2_out/
> <i>project_name</i>_<i>Condition1</i>\_vs\_<i>Condition2</i>\_full_DESeq2_results.txt
> <i>project_name</i>_<i>Condition1</i>\_vs\_<i>Condition2</i>\_counts_normalized_by_DESeq2.txt
> <i>project_name</i>_counts_normalized_by_DESeq2.txt

The RMD and corresponding HTML report file:

> <i>path_start</i>/<i>project_name</i>_deseq2_out/<i>project_name</i>_DESeq2_Report.Rmd
> <i>path_start</i>/<i>project_name</i>_deseq2_out/<i>project_name</i>_DESeq2_Report.html

### Transcript-based differential expression analysis - kallisto/sleuth

Updating...

### acknowledgements
This set of scripts was initially developed to analyze RNA-Seq and DGE data at the [Partners Personalized Medicine PPM](http://pcpgm.partners.org/). Barbara Klanderman is the molecular biologist who led the establishment of PPM RNA-seq lab protocols and played an essential role in determining what components of the reports would be most helpful to PPM wet lab staff. 

