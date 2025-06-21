# MicroAS-seq Pipeline

## Description

MicroAS-seq is a targeted multiplex amplification library construction protocol to detect micro-exon skipping. Here is a comprehensive pipeline for demultiplexing and quantifying sequencing data from MicroAS-seq experiments. This pipeline processes multiplexed sequencing data through adapter trimming, barcode demultiplexing, alignment, and quantification. It's specifically designed for analyzing MicroAS-seq or similar multiplexed sequencing data.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Environment Setup](#environment-setup)
4. [Index Files Deployment](#index-files-deployment)
5. [Input Data Organization](#input-data-organization)
6. [Running the Pipeline](#running-the-pipeline)
7. [Demo Test](#demo-test)
8. [Output Files](#output-files)
9. [Troubleshooting](#troubleshooting)
10. [Citation](#citation)

## Prerequisites

- Linux/Unix operating system
- Conda package manager (Miniconda or Anaconda)
- Basic bioinformatics tools (wget, tar, gzip)
- Sufficient storage space for index files (~30GB recommended)

## Environment Setup

The pipeline uses a single Conda environment ("microAS") containing all required tools:

- Trim Galore (adapter trimming)
- fastq-multx (barcode demultiplexing)
- STAR (alignment)
- Samtools (BAM processing)
- deepTools (normalization)
- featureCounts (quantification)

### 1. Clone the repository

```bash
cd the_dir_you_want_to_install_the_pipeline
git clone https://github.com/princewang2018/MicroAS-seq.git
cd MicroAS-seq
chmod +x microAS

microAS_PATH=$(pwd)
sudo echo "export PATH=\$PATH:$microAS_PATH" >> ~/.bashrc
source ~/.bashrc
```

### 2. Set up Conda environment

```bash
# Install mamba for faster dependency resolution
conda install -n base -c conda-forge mamba -y

# Create environment with Python 3.8
mamba create -n microAS python=3.8 -y
mamba activate microAS

# Install all required tools
mamba install -c bioconda \
    trim-galore \
    fastq-multx \
    star \
    samtools \
    deeptools \
    subread \
    -y
```

### 3. Verify installation

```bash
# Activate your environment
conda activate microAS

# Check all tools are working
trim_galore --version
STAR --version
samtools --version
deeptools --version
featureCounts -v
fastq-multx -h | head -n 5
```

Expected output versions:

- trim-galore: 0.6.6+ 
- STAR: 2.7.3a+
- samtools: 1.9+
- deeptools: 3.1.3+
- featureCounts: 2.0.0+

## Index Files Deployment

### 1. Download genome references

Example for hg38:

```bash
mkdir -p refs/hg38
cd refs/hg38
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/genes/hg38.knownGene.gtf.gz
gunzip *.gz
```

### 2. Build STAR index

```bash
conda activate microAS
mkdir -p /path/to/your/STAR_index_hg38
STAR --runThreadN 16 \
     --runMode genomeGenerate \
     --genomeDir /path/to/your/STAR_index_hg38 \
     --genomeFastaFiles refs/hg38/hg38.fa \
     --sjdbGTFfile refs/hg38/hg38.knownGene.gtf \
     --sjdbOverhang 149
```

You can also use pre-built STAR index compatible with STAR (corresponding version).

## Input Data Organization

Organize your input FASTQ files as follows:

```
work_directory/
├── barcode.txt           # Barcode file
├── sample_1.fq.gz        # Read 1
└── sample_2.fq.gz        # Read 2
```

The barcode file should contain one pair of barcodes per line in the format (tab separation):
```
FT1	ATCACG-CTTGTA
FT2	CGATGT-GGCTAC
FT3	TTAGGC-TAGCTT
FT4	TGACCA-GATCAG
FT5	ACAGTG-ACTTGA
...
```

## Running the Pipeline

Basic command:

```bash
conda activate microAS
cd /path/to/work_directory
microAS \
  -w /path/to/work_directory \
  -b /path/to/barcode.txt \
  -s /path/to/STAR_index_directory \
  -a /path/to/annotation.gtf \
  -t 8
```

Parameters:
- `-w`: Working directory containing input FASTQ files (**Must use absolute path**)
- `-b`: Path to barcode file (**Must use absolute path**)
- `-s`: Path to STAR index directory (Path)
- `-a`: Path to GTF annotation file, both whole genome gtf and personalized gtf for featurecounts are acceptable
- `-t`: Number of threads (default: 8)

## Demo Test

A small test dataset is available at GitHub. To run the demo:

```bash
# Activate environment
conda activate microAS

# Set variables
## microAS_PATH=/path/to/MicroAS-seq 
STAR_INDEX_PATH=/path/to/STAR_index_hg38 

cd $microAS_PATH/Demo_data
microAS \
  -w $microAS_PATH/Demo_data \
  -b $microAS_PATH/Demo_data/barcode_template_FT14.txt \
  -s $STAR_INDEX_PATH \   #Please modified this to your: /path/to/STAR_index_directory
  -a $microAS_PATH/Demo_data/CD47_hg38_gencode_v33.gtf \
  -t 8
```

**Note:** You need to create a STAR index before running the demo. See [Index Files Deployment](#index-files-deployment) section.

## Output Files

The pipeline creates the following directory structure:

```
work_directory/
├── 03.Cleandata/         # Adapter-trimmed files
├── 04.Fastqmultx/        # Demultiplexed FASTQ files
├── 05.Alignment/         # Alignment files (BAM, BAI)
├── 06.Quantification/    # Quantification results
└── microAS_workflow.done # Completion marker
```

## Troubleshooting

1. **Conda environment issues**:
   - Recreate environment if packages fail to load
   
   - If conda activate doesn't work, try:
   
     ```shell
     eval "$(conda shell.bash hook)"
     conda activate microAS
     ```
   
2. **deepTools installation issues**:
   
   - Use Python 3.8+ environment for better compatibility
   - See [deeptools_installation_guide.md](deeptools_installation_guide.md) for detailed solutions
   - If persistent, create separate environment: `mamba create -n deeptools_only python=3.8 deeptools -c bioconda -y`
   
3. **Dependency conflicts**:

   - Clear conda cache: `conda clean --all`
   - Try: `mamba install --force-reinstall [package_name]`
   - Recreate environment if packages fail to load

4. **STAR index creation**:

   - Requires ~30GB disk space and 32GB+ RAM
   - Use `--limitGenomeGenerateRAM` to control memory usage
   - For demo, you can download pre-built indices

5. **Barcode demultiplexing errors**:

   - Verify barcode file format (tab-separated)
   - Check for barcode mismatches
   - Ensure barcode file has Unix line endings: `dos2unix barcode.txt`

6. **Memory errors**:

   - Reduce the number of threads if you encounter memory issues
   - STAR alignment may require significant RAM for large genomes
   - Use `--limitBAMsortRAM` for STAR alignment

7. **Input file errors**:

   - Verify FASTQ files are properly named and paired (ending with `_1.fq.gz` and `_2.fq.gz`)
   - Check file integrity with `zcat file.fq.gz | head`
   - Ensure files are in the work directory specified by `-w`

8. **Annotation file problems**:

   - Ensure GTF file matches your genome version
   - Verify featureCounts can read the annotation file
   - Check GTF format: `head -n 10 annotation.gtf`

9. **Path issues**:

   - Use absolute paths for all parameters (`-w`, `-b`, `-s`, `-a`)
   - Avoid spaces in file paths
   - Check file permissions: `ls -la /path/to/files`

## Citation

If you use this pipeline in your research, please cite:

> Wang, Z., Yang, L., Yang, S., Li, G., Xu, M., Kong, B., Shao, C., & Liu, Z. (2025). Isoform switch of CD47 provokes macrophage-mediated pyroptosis in ovarian cancer. bioRxiv, 2025.2004.2017.649282. https://doi.org/10.1101/2025.04.17.649282

For questions or issues, please open an issue on GitHub or contact 970214035yl@gmail.com, wangzixiang@sdu.edu.cn.