---
title: "Metagenomics tutorial"
excerpt: "An example workflow for assembly based metagenomics"
layout: single
author: "Adam Rivers"
---
By Adam Rivers
{% include toc %}

# This tutorial is still under development

# Overview
Shotgun metagenomics data can be analyzed in several different approaches. Each approach is best suited for a particular group of questions. The methodological approaches can be broken down into three broad areas: read-based approaches, Assembly-based approaches and detection-based approaches. This tutorial takes an assembly-based approach. The key points of the approaches are listed in this table:

Method | Read-based | Assembly-based | Detection-based
-------|------------|----------------|----------------
Description | Read-based metagenomics analyzes unassembled reads. The first methods to be used. still valuable for quantitative analysis especially if relevant references are available. | Assembly-Based workflows attempt to assemble the reads from one or more samples, "bin" contigs from these samples into genomes then analyze the genes and contigs. | Detection-based workflows attempt to identify with very high-precision but lower sensitivity (recall) the presence of organisms of interest, often pathogens.
Typical workflow | 1. Read QC<br>2. Read merging (Bbmerge) <br>3. Mapping to NR for taxonomy data (Blast, Diamond or Last). Kmer-based approaches are also used (Kraken, Clarke).<br> 4. Mapping to functional databases (Kegg, pfam). Cross-mapping can also be used (Mocat2).<br> 5. Summaries of taxonomic and functional distributions can be analyzed. If ids were retained function within taxa analyses can be run.<br> | 1. Read QC<br>2. Read merging<br>3. Read assembly. (Megahit, Spades-meta). Generally individual samples are assembled.  Memory use is an issue. Read error correction and normalization can help.<br> 4. Mapping of reads reads from each sample are mapped back to the contigs from all samples for use in quantification and binning (bowtie2, bbmap).<br> 5. Genome binning. Contig composition and mapping data are used to bin contigs into "genome bins" (Metabat, MAxBin, Concoct).<br> 6.  <em>De-novo</em> gene annotation is done (Prodigal, MetaGeneMArk). Gene annotation is performed as in Read-centric approaches, but computational burden is lower since annotated data is ~100x smaller.<br> 7. Analysis of genome bins, functional gene phylogenetics, comparative genomics. | 1. Read QC<br>2. Read merging (if paired) <br> 3. Alignment or kmer based matching against curated dataset. 4. Heuristic for  determining the correct level to classify at and classification (Megan-LCA, Clarke, proprietary methods).
Typical questions|&bull; What it the bulk taxonomic/functional composition of these samples?<br>&bull; What new kinds of a particular functional gene family can I find?<br>&bull; How do my sites or treatments differ in taxonomic/functional composition?<br> |&bull; What are the functional and metabolic capabilities of specific microbes in my sample?<br>&bull; What is the phylogeny of gene families in my samples?<br>&bull; Do the organisms that inhabit my samples differ?<br>&bull; Are there variants within taxa in my population? | &bull; Are known organisms of interest present in my sample?<br> &bull; are known functional genes e.g. beta lactamses present in my sample?<br>
Examples | [MG-Rast](http://metagenomics.anl.gov/) | [IMG from JGI](https://img.jgi.doe.gov/cgi-bin/mer/main.cgi) | [Taxonomer](http://taxonomer.iobio.io/), [Surpi](http://chiulab.ucsf.edu/surpi/), [One Codex](https://www.onecodex.com/), [CosmosID](http://www.cosmosid.com/)

# Data sets in this tutorial

Many of the initial processing steps in metagenomics are quite computationally intensive. For this reason we will use two data sets in this tutorial: An initial dataset of a mock viral community containing a mixture of single and double stranded DNA viruses, and a second mock bacterial data set for the exploration of data in Anvi'o.

If you are running this tutorial on the Ceres computer cluster the the data is available at:
 ```bash
 # Mock viral files
 /project/microbiome_workshop/metagenome/viral

 # anvi'o files
  /project/microbiome_workshop/metagenome/mapping


# Connecting to Ceres

Ceres is the computer cluster for the USDA Agricultural Research Service's SCInet computing environment. From Terminal or Putty (for Windows users) create a secure shell connection to Ceres
```bash
ssh -X <user.name>@scinet-login.bioteam.net
```
Once you are logged into Ceres you can request access to an interactive node.
In a real analysis you would create a script that runs all the commands in sequence and submit the script through a program called Sbatch, part of the computer's job scheduling software named Slurm.

To request access to an interactive node:
```bash
# Request access to one node of the cluster
# Note that "microbiome" is a special queue for the workshop,
# to see available queues use the command "sinfo"
salloc -p microbiome -N 1 -t 40

# Set up Xvfb graphics software so that anvi'o can generate figures.
# Xvfb creates a virtual X session. This needs to be run in the background.
# If you encounter an error saying the session already exists, then use a
# different session number, e.g. "Xvfb :2"
Xvfb :1 -screen 0 800x600x16 &

# Add a Display variable to the local environment variables
export DISPLAY=:1.0

# Load the required modules
module load bbtools/gcc/64/37.02
module load megahit/gcc/64/1.1.1
```
When you are done at the end of the tutorial end your session like this.
```bash
# to log off shut down the graphics window
killall Xvfb
# sign out of the allocated node
exit
# sign out of Ceres head node
exit
```
