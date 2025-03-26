# Using regenie

Every file created during this analysis will be stored in a main directory called `regenie_gwas_BMI`.
This section runs a total of 47 different jobs, over four stages:

1. [Quality control](#quality-control) runs 22 jobs (one per chromosome)
2. [Merging files](#merging-files) runs 2 jobs, for the merge and the QC of the merged data
3. [Step 1](#step-1-estimate-snps-contribution) runs 1 job for regenie's SNPs contribution estimation (step 1)
4. [Step 2](#step-2-linear-regression) rusn 22 job for regenie's regression (step 2, one per chromosome)

> To optimize your GWAS's overall time, you can perform the QC in parallel to every other step beside step 2 (for which it is needed).
> The rest needs to be done in order.

With our chosen instance (mem1_ssd1_v2_x16) using a `high` priority, the whole GWAS will take about **700 minutes** (11h40, when running each step in order) and cost around **£21.46** (for a total execution time of 2948 minutes).

With the same instance (mem1_ssd1_v2_x16) using a `low` priority, if no jobs are interrupted, it will cost only **£5.74** for the same time.

You can find the details about the jobs run in the following table:

<center>

| Action   |  Time | Execution time | Cost (high) | Cost (low, no interruption) |
|----------|:-----:|:--------------:|:-----------:|:---------------------------:|
| QC       |  2h39 |       30h      |    13.10    |             3.50            |
| Merge    | 51min |      51min     |     0.37    |             0.10            |
| Merge QC | 10min |      10min     |     0.07    |             0.02            |
| Step 1   |  7h10 |      7h10      |     3.13    |             0.83            |
| Step 2   | 48min |      10h57     |     4.78    |             1.28            |

</center>

> Please note that it is most unlikely for jobs to be uninterrupted. In our experience, the whole GWAS using this instance (mem1_ssd1_v2_x16) with a `low` priority and some interruptions has cost around **£12.57** (without accounting for failed jobs, of which there were a few). Since most steps need to be done in order, it also took a total of **54h35** to complete the GWAS from start to finish (over multiple sessions since some jobs failed).
> If you count the failed jobs, you can add £5.36 to the total cost, and 12h04 to the total time taken.

## Input files

Before running a GWAS on DNAnexus using PLINK2, you need to make sure you have these 3 files uploaded to DNAnexus:

* The phenotype: `regenie_BMI.txt`
* The ids of individuals we wish to keep: `white_british.txt`
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

Unlike with PLINK2, we cannot perform the QC at the same time as our GWAS, we must do it before hand, in preparation for running [Step 2](#step-2-linear-regression). The variants are filtered using the following options:

```text
--maf 0.0001 --hwe 1e-50 --geno 0.1 --mind 0.1
```

> Please change the thresholds according to your preferences.

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/regenie_$pheno.txt"
ind_path="/gwas_tutorial/white_british.txt"
ind=$(basename "$ind_path")
cov_path="/gwas_tutorial/covariates.txt"
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

* `QC_pass_c<chrom-number>.snplist` (569.43 Mo total) contains a list of SNPs that pass QC *per chromosome*
* `QC_pass_c<chrom-number>.id` (59.2 Mo total) contains a list of sample IDs that pass QC *per chromosome*

They will be stored into another directory named `QC_lists` to avoid crowding the main repertory for the GWAS.

### Merging files

Before running our GWAS using regenie, we first need to merge all of the genotype call files (chromosome 1 to 22) into one file. This is in preparation for running [Step 1](#step-1-estimate-snps-contribution).

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/regenie_$pheno.txt" # not strictly needed, but swiss-army-knife needs at least one input
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
    --tag="Merge" \
    -iin="$pheno_path" \
    -y

dx cd ../../
```

This command outputs 3 files:

* `c1_c22_merged.bed` (89.18 Go) contains the genotype table for our merged array genotype data
* `c1_c22_merged.bim` (21.47 Mo) contains extended variant information for our merged array genotype data
* `c1_c22_merged.fam` (11.64 Mo) contains the sample information for our merged array genotype data

The files will be stored in a new directory named `merge`. Now we can perform QC on this data as well.

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/regenie_$pheno.txt"
ind_path="/gwas_tutorial/white_british.txt"
ind=$(basename "$ind_path")
merge_path="/gwas_tutorial/regenie_gwas_$pheno/merge/c1_c22_merged"
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
    --tag="Merge" \
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

* `QC_pass_geno_array.snplist` (6.92 Mo) contains a list of SNPs that pass QC
* `QC_pass_geno_array.id` (6.58 Mo) contains a list of sample IDs that pass QC

Like in the [QC step](#quality-control), we need to save both the list of SNPs and the list of sample IDs that pass QC for our array genotype data.
They are stored in the `merge` directory.

### Step 1: Estimate SNPs contribution

The first step of a regenie GWAS is the estimation of how background SNPs contribute to the phenotype. During this step, a subset of genetic markers are used to fit a whole genome regression model that captures a good fraction of the phenotype variance attributable to genetic effects. For more information, check the [official documentation](https://rgcgithub.github.io/regenie/overview/).

> To not require too much space, we gzip the results using the `--gz` option.

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/regenie_$pheno.txt"
cov_path="/gwas_tutorial/covariates.txt"
cov=$(basename "$cov_path")
merge_path="/gwas_tutorial/regenie_gwas_$pheno/merge/c1_c22_merged"
merge=$(basename "$merge_path")
QC_path="/gwas_tutorial/regenie_gwas_$pheno/merge/QC_pass_geno_array"
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
            --covarFile $cov \

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

* `BMI_merged_pred.list` (46 o) contains a list of blup files needed for [Step 2](#step-2-linear-regression)
* `BMI_merged_1.loco.gz` (38.31 Mo) contains per-chromosome LOCO predictions

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command.

### Step 2: Linear regression

The second step of a regenie GWAS is the regression. During this step, whole genome markers are tested for association with the phenotype *conditional upon* the prediction from the regression model in [Step 1](#step-1-estimate-snps-contribution). For more information, check the [official documentation](https://rgcgithub.github.io/regenie/overview/).

> To not require too much space, we gzip the results using the `--gz` option.

```bash
pheno="BMI"
pheno_path="/gwas_tutorial/regenie_$pheno.txt"
cov_path="/gwas_tutorial/covariates.txt"
cov=$(basename "$cov_path")
pred_path="/gwas_tutorial/regenie_gwas_$pheno/merge/${pheno}_merged_pred.list"
pred=$(basename "$pred_path")
loco_path="/gwas_tutorial/regenie_gwas_$pheno/merge/${pheno}_merged_1.loco.gz"

instance="mem1_ssd1_v2_x16"
threads=16
priority="low"
cost_limit=3

dx mkdir -p regenie_gwas_$pheno
dx cd regenie_gwas_$pheno

for chr_num in $(seq 1 22); do
    prefix="/Bulk/Whole genome sequences/Population level genome variants, BGEN format - interim 200k release//ukb24306_c${chr_num}_b0_v1"
    bgen=$(basename "$prefix")
    QC_path="/gwas_tutorial/regenie_gwas_$pheno/QC_lists/QC_pass_c${chr_num}"
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

* `sumstat_c<chrom-number>_BMI.regenie.gz` (919.87 Mo total) contains the values for the regression *per chromosome*

The files will be stored in the main directory, `regenie_gwas_BMI`.

> Please note, when using a binary phenotype you need to add the `--bt` option to the regenie command to perform a logistic regression rather than a linear one.

## Computing the results

Now that all of the summary statistics are computed, we can download them and combine them into one single file.

```bash
pheno="BMI"
results_path="regenie_gwas_$pheno"
stat_path="regenie_statistics_$pheno"

mkdir -p $stat_path

for chr_num in $(seq 1 22); do
    result="sumstat_c${chr_num}_$pheno.regenie"
    dx download "$results_path/$result.gz" -o $stat_path
    gunzip $stat_path/$result
    if [ $chr_num -eq 1 ]; then
        head -n1 "$stat_path/$result" > "sumstat_${pheno}.regenie"
    fi
    tail -n +2 "$stat_path/$result" >> "sumstat_${pheno}.regenie"
done
```

This command outputs 23 files **locally**:

* `sumstat_c<chrom-number>.regenie` (3.30 Go total) contains the values for the regression *per chromosome*
* `sumstat_BMI.regenie` (3.08 Go) contains the concatenated values for the regression

The files will be stored in a new directory named `regenie_statistics_BMI`, locally this time, containing all of the summary statistics per chromosome. The combination of all of them will be located at the same level as `regenie_statistics_BMI`, making it easier to find.

> Although the result files are quite big, their download should only take about a minute.

Congratulations, you have successfully completed a GWAS using regenie on DNAnexus!
