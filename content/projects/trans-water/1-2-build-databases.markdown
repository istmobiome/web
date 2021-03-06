---
title:  Building Annotation Databases
linktitle: Databases
summary:
date: 2019-12-07T16:44:26-05:00
lastmod: 2019-12-07T16:44:26-05:00
draft: false
toc: true
type: docs
weight: 30



# Add menu entry to sidebar.
# - Substitute `example` with the name of your course/documentation folder.
# - name: Declare this menu item as a parent with ID `name`.
# - parent: Reference a parent ID if this page is a child.
# - weight: Position of link in menu.
menu:
  trans-water:
    parent: setup
    name: Databases
    weight: 20

---


<br/>


There are two main types of annotations we are interested in for this metagenomic project---**taxonomic** and **functional**---and there are many, many ways to accomplish both of these goals. This next section involves building the databases and installing any additional tools we need for annotation.

Lets start with the tools and databases for **taxonomic** classification.

## Taxonomic Classification

There are many algorithms and databases for taxonomic classification. We will use [KrakenUniq](https://github.com/fbreitwieser/krakenuniq) to classify short reads. We will also use [Centrifuge](https://ccb.jhu.edu/software/centrifuge/), [Kaiju](https://github.com/bioinformatics-centre/kaiju), and [VirSorter](https://github.com/simroux/VirSorter) for contigs. Anvio has methods of importing data from each of these approaches but if you have a have a favorite tool/database there are workarounds to get most results into the appropriate anvio database.

### Centrifuge

*[Centrifuge](https://ccb.jhu.edu/software/centrifuge/) is a very rapid and memory-efficient system for the classification of DNA sequences from microbial samples...uses a novel indexing scheme based on the Burrows-Wheeler transform (BWT) and the Ferragina-Manzini (FM) index, optimized specifically for the metagenomic classification problem...*

We will use centrifuge to classify assembled reads, or contigs. Centrifuge was installed with anvi'o above so all we need to do here is build the database.


```bash
mkdir centrifuge_dbs
cd centrifuge_dbs/
wget ftp://ftp.ccb.jhu.edu/pub/infphilo/centrifuge/data/p+h+v.tar.gz
tar -zxvf p+h+v.tar.gz && rm -rf p+h+v.tar.gz
```

Extracted, these files use roughly 12GB of disk space.

That's it.


### Kaiju

*[Kaiju](https://github.com/bioinformatics-centre/kaiju)... finds maximum (in-)exact matches on the protein-level using the Burrows–Wheeler transform.* *Kaiju is a program for the taxonomic classification... of metagenomic DNA. Reads are directly assigned to taxa using the NCBI taxonomy and a reference database of protein sequences from microbial and viral genomes.*

Kaiju pretty much does the same thing as centrifige but uses a different methodological approach. First we need to install Kaiju. Again, I will run kaiju in a separate conda environment. Now I can either install kaiju from the source code or as conda package. The later is easier but often conda packages may lag behind the source code versions. I usually compare the release dates of the [conda package](https://anaconda.org/bioconda/kaiju) with the [source code](https://github.com/bioinformatics-centre/kaiju) and look at the number of downloads. In this case, the conda version of kaiju looks fine.

```bash
# create generic environment
conda create -n kaiju
conda activate kaiji
conda install -c bioconda kaiju
```

After kaiju is installed, the next thing to do is generate the database. You can find a description of kaiju databases [here](https://github.com/bioinformatics-centre/kaiju#creating-the-reference-database-and-index). I downloaded and formatted the `nr` and `mar` databases. The `nr` database is a subset of NCBI BLAST nr database containing all proteins belonging to Archaea, Bacteria and Viruses. The `mar` database contains protein sequences from all [Mar databases](https://mmp.sfb.uit.no/).

Simply run the `kaiju-makedb` command and specify a database.

```bash
kaiju-makedb -s mar
# and/or
kaiju-makedb -s nr_euk
```

The `mar` database is 19GB and the `nr_euk` is 90GB.

<details markdown="1"><summary>Show/hide Kaiju database build job details</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 2
#$ -q mThM.q
#$ -l mres=200G,h_data=100G,h_vmem=100G,himem
#$ -cwd
#$ -j y
#$ -N makeDB_e2
#$ -o makeDB_e2_2.log
# ----------------Your Commands------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------THIS Activate the conda anvio support, not anvio -------------- #
#
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate anvio-master
#
which kaiju-makedb
gcc --version
which perl
#
# ----------------For nr_euk db -------------- #
kaiju-makedb -h
kaiju-makedb -s nr_euk
#
# ----------------For mar db -------------- #
kaiju-makedb -h
kaiju-makedb -s mar
#
echo = `date` job $JOB_NAME done
</code></pre>
</details>

### KrakenUniq

*[KrakenUniq](https://github.com/fbreitwieser/krakenuniq) (formerly KrakenHLL) is a novel metagenomics classifier that combines the fast k-mer-based classification of Kraken with an efficient algorithm for assessing the coverage of unique k-mers found in each species in a dataset.*

Installed in separate conda environment

```bash
conda create -n krakenuniq
conda install krakenuniq
conda activate krakenuniq
```

There are two steps to creating a database for analysis, `krakenuniq-download` which downloads the data and `krakenuniq-build` which, yup you guessed it, builds the database. For some reason building a krakenuniq database has become problematic of late. Specifically, `krakenuniq-build` hangs for days trying to write two files. `database.kraken.tsv` and `database.report.tsv`. There are a few issues about this on GitHub (e.g., [here](https://github.com/fbreitwieser/krakenuniq/issues/38) and [here](https://github.com/fbreitwieser/krakenuniq/issues/56)) but unfortunately these have not been addressed and it is unclear if the platform is still being supported. That said, I tried two databases (refseq and microbial-nt) and here is what I did.

```bash
# Download & Build refseq
krakenuniq-download --db DB --taxa "archaea,bacteria,viral,fungi,protozoa" --dust --exclude-environmental-taxa refseq/bacteria refseq/archaea refseq/fungi refseq/protozoa refseq/viral/Any viral-neighbors --threads $NSLOTS
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --jellyfish-hash-size 10000M --max-db-size 300
# Ran this after job failed to clean up
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --clean
# Download & Build microbial-nt
krakenuniq-download --db DB --dust  --exclude-environmental-taxa --threads $NSLOTS microbial-nt
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences
# Ran this after job failed to clean up
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --clean
```

<details markdown="1"><summary>Show/hide Kraken database build job details</summary>
<pre><code>
# /bin/sh
# ----------------Parameters---------------------- #
#$ -S /bin/sh
#$ -pe mthread 3
#$ -q mThM.q
#$ -l mres=450G,h_data=150G,h_vmem=150G,himem
#$ -cwd
#$ -j y
#$ -N job_00_build_kraken_db3
#$ -o job_00_build_kraken_db5.job
#
# ----------------Modules------------------------- #
module load bioinformatics/blast
#
# ----------------Load Envs------------------- #
#
echo + `date` job $JOB_NAME started in `\(QUEUE with jobID=\)`JOB_ID on $HOSTNAME
echo + NSLOTS = $NSLOTS
#
# ----------------Activate Kraken-------------- #
#
export PATH=/home/scottjj/miniconda3/bin:$PATH
source activate krakenuniq
#
# ----------------For refseq DB -------------- #
krakenuniq-download --db DB --taxa "archaea,bacteria,viral,fungi,protozoa" --dust --exclude-environmental-taxa refseq/bacteria refseq/archaea refseq/fungi refseq/protozoa refseq/viral/Any viral-neighbors --threads $NSLOTS
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --jellyfish-hash-size 10000M --max-db-size 300
# ----------------RUN after job fails -------------- #
#krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --clean
#
# ----------------For refseq DB -------------- #
krakenuniq-download --db DB --dust  --exclude-environmental-taxa --threads $NSLOTS microbial-nt
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences
# ----------------RUN after job fails -------------- #
krakenuniq-build --db DB --kmer-len 31 --threads $NSLOTS --taxids-for-genomes --taxids-for-sequences --clean
#
echo = `date` job $JOB_NAME done
</code></pre>
</details>

### VirSorter

*[VirSorter](https://github.com/simroux/VirSorter), a tool designed to detect viral signal in these different types of microbial sequence data in both a reference-dependent and reference-independent manner, leveraging probabilistic models and extensive virome data to maximize detection of novel viruses.*

I generally followed [this recipe](https://github.com/simroux/VirSorter) for installing virsorter except I separated out the installation of dependencies to individual commands because I like to monitor any conflics between packages.

```bash
conda create --name virsorter python==3.6
conda activate virsorter

conda install -c bioconda mcl=14.137
conda install -c bioconda muscle
conda install -c bioconda blast
conda install -c bioconda hmmer=3.1b2
conda install -c bioconda diamond=0.9.14

conda install -c bioconda perl-bioperl
conda install -c bioconda perl-file-which
conda install -c bioconda perl-parallel-forkmanager
conda install -c bioconda perl-list-moreutils

conda install --name virsorter -c bioconda metagene_annotator

git clone https://github.com/simroux/VirSorter.git
cd VirSorter/Scripts/
make clean
make

ln -s /home/scottjj/software/VirSorter/wrapper_phage_contigs_sorter_iPlant.pl /home/scottjj/miniconda3/envs/virsorter/bin/
ln -s /home/scottjj/software/VirSorter/Scripts/ /home/scottjj/miniconda3/envs/virsorter/bin/
```

You may get a `Bio::Seq` error when you run virsorter. If so you need to install the BioPerl GitHub repo. It is best to check first before moving on.

```bash
perl -e "use Bio::Seq;"`
```

If you get an error it means that the install failed (for some reason).

So  clone the BioPerl repo and copy the `Bio/` directory to the virsorter environment.


```bash
git clone https://github.com/bioperl/bioperl-live.git
cp -r Path/to/lib/Bio Path/to/miniconda3/envs/virsorter/lib/5.26.2/
```

Now see if the error persists.

Good to go? Now time to build the virsorter database. Pretty straightfoward actually.

```bash
wget https://zenodo.org/record/1168727/files/virsorter-data-v2.tar.gz
md5sum virsorter-data-v2.tar.gz
# md5sum should return dd12af7d13da0a85df0a9106e9346b45
tar -xvzf virsorter-data-v2.tar.gz
rm virsorter-data-v2.tar.gz
```

The uncompressed database is a little over 13GB.

## Functional Annotation

### Pfam and COG

Now we can install databases for **functional** annotation. Anvi'o has native support for building [Pfam](https://pfam.xfam.org/) and [COG](https://www.ncbi.nlm.nih.gov/COG/) databases.

Anvi'o makes this super simple.

```bash
anvi-setup-ncbi-cogs  --cog-data-dir PATH_TO_COG_DIR -T 10
anvi-setup-pfams  --pfam-data-dir PATH_TO_PFAM_DIR
```

The COG db is 2.3GB and the PFAM db is 283MB.

And thats it. We can add more databases as we need them.

### GhostKOALA/KEGG

Running GhostKOALA/KEGG  was pretty easy thanks to this [handy tutorial](http://merenlab.org/2018/01/17/importing-ghostkoala-annotations/). But since we run the analysis on the GhostKOALA webserver, there is no database to set up here. We provide more details when it comes time to actually set up for the analysis and parse the data.

## Concluding remarks

At this point we should be good to go with the main setup. We may need to install other tools along the way---we can add those instructions here. Or you may decide to use other tools instead. For example, we are using [megahit](https://github.com/voutcn/megahit) for the assemblies but you may want to use [metaspades](https://github.com/ablab/spades) or [idba_ud](https://github.com/loneknightpy/idba). These packages need to be installed separately.
