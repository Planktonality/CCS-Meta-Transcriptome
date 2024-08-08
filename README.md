# CCS-Metatranscriptome

#### Metatranscriptome bioinformatic pipeline for eukaryotic phytoplankton from the California Current System.

by Yuan Yu (Johnson) Lin

## Introduction - Diatoms exhibit proactive transcriptomic response to the Upwelling Conveyor Belt Cycle (UCBC)

Phytoplankton in the California Current System (CCS) are important drivers of biogeochemical cycling and food web dynamics leading up to fish stocks and marine mammals - important aspects of the ecosystem and economy. A unique feature of the CCS is the concept of the Upwelling Conveyor Belt Cycle, which is characterized by a series of idealized zones directly related to the respective light conditions, nutrient status, and biology. Diatoms - a silicifying taxa of the CCS phytoplankton assemblage - are known to dominante the upwelling blooms in this region due to their physiological capabilities. However, our understanding of the molecular shifts within the major phytoplankton groups remain largely understudied. This study aims to address the bioinformatic tools used to assess the transcriptomic and metatranscriptomic activity of marine microalgae in response to the UCBC.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Bioinformatic Pipeline

### Introduction

Sequencing is a burgeoning tool used to characterize marine microorganisms in the world's oceans. As a result, phytoplankton groups are still poorly annotated given the relatively scarce reference genomes/transcriptomes out there. However, as RNA-seq is increasingly used in oceanography, our knowledge of 'omics-based tools will continue to expand. In our current state, we can still derive a great source of knowledge from the large datasets collected from the ocean environment. 

Below is a detailed breakdown of the workflow used to evaluate the metatranscriptomics of the coastal phytoplankton communities. Prior to this workflow, you will have obtained the raw reads (sample.fastq.gz) generated by Illumina Hiseq.

### 1. Quality Control

The first step of the workflow will be to get rid of bad reads and adapter sequences by trimming. There is an abundance of adapter trimming tools out there, but for the purpose of our study, we will use Trim Galore.



