+++
# A Demo section created with the Blank widget.
# Any elements can be added in the body: https://sourcethemes.com/academic/docs/writing-markdown-latex/
# Add more sections by duplicating this file and customizing to your requirements.

widget = "blank"  # See https://sourcethemes.com/academic/docs/page-builder/
headless = true  # This file represents a page section.
active = true  # Activate this widget? true/false
weight = 80  # Order that this section will appear.

title = "About <small>the</small> Site"
subtitle = "The purpose of this site is to make our science tranparent & reproducible."

[design]
  # Choose how many columns the section has. Valid values: 1 or 2.
  columns = "2"

[design.background]
  # Apply a background color, gradient, or image.
  #   Uncomment (by removing `#`) an option to apply it.
  #   Choose a light or dark text color by setting `text_color_light`.
  #   Any HTML color name or Hex value is valid.

  # Background color.
  # color = "navy"

  # Background gradient.
  # gradient_start = "DeepSkyBlue"
  # gradient_end = "SkyBlue"

  # Background image.
#  image = "headers/bubbles-wide.jpg"  # Name of image in `static/img/`.
#  image_darken = 0.6  # Darken the image? Range 0-1 where 0 is transparent and 1 is opaque.
#  image_size = "cover"  #  Options are `cover` (default), `contain`, or `actual` size.
#  image_position = "center"  # Options include `left`, `center` (default), or `right`.
#  image_parallax = true  # Use a fun parallax-like fixed background effect? true/false

  # Text color (true=light or false=dark).
  text_color_light = false

[design.spacing]
  # Customize the section spacing. Order is top, right, bottom, left.
  padding = ["20px", "0", "20px", "0"]

[advanced]
 # Custom CSS.
 css_style = ""

 # CSS class.
 css_class = ""
+++

This site was contructed to capture **bioinformatic workflows, raw data, data products**, and any other helpful information in our quest to understand marine microbes. To that end, we do our best to document every step of the process. We hope that what we lack in fun content we make up for with useful content.

**A quick note about how to use the site**. Most of the site is pretty straightforward. Some presentations, obligatory bios, etc. However the bulk of the site is made up of the various [PROJECTS](#projects) and this is worth a moment of explaination. Think of a PROJECT as a website within this website. Each PROJECT has a home page where you will find a brief project overview plus quick links to the PROJECT workflow, raw data, GitHub repo, code, etc.

Just so you know, some PROJECT home pages link to workflows outside of the istmobiome.rbind.io to  github.io sites. This is an issue related to the difficulty of running R code on a Hugo website.

Keep your eye out for an <i class="fas fa-pencil-alt"> Edit this page</i> link on the bottom of most pages. You can edit these pages in GitHub and then submit a pull request so we can incorporate your suggestions.

Finally, if you are a HYDRA user we have also embedded  scripts for every job we ran on the cluster.  Keep your eye out for the **Show/hide HYDRA script**. The scripts are always hidden because they are long. Click the arrow to see the script. Have fun!

<details markdown="1"><summary>Show/hide example HYDRA  job script</summary>
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
echo + `date` job $JOB_NAME started in $QUEUE with jobID=$JOB_ID on $HOSTNAME
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
