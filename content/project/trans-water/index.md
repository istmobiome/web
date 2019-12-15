---
title: Transisthmian Water Microbiomes
subtitle: Assembly-based metagenomics of coral reef & mangrove water samples from both sides of the Isthmus.
summary: Assembly-based metagenomics of coral reef & mangrove water samples from both sides of the Isthmus.
authors: []
tags: []
categories: []
date: "2019-12-05T00:00:00+01:00"
external_link:
image:
  caption: ""
  focal_point: ""
  preview_only: True
  placement:
links:
- name: Go to Project Site
  url: projects/trans-water/
#- icon: database
#  icon_pack: fas
#  name: Data
#  url: https://projectdigest.github.io/data_availability.html
#- icon: code
#  icon_pack: fas
#  name: Code
#  url: https://projectdigest.github.io/raw_code.txt
#- icon: github
#  icon_pack: fab
#  name: GitHub
#  url: https://github.com/projectdigest/web/
#- icon: newspaper
#  icon_pack: fas
#  name: Publication
#  url: https://projectdigest.github.io/publication.html
publication: null
slides:
url_code: ""
url_pdf: ""
url_slides: ""
url_video: ""

weight: 1
---

Welcome. This site provides the reproducible bioinformatics workflow for our study on the Transisthmian water column microbiome project. Here you will find details on how we setup our computational environment, program names and exact parameters we used throughout every step of the analysis, and how to access raw data. In addition we provide field/lab methods, summary data products, and other (potentially) useful information. Below

Use the buttons at the top of the page or [this link](/projects/trans-water/) to access the molecular data and complete bioinformatic workflow.


<br/>

{{% alert synopsis %}}

SYNOPSIS
<hr>
In this study, we

- Investigate the evolution of free-living marine microbes of both sides of the Isthmus of Panama; the Eastern Pacific (EP) & Western Atlantic (WA).
- Collected a total of 57 water column samples collected from mangroves and coral reefs, dominant benthic habitats.
- Each sample (4L) was filtered through a 0.22 micron filter. DNA was extracted from the filters and used for metagenomic sequencing.
-  Performed separate co-assemblies for EP samples and for WA samples.
-  Used automatic and manual binning to generate metagenome assembled genomes (MAGs) from each assembly.

{{% /alert %}}



```mermaid
graph TD
A[Hard] -->|Text| B(Round)
B --> C{Decision}
C -->|One| D[Result 1]
C -->|Two| E[Result 2]
1 --> 0(metagenomics_workflow_target_rule)
2 --> 0
3(anvi_merge) --> 0
4(anvi_merge) --> 0
5 --> 0
6(gen_qc_report) --> 0
7 --> 0
8 --> 1
9 --> 1
10 --> 1
11 --> 1
12 --> 1
13 --> 1
4 --> 2
14 --> 2
15 --> 3
16 --> 3
17 --> 3
18 --> 3
19 --> 3
20 --> 3
21 --> 3
22 --> 4
23 --> 4
24 --> 4
25 --> 4
14 --> 4
26 --> 4
27 --> 4
19 --> 5
3 --> 5
28(iu_filter_quality_minoche<br/>WAR_CCR) --> 6
29(iu_filter_quality_minoche<br/>EPR_11C) --> 6
30(iu_filter_quality_minoche<br/>WAM_MAR) --> 6
31(iu_filter_quality_minoche<br/>EPR_9A) --> 6
8 --> 7
9 --> 7
10 --> 7
11 --> 7
12 --> 7
13 --> 7
13 --> 8
13 --> 9
32(centrifuge) --> 10
13 --> 10
13 --> 11
13 --> 12
8 --> 12
33 --> 13
34 --> 14
19 --> 15
35 --> 15
18 --> 16
36(krakenuniq_mpa_report) --> 16
21 --> 16
15 --> 17
19 --> 18
37 --> 18
38 --> 19
17 --> 20
39(krakenuniq_mpa_report) --> 20
15 --> 20
18 --> 21
26 --> 22
40 --> 23
14 --> 23
26 --> 24
41(krakenuniq_mpa_report) --> 24
22 --> 24
23 --> 25
27 --> 25
42(krakenuniq_mpa_report) --> 25
43 --> 26
14 --> 26
23 --> 27
44 --> 28
44 --> 29
44 --> 30
44 --> 31
45 --> 32
46 --> 33
47 --> 34
48 --> 35
49 --> 36
50 --> 37
51 --> 38
52 --> 39
53 --> 40
54 --> 41
55 --> 42
56 --> 43
13 --> 45
57 --> 46
58(megahit<br/>EP) --> 47
59(bowtie) --> 48
60 --> 49
61 --> 49
62(bowtie) --> 50
63(megahit<br/>WA) --> 51
64 --> 52
65 --> 52
66(bowtie) --> 53
67 --> 54
68 --> 54
69 --> 55
70 --> 55
71(bowtie) --> 56
67 --> 58
69 --> 58
68 --> 58
70 --> 58
65 --> 59
64 --> 59
72 --> 59
28 --> 60
28 --> 61
60 --> 62
72 --> 62
61 --> 62
60 --> 63
64 --> 63
65 --> 63
61 --> 63
30 --> 64
30 --> 65
69 --> 66
73 --> 66
70 --> 66
31 --> 67
31 --> 68
29 --> 69
29 --> 70
67 --> 71
68 --> 71
73 --> 71
38 --> 72
34 --> 73
```
