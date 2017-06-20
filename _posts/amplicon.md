---
title: "Amplicon worflow example using Dada2 and Phyloseq"
output: html_notebook
---

This is a first draft of an Amplicon sequencing turorual for a planned ARS Microbiome workshop. It is modified from the Dada2 tutorial created by Benjamin Callahan, the Author of Dada2 with permission. https://benjjneb.github.io/dada2/tutorial.html

modified by Adam Rivers

Here we walk through version 1.4 of the DADA2 pipeline on a small multi-sample dataset. Our starting point is a set of Illumina-sequenced paired-end fastq files that have been split (or “demultiplexed”) by sample and from which the barcodes/adapters have already been removed. The end product is a sequence variant (SV) table, a higher-resolution analogue of the ubiquitous “OTU table”, which records the number of times each ribosomal sequence variant (SV) was observed in each sample. We also assign taxonomy to the output sequences, and demonstrate how the data can be imported into the popular phyloseq R package for the analysis of microbiome data.

# Starting point
This workflow assumes that the data you are starting with meets certain criteria:

Non-biological nucleotides have been removed (primers/adapters/barcodes…)
Samples are demultiplexed (split into individual per-sample fastqs)
If paired-end sequencing, the forward and reverse fastqs contain reads in matched order
If these criteria are not true for your data (are you sure there aren’t any primers hanging around?) you need to remedy those issues before beginning this workflow. See the FAQ for some recommendations for common issues.

Getting ready
First we load the dada2 library. If you don’t already have the dada2 package, see the [dada2 installation instructions](https://benjjneb.github.io/dada2/dada-installation.html)


{% highlight r %}
library(dada2); packageVersion("dada2")
{% endhighlight %}



{% highlight text %}
## [1] '1.4.0'
{% endhighlight %}
If your dada2 version shouldbe 1.4 or higher.

The data we will work with are the same as those in the [Mothur Miseq SOP](http://www.mothur.org/wiki/MiSeq_SOP) walkthrough. Download the [example data](http://www.mothur.org/w/images/d/d6/MiSeqSOPData.zip) and unzip. These files represent longitudinal samples from a mouse post-weaning and one mock community control. For now just consider them paired-end fastq files to be processed. Define the following path variable so that it points to the extracted directory on your machine:

This example uses data from 

{% highlight r %}
library("dada2")
path <- "/Users/rivers/Documents/Microbiome workshop/Amplicon tutorial/MiSeq_SOP"
list.files(path)
{% endhighlight %}



{% highlight text %}
## character(0)
{% endhighlight %}

If the package successfully loaded and your listed files match those here, you are ready to go through the DADA2 pipeline.

#Filter and Trim

{% highlight r %}
# Sort ensures forward/reverse reads are in same order
fnFs <- sort(list.files(path, pattern="_R1_001.fastq"))
fnRs <- sort(list.files(path, pattern="_R2_001.fastq"))
# Extract sample names, assuming filenames have format: SAMPLENAME_XXX.fastq
sample.names <- sapply(strsplit(fnFs, "_"), `[`, 1)
# Specify the full path to the fnFs and fnRs
fnFs <- file.path(path, fnFs)
fnRs <- file.path(path, fnRs)
{% endhighlight %}

If using this workflow on your own data: The string manipulations may have to be modified, especially the extraction of sample names from the file names.

Examine quality profiles of forward and reverse reads
It is important to look at your data. We start by visualizing the quality profiles of the forward reads:



{% highlight r %}
plotQualityProfile(fnFs[1:2])
{% endhighlight %}



{% highlight text %}
## Error: Input/Output
##   no input files found
##   dirPath: NA
##   pattern: character(0)
{% endhighlight %}

 The forward reads are good quality. We generally advise trimming the last few nucleotides to avoid less well-controlled errors that can arise there. There is no suggestion from these quality profiles that any additional trimming is needed, so we will truncate the forward reads at position 240 (trimming the last 10 nucleotides).

Now we visualize the quality profile of the reverse reads:


{% highlight r %}
plotQualityProfile(fnRs[1:2])
{% endhighlight %}



{% highlight text %}
## Error: Input/Output
##   no input files found
##   dirPath: NA
##   pattern: character(0)
{% endhighlight %}
The reverse reads are significantly worse quality, especially at the end, which is common in Illumina sequencing. This isn’t too worrisome, DADA2 incorporates quality information into its error model which makes the algorithm robust to lower quality sequence, but trimming as the average qualities crash is still a good idea as long as our reads will still overlap. We will truncate at position 160 where the quality distribution crashes.

If using this workflow on your own data: Your reads must overlap after truncation in order to merge them later!!! The tutorial is using 2x250 V4 sequence data, so the forward and reverse reads almost completely overlap and our trimming can be completely guided by the quality scores. If you are using a less-overlapping primer set, like V1-V2 or V3-V4, your truncLen must be large enough to maintain the overlap between them (the more the better).

# Perform filtering and trimming
We define the filenames for the filtered fastq.gz files:


{% highlight r %}
filt_path <- file.path(path, "filtered") # Place filtered files in filtered/ subdirectory
filtFs <- file.path(filt_path, paste0(sample.names, "_F_filt.fastq.gz"))
filtRs <- file.path(filt_path, paste0(sample.names, "_R_filt.fastq.gz"))
{% endhighlight %}
We’ll use standard filtering parameters: maxN=0 (DADA2 requires no Ns), truncQ=2, rm.phix=TRUE and maxEE=2. The maxEE parameter sets the maximum number of “expected errors” allowed in a read, which is a better filter than simply averaging quality scores.

Filter the forward and reverse reads:


{% highlight r %}
out <- filterAndTrim(fnFs, filtFs, fnRs, filtRs, truncLen=c(240,160),
              maxN=0, maxEE=c(2,2), truncQ=2, rm.phix=TRUE,
              compress=TRUE, multithread=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in filterAndTrim(fnFs, filtFs, fnRs, filtRs, truncLen = c(240, 160), : Every input file must have a corresponding output file.
{% endhighlight %}



{% highlight r %}
head(out)
{% endhighlight %}



{% highlight text %}
## Error in head(out): object 'out' not found
{% endhighlight %}
If using this workflow on your own data: The standard filtering parameters are starting points, not set in stone. For example, if too few reads are passing the filter, considering relaxing maxEE, perhaps especially on the reverse reads (eg. maxEE=c(2,5)). If you want to speed up downstream computation, consider tightening maxEE. For pair-end reads consider the length of your amplicon when choosing truncLen as your reads must overlap after truncation in order to merge them later!!!

If using this workflow on your own data: For common ITS amplicon strategies, it is undesirable to truncate reads to a fixed length due to the large amount of length variation at that locus. That is OK, just leave out truncLen. Make sure you removed the forward and reverse primers from both the forward and reverse reads though!

 # Learn the Error Rates
The DADA2 algorithm depends on a parametric error model (err) and every amplicon dataset has a different set of error rates. The  learnErrors method learns the error model from the data, by alternating estimation of the error rates and inference of sample composition until they converge on a jointly consistent solution. As in many optimization problems, the algorithm must begin with an initial guess, for which the maximum possible error rates in this data are used (the error rates if only the most abundant sequence is correct and all the rest are errors).

The following runs in about 1.5 minutes on a 2016 Macbook Pro:


{% highlight r %}
# Start the clock
ptm <- proc.time()
# Learn error rates
errF <- learnErrors(filtFs, multithread=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in open.connection(con, "rb"): cannot open the connection
{% endhighlight %}



{% highlight r %}
# Stop the clock
proc.time() - ptm
{% endhighlight %}



{% highlight text %}
##    user  system elapsed 
##   0.003   0.000   0.003
{% endhighlight %}

{% highlight r %}
# Start the clock
ptm <- proc.time()
# Learn error rates
errR <- learnErrors(filtRs, multithread=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in open.connection(con, "rb"): cannot open the connection
{% endhighlight %}



{% highlight r %}
# Stop the clock
proc.time() - ptm
{% endhighlight %}



{% highlight text %}
##    user  system elapsed 
##   0.052   0.000   0.052
{% endhighlight %}
It is always worthwhile, as a sanity check if nothing else, to visualize the estimated error rates:

{% highlight r %}
plotErrors(errF, nominalQ=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in is(obj, "matrix"): object 'errF' not found
{% endhighlight %}
The error rates for each possible transition (eg. A->C, A->G, …) are shown. Points are the observed error rates for each consensus quality score. The black line shows the estimated error rates after convergence. The red line shows the error rates expected under the nominal definition of the Q-value. Here the black line (the estimated rates) fits the observed rates well, and the error rates drop with increased quality as expected. Everything looks reasonable and we proceed with confidence.

If using this workflow on your own data: Parameter learning is computationally intensive, so by default the learnErrors function uses only a subset of the data (the first 1M reads). If the plotted error model does not look like a good fit, try increasing the nreads parameter to see if the fit improves.

# Dereplication
Dereplication combines all identical sequencing reads into into “unique sequences” with a corresponding “abundance”: the number of reads with that unique sequence. Dereplication substantially reduces computation time by eliminating redundant comparisons.

Dereplication in the DADA2 pipeline has one crucial addition from other pipelines: DADA2 retains a summary of the quality information associated with each unique sequence. The consensus quality profile of a unique sequence is the average of the positional qualities from the dereplicated reads. These quality profiles inform the error model of the subsequent denoising step, significantly increasing DADA2’s accuracy.

Dereplicate the filtered fastq files:


{% highlight r %}
derepFs <- derepFastq(filtFs, verbose=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in open.connection(con, "rb"): cannot open the connection
{% endhighlight %}



{% highlight r %}
derepRs <- derepFastq(filtRs, verbose=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in open.connection(con, "rb"): cannot open the connection
{% endhighlight %}



{% highlight r %}
# Name the derep-class objects by the sample names
names(derepFs) <- sample.names
{% endhighlight %}



{% highlight text %}
## Error in names(derepFs) <- sample.names: object 'derepFs' not found
{% endhighlight %}



{% highlight r %}
names(derepRs) <- sample.names
{% endhighlight %}



{% highlight text %}
## Error in names(derepRs) <- sample.names: object 'derepRs' not found
{% endhighlight %}
If using this workflow on your own data: The tutorial dataset is small enough to easily load into memory. If your dataset exceeds available RAM, it is preferable to process samples one-by-one in a streaming fashion: see the DADA2 Workflow on Big Data for an example.

# Sample Inference
We are now ready to apply the core sequence-variant inference algorithm to the dereplicated data.

Infer the sequence variants in each sample:

{% highlight r %}
system.time(dadaFs <- dada(derepFs, err=errF, multithread=TRUE))
{% endhighlight %}



{% highlight text %}
## Error in dada(derepFs, err = errF, multithread = TRUE): object 'derepFs' not found
{% endhighlight %}

{% highlight r %}
dadaRs <- dada(derepRs, err=errR, multithread=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in dada(derepRs, err = errR, multithread = TRUE): object 'derepRs' not found
{% endhighlight %}
Inspecting the dada-class object returned by dada:


{% highlight r %}
dadaFs[[1]]
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'dadaFs' not found
{% endhighlight %}

The DADA2 algorithm inferred 128 real variants from the 1979 unique sequences in the first sample. There is much more to the  dada-class return object than this (see help("dada-class") for some info), including multiple diagnostics about the quality of each inferred sequence variant, but that is beyond the scope of an introductory tutorial.

If using this workflow on your own data: All samples are simultaneously loaded into memory in the tutorial. If you are dealing with datasets that approach or exceed available RAM, it is preferable to process samples one-by-one in a streaming fashion: see the DADA2 Workflow on Big Data for an example.

If using this workflow on your own data: By default, the dada function processes each sample independently, but pooled processing is available with pool=TRUE and that may give better results for low sampling depths at the cost of increased computation time. See our discussion about pooling samples for sample inference.

# Merge paired reads
Spurious sequence variants are further reduced by merging overlapping reads. The core function here is mergePairs, which depends on the forward and reverse reads being in matching order at the time they were dereplicated.

Merge the denoised forward and reverse reads:
 

{% highlight r %}
mergers <- mergePairs(dadaFs, derepFs, dadaRs, derepRs, verbose=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in is(dadaF, "dada"): object 'dadaFs' not found
{% endhighlight %}



{% highlight r %}
# Inspect the merger data.frame from the first sample
head(mergers[[1]])
{% endhighlight %}



{% highlight text %}
## Error in head(mergers[[1]]): object 'mergers' not found
{% endhighlight %}

We now have a data.frame for each sample with the merged $sequence, its $abundance, and the indices of the merged $forward and  $reverse denoised sequences. Paired reads that did not exactly overlap were removed by mergePairs.

If using this workflow on your own data: Most of your reads should successfully merge. If that is not the case upstream parameters may need to be revisited: Did you trim away the overlap between your reads?

 
Construct sequence table
We can now construct a “sequence table” of our mouse samples, a higher-resolution version of the “OTU table” produced by classical methods:

{% highlight r %}
seqtab <- makeSequenceTable(mergers)
{% endhighlight %}



{% highlight text %}
## Error in class(samples) %in% c("dada", "derep", "data.frame"): object 'mergers' not found
{% endhighlight %}

{% highlight r %}
dim(seqtab)
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'seqtab' not found
{% endhighlight %}


{% highlight r %}
# Inspect distribution of sequence lengths
table(nchar(getSequences(seqtab)))
{% endhighlight %}



{% highlight text %}
## Error in is(object, "character"): object 'seqtab' not found
{% endhighlight %}



{% highlight r %}
hist(nchar(getSequences(seqtab)), main="Distribution of sequence lengths")
{% endhighlight %}



{% highlight text %}
## Error in is(object, "character"): object 'seqtab' not found
{% endhighlight %}
The sequence table is a matrix with rows corresponding to (and named by) the samples, and columns corresponding to (and named by) the sequence variants. The lengths of our merged sequences all fall within the expected range for this V4 amplicon.

If using this workflow on your own data: Sequences that are much longer or shorter than expected may be the result of non-specific priming, and may be worth removing (eg. seqtab2 <- seqtab[,nchar(colnames(seqtab)) %in% seq(250,256)]). This is analogous to “cutting a band” in-silico to get amplicons of the targeted length.

 

# Remove chimeras
The core dada method removes substitution and indel errors, but chimeras remain. Fortunately, the accuracy of the sequences after denoising makes identifying chimeras simpler than it is when dealing with fuzzy OTUs: all sequences which can be exactly reconstructed as a bimera (two-parent chimera) from more abundant sequences.


{% highlight r %}
seqtab.nochim <- removeBimeraDenovo(seqtab, method="consensus", multithread=TRUE, verbose=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in removeBimeraDenovo(seqtab, method = "consensus", multithread = TRUE, : object 'seqtab' not found
{% endhighlight %}



{% highlight r %}
dim(seqtab.nochim)
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'seqtab.nochim' not found
{% endhighlight %}

{% highlight r %}
sum(seqtab.nochim)/sum(seqtab)
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'seqtab.nochim' not found
{% endhighlight %}

The fraction of chimeras varies based on factors including experimental procedures and sample complexity, but can be substantial. Here chimeras make up about 20% of the inferred sequence variants, but those variants account for only about 4% of the total sequence reads.

If using this workflow on your own data: Most of your reads should remain after chimera removal (it is not uncommon for a majority of sequence variants to be removed though). If most of your reads were removed as chimeric, upstream processing may need to be revisited. In almost all cases this is caused by primer sequences with ambiguous nucleotides that were not removed prior to beginning the DADA2 pipeline.

# Track reads through the pipeline
As a final check of our progress, we’ll look at the number of reads that made it through each step in the pipeline:

{% highlight r %}
getN <- function(x) sum(getUniques(x))
track <- cbind(out, sapply(dadaFs, getN), sapply(mergers, getN), rowSums(seqtab), rowSums(seqtab.nochim))
{% endhighlight %}



{% highlight text %}
## Error in cbind(out, sapply(dadaFs, getN), sapply(mergers, getN), rowSums(seqtab), : object 'out' not found
{% endhighlight %}



{% highlight r %}
colnames(track) <- c("input", "filtered", "denoised", "merged", "tabled", "nonchim")
{% endhighlight %}



{% highlight text %}
## Error in colnames(track) <- c("input", "filtered", "denoised", "merged", : object 'track' not found
{% endhighlight %}



{% highlight r %}
rownames(track) <- sample.names
{% endhighlight %}



{% highlight text %}
## Error in rownames(track) <- sample.names: object 'track' not found
{% endhighlight %}



{% highlight r %}
head(track)
{% endhighlight %}



{% highlight text %}
## Error in head(track): object 'track' not found
{% endhighlight %}
Looks good, we kept the majority of our raw reads, and there is no over-large drop associated with any single step.

If using this workflow on your own data: This is a great place to do a last sanity check. Outside of filtering (depending on how stringent you want to be) there should no step in which a majority of reads are lost. If a majority of reads failed to merge, you may need to revisit the truncLen parameter used in the filtering step and make sure that the truncated reads span your amplicon. If a majority of reads failed to pass the chimera check, you may need to revisit the removal of primers, as the ambiguous nucleotides in unremoved primers interfere with chimera identification.

# Assign taxonomy
It is common at this point, especially in 16S/18S/ITS amplicon sequencing, to classify sequence variants taxonomically. The DADA2 package provides a native implementation of the RDP’s naive Bayesian classifier for this purpose. The assignTaxonomy function takes a set of sequences and a training set of taxonomically classified sequences, and outputs the taxonomic assignments with at least minBoot bootstrap confidence.

Appropriately formatted training fastas for the RDP training set 14, the GreenGenes 13.8 release clustered at 97% identity, the Silva reference database v123 (Silva dual license), and the UNITE ITS database (use the General Fasta release files) are available. To follow along, download the silva_nr_v123_train_set.fa.gz file, and place it in the directory with the fastq files.

The following databases are available:
* Maintained:
  * GreenGenes version 13.8
  * RDP version 14
  * Silva version 123 (Silva dual-license)
  * UNITE (General Fasta releases) (version 1.3.3 or later of the dada2 package)
* Contributed:
  * HitDB version 1 (Human InTestinal 16S rRNA)

Note that currently species-assignment training fastas are only available for the Silva and RDP databases. In addition to thanking the folks at RDP, Silva and GreenGenes for making these datasets available, we also want to thank Pat Schloss and the mothur team for making cleaner versions of the Silva and RDP training set available. To be specific, we created the dada2-compatible training fastas from the mothur-compatible Silva.nr_v123 files (described here, and license here), the mothur-compatible 16S rRNA reference (RDP) (described here), and the GreenGenes 13.8 OTUs clustered at 97%.

##Formatting custom databases
Custom databases can be used as well, provided they can be converted to the dada2-compatible training fasta format.

The assignTaxonomy(...) function expects the training data to be provided in the form of a fasta file (or compressed fasta file) in which the taxonomy corresponding to each sequence is encoded in the id line in the following fashion (the second sequence is not assigned down to level 6):

{% highlight r %}
taxtrain <- "/Users/rivers/Documents/Microbiome workshop/Amplicon tutorial/silva_nr_v123_train_set.fa.gz"
taxa <- assignTaxonomy(seqtab.nochim, taxtrain, multithread=TRUE)
{% endhighlight %}



{% highlight text %}
## Error in is(object, "character"): object 'seqtab.nochim' not found
{% endhighlight %}



{% highlight r %}
unname(head(taxa))
{% endhighlight %}



{% highlight text %}
## Error in head(taxa): object 'taxa' not found
{% endhighlight %}

Okay you've done it. You've sequenced, cleaned, clustered, removed chimeras and identified the microbial sequences in your sample.  Now it's time to begin making sense of that data.


# Handoff to phyloseq
The DADA2 pipeline produced a sequence table and a taxonomy table which is appropriate for further analysis in phyloseq. We’ll also include the small amount of metadata we have – the samples are named by the gender (G), mouse subject number (X) and the day post-weaning (Y) it was sampled (eg. GXDY).

## Import into phyloseq:


{% highlight r %}
library(phyloseq); packageVersion("phyloseq")
{% endhighlight %}



{% highlight text %}
## [1] '1.20.0'
{% endhighlight %}



{% highlight r %}
library(ggplot2); packageVersion("ggplot2")
{% endhighlight %}



{% highlight text %}
## [1] '2.2.1'
{% endhighlight %}


{% highlight r %}
# Make a data.frame holding the sample data
samples.out <- rownames(seqtab.nochim)
{% endhighlight %}



{% highlight text %}
## Error in rownames(seqtab.nochim): object 'seqtab.nochim' not found
{% endhighlight %}



{% highlight r %}
subject <- sapply(strsplit(samples.out, "D"), `[`, 1)
{% endhighlight %}



{% highlight text %}
## Error in strsplit(samples.out, "D"): object 'samples.out' not found
{% endhighlight %}



{% highlight r %}
gender <- substr(subject,1,1)
{% endhighlight %}



{% highlight text %}
## Error in substr(subject, 1, 1): object 'subject' not found
{% endhighlight %}



{% highlight r %}
subject <- substr(subject,2,999)
{% endhighlight %}



{% highlight text %}
## Error in substr(subject, 2, 999): object 'subject' not found
{% endhighlight %}



{% highlight r %}
day <- as.integer(sapply(strsplit(samples.out, "D"), `[`, 2))
{% endhighlight %}



{% highlight text %}
## Error in strsplit(samples.out, "D"): object 'samples.out' not found
{% endhighlight %}



{% highlight r %}
samdf <- data.frame(Subject=subject, Gender=gender, Day=day)
{% endhighlight %}



{% highlight text %}
## Error in data.frame(Subject = subject, Gender = gender, Day = day): object 'subject' not found
{% endhighlight %}



{% highlight r %}
samdf$When <- "Early"
{% endhighlight %}



{% highlight text %}
## Error in samdf$When <- "Early": object 'samdf' not found
{% endhighlight %}



{% highlight r %}
samdf$When[samdf$Day>100] <- "Late"
{% endhighlight %}



{% highlight text %}
## Error in samdf$When[samdf$Day > 100] <- "Late": object 'samdf' not found
{% endhighlight %}



{% highlight r %}
rownames(samdf) <- samples.out
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'samples.out' not found
{% endhighlight %}



{% highlight r %}
# Construct phyloseq object (straightforward from dada2 outputs)
ps <- phyloseq(otu_table(seqtab.nochim, taxa_are_rows=FALSE), 
               sample_data(samdf), 
               tax_table(taxa))
{% endhighlight %}



{% highlight text %}
## Error in otu_table(seqtab.nochim, taxa_are_rows = FALSE): object 'seqtab.nochim' not found
{% endhighlight %}



{% highlight r %}
ps <- prune_samples(sample_names(ps) != "Mock", ps) # Remove mock sample
{% endhighlight %}



{% highlight text %}
## Error in sample_names(ps): object 'ps' not found
{% endhighlight %}



{% highlight r %}
ps
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'ps' not found
{% endhighlight %}

# Plot the species richness

{% highlight r %}
plot_richness(ps, x="Day", measures=c("Shannon", "Simpson"), color="When") + theme_bw()
{% endhighlight %}



{% highlight text %}
## Error in otu_table(physeq): object 'ps' not found
{% endhighlight %}

No obvious systematic difference in alpha-diversity between early and late samples.

# Create ordination plots

{% highlight r %}
ord.nmds.bray <- ordinate(ps, method="NMDS", distance="bray")
{% endhighlight %}



{% highlight text %}
## Error in inherits(physeq, "formula"): object 'ps' not found
{% endhighlight %}

{% highlight r %}
plot_ordination(ps, ord.nmds.bray, color="When", title="Bray NMDS")
{% endhighlight %}



{% highlight text %}
## Error in inherits(physeq, "phyloseq"): object 'ps' not found
{% endhighlight %}
Ordination picks out a clear separation between the early and late samples.

#Bar plot


{% highlight r %}
top20 <- names(sort(taxa_sums(ps), decreasing=TRUE))[1:20]
{% endhighlight %}



{% highlight text %}
## Error in otu_table(x): object 'ps' not found
{% endhighlight %}



{% highlight r %}
ps.top20 <- transform_sample_counts(ps, function(OTU) OTU/sum(OTU))
{% endhighlight %}



{% highlight text %}
## Error in taxa_are_rows(physeq): object 'ps' not found
{% endhighlight %}



{% highlight r %}
ps.top20 <- prune_taxa(top20, ps.top20)
{% endhighlight %}



{% highlight text %}
## Error in prune_taxa(top20, ps.top20): object 'top20' not found
{% endhighlight %}



{% highlight r %}
plot_bar(ps.top20, x="Day", fill="Family") + facet_wrap(~When, scales="free_x")
{% endhighlight %}



{% highlight text %}
## Error in inherits(physeq, "phyloseq"): object 'ps.top20' not found
{% endhighlight %}

Nothing glaringly obvious jumps out from the taxonomic distribution of the top 20 sequences to explain the early-late differentiation.

# Phylogenetic trees of amplicon sequences
It is common to create a phylogenetic tree of the taxa and then use metrics like UNIFRAC distance or just plot datain a phylogentic context.

That can be done in phyloseq too. 

## Align the sequnces

{% highlight r %}
library("msa")
seqs <- getSequences(seqtab.nochim)
{% endhighlight %}



{% highlight text %}
## Error in is(object, "character"): object 'seqtab.nochim' not found
{% endhighlight %}



{% highlight r %}
names(seqs) <- seqs # This propagates to the tip labels of the tree
{% endhighlight %}



{% highlight text %}
## Error in eval(expr, envir, enclos): object 'seqs' not found
{% endhighlight %}



{% highlight r %}
mult <- msa(seqs, method="ClustalW", type="dna", order="input")
{% endhighlight %}



{% highlight text %}
## Error in is(inputSeq, "character"): object 'seqs' not found
{% endhighlight %}

#DODO
* Make tree
* Calculate unifrac with the tree
* place data on tree
see: https://bioconductor.org/packages/release/bioc/vignettes/phyloseq/inst/doc/phyloseq-analysis.html
https://f1000research.com/articles/5-1492/v2
* DESEQ2
* SPIEC-EASI networks
