## DeepVariant Test: Chromosome 19 
Note: I'm running this test on the same virtual machine that I did the original case-study on, so instead of re-installing binaries for deep variant (packages, models, etc). I just created a new directory for chr19 data and created a new script that called the deep variant software binaries from the case-study directory while using the data in chr19 directory. It might be more beneficial(and economical) to request another machine entirely with more appropriate lower compute resources for future single chromosome tests.

### Set Compute Resources:
Allocate virtual machine resources, enough for a DeepVariant run on only 1 chromosome as opposed to whole genome:

```


tba


```
### Set Preliminaries:
load all the data into appropriate directories. (details tba)

Create a screen to run test:

This test will take some time to run, so create a screen in order to let the machine continue to run in the background:
```
screen -S chr19test
```
Once you have the test running you can use `crtl+a +d` to detch from screen and the program will continue to run on the machine. Once you detach it will be safe disconnect from gcp/close laptop/etc. (NOTE: this does mean to stop machine instance, keep gcp machine instance running). To re-attach screen and check on progress type command `screen -rd chr19test`.

In screen make some script variables:
```
BASE="${HOME}/chr19"

BIN_VERSION="0.5.1"
MODEL_VERSION="0.5.0"
MODEL_CL="182548131"

INPUT_DIR="${BASE}/input"
BIN_DIR="${HOME}case-study/input/bin"
MODELS_DIR="${HOME}case-study/input/models"
MODEL="${MODELS_DIR}/model.ckpt"
DATA_DIR="${INPUT_DIR}/illumina_data"
REF="${DATA_DIR}/ref/ref_hg19_WG.fa.gz"
BAM="${DATA_DIR}/bam/NA12878_S1.bam"
TRUTH_VCF="${DATA_DIR}/vcf/NA12878_pg.vcf.gz"
TRUTH_BED="${DATA_DIR}/confident_bed/ConfidentRegions.bed"

N_SHARDS="64" 
#not sure if shards this is necessary w/1 chromosome 

OUTPUT_DIR="${BASE}/output"
EXAMPLES="${OUTPUT_DIR}/chr19_NA12878_S1.tfrecord@${N_SHARDS}.gz"
CALL_VARIANTS_OUTPUT="${OUTPUT_DIR}/chr19_NA12878_S1.cvo.tfrecord.gz"
OUTPUT_VCF="${OUTPUT_DIR}/chr19_NA12878_S1.output.vcf.gz"
LOG_DIR="${OUTPUT_DIR}/logs"
```

Create Local Directory Structure:

if using the same bin and model directories from case-study, you shouldn't need to make new ones, just make sure bash script points to correct locations. Only need:

```
mkdir -p "${OUTPUT_DIR}"
mkdir -p "${LOG_DIR}"
```

If running directly (without first going through case-study tutorial):
```
mkdir -p "${OUTPUT_DIR}"
mkdir -p "${LOG_DIR}"
mkdir -p "${BIN_DIR}"
mkdir -p "${DATA_DIR}"
mkdir -p "${MODELS_DIR}"
```


## Run Make Examples
If you want to run DeepVariant only on a select region, such as a specific chromosome, use flag `--regions chr#` (see DeepVariant docs for other different region syntax)
To run make examples using defined preliminaries above:

```
( time seq 0 $((N_SHARDS-1)) | \
  parallel --halt 2 --joblog "${LOG_DIR}/log" --res "${LOG_DIR}" \
    python "${BIN_DIR}"/make_examples.zip \
      --mode calling \
      --ref "${REF}" \
      --reads "${BAM}" \
      --regions chr19 \
      --examples "${EXAMPLES}" \
      --task {}
) >"${LOG_DIR}/make_examples.log" 2>&1
```

## Call Variants
```
( time python "${BIN_DIR}"/call_variants.zip \
    --outfile "${CALL_VARIANTS_OUTPUT}" \
    --examples "${EXAMPLES}" \
    --checkpoint "${MODEL}" \
    --batch_size 32
) >"${LOG_DIR}/call_variants.log" 2>&1
```

## Post-Process Variants (VCF only):
```
( time python "${BIN_DIR}"/postprocess_variants.zip \
    --ref "${REF}" \
    --infile "${CALL_VARIANTS_OUTPUT}" \
    --outfile "${OUTPUT_VCF}"
) >"${LOG_DIR}/postprocess_variants.log" 2>&1
```

## Happy Compare:
```
sudo docker pull pkrusche/hap.py
sudo docker run -it \
-v "${DATA_DIR}:${DATA_DIR}" \
-v "${OUTPUT_DIR}:${OUTPUT_DIR}" \
pkrusche/hap.py /opt/hap.py/bin/hap.py \
  "${TRUTH_VCF}" \
  "${OUTPUT_VCF}" \
  --preprocess-truth \
  -f "${TRUTH_BED}" \
  -r "${REF}" \
  -o "${OUTPUT_DIR}/happy.output" \
  -l chr19
  ```
