# Home

## Overview

This tutorial provides a step-by-step guide on the use of DNAnexus to perform a GWAS on the UK Biobank data.
We will run everything from the command line, but will monitor our jobs on the project's web page.

This tutorial assumes that you already have a researcher account on the UKBiobank site, and have access to a DNAnexus project.
If not, please create an account [here](https://ams.ukbiobank.ac.uk/ams/signup), and once it has been validated, create an account on [UKB-RAP](https://ukbiobank.dnanexus.com/register) with your UK Biobank credentials to access DNAnexus. To setup your first project, please check the [official documentation on the matter](https://dnanexus.gitbook.io/uk-biobank-rap/getting-started/quickstart/creating-a-project).

This tutorial will guide you through every step needed in order to perform a basic GWAS using whole genome sequences for chromosome 1 to 22 for both [PLINK2](https://www.cog-genomics.org/plink/2.0/) and [regenie](https://rgcgithub.github.io/regenie/). In order to parallelize the analyses, we will perform 22 different GWAS, one for each chromosome, and combine the results locally to reduce cost.

As an example, we will perform a linear regression on the **BMI index** ([21001](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21001)) using **whole genome sequencing data**, specifically the **interim 200k release** (`"/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release/"`). However, you can use any data that you need, simply keep in mind that paths need to be changed in the scripts.

> This tutorial is written for Linux operating systems. Commands may vary accross operating systems.

The [official GWAS guide using Alzheimer's disease](https://dnanexus.gitbook.io/uk-biobank-rap/science-corner/gwas-using-alzheimers-disease) and the [official github page for a GWAS on the Research Analysis Platform using regenie](https://github.com/dnanexus/UKB_RAP/tree/main/GWAS) were useful material when writing the [regenie section](regenie.md).

If you want to try other tools on DNAnexus, we recommend the following github page: [ukb-rap-tools](https://github.com/pjgreer/ukb-rap-tools) by [Phil Greer](https://github.com/pjgreer).

## Durations and costs

The tutorial has three main sections: [Input files](input.md), [Using PLINK2](plink.md) and [Using regenie](regenie.md).

> Please note, when first using your account you have an initial credit of £40. Running all of this tutorial with the same instance and priority as us might go over this budget.
> If it is not done already, your project should be billed to a wallet which is different from your initial credit.

### Input files

The [first section](input.md) is done locally and does not cost anything.

> Please be aware that storing files onto DNAnexus will result in a monthly cost. You may check its current value in the *SETTINGS* tab on your project's web page, or compute it using the [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf).

### Using PLINK2

The [second section](plink.md) will run 22 different jobs (one per chromosome). The cost and duration of the job depends on the instance used.
With our chosen instance (mem1_ssd1_v2_x16), with a `high` priority, the whole GWAS will take about **200 minutes** (3h20) and cost around **£17.04** (according to the [UKB RAP Rate Card v3.0](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf), for a total execution time of 2341 minutes).

> Please note that this cost can be reduced drastically using a `low` priority. If none of the jobs are interrupted, with this instance, it will cost only **£4.56**.
> However it is most likely that jobs will be interrupted. In our experience, the whole GWAS using this instance on a `low` priority with some interruptions has cost around **£9.30** altough it took almost **7h30** to complete.

### Using regenie

The [third section](regenie.md) runs more jobs than other sections, with a total of X different jobs.

## Requirements

To follow this tutorial, you will only need [Python 3](https://www.python.org/downloads/).

## Final architecture

At the end of this tutorial, your DNAnexus project's architecture should look like this:

```text
├── Bulk
├── Showcase metadata
├── gwas_tutorial
│   ├── plink_gwas_BMI
│   │   ├── sumstat_c1.BMI.glm.linear
│   │   ├── ...
│   │   └── sumstat_c22.BMI.glm.linear
│   ├── reg_gwas_BMI
│   │   ├── merge
│   │   ├── QC_lists
│   │   ├── sumstat_c1_BMI.regenie
│   │   ├── ...
│   │   └── sumstat_c22_BMI.regenie
│   ├── covariates.txt
│   ├── plink_BMI.txt
│   ├── reg_BMI.txt
│   └── white_british.txt
├── app-id
└── app-id.dataset
```

> This will vary based on whether you use both PLINK2 and regenie, or only one, and if you have changed files/repertory names. Please note, if modified, files and repertory names have to be the same across all commands.
