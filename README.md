# FEELnc 
 Fast and Effective Extraction of Long non-coding RNAs

*Version (18/02/2015)*

## Introduction

This document is intended to give a (minimal) description of the FEELnc pipeline in order to annotate long non-coding RNAs (lncRNAs).

Currently, FEELnc is composed of 3 modules (See *Launch FEELnc 3-step pipeline* for more details):

	* FEELnc_filter.pl	: Extract, filter candidate transcripts
	* FEELnc_codpot.pl	: Compute the coding potential of candidate transcripts
	* FEELnc_classifier.pl: Classify lncRNAs based on their genomic localization wrt mRNAs 


To get help on each module, you can type :

	FEELnc_filter.pl --help
	# Or
    FEELnc_filter.pl --man 


## Input files

The formats used to describe genes, transcripts, exon is **.GTF** and **.FASTA** for genome file. 

Basically, FEELnc users should have the following minimal input files:

	- Infile.GTF          (-i,--infile)   : input GTF file (e.g cufflinks transcripts.GTF)
	- ref_annotation.GTF  (-a,--mRNAfile) : GTF annotation file*
	- ref_genome.FASTA    (-g,--genome)   : genome FASTA file or directory with individual chrom FASTA files


\* *Note: It is recommended to only extract protein_coding transcripts (mRNAs) from the reference annotation file (ref_annotation.GTF) when this information is available, either manually or better by using the option :*
**--biotype transcript_biotype=protein_coding**. 
In doing so, you will not remove lncRNAs overlapping other non-coding RNAs or pseudogenes.... 
   
 
-------------------------
## Installation and requirements


### Requirements
	
The following software and libraries must be installed on your machine:

- [Perl5+](https://www.perl.org/) : tested with version 5.18.2
 * [Bioperl](http://www.bioperl.org/wiki/Main_Page)  : tested with version BioPerl-1.6.924
 * [Parralell::ForkManager](http://search.cpan.org/~szabgab/Parallel-ForkManager-1.07/lib/Parallel/ForkManager.pm) : tested with version 1.07
- R [Rscript](http://cran.r-project.org): tested with version 3.1.0.
 * [ROCR](https://rocr.bioinf.mpi-sb.mpg.de/) R library (type "install.packages('ROCR')" in a R session)
- [CPAT tool](http://rna-cpat.sourceforge.net/#installation): Coding Potential Assessment Tool: tested with version 1.2.2. 


### Installation

Clone the FEELnc git:

	git clone https://github.com/tderrien/FEELnc.git

Go to FEELnc directory

	cd FEELnc

export PERL5LIB, FEELNCPATH and add it to your PATH

	export PERL5LIB=${PWD}/lib/
	export FEELNCPATH=${PWD}
	export PATH=$PATH:$FEELNCPATH/scripts/

### Test with toy example:

	cd test/
	
	# Filter
	FEELnc_filter.pl -i transcript_chr38.gtf -a annotation_chr38.gtf \
    -b transcript_biotype=protein_coding > candidate_lncRNA.gtf


	# Coding_Potential
    # Note1 :  as a test, the training is only done on  100 tx (-n 100 option)
	FEELnc_codpot.pl -i candidate_lncRNA.gtf -a annotation_chr38.gtf -g genome_chr38.fa -n 100 
    
    # Classifier
	FEELnc_classifier.pl -i candidate_lncRNA.gtf.lncRNA.gtf -a annotation_chr38.gtf > candidate_lncRNA_classes.txt


-------------------------
## Launch the 3-step pipeline

### 1- FEELnc_filter.pl

The first step of the pipeline (FEELnc_filter) consists in filtering out unwanted/spurious transcripts and/or transcripts overlapping (in sense) exons of the reference annotation 
and especially protein_coding exons as they more probably correspond to new mRNA isoforms (see -b,--biotype option).

	# Usage:
    FEELnc_filter.pl -i infile.gtf -a annotation_mRNA.gtf > candidate_lncRNA.gtf


If your reference annotation ("*ref_annotation.GTF*") contains transcript_biotype information (e.g protein_coding, pseudogene, miRNA...), you can subselect a specific transcript biotype to make the overlap with.

    FEELnc_filter.pl -i infile.gtf \
	-a ref_annotation.GTF \
	-b transcript_biotype=protein_coding \
	> candidate_lncRNA.gtf

This option is highly recommended if you don't want to remove transcripts 
overlapping with other transcripts than mRNAs (e.g lincRNA, miRNA, pseudogene...).
For stranded RNASeq protocol, it is also possible  to include monoexonic lncRNAs that are antisense to mRNAs e.g 

	FEELnc_filter.pl -i infile.gtf \
	-a ref_annotation.GTF \
	-b transcript_biotype=protein_coding \
	--monoex=-1
	> candidate_lncRNA.gtf


**- FULL OPTIONS (FEELnc_filter.pl --help) :**
```
  * General:
      --help                Print this help
      --man                 Open man page
      --verbosity           Level of verbosity

  * Mandatory arguments:
      -i,--infile=file.gtf          Specify the GTF file to be filtered (such as a cufflinks transcripts/merged .GTF file) 
      -a,--mRNAfile=file.gtf        Specify the annotation GTF file to be filtered on based on sense exon overlap (file of protein coding annotation or whole reference annotation 'ref_annotation.GTF')

  * Filtering arguments:
      -s,--size=200                 Keep transcript with a minimal size (default 200)
      -b,--biotype                  Only consider transcript(s) from the reference annotation having this(these) biotype(s) (e.g : -b transcript_biotype=protein_coding,pseudogene) [default undef i.e all transcripts]
      -l,--linconly                 Keep only long intergenic/interveaning ncRNAs [default FALSE]. 
      --monoex=-1|0|1               Keep monoexonic transcript(s): mode to be selected from : -1 keep monoexonic antisense (for RNASeq stranded protocol), 1 keep all monoexonic, 0 remove all monoexonic   [default 0]
      --biex=25                     Discard biexonic transcripts having one exon size lower to this value (default 25)

  * Overlapping specification:
      -f,--minfrac_over=0           minimal fraction out of the candidate lncRNA size to be considered for overlap [default 0 i.e 1nt]
      -p,--proc=4                   number of thread for computing overlap [default 4]

  * Log output:
      -o,--outlog=file.log          Specify the log file of output which [default infile.log]

```


### 2- FEELnc_codpot.pl

The second step of the pipeline (FEELnc_codpot) aims at computing the CPS i.e the coding potential score (between [0-1]) foreach of the candidate transcripts in the candidate_lncRNA.gtf file.

**- INPUT :**
 
It makes use of the CPAT tool which is an alignment-free program (thus very fast) which relies on  intrinsic properties of the fasta sequences of  two training files:
	
	- known_mRNA.gtf   : a set of known protein_coding transcripts
	- known_lncRNA.gtf : a set of known lncRNA transcripts
    
If you have a set of known lncRNAs, you could run the module like:

	FEELnc_codpot.pl -i candidate_lncRNA.gtf -a known_mRNA.gtf -l known_lncRNA.gtf

However, for most organisms, the set of known_lncRNA transcripts is not known and thus 
a set of genomic intergenic regions are automatically extracted as the lncRNA training set. 
In this case, the reference genome file is required (ref_genome.FA)

    FEELnc_codpot.pl -i candidate_lncRNA.gtf -a known_mRNA.gtf -g ref_genome.FA 


As in the previous module, if your reference annotation file  ("*ref_annotation.GTF*") contains additionnal fields such **transcript_biotype** and/or **transcript_status** in the [GENCODE annotation](http://www.gencodegenes.org/gencodeformat.html) or [ENSEMBL](http://www.ensembl.org), you can extract them manually or by using the **-b option** (as  to get the best training set of known mRNAs.

    FEELnc_codpot.pl -i candidate_lncRNA.gtf -a ref_annotation.GTF \
    -g ref_genome.FA \
    -b transcript_biotype=protein_coding -b transcript_status=KNOWN



To calculate the CPS cutoff separating coding (mRNAs) versus long non-coding RNAs (lncRNAs), 
FEELnc_codpot uses a R script that will make a 10 fold cross-validation on the input training files and finally,  extracts the CPS that maximizes sensitivity (Sn) and Specificity (Sp) (thanks to the ROCR library)


**- OUTPUT :**

If your input file is called **INPUT**, this second module will create 4 output files:
 
 	- INPUT.cpat		: gathering all CPAT metric together with the CPS for all input tx
	 - INPUT.Cutoff.png  : the .png image of the Two Graphic ROC curves to determine the optimal CPS cutoff value.
     - INPUT.lncRNA.gtf  : a .GTF file of the transcripts below the CPS (Your final set of lncRNAs)
     - INPUT.mRNA.gtf    : a .GTF file of the transcripts above the CPS (a a priori new set of mRNAs)
     
**- FULL OPTIONS (FEELnc_codpot.pl --help) :**
 
```
  * General:
      --help                Print this help
      --man                 Open man page
      --verbosity           Level of verbosity

  * Mandatory arguments:
      -i,--infile=file.gtf/.fasta           Specify the .GTF or .FASTA file  (such as a cufflinks transcripts/merged .GTF or .FASTA file) 
      -a,--mRNAfile=file.gtf/.fasta         Specify the annotation .GTF or .FASTA file  (file of protein coding transcripts .GTF or .FASTA file)

  * Optional arguments:
      -g,--genome=genome.fa                         genome file or directory with chr files (mandatory if input is .GTF) [ default undef ]
      -l,--lncRNAfile=file.gtf/.fasta       specify a known set of lncRNA for training .GTF or .FASTA  [ default undef ]
      -b,--biotype                  only consider transcripts having this(these) biotype(s) from the reference annotation (e.g : -b transcript_biotype=protein_coding,pseudogene) [default undef i.e all transcripts]
      -n,--numtx=2000               Number of transcripts required for the training [default 2000 ]
      -c,--cpatcut=[0-1]                    CPAT coding potential cutoff [default undef i.e will compute best cutoff]

  *Intergenic lncrna extraction:
            -to be added

```

### 3- FEELnc_classifier.pl

The last step of the pipeline consists in classifying new lncRNAs w.r.t to the localisation and the direction of transcription of proximal mRNAs.

Indeed, classifying lncRNAs with mRNA could help to predict functions for lncRNAs.
For all newly identified lncRNAs transcripts, a sliding window strategy is used to check for possible overlap with transcripts in the reference annotation.

If an overlap is found, the lncRNAs is considered as **GENIC** otherwise it is **INTERGENIC** (lincRNA) . 

Then, subclasses are defined according the features and direction of overlap (See OUTPUT for full details). 

Foreach lncRNA interaction, a best lncRNA:mRNA interaction is identified in the output file by a line starting with '*' and defined as followed:

 - for **INTERGNIC** : the best mRNA partner is the closest to the lincRNA
 - for **GENIC**	: the best mRNA partner is by rule of priority (exonic then the fraction of exonicof overlap) then intronic then containing.


```
	FEELnc_classifier.pl -i lncRNA.gtf -a  ref_annotation.GTF > lncRNA_classes.txt
```


**- OUTPUT :**


A summary of the number of the input parameters and numbers of interactions is given in the beginning of the file.

The classes are defined as in Derrien et al, Genome Research. 2012, and can be prioritized according to :
 
- **Intergenic lncRNAs** (i.e lincRNAs)
 - *divergent*  : when the lincRNA is transcribed in an opposite direction (head to head) w.r.t to the closest mRNA
 - *convergent*: when the lincRNA is transcribed in a convergent direction w.r.t to the closest mRNAs.
 - *same_strand*: when the lincRNA is transcribed in a same starnd w.r.t to the closest mRNA

- **Genic lncRNAs** (lncRNAs overlapping mRNAs)
 - *Exonic* :
    - antisense : at least one lncRNA exon overlaps in antisense an mRNA exon
    - sense : there should not be since there are filtered in the first step
 - *Intronic* :
    - antisense : lncRNA exon overlaps in antisense mRNA introns (but none exons)
    - sense : lncRNA exon overlaps in sense mRNA introns (but none exons)
 - *containing*:
    - antisense : lncRNA intron overlaps antisense mRNA     
    - sense : lncRNA intron overlaps sense mRNA exons


Example:

```
#FEELnc Classification
#lncRNA file :  lncrna : candidate_lncRNA.gtf.lncRNA.gtf 
#mRNA file : REF_ANNOTATION.gtf
#Minimal window size : 10000
#Maximal window size : 10000
#Number of lncRNA : 3027 
#Number of mRNA : 17954
#Number of interaction : 2132 
#Number of lncRNA without interaction : 1819
#List of lncRNA without interaction : (...)
#INTERACTIONS
*lncRNA        TCONS_00018990  mRNA         ENS00000029020            sense        genic                            0      Status=containing   Subtype=none
lncRNA         TCONS_00018990  mRNA         ENS00000001947            antisense    intergenic                       5441   Status=divergent    Subtype=downstream
*lncRNA        TCONS_00027880  mRNA         ENS00000007999            antisense    genic                            0      Status=overlapping  Subtype=exonic
*lncRNA        TCONS_00011066  mRNA         ENS00000002091            antisense    intergenic                       9386   Status=convergent   Subtype=upstream
```

Here is showed 4 interactions concerning 3 lncRNAs (TCONS_00018990, TCONS_00027880 and TCONS_00011066) where one lncRNA (TCONS_00018990) has 2 interactions with a window size of 10,000 nt.
(The best interactions are marked as ***lncRNA**)

The 3 best interactions are of classes and sub-classes:

	- genic      -> containing -> sense       : TCONS_00018990::ENS00000029020
	- genic      -> exonic     -> antrisense  : TCONS_00027880::ENS00000007999
	- intergenic -> convergent -> upstream    : TCONS_00011066::ENS00000002091 (  with a distance between the lncRNA and the mRNA of 9,386bp.)
	
\* **Note1**: At the moment, the interactions are computed with the reference file (-a option).
Therefore, the possibly newly identified mRNAs in the previous step are not included by default (but you could include them by (cuff)merging with you reference annotation).

\* **Note2**:  you may see a warning message like this:

	lncRNA_ID and ENSXXXX are overlapping in the same strand
Depending on your filtering options, this may correspond to a non-protein-coding transcript (pseudogene, miRNA) which overlaps the lncRNA
	
**- FULL OPTIONS (FEELnc_classifier.pl --help) :**

``` 
    General:
      --help                Print this help
      --man                 Open man page
      --verbosity           Level of verbosity

    Mandatory arguments:
      -i,--lncrna=file.gtf Specify the lncRNA GTF file 
      -a,--mrna=file.gtf    Specify the annotation GTF file (file of protein coding annotaion)

    Filtering arguments:
      -w,--window=200               Size of the window around the lncRNA to compute interactins/classification [default 10000]
      -m, --maxwindow=10000 Maximal size of the window during the expansion process [default 10000]
```

## Warnings
 
 - In the installation (and/or) the classifier step, you may see a warning like 
```
Can't call method "close" on an undefined value Bio/DB/SeqFeature/Store/berkeleydb.pm
```

## Authors

 - Fabrice Legeai
 - Thomas Derrien

## Acknowledgments

