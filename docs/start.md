# Getting started

## Login

In order to connect to DNAnexus remotely, you first need to install the `dxpy` package.

```bash
pip3 install dxpy
```

To enable tab completion, run the following command, or add it to your `.bashrc`:

```bash
eval "$(register-python-argcomplete dx|sed 's/-o default//')"
```

You can now enter your DNAnexus credentials to access your project remotely by using the following command:

```bash
dx login
```

Your authentication token and your current project settings have now been saved in a local configuration file, and you're ready to start accessing your project.

<blockquote>

By default, your information expires in 30 days, but this can be changed using the <code>--timeout</code> option.

For instance, if you want your info to expire in <b>6 months</b>, use the following command. The <code>-help</code> option is useful if you want to know more about the <code>--timeout</code> input format.

```bash
dx login --timeout 6M
```

</blockquote>

If you have access to several projects on DNAnexus, you need to choose the specific one in which you want to perform your GWAS.
Please use the `dx select` command or check the official documentation on [Project Navigation](https://documentation.dnanexus.com/user/projects/project-navigation#changing-your-current-project) for more information.

If you have access to only one project, it will already be selected and you can go on with this tutorial.

## Architecture

When using the command line, you have access to the architecture on DNAnexus and your own local architecture.
The basic commands like `ls`, `cd` or `mv` all have equivalents to run on the DNAnexus architecture rather than the local one, simply add `dx` before them.

A quick way to check the difference is to run the same command on both architecture:

* `ls` will lists the files in your **current directory**, as you probably already know
* `dx ls` will lists the files in your **DNAnexus current directory**, which should be the root of your project if you are first starting (with the `Bulk` and `Showcase metadata` folders)

> You can find the official documentation for all `dx` commands we will use [here](https://documentation.dnanexus.com/user/helpstrings-of-sdk-command-line-utilities).

By default, any job run from the command line will output in your current DNAnexus repertory, meaning at the root of your project.
To keep your project tidy, we can create a new folder and move into it, so that all jobs will output there.

Therefore, you can create a new folder and move into it, to help keep your project tidy.

```bash
dx mkdir gwas_tutorial
dx cd gwas_tutorial
```

> If multiple people have access and frequently run jobs on the project chosen, we recommend having your own directory like `WKD_<your-name>` in which you will then create the `gwas_tutorial` directory.

## First job

If this is your first time using DNAnexus altogether, please go to the following page titled [About jobs](jobs.md). It will take you through the fondamentals of jobs on DNAnexus, and will help you understand how and why the code works.

If you are already familiar with how DNAnexus jobs work, please skip ahead to the [Input files page](input.md), which will start the GWAS tutorial.
