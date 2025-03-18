# Using regenie

Every file created during this analysis will be stored in a main directory called `regenie_gwas_BMI`.

## Input files

Before running a GWAS on DNAnexus using PLINK2, you need to make sure you have these 3 files uploaded to DNAnexus:

* The phenotype: `regenie_BMI.txt`
* The ids of individuals we wish to keep: `white british.txt`
* The covariates to use: `covariates.txt`

You can check their presence with the following command:

```bash
dx ls
```

Please refer to the [Input files section](input.md) if you don't have these files.

## Running a GWAS

On DNAnexus, regenie is available either as part of the [Swiss Army Knife app](https://ukbiobank.dnanexus.com/app/swiss-army-knife) (`swiss-army-knife`) or as its [own app](https://ukbiobank.dnanexus.com/app/regenie) called `app-regenie`.
We choose to use the same instance for all GWASs, to simplify the code, but this can be changed to your liking. Same for the priority and the cost limit.

### Quality control

Unlike with PLINK2, we cannot perform the QC at the same time as our GWAS, we must do it before hand, in preparation for running [Step 2](#step-2-linear-or-logistic-regression). The variants are filtered using following options:

```text
--maf 0.0001 --hwe 1e-50 --geno 0.1 --mind 0.1
```

> Please change the thresholds according to your preferences.

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
ind_path="/WKD_<your-name>/white_british.txt"
ind=$(basename "$ind_path")
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno/QC_lists
dx cd regenie_gwas_$pheno/QC_lists

for chr_num in $(seq 1 22); do
    prefix="/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release//ukb24306_c${chr_num}_b0_v1"
    bgen=$(basename "$prefix")

    plink_command="plink2 \
                --threads $threads \
                --maf 0.0001 \
                --hwe 1e-50 \
                --geno 0.1 \
                --mind 0.1 \
                --write-snplist allow-dups \
                --write-samples \
                --no-id-header \
                --keep $ind \
                --bgen $bgen.bgen ref-first \
                --sample $bgen.sample \
                --pheno regenie_$pheno.txt \
                --no-psam-pheno \
                --out QC_pass_c${chr_num}"

    dx run swiss-army-knife \
        --priority "$priority" --cost-limit "$cost_limit" \
        -icmd="$plink_command" \
        --instance-type $instance \
        --name="regenie_QC_${pheno}_c${chr_num}" \
        --tag="regenie" \
        --tag="QC" \
        --tag="$pheno" \
        --tag="c${chr_num}" \
        -iin="$ind_path" \
        -iin="$pheno_path" \
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -y

done

dx cd ../../
```

This command outputs 44 files:

* `QC_pass_c<chrom-number>.snplist` contains a list of SNPs that pass QC *per chromosome*
* `QC_pass_c<chrom-number>.id` contains a list of sample IDs that pass QC *per chromosome*

They will be stored into another directory named `QC_lists` to avoid crowding the main repertory for the GWAS.

### Step 0: Merging files

Before running our GWAS using regenie, we first need to merge all of the genotype call files (chromosome 1 to 22) into one file. This is in preparation for running [Step 1](#step-1-estimate-snps-contribution).

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt" # not strictly needed, but swiss-army-knife needs at least one input
geno_array="/mnt/project/Bulk/Genotype\ Results/Genotype\ calls/ukb22418_c[1-9]*"

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno/merge
dx cd regenie_gwas_$pheno/merge

merge_cmd="cp $geno_array . ; \
        ls *.bed | sed -e 's/.bed//g' > files_to_merge.txt; \
        plink --merge-list files_to_merge.txt --make-bed \
        --autosome --out c1_c22_merged; \
        rm files_to_merge.txt;
        rm $geno_array;"

dx run swiss-army-knife \
    --priority "$priority" --cost-limit "$cost_limit" \
    -icmd="$merge_cmd" \
    --instance-type $instance \
    --name="regenie_step0_c1_c22" \
    --tag="regenie" \
    --tag="Step 0" \
    -iin="$pheno_path" \
    -y

dx cd ../../
```

This command outputs 3 files:

* `c1_c22_merged.bed` contains the genotype table for our merged array genotype data
* `c1_c22_merged.bim` contains extended variant information for our merged array genotype data
* `c1_c22_merged.fam` contains the sample information for our merged array genotype data

The files will be stored in a new directory named `merge`.

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
ind_path="/WKD_<your-name>/white_british.txt"
ind=$(basename "$ind_path")
merge_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/c1_c22_merged"
merge=$(basename "$merge_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno/merge
dx cd regenie_gwas_$pheno/merge

plink_command="plink2 \
            --threads $threads \
            --maf 0.0001 \
            --hwe 1e-50 \
            --geno 0.1 \
            --mind 0.1 \
            --write-snplist \
            --write-samples \
            --no-id-header \
            --keep $ind \
            --bfile $merge \
            --pheno $pheno.txt \
            --no-psam-pheno \
            --out QC_pass_geno_array"

dx run swiss-army-knife \
    --priority "$priority" --cost-limit "$cost_limit" \
    -icmd="$plink_command" \
    --instance-type $instance \
    --name="regenie_QC_step0_${pheno}_merged" \
    --tag="regenie" \
    --tag="Step 0" \
    --tag="QC" \
    --tag="$pheno" \
    --tag="c${chr_num}" \
    -iin="$ind_path" \
    -iin="$pheno_path" \
    -iin="$merge_path.bed" \
    -iin="$merge_path.bim" \
    -iin="$merge_path.fam" \
    -y

dx cd ../../
```

This command outputs 2 files:

* `QC_pass_geno_array.snplist` contains a list of SNPs that pass QC
* `QC_pass_geno_array.id` contains a list of sample IDs that pass QC

Like in the [QC step](#quality-control), we need to save both the list of SNPs and the list of sample IDs that pass QC for our array genotype data.
They are stored in the `merge` directory.

### Step 1: Estimate SNPs contribution

The first step of a regenie GWAS is the estimation of how background SNPs contribute to the phenotype. During this step, a subset of genetic markers are used to fit a whole genome regression model that captures a good fraction of the phenotype variance attributable to genetic effects ([regenie official documentation](https://rgcgithub.github.io/regenie/overview/)).

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")
merge_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/c1_c22_merged"
merge=$(basename "$merge_path")
QC_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/QC_pass_geno_array"
QC=$(basename "$QC_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno/merge
dx cd regenie_gwas_$pheno/merge

regenie_command="regenie \
            --threads $threads \
            --step 1 \
            --bsize 1000 \
            --loocv \
            --gz \
            --extract $QC.snplist \
            --keep $QC.id \
            --bed $merge \
            --phenoFile regenie_$pheno.txt \
            --phenoCol $pheno \
            --covarFile $cov \> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command.

            --covarCol PC{1:18} \
            --out ${pheno}_merged"

dx run swiss-army-knife \
    --priority "$priority" --cost-limit "$cost_limit" \
    -icmd="$regenie_command" \
    --instance-type $instance \
    --name="regenie_step1_${pheno}" \
    --tag="regenie" \
    --tag="Step 1" \
    --tag="$pheno" \
    -iin="$pheno_path" \
    -iin="$cov_path" \
    -iin="$merge_path.bed" \
    -iin="$merge_path.bim" \
    -iin="$merge_path.fam" \
    -iin="$QC_path.snplist" \
    -iin="$QC_path.id" \
    -y

dx cd ../../
```

This command outputs 2 files:

* `BMI_merged_pred.list` contains a list of blup files needed for [Step 2](#step-2-linear-or-logistic-regression)
* `BMI_merged_1.loco.gz` contains per-chromosome LOCO predictions

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command.

### Step 2: Linear regression

The second step of a regenie GWAS is the regression. During this step, whole genome markers are tested for association with the phenotype *conditional upon* the prediction from the regression model in [Step 1](#step-1-estimate-snps-contribution) ([regenie official documentation](https://rgcgithub.github.io/regenie/overview/)).

> To not require too much space, we gzip the results using the `--gz` option.

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")
pred_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/${pheno}_merged_pred.list"
pred=$(basename "$pred_path")
loco_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/${pheno}_merged_1.loco.gz"

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

for chr_num in $(seq 1 22); do
    prefix="/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release//ukb24306_c${chr_num}_b0_v1"
    bgen=$(basename "$prefix")
    QC_path="/WKD_<your-name>/regenie_gwas_$pheno/QC_lists/QC_pass_c${chr_num}"
    QC=$(basename "$QC_path")
    
    regenie_command="regenie \
                --threads $threads \
                --step 2 \
                --bsize 200 \
                --approx \
                --firth-se \
                --firth \
                --gz \
                --pred $pred \
                --extract $QC.snplist \
                --keep $QC.id \
                --bgen $bgen.bgen \
                --ref-first \
                --sample $bgen.sample \
                --phenoFile regenie_$pheno.txt \
                --phenoCol $pheno \
                --covarFile $cov \
                --covarCol Sex \
                --covarCol Age \
                --covarCol PC{1:18} \
                --out sumstat_c${chr_num}"

    dx run swiss-army-knife \
        --priority "$priority" --cost-limit "$cost_limit" \
        -icmd="$regenie_command" \
        --instance-type $instance \
        --name="regenie_step2_${pheno}_c${chr_num}" \
        --tag="regenie" \
        --tag="Step 2" \
        --tag="$pheno" \
        --tag="c${chr_num}" \
        -iin="$pheno_path" \
        -iin="$cov_path" \
        -iin="$pred_path" \
        -iin="$loco_path" \
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -iin="$QC_path.snplist" \
        -iin="$QC_path.id" \
        -y
done

dx cd ../
```

This command outputs 22 files:

* `sumstat_c<chrom-number>_BMI.regenie.gz` contains the values for the regression *per chromosome*

The files will be stored in the main directory, `regenie_gwas_BMI`.

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command to perform a logistic regression rather than a linear one.

## Computing the results

Now that all of the summary statistics are computed, we can download them and combine them into one single file.

```bash
pheno="BMI"
output_path="regenie_gwas_$pheno"
stat_path="$output_path/statistics"

mkdir -p $stat_path

for chr_num in $(seq 22 22); do
    result="sumstat_c${chr_num}_$pheno.regenie"
    dx download "$output_path/$result.gz" -o $stat_path
    gunzip $stat_path/$result
    if [ $chr_num -eq 1 ]; then
        head -n1 "$stat_path/$result" > "$output_path/sumstat_${pheno}.regenie"
    fi
    tail -n +2 "$stat_path/$result" >> "$output_path/sumstat_${pheno}.regenie"
done
```

This command outputs 23 files:

* `sumstat_c<chrom-number>.regenie` contains the values for the regression *per chromosome*
* `sumstat_BMI.regenie` contains the concatenated values for the regression

The files will be stored in a new directory named `regenie_gwas_BMI`, locally this time, with inside another directory named `statistics` containing all of the summary statistics per chromosome. The combination of all of them will be located at the same level than `statistics`, making it easier to find.

Congratulations, you have successfully completed a GWAS using regenie on DNAnexus!
