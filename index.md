# Basic long-read sequencing QC 

This page cotains a tutorial for basic Quality Control (QC) of Long-Read sequencing data, mainly oriented to data produced by Oxford Nanopore Tenchnologies (**ONT**) and Pacific Biosciences (**PacBio**). The material was initially created for the Computational Sessions of the [2022 JAX Long Read Sequencing Workshop ](https://www.jax.org/education-and-learning/education-calendar/2022/may/long-read-sequencing-workshop) but I will try to keep it updated with new methods/strategies.

## **Table of Contents** <a name="TABLE"></a>

[**Tutorial Set Up**](#SETUP)
+ [**Unix**](#UNIX)
+ [**I don't have Unix, what do I do?**](#DONTHAVE)

------------------

## Tutorial Set Up <a name="SETUP"></a>

### Unix <a name="UNIX"></a>
Unix is the standard operating system used in scientific research. In fact, most up-to-date tools in bioiformatics, including the ones we are going to use in this tutorial, are run in Unix. In the context of this tutorial (and Bioiformatics in general) there are few aspects of Unix to highligth: 

- Unix is largely command line driven. 
- Unix is used by many of the most powerful computers at bioinformatics centres and also on many desktops and laptops (MacOS is largely UNIX compatible).
- Unix is free. 
- There are many distributions of Unix such as Ubuntu, RedHat, Fedora, etc. These are all Unix, but they bundle up extra software in a different way or combinations. 

There are seveal **really good** free on-line tutorials that introduce Unix and cover some of the basics that will allow you to be more comfortable with the command-line, here some of them:

1. [Software Carpentry Foundation Shell Novice Course](https://swcarpentry.github.io/shell-novice/)
2. [Bioinformatics Workbook Unix Basics](https://bioinformaticsworkbook.org/Appendix/Unix/unix-basics-1.html#gsc.tab=0)
3. [HadrienG Tutorial on Command Line](https://www.hadriengourle.com/tutorials/command_line/)

[Table of Contents](#TABLE)


### I don't have Unix, what do I do? <a name="DONTHAVE"></a>
In case you don’t work under any Unix distribution already, e.g. you have a Windows machine, one of the easiest way to start is by setting up a virtual machine with one of the most widely used Unix distributions: **Ubuntu**. This page gives detailed instructions on how to set up an Ubuntu environment in any computer using VirtualBox: [Install Ubuntu using a VM](https://ubuntu.com/tutorials/how-to-run-ubuntu-desktop-on-a-virtual-machine-using-virtualbox#1-overview) 

[Table of Contents](#TABLE)

### Tools
Tools needed for this tutorial and the instructions for installing them are:

- **SeqKit**: [https://bioinf.shenwei.me/seqkit/](https://bioinf.shenwei.me/seqkit/)
- **SeqFu**: [https://telatin.github.io/seqfu2/](https://telatin.github.io/seqfu2/)
- **NanoPack**:[https://github.com/wdecoster/nanopack](https://github.com/wdecoster/nanopack)
- **pycoQC**: [https://a-slide.github.io/pycoQC/](https://a-slide.github.io/pycoQC/)

[Table of Contents](#TABLE)

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

[Table of Contents](#TABLE)

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
[Table of Contents](#TABLE)

------------------

## Quick QC overview using SeqKit

[**SeqKit**](https://bioinf.shenwei.me/seqkit/) is an easy-to-install and easy-to-use toolkit for FASTA/Q file manipulation. 
Contain more than 37 commands [usages and examples](https://bioinf.shenwei.me/seqkit/#subcommands)


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

Among the many commands, we can use **_stats_** for obtaining simple statistics of FASTA/Q files. This could be our first approach to determining the quality of the sequencing data:

Assuming you downloaded the [sampleData.tar](https://thejacksonlaboratory.box.com/s/9ny2zvx3pby1yp3b775c9jraik9jerzp) file and extracted the fastq files in to the directory sampleData/:

```markdown
/full/path/to/data/sampleData$ ls
ONTLIG.fastq.gz        ONTRapid.fastq.gz   PacBioHiFi.fastq.gz
ONTLIGnoFrag.fastq.gz  PacBioCLR.fastq.gz  PacBioRSII.fastq.gz
```
You can run:

```
/full/path/to/data/sampleData$ seqkit stats --all --basename --tabular *
file	format	type	num_seqs	sum_len	min_len	avg_len	max_len	Q1	Q2	Q3	sum_gap	N50	Q20(%)	Q30(%)
ONTLIG.fastq.gz	FASTQ	DNA	10000	116811938	112	11681.2	122683	3530.0	5780.0	15452.0	0	22534	27.79	0.00
ONTLIGnoFrag.fastq.gz	FASTQ	DNA	10000	101400610	136	10140.1	248649	633.5	1609.57184.0	0	47119	17.97	6.38
ONTRapid.fastq.gz	FASTQ	DNA	10000	93959361	95	9395.9	206567	1482.5	4341.511488.0	0	20663	29.42	0.67
PacBioCLR.fastq.gz	FASTQ	DNA	10000	190168507	50	19016.9	103956	4615.0	14264.5	29390.0	0	33064	0.00	0.00
PacBioHiFi.fastq.gz	FASTQ	DNA	10000	129214761	1473	12921.5	18862	12292.5	12880.0	13535.5	0	12947	98.40	96.19
PacBioRSII.fastq.gz	FASTQ	DNA	10000	170749124	6	17074.9	57233	7802.0	17318.5	25082.5	0	24059	0.00	0.00
```
**Note**: In my command I use the '*' aterisk (also called star) character insthead of the full file names. The asterisk is wildcard character, which is replaced by any number of characters in a filename. In this way, I am telling SeqKit to parse all files in that directory, instead of typing all file names. 

**_seqkit stats --all_** command will provide the following information for each fastq file:

```
file: File name
format: File format Fastq/Fasta
type: Moleculte type: DNA/RNA/Protein
num_seqs: Total number of sequences (reads in the case of fastq)
sum_len: Total number of bases in the file
min_len: Shortest sequence in file
avg_len: Average sequence length
max_len: Largest sequence in file
Q1: Read length first quartile
Q2: Read length second quartile
Q3: Read length third quartile
sum_gap: Total number of gaps in the sequence file
N50: Length of the shortest sequence for which longer and equal length sequences cover at least 50% of total bases in file. 
Q20(%): Percentage of reads with average Quality Phred Score grater or equal to 20. 
Q30(%): Percentage of reads with average Quality Phred Score grater or equal to 30. 
```
[Table of Contents](#TABLE)

------------------

## Quick QC overview using SeqFu

[**SeqFu**](https://telatin.github.io/seqfu2/) is another general-purpose program to manipulate and parse information from FASTA/FASTQ files. In many ways is similar to SeqKit, but SeqFu author's claim their tool is four times faster than SeqKit on datasets with large sequences ([Telatin et al 2021](https://www.mdpi.com/2306-5354/8/5/59/htm)).

```markdown
$ seqfu
SeqFu - Sequence Fastx Utilities
version: 1.10.0

  · count [cnt]         : count FASTA/FASTQ reads, pair-end aware
  · deinterleave [dei]  : deinterleave FASTQ
  · derep [der]         : feature-rich dereplication of FASTA/FASTQ files
  · interleave [ilv]    : interleave FASTQ pair ends
  · lanes [mrl]         : merge Illumina lanes
  · list [lst]          : print sequences from a list of names
  · metadata [met]      : print a table of FASTQ reads (mapping files)
  · rotate [rot]        : rotate a sequence with a new start position
  · sort [srt]          : sort sequences by size (uniques)
  · stats [st]          : statistics on sequence lengths

  · cat                 : concatenate FASTA/FASTQ files
  · grep                : select sequences with patterns
  · head                : print first sequences
  · rc                  : reverse complement strings or files
  · tab                 : tabulate reads to TSV (and viceversa)
  · tail                : view last sequences
  · view                : view sequences with colored quality and oligo matches

Type 'seqfu version' or 'seqfu cite' to print the version and paper, respectively.
Add --help after each command to print its usage.
```

The command we can use for sumarizing sequence stats is, again: **stats**:

```markdown
/full/path/to/data/sampleData$ seqfu stats --nice --basename *
┌──────────────┬───────┬───────────┬─────────┬───────┬───────┬───────┬───────────┬──────┬────────┐
│ File         │ #Seq  │ Total bp  │ Avg     │ N50   │ N75   │ N90   │ auN       │ Min  │ Max    │
├──────────────┼───────┼───────────┼─────────┼───────┼───────┼───────┼───────────┼──────┼────────┤
│ ONTLIG       │ 10000 │ 116811938 │ 11681.2 │ 22534 │ 10980 │ 3602  │ 26051.972 │ 112  │ 122683 │
│ ONTLIGnoFrag │ 10000 │ 101400610 │ 10140.1 │ 47119 │ 20806 │ 5513  │ 52720.370 │ 136  │ 248649 │
│ ONTRapid     │ 10000 │ 93959361  │ 9395.9  │ 20663 │ 10055 │ 4590  │ 30500.710 │ 95   │ 206567 │
│ PacBioCLR    │ 10000 │ 190168507 │ 19016.9 │ 33064 │ 20675 │ 11360 │ 33436.589 │ 50   │ 103956 │
│ PacBioHiFi   │ 10000 │ 129214761 │ 12921.5 │ 12947 │ 12346 │ 11916 │ 4181.999  │ 1473 │ 18862  │
│ PacBioRSII   │ 10000 │ 170749124 │ 17074.9 │ 24059 │ 17718 │ 12266 │ 21843.122 │ 6    │ 57233  │
└──────────────┴───────┴───────────┴─────────┴───────┴───────┴───────┴───────────┴──────┴────────┘
```
Few things to highligth regarding **SeqFu** vs. **SeqKit**:
1. **SeqFu** parameter _--nice_ makes stats looking much better than the _--tabular_ in **SeqKit**.
2. **SeqFu** also calculates N75 and N90, which in some cases migth be useful. 
3. **SeqFu** does not provide Quality stats (e.g. Q20% / Q30%). In theory, **SeqFu** provides another command to sumarize quality scores: [_qual_](https://telatin.github.io/seqfu2/tools/qual.html), but it did not work for me.
4. **SeqFu** provides another interesting statistics based on the length of sequences called **auN**. auN is probably more interesting in the context of genome assembly. You can learn more about this measure [here](https://lh3.github.io/2020/04/08/a-new-metric-on-assembly-contiguity).

[Table of Contents](#TABLE)

------------------

## Comprehensive sequence evaluation using NanoPack

[**NanoPack**](https://github.com/wdecoster/nanopack) is a set of tools for visualization and processing of long-read sequencing data from Oxford Nanopore Technologies and Pacific Biosciences.

Among different tools **NanoPack** offers, we will use two that are very helpful for long-read QC: **NanoPlot** and **NanoComp**

- [**NanoPlot**](https://github.com/wdecoster/NanoPlot): Creates many relevant plots derived from reads (fastq), alignments (bam) and basecaller summary files. 

- [**NanoComp**](https://github.com/wdecoster/nanocomp): Compares multiple runs on read length and quality based on reads (fastq), alignments (bam) or albasecaller bacore summary files.


### NanoPlot


```markdown
$ NanoPlot --help

usage: NanoPlot [-h] [-v] [-t THREADS] [--verbose] [--store] [--raw] [--huge]
                [-o OUTDIR] [-p PREFIX] [--tsv_stats] [--maxlength N]
                [--minlength N] [--drop_outliers] [--downsample N]
                [--loglength] [--percentqual] [--alength] [--minqual N]
                [--runtime_until N] [--readtype {1D,2D,1D2}] [--barcoded]
                [--no_supplementary] [-c COLOR] [-cm COLORMAP]
                [-f {eps,jpeg,jpg,pdf,pgf,png,ps,raw,rgba,svg,svgz,tif,tiff}]
                [--plots [{kde,hex,dot,pauvre} [{kde,hex,dot,pauvre} ...]]]
                [--listcolors] [--listcolormaps] [--no-N50] [--N50]
                [--title TITLE] [--font_scale FONT_SCALE] [--dpi DPI]
                [--hide_stats]
                (--fastq file [file ...] | --fasta file [file ...] | --fastq_rich file [file ...] | --fastq_minimal file [file ...] | --summary file [file ...] | --bam file [file ...] | --ubam file [file ...] | --cram file [file ...] | --pickle pickle | --feather file [file ...])

CREATES VARIOUS PLOTS FOR LONG READ SEQUENCING DATA.

General options:
  -h, --help            show the help and exit
  -v, --version         Print version and exit.
  -t, --threads THREADS
                        Set the allowed number of threads to be used by the script
  --verbose             Write log messages also to terminal.
  --store               Store the extracted data in a pickle file for future plotting.
  --raw                 Store the extracted data in tab separated file.
  --huge                Input data is one very large file.
  -o, --outdir OUTDIR   Specify directory in which output has to be created.
  -p, --prefix PREFIX   Specify an optional prefix to be used for the output files.
  --tsv_stats           Output the stats file as a properly formatted TSV.

Options for filtering or transforming input prior to plotting:
  --maxlength N         Hide reads longer than length specified.
  --minlength N         Hide reads shorter than length specified.
  --drop_outliers       Drop outlier reads with extreme long length.
  --downsample N        Reduce dataset to N reads by random sampling.
  --loglength           Additionally show logarithmic scaling of lengths in plots.
  --percentqual         Use qualities as theoretical percent identities.
  --alength             Use aligned read lengths rather than sequenced length (bam mode)
  --minqual N           Drop reads with an average quality lower than specified.
  --runtime_until N     Only take the N first hours of a run
  --readtype {1D,2D,1D2}
                        Which read type to extract information about from summary. Options are 1D, 2D,
                        1D2
  --barcoded            Use if you want to split the summary file by barcode
  --no_supplementary    Use if you want to remove supplementary alignments

Options for customizing the plots created:
  -c, --color COLOR     Specify a valid matplotlib color for the plots
  -cm, --colormap COLORMAP
                        Specify a valid matplotlib colormap for the heatmap
  -f, --format {eps,jpeg,jpg,pdf,pgf,png,ps,raw,rgba,svg,svgz,tif,tiff}
                        Specify the output format of the plots.
  --plots [{kde,hex,dot,pauvre} [{kde,hex,dot,pauvre} ...]]
                        Specify which bivariate plots have to be made.
  --listcolors          List the colors which are available for plotting and exit.
  --listcolormaps       List the colors which are available for plotting and exit.
  --no-N50              Hide the N50 mark in the read length histogram
  --N50                 Show the N50 mark in the read length histogram
  --title TITLE         Add a title to all plots, requires quoting if using spaces
  --font_scale FONT_SCALE
                        Scale the font of the plots by a factor
  --dpi DPI             Set the dpi for saving images
  --hide_stats          Not adding Pearson R stats in some bivariate plots

Input data sources, one of these is required.:
  --fastq file [file ...]
                        Data is in one or more default fastq file(s).
  --fasta file [file ...]
                        Data is in one or more fasta file(s).
  --fastq_rich file [file ...]
                        Data is in one or more fastq file(s) generated by albacore, MinKNOW or guppy
                        with additional information concerning channel and time.
  --fastq_minimal file [file ...]
                        Data is in one or more fastq file(s) generated by albacore, MinKNOW or guppy
                        with additional information concerning channel and time. Is extracted swiftly
                        without elaborate checks.
  --summary file [file ...]
                        Data is in one or more summary file(s) generated by albacore or guppy.
  --bam file [file ...]
                        Data is in one or more sorted bam file(s).
  --ubam file [file ...]
                        Data is in one or more unmapped bam file(s).
  --cram file [file ...]
                        Data is in one or more sorted cram file(s).
  --pickle pickle       Data is a pickle file stored earlier.
  --feather file [file ...]
                        Data is in one or more feather file(s).

EXAMPLES:
    NanoPlot --summary sequencing_summary.txt --loglength -o summary-plots-log-transformed
    NanoPlot -t 2 --fastq reads1.fastq.gz reads2.fastq.gz --maxlength 40000 --plots hex dot
    NanoPlot --color yellow --bam alignment1.bam alignment2.bam alignment3.bam --downsample 10000

```

Run examples for two of the libraries:

```
$ NanoPlot --fastq sampleData/ONTLIGnoFrag.fastq.gz --N50 -o NanoPlotONTLIGnoFrag
$ NanoPlot --fastq sampleData/PacBioHiFi.fastq.gz --N50 -o NanoPlotPacBioHiFi
```

You can check the results for sampled data [**ONTLIGnoFrag**](https://htmlpreview.github.io/?https://github.com/gabyrech/LongReadsQC/blob/gh-pages/NanoPlotONTLIGnoFrag-report.html) and [**PacBioHiFi**](https://htmlpreview.github.io/?https://github.com/gabyrech/LongReadsQC/blob/gh-pages/NanoPlotPacBioHiFi-report.html)


### NanoComp

Use **NanoComp** if you want to compare statistics between different sets of sequences. You can use different input file formats (fastq, fasta, bam, summary files and more...)

```markdown
$ NanoComp --help

usage: NanoComp [-h] [-v] [-t THREADS] [-o OUTDIR] [-p PREFIX] [--verbose]
                [--raw] [--store] [--tsv_stats] [--readtype {1D,2D,1D2}]
                [--maxlength N] [--minlength N] [--barcoded]
                [--split_runs TSV_FILE]
                [-f {eps,jpeg,jpg,pdf,pgf,png,ps,raw,rgba,svg,svgz,tif,tiff}]
                [-n names [names ...]] [-c colors [colors ...]]
                [--plot {violin,box,ridge,false}] [--title TITLE] [--dpi DPI]
                (--fasta file [file ...] | --fastq files [files ...] | --summary files [files ...] | --bam files [files ...] | --ubam file [file ...] | --cram file [file ...] | --pickle file [file ...] | --feather file [file ...])

Compares long read sequencing datasets.

General options:
  -h, --help            show the help and exit
  -v, --version         Print version and exit.
  -t, --threads THREADS
                        Set the allowed number of threads to be used by the script
  -o, --outdir OUTDIR   Specify directory in which output has to be created.
  -p, --prefix PREFIX   Specify an optional prefix to be used for the output files.
  --verbose             Write log messages also to terminal.
  --raw                 Store the extracted data in tab separated file.
  --store               Store the extracted data in a pickle file for future plotting.
  --tsv_stats           Output the stats file as a properly formatted TSV.

Options for filtering or transforming input prior to plotting:
  --readtype {1D,2D,1D2}
                        Which read type to extract information about from summary. Options are 1D, 2D,
                        1D2
  --maxlength N         Drop reads longer than length specified.
  --minlength N         Drop reads shorter than length specified.
  --barcoded            Barcoded experiment in summary format, splitting per barcode.
  --split_runs TSV_FILE
                        File: Split the summary on run IDs and use names in tsv file. Mandatory header
                        fields are 'NAME' and 'RUN_ID'.

Options for customizing the plots created:
  -f, --format {eps,jpeg,jpg,pdf,pgf,png,ps,raw,rgba,svg,svgz,tif,tiff}
                        Specify the output format of the plots.
  -n, --names names [names ...]
                        Specify the names to be used for the datasets
  -c, --colors colors [colors ...]
                        Specify the colors to be used for the datasets
  --plot {violin,box,ridge,false}
                        Which plot type to use: 'box', 'violin' (default), 'ridge' (joyplot) or 'false'
                        (no plots)
  --title TITLE         Add a title to all plots, requires quoting if using spaces
  --dpi DPI             Set the dpi for saving images

Input data sources, one of these is required.:
  --fasta file [file ...]
                        Data is in (compressed) fasta format.
  --fastq files [files ...]
                        Data is in (compressed) fastq format.
  --summary files [files ...]
                        Data is in (compressed) summary files generated by albacore or guppy.
  --bam files [files ...]
                        Data is in sorted bam files.
  --ubam file [file ...]
                        Data is in one or more unmapped bam file(s).
  --cram file [file ...]
                        Data is in one or more sorted cram file(s).
  --pickle file [file ...]
                        Data is in one or more pickle file(s) from using NanoComp/NanoPlot.
  --feather file [file ...]
                        Data is in one or more feather file(s).

EXAMPLES:
    NanoComp --bam alignment1.bam alignment2.bam --outdir compare-runs
    NanoComp --fastq reads1.fastq.gz reads2.fastq.gz reads3.fastq.gz  --names run1 run2 run3
```

In our case, we can use the fastq files to compare these datasets. You can try to run **NanoComp** using the [sampleData.tar](https://thejacksonlaboratory.box.com/s/9ny2zvx3pby1yp3b775c9jraik9jerzp) dataset. The command should be something like this:

```
$ NanoComp -t 8 --fastq sampleData/ONTRapid.fastq.gz \
  sampleData/ONTLIG.fastq.gz sampleData/ONTLIGnoFrag.fastq.gz \
  sampleData/PacBioRSII.fastq.gz sampleData/PacBioCLR.fastq.gz \
  sampleData/PacBioHiFi.fastq.gz -o NanoCompSampleData \
  --names ONTRapid ONTLIG ONTLIGnoFrag PacBioRSII PacBioCLR PacBioHiFi
```

I run **NanoComp** for the full set of sequences downloaded from SRA using this command:

```
$ NanoComp -t 8 --fastq SRR11523179/SRR11523179.fastq.gz \
  SRR11434959/SRR11434959.fastq.gz SRR12801740/SRR12801740.fastq.gz \
  SRR11434956/SRR11434956.fastq.gz SRR11434960/SRR11434960.fastq.gz \
  SRR11434954/SRR11434954.fastq.gz -o NanoCompBox \
  --names ONTRapid ONTLIG ONTLIGnoFrag PacBioRSII PacBioCLR PacBioHiFi --plot 'box' --dpi 80
```

**You can check the results here:** [NanoComp Results for fastq files download from SRA](https://htmlpreview.github.io/?https://github.com/gabyrech/LongReadsQC/blob/gh-pages/NanoCompBox/NanoComp-report.html)



[Table of Contents](#TABLE)

------------------

## Comprehensive sequence evaluation using pycoQC

[**pycoQC**](https://a-slide.github.io/pycoQC/)







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
