# Input files

This section is done on [**DXJupyterLab**](https://documentation.dnanexus.com/user/jupyter-notebooks), launched via the *TOOLS* tab on DNAnexus's web page, with the following configuration:

* Cluster configuration: **Single Node**
* Instance type: **mem1_ssd1_v2_x4**
* Duration (in hours): **1**

The estimated price is around **£0.1136** (corresponding to the rates as of June 2025, they may have changed). However, since we don't need the full hour to run the code (only a few minutes) the final cost will be reduced.

> Please be aware that storing files onto DNAnexus will result in a monthly cost. You may check its current value in the *SETTINGS* tab on your project's web page, or compute it using the [UKB RAP Rate Card](https://20779781.fs1.hubspotusercontent-na1.net/hubfs/20779781/Product%20Team%20Folder/Rate%20Cards/BiobankResearchAnalysisPlatform_Rate%20Card_Current.pdf).

## Genetic data

On DNAnexus, large items, like genetic data, are stored in the `Bulk` folder at the root of your project. Among them, you can find whole exome sequencing data (`/Bulk/Exome sequences/`) or whole genome sequencing data (`/Bulk/DRAGEN WGS/` or `/Bulk/Previous WGS releases/`).

Whole genome sequencing data is available in multiple formats: BGEN format (`.bgen` and `.sample`), PLINK format (`.bed`, `.bim` and `.fam`), PLINK2 format (`.pgen`, `.psam` and `.pvar`) or pVCF format (`.vcf.gz.tbi`). Feel free to browse the `Bulk` folder for more information on the data available.

As written on the [About jobs page](jobs.md), input files can be either downloaded or mounted to the worker.
We have tested both to see which one was quicker.

### PLINK2

After some trial and error, we have found that downloading the input in the PLINK2 format was the quickest way to perform a GWAS with PLINK2.
This makes sense seeing as the PLINK2 format is innate to PLINK2 and the most compressed of all available formats, but we were surprised to found that mounting was slower.

The path to the genetic data chosen is the following: `/Bulk/DRAGEN WGS/DRAGEN population level WGS variants, PLINK format [500k release]/`.

> Please be aware, when working on previous releases, we had found that using the BGEN format without mounting it was quicker than downloading or mounting the PLINK files.

### Regenie

We first wanted to use the DRAGEN 500k data for Regenie as well, but that has proven to be quite difficult for a number of reasons.
Namely, as June 2025, the DRAGEN 500k `.psam` file **does not** have a header, which results in `ERROR: header does not have the correct format.` when using Regenie ([Issue about this error](https://github.com/rgcgithub/regenie/issues/105)). Therefore, we cannot use the PLINK2 data.

Regenie's Step 2 is optimized for input genotype data in BGEN *v1.2* format.
However, the BGEN data for DRAGEN 500k is **very** large: the file for chromosome 1 is bigger than a tebibyte (1.01 TiB).
Thus, higher instances are needed, costs add up and execution time is increased (since PLINK2 is used and has to translate the format).

Therefore, we choose to use previous releases, which are way less heavy (chromosome 1 is 71.49 GiB).

Regarding the mount or download of the data, there seems to be an error when mounting data to Regenie ([Corresponding issue](https://github.com/rgcgithub/regenie/issues/365)).
Because of this, we confirm the download the input, which should take less time and will garantee complete results.

The path to the genetic data chosen is the following: `/Bulk/Previous WGS releases/GATK and GraphTyper WGS/GraphTyper population level genome variants, BGEN format [200k release]/`.

> Please be aware, we have not tested all possibilities, and specific formats might work better with specific data.
> Therefore, if you choose to use any other data, you might want to check which is better.

## Additional data

In order to run our GWAS, apart from the genetic and sample data, we need 3 additionnal files:

* The phenotype (for instance, BMI)
* The ids of individuals we wish to keep (for instance, white british)
* The covariates to use (for instance, 16 genetic principal components, the sex and the age)

This tutorial will guide you through the local creation of these files using DNAnexus data, and their upload to your project using `dx upload`.

### Phenotype

To extract the phenotype that we want, we first need to download all available data-fields in our dataset.
The files downloaded will have an incredibly long name, we want to rename it to a more manageable name like `ukbb`.

```python
"""Extract dataset information."""
import os
import subprocess

import dxpy

# Extraction
dispensed_dataset_id = dxpy.find_one_data_object(typename='Dataset', name='app*.dataset', folder='/', name_mode='glob')['id']
project_id = dxpy.find_one_project()["id"]
dataset = (':').join([project_id, dispensed_dataset_id])

cmd = ["dx", "extract_dataset", dataset, "-ddd", "--delimiter", ","]
subprocess.check_call(cmd)

# Renaming the files
for filename in os.listdir("."):
    if filename.startswith("app"):
        separated = filename.split('.')
        newname = f"ukbb.{'.'.join(separated[2:])}"
        os.rename(filename, newname)
```

This command outputs 3 files:

* `ukbb.data_dictionary.csv` (6.7 Mo) contains a table with participants as the rows and metadata along the columns, including field names (see table below for naming convention)
* `ukbb.codings.csv` (6.2 Mo) contains a lookup table for the different medical codes, including ICD-10 codes that will be displayed in the diagnosis field column (p41270)
* `ukbb.entity_dictionary.csv` (1.9 Ko) contains the different entities that can be queried, where participant is the main entity that corresponds to most phenotype fields

> Please note, when running the `extract_dataset` command you might encounter a `('Invalid JSON received from server', 200)` error. If this happens, you simply need to rerun the code.

We first need to get the field name of the phenotype(s) we want to extract. As an example, we will extract the BMI index ([21001](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21001)), but you can extract any number of phenotypes.

For the main participant phenotype entity, the Research Analysis Platform (UKB-RAP) uses field names with the following convention:

| Type of field                 | Syntax for field name           | Example     |
|-------------------------------|---------------------------------|-------------|
| Neither instanced nor arrayed | `p<FIELD>`                      | `p31`       |
| Instanced but not arrayed     | `p<FIELD>_i<INSTANCE>`          | `p40005_i0` |
| Arrayed but not instanced     | `p<FIELD>_a<ARRAY>`             | `p41262_a0` |
| Instanced and arrayed         | `p<FIELD>_i<INSTANCE>_a<ARRAY>` | `p93_i0_a0` |

This means one phenotype ID can actually have multiple data field. For example, BMI has four instances.
The following python script will first extract every array or instance associated to your phenotype (`field_names_for_ids()`), then keep only the first one for each (`select_first()`). We choose to keep only the first instance since multiple columns will be analyzed as separate phenotypes.

> Please note, the file computed is temporary, it is not necessary to rename it since it's not the final phenotype file.
> Moreover, since `extract_dataset` has no *overwrite* option by design, we implemented it ourselves.
> Running the following code will first delete `pheno_extract.csv` if it's present, allowing for the extraction to happen.

```python
"""Extract phenotype(s) based on field ID(s)."""

import os
import re
import subprocess

import dxpy
import pandas as pd

def field_names_for_ids(filename, field_ids):
    """ Convert data-field id to corresponding field name.

    Parameters
    ----------
    filename : str
        Path to the '.dataset.data_dictionary.csv' file.
    field_ids : list
        All field ids.
    data.frame (pandas)
        Projects's .data_dictionary.csv file.

    Returns
    -------
    list
        All corresponding field names.
    """
    data = pd.read_csv(filename, sep=',')
    field_names = ["eid"]
    for _id in field_ids:
        select = list(data[data.name.str.match(r'^p{}(_i\d+)?(_a\d+)?$'.format(_id))].name.values)
        field_names += select

    field_names = [f"participant.{f}" for f in field_names]
    return field_names

def select_first(field_names):
    """Select only first data for a field.

    Parameters
    ----------
    field_names : list
        All corresponding field names.

    Returns
    -------
    list
        All corresponding field names.
    """
    selection = []
    for field in field_names:
        # Not instanced nor arrayed
        if not re.search("_", field):
            selection.append(field)
        else:
            f_inst = re.search("_i", field)
            f_arr = re.search("_a", field)
            f_inst_0 = re.search("i0", field)
            f_arr_0 = re.search("a0", field)

            # Instanced and arrayed
            if f_inst_0 and f_arr_0:
                selection.append(field)
                continue
            # Instanced not arrayed
            if f_inst_0 and not f_arr:
                selection.append(field)
            # Arrayed not instanced
            if f_arr_0 and not f_inst:
                selection.append(field)
    return selection

# Input
DATASET = dxpy.find_one_data_object(typename='Dataset',
                                    name='app*.dataset',
                                    folder='/',
                                    name_mode='glob')['id']
FILENAME = "ukbb.data_dictionary.csv"
OUTPUT = "pheno_extract.csv"
FIELD_ID = [21001] # BMI id

# Convert id to names
FIELD_NAMES = field_names_for_ids(FILENAME, FIELD_ID)
FIELD_NAMES = select_first(FIELD_NAMES)
FIELD_NAMES = ",".join(FIELD_NAMES)

# Extract phenotype(s)
if os.path.exists(OUTPUT):
    os.remove(OUTPUT)
cmd = ["dx", "extract_dataset", DATASET, "--fields", FIELD_NAMES,
        "--delimiter", ",", "--output", OUTPUT]
subprocess.check_call(cmd)
```

This command outputs 1 file:

* `pheno_extract.csv` (8.0 Mo) contains the values for participant IDs and the first instance of BMI values.

<blockquote>

Please be aware, extracting a huge quantity of phenotypes may consistently run into a <code>('Invalid JSON received from server', 200)</code> error.
If so, we suggest extracting multiple files, and combining them into one. You can use the following command to do so:

```python
%%bash
paste -d',' file1.csv file2.csv > merged.csv
```

</blockquote>

PLINK2 and Regenie use the **same formatting** for the phenotype file, thus we can use the same file for both.
We first need to duplicate the individuals ID, mark missing values as 'NA' (which is recognized by both Regenie and PLINK2 with the necessary options) and name our phenotype.

More information on phenotype files formatting can be found [here](https://www.cog-genomics.org/plink/2.0/input#pheno) for PLINK2 and [here](https://rgcgithub.github.io/regenie/options/#phenotype-file-format) for Regenie.

```python
"""Format phenotype file."""

import pandas as pd

# Input
FILENAME = "pheno_extract.csv"
PHENOTYPE = "BMI"

def format_phenotype(filename, phenotype):
    """ Save phenotype to correct format.

    Parameters
    ----------
    filename : str
        Phenotype file to format.
    phenotype : str
        Phenotype name.
    """
    output = f"{phenotype}.txt"

    # Format file
    data = pd.read_csv(filename, sep=',', skiprows=1, names=["IID", phenotype])
    fid = data["IID"].rename("FID")
    merged = pd.concat([fid, data], axis=1)

    merged.to_csv(output, sep='\t', index=False, header=True, na_rep='NA')

# Save phenotype
format_phenotype(FILENAME, PHENOTYPE)
```

This command outputs 1 file:

* `BMI.txt` (11.42 MiB) contains the formatted phenotype values

You can now upload the formated phenotype file.

```bash
%%bash 
dx upload BMI.txt -p --path /gwas_tutorial/
```

### Individual ids

Running a GWAS on a specific population helps reduce the bias caused by population stratification. It is therefore an important step.

To filter out individuals based on their ethnic background, we can use the [*phenotype extraction script*](#phenotype) and the data-field [21000](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21000). You simply need to **change the value** of `FIELD_ID`:

```python
# Input
FILENAME = "ukbb.data_dictionary.csv"
FIELD_ID = [21000] # Ethnic background id

# Convert id to names
FIELD_NAMES = field_names_for_ids(FILENAME, FIELD_ID)
FIELD_NAMES = select_first(FIELD_NAMES)
FIELD_NAMES = ",".join(FIELD_NAMES)

# Extract population(s)
if os.path.exists(OUTPUT):
    os.remove(OUTPUT)
cmd = ["dx", "extract_dataset", DATASET, "--fields", FIELD_NAMES,
        "--delimiter", ",", "--output", OUTPUT]
subprocess.check_call(cmd)
```

This field uses the [1001 data encoding](https://biobank.ndph.ox.ac.uk/ukb/coding.cgi?id=1001). In this code, *1001* means *white british* which is the main ethnic group, and the one we want to select.

```bash
%%bash
pop_code=1001
pop_name="white_british"
population="pheno_extract.csv"

awk -F "," -v var="$pop_code" '$2~var{print $1,$1}' $population > $pop_name.txt
```

This command outputs 1 file:

* `white_british.txt` (6.75 MiB) contains the participants IDs of the white british ethnic background

You can now upload the ids of your individuals.

```bash
%%bash 
dx upload white_british.txt -p --path /gwas_tutorial/
```

### Covariates

The genetic principal components from the UKBB individuals are stored in the [22009](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=22009) data field.
We could use our [phenotype extraction script](#phenotype), but it appears to constantly end in a [`BadJSONInReply` error](http://autodoc.dnanexus.com/bindings/python/current/_modules/dxpy/exceptions.html). We have found a [community post by Alex Shemy](https://community.dnanexus.com/s/question/0D582000000PW3bCAG/endtoendgwasphewasgwasphenotypesamplesqcipynb) which seems to encounter the same problem as us. However, it highlights that not everyone encounters this error, and our script might work on your machine.

In any case, we provide the following script which will extract exactly what we want and bypass the error.
We will use 18 variables as covariates: the first 16 PCA components, the sex (data field [31](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=31)) and the age (data field [21003](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21003)) of individuals.

You can extract whatever you want as covariates. However, we have chosen to keep only the first 16 PCA components, following a [study by Privé *et al.* in 2020](https://academic.oup.com/bioinformatics/article/36/16/4449/5838185) which indicates other components capture complex LD structure rather than population structure.

More information on covariates files formatting can be found [here](https://www.cog-genomics.org/plink/2.0/input#covar) for PLINK2 and [here](https://rgcgithub.github.io/regenie/options/#covariate-file-format) for Regenie.

```bash
%%bash
dataset=$(dx ls /app*.dataset --brief)
field_names="participant.eid,participant.eid,"

for i in {1..16}
do
  field_names+="participant.p22009_a$i,"
done

field_names+="participant.p31,participant.p21003_i0" # Sex and age

dx extract_dataset $dataset --fields $field_names --delimiter "," --output covariates.txt

echo -e "FID,IID,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10,PC11,PC12,PC13,PC14,PC15,PC16,Sex,Age" > file.tmp
tail -n+2 covariates.txt >> file.tmp
awk -F , '$3!=""' file.tmp > covariates.txt # Remove ind with no PC data
sed 's/,/\t/g' covariates.txt > file.tmp
mv file.tmp covariates.txt
```

This command outputs 1 file:

* `covariates.txt` (74.59 MiB) contains the first 16 PCA components, the sex and the age of all participants

You can now upload the covariates file.

```bash
%%bash 
dx upload covariates.txt -p --path /gwas_tutorial/
```

> Once all of theses steps are complete, make sure all 3 files are correctly uploaded and end the session to cut costs.
> You can do so by selecting *End Session* from the *DNAnexus* tab of the notebook.
