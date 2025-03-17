# Home

## Overview

This tutorial provides a step-by-step guide on the use of DNAnexus to perform a GWAS on the UK Biobank data.
We will run everything from the command line, but will monitor our jobs on the project's web page.

This tutorial assumes that you already have a researcher account on the UKBiobank site, and have access to a DNAnexus project.
If not, please create an account [here](https://ams.ukbiobank.ac.uk/ams/signup), and once it has been validated, create an account on [UKB-RAP](https://ukbiobank.dnanexus.com/register) with your UK Biobank credentials to access DNAnexus.

We will see both the use of [PLINK2](https://www.cog-genomics.org/plink/2.0/) and [regenie](https://rgcgithub.github.io/regenie/) in order to perform a basic GWAS using whole genome sequences for chromosome 1 to 22.

In order to parallelize the analyses, we will perform 22 different GWAS, one for each chromosome, and combine the results locally.

## Requirements

To follow this tutorial, you will need [Python 3](https://www.python.org/downloads/).

## Final architecture

At the end of this tutorial, your DNAnexus project's architecture should look like this:

```text
├── app-id
├── app-id.dataset
├── Bulk
├── metadata
├── Showcase
└── WKD_<your-name>
    ├── covariates.txt
    ├── plink_BMI.txt
    ├── plink_gwas_BMI
    ├── regenie_BMI.txt
    ├── regenie_gwas_BMI
    │   ├── merge
    │   └── QC_lists
    └── white_british.txt
```

> This will vary based on whether you use both PLINK2 and regenie, or only one, and if you have changed files/repertory names. Please note, if modified, files and repertory names have to be the same across all commands.
