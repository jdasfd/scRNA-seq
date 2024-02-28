# ATAC-seq

```bash
cpanm -nq https://github.com/wang-q/App-Anchr/archive/0.5.4.tar.gz
```

```bash
mkdir -p ~/data/snapATAC/sra
cd ~/data/snapATAC/sra

cat PRJNA635415.txt |
    mlr --icsv --otsv cat |
    tsv-filter -H --str-eq "Assay\ Type":ATAC-seq |
    head -n 3 |
    tsv-select -H -f Experiment,"Sample\ Name",Bases \
    > SraRunTable.tsv

cat SraRunTable.tsv |
    sed '1 s/^/#/' |
    keep-header -- tsv-sort -k2,2 -k3,3nr |
    tsv-uniq -H -f "Sample\ Name" --max 1 |
    mlr --itsv --ocsv cat \
    > source.csv

anchr ena info | perl - -v source.csv > ena_info.yml
anchr ena prep | perl - ena_info.yml

mlr --icsv --omd cat ena_info.csv

aria2c -j 4 -x 4 -s 2 --file-allocation=none -c -i ena_info.ftp.txt

md5sum --check ena_info.md5.txt
```

```bash

```
