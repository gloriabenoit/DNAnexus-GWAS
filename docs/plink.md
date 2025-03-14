# Using PLINK2

## Input files

Before running a GWAS on DNAnexus using PLINK2, you need to make sure you have these 3 files uploaded to DNAnexus:

* The phenotype (BMI.txt)
* The ids of individuals we wish to keep (white british.txt)
* The covariates to use (covariates.txt)

You can check their presence with the following command:

```bash
dx ls
```

Please refer to the [Input files section](input.md) if you don't have these files.

## Running a GWAS

On DNAnexus, PLINK2 is available as part of the Swiss Army Knife app.
We choose to use the same instance for all GWASs, to simplify the code, but this can be changed to your liking. Same for the priority and the cost limit. We also run the QC at the same time as our GWAS, please change the thresholds according to your preferences.

```bash
pheno_path="/WKD_<your-name>/BMI.txt"
pheno=$(basename "$pheno_path" .txt)
ind_path="/WKD_<your-name>/white_british.txt"
ind=$(basename "$ind_path")
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")

instance="mem2_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p gwas_$pheno

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
                --pheno $pheno.txt \
                --bgen $bgen.bgen ref-first \
                --sample $bgen.sample \
                --no-psam-pheno \
                --out gwas_$pheno/sumstat_c${chr_num}"

    dx run swiss-army-knife \
        --priority "$priority" --cost-limit "$cost_limit" \
        -icmd="$plink_command" \
        --instance-type "$instance" \
        --name="gwas_${pheno}_c${chr_num}" \
        -iin="$ind_path" \
        -iin="$cov_path" \
        -iin="$pheno_path" \
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -y
done
```

This script will create a new directory named `gwas_<phenotype>` in the project on DNAnexus. In this folder, you will find a total of 44 files, named `sumstat_c<num>` (two per chromosome, the summary statistics and the log).

## Computing the results

Now that all of the summary statistics are computed, we can clean them up and combine them into one clean file. We will create a new directory named `gwas_<phenotype>`, locally this time, with inside another directory named `statistics` containing all of the summary statistics per chromosome. The combination of all of them will be located at the same level than `statistics`, making it easier to find.

```bash
pheno="BMI"
type="linear" # to change accordingly
output_path="gwas_$pheno"
stat_path="$output_path/statistics"

mkdir -p $output_path
mkdir -p $stat_path

for chr_num in $(seq 1 22); do
    result="sumstat_c${chr_num}.PHENO1.glm.$type"
    dx download "gwas_$pheno/$result" -o $stat_path
    if [ $chr_num -eq 1 ]; then
        head -n1 "$stat_path/$result" > "$output_path/sumstat_${pheno}.ADD"
    fi
    head -n1 "$stat_path/$result" > "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "$output_path/sumstat_${pheno}.ADD"
done
```

Congratulations, you have successfully completed a GWAS using PLINK2 on DNAnexus!
