# First setup

In order to connect to DNAnexus remotely, you first need to install the `dxpy` package.

```bash
pip3 install dxpy
```

To enable tab completion, run the following command, or add it to your `.bashrc`:

```bash
eval "$(register-python-argcomplete dx|sed 's/-o default//')"
```

> You can find the official documentation for all `dx` commands we will use [here](https://documentation.dnanexus.com/user/helpstrings-of-sdk-command-line-utilities).

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

If you have access to multiple project on DNAnexus, you need to choose the specific one in which you want to perform your GWAS.
Please use the `dx select` command or check the official documentation on [Project Navigation](https://documentation.dnanexus.com/user/projects/project-navigation#changing-your-current-project) for more information.

If you have access to only one project, it will be already selected and you can go on with this tutorial.

By default, any job prompted here will output in your current DNAnexus repertory, meaning at the root of your project.
To keep your project tidy, we can create a new folder and move into it, thus all jobs will output there.


 Therefore, you can create a new folder and move into it, to help keep your project tidy.

```bash
dx mkdir gwas_tutorial
dx cd gwas_tutorial
```

> If multiple people have access and frequently run jobs on the project chosen, we recommend having your own directory like `WKD_<your-name>` in which you will then create the `gwas_tutorial` directory.
