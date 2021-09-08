# SC_CRISPR
This repository holds the scripts used to process **S**ingle **C**ell **CRISPR** Screening data
Currently the pipeline begins from output of "cellranger count"


## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes.

### Prerequisites
The following components were used to run the software. Different versions of each may or may not be compatible.

- OS: CentOS Linux release 7.1.1503 (Core)
- Cellranger: [cellranger-3.0.2](https://support.10xgenomics.com/single-cell-gene-expression/software/downloads/latest)
  - Reference genome: refdata-cellranger-GRCh38-3.0.0
- Perl: perl 5, version 16, subversion 3 (v5.16.3)
- Python: Python 2.7.5
- Bowtie (Optional): bowtie-0.12.5

The following python packages are required for generating the pdf quality control report:
- numpy
- matplotlib
- reportlab

### Installing
Assumes perl and python3 are already installed.
Download or clone SC_CRISPR.

Install [Cell Ranger](https://support.10xgenomics.com/single-cell-gene-expression/software/downloads/latest)
Install python packages
```
python -m pip install -U pip
python -m pip install -U numpy scipy matplotlib reportlab
```

Install Bowtie (optional)
- Download appropriate zip file from [here](https://sourceforge.net/projects/bowtie-bio/files/bowtie/old/0.12.5/)
- Extract to the desired directory
- Add bowtie directory to $PATH


## Running the pipeline
### Run Cell Ranger (Unless using bowtie alignment method)
Detailed instructions can be found [here](https://support.10xgenomics.com/single-cell-gene-expression/software/pipelines/latest/using/count)
Alternately, if starting from fastq files made by bcl2fastq it may be useful to tweak the provided [run_cellranger](scripts_related/run_cellranger.pl) script to handle the cellranger run
1. run cellranger count  
2. run cellranger mat2csv  
  The output file should be named feature_barcode.csv and stored directly in the cellranger outs folder.  

### Run gRNA detection and Sequence Parsing
1. Provide a design file and a cassette file. Example files are provided [here](example_inputs.zip)
2. Provide the correct entries in [pipeline_inputs](#pipeline_inputstxt)
3. Run the pipeline.pl script
```
  perl pipeline.pl <pipeline_inputs file>
```
4. Run the create_report.py script to assess quality
```
  python3 create_report.py <run_output_dir> cell2gRNA.txt
```
5. Map the gene expression matrix to include cell barcode, gRNA, target gene
```
  perl ana1_gRNA2expression.pl <run_folder> <cell2gRNA filename>
```

### I/O Descriptions  
#### Inputs  
***pipeline_inputs.txt***  
This file has the following format. The order of rows doesn't matter.  

```
  sample_name	<sample name>
  input_format	<cellranger OR fastq>
  data_input_folder	<path-to-outs-folder OR path-to-fastq-folder>

  design_file	<path-to-design-file>
  cassette_file	<path-to-cassette-file>
  method_align	<cellranger OR bowtie>

  #Advanced Options
  gRNA_only	<true OR false>
  ignore_aligned	<true OR false>
  
  #Bowtie Options
  bowtie_file	<path-to-bowtie-software-file>
  bowtie_length	<maximum bowtie alignment length>
```  
***sample_name*** is the name of the sample, used to name the output folder of the run. For consistency we recommend that sample_name is the same as the name of the raw data folder  
***input_format*** dictates whether the files used as input are single-cell fastq files, or processed data from cellranger. IMPORTANT: for the fastq option, each fastq file is a collection of single ended reads from a single clone.  
***data_input_folder*** is either the output folder from cellranger or a directory containing fastq files. The fastq directory must contain only single ended read files, omitting index reads. The default location for cellranger outputs is &lt;sample-name>/&lt;outs>  

***design_file*** is a single column file with the list of designed gRNAs (can be 20 or 21 bases, if it's 21 bases then it starts with a G)  
***cassette_file*** is a file with a single line containing the entire transcribed sequence containing a gRNA. This will be used to determine which reads are gRNA-related. These reads will be diverted to cell2gRNA mapping instead of expression analysis  
***method_align*** selects whether we use cellranger STAR or a custom bowtie alignment.  

***gRNA_only*** = true results in omission of the seq_expr file when it is not necessary to look at transcriptome sequence. ie. if using cellranger's alignment result or if only interested in gRNA mapping. This is required to be false if using bowtie alignment.  
***ignore_aligned*** = true will result in a faster but incomplete parsing of cellranger. This ignores all sequences that were aligned to the reference, leaving only exogenous sequences for gRNA mapping. This may be useful for a rapid quality check, such as to get a first impression of whether an experimental procedure works.

***bowtie_file*** is the location of your bowtie alignment software package. If using custom bowtie alignment, this is required.
***bowtie_length*** dictates the maximum bowtie alignment length. The raw sequence data will be truncated as the maximum bowtie alignment length before alignment. 

#### Outputs
Outputs are saved in a folder with the name structure: runs\_&lt;datetime>\_samplename  
***stats_qc.txt*** contains basic quality control stats including read counts  
***stats_cell2gRNA.txt*** contains basic stats on the gRNAs identified  
***cell2gRNA.txt*** is a tab-delimited file with the following fields:  
1. cell barcode  
2. number of distinct gRNAs  
3. gRNA sequences [separated by commas]  
4. number of molecules per gRNA [separated by commas]  
5. number of reads per gRNA [separated by commas]  
6. UMI sequences [separated by commas within a gRNA, different gRNAs separated by vertical bars]  
7. number of reads per molecule [separated by commas within a gRNA, different gRNAs separated by vertical bars]  

***expr_mat*** is a matrix of UMI count per gene for each cell.  
***misc*** various other intermediate files are also included such as sequence reads. In addition, all inputs are copied and included in the output folder.
If using custom bowtie alignment, there would be following files:
***refseq.xls*** is the bowtie alignment result based on refseq reference sequence. 
***not_refseq.fa*** is the sequence couldn't be aligned to refseq reference sequence.
***genome.xls*** is the bowtie alignment result based on whole genome sequence. 
***not_genome.fa*** is the sequence couldn't be aligned to whole genome sequence.