# ResBaz2016

IMPORTANT:
THIS TUTORIAL WILL NOT RUN WITH LESS THAN 4GB OF RAM.
WE WILL BE RUNNING THIS ON MY VM AT MGMIC.OSCER.OU.EDU. 
IF YOU TRY TO REPRODUCE THIS ON ANOTHER MACHINE, MAKE SURE
IT HAS THE APPROPRIATE MEMORY SIZE. 
IT MAY BE POSSIBLE FOR YOU TO REPRODUCE THIS BY INSTALLING
Boot2Docker AND TO RUN THIS TUTORIAL LOCALLY i.e. ON YOUR LAPTOP.
IF YOU ARE USING A LINUX MACHINE, MAKE SURE TO INSTALL DOCKER FIRST.

--------
To get started, open a terminal window and ssh into my VM. Use your appropriate login info.
For the purpose of this tutorial, I'm assuming you are 'student01'.

```sh
ssh -l student01 mgmic.oscer.ou.edu
```
Enter the password I provide during the workshop.

Running a docker repo usually requires that you first download the docker file to your server.  I have already done this for you, so you don't need to worry about this.  However, if you are using Boot2Docker or your own linux machine, you may pull the docker with the following command (do not do this now):

>docker pull bwawrik/qiime:latest

You will now need to launch the docker and mount you data directories.  You can do this by simply typing 

```sh
sh qiime.sh
```
The shell script executes the following docker command:

>docker run -t -i -v /data/static/sequence_data/RESBAZ/student01:/data -v /data/static/sequence_data/RESBAZ/DATABASES:/data/DATABASES bwawrik/qiime:latest

The command mounts your data directory as well as the SILVA database (we'll need this below).

Navigate to your data directory

```sh
cd /data
```

Deploy usearch version 5.2.236 and 6.1.544. Qiime does not use the latest version of usearch and will throw an error if you try. Since this software has to be licensed, I cannot include it in the docker, which is in a public repository.  Run the following commands to install usearch licensed to the Wawrik lab. Please keep in mind that you will need to get your own license (free), if you are going to use my docker beyond the tutorial described here.

```sh
wget http://mgmic.oscer.ou.edu/sequence_data/tutorials/install_usearch.sh
sh install_usearch.sh
```
You can check if usearch is installed correctly by simply typing
>usearch

The install_usearch.sh shell script contains several commands, which you could run manually as follows (do NOT do this now)

```sh
mkdir -p /opt/local/software/usearch cd /opt/local/software/usearch
wget http://mgmic.oscer.ou.edu/sequence_data/tutorials/usearch5.2.236_i86linux32
wget http://mgmic.oscer.ou.edu/sequence_data/tutorials/usearch6.1.544_i86linux32
chmod 777 * 
cd /usr/local/bin 
ln -s /opt/local/software/usearch/usearch5.2.236_i86linux32 ./usearch 
ln -s /opt/local/software/usearch/usearch6.1.544_i86linux32 ./usearch61
```

Now change your directory to /data, dowload the tutorial sample data & mapping file, and unzip the files

```sh
cd /data
wget https://github.com/bwawrik/MBIO5810/raw/master/sequence_data/GOM_R1.fastq.gz
wget https://github.com/bwawrik/MBIO5810/raw/master/sequence_data/GOM_R2.fastq.gz
wget https://github.com/bwawrik/MBIO5810/raw/master/sequence_data/GoM_Sept_Mapping.txt
gunzip *.gz
```

Look at the mapping file

```sh
cat GoM_Sept_Mapping.txt
```

the file should be in the following format:
```sh
#SampleID    BarcodeSequence    LinkerPrimerSequence    Area    Description
#Descriptive string identifying the sample              
sample1   CAACACCA    CAAGAGTTTGATCCTGGCTCAG    tag   none
sample2    CAACACGT    CAAGAGTTTGATCCTGGCTCAG    tag    none
sample3    CAACAGCT    CAAGAGTTTGATCCTGGCTCAG    tag    none
```

Now Check the mappigng file for errors

```sh
validate_mapping_file.py -m GoM_Sept_Mapping.txt -o mapping_output -v
cat mapping_output/GoM_Sept_Mapping.log
```

- You should get an output saying that :

>No errors or warnings found in mapping file

- Join the fastq files to stich reads together (Note: p: percent maximum difference; m: minimum overlap; o: output directory)

```sh
fastq-join  GOM_R1.fastq  GOM_R2.fastq -p 3 -m 50 -o GoM_16S_Sept.fastq
mv  GoM_16S_Sept.fastqjoin  GoM_16S_Sept.fastq
```
Do you understand the output of the fastq-join command ?

Extract your reads and barcodes

```sh
extract_barcodes.py -f GoM_16S_Sept.fastq -m GoM_Sept_Mapping.txt --attempt_read_reorientation -l 12 -o processed_seqs
```

Split your libraries based on the barcodes that are found in the sequences

```sh
split_libraries_fastq.py -i processed_seqs/reads.fastq -b processed_seqs/barcodes.fastq -m  GoM_Sept_Mapping.txt -o processed_seqs/Split_Output/ --barcode_type 12
```

### DEFAULT QIIME ANALYSIS USING GREENGENES

Pick your OTUs using default parameters

```sh
pick_open_reference_otus.py -i processed_seqs/Split_Output/seqs.fna -o OTUs
```

Inspect the BIOM file

```sh
biom summarize-table -i OTUs/otu_table_mc2_w_tax_no_pynast_failures.biom
```
Do you understand the output ?

Run QIIME core diversity analysis

```sh
core_diversity_analyses.py -o cdout/ -i  OTUs/otu_table_mc2_w_tax_no_pynast_failures.biom -m GoM_Sept_Mapping.txt -t OTUs/rep_set.tre -e 20
```

To view your output, open a web browser and enter the following URL:

```sh
http://mgmic.oscer.ou.edu/sequence_data/RESBAZ/student01/cdout/
```

### QIIME ANALYSIS USING SILVA 111

The commands above use a closed reference OTU picking approach with a pre-deployed version of Greengenes at 90% identity. This is not always optimal.  For amplicon, I prefer open reference OTU picking with Silva 111 (or later).  

The following commands allow you to install  Silva111.  I have already done this for you under /data/static/sequence_data/ResBaz/DATABASES so you will not need to execute the following commands

```sh 
mkdir -p /data/DATABASES/16S
cd /data/DATABASES/16S
wget http://www.arb-silva.de/fileadmin/silva_databases/qiime/Silva_111_release.tgz
tar -xvf Silva_111_release.tgz
cd Silva_111_post/rep_set_aligned
gunzip *
cd ..
cd rep_set
gunzip *
```

Download the parameters file needed for running the analysis using Silva

```sh
wget https://github.com/bwawrik/MBIO5810/raw/master/sequence_data/qiime_parameters_silva111.par
```

Pick your OTUs

```sh
pick_de_novo_otus.py -i processed_seqs/Split_Output/seqs.fna -o OTUs_silva -p qiime_parameters_silva111.par
```

Inspect the BIOM file

```sh
biom summarize-table -i OTUs_silva/otu_table.biom 
``` 

Look at the number of sequences in each sample.  In the next command you need to set the '-e' parameter, which is the sampling depth for rarefaction.  'e' should not exceed the lowest number in the result form this command.

Run QIIME core diversity analysis

```sh
core_diversity_analyses.py -o cdout_silva/ -i  OTUs_silva/otu_table.biom -m GoM_Sept_Mapping.txt -t OTUs_silva/rep_set.tre -e 20
```

To view your output, open a web browser and enter the following URL:

```sh
http://mgmic.oscer.ou.edu/sequence_data/RESBAZ/student01/cdout_silva/
```


Which analysis was faster ? Which produces better results ? Why ?
