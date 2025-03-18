# About jobs

This page aims to go through the basics of jobs on DNAnexus. You do not need to read it to complete the tutorial, but it can help if this is your first time using DNAnexus, or if you are unsure about how some things work.

## Execution

On DNAnexus, when a job is executed, a worker is spun up in the cloud, then the job's code and inputs are downloaded to that worker and executed. This is the standard behavior, but inputs can also be mounted to the worker (accessed dynamically) to avoid their downloading costs.

### Inputs

Since jobs are run on workers, you need to manually specify every file used as input.

To add a file as input, you need to use the `-iin` option to your `dx run` command. The path to your file needs to be absolute so that there is no confusion as to where it is located.

When mounting your files, you don't need to input them manually anymore. You simply need to add `/mnt/project/` before your absolute path.

### Instances

To choose an instance for your job is to choose a specific worker on which your code will be executed.
Instances have 3 main metrics:

* The number of cores (individual processing units)
* The memory capacity (components used for short-term data retention)
* The storage capacity (components used for long-term data retention)

You might have no clue on what ressources your job need to run properly, the simplest course of action is to start with a small instance and increase its capacities if you encounter any errors. Please check the [Errors when running section](#errors-when-running) if you don't know how to interpret your error message.

> Please note, this tutorial uses the same instance for every job. Depending on the data you want to use, you might need a bigger instance. If your goal is full optimization, then you might prefer a smaller one. All of this is up to you.

To specify the instance you chose, you need to add the `--instance-type` option to your `dx run` command.

For more information on instances, check the [official doc on instance types](https://documentation.dnanexus.com/developer/api/running-analyses/instance-types).

### Priority

Jobs can have three types of priority:

* `high` means your job will be completed once the ressources are available
* `low` means your job will start once the ressources are available, but may be interrupted if `high` priority jobs need them
* `normal` means your job will wait for 15min to run in a `low` priority, and if no ressources are availables by then, will run in a `high` priority

Although a `high` priority assures for job completion, it is also pricier than a `low` priority one.

> Please note, this tutorial will only run `low` priority jobs to reduce costs.

To specify the instance you chose, you need to add the `--priority` option to your `dx run` command.

For more information on priorities, check the [official doc on job priority](https://dnanexus.gitbook.io/uk-biobank-rap/working-on-the-research-analysis-platform/managing-jobs/managing-job-priority).

### Tags

Tags, although optional, are very pratical when using DNAnexus regularly.
They act as keywords associated to a job, and help if you need to navigate the *MONITOR* tab. You can find more information in the [Monitoring section](#monitoring).

To add a tag to your job, you need to add the `--tag` option to your `dx run` command.

## Monitoring

You can monitor your jobs directly on your project's web page in the *MONITOR* tab.
On this page, they are sorted in ascending order of departure date (the latest comes first).

Basic information about the job is displayed, like its name, duration, cost, *etc*...

A job has 3 possible states:

* `Done` indicates that the job has completed successfully
* `Waiting` indicates that the ressources needed to run this job are not available yet, it will start once they are
* `Failed` indicates the job has failed due to an error

Once a job is completed (either `Done` or `Failed`), you will receive an email updating you of said completion.
It will also appear in the `Notifications` tab (bell icon in the top right corner).

### Filtering jobs

After running a lot of jobs, you might find your *MONITOR* page to be quite crowded. This is especially true with this tutorial where some jobs are run for each of the 22 chromosomes. To find a specific job, you can filter out the *MONITOR* page.

In this tutorial, all jobs have specific [tags](#tags) which help with filtering.
By default, tags are not used in filtering on the *MONITOR* page, you need to add them in the `Filter settings` tab (three bars in the top right corner).

> Please note, in this tutorial, jobs are tagged with the software used (plink or regenie), the phenotype (BMI), the step of the analysis (QC, GWAS, Step 0 to 2) and lastly the chromosome number (c1 to 22) if jobs are separated per chromosome.

## Errors when running

A failed job running on DNAnexus has only 3 errors (apart from errors in the code):

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

Jobs with a low priority can be interrupted if its ressources can be used for a high priority jobs. A low priority job can be interrupted at most 10 times, then it will result in the following error:

```text
The machine running the job was terminated by the cloud provider
```

You simply need to restart your job, either on the same instance or another one, if the one used is too popular.

### Others

If the error is unclear to you, check the log of your job. You will most likely get an answer there.