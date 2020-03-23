---
title: Assembling & Annotating Metagenomes
linktitle: Assembly
summary:
date: 2019-12-07T16:44:26-05:00
lastmod: 2019-12-07T16:44:26-05:00
draft: false
toc: true
type: docs
weight: 40
bibliography: [files/cite.bib]
link-citations: true
highlight: true
# Add menu entry to sidebar.
# - Substitute `example` with the name of your course/documentation folder.
# - name: Declare this menu item as a parent with ID `name`.
# - parent: Reference a parent ID if this page is a child.
# - weight: Position of link in menu.
menu:
  trans-water:
    parent: processing
    name: Assembly & Annotations
    weight: 10
---

<br/>

{{% alert synopsis %}}
This page describes the steps and workflows we used to process the data. The main objectives of this section are:

1) **TRIMMING** adaptor sequences using [Trimmomatic](https://github.com/timflutre/trimmomatic).
2) **QUALITY-FILTERING** of raw reads using [IU filter quality Minoche](https://github.com/merenlab/illumina-utils).
3) **CO-ASSEMBLING**  metagenomic samples using [MEGAHIT](https://github.com/voutcn/megahit).
4) **GENERATING** a contigs database & gene calling using [PRODIGAL](https://github.com/hyattpd/Prodigal).
5) **TAXONOMIC ANNOTATION** of short reads using [KrakenUniq](https://github.com/fbreitwieser/krakenuniq).
6) **RECRUITMENT** of  reads to assembly, a.k.a. mapping using [BOWTIE2](https://github.com/BenLangmead/bowtie2).
7) **PROFILING** the mapping results.
8) **TAXONOMIC ANNOTATION** of genes using [KAIJU](https://github.com/bioinformatics-centre/kaiju) & [CENTRIFUGE](https://github.com/DaehwanKimLab/centrifuge).
9) **FUNCTIONAL ANNOTATION** of genes using [Pfams](https://pfam.xfam.org/), [COGS](https://www.ncbi.nlm.nih.gov/COG/), [KEGG](http://www.kegg.jp/ghostkoala/).
10) **MERGING** the profiles from each sample.

{{% /alert %}}

## About

In this section of the workflow we begin with raw, paired-end Illumina data. We will use a [Snakemake](https://doi.org/10.1093/bioinformatics/bts480) workflow in the anvi'o environment for most of the steps as well as some additional tools for a few stpes. The overall structure of our workflow was modelled after the one described by Delmont & Eren on [Recovering Microbial Genomes from TARA Oceans Metagenomes](http://merenlab.org/data/tara-oceans-mags/).

Specifically we will:

  1) Perform adapter trimming using TRIMMOMATIC.
  2) Use the anvi'o Snakemake workflow to: a) quality filter trimmed, raw reads;  b) co-assemble metagenomic sets; c) recruit reads; d) profile the mapping results; e) merge profile dbs; f) classify reads and genes; g) annotate genes; h) run HMM profiles.
  3) Use some external tools for annotation and clssification.

## Trimming

Before we run the workflow we must trim Illumina adaptors. First we need to make some directories to put the output files.

```bash
mkdir 00_TRIMMED
mkdir 00_TRIMMED_UNPAIRED
```

Next we run [TRIMMOMATIC](https://doi.org/10.1093/bioinformatics/btu170) assuming the raw data is in a directory called `RAW`. Here is an example for a single sample (`ML1699`). Since we ran these samples on a NextSeq, there are 4 files per paired-end. So we have 8 files total. We need to run the command 4 times, one for each pair. Here is the command for the first pair

```bash
runtrimmomatic PE -threads $NSLOTS RAW/ML1699_S6_L001_R1_001.fastq.gz /
                                   RAW/ML1699_S6_L001_R2_001.fastq.gz /
                                   00_TRIMMED/ML1699_S6_L001_R1_001_trim.fastq.gz /
                                   00_TRIMMED_UNPAIRED/ML1699_S6_L001_R1_001_trim_unpaired.fastq.gz /
                                   00_TRIMMED/ML1699_S6_L001_R2_001_trim.fastq.gz /
                                   00_TRIMMED_UNPAIRED/ML1699_S6_L001_R2_001_trim_unpaired.fastq.gz /
          ILLUMINACLIP:/share/apps/bioinformatics/trimmomatic/0.33/adapters/NexteraPE-PE.fa:2:30:10 /
          MINLEN:40
```

Now we run the command for the other 3 pairs,

* ML1699_S6_L002_R1_001 & ML1699_S6_L002_R2_001
* ML1699_S6_L003_R1_001 & ML1699_S6_L003_R2_001
* ML1699_S6_L004_R1_001 & ML1699_S6_L004_R2_001

And then do the same for the rest of the samples.

<details markdown="1"><summary>Show/hide HYDRA TRIMMOMATIC job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 16
#$ -q sThC.q
#$ -cwd
#$ -j y
#$ -N job_00_trimmomatic
#$ -o hydra_logs/job_00_trimmomatic_redo.log
#
# ----------------Modules------------------------- #
module load bioinformatics/trimmomatic
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------COMMANDS------------------- #
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>

## Snakemake Workflow

Thanks to the anvi'o implementation of Snakemake we can run many commands suquentially and/or simutaneously. There is plenty of good documentation on the anvi'o website about setting up snakemake workflows [here](http://merenlab.org/2018/07/09/anvio-snakemake-workflows/) so we will refrain from any lengthy explainations.

You can see all the settings we used and download our configs file [here](../files/default-config.txt) or grab it from  the folded code below. Some commands we chose not to run but they remain in the workflow for posterity.

We decided to do 2 separate assemblies, one for the Western Atlantic samples (n = 29) and one for the Eastern Pacific samples (n = 28). To do this we needed a separate file called `samples.txt` which can be downloaded [here](../files/samples.txt). This file tells anvi'o where to find the **trimmed** fastq files for each sample and what group the sample belongs to. The file is a four-column, tab-delimited file. We can use  a two  samples from each assembly to illustrate the format.


| sample | group |               r1                        |                r2                       |
|--------|-------|-----------------------------------------|-----------------------------------------|
|EPR_9A  |    EP | comma separated list of forward reads   | comma separated list of reverse reads   |
|EPR_11A |    EP | comma separated list of forward reads   | comma separated list of reverse reads   |
|WAR_MAR |    WA | comma separated list of forward reads   | comma separated list of reverse reads   |
|WAM_MYS |    WA | comma separated list of forward reads   | comma separated list of reverse reads   |

Remember, there are 4 fastq files *per* direction (forward or reverse) *per* sample. So each comma separated list in the `r1` column needs four file names and the same with the `r2` column. And you must include *relative path names*.

Now lets visualize the anvi'o Snakemake workflow.






<!--html_preserve-->{{% figure src="/img/dag.png" title=" Colors indicate broad divisions of workflow: sky blue, short-read prep & co-assembly; blueish green, short-read mapping to assembly; orange, taxonomic or functional classification; yellow, automatic binning; reddish purple, databases construction. " lightbox="true" alt="Hum, looks like there is something wrong here." %}}<!--/html_preserve-->

Directed acyclic graph (DAG) of the metagenomic workflow where edge connections represent dependencies and nodes represent commands. The workflow begins with raw data (trimmed of Illumina adapters) and continues up to and including automatic binning of contigs. At the end of this workflow we then proceed with manual binning and MAG generation. For simplicity, only two samples from the Eastern Pacific & two samples from the Western Atlantic are shown. We also added nodes for Virsorter and Kaiju annotations since these are not part of the workflow. You can  download an image of the workflow [here](../../../img/dag.png).

<details markdown="1"><summary>Show/hide JSON-formatted configuration file.</summary>
<pre><code>
{
    "fasta_txt": "",
    "anvi_gen_contigs_database": {
        "--project-name": "{group}",
        "--description": "",
        "--skip-gene-calling": "",
        "--external-gene-calls": "",
        "--ignore-internal-stop-codons": "",
        "--skip-mindful-splitting": "",
        "--contigs-fasta": "",
        "--split-length": "",
        "--kmer-size": "",
        "--prodigal-translation-table": "",
        "threads": ""
    },
    "centrifuge": {
        "threads": 2,
        "run": true,
        "db": "/pool/genomics/stri_istmobiome/dbs/centrifuge_dbs/p+h+v"
    },
    "anvi_run_hmms": {
        "run": true,
        "threads": 5,
        "--installed-hmm-profile": "",
        "--hmm-profile-dir": ""
    },
    "anvi_run_ncbi_cogs": {
        "run": true,
        "threads": 3,
        "--cog-data-dir": "/pool/genomics/stri_istmobiome/dbs/cog_db/",
        "--sensitive": "",
        "--temporary-dir-path": "/pool/genomics/stri_istmobiome/dbs/cog_db/tmp/",
        "--search-with": ""
    },
    "anvi_run_scg_taxonomy": {
        "run": true,
        "threads": 6,
        "--scgs-taxonomy-data-dir": "/pool/genomics/stri_istmobiome/dbs/scgs-taxonomy-data/"
    },
    "anvi_script_reformat_fasta": {
        "run": true,
        "--keep-ids": "",
        "--exclude-ids": "",
        "--min-len": "1000",
        "threads": ""
    },
    "emapper": {
        "--database": "bact",
        "--usemem": true,
        "--override": true,
        "path_to_emapper_dir": "",
        "threads": ""
    },
    "anvi_script_run_eggnog_mapper": {
        "--use-version": "0.12.6",
        "run": "",
        "--cog-data-dir": "",
        "--drop-previous-annotations": "",
        "threads": ""
    },
    "samples_txt": "samples.txt",
    "metaspades": {
        "additional_params": "--only-assembler",
        "threads": 7,
        "run": "",
        "use_scaffolds": ""
    },
    "megahit": {
        "--min-contig-len": 1000,
        "--memory": 0.8,
        "threads": 15,
        "run": true,
        "--min-count": "",
        "--k-min": "",
        "--k-max": "",
        "--k-step": "",
        "--k-list": "",
        "--no-mercy": "",
        "--no-bubble": "",
        "--merge-level": "",
        "--prune-level": "",
        "--prune-depth": "",
        "--low-local-ratio": "",
        "--max-tip-len": "",
        "--no-local": "",
        "--kmin-1pass": "",
        "--presets": "meta-sensitive",
        "--mem-flag": "",
        "--use-gpu": "",
        "--gpu-mem": "",
        "--keep-tmp-files": "",
        "--tmp-dir": "",
        "--continue": true,
        "--verbose": ""
    },
    "idba_ud": {
        "--min_contig": 1000,
        "threads": 7,
        "run": "",
        "--mink": "",
        "--maxk": "",
        "--step": "",
        "--inner_mink": "",
        "--inner_step": "",
        "--prefix": "",
        "--min_count": "",
        "--min_support": "",
        "--seed_kmer": "",
        "--similar": "",
        "--max_mismatch": "",
        "--min_pairs": "",
        "--no_bubble": "",
        "--no_local": "",
        "--no_coverage": "",
        "--no_correct": "",
        "--pre_correction": ""
    },
    "iu_filter_quality_minoche": {
        "run": true,
        "--ignore-deflines": true,
        "--visualize-quality-curves": "",
        "--limit-num-pairs": "",
        "--print-qual-scores": "",
        "--store-read-fate": "",
        "threads": ""
    },
    "gzip_fastqs": {
        "run": true,
        "threads": ""
    },
    "bowtie": {
        "additional_params": "--no-unal",
        "threads": 3
    },
    "samtools_view": {
        "additional_params": "-F 4",
        "threads": ""
    },
    "anvi_profile": {
        "threads": 10,
        "--sample-name": "{sample}",
        "--overwrite-output-destinations": true,
        "--report-variability-full": "",
        "--skip-SNV-profiling": "",
        "--profile-SCVs": true,
        "--description": "",
        "--skip-hierarchical-clustering": "",
        "--distance": "",
        "--linkage": "",
        "--min-contig-length": "",
        "--min-mean-coverage": "",
        "--min-coverage-for-variability": "",
        "--cluster-contigs": "",
        "--contigs-of-interest": "",
        "--queue-size": "",
        "--write-buffer-size": 10000,
        "--max-contig-length": "",
        "--max-coverage-depth": "",
        "--ignore-orphans": ""
    },
    "anvi_merge": {
        "--sample-name": "{group}",
        "--overwrite-output-destinations": true,
        "--description": "",
        "--skip-hierarchical-clustering": "",
        "--enforce-hierarchical-clustering": "",
        "--distance": "",
        "--linkage": "",
        "threads": ""
    },
    "import_percent_of_reads_mapped": {
        "run": true,
        "threads": ""
    },
    "krakenuniq": {
        "threads": 3,
        "--gzip-compressed": true,
        "additional_params": "",
        "run": true,
        "--db": "/scratch/genomics/scottjj/kraken_dbs/DB/"
    },
    "remove_short_reads_based_on_references": {
        "delimiter-for-iu-remove-ids-from-fastq": " ",
        "dont_remove_just_map": "",
        "references_for_removal_txt": "",
        "threads": ""
    },
    "anvi_cluster_contigs": {
        "--collection-name": "{driver}",
        "run": true,
        "--driver": "concoct",
        "--just-do-it": "",
        "--additional-params-concoct": "",
        "--additional-params-metabat2": "",
        "--additional-params-maxbin2": "",
        "--additional-params-dastool": "",
        "--additional-params-binsanity": "",
        "threads": 10
    },
    "gen_external_genome_file": {
        "threads": ""
    },
    "export_gene_calls_for_centrifuge": {
        "threads": ""
    },
    "anvi_import_taxonomy_for_genes": {
        "threads": ""
    },
    "annotate_contigs_database": {
        "threads": ""
    },
    "anvi_get_sequences_for_gene_calls": {
        "threads": ""
    },
    "gunzip_fasta": {
        "threads": ""
    },
    "reformat_external_gene_calls_table": {
        "threads": ""
    },
    "reformat_external_functions": {
        "threads": ""
    },
    "import_external_functions": {
        "threads": ""
    },
    "anvi_run_pfams": {
        "run": true,
        "--pfam-data-dir": "/pool/genomics/stri_istmobiome/dbs/pfam_db",
        "threads": 10
    },
    "iu_gen_configs": {
        "--r1-prefix": "",
        "--r2-prefix": "",
        "threads": ""
    },
    "gen_qc_report": {
        "threads": ""
    },
    "merge_fastqs_for_co_assembly": {
        "threads": ""
    },
    "merge_fastas_for_co_assembly": {
        "threads": ""
    },
    "bowtie_build": {
        "threads": ""
    },
    "anvi_init_bam": {
        "threads": ""
    },
    "krakenuniq_mpa_report": {
        "threads": ""
    },
    "import_krakenuniq_taxonomy": {
        "--min-abundance": "",
        "threads": ""
    },
    "anvi_summarize": {
        "additional_params": "",
        "run": "",
        "threads": ""
    },
    "anvi_split": {
        "additional_params": "",
        "run": "",
        "threads": ""
    },
    "references_mode": "",
    "all_against_all": "",
    "kraken_txt": "",
    "collections_txt": "",
    "output_dirs": {
        "FASTA_DIR": "02_FASTA",
        "CONTIGS_DIR": "03_CONTIGS",
        "QC_DIR": "01_QC",
        "MAPPING_DIR": "04_MAPPING",
        "PROFILE_DIR": "05_ANVIO_PROFILE",
        "MERGE_DIR": "06_MERGED",
        "TAXONOMY_DIR": "07_TAXONOMY",
        "SUMMARY_DIR": "08_SUMMARY",
        "SPLIT_PROFILES_DIR": "09_SPLIT_PROFILES",
        "LOGS_DIR": "00_LOGS"
    },
    "max_threads": ""
}
</code></pre>
</details>

And here are the commands we used to run the workflow.

```bash
anvi-run-workflow -w metagenomics -c default-config.txt --additional-params --jobs 63 --resources nodes=63 --keep-going --rerun-incomplete --unlock
anvi-run-workflow -w metagenomics -c default-config.txt --additional-params --jobs 63 --resources nodes=63 --keep-going --rerun-incomplete
```

If a Snakemake job fails the defualt behavior is to lock the workflow and because large jobs can fail for a lot of reasons, we decided to always include an identical command first with the `--unlock` flag. This will  unlock the workflow and then the second command will execute. You can find an explaination of this [here](http://merenlab.org/2018/07/09/anvio-snakemake-workflows/#how-can-i-restart-a-failed-job).

<details markdown="1"><summary>Show/hide HYDRA SNAKEMAKE job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 63
#$ -q mThM.q
#$ -l mres=693G,h_data=11G,h_vmem=11G,himem
#$ -cwd
#$ -j y
#$ -N job_01_run_mg_workflow
#$ -o hydra_logs/job_01_run_mg_workflow_megahit_rerun_2500.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------CALLING ANVIO------------------- #
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
# ----------------COOLIO?------------------- #
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------TEMP DIRECTORIES------------------- #
rm -r /pool/genomics/stri_istmobiome/dbs/cog_db/tmp/
mkdir -p /pool/genomics/stri_istmobiome/dbs/cog_db/tmp/
#
rm -r /pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/
mkdir -p /pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/
TMPDIR="/pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/"
#
# ----------------COMMANDS------------------- #
#
anvi-run-workflow -w metagenomics -c default-config.txt --additional-params --jobs 63 --resources nodes=63 --keep-going --rerun-incomplete --unlock
anvi-run-workflow -w metagenomics -c default-config.txt --additional-params --jobs 63 --resources nodes=63 --keep-going --rerun-incomplete
echo = `date` job $JOB_NAME don
</code></pre>
</details>

There are many tools used in the workflow that need to be cited. Also, there are several tools that we  run outside of the snakemake workflow. Results from some of these need to be added to the individual `PROFILE.db`'s or the merged `PROFILE.db`. Therefore, before the `anvi-merge` finished we killed the job, ran the analyses described below, and finally restarted the workflow to finish the missing step. Cumbersome, yes, but it got the job done.

## VirSorter annotation

We ran Virsorter on conigs from both assemblies (WA & EP) using our newly created `contig.db`'s generated after the co-assembly step. We ran Virsorter with and without the `--virome` flag. See [issue 40](https://github.com/simroux/VirSorter/issues/40) on the Virsorter GitHub site for a discussion of why we did this. Here are the VirSorter commands.

```bash
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/WA/WA-contigs.fa --ncpu $NSLOTS --db 2 --wdir 07_TAXONOMY/VIRSORTER/WA --data-dir $PATH/virsorter-data --diamond
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/WA/WA-contigs.fa --ncpu $NSLOTS --db 2 --wdir 07_TAXONOMY/VIRSORTER_VIROME/WA --data-dir $PATH/virsorter-data --diamond --virome
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/EP/EP-contigs.fa --ncpu $NSLOTS --db 2 --wdir 07_TAXONOMY/VIRSORTER/EP --data-dir $PATH/virsorter-data --diamond
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/EP/EP-contigs.fa --ncpu $NSLOTS --db 2 --wdir 07_TAXONOMY/VIRSORTER_VIROME/EP --data-dir $PATH/virsorter-data --diamond --virome
```

<details markdown="1"><summary>Show/hide HYDRA VIRSORTER job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 25
#$ -q mThC.q
#$ -l mres=5G,h_data=5G,h_vmem=5G
#$ -cwd
#$ -j y
#$ -N job_03_run_virsorter
#$ -o hydra_logs/job_03_run_virsorter.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
# ----------------Calling Virsorter------------------- #
#
source activate virsorter
which perl
#
# ----------------Make directories------------------- #
mkdir 07_TAXONOMY/
mkdir 07_TAXONOMY/VIRSORTER/
#
# ----------------COMMANDS------------------- #
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/WA/WA-contigs.fa --ncpu 25 --db 2 --wdir 07_TAXONOMY/VIRSORTER/WA --data-dir /pool/genomics/stri_istmobiome/dbs/virsorter/virsorter-data --diamond
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/WA/WA-contigs.fa --ncpu 25 --db 2 --wdir 07_TAXONOMY/VIRSORTER_VIROME/WA --data-dir /pool/genomics/stri_istmobiome/dbs/virsorter/virsorter-data --diamond --virome
#
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/EP/EP-contigs.fa --ncpu 25 --db 2 --wdir 07_TAXONOMY/VIRSORTER/EP --data-dir /pool/genomics/stri_istmobiome/dbs/virsorter/virsorter-data --diamond
wrapper_phage_contigs_sorter_iPlant.pl -f 02_FASTA/EP/EP-contigs.fa --ncpu 25 --db 2 --wdir 07_TAXONOMY/VIRSORTER_VIROME/EP --data-dir /pool/genomics/stri_istmobiome/dbs/virsorter/virsorter-data --diamond --virome
#
source deactivate
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>


## Kaiju annotation

In addition to the Centrifuge classification we are going to use [Kaiju](https://doi.org/10.1038/ncomms11257) to classify gene calls. We will do this against the [two databases](https://github.com/bioinformatics-centre/kaiju#creating-the-reference-database-and-index) we built [earlier](../2-setup-databases/#kaiju).

First we need to activate anvi'o, make a directory for the output, and grab the gene calls. Note that `$KAIJU` is the path to the input/output directory, which in this case is the new directory, `07_TAXONOMY/KAIJU/`.

```bash
mkdir 07_TAXONOMY/KAIJU/
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/EP-contigs.db -o $KAIJU/EP_gene_calls.fna
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/WA-contigs.db -o $KAIJU/WA_gene_calls.fna
```

Next we run the Kaiju commands against the `nr_euk` database and then the `mar` database. Note that `$K_FILES` is the path to the database and `$KAIJU` is the path to the input/output directory.

```bash
# against nr_euk db
kaiju -t $K_FILES/nr_db/nodes.dmp -f $K_FILES/nr_db/nr_euk/kaiju_db_nr_euk.fmi -i $KAIJU/EP_gene_calls.fna -o $KAIJU/EP_kaiju_nr.out -z 16 -v
kaiju -t $K_FILES/nr_db/nodes.dmp -f $K_FILES/nr_db/nr_euk/kaiju_db_nr_euk.fmi -i $KAIJU/WA_gene_calls.fna -o $KAIJU/WA_kaiju_nr.out -z 16 -v
# against mar db
kaiju -t $K_FILES/marine_db/nodes.dmp -f $K_FILES/marine_db/marine_db/kaiju_db_mar.fmi -i $KAIJU/EP_gene_calls.fna -o $KAIJU/EP_kaiju_mar.out -z 16 -v
kaiju -t $K_FILES/marine_db/nodes.dmp -f $K_FILES/marine_db/marine_db/kaiju_db_mar.fmi -i $KAIJU/WA_gene_calls.fna -o $KAIJU/WA_kaiju_mar.out -z 16 -v
```

Finally, we need to add taxon names to the output in order to import the taxonomy into the `contig.db` later on. We will also compare the output of the two classifications.

```bash
# against nr_euk db
kaiju-addTaxonNames -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/EP_kaiju_nr.out -o $KAIJU/EP_kaiju_nr.names -r superkingdom,phylum,order,class,family,genus,species
kaiju-addTaxonNames -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/WA_kaiju_nr.out -o $KAIJU/WA_kaiju_nr.names -r superkingdom,phylum,order,class,family,genus,species
# against mar db
kaiju-addTaxonNames -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/EP_kaiju_mar.out -o $KAIJU/EP_kaiju_mar.names -r superkingdom,phylum,order,class,family,genus,species
kaiju-addTaxonNames -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/WA_kaiju_mar.out -o $KAIJU/WA_kaiju_mar.names -r superkingdom,phylum,order,class,family,genus,species
```

<details markdown="1"><summary>Show/hide HYDRA KAIJU job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 20
#$ -q sThC.q
#$ -l mres=120G,h_data=6G,h_vmem=6G
#$ -cwd
#$ -j y
#$ -N job_04_taxonomic_classification_kaiju
#$ -o hydra_logs/job_04_taxonomic_classification_kaiju.job
#
# ----------------Modules------------------------- #
#
# ----------------Load Envs------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
export PATH=/home/scottjj/miniconda3/envs/kaiju/bin:$PATH
export PATH=/home/scottjj/miniconda3/envs/krona/bin:$PATH
export PATH=/pool/genomics/stri_istmobiome/dbs/kaiju_db/:$PATH
#
#NOT SURE IF THE FOLLOWING TWO LINES ARE NEEDED
export PERL5LIB="/home/scottjj/miniconda3/envs/kaiju/lib/5.26.2"
export PERL5LIB="/home/scottjj/miniconda3/envs/kaiju/lib/5.26.2/x86_64-linux-thread-multi:$PERL5LIB"
# ----------------SETUP KAIJU Directories-------------- #
#
mkdir 07_TAXONOMY/KAIJU/
KAIJU='/pool/genomics/stri_istmobiome/data/seq/TRANS_WATER/07_TAXONOMY/KAIJU/'
K_FILES='/pool/genomics/stri_istmobiome/dbs/kaiju_db'
#
# ----------------Get Gene Files------------------- #
source activate anvio-6.1
#
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/EP-contigs.db -o $KAIJU/EP_gene_calls.fna
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/WA-contigs.db -o $KAIJU/WA_gene_calls.fna
#
source deactivate
#
# ----------------RUN KAIJU------------------- #
source activate kaiju
which kaiju
gcc --version
which perl
# ----------------AGAINST nr DB------------------- #
#
kaiju -t $K_FILES/nr_db/nodes.dmp -f $K_FILES/nr_db/nr_euk/kaiju_db_nr_euk.fmi -i $KAIJU/EP_gene_calls.fna -o $KAIJU/EP_kaiju_nr.out -z 16 -v
kaiju -t $K_FILES/nr_db/nodes.dmp -f $K_FILES/nr_db/nr_euk/kaiju_db_nr_euk.fmi -i $KAIJU/WA_gene_calls.fna -o $KAIJU/WA_kaiju_nr.out -z 16 -v
#
kaiju-addTaxonNames -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/EP_kaiju_nr.out -o $KAIJU/EP_kaiju_nr.names -r superkingdom,phylum,order,class,family,genus,species
kaiju-addTaxonNames -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/WA_kaiju_nr.out -o $KAIJU/WA_kaiju_nr.names -r superkingdom,phylum,order,class,family,genus,species
#
# ----------------AGAINST marine DB------------------- #
#
kaiju -t $K_FILES/marine_db/nodes.dmp -f $K_FILES/marine_db/marine_db/kaiju_db_mar.fmi -i $KAIJU/EP_gene_calls.fna -o $KAIJU/EP_kaiju_mar.out -z 16 -v
kaiju -t $K_FILES/marine_db/nodes.dmp -f $K_FILES/marine_db/marine_db/kaiju_db_mar.fmi -i $KAIJU/WA_gene_calls.fna -o $KAIJU/WA_kaiju_mar.out -z 16 -v
#
kaiju-addTaxonNames -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/EP_kaiju_mar.out -o $KAIJU/EP_kaiju_mar.names -r superkingdom,phylum,order,class,family,genus,species
kaiju-addTaxonNames -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/WA_kaiju_mar.out -o $KAIJU/WA_kaiju_mar.names -r superkingdom,phylum,order,class,family,genus,species
#
source deactivate
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>

## Import Kaiju

Now its is time to add the Kaiju annotation to the `contig.db`. We will also use the opportunity to generate some [Krona plots](https://github.com/marbl/Krona/wiki).

First we need to take the Kaiju-formmated output file and format it from Krona. We can do this for the results from both annotations.

```bash
# from nr_euk db
kaiju2krona -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/WA_kaiju_nr.out -o $KAIJU/WA_kaiju_nr.out.krona
kaiju2krona -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/EP_kaiju_nr.out -o $KAIJU/EP_kaiju_nr.out.krona
# from mar db
kaiju2krona -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/WA_kaiju_mar.out -o $KAIJU/WA_kaiju_mar.out.krona
kaiju2krona -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/EP_kaiju_mar.out -o $KAIJU/EP_kaiju_mar.out.krona
```

While we are at it, lets also generate some summary files of taxonomic content, again from both annotations.

```bash
# from nr_euk db
kaiju2table -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -r class $KAIJU/WA_kaiju_nr.out -l phylum,class,order,family -o $KAIJU/WA_kaiju_nr.out.summary
kaiju2table -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -r class $KAIJU/EP_kaiju_nr.out -l phylum,class,order,family -o $KAIJU/EP_kaiju_nr.out.summary
# from mar db
kaiju2table -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -r class $KAIJU/WA_kaiju_mar.out -l phylum,class,order,family -o $KAIJU/WA_kaiju_mar.out.summary
kaiju2table -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -r class $KAIJU/EP_kaiju_mar.out -l phylum,class,order,family -o $KAIJU/EP_kaiju_mar.out.summary
```

Now we will the output files from above to generate Krona plots for each assembly, each annotation, and send the output to  HTML files. For this we use the krona conda package.

```bash
# from nr_euk db
ktImportText -o $KAIJU/WA_kaiju_nr.out.html $KAIJU/WA_kaiju_nr.out.krona
ktImportText -o $KAIJU/EP_kaiju_nr.out.html $KAIJU/EP_kaiju_nr.out.krona
#from mar db
ktImportText -o $KAIJU/WA_kaiju_mar.out.html $KAIJU/WA_kaiju_mar.out.krona
ktImportText -o $KAIJU/EP_kaiju_mar.out.html $KAIJU/EP_kaiju_mar.out.krona
```
Finally, it is time to import the Kaiju taxonomies into the `contig.db` for each assembly. As we will see later, the `mar db` annotations called very few viral sequences, even though we know from Virsorter there are a lot of viruses. Therefore we decided to use the `nr db` annotations.

```bash
anvi-import-taxonomy-for-genes -c 03_CONTIGS/EP-contigs.db -p kaiju -i $KAIJU/EP_kaiju_nr.names --just-do-it
anvi-import-taxonomy-for-genes -c 03_CONTIGS/WA-contigs.db -p kaiju -i $KAIJU/WA_kaiju_nr.names --just-do-it
```

<details markdown="1"><summary>Show/hide HYDRA KAIJU import and summarize job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 5
#$ -q sThC.q
#$ -l mres=25G,h_data=5G,h_vmem=5G
#$ -cwd
#$ -j y
#$ -N job_05_kaiju_summary
#$ -o hydra_logs/job_05_kaiju_summary.job
#
# ----------------Modules------------------------- #
#
# ----------------Load Envs------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
export PATH=/home/scottjj/miniconda3/envs/kaiju/bin:$PATH
export PATH=/home/scottjj/miniconda3/envs/krona/bin:$PATH
export PATH=/pool/genomics/stri_istmobiome/dbs/kaiju_db/:$PATH
#
#NOT SURE IF THE FOLLOWING TWO LINES ARE NEEDED
export PERL5LIB="/home/scottjj/miniconda3/envs/kaiju/lib/5.26.2"
export PERL5LIB="/home/scottjj/miniconda3/envs/kaiju/lib/5.26.2/x86_64-linux-thread-multi:$PERL5LIB"
# ----------------SETUP KAIJU Directories-------------- #
#
KAIJU='/pool/genomics/stri_istmobiome/data/seq/TRANS_WATER/07_TAXONOMY/KAIJU/'
K_FILES='/pool/genomics/stri_istmobiome/dbs/kaiju_db/'
#
# ----------------Activate------------------- #
source activate kaiju
which kaiju
gcc --version
which perl
# ----------------get files for KRONA plots------------------- #
#
# ----------------nr_euk db------------------- #
kaiju2krona -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/WA_kaiju_nr.out -o $KAIJU/WA_kaiju_nr.out.krona
kaiju2krona -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -i $KAIJU/EP_kaiju_nr.out -o $KAIJU/EP_kaiju_nr.out.krona
#
kaiju2table -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -r class $KAIJU/WA_kaiju_nr.out -l phylum,class,order,family -o $KAIJU/WA_kaiju_nr.out.summary
kaiju2table -t $K_FILES/nr_db/nodes.dmp -n $K_FILES/nr_db/names.dmp -r class $KAIJU/EP_kaiju_nr.out -l phylum,class,order,family -o $KAIJU/EP_kaiju_nr.out.summary
#
# ----------------mar db------------------- #
kaiju2krona -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/WA_kaiju_mar.out -o $KAIJU/WA_kaiju_mar.out.krona
kaiju2krona -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -i $KAIJU/EP_kaiju_mar.out -o $KAIJU/EP_kaiju_mar.out.krona
#
kaiju2table -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -r class $KAIJU/WA_kaiju_mar.out -l phylum,class,order,family -o $KAIJU/WA_kaiju_mar.out.summary
kaiju2table -t $K_FILES/marine_db/nodes.dmp -n $K_FILES/marine_db/names.dmp -r class $KAIJU/EP_kaiju_mar.out -l phylum,class,order,family -o $KAIJU/EP_kaiju_mar.out.summary
conda deactivate
# ----------------Build KRONA plots------------------- #
source activate krona
ktImportText -o $KAIJU/WA_kaiju_nr.out.html $KAIJU/WA_kaiju_nr.out.krona
ktImportText -o $KAIJU/EP_kaiju_nr.out.html $KAIJU/EP_kaiju_nr.out.krona
#
ktImportText -o $KAIJU/WA_kaiju_mar.out.html $KAIJU/WA_kaiju_mar.out.krona
ktImportText -o $KAIJU/EP_kaiju_mar.out.html $KAIJU/EP_kaiju_mar.out.krona
conda deactivate
#--------------------ANVIO PARSER--------------#
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
anvi-import-taxonomy-for-genes -c 03_CONTIGS/EP-contigs.db -p kaiju -i $KAIJU/EP_kaiju_nr.names --just-do-it
anvi-import-taxonomy-for-genes -c 03_CONTIGS/WA-contigs.db -p kaiju -i $KAIJU/WA_kaiju_nr.names --just-do-it
#
source deactivate
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>

## Kaken annotation

In this section we use `krakenuniq` to classify the short reads. Because of the memory demands of `krakenuniq`, we could not get this to work in the Snakemake workflow so we ran the analysis separately. For several of these commands we used lists of sample names and  `for` loops to run through each sample.

Ok, here we run `krakenuniq` and several reporting and summary steps against the short read data for each sample using the [list.txt](../files/list.txt) file.

```bash
for sample in `cat list.txt`
do
    krakenuniq --report-file $KRAKEN/$sample-REPORT.tsv 01_QC/$sample-QUALITY_PASSED_R1.fastq.gz 01_QC/$sample-QUALITY_PASSED_R2.fastq.gz --db $K_FILES/DB/ --threads 2 --preload  --fastq-input --gzip-compressed --paired --output $KRAKEN/$sample-kraken.out

    krakenuniq-report --db $K_FILES/DB/ $KRAKEN/$sample-kraken.out > $KRAKEN/$sample-kraken_report.txt
    krakenuniq-mpa-report --header-line --db $K_FILES/DB/ $KRAKEN/$sample-kraken.out > $KRAKEN/$sample-kraken_mpa_report.txt
    krakenuniq-translate --db $K_FILES/DB/ $KRAKEN/$sample-kraken.out > $KRAKEN/$sample-kraken.trans
done
```

Next, we format the output to make Krona plots.

```bash
for sample in `cat list.txt`
do
    $KRA_to_KRON/kraken_to_krona.py $KRAKEN/$sample-kraken.trans > $KRAKEN/$sample-kraken.krona
done
```

And finally construct Krona plots.

```bash
for sample in `cat list.txt`
do
    ktImportText -o $KRAKEN/$sample-kraken.html $KRAKEN/$sample-kraken.krona
done
```
We can also make standalone HTML pages from each assembly containing all Krona plots.

```bash
ktImportText -o $KRAKEN/EP-kraken.html $KRAKEN/EPM_12A1-kraken.krona $KRAKEN/EPM_12A2-kraken.krona $KRAKEN/EPM_12A3-kraken.krona $KRAKEN/EPM_12A4-kraken.krona $KRAKEN/EPM_13A1-kraken.krona $KRAKEN/EPM_13A2-kraken.krona $KRAKEN/EPM_13A3-kraken.krona $KRAKEN/EPM_13A4-kraken.krona $KRAKEN/EPM_14A1-kraken.krona $KRAKEN/EPM_14A2-kraken.krona $KRAKEN/EPM_14A3-kraken.krona $KRAKEN/EPM_14A4-kraken.krona $KRAKEN/EPR_10A-kraken.krona $KRAKEN/EPR_10B-kraken.krona $KRAKEN/EPR_11A-kraken.krona $KRAKEN/EPR_11B-kraken.krona $KRAKEN/EPR_11C-kraken.krona $KRAKEN/EPR_13B-kraken.krona $KRAKEN/EPR_13C-kraken.krona $KRAKEN/EPR_14B-kraken.krona $KRAKEN/EPR_14C-kraken.krona $KRAKEN/EPR_14E-kraken.krona $KRAKEN/EPR_15A-kraken.krona $KRAKEN/EPR_15B-kraken.krona $KRAKEN/EPR_8A-kraken.krona $KRAKEN/EPR_8B-kraken.krona $KRAKEN/EPR_9A-kraken.krona $KRAKEN/EPR_9B-kraken.krona
#
ktImportText -o $KRAKEN/WA-kraken.html $KRAKEN/WAM_ALR-kraken.krona $KRAKEN/WAM_CCR-kraken.krona $KRAKEN/WAM_IPI-kraken.krona $KRAKEN/WAM_MAR-kraken.krona $KRAKEN/WAM_MYS-kraken.krona $KRAKEN/WAM_PBL-kraken.krona $KRAKEN/WAM_PJN-kraken.krona $KRAKEN/WAM_PPR-kraken.krona $KRAKEN/WAM_PST-kraken.krona $KRAKEN/WAM_ROL-kraken.krona $KRAKEN/WAM_SCR-kraken.krona $KRAKEN/WAM_SGN-kraken.krona $KRAKEN/WAM_SIS-kraken.krona $KRAKEN/WAM_TWN-kraken.krona $KRAKEN/WAR_ALR-kraken.krona $KRAKEN/WAR_CCR-kraken.krona $KRAKEN/WAR_IPI-kraken.krona $KRAKEN/WAR_MAR-kraken.krona $KRAKEN/WAR_MYS-kraken.krona $KRAKEN/WAR_PBL-kraken.krona $KRAKEN/WAR_PJN-kraken.krona $KRAKEN/WAR_PPR-kraken.krona $KRAKEN/WAR_PSK-kraken.krona $KRAKEN/WAR_PST-kraken.krona $KRAKEN/WAR_RNW-kraken.krona $KRAKEN/WAR_ROL-kraken.krona $KRAKEN/WAR_SCR-kraken.krona $KRAKEN/WAR_SGL-kraken.krona $KRAKEN/WAR_SIS-kraken.krona
```

Now we have a Krona plot page for the EP samples (`EP-kraken.html`) and the WA samples (`WA-kraken.html`).

<details markdown="1"><summary>Show/hide HYDRA Kraken classification job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q mThM.q
#$ -l mres=150G,h_data=150G,h_vmem=150G,himem
#$ -cwd
#$ -j y
#$ -N job_04_taxonomic_classification_kraken_reads
#$ -o hydra_logs/job_04_taxonomic_classification_kraken_reads.job
#
# ----------------Modules------------------------- #
#
# ----------------Load Envs------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Kraken -------------- #
#
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate kraken
# ----------------SETUP Kraken Directories-------------- #
#
mkdir 07_TAXONOMY/KRAKEN/
mkdir 07_TAXONOMY/KRAKEN/READS
#
KRAKEN='/data/genomics/stri_istmobiome/TRANS_WATER_MG_DATA/07_TAXONOMY/KRAKEN/READS/'
K_FILES='/pool/genomics/stri_istmobiome/dbs/kraken_dbs/'
KRA_to_KRON='/home/scottjj/miniconda3/envs/metawrap-env/bin/metawrap-scripts/'
# ----------------Run Kraken------------------- #
#
for sample in `cat list.txt`
do
    krakenuniq --report-file `\(KRAKEN/\)`sample-REPORT.tsv 01_QC/$sample-QUALITY_PASSED_R1.fastq.gz 01_QC/$sample-QUALITY_PASSED_R2.fastq.gz --db $K_FILES/DB/ --threads 2 --preload  --fastq-input --gzip-compressed --paired --output `\(KRAKEN/\)`sample-kraken.out
    krakenuniq-report --db $K_FILES/DB/ `\(KRAKEN/\)`sample-kraken.out > `\(KRAKEN/\)`sample-kraken_report.txt
    krakenuniq-mpa-report --header-line --db $K_FILES/DB/ `\(KRAKEN/\)`sample-kraken.out > `\(KRAKEN/\)`sample-kraken_mpa_report.txt
    krakenuniq-translate --db $K_FILES/DB/ `\(KRAKEN/\)`sample-kraken.out > `\(KRAKEN/\)`sample-kraken.trans
done
#
source deactivate
#
# ----------------FORMAT FOR KRONA------------------- #
#
source activate metawrap-env
#
for sample in `cat list.txt`
do
    $KRA_to_KRON/kraken_to_krona.py `\(KRAKEN/\)`sample-kraken.trans > `\(KRAKEN/\)`sample-kraken.krona
done
source deactivate
# ----------------MAKE KRONA PLOTS------------------- #
#
source activate krona_env
#
for sample in `cat list.txt`
do
    ktImportText -o `\(KRAKEN/\)`sample-kraken.html `\(KRAKEN/\)`sample-kraken.krona
done
#
ktImportText -o $KRAKEN/EP-kraken.html $KRAKEN/EPM_12A1-kraken.krona $KRAKEN/EPM_12A2-kraken.krona $KRAKEN/EPM_12A3-kraken.krona $KRAKEN/EPM_12A4-kraken.krona $KRAKEN/EPM_13A1-kraken.krona $KRAKEN/EPM_13A2-kraken.krona $KRAKEN/EPM_13A3-kraken.krona $KRAKEN/EPM_13A4-kraken.krona $KRAKEN/EPM_14A1-kraken.krona $KRAKEN/EPM_14A2-kraken.krona $KRAKEN/EPM_14A3-kraken.krona $KRAKEN/EPM_14A4-kraken.krona $KRAKEN/EPR_10A-kraken.krona $KRAKEN/EPR_10B-kraken.krona $KRAKEN/EPR_11A-kraken.krona $KRAKEN/EPR_11B-kraken.krona $KRAKEN/EPR_11C-kraken.krona $KRAKEN/EPR_13B-kraken.krona $KRAKEN/EPR_13C-kraken.krona $KRAKEN/EPR_14B-kraken.krona $KRAKEN/EPR_14C-kraken.krona $KRAKEN/EPR_14E-kraken.krona $KRAKEN/EPR_15A-kraken.krona $KRAKEN/EPR_15B-kraken.krona $KRAKEN/EPR_8A-kraken.krona $KRAKEN/EPR_8B-kraken.krona $KRAKEN/EPR_9A-kraken.krona $KRAKEN/EPR_9B-kraken.krona
#
ktImportText -o $KRAKEN/WA-kraken.html $KRAKEN/WAM_ALR-kraken.krona $KRAKEN/WAM_CCR-kraken.krona $KRAKEN/WAM_IPI-kraken.krona $KRAKEN/WAM_MAR-kraken.krona $KRAKEN/WAM_MYS-kraken.krona $KRAKEN/WAM_PBL-kraken.krona $KRAKEN/WAM_PJN-kraken.krona $KRAKEN/WAM_PPR-kraken.krona $KRAKEN/WAM_PST-kraken.krona $KRAKEN/WAM_ROL-kraken.krona $KRAKEN/WAM_SCR-kraken.krona $KRAKEN/WAM_SGN-kraken.krona $KRAKEN/WAM_SIS-kraken.krona $KRAKEN/WAM_TWN-kraken.krona $KRAKEN/WAR_ALR-kraken.krona $KRAKEN/WAR_CCR-kraken.krona $KRAKEN/WAR_IPI-kraken.krona $KRAKEN/WAR_MAR-kraken.krona $KRAKEN/WAR_MYS-kraken.krona $KRAKEN/WAR_PBL-kraken.krona $KRAKEN/WAR_PJN-kraken.krona $KRAKEN/WAR_PPR-kraken.krona $KRAKEN/WAR_PSK-kraken.krona $KRAKEN/WAR_PST-kraken.krona $KRAKEN/WAR_RNW-kraken.krona $KRAKEN/WAR_ROL-kraken.krona $KRAKEN/WAR_SCR-kraken.krona $KRAKEN/WAR_SGL-kraken.krona $KRAKEN/WAR_SIS-kraken.krona
#
source deactivate
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>


## Import Kraken Annotations

At this point we now add the kraken summary annotations to each individual `PROFILE.db` created in the Snakemake workflow. Again we used lists of sample names and  `for` loops to run through each sample but since `PROFILE.db` files are separated by assembly we used one list for the EP samples ([list_EP.txt](../files/list_EP.txt)) and another for the WA samples ([list_WA.txt](../files/list_WA.txt)).

```bash
for sample in `cat list_WA.txt`
do
    anvi-import-taxonomy-for-layers -p 05_ANVIO_PROFILE/WA/$sample/PROFILE.db --parse krakenuniq -i 07_TAXONOMY/KRAKEN/READS/$sample-kraken_mpa_report.txt
done
for sample in `cat list_EP.txt`
do
    anvi-import-taxonomy-for-layers -p 05_ANVIO_PROFILE/EP/$sample/PROFILE.db --parse krakenuniq -i 07_TAXONOMY/KRAKEN/READS/$sample-kraken_mpa_report.txt
done
```

<details markdown="1"><summary>Show/hide HYDRA Import Kraken taxonomy job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -q sThC.q
#$ -l mres=5G,h_data=5G,h_vmem=5G
#$ -cwd
#$ -j y
#$ -N job_06_import_taxonomy_for_layers
#$ -o hydra_logs/job_06_import_taxonomy_for_layers.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------Import Kraken Annotations------------------- #
for sample in `cat list_WA.txt`
do
    anvi-import-taxonomy-for-layers -p 05_ANVIO_PROFILE/WA/$sample/PROFILE.db --parse krakenuniq -i 07_TAXONOMY/KRAKEN/READS/$sample-kraken_mpa_report.txt
done
#
for sample in `cat list_EP.txt`
do
    anvi-import-taxonomy-for-layers -p 05_ANVIO_PROFILE/EP/$sample/PROFILE.db --parse krakenuniq -i 07_TAXONOMY/KRAKEN/READS/$sample-kraken_mpa_report.txt
done
#
source deactivate
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>


## SCG Taxonomy &  Pfam

Here we run  single-copy core genes in the `contigs.db` with taxonomic names against a local SCG taxonomy database. After this we can run  `anvi-estimate-scg-taxonomy` to estimate taxonomy at genome-, collection-, or metagenome-level. We also needed to rerun the Pfam analysis because the original workflow failed at this step because of an improperly entered db path.

```bash
# Against SCG
anvi-run-scg-taxonomy -c 03_CONTIGS/WA-contigs.db -P 1 -T $NSLOTS
anvi-run-scg-taxonomy -c 03_CONTIGS/EP-contigs.db -P 1 -T $NSLOTS
# Against Pfam
anvi-run-pfams -c 03_CONTIGS/WA-contigs.db --pfam-data-dir /pool/genomics/stri_istmobiome/dbs/pfam_db/ -T $NSLOTS
anvi-run-pfams -c 03_CONTIGS/EP-contigs.db --pfam-data-dir /pool/genomics/stri_istmobiome/dbs/pfam_db/ -T $NSLOTS
```

<details markdown="1"><summary>Show/hide HYDRA SCG & Pfam job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 20
#$ -q mThM.q
#$ -l mres=200G,h_data=10G,h_vmem=10G,himem
#$ -cwd
#$ -j y
#$ -N job_07_run_scg_tax_and_pfam
#$ -o hydra_logs/job_07_run_scg_tax_and_pfam.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------Setup tmp directories------------------- #
rm -r /pool/genomics/stri_istmobiome/dbs/scgs-taxonomy-data/tmp_data/
mkdir -p /pool/genomics/stri_istmobiome/dbs/scgs-taxonomy-data/tmp_data/
TMPDIR="/pool/genomics/stri_istmobiome/dbs/scgs-taxonomy-data/tmp_data/"
#
rm -r /pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/
mkdir -p /pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/
TMPDIR="/pool/genomics/stri_istmobiome/dbs/pfam_db/tmp_data/"
#
# ----------------run anvio commands------------------- #
#
anvi-run-scg-taxonomy -c 03_CONTIGS/WA-contigs.db -P 1 -T $NSLOTS
anvi-run-scg-taxonomy -c 03_CONTIGS/EP-contigs.db -P 1 -T $NSLOTS
#
anvi-run-pfams -c 03_CONTIGS/WA-contigs.db --pfam-data-dir /pool/genomics/stri_istmobiome/dbs/pfam_db/ -T $NSLOTS
anvi-run-pfams -c 03_CONTIGS/EP-contigs.db --pfam-data-dir /pool/genomics/stri_istmobiome/dbs/pfam_db/ -T $NSLOTS
#
echo = `date` job $JOB_NAME done
</code></pre>
</details>


## Merge Profiles

Now that everything is added into the `contig.db` for the EP and WA assemblies and layer taxonomy is added to the individual `PROFILE.db`, its now time to merge all `PROFILE.db` into a single database for each assembly.

```bash
anvi-merge 05_ANVIO_PROFILE/EP/*/PROFILE.db -c 03_CONTIGS/EP-contigs.db -o 06_MERGED/EP
anvi-merge 05_ANVIO_PROFILE/WA/*/PROFILE.db -c 03_CONTIGS/WA-contigs.db -o 06_MERGED/WA
```

<details markdown="1"><summary>Show/hide HYDRA  Merge Profile databases job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q mThM.q
#$ -l mres=200G,h_data=100G,h_vmem=100G,himem
#$ -cwd
#$ -j y
#$ -N job_08_merge_profile_wa
#$ -o hydra_logs/job_08_merge_profile_wa.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------run anvio commands------------------- #
#
anvi-merge 05_ANVIO_PROFILE/EP/*/PROFILE.db -c 03_CONTIGS/EP-contigs.db -o 06_MERGED/EP
anvi-merge 05_ANVIO_PROFILE/WA/*/PROFILE.db -c 03_CONTIGS/WA-contigs.db -o 06_MERGED/WA
#
echo = `date` job $JOB_NAME done
</code></pre>
</details>


## Import Virsorter

Now that we have a merged profile database for each assembly, we can deal with the virsorter data. We did this last for two reasons. First, we wanted to get a better idea of  the total abundance of the viral community by looking at the other taxonomic annotations. Because the viral community is > 10% we decided to use the virome decontamination data from Virsorter. Second, we need to add the VirSorter annotations to the merged `PROFILE.dbs` as a `COLLECTION`.

This is a multi-step process that is explained in great detail [here](http://merenlab.org/2018/02/08/importing-virsorter-annotations/). But please note that depending on the state of affairs with VirSorter and/or anvi'o, you may need to modify the output of `anvi-export-gene-calls` per VirSorter GitHub [issue 65](https://github.com/simroux/VirSorter/issues/65).


<details markdown="1"><summary>Show/hide  The issue and the fix</summary>
<pre><code>
The tool virsorter_to_anvio.py uses the output of anvi-export-gene-calls (among other files)
to import virsorter annotation into anvio dbs. The script relies on the output columns to be
in a particular order. That order has changed at some point in anvio's recent past.
#
VirSorter requires the following format:
gene_callers_id contig start stop direction partial source version
#
But it is now:
gene_callers_id contig direction partial source start stop version aa_sequence
#
So the columns need to be rearranged or the script changed.
I wish I could offer I script fix :) but I just used awk.
#
awk 'BEGIN {FS="\t"; OFS="\t"} {print $1, $2, $6, $7, $3, $4, $5, $8}' all_gene_calls_TEMP.txt > all_gene_calls.txt
#
This rearranges and eliminates the aa_sequence column.
</code></pre>
</details>

First we need to grab some parsing scripts.

```bash
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/virsorter_to_anvio.py -P $VIRSORTER/helper_scripts
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/hallmark_to_function_files/db1_hallmark_functions.txt -P $VIRSORTER/helper_scripts
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/hallmark_to_function_files/db2_hallmark_functions.txt -P $VIRSORTER/helper_scripts
```

Next, export the files we need for the annotations from the contig databases.

```bash
anvi-export-table 03_CONTIGS/WA-contigs.db  --table splits_basic_info -o $VIRSORTER/WA-splits_basic_info.txt
anvi-export-table 03_CONTIGS/EP-contigs.db  --table splits_basic_info -o $VIRSORTER/EP-splits_basic_info.txt
#
anvi-export-gene-calls -c 03_CONTIGS/WA-contigs.db -o $VIRSORTER/WA-all_gene_calls.txt
anvi-export-gene-calls -c 03_CONTIGS/EP-contigs.db -o $VIRSORTER/EP-all_gene_calls.txt
```
Run the VirSorter parsing scripts.

```bash
python $VIRSORTER/helper_scripts/virsorter_to_anvio.py --db 2 -a $VIRSORTER/WA/Metric_files/VIRSorter_affi-contigs.tab -g $VIRSORTER/WA/VIRSorter_global-phage-signal.csv -s $VIRSORTER/WA-splits_basic_info.txt -n $VIRSORTER/WA-all_gene_calls.txt -f $VIRSORTER/helper_scripts/db2_hallmark_functions.txt -A $VIRSORTER/WA-virsorter_additional_info.txt -F $VIRSORTER/WA-virsorter_annotations.txt -C $VIRSORTER/WA-virsorter_collection.txt
#
python $VIRSORTER/helper_scripts/virsorter_to_anvio.py --db 2 -a $VIRSORTER/EP/Metric_files/VIRSorter_affi-contigs.tab -g $VIRSORTER/EP/VIRSorter_global-phage-signal.csv -s $VIRSORTER/EP-splits_basic_info.txt -n $VIRSORTER/EP-all_gene_calls.txt -f $VIRSORTER/helper_scripts/db2_hallmark_functions.txt -A $VIRSORTER/EP-virsorter_additional_info.txt -F $VIRSORTER/EP-virsorter_annotations.txt -C $VIRSORTER/EP-virsorter_collection.txt
```
If you get an error you may need to reformat the gene calls files.

```bash
awk 'BEGIN {FS="\t"; OFS="\t"} {print $1, $2, $6, $7, $3, $4, $5, $8}' all_gene_calls_TEMP.txt > all_gene_calls.txt
```

Finally, import all of the data in to the contig and profile dbs from each assembly.

```bash
anvi-import-misc-data $VIRSORTER/WA-virsorter_additional_info.txt -p 06_MERGED/WA/PROFILE.db  --target-data-table items
anvi-import-collection $VIRSORTER/WA-virsorter_collection.txt -c 03_CONTIGS/WA-contigs.db -p 06_MERGED/WA/PROFILE.db -C VIRSORTER
anvi-import-functions -c 03_CONTIGS/WA-contigs.db -i $VIRSORTER/WA-virsorter_annotations.txt
#
anvi-import-misc-data $VIRSORTER/EP-virsorter_additional_info.txt -p 06_MERGED/EP/PROFILE.db  --target-data-table items
anvi-import-collection $VIRSORTER/EP-virsorter_collection.txt -c 03_CONTIGS/EP-contigs.db -p 06_MERGED/EP/PROFILE.db -C VIRSORTER
anvi-import-functions -c 03_CONTIGS/EP-contigs.db -i $VIRSORTER/EP-virsorter_annotations.txt
```

<details markdown="1"><summary>Show/hide HYDRA Virsorter annotation import job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 10
#$ -q sThC.q
#$ -l mres=50G,h_data=5G,h_vmem=5G
#$ -cwd
#$ -j y
#$ -N job_09_parse_virsorter
#$ -o hydra_logs/job_09_parse_virsorter.log
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
# ----------------Get VirSorter Scripts------------------- #
VIRSORTER='/pool/genomics/stri_istmobiome/data/seq/TRANS_WATER/07_TAXONOMY/VIRSORTER_VIROME/'
#
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/virsorter_to_anvio.py -P $VIRSORTER/helper_scripts
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/hallmark_to_function_files/db1_hallmark_functions.txt -P $VIRSORTER/helper_scripts
wget https://raw.githubusercontent.com/brymerr921/VirSorterParser/master/hallmark_to_function_files/db2_hallmark_functions.txt -P $VIRSORTER/helper_scripts
# ----------------Export files for annotation------------------- #
anvi-export-table 03_CONTIGS/WA-contigs.db  --table splits_basic_info -o $VIRSORTER/WA-splits_basic_info.txt
anvi-export-table 03_CONTIGS/EP-contigs.db  --table splits_basic_info -o $VIRSORTER/EP-splits_basic_info.txt
#
anvi-export-gene-calls -c 03_CONTIGS/WA-contigs.db -o $VIRSORTER/WA-all_gene_calls.txt
anvi-export-gene-calls -c 03_CONTIGS/EP-contigs.db -o $VIRSORTER/EP-all_gene_calls.txt
# ----------------run virsorter command for anvio import------------------- #
python $VIRSORTER/helper_scripts/virsorter_to_anvio.py --db 2 -a $VIRSORTER/WA/Metric_files/VIRSorter_affi-contigs.tab -g $VIRSORTER/WA/VIRSorter_global-phage-signal.csv -s $VIRSORTER/WA-splits_basic_info.txt -n $VIRSORTER/WA-all_gene_calls.txt -f $VIRSORTER/helper_scripts/db2_hallmark_functions.txt -A $VIRSORTER/WA-virsorter_additional_info.txt -F $VIRSORTER/WA-virsorter_annotations.txt -C $VIRSORTER/WA-virsorter_collection.txt
#
python $VIRSORTER/helper_scripts/virsorter_to_anvio.py --db 2 -a $VIRSORTER/EP/Metric_files/VIRSorter_affi-contigs.tab -g $VIRSORTER/EP/VIRSorter_global-phage-signal.csv -s $VIRSORTER/EP-splits_basic_info.txt -n $VIRSORTER/EP-all_gene_calls.txt -f $VIRSORTER/helper_scripts/db2_hallmark_functions.txt -A $VIRSORTER/EP-virsorter_additional_info.txt -F $VIRSORTER/EP-virsorter_annotations.txt -C $VIRSORTER/EP-virsorter_collection.txt
# ----------------import annotation data------------------- #
#
anvi-import-misc-data $VIRSORTER/WA-virsorter_additional_info.txt -p 06_MERGED/WA/PROFILE.db  --target-data-table items
anvi-import-collection $VIRSORTER/WA-virsorter_collection.txt -c 03_CONTIGS/WA-contigs.db -p 06_MERGED/WA/PROFILE.db -C VIRSORTER
anvi-import-functions -c 03_CONTIGS/WA-contigs.db -i $VIRSORTER/WA-virsorter_annotations.txt
#
anvi-import-misc-data $VIRSORTER/EP-virsorter_additional_info.txt -p 06_MERGED/EP/PROFILE.db  --target-data-table items
anvi-import-collection $VIRSORTER/EP-virsorter_collection.txt -c 03_CONTIGS/EP-contigs.db -p 06_MERGED/EP/PROFILE.db -C VIRSORTER
anvi-import-functions -c 03_CONTIGS/EP-contigs.db -i $VIRSORTER/EP-virsorter_annotations.txt
#
deactivate
#
echo = `date` job $JOB_NAME done
</code></pre>
</details>


## GhostKOALA Annotations

The last thing we can do is run KEGG annotations using the [GhostKOALA server](http://www.kegg.jp/ghostkoala/). Running GhostKOALA/KEGG  was pretty easy thanks to this [handy tutorial](http://merenlab.org/2018/01/17/importing-ghostkoala-annotations/).

The steps we used to annotate genes with GhostKOALA/KEGG are reproduced from the tutorial.

A little setup first. Make a directory for the analysis and grab the parsing script.
```bash
mkdir -p GhostKOALA/
git clone https://github.com/edgraham/GhostKoalaParser.git
```

Now we need to export anvio gene calls as amino acid sequences.

```bash
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/WA-contigs.db --get-aa-sequences -o GhostKOALA/WA-protein-sequences.fa
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/EP-contigs.db --get-aa-sequences -o GhostKOALA/EP-protein-sequences.fa
```

Because of a peculiarity with the way the server handles fasta files we need to modify the fatsa headers. Anvio gene calls begin with a digit and the server doesn't like that. So we will the prefix `genecall` to every gene call ID.

```bash
sed -i 's/>/>genecall_/' GhostKOALA/WA-protein-sequences.fa
sed -i 's/>/>genecall_/' GhostKOALA/EP-protein-sequences.fa
```

At this point we can upload our modified amino acid gene call files to the [GhostKOALA](http://www.kegg.jp/ghostkoala/) server.

{{% alert warning %}}
You can only run one instance of GhostKOALA at a time per email address and the upload limit is 300Mb. You may need to split large fasta files in to smaller chunks.
{{% /alert %}}

<details markdown="1"><summary>Show/hide HYDRA GhostKOALA annotation import job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 20
#$ -q sThM.q
#$ -l mres=140G,h_data=7G,h_vmem=7G,himem
#$ -cwd
#$ -j y
#$ -N job_06_run_ghostkoala
#$ -o hydra_logs/job_01_run_ghostkoala.log
#$ -M scottjj@si.edu
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------Setup -------------- #
#
mkdir -p GhostKOALA/
git clone https://github.com/edgraham/GhostKoalaParser.git
# ----------------Export Anvio Gene Calls -------------- #
#
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/WA-contigs.db --get-aa-sequences -o GhostKOALA/WA-protein-sequences.fa
anvi-get-sequences-for-gene-calls -c 03_CONTIGS/EP-contigs.db --get-aa-sequences -o GhostKOALA/EP-protein-sequences.fa
# ----------------Reformat fata headers -------------- #
#
sed -i 's/>/>genecall_/' 07_GhostKOALA/WA-protein-sequences.fa
sed -i 's/>/>genecall_/' 07_GhostKOALA/EP-protein-sequences.fa
# ----------------STOP!!! Run GhostKOALA-------------- #
# At this point you must upload modified gene calls to the SERVER
# http://www.kegg.jp/ghostkoala/
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>

## Import GhostKOALA

Once the jobs have finished we can import the data into our contig databases. In the repository you cloned earlier there is a file called `KO_Orthology_ko00001.txt`. [See this](http://merenlab.org/2018/01/17/importing-ghostkoala-annotations/#generate-the-kegg-orthology-table) section for an explaination of the next step about converting the KEGG Orthology assignments to functions. We need this htext file to match the orthologies with function.

```
mkdir GhostKOALA/GhostKoalaParser
wget 'https://www.genome.jp/kegg-bin/download_htext?htext=ko00001&format=htext&filedir=' -O GhostKOALA/GhostKoalaParser/ko00001.keg
# Set the variable path
kegfile="GhostKOALA/GhostKoalaParser/ko00001.keg"
```
Now we need to parse the file using this code snippet from the tutorial.

```bash
while read -r prefix content
do
    case "$prefix" in A) col1="$content";; \
                      B) col2="$content" ;; \
                      C) col3="$content";; \
                      D) echo -e "$col1\t$col2\t$col3\t$content";;
    esac
done < <(sed '/^[#!+]/d;s/<[^>]*>//g;s/^./& /' < "$kegfile") > GhostKOALA/GhostKoalaParser/KO_Orthology_ko00001.txt
```

Time to parse the annotation file...

```bash
python GhostKOALA/GhostKoalaParser/KEGG-to-anvio --KeggDB GhostKOALA/GhostKoalaParser//KO_Orthology_ko00001.txt -i GhostKOALA/wa_ko.txt -o GhostKOALA/wa-KeggAnnotations-AnviImportable.txt
python GhostKOALA/GhostKoalaParser/KEGG-to-anvio --KeggDB GhostKOALA/GhostKoalaParser//KO_Orthology_ko00001.txt -i GhostKOALA/ep_ko.txt -o GhostKOALA/ep-KeggAnnotations-AnviImportable.txt
```

...and parse the KEGG taxonomy

```bash
python GhostKOALA/GhostKoalaParser/GhostKOALA-taxonomy-to-anvio GhostKOALA/wa.out.top GhostKOALA/wa-KeggTaxonomy.txt
python GhostKOALA/GhostKoalaParser/GhostKOALA-taxonomy-to-anvio GhostKOALA/ep.out.top GhostKOALA/ep-KeggTaxonomy.txt
```

Now we can import it all into the anvio contig databases.

```bash
anvi-import-functions -c 03_CONTIGS/WA-contigs.db -i GhostKOALA/wa-KeggAnnotations-AnviImportable.txt
anvi-import-functions -c 03_CONTIGS/EP-contigs.db -i GhostKOALA/ep-KeggAnnotations-AnviImportable.txt
#
anvi-import-taxonomy-for-genes -c 03_CONTIGS/WA-contigs.db -i GhostKOALA/wa-KeggTaxonomy.txt -p default_matrix
anvi-import-taxonomy-for-genes -c 03_CONTIGS/EP-contigs.db -i GhostKOALA/ep-KeggTaxonomy.txt -p default_matrix
```

And that's it!

<details markdown="1"><summary>Show/hide HYDRA GhostKOALA parsing job script</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 20
#$ -q sThM.q
#$ -l mres=140G,h_data=7G,h_vmem=7G,himem
#$ -cwd
#$ -j y
#$ -N job_06_run_ghostkoala2
#$ -o hydra_logs/job_01_run_ghostkoala2.log
#$ -M scottjj@si.edu
#
# ----------------Modules------------------------- #
module load gcc/4.9.2
#
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Anvio -------------- #
#
export PATH=/home/scottjj/miniconda3:$PATH
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which python
python --version
source /home/scottjj/virtual-envs/anvio-master/bin/activate
which python
python --version
which anvi-interactive
diamond --version
anvi-self-test -v
#
# ----------------Generate the KEGG orthology table-------------- #
mkdir GhostKOALA/GhostKoalaParser
wget 'https://www.genome.jp/kegg-bin/download_htext?htext=ko00001&format=htext&filedir=' -O GhostKOALA/GhostKoalaParser/ko00001.keg
#
kegfile="GhostKOALA/GhostKoalaParser/ko00001.keg"
#
while read -r prefix content
do
    case "$prefix" in A) col1="$content";; \
                      B) col2="$content" ;; \
                      C) col3="$content";; \
                      D) echo -e "$col1\t$col2\t$col3\t$content";;
    esac
done < <(sed '/^[#!+]/d;s/<[^>]*>//g;s/^./& /' < "$kegfile") > GhostKOALA/GhostKoalaParser/KO_Orthology_ko00001.txt
#
# ----------------Parsing the results from GhostKOALA-------------- #
#
python GhostKOALA/GhostKoalaParser/KEGG-to-anvio --KeggDB GhostKOALA/GhostKoalaParser//KO_Orthology_ko00001.txt -i GhostKOALA/wa_ko.txt -o GhostKOALA/wa-KeggAnnotations-AnviImportable.txt
python GhostKOALA/GhostKoalaParser/KEGG-to-anvio --KeggDB GhostKOALA/GhostKoalaParser//KO_Orthology_ko00001.txt -i GhostKOALA/ep_ko.txt -o GhostKOALA/ep-KeggAnnotations-AnviImportable.txt
#
python GhostKOALA/GhostKoalaParser/GhostKOALA-taxonomy-to-anvio GhostKOALA/wa.out.top GhostKOALA/wa-KeggTaxonomy.txt
python GhostKOALA/GhostKoalaParser/GhostKOALA-taxonomy-to-anvio GhostKOALA/ep.out.top GhostKOALA/ep-KeggTaxonomy.txt
# ----------------Importing GhostKOALA results-------------- #
#
anvi-import-functions -c 03_CONTIGS/WA-contigs.db -i GhostKOALA/wa-KeggAnnotations-AnviImportable.txt
anvi-import-functions -c 03_CONTIGS/EP-contigs.db -i GhostKOALA/ep-KeggAnnotations-AnviImportable.txt
#
anvi-import-taxonomy-for-genes -c 03_CONTIGS/WA-contigs.db -i GhostKOALA/wa-KeggTaxonomy.txt -p default_matrix
anvi-import-taxonomy-for-genes -c 03_CONTIGS/EP-contigs.db -i GhostKOALA/ep-KeggTaxonomy.txt -p default_matrix
#
echo = `date` job $JOB_NAME don
</code></pre>
</details>

## Conclusion

This section of the workflow is complete. Lets take a look at the what we have so far.

  1) `00_LOGS` Individual log files for each step of the snakemake workflow.
  2) `00_TRIMMED` Eight trimmed, compressed fastq files for each sample.
  3) `01_QC` Merge forward (R1) and reverse (R2) QC'ed fastq files and QC `STATS` file for each sample. Also the `qc-report.txt` file which is a summary table of all QC results.
  4) `02_FASTA` A contig fasta file and reformat report for each assembly.
  5) `03_CONTIGS` An annotated contig database for each assembly. Taxonomic annotations = `centrifuge`.
  6) `04_MAPPING` Short read BAM files.
  7) `05_ANVIO_PROFILE` Individual profile database for each sample.
  8) `06_MERGED` Single merged profile database for each assembly.



Now we have fully annotated contig databases, individual and merged profile databases, and summary data to work with. In the next section we can look at some of the summary data and then move on to rebuilding genomes from the assemblies.
