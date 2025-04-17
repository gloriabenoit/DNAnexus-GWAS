# Using PLINK2

<blockquote id="warning">
    Please be aware, we are currently working towards updating this tutorial for the 500k Whole Genome Sequencing Release.
</blockquote>

Every file created during this analysis will be stored in a main directory called `plink_gwas_BMI`.
This section will run 22 different jobs (one per chromosome).

With our chosen instance (mem1_ssd1_v2_x16) using a `high` priority, the whole GWAS will take about **200 minutes** (3h20) and cost around **£17.04** (for a total execution time of 2341 minutes).

With the same instance (mem1_ssd1_v2_x16) using a `low` priority, if no jobs are interrupted, it will cost only **£4.56** for the same time.

> Please note that it is most unlikely for jobs to be uninterrupted. In our experience, the whole GWAS using this instance (mem1_ssd1_v2_x16) with a `low` priority and some interruptions has cost around **£9.30** altough it took almost **7h30** to complete from start to finish (with no failed jobs).

## Input files

Before running a GWAS on DNAnexus using PLINK2, you need to make sure you have these 3 files uploaded to DNAnexus:

* The phenotype: `plink_BMI.txt`
* The ids of individuals we wish to keep: `white british.txt`
* The covariates to use: `covariates.txt`

You can check their presence with the following command:

```bash
dx ls
```

Please refer to the [Input files section](input.md) if you don't have these files.

## Running a GWAS

On DNAnexus, PLINK2 is available as part of the [Swiss Army Knife app](https://ukbiobank.dnanexus.com/app/swiss-army-knife) (`swiss-army-knife`).
We choose to use the same instance for all GWASs, to simplify the code, but this can be changed to your liking. Same for the priority and the cost limit.

### Quality control

We perform the QC at the same time as our GWAS. The variants are filtered using the following options:

```text
--maf 0.0001 --hwe 1e-50 --geno 0.1 --mind 0.1
```

> Please change the thresholds according to your preferences.

### Linear regression

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/plink_$pheno.txt"
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
    prefix="/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release//ukb24306_c${chr_num}_b0_v1"
    bgen=$(basename "$prefix")

    plink_command="plink2 \
                --threads $threads \
                --maf 0.0001 \
                --hwe 1e-50 \
                --geno 0.1 \
                --mind 0.1 \
                --glm \
                --keep $ind \
                --covar $cov \
                --covar-name PC1,PC2,PC3,PC4,PC5,PC6,PC7,PC8,PC9,PC10,PC11,PC12,PC13,PC14,PC15,PC16,PC17,PC18,Age,Sex \
                --pheno plink_$pheno.txt \
                --bgen $bgen.bgen ref-first \
                --sample $bgen.sample \
                --no-psam-pheno \
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
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -y
done

dx cd ../
```

This command outputs 22 files:

* `sumstat_c<chrom-number>.BMI.glm.linear` (59.22 Go total) contains the values for the regression *per chromosome*

The files will be stored in the main directory, `plink_gwas_BMI`.

> Please note, the commands are the same whether your phenotype is quantitative or binary. Only the name of the output file will change (`.glm.linear` or `.glm.logistic.hybrid` for linear and logistic regression respectively).

## Computing the results

Now that all of the summary statistics are computed, we can download them, clean them up and combine them into one single file.

PLINK2 outputs a value for each of the covariates, in addition to the global p-value. However, these do not interest us, we only want to keep the global p-value. The following scripts helps clean up the p-values, and concatenates them into a single file.

> Please change the value of `type` based on the regression performed: *"linear"* or *"logistic.hybrid"*.

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
        head -n1 "$stat_path/$result" > "sumstat_${pheno}.ADD"
    fi
    head -n1 "$stat_path/$result" > "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "sumstat_${pheno}.ADD"
    rm "$stat_path/$result"
done
```

This command outputs 23 files **locally**:

* `sumstat_c<chrom-number>.ADD` (2.91 Go total) contains the cleaned up values for the regression *per chromosome*
* `sumstat_BMI.ADD` (2.71 Go) contains the concatenated cleaned up values for the regression

> We choose to delete the original `.BMI.glm.linear` files, seeing as they are quite heavy.

The files will be stored in a new directory named `plink_statistics_BMI`, locally this time, containing all of the summary statistics per chromosome. The combination of all of them will be located at the same level as `plink_statistics_BMI`, making it easier to find.

> Although the result files are quite big, their download should only take around 5 minutes.

Congratulations, you have successfully completed a GWAS using PLINK2 on DNAnexus!
