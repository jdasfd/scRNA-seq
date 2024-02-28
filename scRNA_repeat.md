# Repeat a single cell RNA sequencing result

## Preparation

```bash
cargo install --git https://github.com/wang-q/anchr --branch main
```

## Downloading data

```bash
mkdir -p ~/data/sc_npc/sra
cd ~/data/sc_npc/sra

cat PRJNA680328.txt |
    mlr --icsv --otsv cat |
    tsv-select -H -f Experiment,source_name,Bases \
    > PRJNA680328.tsv

cat PRJNA680328.tsv |
    sed '1 s/^/#/' |
    keep-header -- sort -k2,2 -k3,3nr |
    tsv-uniq -H -f source_name --max 1 |
    mlr --itsv --ocsv cat \
    > source.csv

anchr ena info | perl - -v source.csv > ena_info.yml
anchr ena prep | perl - ena_info.yml

mlr --icsv --omd cat ena_info.csv

aria2c -j 4 -x 4 -s 2 --file-allocation=none -c -i ena_info.ftp.txt

md5sum --check ena_info.md5.txt
```

```bash
wget https://www.ncbi.nlm.nih.gov/geo/download/?acc=GSE162025&format=file
```
