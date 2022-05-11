# Basic long-read sequencing QC 

This page cotains a tutorial for basic Quality Control (QC) of Long-Read sequencing data, mainly oriented to data produced by Oxford Nanopore Tenchnologies (**ONT**) and Pacific Biosciences (**PacBio**). The material was initially created for the Computational Sessions of the [2022 JAX Long Read Sequencing Workshop ](https://www.jax.org/education-and-learning/education-calendar/2022/may/long-read-sequencing-workshop) but I will try to keep it updated with new methods/strategies.

## Tutorial Set Up

### Unix 
Unix is the standard operating system used in scientific research. In fact, most up-to-date tools in bioiformatics, including the ones we are going to use in this tutorial, are run in Unix. In the context of this tutorial (and Bioiformatics in general) there are few aspects of Unix to highligth: 

- Unix is largely command line driven. 
- Unix is used by many of the most powerful computers at bioinformatics centres and also on many desktops and laptops (MacOS is largely UNIX compatible).
- Unix is free. 
- There are many distributions of Unix such as Ubuntu, RedHat, Fedora, etc. These are all Unix, but they bundle up extra software in a different way or combinations. 

There are seveal **really good** free on-line tutorials that introduce Unix and cover some of the basics that will allow you to be more comfortable with the command-line, here some of them:

1. [Software Carpentry Foundation Shell Novice Course](https://swcarpentry.github.io/shell-novice/)
2. [Bioinformatics Workbook Unix Basics](https://bioinformaticsworkbook.org/Appendix/Unix/unix-basics-1.html#gsc.tab=0)
3. [HadrienG Tutorial on Command Line](https://www.hadriengourle.com/tutorials/command_line/)

### I don't have Unix, what do I do?
In case you don’t work under any Unix distribution already, e.g. you have a Windows machine, one of the easiest way to start is by setting up a virtual machine with one of the most widely used Unix distributions: **Ubuntu**. This page gives detailed instructions on how to set up an Ubuntu environment in any computer using VirtualBox: [Install Ubuntu using a VM](https://ubuntu.com/tutorials/how-to-run-ubuntu-desktop-on-a-virtual-machine-using-virtualbox#1-overview) 

### Tools
Tools needed for this tutorial and the instructions for installing them are:

- **SeqKik**: [https://bioinf.shenwei.me/seqkit/](https://bioinf.shenwei.me/seqkit/)
- **SeqFu**: [https://telatin.github.io/seqfu2/](https://telatin.github.io/seqfu2/)
- **NanoPack**:[https://github.com/wdecoster/nanopack](https://github.com/wdecoster/nanopack)
- **pycoQC**: [https://a-slide.github.io/pycoQC/](https://a-slide.github.io/pycoQC/)

### Data Set
For this tutorial we will use data published by [Tvedte *et al.* 2021](https://academic.oup.com/g3journal/article/11/6/jkab083/6188627):

> Eric S Tvedte, Mark Gasser, Benjamin C Sparklin, Jane Michalski, Carl E Hjelmen, J Spencer Johnston, Xuechu Zhao, Robin Bromley, Luke J Tallon, Lisa Sadzewicz, David A Rasko, Julie C Dunning Hotopp, *Comparison of long-read sequencing technologies in interrogating bacteria and fly genomes*, **G3**,11, 6, 2021, https://doi.org/10.1093/g3journal/jkab083.

In this work, authors performed whole-genome sequencing of the bacteria *Escherichia coli* using three different **PacBio** protocols (*Sequel II CLR, Sequel II HiFi, RS II*) and three **ONT** protocols (Rapid Sequencing and Ligation Sequencing with and without fragmentation step) in order to compare genome assemblies. We will make use of this dataset in order to explore sequencing data produced by the different approaches.

**NOTE**: There are several parameters and steps that migth affect sequencing results (e.g. DNA extraction, library preparation, sequencing instruments, versions of programs used in the analysis, version of library kits, etc.) Results obtained by Tvedte *et al.* 2021 migth no **NOT** be generalizable to other sequencing experiments in term of yield, quality and read length!!!

Organism|Library|SRA accession
-------|---------|-------------
*E. coli*|ONT RAPID|SRR11523179 
*E. coli*|ONT LIG|SRR11434959
*E. coli*|ONT LIG noFrag|SRR12801740 
*E. coli*|PacBio RS II|SRR11434956 
*E. coli*|Pacbio Sequel II CLR|SRR11434960 
*E. coli*|Pacbio Sequel II HiFi|SRR11434954 

#### Toy dataset:
I have prepared a toy dataset from these six libraries by randomly subsampling 10,000 reads from each. You can use this dataset to rund this tutorial. You can download here: [sampleData.tar](https://thejacksonlaboratory.box.com/s/9ny2zvx3pby1yp3b775c9jraik9jerzp) 

If you want to download the full data set, you can use [SRAToolkit](https://github.com/ncbi/sra-tools), specifically commands **prefetch** and **fastq-dump** as follow:

```markdown
$ prefetch SRR11523179
$ fastq-dump --gzip --outdir SRR11523179 SRR11523179/SRR11523179.sra
```

Little trick: You can download all SRA accessions at once using:
```markdown
$ prefetch $(<SraAccList.txt)
```
Where *SraAccList.txt* is a text file containing all SRA accessions, one per line:

```markdown
$ cat SraAccList.txt
SRR11523179
SRR11434959
SRR12801740
SRR11434956
SRR11434960
SRR11434954
```
You can use the same strategy for extracting the fastq files from the prefetched Runs in compressed SRA format using **fastq-dump**:

```markdown
$ fastq-dump --gzip $(<SraAccList.txt)
```
------------------
## Quick QC overview using SeqKit


```markdown
$ seqkit
SeqKit -- a cross-platform and ultrafast toolkit for FASTA/Q file manipulation

Version: 0.14.0

Author: Wei Shen <shenwei356@gmail.com>

Documents  : http://bioinf.shenwei.me/seqkit
Source code: https://github.com/shenwei356/seqkit
Please cite: https://doi.org/10.1371/journal.pone.0163962

Usage:
  seqkit [command]

Available Commands:
  amplicon        retrieve amplicon (or specific region around it) via primer(s)
  bam             monitoring and online histograms of BAM record features
  common          find common sequences of multiple files by id/name/sequence
  concat          concatenate sequences with same ID from multiple files
  convert         convert FASTQ quality encoding between Sanger, Solexa and Illumina
  duplicate       duplicate sequences N times
  faidx           create FASTA index file and extract subsequence
  fish            look for short sequences in larger sequences using local alignment
  fq2fa           convert FASTQ to FASTA
  fx2tab          convert FASTA/Q to tabular format (with length/GC content/GC skew)
  genautocomplete generate shell autocompletion script
  grep            search sequences by ID/name/sequence/sequence motifs, mismatch allowed
  head            print first N FASTA/Q records
  help            Help about any command
  locate          locate subsequences/motifs, mismatch allowed
  mutate          edit sequence (point mutation, insertion, deletion)
  pair            match up paired-end reads from two fastq files
  range           print FASTA/Q records in a range (start:end)
  rename          rename duplicated IDs
  replace         replace name/sequence by regular expression
  restart         reset start position for circular genome
  rmdup           remove duplicated sequences by id/name/sequence
  sample          sample sequences by number or proportion
  sana            sanitize broken single line fastq files
  scat            real time recursive concatenation and streaming of fastx files
  seq             transform sequences (revserse, complement, extract ID...)
  shuffle         shuffle sequences
  sliding         sliding sequences, circular genome supported
  sort            sort sequences by id/name/sequence/length
  split           split sequences into files by id/seq region/size/parts (mainly for FASTA)
  split2          split sequences into files by size/parts (FASTA, PE/SE FASTQ)
  stats           simple statistics of FASTA/Q files
  subseq          get subsequences by region/gtf/bed, including flanking sequences
  tab2fx          convert tabular format to FASTA/Q format
  translate       translate DNA/RNA to protein sequence (supporting ambiguous bases)
  version         print version information and check for update
  watch           monitoring and online histograms of sequence features
```


------------------
## FORMAT
You can use the [editor on GitHub](https://github.com/gabyrech/gabyrech.githhub.io/edit/gh-pages/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [Basic writing and formatting syntax](https://docs.github.com/en/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/gabyrech/gabyrech.githhub.io/settings/pages). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://docs.github.com/categories/github-pages-basics/) or [contact support](https://support.github.com/contact) and we’ll help you sort it out.
