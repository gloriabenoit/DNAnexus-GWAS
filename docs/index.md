# Home

## Overview

This tutorial provides a step-by-step guide on the use of DNAnexus to performe a GWAS on the UKBiobank data.
We will run everything from the command line, but will monitor our jobs on the project's web page.

This tutorial assumes that you already have a researcher account on the UKBiobank site, and have access to a DNAnexus project.
If not, please create an account [here](https://ams.ukbiobank.ac.uk/ams/signup), and once it has been validated, create an account on [UKB-RAP](https://ukbiobank.dnanexus.com/register) to access DNAnexus.

We will see both the use of [PLINK2](https://www.cog-genomics.org/plink/2.0/) and [regenie](https://rgcgithub.github.io/regenie/) in order to perform a basic GWAS using whole genome sequences for chromosome 1 to 22.

In order to parallelize the analyses, we will perform 22 different GWAS, one for each chromosome, and combine the results locally.

## Requirements

To follow this tutorial, you will need [Python 3](https://www.python.org/downloads/).
