# Single cell RNA sequencing analysis

## Preparation

- Anchr - the Assembler of N-free CHRomosomes (author: wang-q)

```bash
cargo install --git https://github.com/wang-q/anchr --branch main
anchr help
```

## Data collection

### Downloading data

`PRJNA577177`

```bash
mkdir -p /mnt/e/scrna/sra
cd /mnt/e/scrna/sra

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
    keep-header -- tsv-sort -k2,2 -k3,3nr |
    tsv-uniq -H -f Sample_Name --max 1 |
    mlr --itsv --ocsv cat \
    > source.csv

anchr ena info | perl - -v source.csv > ena_info.yml
anchr ena prep | perl - ena_info.yml

mlr --icsv --omd cat ena_info.csv

aria2c -j 4 -x 4 -s 2 --file-allocation=none -c -i ena_info.ftp.txt

md5sum --check ena_info.md5.txt
```
