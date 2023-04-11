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
