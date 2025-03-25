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

## Structure

The tutorial is separated into four main sections:

1. Fondamentals for first-time use of DNAnexus ([Getting started](start.md), [About jobs](jobs.md))
2. Necessary files for GWAS input ([Input files](input.md))
3. Running a GWAS ([Using PLINK2](plink.md), [Using regenie](regenie.md))
4. Generating plots from results ([Visualizing results](results.md))

> Please note, the [PLINK2](plink.md) and [regenie](regenie.md) sections are independant of each other and can be done separately. However, you will need to follow first the [Getting started](start.md) and [Input files](input.md) pages to make sure you have everything necessary to their completion.

## Requirements

To follow this tutorial, you will only need [Python 3](https://www.python.org/downloads/).

## Durations and costs

Please note, when first using your account you have an initial credit of £40. Running all of this tutorial with the same instance and priority as us might go over this budget. If it is not done already, your project should be billed to a wallet which is different from your initial credit. Please check the [official documentation](https://documentation.dnanexus.com/admin/billing-and-account-management) if you are unsure on how to proceed.

Jobs cost will be computed based on the official [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf) (v3.0).

> Please remember the cost and duration of a job depends on the instance and priority used. Execution time may also vary for the same instance.

### Input files

The [input files extraction/upload](input.md) is done locally and does not cost anything.

> Please be aware that storing files onto DNAnexus will result in a monthly cost. You may check its current value in the *SETTINGS* tab on your project's web page, or compute it using the [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf).

### Using PLINK2

The [PLINK2 GWAS](plink.md) will run 22 different jobs (one per chromosome).

With our chosen instance (mem1_ssd1_v2_x16) using a `high` priority, the whole GWAS will take about **200 minutes** (3h20) and cost around **£17.04** (for a total execution time of 2341 minutes).

With the same instance (mem1_ssd1_v2_x16) using a `low` priority, if no jobs are interrupted, it will cost only **£4.56** for the same time.

> Please note that it is most unlikely for jobs to be uninterrupted. In our experience, the whole GWAS using this instance (mem1_ssd1_v2_x16) with a `low` priority and some interruptions has cost around **£9.30** altough it took almost **7h30** to complete from start to finish (with no failed jobs).

### Using regenie

The [regenie GWAS](regenie.md) runs a total of 47 different jobs:

* 22 jobs for the quality control (one per chromosome)
* 2 jobs for the merging and QC of merged data
* 1 job for regenie's step 1 (SNPs contribution estimation)
* 22 job for regenie's step 2 (regression, one per chromosome)

> To optimize your GWAS's overall time, you can perform the QC in parallel to every other step beside step 2 (for which it is needed).
> The rest needs to be done in order.

With our chosen instance (mem1_ssd1_v2_x16) using a `high` priority, the whole GWAS will take about **700 minutes** (11h40, when running each step in order) and cost around **£21.46** (for a total execution time of 2948 minutes).

With the same instance (mem1_ssd1_v2_x16) using a `low` priority, if no jobs are interrupted, it will cost only **£5.74** for the same time.

You can find the details about the jobs run in the following table:

<center>

| Action   |  Time | Execution time | Cost (high) | Cost (low, no interruption) |
|----------|:-----:|:--------------:|:-----------:|:---------------------------:|
| QC       |  2h39 |       30h      |    13.10    |             3.50            |
| Merge    | 51min |      51min     |     0.37    |             0.10            |
| Merge QC | 10min |      10min     |     0.07    |             0.02            |
| Step 1   |  7h10 |      7h10      |     3.13    |             0.83            |
| Step 2   | 48min |      10h57     |     4.78    |             1.28            |

</center>

> Please note that it is most unlikely for jobs to be uninterrupted. In our experience, the whole GWAS using this instance (mem1_ssd1_v2_x16) with a `low` priority and some interruptions has cost around **£12.57** (without accounting for failed jobs, of which there were a few). Since most steps need to be done in order, it also took a total of **54h35** to complete the GWAS from start to finish (over multiple sessions since some jobs failed).
> If you count the failed jobs, you can add £5.36 to the total cost, and 12h04 to the total time taken.

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
