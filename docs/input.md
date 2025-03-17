# Input files

## Genetic data

On DNAnexus, whole genome sequences data are saved in multiple formats: BGEN format (`.bgen` and `.sample`), PLINK format (`.bed`, `.bim` and `.fam`) or pVCF format (`.vcf.gz.tbi`), among others. For more information on the data available, you can run the following command:

```bash
dx ls "/Bulk/Whole genome sequences"
```

On DNAnexus, when a job is executed, a worker is spun up in the cloud, then the job's code and inputs are downloaded to that worker and exectuted. This is the standard behavior, but inputs can also be mounted to avoid their downloading costs.

### PLINK2

PLINK2 can read both BGEN and PLINK format. Therefore, we have 4 ways of inputing genetic data:

<center>

| File format | Mounted |
|-------------|---------|
| BGEN        | No      |
| BGEN        | Yes     |
| PLINK       | No      |
| PLINK       | Yes     |

</center>

After some trial and error, we have found that using the BGEN format without mounting it was the quickest way to perform a GWAS using PLINK2. Downloading and translating BGEN files locally was quicker than downloading or mounting the PLINK files.

Therefore, in this tutorial, we will input BGEN files directly. The path to the genetic data is the following: `"/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release/"`.

## Additionnal data

In order to run our GWAS, apart from the genetic and sample data, we need 3 additionnal files:

* The phenotype (for instance, BMI)
* The ids of individuals we wish to keep (for instance, white british)
* The covariates to use (for instance, 18 genetic principal components, the sex and the age)

Theses files can either be uploaded directly to your projects using `dx upload`, or created using DNAnexus data. This tutorial will guide you through the second option.

> All three files will be uploaded to your current `dx` repertory (`WKD_<your-name>` if you are closely following this tutorial).
> If you want to upload them somewhere else, you can either move directory with `dx cd` before uploading, or use the `--path` or `--destination` option to specify the DNAnexus path to upload the files.

### Phenotype

To extract the phenotype that we want, we first need to download all available data-fields in our dataset.
You will need the `record-id`, which is the id of the `.dataset` file at the root of your DNAnexus project. On your project's web page, you can find the id by selecting the `.dataset` file and searching for the `ID` in the info panel that appears on the right.

> Please note, when running the `extract_dataset` command you might encounter a `('Invalid JSON received from server', 200)` error. If this happens, you simply need to rerun the code.

```bash
dx extract_dataset <record-id> -ddd --delimiter ","
```

This command will output 3 files:

* `<app-id>.data_dictionary.csv` contains a table with participants as the rows and metadata along the columns, including field names (see table below for naming convention)
* `<app-id>.codings.csv` contains a lookup table for the different medical codes, including ICD-10 codes that will be displayed in the diagnosis field column (p41270)
* `<app-id>.entity_dictionary.csv` contains the different entities that can be queried, where participant is the main entity that corresponds to most phenotype fields

To make it easier, you can rename theses files to a shorter name like `ukbb`.

```bash
rename <app-id> ukbb app*
```

We first need to get the field name of the phenotype(s) we want to extract. As an example, we will extract the BMI index ([21001](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21001)), but you can extract any number of phenotypes.

For the main participant phenotype entity, the Research Analysis Platform (UKB-RAP) uses field names with the following convention:

<center>

| Type of field                 | Syntax for field name           | Example     |
|-------------------------------|---------------------------------|-------------|
| Neither instanced nor arrayed | `p<FIELD>`                      | `p31`       |
| Instanced but not arrayed     | `p<FIELD>_i<INSTANCE>`          | `p40005_i0` |
| Arrayed but not instanced     | `p<FIELD>_a<ARRAY>`             | `p41262_a0` |
| Instanced and arrayed         | `p<FIELD>_i<INSTANCE>_a<ARRAY>` | `p93_i0_a0` |

</center>

This means one phenotype ID can actually have multiple data field. For example, BMI has four instances.
The following python script will extract the phenotype that you wish, and every array or instance associated.
It will ouptput `pheno_extract.csv` which contains individual ids and any phenotype you chose to extract.

```python
""" Extract phenotype(s) from UKBB based on field ID(s). """

import os
import subprocess
import pandas as pd

# Input
FILENAME = "ukbb.dataset.data_dictionary.csv"
OUTPUT = "pheno_extract.csv"
DATASET = "<record-id>"
FIELD_ID = [21001] # BMI id

def field_names_for_ids(filename, field_ids):
    """ Converts data-field id to corresponding field name.

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

# Convert id to names
FIELD_NAMES = field_names_for_ids(FILENAME, FIELD_ID)
FIELD_NAMES = ",".join(FIELD_NAMES)

# Extract phenotype(s)
if os.path.exists(OUTPUT):
  os.remove(OUTPUT)
cmd = ["dx", "extract_dataset", DATASET, "--fields", FIELD_NAMES,
        "--delimiter", ",", "--output", OUTPUT]
subprocess.check_call(cmd)
```

> Please be aware, since `extract_dataset` has no *overwite* option by design, we implemented ourselves.
> Running the previous code will first delete `pheno_extract.csv` if it's present, allowing for the extraction to happen.

PLINK and regenie use the same formatting for the phenotype file, with a single exception : the code for missing values.
Apart from this, both need to duplicate the individuals ids and only keep the first instance for our phenotype.

More information on phenotype files formatting can be found [here](https://www.cog-genomics.org/plink/2.0/input#pheno) for PLINK2 and [here](https://rgcgithub.github.io/regenie/options/#phenotype-file-format) for regenie.

```python
""" Format phenotype file. """

import pandas as pd

# Input
FILENAME = "pheno_extract.csv"
PHENOTYPE = "BMI"
SOFTWARE = 'p' # for PLINK2 or 'r' for regenie.

def format_phenotype(filename, phenotype, software):
    """ Save phenotype according to software format.

    Parameters
    ----------
    filename : str
        Phenotype file to format.
    phenotype : str
        Phenotype name.
    format : str { 'p', 'r'}
        Either 'p' for PLINK2 or 'r' for regenie,
        to correctly format the file.
    """
    # Software specific format.
    if software == 'p':
        na_val = -9
        output = f"plink_{phenotype}.txt"
    elif software == 'r':
        na_val = 'NA'
        output = f"regenie_{phenotype}.txt"

    # Data
    data = pd.read_csv(filename, sep=',')

    # To be changed accordingly
    pheno = data.iloc[:,[0,1]]
    pheno = pheno.rename(columns={pheno.columns[0]: "IID", pheno.columns[1]: phenotype})
    data = data.rename(columns={data.columns[0]: "FID"})
    merged = pd.concat([data.iloc[:, 0], pheno], axis=1)

    # Save output
    merged.to_csv(output, sep='\t', index=False, header=True, na_rep=na_val)

# Save phenotype
format_phenotype(FILENAME, PHENOTYPE, SOFTWARE)
```

> You can modify the `format_phenotype` function to your heart's desire, depending on what you wish to do with the phenotypes.
> This is only a suggestion, and might not work for more specific phenotypes.

You can now upload the formated phenotype file.

```bash
dx upload plink_BMI.txt
dx upload regenie_BMI.txt
```

> For the sake of this tutorial, we upload two files: one for each type of formatting.

### Individual ids

Running a GWAS on a specific population helps reduce the bias caused by population stratification. It is therefore an important step.

To filter out individuals based on their ethnic background, we can use the [*phenotype extraction script*](#phenotype) and the data-field [21000](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21000). You simply need to replace the following line:

```python
FIELD_ID = [21000] # ethnic background id
```

This field uses the [1001 data encoding](https://biobank.ndph.ox.ac.uk/ukb/coding.cgi?id=1001). In this code, *1001* means *white british* which is the main ethnic group, and the one we want to select.

```bash
pop_code=1001
pop_name="white_british"
population="pheno_extract.csv"

awk -F "," -v var="$pop_code" '$2~var{print $1,$1}' $population > $pop_name.txt
```

This code will output the file `white_british.txt`, containing the ids of the individuals of the white british ethnic background.

You can now upload the ids of your individuals.

```bash
dx upload white_british.txt
```

### Covariates

The genetic principal components from the UKBB individuals are stored in the [22009](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=22009) data field.
We could use our phenotype extraction algorithm, but for some reason, it is not working. Therefore, we will use another script.

We will use 20 variables as covariates: the first 18 PCA components, the sex ([31](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=31)) and the age ([21003](https://biobank.ndph.ox.ac.uk/ukb/field.cgi?id=21003)) of individuals. However, you can extract whatever you want as covariates.

More information on covariates files formatting can be found [here](https://www.cog-genomics.org/plink/2.0/input#covar) for PLINK2 and [here](https://rgcgithub.github.io/regenie/options/#covariate-file-format) for regenie.

```bash
record_id="<record-id>"
field_names="participant.eid,participant.eid,"

for i in {1..18}
do
  field_names+="participant.p22009_a$i,"
done

field_names+="participant.p31,participant.p21003_i0" # Sex and age

dx extract_dataset $record_id --fields $field_names --delimiter "," --output covariates.txt

echo -e "FID,IID,PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10,PC11,PC12,PC13,PC14,PC15,PC16,PC17,PC18,Sex,Age" > file.tmp
tail -n+2 covariates.txt >> file.tmp
awk -F , '$3!=""' file.tmp > covariates.txt # Remove ind with no PC data
sed 's/,/\t/g' covariates.txt > file.tmp
mv file.tmp covariates.txt
```

You can now upload the covariates file.

```bash
dx upload covariates.txt
```
