# IsoSeq: PacBio Isoform Sequencing.

The Isoform Sequencing (IsoSeq) application refers to [PacBio’s proprietary methods and applications for transcriptome sequencing](http://www.pacb.com/applications/rna-sequencing/). The IsoSeq application generates full-length cDNA sequences — from the 5’ end of transcripts to the poly-A tail — eliminating the need for transcriptome reconstruction using isoform-inference algorithms. The Iso-Seq method generates accurate information about alternatively spliced exons and transcriptional start sites. It also delivers information about poly-adenylation sites for transcripts up to 10 kb in length across the full complement of isoforms within targeted genes or the entire transcriptome.

This document describes the IsoSeq application in SMRTAnalysis v3.0 release, which includes SMRTLink v1.0. Previous IsoSeq documents can be found in its [github wiki](https://github.com/PacificBiosciences/cDNA_primer/wiki).

Table of contents
=================

  * [Overview](#overview)
  * [Manual](#manual)
    * [Running with SMRTAnalysis](#running-with-smrtanalysis)
    * [Running on the Command-Line](#running-on-the-command-line)
    * [Running on the Command-Line with pbsmrtpipe](#running-on-the-command-line-with-pbsmrtpipe)
  * [Advanced Analysis Options](#advanced-analysis-options)
    * [SMRTLink/pbsmrtpipe IsoSeq Options](#smrtlinkpbsmrtpipe-isoseq-options)
    * [Classify Options](#classify-options)
    * [Cluster Options](#cluster-options)
    * [Subset Options](#subset-options)
  * [Output Files](#output-files)
    * [Classify Output Files](#classify-output-files)
    * [Cluster Output Files](#cluster-output-files)
  * [Algorithm Modules](#algorithm-modules)
  * [Diff SMRTAnalysis v3.0 vs v2.3](#diff-smrtanalysis-v30-vs-v23)
  * [Handling RS and RS II data](#handling-rs-and-rs-ii-data)
  * [Glossary](#glossary)

try the first time
## Overview

![isoseq expanded](https://cloud.githubusercontent.com/assets/12494820/11380910/c9775812-92ad-11e5-97e3-5c3849ce6fea.png)

Analyses are performed in three stages, CCS, Classify and Cluster. Cluster employs the Iterative Clustering and Error correction (ICE) algorithm. For analyses performed on the command-line, there is an optional tool, Subset, for subsetting the IsoSeq results.
* __CCS__
  * CCS is the first stage of an IsoSeq analysis. CCS builds circular consensus sequences (CCSs) from your subreads. More information about CCS is available  [here](https://github.com/PacificBiosciences/pbccs/blob/master/README.md).
* __Classify__
  * Classify is the second stage of an IsoSeq analysis. The key output of Classify is a file of full-length non-chimeric reads, and a file of non-full length reads. The key input of Classify is the circular consensus sequences generated from CCS. Classify will identify and remove polyA/T tails, remove primers, and identify read strandedness. Classify also removes artificial concatemers, but does not remove PCR chimeras. 
* __Cluster__
  * Cluster is the third stage of an IsoSeq analysis. The key outputs of Cluster is a file of polished, high-quality consensus sequences, and a file of polished, low-quality consensus sequences. The key input of clustering is the file of full-length non-chimeric reads, and a file of non-full length reads outputted by Classify. 
* __Subset__
  * Subset is an optional program which can be used to subset the output files for particular classes of sequences, such as non-chimeric reads, or non-full-length reads.

##Manual

There are three ways to run IsoSeq: Using SMRTLink, on the command line, and on the command line using pbsmrtpipe so that you can run the whole IsoSeq analysis with one command given to pbsmrtpipe. 

##Running with SMRTAnalysis

To run Isoseq using SMRTAnalysis, follow the usual steps for analysing data on SMRTAnalysis. TODO: Link to document explaining SMRTAnalysis. 

##Running on the Command Line

On the command line, the analysis is performed in 3 steps:

1. Run CCS on your subreads, generating a CCS BAM file. Then generate an XML from the BAM file.
2. Run Classify on your CCSs with the XML as input, generating a FASTA of annotated sequences.
3. Run Cluster on the FASTA produced by Classify, generating polished isoforms. 

__Step 1. CCS__

First, convert your subreads to circular consensus sequences. You can do this with the command:

     ccs --minLength=300 --minPasses=1 --minZScore=-999 --maxDropFraction=0.8 --minPredictedAccuracy=0.8 --minSnr=4 ccs.bam subreads.bam

Where ccs.bam is where the CCSs will be output, and subreads.bam is the file containing your subreads. CCS options are described in [pbccs doc](https://github.com/PacificBiosciences/pbccs/blob/master/README.md). If you think that you have transcripts of interest that are less than 300 bp in length, be sure to adjust the `minLength` parameter. 
Next, you will generate an XML file from your CCSs. You can do this with the commmand:

     dataset create --type ConsensusReadSet ccs.xml ccs.bam

Where `ccs.xml` is the name of the XML file you are generating and `ccs.bam` is the name of the BAM file you generated previously using the `ccs` command. 

__Step 2. Classify__

Classify can be run at the command line as follows:

     pbtranscript classify [OPTIONS] ccs.xml isoseq_draft.fasta --flnc=isoseq_flnc.fasta --nfl=isoseq_nfl.fasta
 
Where `ccs.xml` is the XML file you generated in Step 1.

Where `isoseq_flnc.fasta` contains only the full-length, non-chimeric reads.

And where `isoseq_nfl.fasta` contains all non-full-length reads.
 
 Or you can run classify creating XML files instead of FASTA files as follows:
 
     pbtranscript classify [OPTIONS] ccs.xml isoseq_draft.fasta --flnc=isoseq_flnc.contigset.xml --nfl=isoseq_nfl.contigset.xml

Where `ccs.xml` is the XML file you generated in Step 1.

Where `isoseq_flnc.contigset.xml` contains only the full-length, non-chimeric reads.

And where `isoseq_nfl.contigset.xml` contains all non-full-length reads.

**Note**: One can always use `pbtranscript subset` to further subset `isoseq_draft.fasta` if `--flnc` and `--nfl` are not specified when you run `pbtranscript classify`. For example:

    pbtranscript subset isoseq_draft.fasta isoseq_flnc.fasta --FL --nonChimeric

__Step 3. Cluster and Polish__

`cluster` can be run at the command line as follows:

     pbtranscript cluster [OPTIONS] isoseq_flnc.fasta polished_clustered.fasta --quiver --nfl=isoseq_nfl.fasta --bas_fofn=my.subreadset.xml

Or

     pbtranscript cluster [OPTIONS] isoseq_flnc.contigset.xml polished_clustered.contigset.xml --quiver --nfl=isoseq_nfl.contigset.xml --bas_fofn=my.subreadset.xml

**Note**: `--quiver --nfl=isoseq_nfl.fasta|contigset.xml` must be specified in order to get Quiver polished consensus isoforms.

Optionally, you may call the following command to run ICE and create unpolished consensus isoforms only.

     pbtranscript cluster [OPTIONS] isoseq_flnc.fasta unpolished_clustered.fasta


##Running on the Command-Line with pbsmrtpipe
###Install pbsmrtpipe
pbsmrtpipe is a part of `smrtanalysis-3.0` package and will be installed
if `smrtanalysis-3.0` has been installed on your system. Or you can [download   pbsmrtpipe](https://github.com/PacificBiosciences/pbsmrtpipe) and [install](http://pbsmrtpipe.readthedocs.org/en/master/).
    
You can verify that pbsmrtpipe is running OK by:

    pbsmrtpipe --help

### Create a dataset
Now create an XML file from your subreads.

```
dataset create --type SubreadSet my.subreadset.xml subreads1.bam subreads2.bam ...
```
This will create a file called `my.subreadset.xml`. 


### Create and edit isoseq options and global options for `pbsmrtpipe`.
Create a global options XML file which contains SGE related, job chunking and
job distribution options that you may modify by:

```
 pbsmrtpipe show-workflow-options -o global_options.xml
```

Create an isoseq options XML file which contains isoseq related options that 
you may modify by:
```
 pbsmrtpipe show-template-details pbsmrtpipe.pipelines.sa3_ds_isoseq -o isoseq_options.xml
```

The entries in the options XML files have the format:

```
 <option id="pbtranscript.task_options.min_seq_len">
            <value>300</value>
        </option>
```

**Note**: If you only want to run IsoSeq Classify without Cluster, please 
create an XML for IsoSeq Classify Only.

```
 pbsmrtpipe show-template-details pbsmrtpipe.pipelines.sa3_ds_isoseq_classify -o isoseq_classify_options.xml
```

And you can modify options using your favorite text editor, such as vim.

### Run IsoSeq from pbsmrtpipe
Once you have set your options, you are ready to run isoseq via pbsmrtpipe:

```
pbsmrtpipe pipeline-id pbsmrtpipe.pipelines.sa3_ds_isoseq -e eid_subread:my.subreadset.xml --preset-xml=isoseq_options.xml --preset-xml=global_options.xml
```

## Advanced Analysis Options

## SMRTLink/pbsmrtpipe IsoSeq Options

You may modify IsoSeq advanced analysis parameters for SMRTLink or pbsmrtpipe as follows. 

| Module |           Parameter           | pbsmrtpipe name |     Default      |  Explanation      |
| ------ | -------------------------- | ------- | --------------------------- | ----------------- |
| CCS | Max. dropped fraction  | max_drop_fraction | 0.08  | Maximum fraction of subreads that can be dropped before giving up |
| CCS | Minimum length | min_length | 300 | Sets a minimum length requirement for the median size of insert reads in order to generate a consensus sequence. If the targeted template is known to be a particular size range, this can filter out alternative DNA templates. |
| CCS | Minimum Number of Passes | min_passes | 1 | Sets a minimum number of passes for a ZMW to be emitted. This is the number of full passes. Full passes must have an adapter hit before and after the insert sequence and so does not include any partial passes at the start and end of the sequencing reaction. Additionally, the full pass count does not include any reads that were dropped by the Z-Filter. |
| CCS | Minimum Predicted Accuracy | min_predicted_accuracy | 0.8 | The minimum predicted accuracy of a read. CCS generates an accuracy prediction for each read, defined as the expected percentage of matches in an alignment of the consensus sequence to the true read. A value of 0.99 indicates that only reads expected to be 99% accurate are emitted. |
| CCS | Minimum read score | min_read_score | 0.75 | Minimum read score of input subreads |
| CCS | Minimum SNR | min_snr | 4 | This filter removes data that is likely to contain deletions. SNR is a measure of the strength of signal for all 4 channels (A, C, G, T) used to detect basepair incorporation. The SNR can vary depending on where in the ZMW a SMRTbell stochastically lands when loading occurs. SMRTbells that land near the edge and away from the center of the ZMW have a less intense signal, and as a result can contain sequences with more "missed" basepairs. This value sets the threshold for minimum required SNR for any of the four channels. Data with SNR < 3.75 is typically considered lower quality. |
| CCS | Minimum Z Score | min_zscore | -9999 | The minimum Z-Score for a subread to be included in the consensus generating process. |
| Classify | Ignore polyA | ignore_polya | FALSE | FL does not require polyA tail (default: turned off) |
| Classify | Min. seq. length | min_seq_len | 300 | Minimum sequence length to output (default: 300) |
| Cluster | Minimum Quiver Accuracy | hq_quiver_min_accuracy | 0.99 | Minimum allowed quiver accuracy to classify an isoform as hiqh-quality. |
| Cluster-Polish | Trim QVs 3' | qv_trim_3p | 30 | Ignore QV of n bases in the 3' end. |
| Cluster-Polish | Trim QVs 5' | qv_trim_5p | 100 | Ignore QV of n bases in the 5' end. |

**Note**: The IsoSeq Classify Only protocol does not perform isoform level clustering and only uses a subset of advanced analysis parameters.


## Classify Options
In order to display Classify advanced options via command line: `pbtranscript classify --help`.

|      Type       |  Parameter |  Example      |  Explanation      |
| --------------- | ---------- | ------------- | ----------------- |
| positional | readsFN  | ccs.bam,xml,fasta  | First positional argument. It specifies input ccs reads in bam, dataset xml, or fasta format |
| positional | outReadsFN | isoseq_draft.fasta,contigset.xml | Second positional argument. Output file which contains all classified reads in fasta or contigset xml format |
| optional | Help |  -h, --help | This prints the help message |
| optional | Full-Length Non-Chimeric | --flnc FLNC_FA.fasta|contigset.xml | Outputs full-length non-chimeric reads in fasta or contigset xml format|
| optional | Output Non-Full-Length | --nfl NFL_FA.fasta | Outputs non-full-length reads in fasta or contigset xml format |
| HMMER | HMMER Directory | -d OUTDIR, --outDir OUTDIR  | Directory to store HMMER output (default: output/) |
| HMMER | Summary | -summary SUMMARY_FN.txt | TXT file to output classify summary (default: out.classify_summary.txt) |
| HMMER | Primers File | -p PRIMERFN, --primer PRIMERFN  | Primer fasta file (default: primers.fa) |
| HMMER | Primers Report | --report PRIMERREPORTFN  | CSV file of primer info. Contains the same info found in the description lines of the output FASTA (default: out.primer_info.csv) |
| HMMER | CPUs | --cpus CPUS  | Number of CPUs to run HMMER (default: 8) |
| Chimera-detection | Minimum Sequence Length | --min_seq_len MIN_SEQ_LEN   | Minimum sequence length to output (default: 300) |
| Chimera-detection | Minimum PHMMER Score | --min_score MIN_SCORE   | Minimum phmmer score for primer hit (default: 10) |
| Chimera-detection | Non-Full-Length Chimeras | --detect_chimera_nfl   | Detect chimeric reads among non-full-length reads. Non-full-length non-chimeric/chimeric reads will saved to outDir/nflnc.fasta and outDir/nflc.fasta. |
| Read-Extraction | Ignore polyA | --ignore_polyA   | FL does not require polyA tail (default: turned off) |


## Cluster Options
In order to show IsoSeq Cluster advanced options via command line: `pbtranscript cluster`.

| Type  |  Parameter          |     Example      |  Explanation      |
| ----- | ------------------ | ---------------- | ----------------- |
| positional | Input Reads  | isoseq_flnc.fasta,contigset.xml  | Input full-length non-chimeric reads in fasta or contigset xml format, used for clustering consensus isoforms |
| positional | Output Isoforms | out.fasta,congitset.xml | Output predicted (unpolished) consensus isoforms in fasta file. |
| optional | Help  | -h, --help | This prints the help message |
| optional | Input Non-Full-Length  | --nfl_fa NFL_FA.fasta | Input non-full-length reads in fasta format, used for polishing consensus isoforms, e.g., isoseq_nfl.fasta |
| optional | CCS QVs FOFN  | --ccs_fofn CCS_FOFN | A ccs.fofn or ccs.bam or ccs.xml file. If not given, assume there is no QV information available. |
| optional | Reads QVs FOFN |  --bas_fofn BAS_FOFN  | A FOFN of bax/bas.h5, or bam, or bam.xml files (e.g., my.subreadset.xml), which contain quality values of raw reads and subreads |
| optional | Output Directory  | -d ROOT_DIR, --outDir ROOT_DIR | Directory to store temporary and output cluster files.(default: output/) |
| optional | Temp Directory  | --tmp_dir TMP_DIR | Directory to store temporary files.(default, write to root_dir/tmp.). |
| optional | Summary  | --summary SUMMARY_FN | TXT file to output cluster summary (default: my.cluster_summary.txt) |
| optional | Report  | --report REPORT_FN | CSV file, each line contains a cluster, an associated read of the cluster and the read type |
| optional | Pickle  | --pickle_fn PICKLE_FN | Developers' option, from which all clusters can be reconstructed. |
| ICE | cDNA  | --cDNA_size {under1k,between1k2k,between2k3k,above3k} | Estimated cDNA size. |
| ICE | Quiver  | --quiver | Call quiver to polish consensus isoforms using non-full-length non-chimeric CCS reads. |
| ICE | Finer Quiver  | --use_finer_qv | Use finer classes of QV information from CCS input instead of a single QV from FASTQ. This option is slower and consumes more memory. |
| SGE | Run SGE  | --use_sge | Instructs Cluster to use SGE |
| SGE | Maximum SGE Jobs  | --max_sge_jobs MAX_SGE_JOBS | The maximum number of jobs that will be submitted to SGE concurrently. |
| SGE | SGE Job ID  | --unique_id UNIQUE_ID | Unique ID for submitting SGE jobs. |
| SGE | BLASR Cores  | --blasr_nproc BLASR_NPROC | Number of cores for each BLASR job. |
| SGE | Quiver CPUs  | --quiver_nproc QUIVER_NPROC | Number of CPUs each quiver job uses. |
| IceQuiver High QV/Low QV | Minimum Quiver Accuracy  | --hq_quiver_min_accuracy HQ_QUIVER_MIN_ACCURACY | Minimum allowed quiver accuracy to classify an isoform as hiqh-quality. |
| IceQuiver High QV/Low QV | Trim QVs 5'  | --qv_trim_5 QV_TRIM_5 | Ignore QV of n bases in the 5' end. |
| IceQuiver High QV/Low QV | Trim QVs 3'  | --qv_trim_3 QV_TRIM_3 | Ignore QV of n bases in the 3' end. |
| IceQuiver High QV/Low QV | High-Quality Isoforms FASTA  | --hq_isoforms_fa HQ_ISOFORMS_FA | Quiver polished, high quality isoforms in fasta, default: root_dir/output/all_quivered_hq.fa |
| IceQuiver High QV/Low QV | High-Quality Isoforms FASTQ  | --hq_isoforms_fq HQ_ISOFORMS_FQ | Quiver polished, high quality isoforms in fastq, default: root_dir/output/all_quivered_hq.fq |
| IceQuiver High QV/Low QV | Low-Quality Isoforms FASTA  | --lq_isoforms_fa LQ_ISOFORMS_FA | Quiver polished, low quality isoforms in fasta, default: root_dir/output/all_quivered_lq.fa |
| IceQuiver High QV/Low QV | Low-Quality Isoforms FASTQ  | --lq_isoforms_fq LQ_ISOFORMS_FQ | Quiver polished, low quality isoforms in fastq, default: root_dir/output/all_quivered_lq.fq |

## Subset Options
In order to show pbtranscript Subset options via command line: `pbtranscript subset`.

| Type  |  Parameter          |     Example      |  Explanation      |
| ----- | ------------------ | --------------------------- | ----------------- |
| positional | Input Sequences  | isoseq_draft.fasta  | Input fasta file (usually isoseq_draft.fasta) |
| positional | Output Sequences | isoseq_subset.fasta | Output fasta file |
| optional | Help  | -h, --help | This prints the help message |
| optional | Output Full-length  | --FL | Reads to output must be Full-Length, with 3' primer and 5' primer and polyA tail seen. |
| optional | Output Non-Full-length  | --nonFL | Reads to output must be Non-Full-Length reads. |
| optional | Output Non-Chimeric  | --nonChimeric | Reads to output must be non-chimeric reads. |
| optional | Output Read-Length  | --printReadLengthOnly | Only print read lengths, no read names and sequences. |
| optional | Ignore polyA Tails  |  --ignore_polyA | FL does not require polyA tail (default: turned off) |

## Output Files
## Classify Output Files
__Classify FASTA Output (isoseq_*.fasta)__

`isoseq_flnc.fasta` contains all full-length, non-artificial-concatemer reads.

`isoseq_nfl.fasta` contains all non-full-length reads. 

`isoseq_draft.fasta` is an intermediate file in order to get full-length reads, which you can ignore.

Reads in these FASTA files look like the following:

```
>m140121_100730_42141_c100626750070000001823119808061462_s1_p0/119/30_1067_CCS strand=+;fiveseen=1;polyAseen=1;threeseen=1;fiveend=30;polyAend=1067;threeend=1096;primer=1;chimera=0
ATAAGACGACGCTATATG
 ```
These lines have the format: 
```
<movie_name>/<ZMW>/<start>_<end>_CCS INFO
```
The info fields are:
* strand: either + or -,  whether a read is forward or reverse-complement cdna,
* fiveseen: whether or not 5' prime is seen in this read, 1 yes, 0 no
* polyAseen: whether or not poly A tail is seen, 1 yes, 0 no
* threeseen: whether or not 3' prime is seen, 1 yes, 0 no
* fiveend: start position of 5'
* threeend: start position of 3' in read
* polyAend: start position of polyA in read
* primer: index of primer seen in this read (remember  primer fasta file >F0 xxxxx >R0 xxxxx >F1 xxxxx >R1 xxxx)
* chimera: whether or not this read is classified as  a chimeric cdna

**Note**: Reads in `isoseq-flnc.fasta` are always **strand-specific**. That is, the 5' and 3' primer (and sometimes the polyA tail) are used to tell whether the read is in the right strand. If needed, the scripts described here reverse-complement the original read and produce the sequence that is supposed to be the transcript. Non-full-length reads in `isoseq_nfl.fasta` on the other hand, could be in either orientation.

__Summary (classify_summary.txt)__ 
This file contains the following statistics:
* Number of reads of insert
* Number of five prime reads
* Number of three prime reads
* Number of poly-A reads
* Number of filtered short reads
* Number of non-full-length reads
* Number of full-length reads
* Number of full-length non-chimeric reads
* Average full-length non-chimeric read length

**Note**: By seeing that the number of full-length, non-chimeric (flnc) reads is only 
a little less than the number of full-length reads, we can confirm that the number of
artificial concatemers is very low. This indicates a successful SMRTbell library prep.

##Cluster Output Files

__Summary (cluster_summary.txt)__
This file contains the following statistics:
* Number of consensus isoforms
* Average read length of consensus isoforms


__Report (cluster_report.csv)__
This is a csv file each line of which contains the following fields:
* cluster_id: ID of a consensus isoforms from ICE.
* read_id   : ID of a read which supports the consensus isoform.
* read_type : Type of the supportive read


## Algorithm Modules

__CCS__

`pbccs` is a tool to create circular consensus sequences (ccs) sequence
from raw subreads for PacBio sequences. See [pbccs doc](https://github.com/PacificBiosciences/pbccs/blob/master/README.md) for usage.


__Classify__

Isoseq Classify classifies reads into full-length or non-full-length
reads, artifical-concatemer chimeric or non-chimeric read.

In order to classify a read as full-length or non-full-length, we search
for primers and polyA within reads. If and only if both primers and polyAs 
are seen in a read, we classify it as a full-length read. Otherwise, we
classify this read as non-full-length. We also removes primers and polyAs
from reads and identify reads strands based on these info.

**Note**: The current version of IsoSeq in SMRTLink 1.0 by default recognizes
Clontech SMRTer primers.

**Note**: In SMRTLink 1.0, custom primers are __NOT__ supported. In order to
use custom primers, You must call pbtranscript from command line like

```pbtranscript classify --primer your_primer_fasta ...```

Where your_primer_fasta is a FASTA file with the following format:

```
    >F0
    5' sequence here
    >R0
    3' sequence here (but in reverse complement)
```

Next, we further look into full-length reads and classify them into 
artificial-concatemer chimeric reads or non-chimeric reads by locating primer hits
within reads.

  * __HMMER__: We use `phmmer` in __HMMER__ package to detect locations of
               primer hits within reads and classify reads which have primer
               hits in the middle of sequences as artificial-concatemer chimeric.

__Cluster__

IsoSeq Cluster performs isoform level clustering using the Iterative Clustering
and Error correction (ICE) algorithm, which iteratively classifies full-length
non-chimeric ccs reads into clusters and builds consensus sequences of
clusters using `pbdagcon`.

ICE is customized to work well on alternative isoforms and alternative 
polyadenlynation sites, but not on SNP analysis and SNP based highly complex 
gene families.

For a detailed explanation of ICE, please refer to the [Iso-Seq webinar recording and slides](https://github.com/PacificBiosciences/cDNA_primer/wiki/Understanding-PacBio-transcriptome-data#isoseq).

  * __pbdagcon__: [`pdagcon`](https://github.com/PacificBiosciences/pbdagcon) is a
                  tool which builds consensus sequences using Directed Acyclic Graph
                  Consensus.

__Polish__

IsoSeq Polish further polishes consensus sequenecs of clusters (i.e., `pbdagcon` output)
taking into account 
We assign not only full-length non-chimeric ccs reads but also non-full-length ccs 
reads into clusters based on similarity. Then for each cluster, we align raw subreads
of its assigned zmws towards its consensus sequence. Finally, we load quality values
to these alignments and polish the consensus sequence using `Quiver`.
    
  * __Quiver__: [`Quiver`](https://github.com/PacificBiosciences/GenomicConsensus) 
                is a consensus and variant calling algorithm for PacBio reads.
                `Quiver` finds the maximum likelihood template sequence given
                PacBio reads of the template. It is used by IsoSeq to polish 
                consensus isoforms. `Quiver` uses quality values and creates 
                higher-quality consensus sequence comapred with `pbdagcon`, but is
                more time-consuming.


## Diff SMRTAnalysis v3.0 vs v2.3

PacBio will change the output of its Sequel instruments to 
[BAM](http://pacbiofileformats.readthedocs.org/en/3.0/BAM.html) format oppose to
bax.h5 of its RS and RS-II instruments. Major differences between IsoSeq in 
SMRTAnalysis v3.0 and IsoSeq in SMRTAnalysis v2.3 are listed in the table below.

*Note*: Functions of IsoSeq have NOT been changed since v2.3, and IsoSeq-Tofu has NOT been integrated.

| IsoSeq in SMRTAnalysis v3.0 | IsoSeq in SMRTAnalysis v2.3  |
| --------------------------- | ---------------------------- |
| SMRTAnalysis Web Server: SMRTLink | SMRTAnalysis Web Server: SMRTPortal |
| Works on data from Sequel | Works on data from RS and RS II |
| Input PacBio reads are stored in BAM | Input PacBio reads are stored in bax.h5 format |
| Supports PacBio [DataSet](http://pbsmrtpipe.readthedocs.org/en/master/getting_started.html#appendix-b-working-with-datasets) | Does *NOT* support PacBio Dataset |
| Uses new algorithm pbccs to create CCS reads | Uses `ConsensusTools.sh` to create CCS reads |
| Does *NOT* support using customer primers from SMRTLink | Supports using customer primers from SMRTPortal |
| Does *NOT* support using `GMAP` to align consensus isoforms to reference from SMRTLink | Supports using `GMAP` to align consensus isoforms to reference from SMRTPortal |
| SMRTLink has two protocols: `IsoSeq Classify Only` and `IsoSeq`. The `IsoSeq Classify Only` protocol only classifies reads, while the `IsoSeq` protocol not only classifies reads but also generates consensus isoforms using ICE and polish them using `Quiver`. | SMRTPortal has one protocol: RS_IsoSeq, which provides options such that users can calssify reads, or run ICE and generate unpolished consensus isoforms or polish consensus isoforms using `Quiver`. |


##Handling RS and RS II data

If you want to run IsoSeq on existing RS or RS-II data, you will need to convert 
reads in bax.h5 files to BAM files.

__Converting RS and RS-II data to BAM with SMRTLink__

TODO: points to SMRTLink Doc.

__Converting RS and RS-II data to BAM from command line__

```
  ls path_to_your_input/movie.bax.h5
  bam2bax path_to_your_input/movie.bax.h5 -o path_to_your_output/movie --subreads
  ls path_to_your_output/movie.subreads.bam path_to_your_output/movie.scraps.bam
  # path_to_your_output/movie.subreads.bam and path_to_your_output/movie.scraps.bam should be created.
```

## Glossary
* __Chimera__
  * IsoSeq Classify classifies reads as artificial-concatemer chimeric or non-chimeric
    based on whether or not primers are found in the middle of the sequence.

* __High QV | Low QV__
  * Isoseq Cluster generates polished consensus isoforms are classified into either
    high quality or low quality isoforms. We classify an isoform as high quality if 
    its conseusus accuracy is no less than a cut-off, otherwise low quality.
    The default cut-off is **0.99**. You may change this value from command line, or
    via SMRTLink Advanced Analysis Parameters when creating an IsoSeq job.


