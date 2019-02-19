**_Advanced Data Analysis - Introduction to NGS data analysis_**<br>
*Department of Animal and Plant Sciences, University of Sheffield*

# Differential gene expression analyses
#### Alison Wright, Nicola Nadeau, Victor Soria-Carrasco

The aim of this practical is to learn how to perform differential gene expression analyses. We will be using a dataset of expression data for 4 individuals of *Heliconius melpomene*. For each individual, two different wing regions have been sequenced. We will try to identify genes that are differentially expressed between wing regions. Samples are labelled I or A. I is the part of the wing that is iridescent, A is the top part of the wing, which is called the androchonial region.

## Table of contents 

1. Obtaining expression values

2. Introduction to edgeR

3. Filter expression data

4. Normalisation of gene expression

5. Visualisation of gene expression

6. Identify differentially expressed genes

---

## Initial set up
First of all, this tutorial must be run using an interactive session in ShARC. You will also submit jobs to ShARC. For that, you should log in into ShARC with `ssh`, and then request an interactive session with `qrsh`. Your shell prompt should show `sharc-nodeXXX` (XXX being a number between 001 and 172) and not `@sharc-login1` nor `@sharc-login2`.

For this particular tutorial, we are going to create and work on a directory called `DE` in your /fastdata/$USER directory:

        cd /fastdata/$USER
        mkdir DE
        cd DE

Remember you can check your current working directory anytime with the command `pwd`.
It should show something like:

        pwd
        /fastdata/$USER/DE

---

## 1.Obtaining expression values

Discussion of different measures of gene expression: RPKM,FPKM,CPM. Need read counts to calculate these.

**Cufflinks output files: isoforms.fpkm_tracking & genes.fpkm_tracking**
Cufflinks generates files that contain the estimated isoform or gene-level expression values as Fragments Per Kilobase of exon model per Million mapped fragments (FPKM). It is possible to use this FPKM information for an initial assessment of gene expression but expression analyses may require further processing of this data (eg normalisation). Therefore, we do not recommend conducting analyses on these raw FPKM values.

Instead, we can use [HTSeq](https://htseq.readthedocs.io/en/release_0.11.1/) to extract read counts for each gene for downstream analyses.

* **Sort BAM files with [Samtools](http://www.htslib.org/doc/samtools-1.0.html).**

HTSeq requires BAM files to be sorted either by read name or by alignment position. 

        >samtools sort file.bam - o file.sorted.bam

The default sort is by coordinate. However, you can use the sort order (SO) flag in the BAM header to check if the file has been sorted. 

        >samtools view -H file.bam | head

* **Extract read counts.**

Use [htSeq-count](https://htseq.readthedocs.io/en/release_0.11.1/count.html) to extract read counts for each transcript.

        >htseq-count -f BAM/SAM file.bam merged.gtf -r order > file.htseq

**-r** For paired-end data, the alignment have to be sorted either by read name or by alignment position. If your data is not sorted, use the samtools sort function of samtools to sort it. Use this option, with name or pos for <order> to indicate how the input data has been sorted. The default is name.
    
**-s <yes/no/reverse>** Specify whether the data is from a strand-specific assay (default: yes)
    
htseq-count outputs a table with counts for each feature, followed by the special counters, which count reads that were not counted for any feature for various reasons. 

## a. PRACTICAL ACTIVITY

Extract read counts with htseq-count(https://htseq.readthedocs.io/en/release_0.11.1/count.html) for each gene.

* Htseq-count takes a couple of hours to run, so lets just focus on one individual (96I). Check that the BAM file for this individual is already located in your fastdata folder.

        cd /fastdata/$USER/align/Quality_assessment
        
        ls 
	
* Check that the BAM file is sorted, and if so whether it is sorted by read name or by alignment position. You need to know this for htseq-count.
        
        qrsh
	
        source /usr/local/extras/Genomics/.bashrc

        samtools view -H 96I.bam | head

* Use htseq-count to extract read counts. You can run this in interactive mode. Remember to specify with -r whether the BAM file is sorted by read name (`name`) or alignnment position (`pos`). This command may take a while to finish so move onto the next session.
        
        source activate htseq

        htseq-count -f bam 96I.bam /fastdata/$USER/align/Cufflinks_output/merged_asm/merged_allstranded.gtf -r [name/pos] > 96I.htseq 

* It is not feasible to run htseq-count on all our samples in the practical due to time contains. We have already generated read count files for all the samples. There should be 8 files, one for each sample.
	
        /usr/local/extras/Genomics/workshops/NGS_AdvSta_2019/NGS_data/htseqCounts
        
* Copy the read count folder into your fastdata folder.

        cp -r /usr/local/extras/Genomics/workshops/NGS_AdvSta_2019/NGS_data/htseqCounts /fastdata/$USER/DE/htseqCounts

        cd /fastdata/$USER/DE
        
* We need to merge the read counts across samples into one file. You can do this using a custom python script. 

        python /usr/local/extras/Genomics/workshops/NGS_AdvSta_2019/NGS_data/scripts/merge_counts.py /fastdata/$USER/DE/htseqCounts/
	
* How many genes are there? You can count this with Unix (`wc -l`) and minus one for the header. Remember Htseq-count calculates read counts for each gene by merging across transcripts.

        wc -l htseqCounts/htseq.merged.txt
        
* The downstream analyses we will perform next in R on the desktop. Copy the merged read count file onto your desktop. You need to open a new terminal window to do this.

        cd Desktop

        scp $USER@sharc.shef.ac.uk:/fastdata/$USER/DE/htseqCounts/htseq.merged.txt .

---

## 2.Introduction to edgeR

We will use [edgeR](https://bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf) to perform differential gene expression analyses. This is implemented in R. 

We have read count data for 4 individuals of *Heliconius melpomene*. For each individual, two different wing regions have been sequenced. We will try to identify genes that are differentially expressed between wing regions. Samples are labelled I or A. I is the part of the wing that is iridescent, A is the top part of the wing, which is called the androchonial region.

![alt text](https://github.com/alielw/APS-NGS-day2-PM/blob/master/Sample_id.png)

## b. PRACTICAL ACTIVITY

* Lets read in the data to R.

        library(edgeR)

        data <- read.table("~/Desktop/htseq.merged.txt",stringsAsFactors=F,header=T, row.names=1)

        names(data)

        dim(data)

* Lets load edgeR. You may need to install the package. Instructions are [here](https://bioconductor.org/packages/release/bioc/html/edgeR.html)

        library(edgeR)

* We have two treatments or conditions: A and I. We need to find this information from the file header and specify it in R.

        conditions <- factor(c("I","A","I","I","A","A","A","I"))

* [edgeR](https://bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf) stores data in a simple list-based data object called a `DGEList`. This type of object is easy to use because it can be manipulated like any list in R. The function readDGE makes a DGEList object directly. If the table of counts is already available as a matrix or a data.frame, x say, then a DGEList object can be made by:

        expr <- DGEList(counts=data)

* We can add conditions at the same time:

        expr <- DGEList(counts=data, group=conditions)

        expr$samples

---

## 3.Filter expression data

Genes with very low counts across all samples provide little evidence for differential expression. In the biological point of view, a gene must be expressed at some minimal level before it is likely to be translated into a protein or to be biologically important. Therefore, we need to filter genes with biologically irrelevant expression. 

The developers of edgeR recommend that gene is required to have a count of 5-10 in a library to be considered expressed in that library. However, users should filter with count-per-million (`CPM`) rather than filtering on the read counts directly, as the latter does not account for differences in library sizes between samples. Therefore, they recommend filtering on a CPM of 1.

We can calculate count-per-million (`CPM`) using cpm(`DGEList`).

        cpm_data <- cpm(expr)

## c. PRACTICAL ACTIVITY

* Filter expression to remove lowly expressed genes. We do not want to exclude genes specific to one wing region (ie I or A-limited genes). Therefore, a sensible filtering approach is to filter genes that are not expressed > 1 CPM in at least half of the samples. We have 8 samples.

        keep <- rowSums(cpm(expr)>1) >=4
        expr_filtered <- expr[keep,,keep.lib.sizes=FALSE]

* How many genes remain? Does this seem a sensible number of expressed genes? There are 12,669 annotated genes in the *Heliconius melpomene* reference. 

        dim(expr_filtered)

## 4. Normalisation of gene expression

We need to normalise gene expression across samples before conducting differential gene expression analyses. There are a number of sources of variation we need to account for.

* **Sequencing depth**
The most obvious technical factor that affects the read counts, other than gene expression levels, is the sequencing depth of each sample. edgeR controls for varying sequencing depths as represented by differing library sizes. This is part of the basic modeling procedure and flows automatically into downstream statistical analyses. It doesn’t require any user intervention.

* **RNA composition**
The second most important technical influence on differential expression is one that is RNA composition. RNA-seq provides a measure of the relative abundance of each gene in each RNA sample, but does not provide any measure of the total RNA output on a per-cell basis. This commonly becomes important when a small number of genes are very highly expressed in one sample, but not in another. The highly expressed genes can consume a substantial proportion of the total library size, causing the remaining genes to be under-sampled in that sample. Unless this RNA composition effect is adjusted for, the remaining genes may falsely appear to be down-regulated in that sample. This is normally a problem for cross-species comparisons. 

Variation in RNA composition can be accounted for by the calcNormFactors function. This normalizes for RNA composition by finding a set of scaling factors for the library sizes that minimize the log-fold changes between the samples for most genes. The default method for computing these scale factors uses a trimmed mean of M- values (TMM) between each pair of samples. Further information can be found in the [edgeR manual](https://bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf) 

	expr_norm = calcNormFactors(expr_filtered)

---

## 5. Visualisation of gene expression

There are a number of ways to visualise gene expression data. Heatmaps and PCA plots are widely used approaches. We will cover heatmaps today as Day 3 will explore PCA plots.

## d. PRACTICAL ACTIVITY

* Calculate log CPM

        cpm_log <- cpm(expr_filtered, log = TRUE)
	
* Perform clustering of expression

        install.packages(pvclust)
	
        library(pvclust)
	
        bootstraps = pvclust(cpm_log, method.hclust="average", method.dist="euclidean")

        plot(bootstraps)

How do the samples cluster? What can you conclude about the genomic architecture of wing iridescence? Is it likely that many or a few genes encode this phenotype?
	
* Plot heatmap

        install.packages(pvclust)
	
        library(pheatmap)
	
        install.packages(colorRamps)

        library(colorRamps)

        palette2 <-colorRamps::"matlab.like2"(n=200)

        pheatmap(cpm_log, show_colnames=T, show_rownames=F, color = palette2,clustering_distance_cols = "euclidean", clustering_method="average") 

---

## 6. Identify differentially expressed genes in a pairwise comparison

Next we can use edgeR to identify differentially expressed genes between two treatments or conditions. We have a pairwise comparison between two groups. Further information about each step can be found in the [edgeR manual](https://bioconductor.org/packages/release/bioc/vignettes/edgeR/inst/doc/edgeRUsersGuide.pdf). You can also refer for the approach that is appropriate if you have more than two treatments. 

Briefly, edgeR uses the quantile-adjusted conditional maximum likelihood (qCML) method for experiments with single factor. The qCML method calculates the likelihood by conditioning on the total counts for each tag, and uses pseudo counts after adjusting for library sizes. The edgeR functions `estimateCommonDisp` and `exactTest` produce a pseudo counts. We can then proceed with determining differential expression using the `exactTest` function. The exact test is based on the qCML methods. We can compute exact p-values by summing over all sums of counts that have a probability less than the probability under the null hypothesis of the observed sum of counts. The exact test for the negative binomial distribution has strong parallels with Fisher’s exact test.

## e. PRACTICAL ACTIVITY

* Estimating common dispersion.

        expr_filtered<- estimateCommonDisp(expr_filtered)

* Estimating tagwise dispersion.

        expr_filtered<- estimateCommonDisp(expr_filtered)

        levels(expr_filtered$samples$group)

* Exact test to calculate logFC, logCPM and PValue for every gene.

        et <- exactTest(expr_filtered, pair=c("A","I"))

        et

* Correct for multiple testing.

        p <- et$table$PValue

        p_FDR <- p.adjust(p, method = c("fdr"), n = length(p))

        print(length(p_FDR))

        table_de <- et$table

        table_de$Padj <- p_FDR

* Print results to a new file.

        filename = paste("~/Desktop/Wing-bias-fdr.txt")

        print(filename)

        write.table(table_de, file=filename,quote=F, sep="\t")
	
* Identify differentially expressed genes between I and A.

Normally, we use a fdr p-value threshold < 0.05. It is also important to consider imposing a fold change threshold eg logFC of 1. As we conducted `exactTest(expr, pair=c("A","I"))`, positive logFC means I > A (I-biased), negative logFC means A > I (A-biased).

	I-biased genes

		print(length(which(table_de$Padj < 0.05 & table_de$logFC > 1)))
	
		table_de[which(table_de$Padj < 0.05 & table_de$logFC > 1),]

	A-biased genes
	
		print(length(which(table_de$Padj < 0.05 & table_de$logFC < -1)))
	
		table_de[which(table_de$Padj < 0.05 & table_de$logFC < -1),]
	
* Lets look up and check whether any of these differentially expressed genes are annotated, and find out gene features. A note of caution here, the *Heliconius melpomene* is not well annotated relative to genomes of model species.

	First, look up the gene in the gtf file to identify its annotated gene name. eg
	
		grep "XLOC_010550" /fastdata/$USER/align/Cufflinks_output/merged_asm/merged_allstranded.gtf
	
	Then find the Ensembl gene id. This specified in the `oId` flag and starts with HMEL... eg
	
		HMEL031499g1.t1
	
	Look up the gene id in [Lepbase](http://ensembl.lepbase.org/Heliconius_melpomene_melpomene_hmel2/Info/Index). You need to drop the transcript info from the gene name eg
		
		HMEL031499g1
	
	What information can you find out about the gene? Does it have any orthologs and if so can you infer the function of this gene?
	
---
