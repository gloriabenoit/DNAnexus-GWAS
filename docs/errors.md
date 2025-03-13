# Errors when running jobs

A job running on DNAnexus has 4 outcomes:

* Done: The job was successful.
* Failed: Out of Memory
* Failed: Low disk space
* Failed: Terminated by the cloud provider (only for low priority jobs)

We will take you through all 3 failed outcomes, and what you can do to avoid them.

## Out of memory

If the cause of failure is the following:

```text
Error while running the command (please refer to the job log for more information). Warning: Out of memory error occurred during this job.
```

You need to choose an instance with a bigger memory. The memory infix in the instance is `mem`. For more information on instances, check the [official doc on instance types](https://documentation.dnanexus.com/developer/api/running-analyses/instance-types).

## Low disk space

If the cause of failure is the following:

```text
Error while running the command (please refer to the job log for more information). Warning: Low disk space during this job.
```

You need to choose an instance with a bigger storage. The storage infix in the instance is `ssd` or `hdd`. For more information on instances, check the [official doc on instance types](https://documentation.dnanexus.com/developer/api/running-analyses/instance-types).

## Interruption limit

Jobs with a low priority can be interrupted if its ressources can be used for a high priority jobs. A low priority job can be interrupted at most 10 times, then it will result in the following error:

```text
The machine running the job was terminated by the cloud provider
```

You simply need to restart your job, either on the same instance or another one, if the one used is too popular.
