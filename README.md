# CCS-Metatranscriptome

#### Metatranscriptome bioinformatic pipeline for eukaryotic phytoplankton from the California Current System.

by Yuan Yu (Johnson) Lin

## Introduction - Diatoms exhibit proactive transcriptomic response to the Upwelling Conveyor Belt Cycle (UCBC)

Phytoplankton in the California Current System (CCS) are important drivers of biogeochemical cycling and food web dynamics leading up to fish stocks and marine mammals - important aspects of the ecosystem and economy. A unique feature of the CCS is the concept of the Upwelling Conveyor Belt Cycle, which is characterized by a series of idealized zones directly related to the respective light conditions, nutrient status, and biology. Diatoms - a silicifying taxa of the CCS phytoplankton assemblage - are known to dominante the upwelling blooms in this region due to their physiological capabilities. However, our understanding of the molecular shifts within the major phytoplankton groups remain largely understudied. This study aims to address the bioinformatic tools used to assess the transcriptomic and metatranscriptomic activity of marine microalgae in response to the UCBC.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Bioinformatic Pipeline

### Introduction

Sequencing is a burgeoning tool used to characterize marine microorganisms in the world's oceans. As a result, phytoplankton groups are still poorly annotated given the relatively scarce reference genomes/transcriptomes out there. However, as RNA-seq is increasingly used in oceanography, our knowledge of 'omics-based tools will continue to expand. In our current state, we can still derive a great source of knowledge from the large datasets collected from the ocean environment. 

Below is a detailed breakdown of the workflow used to evaluate the metatranscriptomics of the coastal phytoplankton communities. Prior to this workflow, you will have obtained the raw reads (sample.fastq.gz) generated by Illumina HiSeq. This pipeline will be run exclusively on the Linux-based Longleaf cluster at the University of North Carolina, Chapel Hill. Each step of the pipeline may be resource intensive, so it is best to submit them as jobs using the SLURM scheduler. To do this, simply create your job file using:

``nano job.sh``

and copy and paste the below scripts into the job file. Take note of how much memory you give to the job, for steps that do not require as many resources like transcriptomes, 16G should be enough, but for larger metatranscriptomic datasets, as much as 250G can be used.

In our scenario, we'll use 3 samples:
<br> S1 for sample 1
<br> S2 for sample 2
<br> S3 for sample 3

*Additionally, take note of where your working directory will be and which directory you want your outputs to be in. Anything displayed in brackets [] here will be subject to change according to your project directory. When you actually run the code on the command line, take these brackets out.

### 1. Quality Control

The first step of the workflow will be to get rid of bad reads and adapter sequences by trimming. There is an abundance of adapter trimming tools out there, but for the purpose of our study, we will use Trim Galore. We will also use fastqc to obtain summary statistics of your trimmed reads. You can also run fastqc on your raw reads to compare with that of your trimmed reads. Keep in mind that the below script uses a "for" loop to perform Trim Galore on all of your samples at once.

```
#!/bin/bash

indir=[directory containing your raw reads generated by Illumina HiSeq, i.e. /proj/lab/project/raw_reads]
outdir=[directory containing your trimmed reads, i.e. /proj/lab/project/trimmed_reads]

samples='S1 S2 S3'

for s in $samples; do
        echo ${s}
	jobfile="trim${s}.sh"
	echo $jobfile
	cat <<EOF > $jobfile
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --time=0-24:00:00
#SBATCH --mem=16G
#SBATCH --ntasks=4
#SBATCH -J trim
#SBATCH -o trim.%A.out
#SBATCH -e trim.%A.err

module load trim_galore
module load pigz
echo 'BEGIN'
date
hostname
trim_galore -j 2 --illumina --paired --fastqc -o [outdirectory: /proj/lab/project/trimmed_reads] \\
$indir/${s}_R1_001.fastq.gz \\
$indir/${s}_R2_001.fastq.gz

echo 'END'
date

EOF

sbatch $jobfile

done 
```

### 2. Assembly

After we generated the trimmed reads, we can assemble the reads using rnaSPAdes, which is a tool built for fast and parallel de novo transcriptome assembly. The output will be a fasta file containing a list of assembled contigs based on your forward and reverse trimmed reads.

```
#!/bin/bash

indir=[directory containing trimmed reads, i.e. /proj/lab/project/trimmed_reads]
outdir=[directory containing assembled reads for each of your samples in .fasta format, i.e. /proj/lab/project/assembly]

samples='S1 S2 S3'

for s in $samples; do
        echo ${s}
	jobfile="assembly${s}.sh"
	echo $jobfile
	cat <<EOF > $jobfile
#!/bin/bash
#SBATCH --nodes=1
#SBATCH --time=0-24:00:00
#SBATCH --mem=16G
#SBATCH --ntasks=4
#SBATCH -J assembly
#SBATCH -o assembly.%A.out
#SBATCH -e assembly.%A.err

module load spades
echo 'BEGIN'
date
hostname
rnaspades.py \
 --pe1-1 $indir/${s}_R1_001_val_1.fq.gz \\
 --pe1-2 $indir/${s}_R2_001_val_2.fq.gz \\
 -o $outdir
echo 'END'
date

EOF

sbatch $jobfile

done
```
This job will produce a transcripts.fasta file for each of your samples in a separate directory created by the outdir command. The next step is to assemble all of these .fasta files into a grand assembly which will act as a transcriptome for your next steps. To do so, you will have to concatenate each transcripts.fasta file using the `./cat` command on the cluster, and name your file accordingly. In this example, we'll name our file assembly.fasta, for future scripts that will use this mega assembly.

### 3. CD-HIT-EST

After you have generated the grand assembly, you will perform CD-HIT-EST to cluster and reduce the redundancy of your sequences. It is an important tool used in handling large datasets and can improve the performance of other sequencing analyses. Information on this program can be found here: https://sites.google.com/view/cd-hit

```
#!/bin/bash

#SBATCH -p snp
#SBATCH -q snp_access
#SBATCH --nodes=1
#SBATCH --time=5-00:00:00
#SBATCH --mem=700g
#SBATCH --ntasks=255
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=johnsony@live.unc.edu
#SBATCH -J cdhit
#SBATCH -o cdhit.%A.out
#SBATCH -e cdhit.%A.err

cd /proj/lab/project/assembly

hostname
date
cd-hit-est -i assembly.fasta -o assembly_cdhit.fasta -T 12 -c 1 -n 11 -d 0 -M 700000 -T 255 -G o -aS 1 -aL 1
date
```

### 4. Annotation

Steps 4 and Step 5 (Alignment) can be run at the same time, with annotation, we will need to annotate your assemblycdhit.fasta file to both a functional (KEGG) and taxonomic (phyloDB) database using the DIAMOND command.

#### Functional

```
#!/bin/bash

#SBATCH -p snp
#SBATCH -q snp_access
#SBATCH --nodes=1
#SBATCH --time=5-00:00:00
#SBATCH --mem=700g
#SBATCH --ntasks=255
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=johnsony@live.unc.edu
#SBATCH -J kegg
#SBATCH -o kegg.%A.out
#SBATCH -e kegg.%A.err

module load diamond

indir=[directory containing your clustered and de-duplicated grand assembly, i.e. /proj/lab/project/assembly]
outdir=[directory containing your output from DIAMOND blast, i.e. /proj/lab/project/annotation]

diamond makedb --in /nas/longleaf/data/KEGG/KEGG/genes/fasta/genes.pep.fasta -d keggdb

diamond blastx -d keggdb \
-q /proj/lab/project/assembly_cdhit.fasta
-o /proj/lab/project/annotation/KEGG.m8 \
-p 12 -e 0.001 -k 10
```

#### Taxonomic

```
#!/bin/bash

#SBATCH -p general
#SBATCH --nodes=1
#SBATCH --time=0-24:00:00
#SBATCH --mem=500G
#SBATCH --ntasks=12
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=johnsony@live.unc.edu
#SBATCH -J UTphylo
#SBATCH -o UTphylo.%A.out
#SBATCH -e UTphylo.%A.err

module load diamond

cd /proj/lab/project/annotation
outdir=/proj/lab/project/annotation/
diamond makedb --in /proj/marchlab/data/phylodb/phylodb_1.076.pep.fa -d phylodb

diamond blastx -d /proj/marchlab/data/phylodb/diamond_db/phylodb \
-q /proj/lab/project/assembly_cdhit.fasta \
-o /proj/lab/project/annotation/phyloDB.m8 \
-p 12 -e 0.000001 -k 1
```

### 5. Alignment

Now that we have our pair-end sequences, the grand assembly, and the annotations for the assembly, the final step is to quantify the sequence counts and abundances so we can examine the transcript expression. We will be using the tool SALMON to perform this task. However, we will first need to convert our grand assembly into a salmon index from which we will align our sequences. The following is the script to do this:

```
#!/bin/bash

#SBATCH -p bigmem
#SBATCH -q bigmem_access
#SBATCH -N 1
#SBATCH -t 01-00:00:00
#SBATCH --mem=500g
#SBATCH --cpus-per-task=36
#SBATCH --constraint="[rhel7|rhel8]"
#SBATCH --mail-type=BEGIN,END,FAIL
#SBATCH --mail-user=johnsony@live.unc.edu
#SBATCH -J salmonindex
#SBATCH -o salmonindex.%A.out
#SBATCH -e salmonindex.%A.err
 
module add salmon

echo 'BEGIN'
date
hostname

cd /proj/lab/project/assembly
outdir=/proj/lab/project/alignment

salmon index -t /proj/lab/project/assembly/assembly_cdhit.fasta -i salmon_index

echo 'END'
date
```

After the previous step, you will then perform the alignment using SALMON. Again, we will be using a for loop to align all of the samples to your salmon_index at once. The job will produce a corresponding file for your alignment output for each sample, containing quant.sf files that you will import into DESeq2 later on.

```
#!/bin/bash

indir=[directory containing your trimmed reads, i.e. /proj/lab/project/trimmed_reads]
outdir=[directory containing your SALMON alignment output, i.e. /proj/lab/project/alignment]
mkdir -p Psn

samples='S1 S2 S3'

for s in $samples; do
    echo ${s}
    R1=`ls -l $indir | grep -o ${s}_R1.fq.gz`
    R2=`ls -l $indir | grep -o ${s}_R2.fq.gz`
    echo ${R1}
    echo ${R2}
    jobfile="psnquant${s}.sh"
    echo $jobfile
    cat <<EOF > $jobfile
#!/bin/bash
#SBATCH -N 1
#SBATCH -t 05-00:00:00
#SBATCH --mem=250g
#SBATCH -n 16
#SBATCH -o salmonalign.%A.out
#SBATCH -e salmonalign.%A.err


module add salmon
echo 'BEGIN'
date
hostname
salmon quant -i /proj/lab/project/alignment/salmon_index -l A \\
        -1 $indir/${R1} \\
        -2 $indir/${R2} \\
        -p 5 --validateMappings \\
        -o quants/${s}_quant
echo 'END'
date

EOF

    sbatch $jobfile

done
```



