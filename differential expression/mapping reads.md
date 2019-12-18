# Mapping reads
   
   
### 1. Create a genome index file
 Indices allow the bowtie aligner to narrow down the potential origin of a query sequence within the genome, saving both time and memory.

```ruby
module load GCC/7.3.0-2.30  OpenMPI/3.1.1
module load Bowtie2/2.3.4.2
bowtie2-build S_lycopersicum_chromosomes.fa S_lycopersicum_chromosomes

```
* Four index files were created with the neame *GCA_003112345.1_ASM311234v1_genomic.1.bt2 .....GCA_003112345.1_ASM311234v1_genomic.4.bt2* in */mnt/home/ranawee1/01_A_annua_trichome/differential_expression*

### 2. Read qulity check before trimming
It is essential to see the qulity of RNA seq files. 
```
/mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/do_all_fast_QC_nontrimmed.py
```
This script requries the working directory and print the output to file name you like by doing **<- file name** 
```ruby
import os, sys
path = sys.argv[1]
os.chdir(path)
for root, dirs, files in os.walk(path):
        for f in files:
                if f.endswith(".fastq"): 
                    print("fastqc -o /mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/fastQC_before_trimming/ -f fastq %s" %(f))
                    
```
Then,
```
python qsub_slurm.py -f submit -c do_all_fast_QC_nontrimmed_cmd.txt -p 4 -u ranawee1 -w 1200  -m 10 -mo 'FastQC' -wd ./
```

### 3. Trimming the reads
Data trimming is a essential  step in analysing RNA seq. 
We need to remove any adapter sequnces that might be present in the RNA sequnce reads.
Also, we are pooling out the low quality bases from the sequnce reads.
```
/mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/do_all_trimming.py 
```
This script requries the working directory and print the output to file name you like by doing **<- file name** 
```ruby
import os, sys
path = sys.argv[1]
os.chdir(path)
for root, dirs, files in os.walk(path):
    for f in files:
        if f.endswith("_1.fastq"): 
            print("java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.38.jar PE -threads 4", f, f.replace("_1.fastq", "_2.fastq"), f+".trimP", f+".trimU", f.replace("_1.fastq", "_2.fastq")+".trimP", f.replace("_1.fastq", "_2.fastq")+".trimU", "ILLUMINACLIP:all_PE_adapters.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20")
```
* This creates set of outputs like following
```
java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.38.jar PE -threads 4 SRR2239886_1.fastq SRR2239886_2.fastq SRR2239886_1.fastq.trimP SRR2239886_1.fastq.trimU SRR2239886_2.fastq.trimP SRR2239886_2.fastq.trimU ILLUMINACLIP:all_PE_adapters.fa:2:30:10 LEADING:3 TRAILING:3 SLIDINGWINDOW:4:20
```
```diff
- latform and the path - <java -jar $EBROOTTRIMMOMATIC/trimmomatic-0.38.jar>
+ specify the inputs and outputs - <input file_forfard><input_file_reverse><Output_file_forward_paired><Output_file_forward_unpaired><Output_file_reverse_paired><Output_file_reverse_unpaired> 
! ILLUMINACLIP:<fasta_WithAdapters>:<seed mismatches>:<palindrome clip threshold>:<simple clip threshold>
! ILLUMINACLIP:all_PE_adapters.fa:2:30:10   
!   ILLUMINACLIP -  This step is used to find and remove Illumina adapters.
!   all_PE_adapters.fa -  The file containing the adapter sequnces
!   Seed_Mismatches - specifies the maximum mismatch count which will still allow a full match to be performed
!   Palindrome_ClipThreshold: specifies how accurate the match between the two 'adapter ligated' reads must be for PE             palindrome read alignment.
!   Simple_Clip_Threshold: specifies how accurate the match between any adapter etc. sequence must be against a read.
! SLIDINGWINDOW:<windowSize>:<requiredQuality>
! SLIDINGWINDOW:4:20
!   Window_Size: specifies the number of bases to average across
!   Required_Quality: specifies the average quality required.
```

Then,
```
python qsub_slurm.py -f submit -c do_all_trimming_cmd.txt -p 4 -u ranawee1 -w 1200  -m 10 -mo 'Trimmomatic/0.38-Java-1.8.0_162' -wd ./
```
* The trimming results in four output files:
  * paired forward and revers file - Both reads survived the procesing
  * unpaired forward and revers - Both reads did not survive the filtering

### 4. Read qulity check after trimming

We need to chek the read qulity of all the paired and non paired files
```
/mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/do_all_fast_QC_trimmedP.py
/mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/do_all_fast_QC_trimmedU.py

```
This script requries the working directory and print the output to file name you like by doing **<- file name** 
```ruby
import os, sys
path = sys.argv[1]
os.chdir(path)
for root, dirs, files in os.walk(path):
        for f in files:
                if f.endswith(".trimP"): 
                    print("fastqc -o /mnt/home/ranawee1/01_Solanum_lycopercicum_trichome/difrential_expression/fastQC_after_trimming/ -f fastq %s" %(f))
```

Then,
```
python qsub_slurm.py -f submit -c fastQC_trimmed_cmd.txt -p 4 -u ranawee1 -w 1200  -m 10 -mo 'FastQC' -wd ./
```
We did this for both paired and non-paired files

### Mapping reads to genome
