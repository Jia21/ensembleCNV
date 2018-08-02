# ensembleCNV

## Method Description

EsembleCNV is a novel ensemble learning framework to detect and genotype copy number variations (CNVs) using single nucleotide polymorphism (SNP) array data. EnsembleCNV a) identifies and eliminates batch effects at raw data level; b) assembles individual CNV calls into CNV regions (CNVRs) from multiple existing callers with complementary strengths by a heuristic algorithm; c) re-genotypes each CNVR with local likelihood model adjusted by global information across multiple CNVRs; d) refines CNVR boundaries by local correlation structure in copy number intensities; e) provides direct CNV genotyping accompanied with confidence score, directly accessible for downstream quality control and association analysis. 

More details can be found in the manuscript:

Zhongyang Zhang, Haoxiang Cheng, Xiumei Hong, Antonio F. Di Narzo, Oscar Franzen, Shouneng Peng, Arno Ruusalepp, Jason C. Kovacic, Johan LM Bjorkegren, Xiaobin Wang, Ke Hao (2018) EnsembleCNV: An ensemble machine learning algorithm to identify and genotype copy number variation using SNP array data. bioRxiv 356667; doi: https://doi.org/10.1101/356667 

The detailed step-by-step instructions are listed as follows.

## Table of Contents

- [01 Initial call](#01-initial-call)
  - [Prepare chromosome-wise LRR and BAF matrices](#prepare-chromosome-wise-lrr-and-baf-matrices-for-ensembleCNV)
  - [Prepare data for running individual CNV callers](#prepare-data-for-running-ipq)
    - [Run PennCNV](#call-penncnv)
    - [Run QuantiSNP](#call-quantisnp)
    - [Run iPattern](#call-ipattern)
- [02 Batch effect](#02-batch-effect)
  - [snp-level LRR statistics](#snp-level-lrr-statics)
  - [sample-level IPQ 10 statics](#sample-level-ipq-10-statics)
- [03 Create CNVR](#03-create-cnvr)
  - [create CNVR for individual CNV calling method](#create-cnvr-for-individual-cnv-calling-method)
  - [ensembleCNV](#ensenmblecnv)
- [04 genotype](#04-genotype)
  - [split cnvrs into batches](#split-cnvr-into-batches)
  - [regenotype](#regenotype)
  - [combine prediction results](#combine-prediction-results)
- [05 boundary refinement](#05-boundary-refinement)
- [06 performance assessment](#06-performance-assessment)
  - [compare duplicate pairs consistency rate](#compare-duplicate-pairs-consistency-rate)
- [test](#test)
  - [test ensembleCNV](#test-ensemblecnv)
  - [test regenotype](#test-regenotype)


## 01 Initial call

The pipeline begins with running inividual CNV callers, including [iPattern](https://www.ncbi.nlm.nih.gov/pubmed/?term=21552272), [PennCNV](http://penncnv.openbioinformatics.org/en/latest/), and [QuantiSNP](https://sites.google.com/site/quantisnp/), to make initial CNV calls. The raw data comes from the [final report](http://jp.support.illumina.com/content/dam/illumina-support/documents/documentation/software_documentation/genomestudio/genomestudio-2-0/genomestudio-genotyping-module-v2-user-guide-11319113-01.pdf) generated by Illumina [GenomeStudio](https://support.illumina.com/array/array_software/genomestudio.html). 

In the GenomeStudio, the exported final report text file typically contains 10-columns:
• Sample ID 
• SNP Name 
• Chr
• Position
• allele1-Forward (or allele1-Top)
• allele2-Forward (or allele2-Top)
• X
• Y
• Log R ratio (not used by iPattern)
• B allele freq (not used by iPattern)



The raw data needs to be converted into proper format required by ensembleCNV as well as inividual CNV callers.

### Prepare chromosome-wise LRR and BAF matrices for ensembleCNV

Before running this script, the following data must by supplied.
(1) generate LRR and BAF (tab format) matrix from finalreport
```perl
perl finalreport_to_matrix_LRR_and_BAF.pl \
path_to_finalreport \
path_to_save_matrix_in_tab_format
```
(2) tansform tab format matrix to .rds format
```sh
./tranform_from_tab_to_rds.R path_input path_output chr_start chr_end
```

### prepare data for running IPQ

iPattern
```sh
perl finalreport_to_iPattern.pl \
-prefix path_to_save_ipattern_input_file \
-suffix .txt \
path_to_finalreport
```

PennCNV
```sh
perl finalreport_to_PennCNV.pl \
-prefix path_to_save_penncnv_input_file \
-suffix .txt \
path_to_finalreport
```

QuantiSNP
```sh
perl finalreport_to_QuantiSNP.pl \
-prefix path_to_save_quantisnp_input_file \
-suffix .txt \
path_to_finalreport
```

### call PennCNV

Here, calling PennCNV including following 5 steps:

prepare files containing SNP.pfb and SNp.gcmodel for running PennCNV:
```sh
step.0.prepare.files.sh contains all commands 
```

run PennCNV through submiting jobs:
```sh 
./step.1.run.PennCNV.jobs.R \
-a path/to/dat \
-b path/to/res_job \
-c path/to/SNP.pfb \
-d path/to/SNP.gcmodel \
-e path/to/penncnv/2011Jun16/lib/hhall.hmm
```

check jobs and resubmit unfinishing callings:
```sh
./step.2.check.PennCNV.R \
-a path/to/dat \
-b path/to/res_job \
-c path/to/SNP.pfb \
-d path/to/SNP.gcmodel \
-e path/to/penncnv/2011Jun16/lib/hhall.hmm
```

combine all PennCNV calling results (sample based):
```sh
perl step.3.combine.PennCNV.pl \
--in_dir path/to/res/ \
--out_dir path/to/output/
```

clean PennCNV and generate final results:
```sh
./step.4.clean.PennCNV.R \
-i path/to/result/folder \
-p path/to/SNP.pfb \
-n saving_name
```

### call QuantiSNP

Here, calling QuantiSNP including 3 steps:

prepare QuantiSNP and submit jobs:
```sh
./step.1.prepare.QuantiSNP.R \
-i path/to/data/folder \
-o path/to/result/folder
```
check jobs and resubmit:
```sh
./step.2.check_QuantiSNP.R \
-d path/to/data/folder \
-r path/to/callingCNV/folder 
```

combine CNV calling results:
running this script, you need to add "in_dir", "out_dir", "out_file" information in the script.
```sh
perl step.3.combine.QuantiSNP.pl
```


### call iPattern

sample script for calling iPattern:
```sh
script "run.R" contains all needed running command.
```


## 02 batch effect

### snp-level LRR statics
PCA on snp-level LRR statics from randomly select 100000 snps

```sh
./step.1.randomly.select.snp.R file_snps path_output

perl step.2.generate.snps.LRR.matrix.pl (add "file_snps_selected", "finalreport", "file_matrix_LRR")

step.3.pca.new.R ( add "filename_matrix", "path_input")
``` 

### sample-level IPQ 10 statics
PCA on sample-level iPattern, PennCNV and QuantiSNP generated 10 statics

```sh

generate iPattern, PennCNV and QuantiSNP calling sample level statics data using step.1.generate.data.R script

do PCA using step.2.pca.R
```

## 03 create CNVR

Here, create CNVR for both individual CNV calling method and ensembleCNV.

First, CNV calling results (.rds format) from iPattern, PennCNV and QuantiSNP 
```sh
step.1.data.R ( "path_output", "file_ipattern", "file_penncnv", "file_quantisnp" )
```

Second, 
### create CNVR for individual CNV calling method
individual method:
```sh 
./step.2.create.CNVR.IPQ.R --help for detail
```
### ensembleCNV
ensembleCNV from iPattern, PennCNV and QuantiSNP CNV calling results.
```sh
./step.2.ensembleCNV.R --help for detail
```

Third, generate matrix for individual CNV calling method:
```sh
./step.3.generate.matrix.R file_cnv cnvCaller path_output
```

## 04 genotype

genotyping for all CNVRs containing two main steps:

split all cnvrs generated from ensembleCNV step into chromosome based batches.
```sh
./step.1.split.cnvrs.into.batches.R --help for detail
```

regenotype CNVRs in one batch:
(1) path_data contains:
samples_QC.rds (from PennCNV with columns: LRR_mean and LRR_sd )
duplicate.pairs.rds (two columns: sample1.name sample2.name )
SNP.pfb (from PennCNV)
cnvs_step3_clean.rds (from ensembleCNV step)
cnvrs_batch_annotated.rds (cnvrs adding columns: batch)
(2) path_matrix (LRR and BAF folder to save matrix data)
(3) path_sourcefile (all scripts need to be source )
(4) path_result (save all regenotype results)

```sh
sample code:
./step.2.regenotype.each.chr.each.batch.R \
-c 1 -b 1 -t 0 -p path_data -o path_result -m path_matrix -s path_sourcefile

./step.2.regenotype.each.chr.each.batch.R --help for detail

```

combine all sample-based regenotype results.
and, generate mat_CN.rds (matrix of regenotype copy number),
matrix_GQ.rds (matrix of regenotype gq score), 
CNVR_ID.rds (rownames of matrix),
Sample_ID.rds( columns of matrix).

explation:
path_cnvr (with cnvrs_annotated_batch.rds)
path_pred (with chr-batch-based regenotype results)
path_res (save results: matrix_CN.rds matrix_GQ.rds)
```sh
./step.5.prediction.results.R n.samples path_cnvr path_pred pred_res
```

## 05 boundary refinement

There are 5 steps in boundary refinement, as following:

All scripts are in folder 05_boundary_refinement.

The main part is script named as step.2.boundary_refinement.R:
```sh
./step.2.boundary_refinement.R --help for detail
```

## 06 performance assessment

summary compare results between all CNV calling methods with ensembleCNV method.
copy all following files to path_input:
dup_samples.rds with columns: sample1.name, sample1.name
matrix_iPattern.rds; matrix_PennCNV.rds; matrix_QuantiSNP.rds; 
matrix_IPQ_intersect.rds; matrix_IPQ_union.rds; matrix_ensembleCNV.rds

### compare duplicate pairs consistency rate

```sh
./compare.dups.consistency.R path_input cohort_name path_output
```
## test
here, we supply a samll test example for user to test.

copy all the scripts and data folder in test working folder.

### test ensembleCNV

```sh
mkdir res
./step.2.ensembleCNV.R \
-i ./test_data_ensembleCNV/cnvr1.ipattern.rds \
-p ./test_data_ensembleCNV/cnvr1.penncnv.rds \
-q ./test_data_ensembleCNV/cnvr1.quantisnp.rds \
-s ./test_data_ensembleCNV/SNP.cnvr1.pfb \
-c ./test_data_ensembleCNV/chr_centromere_hg19.rds \
-o ./res
```

### test regenotype

```sh
mkdir script; cp 04_genotype/script/* .
```
and, run test_regenotype.R line by line to generate regenotype copyt number results.


