

.. _celera-assembler: `Celera Assembler <http://wgs-assembler.sourceforge.net>`

.. _tutorial:

Canu Tutorial
=============

Canu assembles reads from PacBio RS II or Oxford Nanopore MinION instruments into
uniquely-assemblable contigs, unitigs.  Canu owes lots of it design and code to
`celera-assembler`_.

Canu can be run using hardware of nearly any shape or size, anywhere from laptops to computational
grids with thousands of nodes.  Obviouisly, larger assemblies will take a long time to compute on
laptops, and smaller assemblies can't take advantage of hundreds of nodes, so what is being
assembled plays some part in determining what hardware can be effectively used.

Most algorithms in canu have been multi-threaded (to use all the cores on a single node),
parallelized (to use all the nodes in a grid), or both (all the cores on all the nodes).

.. _canu-command:

Canu, the command
~~~~~~~~~~~~~~~~~~~~~~

The **canu** command is the 'executive' program that runs all modules of the assembler.  It oversees
each of the three top-level tasks (correction, trimming, unitig construction), each of which
consists of many steps.  Canu ensures that input files for each step exist, that each step
successfully finished, and that the output for each step exists.  It does minor bits of processing,
such as reformatting files, but generally just executes other programs.

::

 canu [-correct | -trim | -assemble | -trim-assemble] \
   [-s <assembly-specifications-file>] \
    -p <assembly-prefix> \
    -d <assembly-directory> \
    genomeSize=<number>[g|m|k] \
    [other-options] \
    [-pacbio-raw | -pacbio-corrected | -nanopore-raw | -nanopore-corrected] *fastq

Two options are mandatory, -d, to set the working directory, and -p, to set the file name prefix.
All work is performed and output appears in the working directory.  The directory need not exist
before starting.  Most files in this directory have file names beginning with the file name
prefix, however, running two canu commands in the same directory will probably lead to confusion.

The -s option will import a list of parameters from the supplied specification ('spec') file.  These
parameters will be applied before any from the command line are used, providing a method for
setting commonly used parameters, but overriding them for specific assemblies.

By default, all three top-level tasks are performed.  It is possible to run exactly one task by
using the -correct, -trim or -assemble options.  These options can be useful if you want to correct
reads once and try many different assemblies.  We do exactly that in the :ref:`quickstart`.

Parameters are key=value pairs that configure the assembler.  They set run time parameters (e.g.,
memory, threads, grid), algorithmic parameters (e.g., error rates, trimming aggressiveness), and
enable or disable entire processing steps (e.g., don't correct errors, don't search for subreads).
They are described later.  One parameter is required: the genomeSize (in bases, with common
SI prefixes allowed, for example, 4.7m or 2.8g; see :ref:`genomeSize`).  Parameters are listed in
the :ref:`parameter-reference`, but the common ones are described in this document.

Reads are supplied to canu by options that options that describe how the reads were generated, and
what level of quality they are, for example, -pacbio-raw indicates the reads were generated on a
PacBio RS II instrument, and have had no processing done to them.  Each file of reads supplied this
way becomes a 'library' of reads.  The reads should have been (physically) generated all at the same
time using the same steps, but perhaps sequenced in multiple batches.  In canu, each library has a
set of options setting various algorithmic parameters, for example, how aggressively to trim.  To
explicitly set library parameters, a text 'gkp' file describing the library and the input files must
be created.  Don't worry too much about this yet, it's an advanced feature, fully described in
Section :ref:`gkp-files`.

The read-files contain sequence data in either FASTA or FASTQ format (or both!  A quirk of the
implementation allows files that contain both FASTA and FASTQ format reads).  The files can be
uncompressed, gzip, bzip2 or xz compressed.  We've found that "gzip -1" provides good compression
that is fast to both compress and decompress.  For 'archival' purposes, we use "xz -9".

.. _canu-pipeline:

Canu, the pipeline
~~~~~~~~~~~~~~~~~~~~~~

The canu pipeline, that is, what it actually computes, comprises of computing overlaps and
processing the overlaps to some result.  Each of the three tasks (read correction, read trimming and
unitig construction) follow the same pattern:

* Load reads into the read database, gkpStore.
* Compute k-mer counts in preparation for the overlap computation.
* Compute overlaps.
* Load overlaps into the overlap database, ovlStore.
* Do something interesting with the reads and overlaps.

  * The read correction task will replace the original noisy read sequences with consensus sequences
    computed from overlapping reads.
  * The read trimming task will use overlapping reads to decide what regions of each read are
    high-quality sequence, and what regions should be trimmed.  After trimming, the single largest
    high-quality chunk of sequence is retained.
  * The unitig construction task finds sets of overlaps that are consistent, and uses those to place
    reads into a multialignment layout.  The layout is then used to generate a consensus sequence
    for the unitig.



.. _module-tags:

Module Tags
~~~~~~~~~~~~~~~~~~~~~~

Because each of the three tasks share common algorithms (all compute overlaps, two compute
consensus sequences, etc), parameters are differentiated by a short prefix 'tag' string.  This lets
canu have one generic parameter that can be set to different values for each stage in each task.
For example, "corOvlMemory" will set memory usage for overlaps being generated for read correction;
"obtOvlMemory" for overlaps generated for Overlap Based Trimming; "utgOvlMemory" for overlaps
generated for unitig construction.

The tags are:

+--------+-------------------------------------------------------------------+
|Tag     | Usage                                                             |
+========+===================================================================+
|master  | the canu script itself, and any components that it runs directly  |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|cns     | unitig consensus generation                                       |
+--------+-------------------------------------------------------------------+
|cor     | read correction generation                                        |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|red     | read error detection                                              |
+--------+-------------------------------------------------------------------+
|oea     | overlap error adjustment                                          |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|ovl     | the standard overlapper                                           |
+--------+-------------------------------------------------------------------+
|corovl  | the standard overlapper, as used in the correction phase          |
+--------+-------------------------------------------------------------------+
|obtovl  | the standard overlapper, as used in the trimming phase            |
+--------+-------------------------------------------------------------------+
|utgovl  | the standard overlapper, as used in the assembly phase            |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|mhap    | the mhap overlapper                                               |
+--------+-------------------------------------------------------------------+
|cormhap | the mhap overlapper, as used in the correction phase              |
+--------+-------------------------------------------------------------------+
|obtmhap | the mhap overlapper, as used in the trimming phase                |
+--------+-------------------------------------------------------------------+
|utgmhap | the mhap overlapper, as used in the assembly phase                |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|mmap    | the `minimap <https://github.com/lh3/minimap>`_ overlapper                                      |
+--------+-------------------------------------------------------------------+
|cormmap | the minimap overlapper, as used in the correction phase           |
+--------+-------------------------------------------------------------------+
|obtmmap | the minimap overlapper, as used in the trimming phase             |
+--------+-------------------------------------------------------------------+
|utgmmap | the minimap overlapper, as used in the assembly phase             |
+--------+-------------------------------------------------------------------+
+--------+-------------------------------------------------------------------+
|ovb     | the bucketizing phase of overlap store building                   |
+--------+-------------------------------------------------------------------+
|ovs     | the sort phase of overlap store building                          |
+--------+-------------------------------------------------------------------+

We'll get to the details eventually.

.. _execution:

Execution Configuration
~~~~~~~~~~~~~~~~~~~~~~~~

There are two modes that canu runs in: locally, using just one machine, or grid-enabled, using
multiple hosts managed by a grid engine.  LSF, PBS/Torque, PBSPro, Sun Grid Engine (and
derivations), and Slurm are supported, though LSF has has limited testing. Section
:ref:`grid-engine-config` has a few hints on how to set up a new grid engine.

By default, if a grid is detected the canu pipeline will immediately submit itself to the grid and
run entirely under grid control.  If no grid is detected, or if option ``useGrid=false`` is set,
canu will run on the local machine.

In both cases, Canu will auto-detect available resources and configure job sizes based on the
resources and genome size you're assembling. Thus, most users should be able to run the command
without modifying the defaults. Some advanced options are outlined below. Each stage has the same
five configuration options, and tags are used to specialize the option to a specific stage.  The
options are:

useGrid<tag>=boolean
  Run this stage on the grid, usually in parallel.
gridOptions<tag>=string
  Supply this string to the grid submit command.
<tag>Memory=integer
  Use this many gigabytes of memory, per process.
<tag>Threads
  Use this many compute threads per process.
<tag>Concurrency
  If not on the grid, run this many jobs at the same time.

Global grid options, applied to every job submitted to the grid, can be set with 'gridOptions'.
This can be used to add accounting information or access credentials.

A name can be associated with this compute using 'gridOptionsJobName'.  Canu will work just fine
with no name set, but if multiple canu assemblies are running at the same time, they will tend to
wait for each others jobs to finish.  For example, if two assemblies are running, at some point both
will have overlap jobs running.  Each assembly will be waiting for all jobs named 'ovl_asm' to
finish.  Had the assemblies specified job names, gridOptionsJobName=apple and
gridOptionsJobName=orange, then one would be waiting for jobs named 'ovl_asm_apple', and the other
would be waiting for jobs named 'ovl_asm_orange'.

.. _error-rates:

Error Rates
~~~~~~~~~~~~~~~~~~~~~~


Canu expects all error rates to be reported as fraction error, not as percent error.  We're not sure
exactly why this is so.  Previously, it used a mix of fraction error and percent error (or both!),
and was a little confusing.  Here's a handy table you can print out that converts between fraction
error and percent error.  Not all values are shown (it'd be quite a large table) but we have every
confidence you can figure out the missing values:

==============  =============
Fraction Error  Percent Error
==============  =============
0.01            1%           
0.02            2%           
0.03            3%           
.               .            
.               .            
0.12            12%          
.               .            
.               .            
==============  =============

Canu error rates always refer to the percent difference in an alignment of two reads, not the
percent error in a single read, and not the amount of variation in your reads.  These error rates
are used in two different ways: they are used to limit what overlaps are generated, e.g., don't
compute overlaps that have more than 5% difference; and they are used to tell algorithms what
overlaps to use, e.g., even though overlaps were computed to 5% difference, don't trust any above 3%
difference.

There are seven error rates.  Three error rates control overlap creation (``corOvlErrorRate``,
``obtOvlErrorRate`` and ``utgOvlErrorRate``), and four error rates control algorithms
(``corErrorRate``, ``obtErrorRate``, ``utgErrorRate``, ``cnsErrorRate``).

Recall there are three sets of overlaps generated: one for read correction, one for read trimming,
and one for unitig construction.  The read correction overlaps are typically generated by the 'mhap'
overlapper, which doesn't need an error rate as it doesn't generate alignment based overlaps.  The
other two sets of overlaps are generated by overlapInCore, which is alignment based, and so requires
a maximum error rate.

corOvlErrorRate
  Is not used by mhap or minimap.  For mhap, use ``mhapSensitivity``.
  There is no comparable setting for minimap.  ``corOvlErrorRate`` exists for computing alignment-based
  overlaps between corrected overlaps, but don't do that unless you are incredibly patient and have
  lots of compute time to waste.

  (ADVANCED) It is possible to convert the mhap or minimap overlaps to alignment based overlaps using
  ``obtReAlign=true`` or ``ovlReAlign=true``.  If so, the overlap alignments will be computed using
  the error rate corresponding to the overlaps being generated.

obtOvlErrorRate
  Do not compute overlaps used for trimming above this error rate.  Applies to the
  standard overlapper, and realigning mhap overlaps.

utgOvlErrorRate
  Do not compute overlaps used for unitig construction above this error rate.  Applies
  to the standard overlapper, and realigning mhap overlaps.

For algorithms, coarse grained control over performance can be achieved using the erorr rates.

corErrorRate
  Filter overlaps used for correcting raw reads.

obtErrorRate
  Filter overlaps during OBT's read trimming and read splitting.

utgErrorRate
  Filter overlaps used for generating contigs.  There are MANY more options available for contig creation.

cnsErrorRate
  Inform the consensus genration algorithm of the amount of difference it should expect
  in a read-to-read alignment.

For convenience, two meta options set the error rates used with uncorrected reads or used with corrected reads.

rawErrorRate
  The allowed difference in an overlap between two uncorrected reads.  Sets ``corOvlErrorRate`` and ``corErrorRate``.
  The ``rawErrorRate`` typically does not need to be modified.  It might need to be increased if very early reads are being assembled.
  The default is 0.30 For PacBio reads, and 0.50 for Nanopore reads.

correctedErrorRate
  The allowed difference in an overlap between two corrected reads.  Sets ``obtOvlErrorRate``, ``utgOvlErrorRate``, ``obtErrorRate``, ``utgErrorRate``, and ``cnsErrorRate``.
  The ``correctedErrorRate`` can be adjusted to account for the quality of read correction, for the amount of divergence in the sample being
  assembled, and for the amount of sequence being assembled.  The default is 0.045 for PacBio reads, and 0.144 for Nanopore reads.

  For low coverage datasets (less than 30X), we recommend increasing ``correctedErrorRate`` slightly, by 1% or so.

  For high-coverage datasets (more than 60X), we recommend decreasing ``correctedErrorRate`` slighly, by 1% or so.

  Raising the ``correctedErrorRate`` will increase run time.  Likewise, decreasing ``correctedErrorRate`` will decrease run time, at the risk of missing overlaps and fracturing the assembly.

errorRate (OBSOLETE)
  Prior to Canu 1.5 (or prior to January 27th, 2016) the ``errorRate`` parameter was used to specify the expected error in a single corrected read.  The above seven error rates were then set to three times this value (except for ``corErrorRate``).  Canu currently still accepts the ``errorRate`` parameter, but use is strongly discouraged.


.. _minimum-lengths:

Minimum Lengths
~~~~~~~~~~~~~~~~~~~~~~

Two minimum sizes are known:

minReadLength
  Discard reads shorter than this when loading into the assembler, and when trimming reads.

minOverlapLength
  Do not save overlaps shorter than this.

Overlap configuration
~~~~~~~~~~~~~~~~~~~~~~

The largest compute of the assembler is also the most complicated to configure.  As shown in the
'module tags' section, there are up to eight (!) different overlapper configurations.  For
each overlapper ('ovl' or 'mhap') there is a global configuration, and three specializations
that apply to each stage in the pipeline (correction, trimming or assembly).

Like with 'grid configuration', overlap configuration uses a 'tag' prefix applied to each option.  The
tags in this instance are 'cor', 'obt' and 'utg'.

For example:

- To change the k-mer size for all instances of the ovl overlapper, 'merSize=23' would be used.
- To change the k-mer size for just the ovl overlapper used during correction, 'corMerSize=16' would be used.
- To change the mhap k-mer size for all instances, 'mhapMerSize=18' would be used.
- To change the mhap k-mer size just during correction, 'corMhapMerSize=15' would be used.
- To use minimap for overlap computation just during correction, 'corOverlapper=minimap' would be used.

Ovl Overlapper Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

<tag>Overlapper
  select the overlap algorithm to use, 'ovl' or 'mhap'.

Ovl Overlapper Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~

<tag>ovlHashBlockLength
  how many bases to reads to include in the hash table; directly controls process size
<tag>ovlRefBlockSize
  how many reads to compute overlaps for in one process; directly controls process time
<tag>ovlRefBlockLength 
 same, but use 'bases in reads' instead of 'number of reads'
<tag>ovlHashBits
  size of the hash table (SHOULD BE REMOVED AND COMPUTED, MAYBE TWO PASS)
<tag>ovlHashLoad
  how much to fill the hash table before computing overlaps (SHOULD BE REMOVED)
<tag>ovlMerSize
  size of kmer seed; smaller - more sensitive, but slower

The overlapper will not use frequent kmers to seed overlaps.  These are computed by the 'meryl' program,
and can be selected in one of three ways.

Terminology.  A k-mer is a contiguous sequence of k bases.  The read 'ACTTA' has two 4-mers: ACTT
and CTTA.  To account for reverse-complement sequence, a 'canonical kmer' is the lexicographically
smaller of the forward and reverse-complemented kmer sequence.  Kmer ACTT, with reverse complement
AAGT, has a canonical kmer AAGT.  Kmer CTTA, reverse-complement TAAG, has canonical kmer CTTA.

A 'distinct' kmer is the kmer sequence with no count associated with it.  A 'total' kmer (for lack
of a better term) is the kmer with its count.  The sequence TCGTTTTTTTCGTCG has 12 'total' 4-mers
and 8 'distinct' kmers.

::

 TCGTTTTTTTCGTCG  count
 TCGT             2 distinct-1
  CGTT            1 distinct-2
   GTTT           1 distinct-3
    TTTT          4 distinct-4
     TTTT         4 copy of distinct-4
      TTTT        4 copy of distinct-4
       TTTT       4 copy of distinct-4
        TTTC      1 distinct-5
         TTCG     1 distinct-6
          TCGT    2 copy of distinct-1
           CGTC   1 distinct-7
            GTCG  1 distinct-8


<tag>MerThreshold
  any kmer with count higher than N is not used
<tag>MerDistinct
  pick a threshold so as to seed overlaps using this fraction of all distinct kmers in the input.  In the example above,
  fraction 0.875 of the k-mers (7/8) will be at or below threshold 2.
<tag>MerTotal
  pick a threshold so as to seed overlaps using this fraction of all kmers in the input.  In the example above,
  fraction 0.667 of the k-mers (8/12) will be at or below threshold 2.
<tag>FrequentMers
  don't compute frequent kmers, use those listed in this fasta file

Mhap Overlapper Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~

<tag>MhapBlockSize
  Chunk of reads that can fit into 1GB of memory. Combined with memory to compute the size of chunk the reads are split into.
<tag>MhapMerSize
  Use k-mers of this size for detecting overlaps.
<tag>ReAlign
  After computing overlaps with mhap, compute a sequence alignment for each overlap.
<tag>MhapSensitivity
  Either 'normal', 'high', or 'fast'.

Mhap also will down-weight frequent kmers (using tf-idf), but it's selection of frequent is not exposed.

Minimap Overlapper Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~~
<tag>MMapBlockSize
  Chunk of reads that can fit into 1GB of memory. Combined with memory to compute the size of chunk the reads are split into.
<tag>MMapMerSize
  Use k-mers of this size for detecting overlaps

Minimap also will ignore high-frequency minimzers, but it's selection of frequent is not exposed.
