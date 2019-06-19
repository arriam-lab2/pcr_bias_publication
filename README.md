## Modelling and characterising multi-template PCR biases in 16S rRNA amplicon sequencing data.

Source code, metadata and instructions required to reproduce the study. The instructions assume that you are running a Unix-like operating system (either GNU/Linux or macOS).

### Dependencies

First of all, let's clone the publication repository and some utilities

```
$ git clone https://github.com/arriam-lab2/pcr_bias_publication.git
$ cd pcr_bias_publication && git clone https://github.com/skoblov-lab/utils.git
```

- **Python**. We recommend installing all Python dependencies in a `conda` virtual environment.
```
$ conda create -n pcr python=3.6
$ conda activate pcr
(pcr) $ conda install -c conda-forge -c bioconda biom-format biopython cd-hit h5py jupyter ipython numpy pandas pymc3 scikit-bio scikit-learn scipy seaborn theano=1.0.4 regex tqdm
(pcr) $ pip install editdistance resample rpy2 fn
(pcr) $ pip install --no-cache-dir git+https://github.com/arriam-lab2/biomisc.git
```

- **R**. Since there are far too many ways your R environment might be configured (personally, we use R via an RStudio Server instance running on our server), we will only list all R dependencies and leave it up to you to install them
    - broom
    - compositions
    - dada2
    - DECIPHER
    - docstring
    - dplyr
    - ggfortify
    - ggplot2
    - ggtree
    - glmnet
    - latex2exp
    - MASS
    - phangorn
    - phyloseq
    - phytools
    - plyr
    - reshape2
    - sfsmisc
    - tidyr
    - vegan
    - zCompositions

- **QIIME2**. We use one plugin from QIIME2 for phylogenetic placement. You should install QIIME2 in a separate conda environment following the official instuctions.


### Train a taxonomic classifier

If you have installed Python dependecies following our instructions, all steps in this section are supposed to be performed with the `pcr` conda environment activated.

- Download and preprocess SILVA release 132: `silva.ipynb`
- Reduce redundancy in taxonomy groups
```
$ for group in taxonomy/groups/*.fna; do
    nseq=$(grep -c '^>' ${group})
    groupout="${group%.*}_nonredundant.fna"
    if [ "$nseq" -gt 1 ]; then
        cd-hit-est -c 1.0 -T 8 -M 2000 -i ${group} -o ${groupout} &> ${groupout}.log
    else
        cp ${group} ${groupout}
    fi
done

ls taxonomy/groups/ \
    | grep '_nonredundant.fna$' \
    | awk '{print "taxonomy/groups/"$1}' \
    | xargs cat \
    | gzip -c > taxonomy/trainset.fna.gz

rm -r taxonomy/groups
```
- Train an IDTaxa classifier: `idtaxa.Rmd`

### Preprocess the data

If you have installed Python dependecies following our instructions, all steps in this section are supposed to be performed with the `pcr` conda environment activated.

- Remove primers. We allow up to 2 mismatches (in addition to ambiguity)

```
mkdir -p noprimer
for sample in $(ls raw | awk -F '_' '{print $1"_"$2}' | sort -u); do
    fwdin="raw/${sample}_R1*.fastq.gz"
    revin="raw/${sample}_R2*.fastq.gz"
    fwdout="noprimer/${sample}_R1.fastq"
    revout="noprimer/${sample}_R2.fastq"
    primercut.py -m 2 \  # allow up to 2 mismatches
        -f GTGCCAGCMGCCGCGGTAA \
        -r GGACTACVSGGGTATCTAAT \
        ${fwdin} ${revin} ${fwdout} ${revout} \
        2> "noprimer/${sample}.log"
done
gzip noprimer/*.fastq
```

- Denoise sequences, merge pairs, remove chimera, remove zero-inflated observations, predict taxonomy: `preprocessing.Rmd`

### Phylogenetic placement

Here you must use a QIIME2 conda environment. Run phylogenetic placement wrt the reference GreenGenes 13.8 99% tree using SEPP the `q2-fragment-insertion` plugin in QIIME2. 

```
mkdir -p tree

qiime tools import \
    --input-path dada/sequences.fna \
    --output-path tree/sequences.qza \
    --type 'FeatureData[Sequence]'

qiime fragment-insertion sepp \
    --p-threads 20 \
    --i-representative-sequences tree/sequences.qza \
    --o-tree tree/tree.qza \
    --o-placements tree/placements.qza
    
qiime tools export \
    --input-path tree/tree.qza \
    --output-path tree
```

### Modelling and statistics

Everything is done in `modelling.ipynb`. The notebook contains comments to carry you along the way. 
