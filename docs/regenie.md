# Using regenie

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

Unlike with PLINK2, we cannot perform the QC at the same time as our GWAS, we must do it before hand. The variants are filtered using following options:

```text
--maf 0.0001 --hwe 1e-50 --geno 0.1 --mind 0.1
```

> Please change the thresholds according to your preferences.

For each chromosome, we will compute two files:

* A list of SNPs that pass QC: `QC_pass_c<chrom-number>.snplist`
* A list of sample IDs that pass QC: `QC_pass_c<chrom-number>.id`

They will be stored into another directory called `QC_lists` to avoid crowding the main repertory for the GWAS.

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
ind_path="/WKD_<your-name>/white_british.txt"
ind=$(basename "$ind_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

dx mkdir -p QC_lists
dx cd QC_lists

for chr_num in $(seq 22 22); do
    prefix="/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release//ukb24306_c${chr_num}_b0_v1"
    bgen=$(basename "$prefix")

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
                --bgen $bgen.bgen ref-first \
                --sample $bgen.sample \
                --pheno regenie_$pheno.txt \
                --no-psam-pheno \
                --out QC_pass_c${chr_num}"

    dx run swiss-army-knife \
        --priority low --cost-limit 3 \
        -icmd="$plink_command" \
        --instance-type $instance \
        --name="reg_QC_${pheno}_c${chr_num}" \
        -iin="$ind_path" \
        -iin="$pheno_path" \
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -y
done

dx cd ../../
```

This is in preparation for running [Step 2](#step-2-linear-or-logistic-regression).

### Step 0: Merging files

Before running our GWAS using regenie, we first need to merge all of the genotype call files (chromosome 1 to 22) into one file. This is in preparation for running [Step 1](#step-1-estimate-snps-contribution).

```bash
instance="mem1_ssd1_v2_x16"
threads=16
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt" # not strictly needed, but swiss-army-knife needs at least one input

geno_array="/mnt/project/Bulk/Genotype\ Results/Genotype\ calls/ukb22418_c[1-9]*"

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

dx mkdir -p merge
dx cd merge

merge_cmd="cp $geno_array . ; \
        ls *.bed | sed -e 's/.bed//g' > files_to_merge.txt; \
        plink --threads $threads \
        --merge-list files_to_merge.txt --make-bed \
        --autosome --out c1_c22_merged; \
        rm files_to_merge.txt;"

dx run swiss-army-knife \
    --priority low --cost-limit 3 \
    -icmd="$merge_cmd" \
    --instance-type $instance \
    --name="${pheno}_step0" \
    --tag="Step 0" \
    -iin="$pheno_path" \
    -y

dx cd ../../
```

Like in the [QC step](#quality-control), we need to save both the list of SNPs (`QC_pass_geno_array.snplist`) and the list of sample IDs (`QC_pass_geno_array.id`) that pass QC for our array genotype data.

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

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

dx mkdir -p merge
dx cd merge

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
            --bfile $merged \
            --pheno regenie_$pheno.txt \
            --no-psam-pheno \
            --out QC_pass_geno_array"

dx run swiss-army-knife \
    --priority low --cost-limit 3 \
    -icmd="$plink_command" \
    --instance-type $instance \
    --name="reg_QC_${pheno}_merged" \
    -iin="$ind_path" \
    -iin="$pheno_path" \
    -iin="$merge_path.bed" \
    -iin="$merge_path.bim" \
    -iin="$merge_path.fam" \
    -y

dx cd ../../
```

### Step 1: Estimate SNPs contribution

The first step of a regenie GWAS is the estimation of how background SNPs contribute to the phenotype. During this step, a subset of genetic markers are used to fit a whole genome regression model that captures a good fraction of the phenotype variance attributable to genetic effects ([regenie official documentation](https://rgcgithub.github.io/regenie/overview/)).

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
ind_path="/WKD_<your-name>/white_british.txt"
ind=$(basename "$ind_path")
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")
merge_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/c1_c22_merged"
merge=$(basename "$merge_path")
QC_path="/WKD_<your-name>regenie_gwas_$pheno/merge/QC_pass_geno_array"
QC=$(basename "$QC_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

dx mkdir -p merge
dx cd merge

regenie_command="regenie \
            --threads $threads \
            --step 1 \
            --bsize 1000 \
            --loocv \
            --gz \
            --extract $QC$.snplist \
            --keep $QC.id \
            --bed $merge \
            --phenoFile regenie_$pheno.txt \
            --phenoCol $pheno \
            --covarFile $cov \
            --covarCol Sex \
            --covarCol Age \
            --covarCol PC{1:18} \
            --out ${pheno}_step1"

dx run swiss-army-knife \
    --priority low --cost-limit 3 \
    -icmd="$regenie_command" \
    --instance-type $instance \
    --name="${pheno}_step1" \
    --tag="Step 1" \
    -iin="$ind_path" \
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

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command.

### Step 2: Linear or logistic regression

The second step of a regenie GWAS is the regression. During this step, whole genome markers are tested for association with the phenotype *conditional upon* the prediction from the regression model in [Step 1](#step-1-estimate-snps-contribution) ([regenie official documentation](https://rgcgithub.github.io/regenie/overview/)).

```bash
pheno="BMI"
pheno_path="/WKD_<your-name>/regenie_$pheno.txt"
cov_path="/WKD_<your-name>/covariates.txt"
cov=$(basename "$cov_path")
pred_path="/WKD_<your-name>/regenie_gwas_$pheno/merge/${pheno}_pred.list"
pred=$(basename "$pred_path")

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

for chr_num in $(seq 22 22); do
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
                --bgen $bgen.bgen ref-first \
                --sample $bgen.sample \
                --phenoFile regenie_$pheno.txt \
                --phenoCol $pheno \
                --covarFile $cov \
                --covarCol Sex \
                --covarCol Age \
                --covarCol PC{1:18} \
                --out sumstat_c${chr_num}"

    dx run swiss-army-knife \
        --priority low --cost-limit 3 \
        -icmd="$regenie_command" \
        --instance-type $instance \
        --name="gwas_${pheno}_step2" \
        --tag="Step 2" \
        -iin="$pheno_path" \
        -iin="$cov_path" \
        -iin="$pred_path" \
        -iin="$prefix.sample" \
        -iin="$prefix.bgen" \
        -iin="$QC_path.snplist" \
        -iin="$QC_path.id" \
        -y
done

dx cd ../../
```

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command.

This script will create a new directory named `regenie_gwas_<phenotype>` in the project on DNAnexus. In this folder, you will find a total of 44 files, named `sumstat_c<num>` (two per chromosome, the summary statistics and the log).

## Computing the results

Now that all of the summary statistics are computed, we can clean them up and combine them into one clean file. We will create a new directory named `regenie_gwas_<phenotype>`, locally this time, with inside another directory named `statistics` containing all of the summary statistics per chromosome. The combination of all of them will be located at the same level than `statistics`, making it easier to find.

```bash
pheno="BMI"
type="linear" # to change accordingly
output_path="regenie_gwas_$pheno"
stat_path="$output_path/statistics"

mkdir -p $output_path
mkdir -p $stat_path

for chr_num in $(seq 1 22); do
    result="sumstat_c${chr_num}.$pheno.glm.$type"
    dx download "gwas_$pheno/$result" -o $stat_path
    if [ $chr_num -eq 1 ]; then
        head -n1 "$stat_path/$result" > "$output_path/sumstat_${pheno}.ADD"
    fi
    head -n1 "$stat_path/$result" > "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "$stat_path/sumstat_c${chr_num}.ADD"
    grep "ADD" "$stat_path/$result" >> "$output_path/sumstat_${pheno}.ADD"
done
```

Congratulations, you have successfully completed a GWAS using regenie on DNAnexus!
