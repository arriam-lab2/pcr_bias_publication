## Modelling and characterising multi-template PCR biases in 16S rRNA amplicon sequencing data.
Source code and metadata required to reproduce the study

### Installing dependencies

#TODO

### Train a taxonomic classifier

- Download and preprocess SILVA release 132: `silva.ipynb`
- Reduce redundancy in taxonomy groups
```
for group in taxonomy/groups/*.fna; do
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

- Remove primers

You can use any other way to remove universal primers. We use our our tool: `primercut.py`. You can install it from https://github.com/arriam-lab2/biomisc.git to get primercut.py:
```
pip install --no-cache-dir git+https://github.com/arriam-lab2/biomisc.git
```

- Allow up to 2 mismatches

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

- Run phylogenetic placement wrt the reference GreenGenes 13.8 99% tree using SEPP the `q2-fragment-insertion` plugin in QIIME 2.

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
