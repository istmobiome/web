---
title: Assembley & Annotation Summaries
linktitle: Workflow
summary:
date: 2019-12-07T16:44:26-05:00
lastmod: 2019-12-07T16:44:26-05:00
draft: false
toc: true
type: docs
weight: 110
bibliography: [files/cite.bib]
link-citations: true
# Add menu entry to sidebar.
# - Substitute `example` with the name of your course/documentation folder.
# - name: Declare this menu item as a parent with ID `name`.
# - parent: Reference a parent ID if this page is a child.
# - weight: Position of link in menu.
menu:
  trans-water:
    parent: WORKFLOWS
    name: Data Summary
    weight: 110
exclude_jquery: true
---



<br/>

## About

In this section of the workflow we summarize some of the data generated thus far before moving on to genome reconstruction.

## QC & Contig Stats

### QC results

The first thing we should do is look at the  the results of the initial QC step. For each sample, anvio spits out individual quality control reports. Thankfully anvio also concatonates those files into one table. This table contains information like the number of pairs analyzed, the total pairs passed, etc.






<iframe seamless src="../qc_table/index.html" width="100%" height="575"></iframe>

<br/>

You can  download a text version of the table using the buttons or by clicking [here](../files/qc-report.txt). A standalone and more user friendly HTML version is available [here](../qc_table/index.html).


### Assembly results

Next we can look at the results of the co-assembly, the number  of HMM hits, and the estimated number of genomes. These data not only give us a general idea of assemby quality but will also help us decide parameters for automatic clustering down the road.






<iframe seamless src="../contig_table/index.html" width="100%" height="575"></iframe>

<br/>

You can  download a text version of the table using the buttons or by clicking [here](../files/contig-stats.txt). A standalone and more user friendly HTML version is available [here](../contig_table/index.html).


## Short-read Taxonomy

Next lets take a look at the taxonomic breakdown from the krakenuniq classification of short reads. There is a lot of data here so we decided to present these as standalone HTML pages. There are two pages---one for all EP samples and the other for all WA samples---that contain separate [Krona plots](https://github.com/marbl/Krona/wiki) for each sample. In brief, a Krona plot allow hierarchical data to be explored with multi-layered pie charts. So these charts are interactive. We will use these on a few occasions so it is worth explaining in a little more  detail.

Below is an example of the Kraken taxonomy for all of the EP samples. For demonstration purposes we are placing the top level and expanded plots for sample `EPM_12A3` side-by-side but in practice it will just be a single plot. Inner rings represent higher taxonomic ranks. For example, the two most inner rings are *cellular organisms* and *viruses*. Taxonomic ranks decrease towards the outside. When a grup is expanded the inner ring becomes the highest rank. For example in the plot of the right we expanded the *Proteobacteria* (which is a phylum) so the classes form the inner rings.

<!--html_preserve-->{{% figure src="/img/krona/krona-demo.png" title=" Example Krona plot: The panel on the left controls aspects of the plot. Search by taxon name, select a specific sample, control font size, etc. On the left plot, click once over a taxa and the group is highlighted. Double-click to expand the group as seen on the right. Upper right corner provides a summary of the expanded group." %}}<!--/html_preserve-->

### Separated by Sample

Since the Kraken classification was performed BEFORE assembly we can look at the Krona plots for individual samples.

* [Kraken classifications for all Eastern Pacific samples](../ep-kraken/index.html)
* [Kraken classifications for all Western Atlantic samples](../wa-kraken/index.html)

### Combined by Ocean

If you prefer to look at all samples combined by ocean we got you covered there too.

* [Eastern Pacific Kraken classification](../ep-combo-kraken/index.html)
* [Western Atlantic Kraken classification](../wa-combo-kraken/index.html)

### Making a Table

If we want to create a taxonomic summary table for the samples we can easily do that in anvio by accessing the `layer_additional_data` table from the  merged profile database.

```bash
anvi-export-table 06_MERGED/WA/PROFILE.db --table layer_additional_data -o wa-layer_additional_data.txt
anvi-export-table 06_MERGED/EP/PROFILE.db --table layer_additional_data -o ep-layer_additional_data.txt
```

And then we simply parse out the class data and make a table.






<iframe seamless src="../class_results/index.html" width="100%" height="575"></iframe>

<br/>

You can  download a text version of the table using the buttons or by clicking [here](../files/kraken-class.txt). A standalone and more user friendly HTML version is available [here](../class_results/index.html).

## Mapping Results

Lets go ahead and look at the mapping results. This took a little swindling but in the end this crude approach worked. First we needed all the `bowtie.log` files from the `00_LOGS` directory and grab the first line of each file, which has the total number of *individual reads that were paired* (after QC) in each sample.

```bash
head -1 *.log > head.txt
```

Just like we did above, we needed to grab the `layer_additional_data` from the  merged profile databases. This table contains `total_reads_mapped`, `num_SNVs_reported`, and the taxonomic info from the short read Kraken annotations.

```bash
anvi-export-table 06_MERGED/WA/PROFILE.db --table layer_additional_data -o wa-layer_additional_data.txt
anvi-export-table 06_MERGED/EP/PROFILE.db --table layer_additional_data -o ep-layer_additional_data.txt
```
What we want is a table with `sample`, `total_reads`, `total_reads_mapped`, and `num_SNVs_reported`. After a little fancy grep work in [BBEdit](https://www.barebones.com/products/bbedit/) (the free mode) we have a table the looks like this...






<iframe seamless src="../mapping_results/index.html" width="100%" height="575"></iframe>

<br/>


You can  download a text version of the table using the buttons or by clicking [here](../files/mapping-results.txt). A standalone and more user friendly HTML version is available [here](../mapping_results/index.html).


## Contig Classification

Now we move on to the classification of contigs from each assembly. We can start with the Kaiju classification again using the Krona plots. Remember we classified the contigs the `nr` and `mar`  databases. Lets use the Krona plots to compare the results from these databases for both assemblies.

* [Eastern Pacific Kaiju classification against nr & mar](../ep-kaiju-compare/index.html)
* [Western Atlantic Kaiju classification against nr & mar](../wa-kaiju-compare/index.html)
