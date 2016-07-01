---
title: "Counting reads"
author: "Meeta Mistry, Bob Freeman, Radhika Khetani"
date: "Tuesday, February 16, 2016"
---

Approximate time: 

## Learning Objectives:

* learn about read counting tools
* learn to generate a count matrix using featureCounts

## Counting reads as a measure of gene expression
<img src="../img/counts-workflow.png" width="400">

Once we have our reads aligned to the genome, the next step is to count how many reads have mapped to each gene. There are many tools that can use BAM files as input and output the number of reads (counts) associated with each feature of interest (genes, exons, transcripts, etc.). 2 commonly used counting tools are [featureCounts](http://bioinf.wehi.edu.au/featureCounts/) and [htseq-count](http://www-huber.embl.de/users/anders/HTSeq/doc/count.html). 

* The above tools only report the "raw" counts of reads that map to a single location (uniquely mapping) and are best at counting at the gene level. Essentially, total read count associated with a gene (*meta-feature*) = the sum of reads associated with each of the exons (*feature*) that "belong" to that gene.

* There are other tools available that are able to account for multiple transcripts for a given gene. In this case the counts are not whole numbers, but have fractions. In the simplest example case, if 1 read is associated with 2 transcripts, it can get counted as 0.5 and 0.5 and the resulting count for that transcript is not a whole number.

* In addition there are tools that will count multimapping reads, but this is a dangerous thing to do since you will be overcounting the total number of reads which can cause issues with normalization and eventually with accuracy of differential gene expression results. 

**Input for counting = multiple BAM files + 1 GTF file**

Simply speaking, the genomic coordinates of where the read is mapped (BAM) are cross-referenced with the genomic coordinates of whichever feature you are interested in counting expression of (GTF), it can be exons, genes or transcripts.

<img src="../img/count-fig2.png" width="600">

**Output of counting = A count matrix, with genes as rows and samples are columns**

These are the "raw" counts and will be used in statistical programs downstream for differential gene expression.

<img src="../img/count-matrix.png" width="300">

### Counting using featureCounts
Today, we will be using the [featureCounts](http://bioinf.wehi.edu.au/featureCounts/) tool to get the *gene* counts. We picked this tool because it is accurate, fast and is relatively easy to use. It counts reads that map to a single location (uniquely mapping) and follows the scheme in the figure below for assigning reads to a gene/exon. 

<img src="../img/union.png" width="300">

featureCounts can also take into account whether your data are **stranded** or not. If strandedness is specified, then in addition to considering the genomic coordinates it will also take the strand into account for counting. If your data are stranded always specify it.

#### Setting up to run featureCounts
First things first, start an interactive session with 4 cores:
	
	$ bsub -Is -n 4 -q interactive bash

Now, change directories to your rnaseq directory and start by creating 2 directories, (1) a directory for the output and (2) a directory for the bam files we generated yesterday:

	$ cd ~/ngs_course/rnaseq/
	$ mkdir results/counts results/STAR/bams
	
Let's move the bam files over to the `results/STAR/bams` directory
	
	$ mv ~/ngs_course/rnaseq/results/STAR/*fq_Aligned*bam ~/ngs_course/rnaseq/results/STAR/bams
	
	# check to make sure the move worked and that only the files we wanted moved over

featureCounts is not available as a module on Orchestra, but we can add the path for it to our `$PATH` variable. 

	$ export PATH=/opt/bcbio/centos/bin:$PATH

> Remember that this export command is going to "put featureCounts in your path" only for this interactive session.
>
> **Modify the export command in your `~/.bashrc` file to be `export PATH=/opt/bcbio/centos/bin:$PATH`.**


#### Running featureCounts

How do we use this tool, what is the command and what options/parameters are available to us?

	$ featureCounts

So, it looks like the usage is `featureCounts [options] -a <annotation_file> -o <output_file> input_file1 [input_file2] ... `, where `-a`, `-o` and input files are required.

We are going to use the following options:

`-T 4 # specify 4 cores`

`-s 2 # these data are "reverse"ly stranded`

and the following are the values for the required parameters:

`-a ~/ngs_course/rnaseq/data/reference_data/chr1-hg19_genes.gtf # required option for specifying path to GTF`

`-o ~/ngs_course/rnaseq/results/counts/Mov10_featurecounts.txt # required option for specifying path to, and name of the text output (count matrix)`

`~/ngs_course/rnaseq/results/STAR/bams/*bam # the list of all the bam files we want to collect count information for`

Let's run this now:

	$ featureCounts -T 4 -s 2 \ 
	  -a ~/ngs_course/rnaseq/data/reference_data/chr1-hg19_genes.gtf \
	  -o ~/ngs_course/rnaseq/results/counts/Mov10_featurecounts.txt \
	  ~/ngs_course/rnaseq/results/STAR/bams/*bam
	  
> If you wanted to collect the information that is on the screen as the job runs, you can modify the command and add the `2>` redirection at the end. This type of redirection will collect all the information from the terminal/screen into a file.

	**DO NOT RUN THIS** 
	# note the last line of the command below
	
	$ featureCounts -T 4 -s 2 \ 
	  -a ~/ngs_course/rnaseq/data/reference_data/chr1-hg19_genes.gtf \
	  -o ~/ngs_course/rnaseq/results/counts/Mov10_featurecounts.txt \
	  ~/ngs_course/rnaseq/results/STAR/bams/*bam \
	  2> /ngs_course/rnaseq/results/counts/Mov10_featurecounts.screen-output

#### featureCounts output

The output of this tool is 2 files, *a count matrix* and *a summary file* that tabulates how many the reads were "assigned" or counted and the reason they remained "unassigned". Let's take a look at the summary file:
	
	$ less results/counts/Mov10_featurecounts.txt.summary
	
Now let's look at the count matrix:
	
	$ less results/counts/Mov10_featurecounts.txt
	
There is information about the genomic coordinates and the length of the gene, we don't need this for the next step, so we are going to extract the columns that we are interested in.
	
	$ cut -f1,7,8,9,10,11,12 results/counts/Mov10_featurecounts.txt > results/counts/Mov10_featurecounts.Rmatrix.txt

The next step is to clean it up a little further by modifying the header line:
	
	$ vim results/counts/Mov10_featurecounts.Rmatrix.txt

### Note on counting PE data

For paired-end (PE) data, the bam file contains information about whether both read1 and read2 mapped and if they were at roughly the correct distance from each other, that is to say if they were "properly" paired. For most counting tools, only properly paired reads are considered by default, and each read pair is counted only once as a single "fragment". 

For counting PE fragments associated with genes, the input bam files need to be sorted by read name (i.e. alignment information about both read pairs in adjoining rows). The alignment tool might sort them for you, but watch out for how the sorting was done. If they are sorted by coordinates (like with STAR), you will need to use `samtools sort` to re-sort them by read name before using as input in featureCounts. If you do not sort you BAM file by read name before using as input, featureCounts assumes that almost all the reads are not properly paired.

### Keeping track of read numbers

It is important to keep track of how many reads you started with and how many of those ended up being associated with genes. This will help you pick out any obvious outliers, and it will also alert you early on about issues with contamination and so on.

The things to keep track of for each sample are the following:
* number of raw reads
* number of reads left after trimming
* number of reads aligned to genome
* number of reads associated with genes 
 
We have an [Excel template](https://dl.dropboxusercontent.com/u/74036176/rna-seq_reads_template.xlsx) available, to get you started.


