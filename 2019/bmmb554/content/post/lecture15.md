---
date: "2019-04-01"
tags: ["SNPs", "variants"]
title: "Lecture 15: Calling variants"
---	

Today we hear a lot about personalized medicine. Yet the *personalization* is defined by the genetic make up of the individual. Today we will discuss how this information can be uncovered from the genomic sequencing data. The figure above shows distribution of rare and common variants in 1,092 human genomes described by the [1000 Genome Consortium](http://www.nature.com/nature/journal/v491/n7422/abs/nature11632.html).

# Calling variants

Variant calling is a complex field that was significantly propelled by advances in DNA sequencing and efforts of large scientific consortia such as the [1000 Genomes](http://www.1000genomes.org). Here we summarize basic ideas central to Genotype and Variant calling. First, let's contrast the two things although they often go together:

 * **Variant calling** - identification of positions where the sequenced sample is different from the reference sequence (or [reference genome graph](https://github.com/vgteam/vg));
 * **Genotype calling** - identifying individual's genotype at variable sites.

A typical workflow for variation discovery involves the following steps (e.g., see Nielsen et al. [2011](http://www.nature.com/nrg/journal/v12/n6/full/nrg2986.html)):

 1. Mapping reads against the reference genome
 2. Thresholding BAM datasets by, for example, retaining paired, properly mapped reads
 3. Performing quality score recalibration
 4. Performing realignment
 5. Performing variant calling/genotype assignment
 6. Performing filtering and genotype quality score recalibration
 7. Annotating variants and performing downstream analyses

However, continuing evolution of variant detection methods has made some of these steps obsolete. For instance, omitting quality score recalibration and re-alignment (steps 3 and 4 above) when using haplotype-aware variant callers such as [FreeBayes](https://github.com/ekg/freebayes) does not have an effect on the resulting calls (see Brad Chapman's methodological comparisons at [bcbio](http://bit.ly/1S9kFJN)). Before going forward with an actual genotype calling in Galaxy let's take a look as some basic ideas behind modern variant callers.

### How does SNP calling and genotyping work?

Consider a set of sequencing reads derived from a diploid individual:

```
REFERENCE: atcatgacggcaGtagcatat
--------------------------------
READ1:     atcatgacggcaGtagcatat
READ2:         tgacggcaGtagcatat
READ3:     atcatgacggcaAtagca
READ4:            cggcaGtagcatat
READ5:     atcatgacggcaGtagc
```

The capitalized position contains a G &#8594; A [transition](https://en.wikipedia.org/wiki/Transition_(genetics)). So, in principle this can be a heterozygous site with two alleles **G** and **A**. A commonly used naïve procedure would define a site as *heterozygous* if there is a non-reference allele with frequency between 20% and 80%. In this case **A** is present in 1/5 or 20% of the cases, so we can say that this is a heterozygous site. Yet it is only represented by a single read and thus is hardly reliable. Here are some of the possibilities that would explain this *variant*. It can be:

 * A true variant
 * Experimental artifact: A library preparation error (e.g., PCR-derived)
 * Base calling error
 * Analysis error: A misalignment (though unlikely in the above example)

The modern variant callers attempt to assign a reliability estimate for each genotype call. This is done using Bayes reasoning (for a great visual explanation see [blog](https://oscarbonilla.com/2009/05/visualizing-bayes-theorem/) by Oscar Bonilla). Here we present a SNP-relevant "translation" on this explanation (with inspiration from [Erik Garrison](https://github.com/ekg)).

Suppose in a population you have $A$ individuals (not to be confused with nucleotide **A**; in this case $A$ is a number of individuals) with a variant. You are performing re-sequencing and observe a variant in $B$ (again, a number) of your sequencing reads. We want to estimate the probability of having the real polymorphism in the population given our observations in sequencing reads. The logic is as follows:

 * The probability of having polymorphism **A** in the population is $P(A) = |A|/|U|$
 * The probability of seeing a variant given our identification approach (i.e., sequencing) is $P(B) = |B|/|U|$
 * Now, the probability of having a variant and it being observed in our sequencing data is the overlap between $A$ and $B$ sets $P(AB) = |AB|/|U|$. This is presented graphically below:

|    |    |     |
|:----:|:----:|:-----:|
| ![](/BMMB554/img/pA.png) | ![](/BMMB554/img/pB.png) | ![](/BMMB554/img/pAB.png) |
| $P(A)$ Polymorphisms | $P(B)$ <br> Variant calls | $P(AB)$ Polymorphisms + Varinat calls |

Now we can ask the following question: *What is the probability of a having a real polymorphism* $A$ *given our observation of variants in reads* $B$? In other words *what is the probability of* $A$ *given* $B$? Or, as stated in the original [blog](https://oscarbonilla.com/2009/05/visualizing-bayes-theorem/): "*given that we are in region $B$ what is the probability that we are in the region $AB$*?":

 * $P(A|B) = \frac{|AB|}{|B|}$
 * Dividing by $|U|$: $P(A|B) = \frac{\frac{|AB|}{|U|}}{\frac{|B|}{|U|}}$
 * Because we know that $P(AB) = \frac{|AB|}{|U|}$ and $P(B) = \frac{|B|}{|U|}$ we can rewrite the equation in the previous bullet point as $P(A|B) = \frac{P(AB)}{P(B)}$

Now, let's ask an opposite question. Given a true polymorphism $A$ what are the chances that we do detect it (i.e., find ourselves in $AB$)? It will be:

$$
	P(B|A) = \frac{P(AB)}{P(A)}
$$

So, because we know that $P(A|B) = \frac{P(AB)}{P(B)}$ and we just reasoned that $P(B|A) = \frac{P(AB)}{P(A)}$, we can say that $P(A|B)P(B) = P(B|A)P(A)$ leading us to the [Bayes formula](http://www.math.cornell.edu/~mec/2008-2009/TianyiZheng/Bayes.html):


$$
    P(A|B) = \frac{P(B|A)P(A)}{P(B)}
$$

Translating this into "genomics terms" the probability of having a genotype $G$ given sequencing reads $S$ is: $P(G|S) = \frac{P(S|G)P(G)}{P(S)}$. Because in a given calculation of $P(G|S)$ reads are fixed we can re-write the Bayes formula in the following way:

$$
	P(G|S) \approx P(S|G)P(G)
$$

 with $P(S)$ becoming a constant. This leaves us with the need to estimate two things:

 1. $P(S|G)$, the data likelihood
 2. $P(G)$, the prior probability for the variant

In the simplest case we can estimate these as follows:

### $P(S|G)$

Suppose $S_i$ is a base in read $i$ corresponding to a genome position with genotype $G$. The probability of seeing $S_i$ given $G$, or $P(S_i|G)$, is given by the quality score of $S_i$ (the quality scores are given by base calling software and reported as [phred scores](https://en.wikipedia.org/wiki/Phred_quality_score)). Thus the genotype likelihood $P(S|G)$ is the product of $P(S_i|G)$ over all $i$. In reality however there are many other sources of uncertainty (in addition to base qualities) that are incorporated in the calculation of data likelihoods including NGS technology-related issues, dependency of error rates on substitution type (e.g., transitions versus transversions), sequencing context etc...

### $P(G)$ - a single sample case

One can assign an equal probability to all possible genotypes, or to source this information based on previously obtained knowledge containing in a database, such as [dbSNP](http://www.ncbi.nlm.nih.gov/SNP/). In this case (as exemplified in [Nielsen et al. 2011](http://www.nature.com/nrg/journal/v12/n6/full/nrg2986.html)) we may, for instance, have a site with a **G/T** polymorphism and genotypes **GG**, **TT**, and **GT** having frequencies of 0.45, 0.45, 0.09, respectively. We will use these values as priors.

### $P(G)$ - a multi-sample case

Genotype calling reliability can be significantly improved when analyzing multiple samples jointly. In this case genotype frequencies can be inferred from allele frequencies using Hardy-Weinberg equilibrium ([HWE](https://en.wikipedia.org/wiki/Hardy%E2%80%93Weinberg_principle)). The following example (again from [Nielsen et al. 2011](http://www.nature.com/nrg/journal/v12/n6/full/nrg2986.html)) illustrates this idea: suppose you are calling genotypes for a single individual using a combination of multiple samples. There are two genotypes, **AT** and **AA**, with equally large genotype likelihoods. If, however, in our collection of multiple samples the frequency of **A** is 1% ($p = 0.01$; $q = 1 - p = 0.99$), then from the HWE we have:

|     |    |     |
|---------|---------|--------|
| 0.0001 | 0.0198 | 0.9801 |
| **AA** ($p^2$)   |   **AT** ($2pq$) |   **TT** ($q^2$) |

This makes it highly unlikely that **AA** is a true genotype of this individual.

## Calling with FreeBayes

[FreeBayes](https://github.com/ekg/freebayes) is an open source variant caller that has been battle-tested by the 1000 Genomes community and is extensively used today (also see [bcbio](https://bcbio.wordpress.com/)). It has a number of features that simplify variant discovery workflows. These include (from FreeBayes github page):

* **Indel realignment is accomplished internally** using a read-independent method, and issues resulting from discordant alignments are dramatically reducedy through the direct detection of haplotypes;
* **The need for base quality recalibration is avoided** through the direct detection of haplotypes. Sequencing platform errors tend to cluster (e.g. at the ends of reads), and generate unique, non-repeating haplotypes at a given locus;
* **Variant quality recalibration is avoided** by incorporating a number of metrics, such as read placement bias and allele balance, directly into the Bayesian model;
* **Ability to incorporate non-diploid cases** such as pooled datasets or data from polyploid samples.

Freebayes is a *haplotype-based* variant caller. This implies that instead of looking at an individual positions within an alignment of reads to the reference genome, it looks at a haplotype window, length of which is dynamically determined (see section 3.2. in [FreeBayes manuscript](http://arxiv.org/pdf/1207.3907v2.pdf)):

|             |
|-------------|
|![](/BMMB554/img/freebayes.png)|
|<small>Looking at a haplotype window makes misalignments tolerable. In this case a low complexity poly(A) stretch is misaligned. As a result looking at individual positions will result in calling multiple spurious varians. In the case of FreeBayes looking at a haplotype identifies two alleles (this is a diploid example) `A(7)` and `A(6)`, while `A(8)` is likely an error. Image by [Erik Garrison](https://github.com/ekg/freebayes)</small>|

# Let's try it

## The data

In this example we will perform variant calling and annotation using [genome in the bottle data](http://jimb.stanford.edu/giab/). Specifically, we will use Ashkenazim Father-Mother-Son trio data from the Personal Genome Project:

* HG002 - NA24385 - huAA53E0 (son)
* HG003 - NA24149 - hu6E4515 (father)
* HG004 - NA24143 - hu8E87A9 (mother)

Yet for a quick tutorial these datasets are way too big, so we created a downsampled (watered down) dataset. This dataset was produced by mapping the trio reads against the `hg19` version of the human genome, merging the resulting bam files together (we use readgroups to label individual reads so they can be traced to each of the original individuals), and restricting alignments to a small portion of chromosome 19 containing the [*POLRMT*](http://www.ncbi.nlm.nih.gov/gene?cmd=Retrieve&dopt=Graphics&list_uids=5442) gene.

Here is what to do to load the data:

## Loading the data

Go to the [data library](https://usegalaxy.org/library/list#folders/F9ff2d127cd7ed6bc) and select both BAM and PED datasets. Then Click **to History** button:

![](/BMMB554/img/library_import.png)

Galaxy will ask you if you want to import these data into a new history, which you might want (in the case below I called this history `genotyping try`):

![](/BMMB554/img/history_import.png)

The datasets will appear in your history:

![](/BMMB554/img/library_import_complete.png)

## Generating and post-processing FreeBayes calls

Select **FreeBayes** from **NGS: Variant Analysis** section of the tool menu (left pane of Galaxy's interface).

### Running FreeBayes

Make sure the top part of the interface looks like shown below. Here we selected `GIAB-Ashkenazim-Trio-hg19` as input and set **Using reference genome** to `hg19` and **Choose parameter selection level** to `5`. The interface should look like this:

![](/BMMB554/img/FreeBayes_settings.png)

Scrolling down to **Tweak algorithmic features?** click `Yes` and set **Calculate the marginal probability of genotypes and report as GQ in each sample field in the VCF output** to `Yes`. This would help us evaluating the quality of genotype calls.

![](https://github.com/nekrut/galaxy/wiki/images/freebayes_gq.png)

Depending on how busy Galaxy is this may take a little bit of time (coffee break?). Eventially this will produce a dataset in [VCF](http://www.1000genomes.org/wiki/Analysis/variant-call-format) format containing 35 putative variants. Before we can continue we need to post-process this dataset by breaking compound variants into multiple independent variants with **VcfAllelicPrimitives** tool found within **NGS: VCF Manipulation** section. This is necessary for ensuring the smooth sailing through downstream analyses:

### Simplifying variant representation

Select FreeBayes output as the input for this tool and make sure **Maintain site and allele-level annotations when decomposing** and **Maintain genotype-level annotations when decomposing** are set to `Yes`:

![](/BMMB554/img/vcfallelicprimitives.png)

**VCFAllelicPrimities** generated a VCF files containing 37 records (the input VCF only contained 35). This is because a multiple nucleotide polymorphism (`TAGG|CAGA`) at position 618851 have been converted to two:

**Before:**

```
chr19 618851 . TAGG CAGA 81.7546
```

**After:**

```
chr19 618851 . T C 81.7546
chr19 618854 . G A 81.7546
```

### Annotating variants with SnpEff

At this point we are ready to begin annotating variants using [SnpEff](http://snpeff.sourceforge.net/SnpEff.html). SnpEff, a project maintained by [Pablo Cingolani](https://www.linkedin.com/in/pablocingolani) "*...annotates and predicts the effects of variants on genes (such as amino acid changes)...*" and so is critical for functional interpretation of variation data.

Select the latest version of annotation database matching genome version against which reads were mapped and VCF produced. In this case it is `GRCh37.75: hg19`:

![](/BMMB554/img/snpeff.png)

SnpEff will generate two outputs: (1) an annotated VCF file and (2) an HTML report. The report contains a number of useful metrics such as distribution of variants across gene features:

![](/BMMB554/img/snpeff_chart.png)

or changes to codons:

![](/BMMB554/img/snpeff_codons.png)

### Manipulating variation data with GEMINI

Now that we have an annotated VCF file it is time to peek inside our variation data. [Aaron Quinlan](http://quinlanlab.org/), creator of [GEMINI](http://gemini.readthedocs.org/en/latest/index.html), calls it *Detective work*.

#### Loading data into GEMINI

The first step is to convert a VCF file we would like to analyze into a GEMINI database. For this we will use **GEMINI Load** tool from **NGS: GEMINI** section. GEMINI takes as input a VCF file and a [PED](http://pngu.mgh.harvard.edu/~purcell/plink/data.shtml) file describing the relationship between samples. In the case of our dataset the PED file looks like this (accessible from [here](https://usegalaxy.org/library/list#folders/F9ff2d127cd7ed6bc/datasets/418b2500e809568b)):

```
#family_id sample_id            paternal_id          maternal_id         sex phenotype ethnicity
family1    HG004_NA24143_mother -9                   -9                   2  1         CEU
family1	   HG003_NA24149_father -9                   -9                   1  1         CEU
family1	   HG002_NA24385_son	HG003_NA24149_father HG004_NA24143_mother 1  2         CEU
```
So let's load data into GEMINI. Set VCF and PED inputs:

![](/BMMB554/img/gemini_load.png)

This creates a sqlite database. To see the content of the database use **GEMINI_db_info**:

![](/BMMB554/img/gemini_db_info.png)

This produce a list of [all tables and fields](https://github.com/nekrut/galaxy/wiki/datasets/gemini_tables.txt) in the database.

#### Querying GEMINI database

GEMINI database is queried using the versatile SQL language (more on SQL [here](http://swcarpentry.github.io/sql-novice-survey)). In Galaxy's version of GEMINI this is done using **GEMINI_query** tool. Within this tool SQL commands are typed directly into the **The query to be issued to the database** text box. Let's begin getting information from some of the tables we discovered with **GEMINI_db_info** tool above.
>
The examples below are taken from "[Intro to Gemini](https://s3.amazonaws.com/gemini-tutorials/Intro-To-Gemini.pdf)" tutorial. For extensive documentation see "[Querying GEMINI](http://gemini.readthedocs.org/en/latest/content/querying.html)".

> #### *Are there "novel" varinats that are not annotated in dbSNP database?*

To answer this question we will type the following query:

```
SELECT count(*) FROM variants WHERE in_dbsnp == 0
```

into **The query to be issued to the database** field of the interface:

![](/BMMB554/img/gemini_query1.png)

As we can see from [output (Click this link to see it)](https://usegalaxy.org/datasets/bbd44e69cb8906b51bb37b9032761321/display/?preview=True) there are 21 variants that are not annotated in dbSNP.

> #### *Which variants are found within POLRMT gene?* 

To answer this type:

```
SELECT * FROM variants WHERE filter is NULL and gene = 'POLRMT'
```

The above query will produce [output](https://usegalaxy.org/datasets/bbd44e69cb8906b5a0bb5b2cc0695697/display/?preview=True) with very large number of columns. To restrict the number of columns to a manageable set let's use this command (you may need to scroll sideways): 

```
SELECT rs_ids, aaf_esp_ea, impact, clinvar_disease_name, clinvar_sig FROM variants WHERE filter is NULL and gene = 'POLRMT'
```

(column definitions can be found [here](http://gemini.readthedocs.org/en/latest/content/database_schema.html))

[Output](https://usegalaxy.org/datasets/bbd44e69cb8906b540d65297cd1d26bb/display/?preview=True) shows varinats found within the *POLRMT* gene.

#### Querying genotypes

GEMINI provides access to genotype, sequencing depth, genotype quality, and genotype likelihoods for each individual (`subjectID`):

 * `gt_types.subjectID` - three types of genotype types: `HOM_REF`, `HET`, 'HOM_ALT`;
 * `gt_quals.subjectID` - genotype quality
 * `gt_depths.subjectID` - total number of reads in this subject at position
 * `gt_ref_depths.subjectID` -  number of reference allele reads in this subject at position
 * `gt_alt_depths.subjectID` - number of alternate allele reads in this subject at position

> #### *At how many sites does child in our trio have a non-reference allele?* 

To answer this we will use two fields of **GEMINI_query** interface. In the **The query to be issued to the database** we will type:

```
SELECT * from variants
```

and in the field **Restrictions to apply to genotype values** we will enter:

```
gt_types.HG002_NA24385_son <> HOM_REF
```

![](/BMMB554/img/gemini_query2.png)

This produce [a list of sites](https://usegalaxy.org/datasets/bbd44e69cb8906b560921700703d0255/display/?preview=True)

> #### *At how many sites both father and son have non reference alleles?*

To answer this we will type the same expression 

```
SELECT * from variants
```

 into **The query to be issued to the database** field and 

 ```
 (gt_types.HG002_NA24385_son <> HOM_REF AND gt_types.HG003_NA24149_father <> HOM_REF)
 ```

 into **Restrictions to apply to genotype values**.

This will produce the following [output](https://usegalaxy.org/datasets/bbd44e69cb8906b5aab445b3cd632ba7/display/?preview=True)


> #### *List genotypes for father and son where they have non-reference alleles.*

Type the following:

```
SELECT gts.HG002_NA24385_son, gts.HG003_NA24149_father from variants
```

into **The query to be issued to the database** and 

```
(gt_types.HG002_NA24385_son <> HOM_REF AND gt_types.HG003_NA24149_father <> HOM_REF)
```

into **Restrictions to apply to genotype values**. Output will look like [this](https://usegalaxy.org/datasets/bbd44e69cb8906b543c67f80be21ed02/display/?preview=True).

#### Using wildcards

Wilcards simply writing SQL expressions when searching across multiple terms. The syntax for genotype filter wilcards is

```
(COLUMN).(SAMPLE_WILDCARD).(SAMPLE_WILDCARD_RULE).(RULE_ENFORCEMENT)
```

Let's try a few examples.


> #### *At which variants all samples are heterozygous?*

Type

```
SELECT chrom, start, end, ref, alt, gene, impact, (gts).(*) FROM variants
```

into **The query to be issued to the database** and 

```
(gt_types).(*).(==HET).(all)
``` 

into **Restrictions to apply to genotype values**. Here we use wildcards for the query 

 * `(gts.*)` = get genotypes for **all** samples

 and genotype filtering

 * `(gt_types).(*).(==HET).(all)`

 the [all operator](http://gemini.readthedocs.org/en/latest/content/querying.html#the-all-operator) implies that want results for **all** afftected individuals). Output will look like [this](https://usegalaxy.org/datasets/bbd44e69cb8906b5819e1404b5e127d1/display/?preview=True).

### Going further

This short tutorial should give you an overall idea on how generate variant data in Galaxy and process it with GEMINI. Yet there is much more to learn. Below we list GEMINI tutorials and links to Galaxy libraries with relevant data:

|     |     |     |     |
|:----------|:-----:|:------:|:----------------:|
| Introduction | [ PDF ](https://s3.amazonaws.com/gemini-tutorials/Intro-To-Gemini.pdf) | [ Sample Data ](https://usegalaxy.org/library/list#folders/F0283ca691a41c352) | [ Galaxy history ]( https://usegalaxy.org/u/aun1/h/gemini-introduction) |
| Identifying *de novo* mutations underlying Mendelian disease | [ PDF ](https://s3.amazonaws.com/gemini-tutorials/Gemini-DeNovo-Tutorial.pdf) | [ Sample Data ](https://usegalaxy.org/library/list#folders/F775008f45cbbf010) | [ Galaxy history ]( https://usegalaxy.org/u/aun1/h/gemini-de-novo-mutations) |
| Identifying autosomal recessive variants underlying Mendelian disease | [ PDF ](https://s3.amazonaws.com/gemini-tutorials/Gemini-Recessive-Tutorial.pdf) | [ Sample Data ](https://usegalaxy.org/library/list#folders/F35b262f5ac8aa63a) | [ Galaxy history ](https://usegalaxy.org/u/aun1/h/gemini-autosomal-recessive) |
| Identifying autosomal dominant variants underlying Mendelian disease | [ PDF ](https://s3.amazonaws.com/gemini-tutorials/Gemini-Dominant-Tutorial.pdf) | [ Sample Data ](https://usegalaxy.org/library/list#folders/F1c4722ad56892a31) | [ Galaxy history ]( https://usegalaxy.org/u/aun1/h/gemini-autosomal-dominant ) |

#### How to use these tutorials?

* Right click on the **PDF** link and open tutorial in a new browser tab
* Right click on **Galaxy history** link and open Galaxy history in another new browser tab
* When Galaxy history interface opens you will need to click **Import history** link highlighted within a red outline in the following figure:

![](https://github.com/nekrut/galaxy/wiki/images/import_history.png)

* If you have a wide screen arrange browsers tabs side by side:

![](/BMMB554/img/side-by-side.png)

* Proceed with tutorial. For example, to repeat the following command from GEMINI tutorial:

![](/BMMB554/img/gemini_command.png)

* Use Galaxy's **GEMINI_load** tool:

![](/BMMB554/img/galaxy_command.png)

* and so on....

# Why call variants: An example from Tibet

Many residents of the Tibetian Plateau live above 4,000 meters where oxygen concentration is approximately 40% lower than at sea level:

|             |
|-------------|
|![](/BMMB554/img/tibet.png)|
|<small>Tibetan Plateau and surrounding areas (from [Wikipedia](https://en.wikipedia.org/wiki/Tibet))</small>|

Tibetian experience a number of adaptations to high altitudes manifesting in the following phenotypic differences:

- lower hemoglobin concentration [Wu:2005](http://science.sciencemag.org/lookup/ijlink?linkType=ABST&journalCode=jap&resid=98/2/598&atom=%2Fsci%2F329%2F5987%2F75.atom)
- higher arterial oxygen saturation [Niermeyer:1995](http://science.sciencemag.org/lookup/ijlink?linkType=ABST&journalCode=jap&resid=98/2/598&atom=%2Fsci%2F329%2F5987%2F75.atom)
- more efficient pulmonary gas exchange [Zhuang:1996](https://www.ncbi.nlm.nih.gov/pubmed/8822225)

What are the genetic causes of these adaptations?

## Sequencing of 50 Human Exomes Reveals Adaptation to High Altitude

[Emilia Huerta-Sanchez and Rasmus Nielsen](http://science.sciencemag.org/content/329/5987/75) decided to investigate this phenomenon using the following experimental design:

### Design

They sequenced exomes from 50 unrelated individuals from two villages (both situated at or above 4,300 m above sea level) in the Tibet Autonomous Region of China. Exomes were enriched using [Numblegen exome capture system](http://sequencing.roche.com/products/nimblegen-seqcap-target-enrichment.html) (approximately 20,000 genes) and sequenced to mean depth of ~18x. 

## SNPs

18x coverage does not guarantee accurate assignment of individual genotypes. To address this challenge Nielsen et al. devised a Bayesian approach that would (1) estimate reliability for SNP calls and (2) compute population allele frequencies for each site. 

They found:

 - 151,825 SNPs > 50% of probability being variable
 - 101,668 SNPs > 99% probability of being variable

Sanger sequencing validated 53 out of 56 that had >95% probability of being variable and minor allele frequencies between 3% and 50%.

### Population history of Tibetians and Han Chinese

The Tibetian exome was compared with 40 Han Chinese from Beijing genomes (HCB) from the 1000 Genomes Project. Beijing is approximately 50 m above sea level and the adsolute Majority of Han Chinese live below 2,000 m. The amount of genetic differentiation between Tibetian and Han samples is low ([F<sub>*ST*</sub>](https://en.wikipedia.org/wiki/Fixation_index) = 0.026). In addition, there is also no differentiation between the two Tibetian villages (F<sub>*ST*</sub>=0.014) and so they are treated as a single population. The paper estimated that the Tibetian and Han populations diverged ~2,750 years ago with Tibetian population shrinking and Han population expanding. 

|             |
|-------------|
|![](https://d2ufo47lrtsv5s.cloudfront.net/content/sci/329/5987/75/F1.large.jpg?width=800&height=600&carousel=1)|
|<small>Two-dimensional unfolded site frequency spectrum for SNPs in Tibetan (x axis) and Han (y axis) population samples. The number of SNPs detected is color-coded according to the logarithmic scale plotted on the right. Arrows indicate a pair of intronic SNPs from the EPAS1 gene that show strongly elevated derived allele frequencies in the Tibetan sample compared with the Han sample.</small>|


### Finding casual SNPs

With two populations we one can potentially find genes with significant allele frequency differences but we will not be able to tell which of the two populations is affected by selection. To do this we need an outgroup represented by a distantly related population. In this study they've chosen a Danish population as an outgroup. 

|             |
|-------------|
|![](https://d2ufo47lrtsv5s.cloudfront.net/content/sci/329/5987/75/F2.large.jpg?width=800&height=600&carousel=1)|
|<small>Population-specific allele frequency change. (A) The distribution of F<sub>*ST*</sub>-based PBS statistics for the Tibetan branches, according to the number of variable sites in each gene. Outlier genes are indicated in red. (B) The signal of selection on EPAS1: Genomic average F<sub>*ST*</sub>-based branch lengths for Tibetan (T), Han (H), and Danish (D) branches (left) and branch lengths for EPAS1, indicating substantial differentiation along the Tibetan lineage (right).</small>|

### *EPAS1*

The strongest signal was observed for the *EPAS1* gene, which has 9% derived allele frequency in Han Chinese and 87% in Tibetian population. To establish functional link between this site and inhabiting high altitudes authors performed association analysis:  "*Significant associations were discovered for erythrocyte count (F test P = 0.00141) and for hemoglobin concentration (F test P = 0.00131), with significant or marginally significant P values for both traits when each village was tested separately
The allele at high frequency in the Tibetan sample was associated with lower erythrocyte quantities and correspondingly lower hemoglobin levels (table S4). Because elevated erythrocyte production is a common response to hypoxic stress, it may be that carriers of the “Tibetan” allele of EPAS1 are able to maintain sufficient oxygenation of tissues at high altitude without the need for increased erythrocyte levels. Thus, the hematological differences observed here may not represent the phenotypic target of selection and could instead reflect a side effect of EPAS1-mediated adaptation to hypoxic conditions.*"

## Altitude adaptation in Tibetans caused by introgression of Denisovan-like DNA

[Emilia Huerta-Sanchez and Rasmus Nielsen](http://www.nature.com/nature/journal/v512/n7513/full/nature13408.html)

|             |
|-------------|
|![](http://www.nature.com/nature/journal/v512/n7513/images/nature13408-f2.jpg)|
|<small>Each column is a polymorphic genomic location (95 in total), each row is a phased haplotype (80 Han and 80 Tibetan haplotypes), and the coloured column on the left denotes the population identity of the individuals. Haplotypes of the Denisovan individual are shown in the top two rows (green). The black cells represent the presence of the derived allele and the grey space represents the presence of the ancestral allele (see Methods). The first and last columns correspond to the first and last positions in Supplementary Table 3, respectively. The red and blue arrows indicate the 32 sites in Supplementary Table 3. The blue arrows represent a five-SNP haplotype block defined by the first five SNPs in the 32.7-kb region. Asterisks indicate sites at which Tibetans share a derived allele with the Denisovan individual.</small>|

|             |
|-------------|
|![](http://www.nature.com/nature/journal/v512/n7513/images/nature13408-f3.jpg)|
|<small>The haplotypes were defined from all the SNPs present in the combined 1000 Genomes and Tibetan samples: 515 SNPs in total within the 32.7-kb EPAS1 region. The Denisovan haplotypes were added to the set of the common haplotypes. The R software package pegas23 was used to generate the figure, using pairwise differences as distances. Each pie chart represents one unique haplotype, labelled with Roman numerals, and the radius of the pie chart is proportional to the log2(number of chromosomes with that haplotype) plus a minimum size so that it is easier to see the Denisovan haplotype. The sections in the pie provide the breakdown of the haplotype representation amongst populations. The width of the edges is proportional to the number of pairwise differences between the joined haplotypes; the thinnest edge represents a difference of one mutation. The legend shows all the possible haplotypes among these populations. The numbers (1, 9, 35 and 40) next to an edge (the line connecting two haplotypes) in the bottom right are the number of pairwise differences between the corresponding haplotypes. We added an edge afterwards between the Tibetan haplotype XXXIII and its closest non-Denisovan haplotype (XXI) to indicate its divergence from the other modern human groups. Extended Data Fig. 5a contains all the pairwise differences between the haplotypes presented in this figure. ASW, African Americans from the south western United States; CEU, Utah residents with northern and western European ancestry; GBR, British; FIN, Finnish; JPT, Japanese; LWK, Luhya; CHS, southern Han Chinese; CHB, Han Chinese from Beijing; MXL, Mexican; PUR, Puerto Rican; CLM, Colombian; TSI, Toscani; YRI, Yoruban. Where there is only one line within a pie chart, this indicates that only one population contains the haplotype.</small>|

