# Readme OrthoSLC (0.1Beta)

**OrthoSLC** is a pipline that perfomrs Reciprocal Best Blast Hit (RBBH) Single Linkage Clustering to obtain Orthologous Genes. <br>

It is: <br>
* lightweight, convenient to install
* **independent** of relational database management systems (e.g., MySQL)
* able to handle more than 1000 genomes.

The pipeline start with annotated genomes, and can produce clusters of gene id, and FASTA files of each cluster.

Note that, pipeline is recommended for sub-species level single copy core genome construction since RBBH may not work well for missions like Human-Microbe core genome construction. 

**Caveat:**<br>
The pipeline is currently available for linux-like system only and have been tested on Ubuntu 20.04 and 18.04.

**Requirement:**<br>
* Python3 (suggest newest stable release or higher),<br>
* C++17 ("must" or higher for compiling) users may also directly use pre-compiled binary files.<br>
or use `install.sh` like following:

```Shell
$ chmod a+x install.sh
$ install.sh /path/to/directory/of/all/src_files \
    /path/to/store/compiled/binary_files
```
* NCBI Blast+ (suggest 2.12 or higher) <br>

Besides callable binary files, we also provide an "all-in-one" Jupyter notebook interface `OrthoSLC_Python_jupyter_interface.ipynb`. It is s relatively slow but fits small sclae analysis and allow users to do customized analysis and modification in between pipeline steps. However, for datasets with 500 or more genomes, we still recommend using the binary files, which is mainly written in C++, for optimal performance.

The programs uses [A simple C++ Thread Pool implementation](https://github.com/progschj/ThreadPool), and sincere thanks to [its contributors](https://github.com/progschj/ThreadPool/graphs/contributors).

**Version change**:<br>
Comparing with `0.1Alpha`, current version `0.1Beta` allow users to set `bin level`, in Step 5 and Step 6, with a linear style instead of exponatial style.

**Note:**<br>
For all steps, users do not need to make the ourput directory manually, program will do that for you.

Bug report: 
* <jingjie.chencharly@gmail.com>

## Step 1 Genome information preparation
The pipeline starts with annotated genomes in fasta format. FASTA file name require strain name and extension (e.g., `strain_A.ffn`, `strain_B.fna`, `strain_C.fasta` etc.).<br> 
Step 1 needs the **path to directory of annotated FASTA files** as input, to genereate a header less, tab separated table, in which the 
* first column is a short ID, 
* second column is the strain name, 
* third column as the absolute path. 

Users can run callable binary file `Step1_preparation` and specifying parameters like following:

```shell
Usage: Step1_preparation -i input/ -o output.txt

  -i or --input_path -------> path/to/input/directory
  -o or --output_path ------> path/to/output.tsv
  -h or --help -------------> display this information
 ```

The short ID of each genome is generated to save computational resources and storage space. Since reciprocal BLAST generates a large volume of files (millions to billions of rows if large number of genomes participated), each row contains the names of the query and subject. If the user provides input FASTA file names like:

* `GCA_900627445.1_PFR31F05_genomic.fasta`
* `GCA_021980615.1_PDT001237823.1_genomic.fasta`

and such file names become part of gene identifier instead of the short ID used in this program, additional 30 ~ 60 GB of storage will be consumed for intermediate files and even more pressure on computing memory for analysis of 1000~ genomes.

## Step 2 FASTA dereplication
Step 2 is to remove potential sequence duplication (e.g., copies of tRNA, some cds). This dereplication is equivalent to 100% clustering, to obtain single copy.<br>
Step 2 **requires the tab separated table output by Step 1 as input**, and specifying a directory for dereplicated files.

**Note**:<br>
For users who are **100% sure** that your annotated files are already single copy, this step may be skipped.

You may run `Step2_simple_derep` like this:

```Shell
Usage: Step2_simple_derep -i input_file -o output/ [options...]

  -i or --input_path -------> path/to/file/output/by/Step1
  -o or --output_path ------> path/to/output/directory
  -u or --thread_number ----> thread number, default: 1
  -h or --help -------------> display this information
```

### <font color="red">Note before next step</font>
After dereplication, users should give a careful check of **size** of dereplicated FASTA files. It is worth noting that if a FASTA file with a very low amount of sequences, getting processed together with all the rest, the final "core" clusters will be heavily affected and may bias your analysis.
    
Since the core genome construction is similar with intersection construction. <font color="red">**It is recommend to remove some very small dereplicated fasta files BEFORE NEXT STEP**, e.g., remove all dereplicated <i>E.coli </i> genomes with file size lower than 2.5MB as most should be larger than 4.0MB.</font>

## Step 3 Sequence preparation
This step requires **path to directory of derplicated FASTA** to generate to 2 files:

1. A FASTA file Combined from all dereplicated FASTA files.<br>
2. A tab separated file of sequence length information of all records.

You may run `Step3_seq_preparation` like following:

```Shell
Usage: Step3_seq_preparation -i input/ -c concatenated.fasta -l seq_len.txt

  -i or --input_path ---------> path/to/input/directory
  -c or --concatenated_fasta -> path/to/output/concatenated.fasta
  -l or --seq_len_tbl --------> path/to/output/sequence_length_table
  -h or --help ---------------> display this information
```

## Step 4 Reciprocal BLAST
Step 4 will carry out the Reciprocal Blast using NCBI Blast. You can get it from [NCBI official](https://ftp.ncbi.nlm.nih.gov/blast/executables/blast+/LATEST/).
 
The pipeline will assist you to:<br>
1. Create databases for each of all dereplicated genomes using `makeblastdb`.
2. Using `blastn` or `blastp` to align the `concatenated.fasta` against each of the database just made and get tabular output.

To create database to BLAST, users should provide **path to directory where all dereplicated FASTA is**, and a **path to output directory where BLAST database is to store**.<br>
You may run `Step4_makeblastdb.py` like following:

```Shell
Usage: python Step4_makeblastdb.py -i input/ -o output/ [options...]

options:

 -i or --input_path ----------> path/to/input/directory
 -o or --output_path ---------> path/to/output/directory
 -c or --path_to_makeblastdb -> path/to/output/makeblastdb, default: makeblastdb
 -u or --thread_number -------> thread number, default: 1
 -t or --dbtype --------------> -dbtype <String, 'nucl', 'prot'>, default: nucl
 -h or --help ----------------> display this information
```

<font color="red">**Note before start**</font>: The BLAST step may take a very long time to run (1100 <i>E. coli</i> genomes will roughly take 2.3 days using 70 cores). Be prepared, use "`screen`" or "`&&`" to prevent accidental termination, and try to do it on a large server.


To perform reciprocal BLAST, users should provide **path to concatenated FASTA producd by step 3**, **path to directory where databases made by `Step4_makeblastdb.py`**, and a **path to output directory where BLAST tabular output** is to store.<br>
You can run `Step4_reciprocal_blast.py` like following:

```Shell
Usage: python Step4_reciprocal_blast.py -i query.fasta -o output/ -d directory_of_dbs/ [options...]

options:

 -i or --query ---------------> path/to/concatenated.fasta
 -d or --dir_to_dbs ----------> path/to/directory/of/dbs
 -o or --output_path ---------> path/to/output/directory
 -c or --path_to_blast -------> path/to/output/blastn or blastp, default: 'blastn'
 -e or --e_value -------------> blast E value, default: 1e-5
 -u or --blast_thread_num ----> blast thread number, default: 1
 -h or --help ----------------> display this information
```

In case you have installed your blast but not exported to `$PATH`, you can simply input `whereis blastn` or `whereis makeblastdb` to get the full path to your blast binary file.<br>

The reason to BLAST against each database sequentially rather than directly using an all-vs-all approach is to 
reduce computational overhead. This can be very useful if the task involves many genomes. For example, if you have 1000 dereplicated genomes to analyze, the total size of concatenated FASTA may reach 5-10 GB. A multi-threaded BLAST job using the `-mt_mode 1` by all-vs-all style could be too memory-intensive to run for such a large dataset.

In addition, sequentially running BLAST will produce one tabular output per database. This will be a better adaptation for the job parallelization of finding reciprocal best hits in later steps, which will apply the hash binning method.

## Step 5 filtering and hash binning
This step is to filter the blast output and to apply hash binning, in order to provide best preparation for reciprocal best find.<br>

Step 5 requires **path to directory of BLAST tabular output as input**, **sequence length information output by Step 3**.

you may run Step5_filter_n_bin like following:
```Shell
Usage: Step5_filter_n_bin -i input/ -o output/ -s seq_len_info.txt [options...]

  -i or --input_path -------> path/to/input/directory
  -o or --output_path ------> path/to/output/directory
  -s or --seq_len_path -----> path/to/output/seq_len_info.txt
  -u or --thread_number ----> thread number, default: 1
  -L or --bin_level --------> binning level, an intger 0 < L <= 9999 , default: 10
  -r or --length_limit -----> length difference limit, default: 0.3
  -h or --help -------------> display this information
```

The pipeline will carry out following treatment to BLAST output:
1. Paralog removal: <br>
If query and subject is from same strain, the hit will be skipped, as to remove paralog.
2. Length ratio filtering:<br>
Within a hit, query length $Q$ and subject length $S$, the ratio $v$ of this 2 length

$$v = \frac{Q}{S}$$

should be within a range $r$, according to [L. Salichos et al](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0018755), $r$ is recommended to be higher than 0.3 which means the shorter sequence should not be shorter than 30% of the longer sequence:

$$r < v < \frac{1}{r}$$<br>

If above condition not met, the hit will be removed from analysis.

3. Non-best-hit removal: <br>
If a query has more than 1 subject hits, only the query-subject pair with highest score will then be kept.
4. Sorting and binning:<br>
For every kept hit, its query and subject will be sorted using Python or C++ built in sort algorithm. This is because in a sequential blast output file, only "**single direction best hit**" can be obtained, its "**reciprocal best hit**" only exist in other files, which poses difficulty doing "**repriprocal finding**". <br>
However, if a query $a$ and its best suject hit $b$, passed filter above, and form $(a, b)$, and in the mean time we sort its rericprocal hit $(b, a)$ from another file into $(a, b)$, then both $(a, b)$ will generate same hash value. This hashed value  will allow us to bin them into same new file. Therefore, after this binning, "**reciprocal finding**" will be turned into "**duplication finding**" within one same file.<br>

<font color="red">**Set bin level:**</font><br>
According to the amount of genomes to analysze, user should provide binning level, which is to set how many bins should be used. Level $L$ should be interger of range $0 < L \le 9999$, and will generate $L$ bins. 

Suggestion is that do not set the bin level too high, especially when less than 200 genomes participated. If such amount of genomes participated analysis, bin level from 10 to 100 should work as most efficient way. 

As tested, an analysis of 30 genomes: 
* A bin level of 10, takes 1.35 seconds to finish, 
* a bin level of 100, takes 3.38 seconds to finish,
* a bin level of 1000, takes 17.4 seconds to finish,

<font color="red">**When to set a high bin level:**</font><br>
Simply speaking, when you have really larger amount of genomes and not enough memory (e.g., more than 1000 genomes and less than 64 GB memory) <br>

The output of BLAST for 1000 genomes can reach 200 GB in size, and if the bin level is set to 100, there will be 100 bins to evenly distribute the data. On average, each bin will contain 1.7 GB of data, which may be too memory-intensive to process in step 6, where reciprocal find is performed (which requires approximately 1.7 GB of memory per bin). However, if the number of bins is increased to 1000, the size of each bin will be reduced to between 100-200 MB, which then facilitate step 6 parallelization.

<font color="red">**Note:**</font><br>
This is the one of the most computation and I/O intensive step, use the C++ based binary file to process for better efficiency.

## Step6 Reciprocal Best find
This Step is to find reciprocal best hits. In `Step5`, query-subject pairs had been binned into different files according to their hash value, therefore, pair $(a, b)$ and its reciprocal pair $(b, a)$ (which was sorted into $(a, b)$ ), will be in the same bin. Thus, a pair found twice in a bin will be reported as a reciprocal best blast pair.

In addition, Step 6 also does hash binning after a reciprocal best hit is comfirmed. Query-subject pairs will be binned by the hash value of query ID, which then put pairs with common elements into same bin to assist faster clustering in next step.

Step 6 requires **path to directory of bins output by Step 5**, and path to output directory.

you may run `Step6_RBF`like following:<br>

```Shell
Usage: Step6_RBF -i input/ -o output/ [options...]

  -i or --input_path -------> path/to/input/directory
  -o or --output_path ------> path/to/output/directory
  -u or --thread_number ----> thread number, default: 1
  -L or --bin_level --------> binning level, an intger 0 < L <= 9999 , default: 10
  -h or --help -------------> display this information
```

<font color="red">**Set bin level:**</font><br>
According to the amount of genomes to analysze, user should provide binning level, which is to set how many bins should be used. evel $L$ should be interger of range  $0 < L \le 9999$, and will generate $L$ bins. 

Suggestion is that do not set the bin level too high, especially when less than 200 genomes participated. If such amount of genomes participated analysis, bin level from 10 to 100 should work as most efficient way. 

As tested, an analysis of 30 genomes, and 10 bins generated by Step 5:
* A bin level of 10, takes 0.66 seconds to finish, 
* a bin level of 100, takes 1.35 seconds to finish,
* a bin level of 1000, takes 6.3 seconds to finish,

<font color="red">**When to set a high bin level:**</font><br>
Simply speaking, when you have really larger amount of genomes and not enough memory (e.g., more than 1000 genomes and less than 64 GB memory) <br>

Less bins could make step 6 faster, but step 7 more memory intensive.

<font color="red">**Note:**</font><br>
This is the one of the most computation and I/O intensive step, use the C++ based binary file to process for better efficiency.

## Step7 Single Linkage Clustering
Step 7 will carry out single linkage clustering on output from step6. Users may perform "**multi-step-to-final**" or "**one-step-to-final**" clustering by adjusting the `compression_size` parameter. In the final cluster file, each row is a cluster (stopped by "\n") and each gene ID is separated by "\t".

In case that large amount genomes participated analysis, it could be memory intensive to reach final cluster in a single step. The pipeline provides ability to extenuate such pressure by reaching final cluster with multiple steps.

You may run `Step7_SLC` like this:<br>

```Shell
Usage: Step7_SLC -i input/ -o output/ [options...]

options:
  -i or --input_path -------> path/to/input/directory
  -o or --output_path ------> path/to/output/directory
  -u or --thread_number ----> thread number, default: 1
  -S or --compression_size -> compression size, default: 10, 'all' means one-step-to-final
  -h or --help -------------> display this information
```

For example, if 1,000 files output by `Step6_RBF` and user should attempt "**multi-step-to-final**" like following:<br>

Set `compression_size` as `2`, the pipeline will cluster every 2 files into 1 output, thus, 500 files will present in `path/to/SLC/output1`

```Shell
Step7_SLC \
-i path/to/dirctory/of/Step6_RBF/output \
-o path/to/SLC/output1 \
-u thread number
-S 2
```

So, if `compression_size` set to `10`, 50 files will present in `path/to/SLC/output2`:<br>

```Shell
Step7_SLC \
-i path/to/SLC/output1 \
-o path/to/SLC/output2 \
-u thread number
-S 10
```

Finally, if `compression_size` set to `50` or `all`, the final 1 cluster can can be obtained in `path/to/SLC/output_final`.<br>

```Shell
Step7_SLC \
-i path/to/SLC/output2 \
-o path/to/SLC/output_final \
-u thread number
-S 50
```

However, "**one-step-to-final**" is doable if you have enough memory and few genomes. To achieve this, simply set the `compression_size` equal to `all`.

## Step 8 Write clusters into FASTA
In Step 8, program will help user to generate FASTA file for each cluster. By providing the final one cluster file generated by Step 7 as input, program produces 3 types of clusters into 3 directories separately.<br>

Noteably, those genomes 
* depreplicated in **Step 2**,  
* not removed because of too low genome size
* participated processes up to this step, 

are used to separate 3 types of clusters.<br>

1. In drectory `accessory_cluster`, FASTA files of clusters, which **do not have genes from all genomes** participated analysis, will be output in this drectory. For example, there are 100 genomes in analysis, a cluster with less than 100 genes will have its FASTA output here. Also, if a cluster has >= 100 genes, but all these genes are from less than 100 genomes, its FASTA will be in this directory.
2. In drectory `strict_core`, each cluster has **exactly 1 gene from every genome** to analyze. Such clusters will have their FASTA files here.
3. In drectory `surplus_core`, each cluster has **at least 1 gene from every genome** to analyze, and **some genomes has more than 1 genes** in this cluster. Such clusters will have their FASTA files here.

Users may also specify if all or some of 3 types of clusters to be output. By specifying, `--cluster_type` or `-t` and provide wated type from `accessory/strict/surplus`, and separted by comma. For example, `-t accessory,strict` would produce only directory of `accessory_cluster` and `strict_core`.

Step 8 **requires the concatenated FASTA file output by Step 3**. user also needs to provide the amount of how many genomes participated analysis. User can use `ls /path/to/dereplicated/Step2 | cat -n` to get the number.

You may run `Step8_write_clusters` like this:<br>
```Shell
Usage: Step8_write_clusters -i input_path -o output/ -f concatenated.fasta [options...]

options:
  -i or --input_path ------> path/to/input/cluster_file
  -o or --output_path -----> path/to/output/directory
  -f or --fasta_path ------> path to concatenated FASTA file
  -c or --total_count -----> amonut of genomes to analyze
  -t or --cluster_type ----> select from < accessory / strict / surplus >, separate by ',', all types if not specified
  -u or --thread_number ---> thread number, default: 1
  -h or --help ------------> display this information
```