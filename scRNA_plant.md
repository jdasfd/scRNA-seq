# Single cell RNA sequencing analysis

## Preparation

- Anchr

The Assembler of N-free CHRomosomes created by the author wang-q [link](https://github.com/wang-q/anchr).

```bash
cargo install --git https://github.com/wang-q/anchr --branch main
anchr help
```

- Cellranger
 
Download cellranger via `wget` or `curl` to my software dir `~/share`. The download link could be found on the website [10xgenomics](https://support.10xgenomics.com/single-cell-gene-expression/software/overview/welcome).

```bash
cd ~/share/
# cellranger
wget -O cellranger-7.1.0.tar.gz "https://cf.10xgenomics.com/releases/cell-exp/cellranger-7.1.0.tar.gz?Expires=1680805080&Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9jZi4xMHhnZW5vbWljcy5jb20vcmVsZWFzZXMvY2VsbC1leHAvY2VsbHJhbmdlci03LjEuMC50YXIuZ3oiLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE2ODA4MDUwODB9fX1dfQ__&Signature=i6cFr79khQt8vYd-lOuN6YiHDMt5~qvtN0DGaSlzZ7lf676CYdL-~msHdxFp1sNQESSGY1GvRF5hBNUzt7OcmNDqz4mDTiPDRrHj3-nkcDmS1YnWqaxXTS7M95pjdRjqt8udJjALr3YKHeZN8uJU6TNf1IIsm7Jgqr5eSM7dJGlFnwPLAz9rzFADKOaeDTG5a-CuEq8-7GL4cbjyNzshvFOThmAUnYFbKQfjennorucsdYD1B2AlJEiEtEykaKUeQ4DhVZZose51R6qMkFj4iCFgMjdB4EgcRyvNkmIQ6j4kZZsEJl6CF8hROOYckalVk1EKlod9LQTHf3fCD3Zlkg__&Key-Pair-Id=APKAI7S6A5RYOXBWRPDA"
# cellranger-atac
wget -O cellranger-atac-2.1.0.tar.gz "https://cf.10xgenomics.com/releases/cell-atac/cellranger-atac-2.1.0.tar.gz?Expires=1680813186&Policy=eyJTdGF0ZW1lbnQiOlt7IlJlc291cmNlIjoiaHR0cHM6Ly9jZi4xMHhnZW5vbWljcy5jb20vcmVsZWFzZXMvY2VsbC1hdGFjL2NlbGxyYW5nZXItYXRhYy0yLjEuMC50YXIuZ3oiLCJDb25kaXRpb24iOnsiRGF0ZUxlc3NUaGFuIjp7IkFXUzpFcG9jaFRpbWUiOjE2ODA4MTMxODZ9fX1dfQ__&Signature=dHBrc-5MMWr-6hTLwDIgorZIupXByEcbI6jjA8hQTsd1aWOfgHGUKXGKsukjmw2zp8ehD2yxduUmpuMyIymctkIEuavk6jYSHS6mekyi8S0hKfE9qk8Zya-VP8gIyqVy5LaFgtFdt164-yVBKjA9kLVdBJ5qghs2WNOhJqQ2es~iH8rdb5L2OSjdv0hHTIuypMQobOKSt27kKfOPtbV-f~~g1d1MxrgBJJXu7JDS-QwLwqbU3eDUPz2IE5XqmsFYEkxAlOlYszdv-kIGxc37AvwBavMJfMzJqIlcVvi53N4szZGqqQ4V4f-p8q0oh2RYJA-fe3sOFK6fBYI0gFeeeQ__&Key-Pair-Id=APKAI7S6A5RYOXBWRPDA"

# tar them
tar -xzvf cellranger-7.1.0.tar.gz
tar -xzvf cellranger-atac-2.1.0.tar.gz

# add path to .bashrc
echo "# Single cell" >> ~/.bashrc
echo 'export PATH=~/share/cellranger-7.1.0:$PATH' >> ~/.bashrc
echo 'export PATH=~/share/cellranger-atac-2.1.0:$PATH' >> ~/.bashrc
echo >> ~/.bashrc
source ~/.bashrc

# test
cellranger -h
cellranger-atac -h
```

- Other software

```bash
# alignment
brew install kallisto
```

## Data collection

### Downloading data

- `PRJNA577177`

```bash
mkdir -p ~/data/scrna/sra
cd ~/data/scrna/sra

cat PRJNA577177.txt |
    mlr --icsv --otsv cat |
    tsv-select -H -f Experiment,"Sample\ Name",Bases |
    tr " " "_" |
    keep-header -- \
        perl -nlae '
            if ( !defined $i ) {
                print $_;
                $i = 1;
                }
            else{
                print "$F[0]\t$F[1]_$i\t$F[2]";
                $i++;
            }
        ' \
    > PRJNA577177.tsv

cat PRJNA577177.tsv |
    sed '1 s/^/#/' |
    keep-header -- sort -k2,2 -k3,3nr |
    tsv-uniq -H -f Sample_Name --max 1 |
    mlr --itsv --ocsv cat \
    > source.csv

anchr ena info | perl - -v source.csv > ena_info.yml
anchr ena prep | perl - ena_info.yml

mlr --icsv --omd cat ena_info.csv

aria2c -j 4 -x 4 -s 2 --file-allocation=none -c -i ena_info.ftp.txt

md5sum --check ena_info.md5.txt
```

### Preparing data and plant ref

- Building Ref genome

It is recommended by 10xgenomics that using ensembl will match the pipeline of cellranger. Go and check the [common ref errors](https://kb.10xgenomics.com/hc/en-us/articles/4707448154381-Common-mkref-errors-when-building-custom-reference-from-NCBI-UCSC-or-RefSeq-genomes).

```bash
mkdir -p ~/data/scrna/GENOME
cd ~/data/scrna/GENOME

# genome
wget https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-56/fasta/arabidopsis_thaliana/dna/Arabidopsis_thaliana.TAIR10.dna.toplevel.fa.gz
# gtf
wget https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-56/gtf/arabidopsis_thaliana/Arabidopsis_thaliana.TAIR10.56.gtf.gz
# decompress them
gzip -d *
# rename
mv Arabidopsis_thaliana.TAIR10.dna.toplevel.fa Atha.fa
mv Arabidopsis_thaliana.TAIR10.56.gtf Atha.gtf

# remove those lines that strand info is ?
#cat Atha.gtf |
#    perl -nlae '
#        next if $F[6] =~ /\?/;
#        next if $F[9] =~ /DA397_mgp3\d/;
#        print;
#    ' > tmp \
#    && mv tmp Atha.gtf

# cellranger will output the following mistake
#_csv.Error: field larger than field limit (131072)
vim /home/jyq/share/cellranger-7.1.0/lib/python/cellranger/reference.py
# copy following lines and add into it
#csv.field_size_limit(500 * 1024 * 1024)

# cellranger only support Atha.gtf
cellranger mkgtf Atha.gtf Atha.filtered.gtf \
    --attribute=gene_biotype:protein_coding

# mkref
cellranger mkref \
    --genome=Atha \
    --fasta=./Atha.fa \
    --genes=./Atha.filtered.gtf \
    --nthreads=6 --memgb=64

#...
#>>> Reference successfully created! <<<
#
#You can now specify this reference on the command line:
#cellranger --transcriptome=/home/jyq/data/scrna/GENOME/Atha ...
```

- fastq files acquired from SRA

Rename them to standard name in order to be accepted by cellranger pipeline.

```bash
cd ~/data/scrna/sra

# rename _1,_2
ls *.fastq.gz |
    perl -pe 's/_.+$//' |
    tsv-uniq |
    parallel -j 1 -k '
        mv {}_1.fastq.gz {}_S1_L001_R1_001.fastq.gz
        mv {}_2.fastq.gz {}_S1_L001_R2_001.fastq.gz
        '
```

### Running via cellranger count

```bash
mkdir -p ~/data/scrna/result
cd ~/data/scrna/result

ls ../sra/*.fastq.gz |
    perl -pe 's/^.*(SRR.+?)_.*$/$1/' |
    tsv-uniq |
    parallel -j 1 -k '
        cellranger count --id {} \
            --sample={} \
            --transcriptome=/home/jyq/data/scrna/GENOME/Atha \
            --fastqs=/home/jyq/data/scrna/sra \
            --localcores=16 --no-bam --nosecondary
    '
```


## References

- [cellranger mkgtf and mkref](https://blog.csdn.net/flashan_shensanceng/article/details/115718337)
- [common mkref errors](https://kb.10xgenomics.com/hc/en-us/articles/4707448154381-Common-mkref-errors-when-building-custom-reference-from-NCBI-UCSC-or-RefSeq-genomes)

- PlantscRNAdb

```bash
mkdir -p ~/data/scrna/DB
cd ~/data/scrna/DB

wget http://ibi.zju.edu.cn/plantscrnadb/download/GSE114615.zip
unzip GSE114615.zip
```

```R
options(repos='http://cran.rstudio.com/')
```
