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

Viruses in Data Set 1 | Type
---------------------|-----
Cellulophaga_phage_phi13:1_514342768 | dsDNA
Cellulophaga_phage_phi38_1_526177551 | dsDNA
Enterobacteria_phage_phiX174_962637 | ssDNA
PSA_HS2  | dsDNA
Cellulophaga_phage_phi18_1_526177061 | dsDNA
Cellulophaga_phage_phi38_2_514343001 | dsDNA
PSA_HM1 | dsDNA
PSA_HS6 | dsDNA
Cellulophaga_phage_phi18_3_526177357 | dsDNA
Enterobacteria_phage_alpha3_194304496 | ssDNA
PSA_HP1 | dsDNA
PSA_HS9 | dsDNA

If you are running this tutorial on the Ceres computer cluster the the data are available at:

 ```bash
 # Mock viral files
 /project/microbiome_workshop/metagenome/viral

 # Anvi'o files
  /project/microbiome_workshop/metagenome/mapping
```

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

# Part 1: Assembly and mapping
Create a directory for the tutorial
```bash
# In your homespace or other desired location, make a
# directory and move into it
mkdir metagenome1
cd metagenome1
# make a data directory
mkdir data
```


The data file we will be working with is here:
```bash
/project/microbiome_workshop/metagenome/viral/10142.1.149555.ATGTC.shuff.fastq.gz
```
Lets take a peek at the data.
```bash
zcat /project/microbiome_workshop/metagenome/viral/10142.1.149555.ATGTC.shuff.fastq.gz | head
```
The output should look like this:
```
@MISEQ05:522:000000000-ALD3F:1:1101:22198:19270 1:N:0:ATGTC
ATGTTCTGAATTAAGCAGGCGGTTTCCATTAATTACCTTTTCCTCTTCCTCTAATCCTATTATGAGATTTTTGAGTAAACTTATTTCTAATTCTGTTGTTTTTATAGCTGTAGTTAAAGCTTCAGATTCTTCATCTGAAACTTTAGTATCT
+
CCDBCFFFFFFFGGGGGGGGGGGGGHHHHHHHHHHHHHHHHHHHHGHHHHHHHHHHHHHHHHHHHHHHHHHHGGGHHHHHHHGHHHHHHHHHHHHHHHHHHHGHHHGHHHHHHHHHHHHHHHHHHHHHHHHHFHHGHHHHHHHHHHGHHFH
@MISEQ05:522:000000000-ALD3F:1:1101:22198:19270 2:N:0:ATGTC
GAGGAATGGTTTGTCTCCTAAAATTGATGAAAGTAGTATTCAAATTTCAGGATTAAAAGGAGTTTCTATTTTGTCTATTGCTTATGATATTAATTATTTAGATACTAAAGTTTCAGATGAAGAATCTGAAGCTTTAACTACAGCTATAAAA
+
BAACCFFFFFFFGGGGGGGGGGHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHGHHGHHIHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHHEHHHHHHHHHHGHGHHHHHHHGHGHHHHHFHH
@MISEQ05:522:000000000-ALD3F:1:1105:15449:9605 1:N:0:ATGTC
TGCTACTACCACAAGCTTCACACGCTAAGGGCACTGCAATATTATAGCTACAATAATGGCAACGCAATTGTTTTTTATGCTGATGATACGTTAGACTTACATCACAATTAGGACATTGCGGAGAATGCCCACACGTAGTACATTCCATGAT
```
We can see that this data are interleaved, paired-end based on the the "1" and "2 "after the initial identifier. From the identifier and the length of the reads we can see that the data was sequenced in 2x150 mode on an Illumina MiSeq instrument.

For speed we will be subsampling our data. the original fasta file has been shuffled to get a random sampling of reads.  To subsample we will use zcat to unzip the file and stream the data out. That data stream will be "piped" (sent to) the program head which will display the first 2 million lines and that displayed text will be written to a file with the ">" operator.

```bash
# Take a subset 500,000 reads of the data
zcat /project/microbiome_workshop/metagenome/viral/10142.1.149555.ATGTC.shuff.fastq.gz \
| head -n 2000000 | gzip  > data/10142.1.149555.ATGTC.subset_500k.fastq.gz
```

Raw sequencing data needs to be processed to remove artifacts. The first step in
this process is to remove contaminant sequences that are present in the sequencing process such as PhiX which is sometimes added as an internal control for sequencing runs.

```bash
# Filter out contaminant reads placing them in their own file
bbduk.sh in=data/10142.1.149555.ATGTC.subset_500k.fastq.gz \
out=data/10142.1.149555.ATGTC.subset_500k.unmatched.fq.gz \
outm=data/10142.1.149555.ATGTC.subset_500k.matched.fq.gz \
k=31 \
hdist=1 \
ftm=5” \
ref=sequencing_artifacts.fa.gz \
stats=data/contam_stats.txt

```
With shotgun paired end sequencing the size of the DNA sequenced is controlled by physically shearing the DNA and selecting DNA of a desired length with SPRI or Ampure beads.  This is an imperfect process and sometimes short pieces of DNA are sequenced.  If the DNA is shorter than the sequencing length (150pb in this case) The other adapter will be sequenced too.

We can trim off this adapter using bbduk.
```bash
# Trim the adapters using the reference file adaptors.fa (provided by bbduk)
bbduk.sh in=data/10142.1.149555.ATGTC.subset_500k.unmatched.fq.gz \
out=data/10142.1.149555.ATGTC.subset_500k.fastq.trimmed.gz \
ktrim=r \
k=23 \
mink=11 \
hdist=1 \
tbo=t
ref=adaptors.fa
```
At this point we could quality trim the right end of the reads but since we are going to merge them some of these low quality reads will likely be fixed so lets skip this for now.

```bash
# Merge the reads together
bbmerge.sh in=data/10142.1.149555.ATGTC.subset_500k.fastq.trimmed.gz \
out=data/10142.1.149555.ATGTC.subset_500k.fastq.merged.gz \
outu=../data/10142.1.149555.ATGTC.subset_500k.unmerged.fastq.gz \
usejni=t
```
Assemble the metagenome with the Succinct DeBruijn graph assembler Megahit
```bash
megahit  -m 20000000000 \
-t 2 \
--read data/10142.1.149555.ATGTC.subset_500k.merged.fastq.gz \
--12 data/10142.1.149555.ATGTC.subset_500k.unmerged.fastq.gz \
--k-list 21,41,61,81,99 \
--no-mercy \
--min-count 2 \
--out-dir data/megahit/
```
Although its not generally required, we can visualize our assemblies by generating a fastg file

```bash
megahit_toolkit contig2fastg 99 \
data/megahit/intermediate_contigs/k99.contigs.fa > data/k99.fastg
```

Download the file to  your local computer by opening a new Terminal window and coping the file

```bash
scp user.name@scinet-login.bioteam.net:~/metagenome1/data/]k99.fastg .
```

Now open Bandage and select **File > Open Graph** and load ```k99.fastg```

Hit the **Draw Graph** button. Now all the Assembled contigs are visible. Some are circular come are linear and some have bubbles in the assembly.

![Main Bandage Screen](/Microbiome-workshop/assets/metagenome/Bandage1.png)

IF you have a Stand alone copy of [Blast+](https://blast.ncbi.nlm.nih.gov/Blast.cgi?PAGE_TYPE=BlastDocs&DOC_TYPE=Download) installed on your computer you can label the contigs with with the reference genomes.

Download the reference fasta file here:
Ref_database.fna : [Download](https://usda-ars-gbru.github.io/Microbiome-workshop/assets/metagenome/Ref_database.fna)

Select **Load from Fasta File** and load ```Ref_database.fna```

Select **Run balst search** then **Close**
![Blast options screen](/Microbiome-workshop/assets/metagenome/Bandage2.png)

The contigs will now be taxonomically annotated. Take some time to explore the structure of the contigs. Which ones look correct? What kinds or errors are present in the assembly?
