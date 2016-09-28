# viGEN
A bioinformatics pipeline for the detection and quantification of viral RNA in human NGS data

## Steps in general
![Image](https://github.com/ICBI/viGEN/blob/master/workflow.png)

## About the data

*	For this tutorial, we used a sample RNA-seq file from liver cancer from public study SRA http://www.ncbi.nlm.nih.gov/bioproject/PRJNA279878 or GEO http://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE65485 (also available at EBI SRA here: http://www.ebi.ac.uk/ena/data/view/SRR1946637). 
* About the dataset: Data consists of Liver cancer patients in China. The authors look for HBV-human integration sites in the dataset of 50 Liver cancer patients in China and 5 adjacent normal tissues. We downloaded the raw reads for one sample SRR1946637 and unzipped them. This sample is from an adjacent normal liver tissue. 

## Workflow steps in detail
### Alignment to human reference
- In this tutorial, we use tool RSEM for alignment
- Install RSEM as shown in https://github.com/bli25ucb/RSEM_tutorial 
- RSEM uses aligner Bowtie1, Bowtie2 or STAR. We chose to use Bowtie1 which is available as pre-compiled executable. For Bowtie1 installation, follow instructions in this manual: http://bowtie-bio.sourceforge.net/manual.shtml#obtaining-bowtie 
- Test RSEM successful installation
  `./rsem-calculate-expression --version`
	`./rsem-calculate-expression --help`
- Generate transcriptome reference for RSEM:
  
    RSEM uses a transcriptome reference for alignment. RSEM has a command called `rsem-prepare-reference` where it uses the genome fasta reference file and the GTF file to create the transcriptome reference. We have generated and provided these files for human hg19 reference genome in the [**RSEM_reference/Rsem_ref_hg19**](https://drive.google.com/drive/folders/0B3-883ME4sP3dDF3ZllrN1JSWmM?usp=sharing) folder via google drive. 
    
    *If you are interested in using another reference, please follow steps provided here to generate transcriptom reference for your file: https://github.com/bli25ucb/RSEM_tutorial*
    
- Once RSEM is installed, and the transcriptome reference is ready, run alignment using RSEM on SRA fastq files using following command: `./rsem-calculate-expression --paired-end -p 8 --output-genome-bam --keep-intermediate-files --bowtie-path /Users/username/Documents/software/bowtie/bowtie-1.1.2 --append-names --estimate-rspd --time input/SRR1946637_1.fastq input/SRR1946637_2.fastq Rsem_ref_hg19/Rsem_ref_hg19 output/SRR1946637`

   Output files: In addition to the aligned BAM file (genome level and transcriptome level), this will generate the unaligned (unmapped) fastq files named **SRR1946637_un_1.fq** and **SRR1946637_un_2.fq**. They consist of the reads that did not align to the human reference. *You can see an archive of the RSEM output files [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3QlpDS0d1UlJKdTg?usp=sharing).*
  
  This workflow is part of “Module 1 (filtered human input files)” in Figure 1 in the viGEN manuscript (currently under preparation).
  
### Create viral reference
- We were interested in exploring all viruses existing in humans. So we first obtained reference genomes of all known and sequenced human viruses obtained from NCBI (as of Sep 2015), and merged them into one file (referred to as the ‘viral reference file’) in fasta file format.Merge all virus fasta file into one big fasta file called **viruses.fa**. For this tutorial, we have provided this file through google drive [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3Wm1FVjdVcEpfek0?usp=sharing).
- We have also indexed the viral reference file, so that these files are ready for alignment tools Bowtie2 (folder name: **virus.bowtie2.refB**) or BWA (folder name: **viral.bwa.ref**) through google drive [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3Wm1FVjdVcEpfek0?usp=sharing).
  
  In case user is interested in creating a reference index the reference file on their own, this is the command to use: `./bowtie2-build /PathToLocation/viruses.fa virus.bowtie2.refB`
  
- NCBI also allows to download information/annotation about these viruses from their web site. This information has been provided as [**Complete_Sequence_info.csv**](https://docs.google.com/spreadsheets/d/1qN_ZcPDPZnJZXDdutjpTlUt8QY9K4VCQvmHdnHZU_j8/edit?usp=sharing).

### Align the unmapped fastq files to the viral reference
  - 	In this tutorial, we use alignment tool Bowtie2.
  -	Install Bowtie2 pre-compiled binary file based on instructions in this manual: http://bowtie-bio.sourceforge.net/bowtie2/manual.shtml#obtaining-bowtie-2 
  -	Alignment generates output in the form of a SAM file. This SAM file contains the reads from various viruses aligned to the viral reference.
  -	Convert SAM to BAM using Samtools
  -	Sort the BAM file coordinate wise
  -	Use `Samtools idx” tool`. This produces a tab delimited file, with each line consisting of a virus sequence name, sequence length, # mapped reads and # unmapped reads. The number of mapped reads is referred to as *genome level counts*. 

  	We have provided a small shell script [**viral.pipeline_public_final.sh**](https://drive.google.com/file/d/0B3-  883ME4sP3RGI3SF9ha0p4QVk/view?usp=sharing) that encompasses all of these above steps. 

	*The complete archive of the input files, intermediate files, script, and output files from this Bowtie2 alignment step is 	available through google drive [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3NUVrNEtHSndWek0?usp=sharing).*

-	Get genome level matrix file
  -	If you have more than one sample, then it might be useful to concatenate the number of mapped reads (genome level counts) from each sample into a matrix format. This file can then be used for any further analysis.
  -	We have provided an Rmarkdown file that has code and output for how to merge all the `Samtools idx` files. See files [**merge.idx.stats.pdf**](https://github.com/ICBI/viGEN/blob/master/merge.idx.stats.pdf) created from [**merge.idx.stats.Rmd**](https://github.com/ICBI/viGEN/blob/master/merge.idx.stats.Rmd) available in this github repository. This code also adds virus annotation using information from **Complete_Sequence_info.csv** file.
  -	The genome level counts from the above step will help us determine the top virus genomes to use for the next step (Gene and CDS quantitation). We use a cutoff genome level count to shortlist the viruses. 
  -  	For this tutorial, we decided to use a threshold genome level count of 100. So all viral genomes that have “genome level count” >= 100 are short listed. This gives us 43 viral genomes used in the next step. This file is available **[here]**(https://docs.google.com/spreadsheets/d/16vSWxLeUdiTXBNzudObbEGFNbXZroWznDnEtjhNF784/edit?usp=sharing)
  -	For this tutorial, we only use one sample, so finding the top viruses was easy. 
  	
	*Note: If you have multiple samples belonging to case and control sub-groups, we recommend:
	-	averaging the genome level counts in each in each sub-group, 
	-	Find the top viruses in each sub-group
	We plan to provide code for this in our Bioconductor package under preparation.*

### Gene and CDS quantification
-	In this step, we use our in-house pipeline to generate gene and CDS counts is for every input BAM file.
-	The region level information is extracted from Gene Feature Format files (GFF) files which are available for most viral genomes from NCBI.
-	Download GFF files for the top viruses. 
  -	For this tutorial, we have provided the GFF files in folder [**un_bowtie_topGff**](**XXXXXXXXXXXXXX**) through google drive. *We plan to provide code that does automatic download of GFF files from NCBI FTP site in our Bioconductor package.*
  -	*Note: Not all fasta files will have GFF files. In this tutorial, out of 43 top viruses, we obtained GFF files for 40 viruses.*
  - 	Input to our in-house pipeline for Gene & CDS quantification: the viral bam files (the BAM file output from previous Bowtie2 step)
  -	We have provided R markdown file **count.regions.in.regions.Rmd**XXXXXXXXXXXX in this github repository, that generates these Gene and CDS counts. *We plan to integrate this code into our Bioconductor package.*
  -	Output : 
    -	This code produces counts for each region in the GFF file. One file is created for each virus (GFF file) for each sample. Three types of counts are calculated - “in region”, “on boundary” and “in gaps”. This output is available as a “.csv” file and “Rdata” files.
    -	Collate all read counts across all samples and across all top viruses to create a matrix. This code (provided as collate.output.files.Rmd) calculates total of “in region” and “on boundary” (referred to as “sum”) when collating.
    -	Add virus annotation using information from “Complete_Sequence_info.csv” file.
    
    A complete archive of the input, intermediate and output files from this step are available in google drive [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3bFE0WEZWYjgweGM?usp=sharing)

  The viruses that have the highest region counts are shown in table below. You can see various gene/CDS regions of Hepatitis B virus showing up on top with highest counts. This is a verification of the HBV status of the sample
  
  XXXXXXXXX insert table XXXXXXXXX 

    -	Application: These gene and CDS count files can be used to compare case and control groups of interest using popular tools like EdgeR (http://bioconductor.org/packages/edgeR/) or DESeq2 in Bioconductor that can accept raw counts. 
    
### Variant calling
- Install Varscan2 from here: https://sourceforge.net/projects/varscan/files/ 
- Run Varscan2 on command line using the following command: `samtools mpileup -B -f /Users/ls483/Documents/SRA.GEO/viral.reference/viruses.fa -d 9999 -Q 17 -q 17 SRR1946637_un.bowtie2.sorted.bam| java -Xmx2g -jar /Users/ls483/Documents/software/varscan2/VarScan.v2.3.9.jar mpileup2cns --output-vcf 1 --min-var-freq 0.01 | awk '/^#/ || $7=="PASS"' > /Users/ls483/Documents/SRA.GEO/output_varscan2/SRR1946637_un.vcf`
- This produces a the variants found in viruses in a standard variant call file (VCF) file format.

* A complete summary of the input, and output files from this step is available via google drive [**here**](https://drive.google.com/drive/folders/0B3-883ME4sP3d0JOVm9qVDBKcnM?usp=sharing).
  
## Citation
* NCBI SRA http://www.ncbi.nlm.nih.gov/bioproject/PRJNA279878
* EBI SRA http://www.ebi.ac.uk/ena/data/view/SRR1946637
If you are using these samples for testing this pipeline, please remember to cite this dataset from NCBI
