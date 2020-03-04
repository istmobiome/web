---
title: Annotation Databases &  Software
linktitle: Databases
summary:
date: 2019-12-07T16:44:26-05:00
lastmod: 2019-12-07T16:44:26-05:00
draft: false
toc: true
type: docs
weight: 40
# Add menu entry to sidebar.
# - Substitute `example` with the name of your course/documentation folder.
# - name: Declare this menu item as a parent with ID `name`.
# - parent: Reference a parent ID if this page is a child.
# - weight: Position of link in menu.
menu:
  trans-water:
    parent: ENVIRONMENT SETUP
    namet: Databases
    weight: 40
---

<br/>

There are two main types of annotations we are interested in for this metagenomic project---**taxonomic** and **functional**---and there are many, many ways to accomplish both of these goals. This next section involves building the databases and installing any additional tools we need for annotation.

Lets start with the tools and databases for **taxonomic** classification.

## Taxonomic Classification

anvi-setup-scg-databases


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

After kaiju is installed, the next thing to do is generate the database. You can find a description of kaiju databases [here](https://github.com/bioinformatics-centre/kaiju#creating-the-reference-database-and-index). I downloaded and formatted the `nr` and `mar` databases.

Simply run the `kaiju-makedb` command and specify a database.

```bash
kaiju-makedb -s mar
# and/or
kaiju-makedb -s nr_euk
```

The `mar` database is 19GB and the `nr_euk` is 90GB.

### KrakenUniq

*[KrakenUniq](https://github.com/fbreitwieser/krakenuniq) (formerly KrakenHLL) is a novel metagenomics classifier that combines the fast k-mer-based classification of Kraken with an efficient algorithm for assessing the coverage of unique k-mers found in each species in a dataset.*

Installed in separate conda environment

```bash
conda create -n krakenuniq
conda install krakenuniq
conda activate krakenuniq
```

I was unable to build a database.

### VirSorter

*[VirSorter](https://github.com/simroux/VirSorter), a tool designed to detect viral signal in these different types of microbial sequence data in both a reference-dependent and reference-independent manner, leveraging probabilistic models and extensive virome data to maximize detection of novel viruses.*

I generally followed [this recipe](https://github.com/simroux/VirSorter) for installing virsorter except I separated out the installation of dependencies to individual commands because I like to monitor any conflics between packages.

```bash
conda create --name virsorter python==3.6 -
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

### Concluding remarks

At this point we should be good to go with the main setup. We may need to install other tools along the way---we can add those instructions here. Or you may decide to use other tools instead. For example, we are using [megahit](https://github.com/voutcn/megahit) for the assemblies but you may want to use [metaspades](https://github.com/ablab/spades) or [idba_ud](https://github.com/loneknightpy/idba). These packages need to be installed separately.
