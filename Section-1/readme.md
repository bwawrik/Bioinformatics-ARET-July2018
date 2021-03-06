## Section 1: Basic Sequence Handling, QC, and Sequence Assembly

### Downloading and launching a docker file

Lets start by getting all the bioinforamtics software you'll need.  Open your powershell and enter:

```sh
docker pull bwawrik/bioinformatics:latest
```

This pulls my bioinformatics docker.  It may take a few minutes since the docker is over six gigabytes in size.

You will also need a data directory to work in.  Powershell should open in your home directory, but just in case, go to your home directory by typing:

```sh
cd ~
```

Now create a data directory.  You can give it any name you wish; just keep track:

```sh
mkdir /docker_data/data/
```

#### In Docker Toolbox
```sh
cd .. # Use ../ as many times as it takes to get to your C:/ directory
# For example, if your root is C:/Users/username, you would cd ../.. to get back to C:/
cd "program files"
cd "docker toolbox"
```

Start your docker by typing:

```sh
cd /docker_data/data
docker run -t -i -v c:/docker_data/data/:/data bwawrik/bioinformatics:latest
```

#### In Docker Toolbox
```sh
cd data/c/Users/your user profile name/your directory name 
#The *U* in User must be capitalized
```

Congratulations!! You are now running my bioinformatics docker! Perform all your analyses in the `/data` directory. When you exit the docker your files will be in `~/data` and accessible to windows.

```sh
NOTE: IF YOU ARE HAVING A PERMISSION ERROR AT THIS STAGE -- HERE IS HOW TO FIX IT:
http://peterjohnlightfoot.com/docker-for-windows-on-hyper-v-fix-the-host-volume-sharing-issue/
```

Now lets find ourselves some data.  For this we will need to go to the NCBI short read archive:

https://www.ncbi.nlm.nih.gov/sra

Select the advanced search:

![SRA search](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/sra_advanced_search_Step_1.PNG)

Lets search for some single cell genomics data from the oceans by using the search terms 'single amplified genome', and 'marine':

![search advanced](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/sra_search_step_2.PNG)

This will give us some options.  I will select entry 13 for demonstration, but for the purpose of the exercise, I'd like everyone to pick a different one.

[Genome sequencing of marine microorganism: single cell AG-341-O18](https://www.ncbi.nlm.nih.gov/sra/SRX3972923[accn])
1 ILLUMINA (NextSeq 500) run: 4.8M spots, 1.3G bases, 650.8Mb downloads
Accession: SRX3972923

Some other data sets from the same project can be found here:

https://trace.ncbi.nlm.nih.gov/Traces/study/?acc=SRP141175

Using the NCBI website is far form intuitive, so don't get discouraged.  Even I will frequently scratch my head why the site just sent me in circles ! Be patient, and you'll find what you need.

In order to download data, you will need to use fastq-dump:
https://ncbi.github.io/sra-tools/fastq-dump.html

For example:

```sh
fastq-dump -I --split-files SRX3973296 --gzip -X 400000
```
Where SRX3973296 represents the accession number of the data set we are trying to download.  Be careful ! A RUN may contain several EXPERIMENTS and a BIOSAMPLE or BIOPROJECT may contain several RUNS, so you could end up pulling a lot of data. In this case, we are using the -X flag to give ourselves about 400000 paired reads to work with.  

the "--gzip" portion of the command means that the file will be downloaded in compressed format.  

Check your folder contents:

```sh
ls
```

You should get an output something like this:

```sh
SRX3973296_1.fastq.gz SRX3973296_2.fastq.gz
```

```sh
NOTE: IF YOUR OUTPUT FILES ARE NOT SHOWING UP IN '~/data' AS EXPECTED -- TRY THIS:

cd data/docker_data/data
Now re-run the 'fastq-dump' code.
```

Lets uncompress the files:

```sh
gunzip *.gz
```
Your directory content should now be this:

```sh
SRX3973296_1.fastq SRX3973296_2.fastq
```
## ASSESSING THE QUALITY OF THE DATA

First, lets run the QC profiler FASTQC on your data:

```sh
fastqc SRX3973296_1.fastq 
fastqc SRX3973296_2.fastq
```

This should create two HTML files you can view in Firefox by locating your data/ directory with you web browser. I will discuss the output in class, but here is an example of the qScore and adapter plots:

![qscore](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/SRX3973273_1_per_base_quality.png)

![adapters](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/SRX3973273_1_adapter_content.png)


As you can see, there are no illumina adapters in the data -- good, they did a good job removing them.  However, there is a fair amount of data that is below a threshold of Q30.  We will remove this from the data before proceeding.


If your data does not contain any illumina adapter contamination, download my two example data files, run fastqc on them and view the output:


```sh
wget https://github.com/OUGenomics/Bioinformatics-ARET-July2018/raw/master/sample_seqs/diox_f_50000.fastq
wget https://github.com/OUGenomics/Bioinformatics-ARET-July2018/raw/master/sample_seqs/diox_r_50000.fastq
fastqc diox_f_50000.fastq
fastqc diox_r_50000.fastq
```

View the data in your web browser.  As you can see, there is significant Illumina adapter contamination in this data set.

![trueseq contamination](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/trueseq_adapter_contamination.PNG)

### REMOVING ILLUMINA ADAPTERS (IF PRESENT)

The program we will use for this is called cutadapt.  Documentation can be found here:

http://cutadapt.readthedocs.io/en/stable/index.html

Lets clean some of this data up. The first thing you will need to do is to find the adapters sequence in the fastqc report. Open the html for the fastqc report in a web browser.  Then grab each of the over-represented sequences and put them in a notepad text file.  The final file, should look something like this:

```sh
-b GATCGGAAGAGCACACGTCTGAACTCCAGTCACACCACTGTATCTCGTAT -b AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -b NNNNNNN -e 0.15 -q 20 -n 1
```
Where each line that starts with a '-b' represents an adapter sequence.  In the above example, this was adapter 'GATCGGAAGAGCACACGTCTGAACTCCAGTCACACCACTGTATCTCGTAT', which I have added as the first line.  I've added another common adapter for good measure, so you can see that it is possible to use multiple adapters in the same command.  The line '-b NNNNNNN' trims poor quality sequences.  We also add the -e -q and -n parameters to constrain our trimming

Now, go back to powersheell and type:

```sh
cutadapt -b GATCGGAAGAGCACACGTCTGAACTCCAGTCACACCACTGTATCTCGTAT -b AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -b NNNNNNN -e 0.15 -q 20 -n 1 -o diox_r_cutadapt.fastq diox_r_50000.fastq
```

Do the same for the reverse reads and run fastqc on them.

```sh
cutadapt -b GATCGGAAGAGCACACGTCTGAACTCCAGTCACACCACTGTATCTCGTAT -b AGATCGGAAGAGCGTCGTGTAGGGAAAGAGTGT -b NNNNNNN -e 0.15 -q 20 -n 1 -o diox_f_cutadapt.fastq diox_f_50000.fastq
fastqc diox_f_cutadapt.fastq
fastqc diox_r_cutadapt.fastq
```

My directory contents look like this, at this juncture:


```sh
[root@688409c9448f data]# ls
SRX3973296_1.fastq        SRX3973296_2_fastqc.zip   diox_f_cutadapt_fastqc.html  diox_r_cutadapt.fastq
SRX3973296_1_fastqc.html  diox_f_50000.fastq        diox_f_cutadapt_fastqc.zip   diox_r_cutadapt_fastqc.html
SRX3973296_1_fastqc.zip   diox_f_50000_fastqc.html  diox_r_50000.fastq           diox_r_cutadapt_fastqc.zip
SRX3973296_2.fastq        diox_f_50000_fastqc.zip   diox_r_50000_fastqc.html
SRX3973296_2_fastqc.html  diox_f_cutadapt.fastq     diox_r_50000_fastqc.zip
```

Check out the fastqc report. You'll notice that the adapter is no longer in diox_r_cutadapt.fastq or diox_f_cutadapt.fastqc.  However, we still ahve some crummy reads in the dataset that need trimming.  You can see this in the per base quality pane:

![per base quality](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/per_base_quality.PNG)

### Lets trim reads to remove poor quality data

We will use a set of software calle HomerTools.  The documentatoin can be found here:

http://homer.ucsd.edu/homer/ngs/index.html

```sh
read_fastq -e base_33 -i diox_r_cutadapt.fastq | trim_seq -m 30 -l 8 --trim=right | write_fastq -o diox_r_cutadapt.q30.fastq -x
read_fastq -e base_33 -i diox_f_cutadapt.fastq | trim_seq -m 30 -l 8 --trim=right | write_fastq -o diox_f_cutadapt.q30.fastq -x
fastqc diox_f_cutadapt.q30.fastq
fastqc diox_r_cutadapt.q30.fastq
```

Repeat these steps with the data you downloaded from the NCBI SRA database. In my case, I didn't notice any adapters, so I can simply run

```sh
read_fastq -e base_33 -i SRX3973296_1.fastq | trim_seq -m 30 -l 8 --trim=right | write_fastq -o SRX3973296_1.q30.fastq -x
read_fastq -e base_33 -i SRX3973296_2.fastq | trim_seq -m 30 -l 8 --trim=right | write_fastq -o SRX3973296_2.q30.fastq -x
fastqc SRX3973296_1.q30.fastq 
fastqc SRX3973296_2.q30.fastq 
```

!!CONGRATULATIONS. YOU NOW HAVE HIGH QUALITY SEQUENCE DATA TO START DOING SCIENCE :)


## RUNNING ASSEMBLIES

Lets benchmark assembly at several different kmer setting for your data to decide which one we should use.  First, make a small sub-set of your data that contains about 10% of the reads.

```sh
head SRX3973296_1.q30.fastq -n 80000 > fs.fastq
head SRX3973296_2.q30.fastq -n 80000 > rs.fastq
```
This should give you about 40,000 paired reads. These are partial files to allow the assembly to complete in a reasonable amount of time. Together the files contain about 4*10^6 bp of sequence (assuming an average read length of ~100bp), which is about 1x coverage of a typical bacterial genome.  For a good assembly of a whole genome, you would typically aim for 100-200X coverage, but even 50X will yield a decent assembly.

### Ray Assembly

*Brief [description](http://denovoassembler.sourceforge.net/index.html) of Ray:*

> Ray is a parallel software that computes de novo genome assemblies with next-generation sequencing data.  Ray is written in C++ and can run in parallel on numerous interconnected computers using the message-passing interface (MPI) standard.

Run a [Ray](http://denovoassembler.sourceforge.net/manual.html) assembly with a [k-mer](https://en.wikipedia.org/wiki/K-mer) setting of 31 as follows
  
```sh
Ray -k 31 -n 4 -p fs.fastq rs.fastq -o ray_31/
```
If you want to do this with multiple cores, control the number of cores with the -n flag (this will depend on how many cores you have assigned using docker).

Repeat this for kmer settings of 15, 21, and 27.  You can try others, but the number needs to be odd.

NOTE: If you are having trouble seeing the contents of the Ray output folder in your web browser you need to give permission:

```sh
chmod 777 ray_31/
```

Download the [N50](https://en.wikipedia.org/wiki/N50_statistic) perl script
 
```sh
wget https://github.com/bwawrik/MBIO5810/raw/master/perl_scripts/N50.pl
```

Then assess the N50 stats on both assemblies.

```sh
perl N50.pl ray_31/Contigs.fasta
```

Do the same for the other kmer settings.

My output is as follows:

```sh
kmer = 15
Total length of sequence:       112251 bp
Total number of sequences:      361
Number of contigs > 1kb:        15
N25 stats:                      25% of total sequence length is contained in the 6 sequences >= 3368 bp
N50 stats:                      50% of total sequence length is contained in the 30 sequences >= 520 bp
N75 stats:                      75% of total sequence length is contained in the 140 sequences >= 172 bp
Total GC count:                 38191 bp
GC %:                           34.02 %

kmer = 25
Total length of sequence:       107959 bp
Total number of sequences:      400
Number of contigs > 1kb:        13
N25 stats:                      25% of total sequence length is contained in the 7 sequences >= 1767 bp
N50 stats:                      50% of total sequence length is contained in the 49 sequences >= 314 bp
N75 stats:                      75% of total sequence length is contained in the 178 sequences >= 156 bp
Total GC count:                 36696 bp
GC %:                           33.99 %

kmer = 31
Total length of sequence:       82607 bp
Total number of sequences:      263
Number of contigs > 1kb:        11
N25 stats:                      25% of total sequence length is contained in the 9 sequences >= 1382 bp
N50 stats:                      50% of total sequence length is contained in the 41 sequences >= 398 bp
N75 stats:                      75% of total sequence length is contained in the 122 sequences >= 187 bp
Total GC count:                 28229 bp
GC %:                           34.17 %


```

### Self-Examination
Which assembly is best ? which kmer setting should you use ? Why ?


### Now Lets assemble a larger portion of your data

Now assemble your whole QCed dataset using the Kmer you deem optimal. We will only keep contigs longer than 500bp for this -- like so:

```sh
Ray -k 25 -n 4 -minimum-contig-length 500 -p SRX3973296_1.q30.fastq SRX3973296_2.q30.fastq -o ray_SRX3973296_k25/
```

Look at your assemlby stats:

```sh
perl N50.pl ray_SRX3973296_k25/Contigs.fasta
```

## RAST

Last but not least, upload your Contigs.fasta to RAST.

http://http://rast.nmpdr.org/

Log in and choose "upload new job":

![upload new](https://github.com/OUGenomics/Bioinformatics-ARET-July2018/blob/master/images/rast_new.png)



