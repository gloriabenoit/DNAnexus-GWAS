# Initial setup

In order to connect to DNAnexus remotely, you first need to install the `dxpy` package.

```bash
pip3 install dxpy
```

To enable tab completion, enter the following command, or add it to your `.bashrc`:

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

For instance, if you want your info to expire in <b>6 months</b>, use the following command. The <code>-help</code> option is useful if you want to know more about the `--timeout` input format.

```bash
dx login --timeout 6M
```

</blockquote>

By default, any job prompted here will output in your current DNAnexus repertory. Therefore, you can create a new folder and move into it, to help keep your poject tidy (especially if multiple people use it).

```bash
dx mkdir WKD_<your-name>
dx cd WKD_<your-name>
```

Now, all job will output in this repertory.

You can find the documentation for all `dx` commands we will use [here](https://documentation.dnanexus.com/user/helpstrings-of-sdk-command-line-utilities).
