# Using PLINK2

Every file created during this analysis will be stored in a main directory called `plink_gwas_BMI`.
This section will run 22 different jobs (one per chromosome).

With our chosen instance (mem1_ssd1_v2_x16) using a `high` priority, the whole GWAS will take about **40 minutes** and cost around **£3.84** (for a total execution time of 528 minutes).

With the same instance (mem1_ssd1_v2_x16) using a `low` priority, if no jobs are interrupted, it will cost only **£1.03** for the same time.

> Please note that it is most unlikely for jobs to be uninterrupted.
> In our experience, when accounting for interrupted/failed jobs, the total cost is around one to two times higher than the *low* cost, but still lower than the *high* cost. Although please keep in mind that the total time needed can be two to five times higher, depending on the interrupted/failed jobs.

## Input files

The path to the genetic data chosen is the following: `/Bulk/DRAGEN WGS/DRAGEN population level WGS variants, PLINK format [500k release]/`.

Before running a GWAS on DNAnexus using PLINK2, you need to make sure you have these 3 files uploaded to DNAnexus:

* The phenotype: `BMI.txt`
* The ids of individuals we wish to keep: `white_british.txt`
* The covariates to use: `covariates.txt`

You can check their presence with the following command:

```bash
dx ls
```

Please refer to the [Input files section](input.md) if you don't have these files.

## Running a GWAS

On DNAnexus, PLINK2 is available either as part of the [Swiss Army Knife app](https://ukbiobank.dnanexus.com/app/swiss-army-knife) (`swiss-army-knife`) or as its [own app](https://ukbiobank.dnanexus.com/app/plink_gwas) called `plink_gwas`. We will use the former.

We choose to use the same instance for all GWASs, to simplify the code, but this can be changed to your liking. Same for the priority and the cost limit.

### Quality control

We perform the QC at the same time as our GWAS. The variants are filtered using the following options:

```text
--maf 0.01 --hwe 1e-50 --geno 0.1
```

> Please change the thresholds according to your preferences, but be aware any change will modify jobs execution time, cost and size.

Please note that we choose not to use the `--mind` filter, although it is a conventional, as it removes 88% of the samples for chromosome 18.
However, we have not checked for this behavior in other data format so it might be specific to the PLINK2 format.

### Linear regression

Following a [study by Rivas and Chang in 2025](https://academic.oup.com/bioinformatics/article/41/3/btaf067/8008994), we will add the `qt-residualize` modifier to the `--glm` option, which performs a single regression on the covariates upfront, and then passes the phenotype residuals.
In our experience, this divides the execution time by 3 while yielding the same results as `--glm` alone, which is why we choose to use it.
It also has the added bonus that it scales greatly when analyzing multiple phenotypes (keeping the same execution time for more than a hundred phenotypes).

> Please be aware that this option may have limits. We recommend making sure that the results are expected.

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/$pheno.txt"
ind_path="/gwas_tutorial/white_british.txt"
ind=$(basename "$ind_path")
cov_path="/gwas_tutorial/covariates.txt"
cov=$(basename "$cov_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p plink_gwas_$pheno
dx cd plink_gwas_$pheno

for chr_num in $(seq 1 22); do
    prefix="/Bulk/DRAGEN\ WGS/DRAGEN\ population\ level\ WGS\ variants,\ PLINK\ format\ [500k\ release]//ukb24308_c${chr_num}_b0_v1"
    pfile=$(basename "$prefix")

    plink_command="plink2 \
                --threads $threads \
                --maf 0.01 \
                --hwe 1e-50 \
                --geno 0.1 \
                --glm 'qt-residualize' 'hide-covar' \
                --keep $ind \
                --covar $cov \
                --covar-name PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10,PC11,PC12,PC13,PC14,PC15,PC16,Age,Sex \
                --pheno $pheno.txt \
                --pfile $pfile \
                --no-psam-pheno \
                --no-input-missing-phenotype \
                --out sumstat_c${chr_num}"

    dx run swiss-army-knife \
        --priority "$priority" --cost-limit "$cost_limit" \
        -icmd="$plink_command" \
        --instance-type "$instance" \
        --name="plink_gwas_${pheno}_c${chr_num}" \
        --tag="plink" \
        --tag="GWAS" \
        --tag="$pheno" \
        --tag="c${chr_num}" \
        -iin="$ind_path" \
        -iin="$cov_path" \
        -iin="$pheno_path" \
        -iin="$prefix.pgen" \
        -iin="$prefix.psam" \
        -iin="$prefix.pvar" \
        -y
done

dx cd ../
```

This command outputs 22 files:

* `sumstat_c<chrom-number>.BMI.glm.linear` (1.26 GiB total) contains the values for the regression *per chromosome*

The files will be stored in the main directory, `plink_gwas_BMI`.

> Please note, the commands are the same whether your phenotype is quantitative or binary. Only the name of the output file will change (`.glm.linear` or `.glm.logistic.hybrid` for linear and logistic regression respectively).

## Computing the results

Now that all of the summary statistics are computed, we can download them and combine them into one single file.

By default, PLINK2 outputs a value for each of the covariates, in addition to the global p-value. However, the `hide-covar`option we have used removes covariate-specific lines from the main report. Therefore, we only need to concatenate the results into a single file.

> Please change the value of `type` based on the regression performed: "*linear*" or "*logistic.hybrid*".

```bash
pheno="BMI"
type="linear" # to change accordingly
results_path="plink_gwas_$pheno"
stat_path="plink_statistics_$pheno"

mkdir -p $stat_path

for chr_num in $(seq 1 22); do
    result="sumstat_c${chr_num}.$pheno.glm.$type"
    dx download "$results_path/$result" -o $stat_path
    if [ $chr_num -eq 1 ]; then
        head -n1 "$stat_path/$result" > "plink_sumstat_${pheno}.tsv"
    fi
    tail -n +2 "$stat_path/$result" >> "plink_sumstat_${pheno}.tsv"
done
```

This command outputs 23 files **locally**:

* `sumstat_c<chrom-number>.glm.linear` (1.23 Go total) contains the values for the regression *per chromosome*
* `plink_sumstat_BMI.tsv` (1.2 Go) contains the concatenated values for the regression

The files will be stored in a new directory named `plink_statistics_BMI`, locally this time, containing all of the summary statistics per chromosome. The combination of all of them will be located outside `plink_statistics_BMI`, at the same level, making it easier to find.

> Although the result files are big, their download should only take about a minute.

Congratulations, you have successfully completed a GWAS using PLINK2 on DNAnexus!
