[![R](https://img.shields.io/badge/-script-276DC3.svg?style=flat&logo=R)](https://cran.r-project.org)

# Project summary

+ Dataset is composed of Perflourooctanesulfonic acid (PFOS) exposed (72h) normal human breast cell line (MCF-10A). 12 samples, 6 control, 6 exposed.
+ The sequencing data is built upon the study: [PFOS induces proliferation, cell-cycle progression, and malignant phenotype in human breast epithelial cells](https://doi.org/10.1007/s00204-017-2077-8). From this study only the control and 1µM cells cultures were used.
+ As PFOS has potentially [endocrine disrupting estrogen activity](https://doi.org/10.1111/j.1365-2605.2008.00870.x) and has been [linked to breast cancer](https://doi.org/10.1186/1476-069X-10-88), the hypothesis is to identify DM regions which overlap genes related to the phenotypes listed above.
+ DNA methylation levels were mapped with Enzymatic Methyl sequencing (EM-seq)

>The code in the repository was used for computational analysis for the paper: [Perfluorooctanesulfonic acid (PFOS) induced cancer related DNA methylation alterations in human breast cells: A whole genome methylome study](https://doi.org/10.1016/j.scitotenv.2024.174864)

# Analysis workflow

Project file structure

```
project/
├── code/
├── data/
├── dump/     # intermediate files
├── GRCh38/   # complementary files
├── results/  # produced by nextflow
├── seqdata/  # sequencing data
└── README.md
```

## Mapping & coverage

Run [nf-core methylseq pipeline](https://nf-co.re/methylseq) to align the sequencing reads to the reference genome and generate the methylation coverage files.

>Sequencing data will be made available upon request

```sh
# Ran on HPC (UPPMAX). Use login node or pipeline is killed when node is killed

# setup env
PROJECT=""
EMAIL=""
PROJDIR=""
cd $PROJDIR

# Load modules
ml bioinfo-tools 
ml Nextflow
nextflow pull nf-core/methylseq

# Nextflow parameters
export VERSION=1.6.1
export NXF_HOME=${PROJDIR}
export PATH=${NXF_HOME}:${PATH}
export NXF_TEMP=$SNIC_TMP
export NXF_LAUNCHER=$SNIC_TMP
export NXF_OPTS='-Xms1g -Xmx4g'

nextflow run nf-core/methylseq -r $VERSION \
  -profile uppmax \
  --project $PROJECT \
  --genome GRCh38 \
  --em_seq \
  --input 'seqdata/*_R{1,2}.fastq.gz' \
  --outdir 'results' \
  --aligner bismark \
  --email $EMAIL \
  -resume
```

## Complementary files

To elucidate the relevance of identified differentially methylated regions (DMRs) information about overlapping genomic features is needed (such as promoters, exon, intron, CpG-island). The database hosted at [University of California Santa Cruz (UCSC) Genomics Institute](https://genome-euro.ucsc.edu/index.html) holds genomic features for various species. Annotations can be exported from the [Table Browser tool](https://genome-euro.ucsc.edu/cgi-bin/hgTables). To download the annotations set the parameters as follow:

1. CpG-islands annotations, save as `cpgislands_GRCh38.bed`

```
clade = "Mammals" 
genome = "Human"
assembly = "Dec. 2013 (GRCh38/hg38)"
group = "Regulation"
track = "CpG-islands"
table = "cpgIslandExt"
output format = "BED"
output filename = "cpgislands_GRCh38.bed"

<click> "get output" 
<click> "get BED"
```

2. [Refseq](https://en.wikipedia.org/wiki/RefSeq) annotations, save as `refseq_UCSC_GRCh38.bed`

```
clade = "Mammals" 
genome = "Human"
assembly = "Dec. 2013 (GRCh38/hg38)"
group = "Genes and Gene Predictions"
track = "NCBI RefSeq"
table = "UCSC RefSeq (refGene)"
output format = "BED"
output filename = "refseq_UCSC_GRCh38.bed"

<click> "get output" 
<click> "get BED"
```

3. [TCGA database](https://portal.gdc.cancer.gov) holds information about genes affected by differences in methylation related to cancer. Save as `frequently-mutated-genes.2023-12-13.tsv`

>This table is included for reproducablility as it holds a screenshot of the queried TCGA database. Generating a new might generate other results. However, feel free to do so to get update results!

```
# go to: https://portal.gdc.cancer.gov, navigate to "Projects" tab. 
# In the left panel, choose breast as "Primary Site" and methylation 
# array as "Experimental Strategy". This will filter out 3 projects of
# which "TCGA-BRCA" is the best match as it contains only the breast 
# tissue and 1,100 cases

# To access the mutated gene names navigate to "Exporation" tab. 
# In the left panel, choose  "TCGA-BRCA" as "Projects". 
# Click on the TSV button (on the right hand side) to download the top genes. 

# NOTE, for rendering limitations the homepage will only show up to 100 genes.
# To increase this number use the URL below and change "genesTable_size=" to 
# a number > 100. Below, I use 2000:

#https://portal.gdc.cancer.gov/exploration?facetTab=genes&filters=%7B%22op%22%3A%22and%22%2C%22content%22%3A%5B%7B%22op%22%3A%22in%22%2C%22content%22%3A%7B%22field%22%3A%22cases.primary_site%22%2C%22value%22%3A%5B%22breast%22%5D%7D%7D%2C%7B%22op%22%3A%22in%22%2C%22content%22%3A%7B%22field%22%3A%22cases.project.project_id%22%2C%22value%22%3A%5B%22TCGA-BRCA%22%5D%7D%7D%5D%7D&genesTable_size=2000&searchTableTab=genes
```

+ ensembl database and reference genome CpG-site positions, run `code/complementary_files.R`, will generate `ensembl_dataset_GRCm39.csv.gz` and `cg_pos_CRGh38.csv.gz`

```sh
Rscript code/complementary_files.R
```

The `data/` and `GRCh38/` folder should contain the following: 

```
data/
└── frequently-mutated-genes.2023-12-13.tsv

GRCh38/
├── cg_pos_CRGh38.csv.gz
├── cpgislands_GRCh38.bed
├── ensembl_dataset_GRCh38.csv.gz
└── refseq_UCSC_GRCh38.bed
```

## Differental methylation analysis

1. DMRs were divided into 2 resolutions, (1) CpG and (2) tile of 100 bp. CpG-sites with low coverage (< 10 reads) and the top 99th percentile (PCR duplicates) were removed. Normalisation was done with scaling factor between samples based on differences between median of coverage distribution. The tiles were only considored if 2 or more CpG-sites where present. Finally, on a group level (control and exposed) CpG-sites were considored if 66% of the samples (4 out of 6) had coverage. Standard deviation (SD) filtering was applied where CpG-site with < 2 SD (little to no variation) were removed as they would not contribute information for downstream analysis.

```sh
# Ran at HPC (UPPMAX)
sbatch code/diffmeth.sh

## will start code/diffmeth.R with different arguments for CpG resolution
```

The `dump/` and `data/` folder should contain the following:

```
dump/
├── diffmeth_1_cpg.csv.gz
└── diffmeth_1_tile100.csv.gz

data/
├── PFOS_MCF-10A_betavalues_matrix_cpg.Rds
└── PFOS_MCF-10A_betavalues_matrix_tile100.Rds
```

2. Generate 3 tables: DMR and DMG, (1) DMR = each row is a dmr_id, (2) DMG = each row is gene with info about DMRs within it, genomic regions, dmr_id, hyper/hypo etc. Significance threshold for DMRs were set to qvalue < 0.05 and meth.diff > ±15 and ±5, CpG-sites and 100 bp tiles, respectively. (3) CGI = each row CpG-island. (4) GO analysis based on genomic regions of significant DMRs: promoter, exon, intron, CGI. The genes used as universe were all genes found in the ensembl database.

```sh
Rscript code/methtable.R
Rscript code/genetable.R
Rscript code/go_analysis.R
```

The `data/` folder should contain the following:

```
data/
├── PFOS_MCF-10A_DMG.Rds
├── PFOS_MCF-10A_DMR.Rds
└── PFOS_MCF-10A_GO.Rds
```

---

>This project was made by the [Karlsson Laboratory](https://karlssonlab.se/research/) at Stockholm University, Sweden.