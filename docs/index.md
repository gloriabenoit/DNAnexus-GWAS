# Home

## Overview

This tutorial provides a step-by-step guide on the use of DNAnexus to perform a GWAS on the UK Biobank data.
We will run everything from the command line, but will monitor our jobs on the project's web page. 
If you'd rather use the terminal, we've also built a package, named [dxlog](https://pypi.org/project/dxlog/), which does mostly the same as the web page but directly from the command line.

This tutorial assumes that you already have a researcher account on the UKBiobank site, and have access to a DNAnexus project.
If not, please [create an account](https://ams.ukbiobank.ac.uk/ams/signup), and once it has been validated, create an account on [UKB-RAP](https://ukbiobank.dnanexus.com/register) with your UK Biobank credentials to access DNAnexus. To setup your first project, please check the [official documentation on the matter](https://dnanexus.gitbook.io/uk-biobank-rap/getting-started/quickstart/creating-a-project).

This tutorial will guide you through every step needed in order to perform a basic GWAS using whole genome sequences for chromosome 1 to 22 for both [PLINK2](https://www.cog-genomics.org/plink/2.0/) and [regenie](https://rgcgithub.github.io/regenie/). In order to parallelize the analyses, we will perform 22 different GWAS, one for each chromosome, and combine the results locally to reduce cost.

> PLINK2 is the most basic and simple tool to perform a GWAS, while regenie is a bit more advanced.
> If you need to choose only one, please perform the [PLINK2 tutorial](plink.md), as it is simpler, quicker and way cheaper to complete.

As an example, we will perform a linear regression on the **BMI index** ([21001](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21001)) using **whole genome sequencing data**, specifically the **DRAGEN WGS 500k release** (`/Bulk/DRAGEN WGS/`, data release *v19.1*) and the **GraphTyper WGS 200k release** (`/Bulk/Previous WGS releases/GATK and GraphTyper WGS/`, data release *v15.1*).
However, you can use any data that you need, simply keep in mind that paths need to be changed in the scripts, and execution time will vary.

> This tutorial is written for Linux operating systems. Commands may vary accross operating systems.

The [official "GWAS guide using Alzheimer's disease"](https://dnanexus.gitbook.io/uk-biobank-rap/science-corner/gwas-using-alzheimers-disease) and the [official github page for a "GWAS on the Research Analysis Platform using regenie"](https://github.com/dnanexus/UKB_RAP/tree/main/GWAS) were useful material when writing the [regenie section](regenie.md).

If you want to try other tools on DNAnexus, we recommend the following github page: [ukb-rap-tools](https://github.com/pjgreer/ukb-rap-tools) by [Phil Greer](https://github.com/pjgreer).

## Structure

The tutorial is separated into four main sections:

1. Fondamentals for first-time use of DNAnexus ([Getting started](start.md), [About jobs](jobs.md))
2. Necessary files for GWAS input ([Input files](input.md))
3. Running a GWAS ([Using PLINK2](plink.md), [Using regenie](regenie.md))
4. Generating plots from results ([Visualizing results](results.md))

> Please note, the [PLINK2](plink.md) and [regenie](regenie.md) sections are independant of each other and can be done separately. However, you will need to follow first the [Getting started](start.md) and [Input files](input.md) pages to make sure you have everything necessary to their completion.

## Requirements

To follow this tutorial, you will only need [Bash](https://en.wikipedia.org/wiki/Bash_(Unix_shell)) and [Python 3](https://www.python.org/downloads/).

## Total cost

Please note, when first using your account you have an initial credit of £40.
Running all of this tutorial with the same instance and priority as us should not go over this budget.
However, if it is not done already, your project should be billed to a wallet which is different from your initial credit.
Please check the [official documentation](https://documentation.dnanexus.com/admin/billing-and-account-management) if you are unsure on how to proceed.

Jobs cost will be computed based on the official [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf) (v3.0).

> Please remember the cost and duration of a job depends on the instance and priority used. Execution time may also vary for the same instance.

## Final architecture

At the end of this tutorial, your DNAnexus project's architecture should look like this:

```text
├── Bulk/
├── Showcase metadata/
├── gwas_tutorial/
│   ├── plink_gwas_BMI/
│   │   ├── sumstat_c1.BMI.glm.linear
│   │   ├── ...
│   │   └── sumstat_c22.BMI.glm.linear
│   ├── reg_gwas_BMI/
│   │   ├── merge/
│   │   ├── QC_lists/
│   │   ├── sumstat_c1_BMI.regenie.gz
│   │   ├── ...
│   │   └── sumstat_c22_BMI.regenie.gz
│   ├── BMI.txt
│   ├── covariates.txt
│   └── white_british.txt
├── app-id
└── app-id.dataset
```

> This will vary based on whether you use both PLINK2 and regenie, or only one, and if you have changed files/repertory names. Please note, if modified, files and repertory names have to be the same across all commands.
