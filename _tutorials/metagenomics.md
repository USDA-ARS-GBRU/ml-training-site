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
Shotgun metagenomics data can be analyzed in several different approaches. Each approach is best suited for a particular group of questions. The methodological approaches can be broken down into three broad areas: read-based approaches, Assembly-based approaches and detection-based approaches.

Method | Read-based | Assembly-based | Detection-based
-------|------------|----------------|----------------
Description | Read-based metagenomics analyzes unassembled reads. The first methods to be used. still valuable for quantitative analysis especially if relevant references are available. | Assembly-Based workflows attempt to assemble the reads from one or more samples, "bin" contigs from these samples into genomes then analyze the genes and contigs. | Detection-based workflows attempt to identify with very high-precision but lower sensitivity (recall) the presence of organisms of interest, often pathogens.
Typical workflow | <ol><li>Read QC</li><li>Read merging</li><li>mapping to NR for taxonomy data  (Blast, Diamond or Last). Kmer-based approaches are also used (Kraken, Clarke).</li><li>Mapping to functional databases (Kegg, pfam). Cross-mapping can also be used (Mocat2)</li><li>Summaries of taxonomic and functional distributions can be analyzed. If ids were retained function within taxa analyses can be run.</li></ol> | |
Typical questions|
Examples | [MG-Rast](http://metagenomics.anl.gov/) | [IMG from JGI](https://img.jgi.doe.gov/cgi-bin/mer/main.cgi) | [Taxonomer](http://taxonomer.iobio.io/), [One Codex](https://www.onecodex.com/), [CosmosID](http://www.cosmosid.com/)
