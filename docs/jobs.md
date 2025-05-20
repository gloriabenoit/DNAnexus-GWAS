# About jobs

If this is your first time using DNAnexus, or if you are unsure about how some things work, please read this page. However, be aware that it is not mandatory to complete the tutorial.

For more detailed information, you can check the [official documentation](https://dnanexus.gitbook.io/uk-biobank-rap) for a [quickstart](https://dnanexus.gitbook.io/uk-biobank-rap/getting-started/quickstart) or about the [key concepts](https://dnanexus.gitbook.io/uk-biobank-rap/getting-started/key-concepts) of the platform.

## Execution

On DNAnexus, when a job is executed, a worker is spun up in the cloud, then the job's code and inputs are downloaded to that worker and executed. This is the standard behavior, but inputs can also be mounted (accessed dynamically) to the worker to avoid their downloading costs.

### Inputs

Since jobs are run on workers, you need to manually specify every file used as input.

To add a file as input, you need to use the `-iin` option to your `dx run` command. The path to your file needs to be absolute so that there is no confusion as to where it is located.

When mounting your files, you don't need to input them manually anymore. You simply need to add `/mnt/project/` before your absolute path in the code.

### Instances

To choose an instance for your job is to choose a specific worker on which your code will be executed.
Instances have 3 main metrics:

* The number of cores (individual processing units)
* The memory capacity (components used for short-term data retention)
* The storage capacity (components used for long-term data retention)

You might have no clue on what ressources your job needs to run properly, the simplest course of action is to start with a small instance and increase its capacities if you encounter any errors. Please check the [Errors when running section](#errors-when-running) if you don't know how to interpret your error message.

> Please note, this tutorial uses the same instance for every job. Depending on the data you want to use, you might need a bigger instance. If your goal is full optimization, then you might prefer a smaller one. All of this is up to you.

To specify the instance you chose, you need to add the `--instance-type` option to your `dx run` command.

For more information on instances, check the official documentation on [instance types](https://documentation.dnanexus.com/developer/api/running-analyses/instance-types). Instances costs can be found on the [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf).

### Priority

Jobs can have three level of priority:

* `high` means your job will be completed once the ressources are available
* `normal` means your job will wait for 15min to run in a `low` priority, and if no ressources are availables by then, will run in a `high` priority
* `low` means your job will start once the ressources are available, but may be interrupted if `high` priority jobs need them

Although a `high` priority assures for job completion, it is also pricier than a `low` priority one (around x3 or x5).

> Please note, this tutorial will only run `low` priority jobs to reduce costs.

To specify the level you chose, you need to add the `--priority` option to your `dx run` command.

For more information on priorities, check the official documentation on [job priority](https://dnanexus.gitbook.io/uk-biobank-rap/working-on-the-research-analysis-platform/managing-jobs/managing-job-priority). Priority costs can be found on the [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf) (where `high` refers to *On-demand* prices while `low` refers to *Spot* prices).

### Tags

Tags, although optional, are very pratical when using DNAnexus regularly.
They act as keywords associated to a job, and help if you need to navigate the *MONITOR* tab. You can find more information in the [Monitoring section](#monitoring).

To add a tag to your job, you need to add the `--tag` option to your `dx run` command.

### Time and cost limits

By default, a job will result in an error if its execution time exceeds 30 days. Please note that this value may vary across apps.
During this tutorial, no jobs will exceed this limit.

Setting a cost limit is optional, but can be useful. For instance, when running a `low` priority job, it can ensure that possible interruptions won't add up causing the cost to rise by stopping the job early. It may also be of use if you don't know the time your job will take, and therefore can't compute the total cost.
Either way, it is useful to avoid spending too much on a single job.

To specify a cost limit, you need to add the `--cost-limit` option to your `dx run` command.

For more information on limits, check the official documentation on [time limits](https://documentation.dnanexus.com/user/running-apps-and-workflows/job-time-limits) or on [cost and spending limits](https://documentation.dnanexus.com/user/running-apps-and-workflows/jobs-and-cost-and-spending-limits).

## Monitoring

You can monitor your jobs directly on your project's web page in the *MONITOR* tab.
On this page, they are sorted in ascending order of launch date (the latest comes first).

Basic information about the job is displayed, like its name, duration, cost, *etc*...

A job has 3 possible states:

* `Done` indicates that the job has completed successfully
* `Waiting` indicates that the ressources needed to run this job are not available yet, it will start once they are
* `Failed` indicates the job has failed due to an error

Once a job is completed (either `Done` or `Failed`), you will receive an email updating you of said completion.
It will also appear in the *Notifications* tab (bell icon in the top right corner).

### Filtering jobs

After running a lot of jobs, you might find your *MONITOR* page to be quite crowded. This is especially true with this tutorial where some jobs are run for each of the 22 chromosomes. To find a specific job, you can filter out the *MONITOR* page using specific values like the state or the ID of the user who launched the execution, for instance.

In this tutorial, all jobs have specific [tags](#tags) which help with filtering.
By default, tags are not used in filtering on the *MONITOR* page, you need to add them in the *Filter settings* tab (three stacked bars in the top right corner).

> Please note, in this tutorial, jobs are tagged with the software used (plink or regenie), the phenotype (BMI), the step of the analysis (QC, GWAS, Merge, Step 1 and 2) and lastly the chromosome number (c1 to c22) if jobs are separated per chromosome.

## Errors when running

A failed job running on DNAnexus has 3 main errors (apart from errors in the code):

* Warning: Out of memory error occurred during this job.
* Warning: Low disk space during this job.
* The machine running the job was terminated by the cloud provider

This tutorial will explain the reason for those errors, and what you can do to avoid them.

### Out of memory

If the cause of failure is the following:

```text
Error while running the command (please refer to the job log for more information). Warning: Out of memory error occurred during this job.
```

You need to choose an instance with a bigger memory. The memory infix in the instance is `mem`. For more information on instances, refer to the [Instances section](#instances).

### Low disk space

If the cause of failure is the following:

```text
Error while running the command (please refer to the job log for more information). Warning: Low disk space during this job.
```

You need to choose an instance with a bigger storage. The storage infix in the instance is `ssd` or `hdd`. For more information on instances, refer to the [Instances section](#instances).

### Interruption limit (only for *low* priority)

Jobs with a low priority can be interrupted if their ressources are needed by high priority jobs. A low priority job can be interrupted at most 10 times, then it will result in the following error:

```text
The machine running the job was terminated by the cloud provider
```

You simply need to restart your job, either on the same instance or another one, if the one used is too popular.

### Others

If the error is unclear to you, check the log of your job. You will most likely get an answer there.
