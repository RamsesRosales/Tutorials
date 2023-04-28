**Install and Run Earl Grey**
===============================
- [Installation](#installation)
    - [Conda Environment](#conda-environment)
    - [Earl Grey installation](#earl-grey-installation)
    - [Earl Grey Configuration](#earl-grey-configuration)
        - [R dependencies](#r-dependencies)
        - [Python modules](#python-modules)
        - [Perl modules](#perl-modules)
- [Example](#example) 

This is a guide on how to install and run [EarlGrey](https://github.com/TobyBaril/EarlGrey).

# **Installation**

I started following [Rhett's](https://github.com/RhettRautsaw) guide on how to install earl grey, make sure to install v 2.0.0, from the release version with a wget instead of git clone, as the latest versions need a different RepeatMasker version that is not available for conda install, alternatively you can try to modify Earl Grey code. Other option you might try is to use the instructions on the [EarlGrey github](https://github.com/TobyBaril/EarlGrey) to install the latest version.

## **Conda Environment**
```
#RepeatModeler Installation
conda create -n repeatModeler mamba
conda activate repeatModeler
mamba install -c bioconda python=3.9 repeatmodeler=2.0.3 ltr_retriever mafft cd-hit ninja genometools-genometools ucsc-fatotwobit ucsc-twobitdup ucsc-twobitinfo ucsc-twobitmask ucsc-twobittofa r
```
## **Earl Grey installation**

```
# EarlGrey Installation
# create repeatmodeler conda environment and then do the following

#If you want to try to modify the latest version of Earl Grey you can use git clone 
git clone https://github.com/TobyBaril/EarlGrey.git

#If you want to be sure you have the right version of RepeatModeler and RepeatMasker in your 
#conda environment use wget to download the right version
# you will need to decompress and untar the file 
wget https://github.com/TobyBaril/EarlGrey/archive/refs/tags/v2.0.tar.gz
pigz -d v2.0.tar.gz
tar xvf v2.0.tar
```
## **Earl Grey Configuration**
```
cd EarlGrey*
chmod -R 755 configure earlGrey scripts/*

# EDIT CONFIGURE FILE 
#	1. Comment out if/else lines associated with creating/updating a new conda environment. 
#	2. Change "conda activate earlGrey" to "conda activate repeatModeler"  --- alternatively, rename your conda environment above as earlGrey
# EDIT EarlGrey
#	1. Comment out if/else lines associate with checking for "earlGrey" conda environment (directly above "# Main #") -- alternatively, rename your conda environment above as earlGrey
#	2. Edit famdb.py call to look in the directory /home/r/rautsaw/.conda/envs/repeatModeler/share/Libraries rather than the conda bin directory
echo "export PATH=\$PATH:$PWD:$PWD/scripts/" >> ~/.bashrc
echo "export PATH=\$PATH:/home/r/rautsaw/.conda/envs/repeatModeler/share/RepeatMasker" >> ~/.bashrc

### if you git cloned EarlGrey you might try to edit Earl Grey with the following command
### did't tryed this so it might lead to further problems of compatibility, but at least 
### it should solve the problem with RepeatModeler

perl -pi -e "s/RepeatModeler -engine ncbi -threads/RepeatModeler -engine ncbi -pa/g" earlGrey

```

then follow configuration steps run

```
./config
```


### **R dependencies**

open R and call the libraries that supposed to be installed, you can check which packages in here

```
cat scripts/install_r_packages.R
```

With install.packages()
- BiocManager
- optparse
- ape

With BiocManager::install()
- plyranges
- BSgenome

```
R
library(<package_name>)
```

if you get an error, that means there was an error in the installation, so install the packages with the following commands, sometimes dependencies installation fails, but works if you install it manually so be aware of errors in dependencies installations, if some of those fails try to directly install that dependency yourself with install.packages(\<dependencie\>)

```
R
install.packages(<package_name>)
BioManager::install(<package_name>)
```

Besides the packages included in the install packages script you will also need the following packages:

With install.packages()
- tidyverse
- plyr

### **Python modules**

python modules are also lacking you can pip install or conda install, you will also need parallel

```
conda activate repeatModeler 
conda install pandas biopython pyfaidx parallel pyranges
```

if you still not get the whole pipeline running you can check in the log file look for any missing library of package

```
grep -C 10 "rror" \<logfile\>
```

in case you need to install other file

### **Perl modules**

The harder issue to solve is that rcMergeRepeatsLoose script does not find the perl modules in RepeatMasker, to solve it you need to go to the location of RepeatMasker in your conda environment, check that with.
```
which RepeatMasker
```
It wil be likely a symbolic link to the script in the location of the full directory with the rest of the scripts.
Once there you need to make symbolic links for the rest of the files ending in .pm (perl module), located in the same path that EarlGrey symbolic link is pointing too.

This first version didn't worked so is here just for recording of the steps I tried
```
cd /home/ramsesr/.conda/envs/repeatModeler/share/RepeatMasker/util
for i in $(ls ../*.pm | perl -p -e "s/\.\.\///g")
do echo $i
ln -s ../$i $i
done
```

This is the right path to generage the links

```
cd /home/ramsesr/.conda/envs/repeatModeler/bin/
for i in $(ls ../share/RepeatMasker/*.pm | perl -p -e 's/.*\/([^\/]*\n)/$1\n/g' )
do echo $i
ln -s ../share/RepeatMasker/$i $i
done

```

# **Example**

Rhett's example on how to run the program, there is example of RepeatModeler and one for EarlGrey

```
# Running RepeatModeler
BuildDatabase -name assembly_db assembly.fasta
RepeatModeler -pa 24 -database assembly_db -LTRStruct
mv RM* 01_RepMod
RepeatMasker -pa 24 -gff -lib 01_RepMod/consensi.fa.classified -dir 02_RepMask assembly.fasta
# Running EarlGrey
# Carefully choose -r parameter because chosing something too broad
# will result in significantly longer runtimes
earlGrey -g assembly.fasta -s assembly -o 03_earlGrey -t 24 -r mammals
```

My example pbs job to run Earl Grey 
```
#PBS -N earl_grey_Clepi-CLP2201
#PBS -l select=1:ncpus=40:mem=1000gb,walltime=168:00:00
#PBS -q bigmem
#PBS -M ramsesr@g.clemson.edu
#PBS -m abe
#PBS -j oe

cd $PBS_O_WORKDIR
module load anaconda3/2022.05-gcc/9.5.0
source activate repeatModeler

cd /zfs/venom/Ramses/HiFi/Clepi-CLP2201/WGS/blood/2021-11-19_Combined

if [ ! -d 04_repeat_EarlGrey ]
    then
        echo making 04_repeat_EarlGrey
        mkdir 04_repeat_EarlGrey
    else
        echo directory exist
fi

cd 04_repeat_EarlGrey

earlGrey -g ../Clepi-CLP2201_*.fasta -s Clepi-CLP2201_WGS_blood -o $PWD -t 40 -r vertebrata
```
- -g paht to fasta file
- -s suffix to add to the results
- -o ourput directory
- -t threads 
- -r database

