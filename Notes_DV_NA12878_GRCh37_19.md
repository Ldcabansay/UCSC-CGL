## Notes: DeepVariant on NA12878_GRCh37_19

### Preliminaries
```
BASE="${HOME}/NA12878_HG001_HiSeq_300x"

#Preliminaries for downloading DV (check to make sure latest model is used):

BIN_VERSION="0.5.1"
MODEL_VERSION="0.5.0"
MODEL_CL="182548131"
BIN_BUCKET="${BUCKET}/binaries/DeepVariant/${BIN_VERSION}/DeepVariant-${BIN_VERSION}+cl-*"
MODEL_BUCKET="${BUCKET}/models/DeepVariant/${MODEL_VERSION}/DeepVariant-inception_v3-${MODEL_VERSION}+cl-${MODEL_CL}.data-wgs_standard"

#Preliminaries for data handling:

INPUT_DIR="${BASE}/input"
BIN_DIR="${HOME}/deepvariant/bin"
MODELS_DIR="${HOME}/deepvariant/models"
MODEL="${MODELS_DIR}/model.ckpt"
DATA_DIR="${BASE}/input"
REF="${DATA_DIR}/ref/GRCh37_WG.fa"
BAM="${DATA_DIR}/bam/RMNISTHS_30xdownsample.bam"
TRUTH_VCF="${DATA_DIR}/vcf/HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_PGandRTGphasetransfer.vcf.gz"
TRUTH_BED="${DATA_DIR}/vcf/HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_nosomaticdel.bed.gz"
#note: make sure VCF and BED files are zipped and indexed

N_SHARDS="64"
OUTPUT_DIR="${BASE}/output"
EXAMPLES="${OUTPUT_DIR}/19_RMNISTHS_30x_CHG001_GRCh37_GIAB.tfrecord@${N_SHARDS}.gz"
CALL_VARIANTS_OUTPUT="${OUTPUT_DIR}/19_RMNISTHS_30x_CHG001_GRCh37_GIAB.cvo.tfrecord.gz"
OUTPUT_VCF="${OUTPUT_DIR}/19_RMNISTHS_30x_CHG001_GRCh37_GIAB.output.vcf.gz"
LOG_DIR="${OUTPUT_DIR}/logs"
```
### Create a local directory structure:
```
mkdir -p "${OUTPUT_DIR}"
mkdir -p "${BIN_DIR}"
mkdir -p "${DATA_DIR}"
mkdir -p "${MODELS_DIR}"
mkdir -p "${LOG_DIR}"
```
### Download binaries, models, and data:

##### Download DeepVariant binaries:

```
time gsutil -m cp -r "${BIN_BUCKET}/*" "${BIN_DIR}"
chmod a+x "${BIN_DIR}"/*
```
Then install the pre-reqs:
```
cd "${BIN_DIR}"; time bash run-prereq.sh; cd -
```
##### Download Models: 
```
time gsutil -m cp -r "${MODEL_BUCKET}/*" "${MODELS_DIR}"
```

##### Download Data:

REF:
Our refs GRCh37_WG.fa  GRCh37_WG.fa.fai were retrieved from local source: /data/users/common/GIAB/ref

```
cd "${DATA_DIR}/ref"
scp -r <user>@<domain>:~/data/users/common/GIAB/ref/GRCh37_WG* .
```
BAM:
```
cd "${DATA_DIR}/bam"
wget ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/NA12878/NIST_NA12878_HG001_HiSeq_300x
```

TruthVCF and TruthBED:
```
cd "${DATA_DIR}/vcf"
#get vcf and index(tbi) file:
wget ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/release/NA12878_HG001/latest/GRCh37/HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_PGandRTGphasetransfer.vcf.gz
wget ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/release/NA12878_HG001/latest/GRCh37/HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_PGandRTGphasetransfer.vcf.gz.tbi
#get bed:
wget ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/release/NA12878_HG001/latest/GRCh37/HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_nosomaticdel.bed
```
If bed isn't zipped and doesnt have an index file, then you'll need to use bgzip to zip and tabix index. Both tools require samtools and htslib packages.

To zip and index bed:
```
bgzip HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_nosomaticdel.bed
tabix HG001_GRCh37_GIAB_highconf_CG-IllFB-IllGATKHC-Ion-10X-SOLID_CHROM1-X_v.3.3.2_highconf_nosomaticdel.bed.gz
```

##### Download some extra packages:
These packages will be used mainly to run deepvariant packages and use hap.py 

```
sudo apt-get -y install parallel
sudo apt-get -y install samtools
sudo apt-get -y install docker.io
```

## Make Examples:

note: check indexing convention on VCF and BED files (ex: chr19 vs 19)
Also, for whole genome run remove `--regions chr#` flag
```
( time seq 0 $((N_SHARDS-1)) | \
  parallel --halt 2 --joblog "${LOG_DIR}/log" --res "${LOG_DIR}" \
    python "${BIN_DIR}"/make_examples.zip \
      --mode calling \
      --ref "${REF}" \
      --reads "${BAM}" \
      --regions 19 \
      --examples "${EXAMPLES}" \
      --task {}
) >"${LOG_DIR}/make_examples.log" 2>&1
```

Tip: you can change machine allocation after running make-examples, 32CPU should be enough for the rest of the pipeline.


## Call Variants:
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
Note: to use hap.py, truth bed file needs to be zipped (bgzip) and indexed (tabix).

To run hap.py on docker image (use if you have no local hap.py software):
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
  -l 19
```  
note: check indexing convention


Alternatively, if you have a local installation of hap.py:
```
#first define variable with happy path 
HAPPY="${HOME}/software/hap.py/hap.py-build/bin/hap.py" 

Then run:
python2 "${HAPPY}" \
"${TRUTH_VCF}" \
"${OUTPUT_VCF}" \
--preprocess-truth \
-f "${TRUTH_BED}" \
-o "${OUTPUT_DIR}/happy.output" \
-r "${REF}" \
-l 19
```
