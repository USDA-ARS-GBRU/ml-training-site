---
title: "Metagenomics tutorial part 1: Quality control, assembly and mapping"
excerpt: "An example workflow for assembly based metagenomics"
layout: single
author: "Adam Rivers"
---
By Adam Rivers
{% include toc %}

# Overview
Shotgun metagenomics data can be analyzed using several different approaches. Each approach is best suited for a particular group of questions. The methodological approaches can be broken down into three broad areas: read-based approaches, assembly-based approaches and detection-based approaches. This tutorial takes an assembly-based approach. The key points of the approaches are listed in this table:

Method | Read-based | Assembly-based | Detection-based
-------|------------|----------------|----------------
Description | Read-based metagenomics analyzes unassembled reads. It was one of the first methods to be used. It is still valuable for quantitative analysis, especially if relevant references are available. | Assembly-based workflows attempt to assemble the reads from one or more samples, "bin" contigs from these samples into genomes then analyze the genes and contigs. | Detection-based workflows attempt to identify with very high-precision but lower sensitivity (recall) the presence of organisms of interest, often pathogens.
Typical workflow | 1. Read QC.<br>2. Read merging (Bbmerge).<br>3. Mapping to NR for taxonomy data (Blast, Diamond or Last). Kmer-based approaches are also used (Kraken, Clarke).<br> 4. Mapping to functional databases (Kegg, Pfam). Cross-mapping can also be used (Mocat2).<br> 5. Summaries of taxonomic and functional distributions can be analyzed. If read IDs were retained function within taxa analyses can be run.<br> | 1. Read QC.<br>2. Read merging.<br>3. Read assembly. (Megahit, Spades-meta). Generally individual samples are assembled.  Memory use is an issue. Read-error correction and normalization can help.<br> 4. Mapping of reads reads from each sample are mapped back to the contigs from all samples for use in quantification and binning (bowtie2, bbmap).<br> 5. Genome binning. Contig composition and mapping data are used to bin contigs into "genome bins" (Metabat, MaxBin, Concoct).<br> 6.  <em>De-novo</em> gene annotation is done (Prodigal, MetaGeneMArk). Gene annotation is performed as in read-centric approaches, but computational burden is lower since annotated data is ~100x smaller.<br> 7. Analysis of genome bins, functional gene phylogenetics, comparative genomics. | 1. Read QC<br>2. Read merging (if paired) <br> 3. Alignment or kmer based matching against curated dataset.<br> 4. Heuristics for determining the correct level to classify at and the  classification (Megan-LCA, Clarke, Surpi, Centrifuge, proprietary methods).
Typical questions|&bull; What it the bulk taxonomic/functional composition of these samples?<br>&bull; What new kinds of a particular functional gene family can I find?<br>&bull; How do my sites or treatments differ in taxonomic/functional composition?<br> |&bull; What are the functional and metabolic capabilities of specific microbes in my sample?<br>&bull; What is the phylogeny of gene families in my samples?<br>&bull; Do the organisms that inhabit my samples differ?<br>&bull; Are there variants within taxa in my population? | &bull; Are known organisms of interest present in my sample?<br> &bull; Are known functional genes e.g. beta-lactamases, present in my sample?<br>
Examples | [MG-Rast](http://metagenomics.anl.gov/) | [IMG from JGI](https://img.jgi.doe.gov/cgi-bin/mer/main.cgi) | [Taxonomer](http://taxonomer.iobio.io/), [Surpi](http://chiulab.ucsf.edu/surpi/), [One Codex](https://www.onecodex.com/), [CosmosID](http://www.cosmosid.com/)

# Data sets in this tutorial

Many of the initial processing steps in metagenomics are quite computationally intensive. For this reason we will use a small dataset from a mock viral community containing a mixture of small single- and double-stranded DNA viruses.

Viruses in Data Set 1 | Type
---------------------|-----
Cellulophaga_phage_phi13:1_514342768 | dsDNA
Cellulophaga_phage_phi38_1_526177551 | dsDNA
Enterobacteria_phage_phiX174_962637 | ssDNA
PSA_HS2 | dsDNA
Cellulophaga_phage_phi18_1_526177061 | dsDNA
Cellulophaga_phage_phi38_2_514343001 | dsDNA
PSA_HM1 | dsDNA
PSA_HS6 | dsDNA
Cellulophaga_phage_phi18_3_526177357 | dsDNA
Enterobacteria_phage_alpha3_194304496 | ssDNA
PSA_HP1 | dsDNA
PSA_HS9 | dsDNA

If you are running this tutorial on the Ceres computer cluster the data are available at:

 ```bash
 # Mock viral files
 /project/microbiome_workshop/metagenome/viral
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

# Load the required modules
module load bbtools/gcc/64/37.02
module load megahit/gcc/64/1.1.1
```

When you are done at the end of the tutorial end your session like this.
```bash
# sign out of the allocated node
exit
# sign out of Ceres head node
exit
```

# Part 1: Assembly and mapping
Create a directory for the tutorial.
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
We can see that this data are interleaved, paired-end based on the the `1` and `2` after the initial identifier. From the identifier and the length of the reads we can see that the data was sequenced in 2x150 mode on an Illumina MiSeq instrument.

For speed we will be subsampling our data. the original fasta file has been shuffled (keeping pairs together) to get a random sampling of reads.  To subsample we will use zcat to unzip the file and stream the data out. That data stream will be "piped" (sent to) the program head which will display the first 2 million lines and that displayed text will be written to a file with the `>` operator.

```bash
# Take a subset 500,000 reads of the data
time zcat /project/microbiome_workshop/metagenome/viral/10142.1.149555.ATGTC.shuff.fastq.gz \
| head -n 2000000 | gzip  > data/10142.1.149555.ATGTC.subset_500k.fastq.gz
```
Time to run: 30 seconds

Raw sequencing data needs to be processed to remove artifacts. The first step in
this process is to remove contaminant sequences that are present in the sequencing process such as PhiX which is sometimes added as an internal control for sequencing runs.

```bash
# Filter out contaminant reads placing them in their own file
time bbduk.sh in=data/10142.1.149555.ATGTC.subset_500k.fastq.gz \
out=data/10142.1.149555.ATGTC.subset_500k.unmatched.fq.gz \
outm=data/10142.1.149555.ATGTC.subset_500k.matched.fq.gz \
k=31 \
hdist=1 \
ftm=5 \
ref=/software/apps/bbtools/gcc/64/37.02/resources/sequencing_artifacts.fa.gz \
stats=data/contam_stats.txt
```
Time to run: 20 seconds

Lets look at the output:

```
BBDuk version 37.02
Initial:
Memory: max=27755m, free=27031m, used=724m

Added 8085859 kmers; time: 	5.780 seconds.
Memory: max=27755m, free=25438m, used=2317m

Input is being processed as paired
Started output streams:	0.141 seconds.
Processing time:   		14.159 seconds.

Input:                  	500000 reads 		75500000 bases.
Contaminants:           	1086 reads (0.22%) 	162900 bases (0.22%)
FTrimmed:               	500000 reads (100.00%) 	500000 bases (0.66%)
Total Removed:          	1086 reads (0.22%) 	662900 bases (0.88%)
Result:                 	498914 reads (99.78%) 	74837100 bases (99.12%)

Time:   			20.114 seconds.
Reads Processed:        500k 	24.86k reads/sec
Bases Processed:      75500k 	3.75m bases/sec
```
0.22% of our reads contained contaminants or were too short after low quality reads were removed, suggesting that our insert size selection went well.

If we look at the `contam_stats.txt` file we see that most of the contaminants were known Illumina linkers. Because so many of these have been erroneously been deposited in the NT database, Blasting the contaminant sequences will turn up some very odd things. The first sequence in our contaminants file is supposedly a Carp.
```
#File   data/10142.1.149555.ATGTC.subset_500k.fastq.gz
#Total  500000
#Matched        935     0.18700%
#Name   Reads   ReadsPct
contam_27       490     0.09800%
contam_11       252     0.05040%
contam_10       81      0.01620%
contam_12       73      0.01460%
contam_9        18      0.00360%
contam_257      13      0.00260%
contam_175      3       0.00060%
contam_256      3       0.00060%
contam_15       1       0.00020%
contam_8        1       0.00020%
```

With shotgun paired end sequencing the size of the DNA sequenced is controlled by physically shearing the DNA and selecting DNA of a desired length with SPRI or Ampure beads.  This is an imperfect process and sometimes short pieces of DNA are sequenced.  If the DNA is shorter than the sequencing length (150pb in this case) The other adapter will be sequenced too.

We can trim off this adapter using bbduk.
```bash
# Trim the adapters using the reference file adaptors.fa (provided by bbduk)
time bbduk.sh in=data/10142.1.149555.ATGTC.subset_500k.unmatched.fq.gz \
out=data/10142.1.149555.ATGTC.subset_500k.trimmed.fastq.gz \
ktrim=r \
k=23 \
mink=11 \
hdist=1 \
tbo=t \
ref=/software/apps/bbtools/gcc/64/37.02/resources/adapters.fa
```
Time to run: 20 seconds

Looking at the output
```
BBBDuk version 37.02
maskMiddle was disabled because useShortKmers=true
Initial:
Memory: max=27751m, free=26737m, used=1014m

Added 216529 kmers; time: 	0.367 seconds.
Memory: max=27751m, free=25289m, used=2462m

Input is being processed as paired
Started output streams:	0.183 seconds.
Processing time:   		20.347 seconds.

Input:                  	498914 reads 		74837100 bases.
KTrimmed:               	3251 reads (0.65%) 	263478 bases (0.35%)
Trimmed by overlap:     	1905 reads (0.38%) 	9896 bases (0.01%)
Total Removed:          	1526 reads (0.31%) 	273374 bases (0.37%)
Result:                 	497388 reads (99.69%) 	74563726 bases (99.63%)

Time:   			20.920 seconds.
Reads Processed:        498k 	23.85k reads/sec
Bases Processed:      74837k 	3.58m bases/sec
```
* 3251 reads had adaptors trimmed because the matched they matched adaptors in the database
* 1905 reads had a few base pairs trimmed because a small portion of an adaptor was detected when the read pairs were provisionally merged.
* 1526 reads were removed because after trimming they were too short.

At this point we could quality trim the right end of the reads but since we are going to merge them some of these low quality reads will likely be fixed so lets skip this for now.

```bash
# Merge the reads together
time bbmerge.sh in=data/10142.1.149555.ATGTC.subset_500k.trimmed.fastq.gz \
out=data/10142.1.149555.ATGTC.subset_500k.merged.fastq.gz \
outu=data/10142.1.149555.ATGTC.subset_500k.unmerged.fastq.gz \
ihist=data/insert_size.txt
usejni=t
```
Here's the output:
```
BBMerge version 37.02
Writing mergable reads merged.
Started output threads.
Total time: 13.311 seconds.

Pairs:               	248694
Joined:              	134307   	54.005%
Ambiguous:           	114162   	45.905%
No Solution:         	225       	0.090%
Too Short:           	0       	0.000%

Avg Insert:          	234.8
Standard Deviation:  	34.9
Mode:                	244

Insert range:        	121 - 291
90th percentile:     	279
75th percentile:     	263
50th percentile:     	239
25th percentile:     	211
10th percentile:     	186
```
From the summary we can see that our average insert size was 234bp and 54% of
reads could be joined. From the insert-size.txt histogram file we can plot the
data and see the shape of the distribution.

![Insert Size Histogram Plot](/Microbiome-workshop/assets/metagenome/insertsize.svg)


# Metagenome assembly

Most or our cleanup work is now done. We can assemble the metagenome with the Succinct DeBruijn graph assembler [Megahit](https://doi.org/10.1093/bioinformatics/btv033).

```bash
megahit  -m 20000000000 \
-t 4 \
--read data/10142.1.149555.ATGTC.subset_500k.merged.fastq.gz \
--12 data/10142.1.149555.ATGTC.subset_500k.unmerged.fastq.gz \
--k-list 21,41,61,81,99 \
--no-mercy \
--min-count 2 \
--out-dir data/megahit/
```
Time to run: 3 minutes

# Looking at the assembly

MEgahit tells us that the longest contig was  129514pb and the N50 was 6410 bp. we can use quast to look at the assembly more.

```bash
module load quast/gcc/64/3.1

quast.py -o data/quast -t 4 -f --meta data/megahit/final.contigs.fa
```

All statistics are based on contigs of size >= 500 bp, unless otherwise noted (e.g., "# contigs (>= 0 bp)" and "Total length (>= 0 bp)" include all contigs).

Assembly |                  final.contigs
---------|-----------------------------
\# contigs (>= 0 bp)  |      1325
\#contigs (>= 1000 bp)  |   993
Total length (>= 0 bp)   |  4995546
Total length (>= 1000 bp) | 4797970
\# contigs                |  1198
Largest contig           |  129514
Total length             |  4948570
GC (%)                   |  34.94
N50                      |  6441
N75                      |  3441
L50                      |  198
L75                      |  462
\# N's per 100 kbp        |  0.00

![Cumulative contig graph](/Microbiome-workshop/assets/metagenome/cumulative_plot.svg)

Although its not generally required, we can also visualize our assemblies by generating a fastg file

```bash
megahit_toolkit contig2fastg 99 \
data/megahit/intermediate_contigs/k99.contigs.fa > data/k99.fastg
```

Download the file to  your local computer by opening a new Terminal window and coping the file

```bash
scp user.name@scinet-login.bioteam.net:~/metagenome1/data/k99.fastg .
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

If we go back to the Ceres command line we can continue with the next common step in read based assembly, mapping. Two common types of mapping can  occur. you can map reads back to a known reference.  This is great if you have closely related organisms of interest and you are trying to understand population variance or quantify known organims.  A number of programs can be used  for mapping. Bowtie2 is the most common but in this case we will use bbmap.

Map reads back to known reference
```bash
bbmap.sh \
  ref=/project/microbiome_workshop/metagenome/viral/data/Ref_database.fna \
  in=data/10142.1.149555.ATGTC.subset_500k.fastq.trimmed.gz \
  out=data/10142.1.149555.ATGTC.map_to_genomes.sam \
  nodisk \
  usejni=t \
  bamscript=bamscript1.sh
```
The mapper writes Text based sam mapping files. It's usually helpful to convert
these files to a binary format called bam which is faster to access and smaller. Fortunately, bbmap will automatically make a little script to do that for us using the program Samtools. To run it enter this command.
```bash
./bamscript1.sh
```

It is also possible to map reads back to the contigs we just generated. Reads are mapped back to contigs for several reasons. Once genes are called gene counts can be derived from mapping data.  Contig mapping data is also used in genome binning.

Map reads back to known reference
```bash
bbmap.sh \
  ref=data/megahit/final.contigs.fa \
  in=data/10142.1.149555.ATGTC.subset_500k.fastq.trimmed.gz \
  out=data/10142.1.149555.ATGTC.map_to_contigs.sam \
  nodisk \
  usejni=t \
  bamscript=bamscript2.sh
```

The mapper writes Text based sam mapping files. It's usually helpful to convert
these files to a binary format called bam which is faster to access and smaller. Fortunately, bbmap will automatically make a little script to do that for us using the program Samtools. To run it enter this command.
```bash
./bamscript2.sh
```

Part one of the metagenome assembly tutorial deals with the most of the heavy computational work that requires computers with high memory and many processors. In real applications this kind of work would be submitted as a batch job using the SLURM scheduler so that it can run without your being logged into Ceres.

The next major of an Aseembly based workflow involves calling genes, annotating genes and binning contigs into genomes.  We will address this in Par2 of the metagenomics tutorial.
