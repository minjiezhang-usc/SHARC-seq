# SHARC-seq

SHARC-Seq pipeline are python scripts for Spatial 2’-Hydroxyl Acylation Reversible Crosslinking (SHARC) sequencing data.


# Overview

Three-dimensional (3D) structures dictate the functions of RNA molecules in a wide variety of biological processes. RNA 3D structure is determined by sparse tertiary contacts which lock RNA into well-defined structures. Direct determination of RNA 3D structures in vivo is difficult due to their large sizes, conformational heterogeneity and dynamics. Here we present a new method, Spatial 2’-Hydroxyl Acylation Reversible Crosslinking (SHARC) sequencing, which uses chemical crosslinkers of defined lengths to measure distances between nucleotides in cellular RNA. 


# System Requirements

Hardware requirements

    SHARC-Seq pipeline package requires only a standard computer with enough RAM to support the in-memory operations.

Software requirements

    OS Requirements

    This package is supported for macOS and Linux. The package has been tested on the following systems:

         macOS: Mojave (10.14.1)
         Linux: Ubuntu 16.04

    Python Dependencies
         SHARC-Seq pipeline mainly depends on the Python scientific stack.
         numpy
         itertools
         seaborn
         math
         random
         pysam
         RNA
         matplotlib
         sklearn.metrics
         PDBParser
    

# SHARC-seq analysis strategy

1, The masked genome indicies

In order to accurately and easily analyze PARIS data, pseudogenes and multicopy genes from gencode, refGene and Dfam were masked from hg38/mm10 genome. And then single copy of them was added back as a separated “chromosome”. For example, multicopy of snRNAs were masked from the basic hg38/mm10 assembly genome, and 9 snRNAs (U1, U2, U4, U5, U6, U11, U12, U4atac and U6atac) were concatenated into one reference, separated by 100nt “N”s, was added back. The curated hg38/mm10 genome contained 25 reference sequences, or “chromosomes”, masked the multicopy genes and added back single copies. This reference is best suited for the PARIS analysis. The curated genome of hg38/mm10 can be downloaded from https://drive.google.com/open?id=1wHSC-mf1jNNClXrVqMugqVmDVT4Crxzz

2, Mapping

Reads were mapped to manually curated hg38 or mm10 genome using STAR program(Dobin, Davis et al. 2013).

    STAR --runThreadN 8 --runMode alignReads --genomeDir OuputPath --readFilesIn SampleFastq  --outFileNamePrefix Outprefix --genomeLoad NoSharedMemory outReadsUnmapped Fastx  --outFilterMultimapNmax 10 --outFilterScoreMinOverLread 0 --outSAMattributes All --outSAMtype BAM Unsorted SortedByCoordinate --alignIntronMin 1 --scoreGap 0 --scoreGapNoncan 0 --scoreGapGCAG 0 --scoreGapATAC 0 --scoreGenomicLengthLog2scale -1 --chimOutType WithinBAM HardClip --chimSegmentMin 5 --chimJunctionOverhangMin 5 --chimScoreJunctionNonGTAG 0 --chimScoreDropMax 80 --chimNonchimScoreDropMin 20

3, Classify alignments

In this step, alignments in the sam file are filtered to remove low-confidence segments, rearranged and classified into 5 groups using gaptypes.py.

    python gaptypes.py input.sam output_prefix glenlog nprocs
		
    glenlog: -1. Scaling factor for gap extension penalty, equivalent to scoreGenomicLengthLog2scale in STAR
    minlen:  15. Minimal length for a segment in an alignment to be considered confident for building the connection database
    nprocs:  10. Number of CPUs to use for the run, depending availability of resources.
		
4, Filter spliced and short gaps

Output files gap1.sam and gapm.sam may contain alignments that have only splicing junctions and short 1-2 nt gaps due to artifacts. These are filtered out using gapfilter.py before further processing. 

    python gapfilter.py annotation insam outsam idloc short
		
    annotation: the file containing the splicing junctions should be in GTF format. 
    idloc:      location of the transcript_ID field, is usually field 11. 
    short:      is set to either yes which means 'remove short 1-2nt gaps', or no, which means 'ignore short 1-2nt gaps'.

5, Cluster alignments to groups

After filtering alignments, To assemble alignments to DGs and NGs using the crssant.py script.

    python crssant.py [-h] [-out OUT] [-cluster CLUSTER] [-n N] [-covlimit COVLIMIT] [-t_o T_O] [-t_eig T_EIG] alignfile genesfile bedgraphs
    
    alignfile: Path to alignedment file (SAM)
    genesfile: Path to gene annotations (BED)
    bedgraphs: Path to genome coverage files. Coma separated 2 files for + and - strands
    
    Optional arguments:
    -h:        show the help message and exit
    -out:      path of output folder
    -cluster:  clustering method, "cliques" or "spectral"(default)
    -n:        Number of threads. Default is 8
    -covlimit: Max coverage to be directly graphed. Default is 1000.
    -t_o:      Overlap threshold 0-1 inclusive. Default: 0.5 for "spectral", 0.1 for "cliques"
    -t_eig:    Eigenratio threshold (positive) for "spectral" only. Default: 5.

6, icSHAPE activity calculation

After DG assembly, use DG_frenquency_icSHAPE.py script to calculate the icSHAPE activity baesed on DGs or reads.

    python DG_frenquency_icSHAPE.py  inputsam  shape_bedpragh  DG_reads_cutoff  DGcommon_ratio  seqlen  extendlen  DG/reads  outputprefix
		
    sam_crssant:     sam file generated by crssant.py
    shape_bedpragh:  shape activity of each nucleotides, in bedgraph format, eg: HEK293_icSHAPE_hs45S.bedgraph
    DG_reads_cutoff: minimum reads number of each DG, eg: 10
    DGcommon_ratio:  minimum ratio to call the common region of each DG, eg: 0.5
    seqlen:          how many nts before 3' end, eg: 15
    extendlen:       how many nts after 3' end, eg: 15
    DG/reads:        DG or each reads was used for analysis
		
7, Calculate the distance between crosslinked two arms

After DG assembly, use DG_rRNA_distance_heatmap.py script to calculate the distance of two arms baesed on DGs or reads, and plot the distance heatmap.

    python DG_rRNA_distance_heatmap.py  sam_crssant  shape_bedpragh  shape_cutoff  bp_arc_file  4v6x.cif/4v6x_distance.txt  DG_reads_cutoff  distance_cutoff  DGcommon_ratio  DG/reads  winbin  outputprefix
		
    sam_crssant:     sam file generated by crssant.py
    shape_bedpragh:  shape activity of each nucleotides, in bedgraph format. eg: HEK293_icSHAPE_hs45S.bedgraph
    shape_cutoff:    cut off value for shape data. positive data will be used to plot Cyto-ME analysis, eg: 0.1
    bp_arc_file:     base pair information generated from PDB file, eg: 4v6x.cif
    4v6x_distance.txt:   matrix_PDB_distance.txt generated from 4v6x
    DG_reads_cutoff: minimum reads number of each DG, eg: 10
    distance_cutoff: minimum distance was considered as tertiary structure, otherwise was considered as inter-molecular interactions, eg: 40
    DGcommon_ratio:  minimum ratio to call the common region of each DG, eg: 0.5
    winbin:          length of winbin, eg: 5
		
8, rRNA dynamic structure analysis

After DG assembly, rRNA dynamic structure can be analyzed using DG_rRNA_dynamic.py scipt.

    python DG_rRNA_dynamic.py  sam_crssant  bp_arc_file  dsRNA.bedpe  expansion_segment.bed  4v6x.cif/4v6x_distance.txt  DG_reads_cutoff  distance_cutoff  DGcommon_ratio  winbin  DG/reads  outputprefix
		
    sam_crssant:     sam file generated by crssant.py
    bp_arc_file:     base pair information generated from PDB file, eg: hs45.arc.bed
    dsRNA.bedpe:     dsRNA information in bedpe format. eg: hs45_dsRNA.bedpe, eg: hs45_dsRNA.bedpe
    expansion_segment.bed:   hs45S_ES.bed
    4v6x.cif:        PDB file downloaed from PDB database
    4v6x_distance.txt:   matrix_PDB_distance.txt generated from 4v6x
    DG_reads_cutoff: minimum reads number of each DG, eg: 10
    distance_cutoff: minimum distance was considered as tertiary structure, otherwise was considered as inter-molecular interactions, eg: 40
    DGcommon_ratio:  minimum ratio to call the common region of each DG, eg: 0.5
    winbin:          window bins, eg: 10
