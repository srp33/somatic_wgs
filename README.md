# Docker U54

## Description

This Docker image applies the bwa mem and samtools view commands to next-generation sequencing 
data (FASTQ files) within an isolated environment. It is designed to process data for one 
paired-end sample at a time. It also uses environment variables, which allow you to customize 
settings for these tools (for example, the reference genome that you wish to use or the number 
of threads you use). It assumes you have placed FASTQ files for a biological sample (two FASTQ 
files because they are paired-end reads) in a directory and that you wish to save the output 
file (BAM) for that sample to a different directory.

## The Big Picture

Remember a few things:

  * You must include four volumes using:
    1. `<location of reference genome>:/data/ref_index`
    2. `<location of reads>:/data/sample_data`
    3. `<location of output directory>:/data/results`
    4. `<location of bam files>:/data/bam_files`
    
    Remember that each volume must be preceded by the `-v` flag and follows the format
    
    ```
    <location on host>:<prescribed location in container>
    ```
    
    The `<location on host>` is
    determined by the user, __but the `<prescribed location in container>` must be one of the
    locations described in this document__. Otherwise the container will fail.
    
  * The docker container must have appropriate permissions:
    1. Read permissions for the reference genome directory
       * This directory also requires write permissions if the reference genome has not been
       previously indexed
    2. Read permissions for the reads
    3. Read/Write permissions for the output directory
    
    Permissions are most easily dealt with using the `--user root:root` argument, however, it is up to 
    the user to resolve any issues with permissions.
    
  * Each container command 
  
## Usage

###### Command Reference

```
docker run \
-v <location of reference genome>:/data/ref_index \
-v <location of reads>:/data/sample_data \
-v <location of output directory>:/data/results \
-v <location of bam files>:/data/bam_files \
--user root:root -it [additional options] \
--rm \
srp33/u54:latest \
<command> <args...>
```

The command for this container can seem rather daunting at first:

```{bash}
docker run \
-v /Applications/U54/ref_index:/data/ref_index \ 
-v /Applications/U54/in_use:/data/sample_data \
-v /Applications/U54/OutputData:/data/results \
-v /Applications/U54/bam_files:/data/bam_files \
--user root:root \
--rm \
srp33/bwamtools:latest \
align \
-t 10 \
-r ucsc.hg19.fasta.gz \
-s 101024
```

I will attempt to break this down and make sense of each argument.

* `-v`
  * This flag allows the user to connect a directory or folder to an identical directory or folder
  in the container. There are three important volumes that may be required to use certain commands:
  
    1. `<location of reference genome>:/data/ref_index`
    
       * `<location of reference genome>` should be a directory where the reference genome is
       found. If the genome has not previously been indexed, the container will run `bwa index`.
       This command requires write permissions in the `/data/ref_index` directory on top of read
       permissions. If the genome has been indexed previously, only read permissions are required.
       __It is vital to pass a directory and not just a file into this volume__, if the user
       passes a file instead of a directory into this container, it will attempt to run 
       `bwa index` and fail.
       
       * `/data/ref_index` is where bwamtools expects to find the reference genome. If it is not
       found, the container will fail and exit.
      
    2. `<location of reads>:/data/sample_data`
    
       * `<location of reads>` is where the reads are located. The easiest way to use this volume
       is by putting the desired reads into a directory and simply referencing the directory
       into the volume. If there are multiple samples in the directory, the container will
       dynamically run its commands a sample at a time, outputting as many `.bam` files as there
       are samples. If the user does not want to move the files to their own directory and the user
       does not want this behavior (i.e. the user only wants to run this container on one sample)
       the user may reference the specific files as follows:
       
         `-v <location of reads>/<file name>:/data/sample_data/<file name>`
       
         The file name does not need to coincide between the file found on the host and that on
         the container, however, the container will dynamically name the outputted `.bam` file 
         based on the name of the reads. It should also be noted that if the first part of the file
         (everything before the first `.`) does not coincide between reads of the same sample, they
         will not be analyzed as being part of the same sample. Thus consistency will allow the user
         to avoid many unforeseen issues.
       
    3. `<location of output directory>:/data/results`
      
       * It is vital for the proper function of this container that the user follow two rules
       concerning the output directory:
         1. __The entire directory must be passed in__. If the user attempts to pass in a file,
         the container will not have permissions for writing within the `results` directory.
         Without these permissions, the container will fail and stop.
         2. __The docker container must have permissions for writing within the `results` directory__.
         This may done by either allowing write permissions using `chmod` although this method is not
         recommended. Instead, it is recommended that the user follow the guidelines defined in the
         `--user <user id>` section of this document.
         
    4. `<location of bam files>:/data/bam_files`
    
       * This volume as been added to more clearly define locations of bam files to be used primarily
       by `sambamba`
       
    In general, it is a good idea to just give read/write permissions to all of the volumes and to
    pass in every volume regardless of the command used, even though each command only requires some,
    but not all, of the volumes.

* `--user <user id>`

  * This flag allows the user to pass read, write, and execute permissions to the container.
  Users can simply pass in `--user root:root` for easy access to all directories, however,
  this may allow more permission to docker than the user is comfortable with. For this reason,
  I will keep this section of the README.
  
  Permissions are particularly important when writing, since most directories, by default,
  will block docker from writing if the proper user id is not passed in. Follow these steps
  to retrieve and use the proper user id:
  
    1. Docker does not use the user names that are probably more known to the user, but instead
    uses the user id that is generated by the computer when a user is made (without going into
    too much detail). Thus, we need to know what user id to pass in. First, let's use the 
    command `ls -lha` to view ownership of our output directory (if needed, the user 
    should create an output directory to avoid ownership conflicts):
    
        ```
        drwxrwxr-x  3 foo foo 4.0K Oct 16 11:21 U54
        ```
        
        Thus we see that `foo` owns the directory named `U54`, but we can't pass `--user foo`
        to the container since it only deals with user ids. So we must get the user id of 
        `foo`
        
    2. To get the user id, we use the command `id`:
    
        ```
        uid=1001(foo) gid=1001(foo) groups=1001(foo)
        ```
        
        What we are most interested in is the `uid` element (however, in this example, `foo`
        only belongs to one group, so all ids are the same, but it should be noted
        that it is likely that there will be more elements when the user runs this command,
        so just use the `uid` that coincides with the owner of the output directory).
        
    3. Pass the `uid` into the `docker run` command:
    
        ```
        docker run <volumes> --user 1001 srp33/bwamtools:latest
        ```
        
        This will allow the docker container to write in the output directory and avoid
        conflicts that would cause errors.
        
  * Sometimes, the user may not have appropriate permissions, but belongs to a group that does.
  If this is the case, it would be appropriate to give the user id as well as the group id as
  follows:
  
    `--user <user id>:<group id>`
        
* __DEPRECATED:__ `-e`

  * This allows the user to assign certain variables within the container. There are only three 
  possible options, two of which are required and the other being optional:
  
    1. __*REQUIRED*__ `REF_GENOME=<name of reference genome>`
    
       * This will allow the container to know exactly which file to use as the reference genome,
       avoiding any need for guessing or risk of getting errors.
       
    2. __*REQUIRED*__ `SAMPLE=<sample id of reads>`
    
       * This is the sample ID for the reads, which will be used to name the `.bam` file outputted
       and run `bwa mem` on `<sample id>.1.fastq.gz` and `<sample id>.2.gz`.
       
    3. __*OPTIONAL*__ `THREADS=<integer>`
    
       * The user can specify how many threads to use. This must be an integer.
       
       * __DEFAULT:__ 1
       
* `--rm`

  * This will remove the container once it stops. `--rm` is not necessarily required (the container
  will still function properly if absent), however, it will reduce clutter.
  
* `[additional options]`

  * Any options that the user deems necessary for `docker run`.
  
* __Deprecated:__ `srp33/bwamtools:latest` __Use:__ `srp33/u54:latest`
  
## Supported Commands

> A comprehensive list of commands that will replace the need for environmental variables. U54 
currently does not support whole word flags. It should also be noted that there are no positional 
arguments to these commands. __All arguments must be preceded by a flag.__

* `align`

  * Uses `bwa` and `samtools` to align reads using a reference genome. 
  There are two required arguments:
  
    * `-r`
    
      * Name of reference genome file
      
    * `-s`
    
      * Sample id of reads
      
    There is an additional optional argument that may be used
    
    * `-t`
    
      * Number of threads to use. __Default:__ 1
    
* `index_bam`

  * Uses `sambamba` to index a `bam` file
  
    * `-b`
    
      * Name of bam file to be indexed
      
    Optional:
    
    * `-t`
    
      * Number of threads. __Default:__ 1
      
* `sort_bam`

  * Uses `sambamba` to sort a `bam` file. __NOTE:__ `bam` file must have been previously indexed and 
  index file (`*.bai`) must be in same directory as `bam` file.
  
    * `-b`
    
       * Name of bam file to be sorted
       
    Optional:
    
    * `-t`
    
       * Number of threads. __Default:__ 1
       
* `mark_duplicates`

  * Uses `sambamba` to mark duplicates in a `bam` file.
  
    * `-b`
    
      * Name of bam file to be read
      
    * `-o`
    
      * Name of output file
      * Saved into `results` directory
      
    Optional:
    
    * '-t'
    
      * Threads
      
* `merge_bams`

  * Uses `sambamba` to merge multiple `bam` files.
  
    * `-b`
    
      * Bam file to be merged with others
      * __NOTE:__ there must be two or more `bam` files for this to run correctly. Each
      `bam` file must be preceded by `-b`.
      
    * `-o`
    
      * Name of output, saved to `bam_files` directory
      
    Optional:
    
    * `-t`
    
      * Thread count
      
* `slice_bam`

  * `sambamba slice`, essentially
  
    * `-b`
    
      * `bam` file to be sliced
      
    * `-r`
    
      * Region to be sliced out of the `bam` file
      
    * `-o`
    
      * Output file, saved to `results` directory
      
    Optional:
    
    * `-t`
    
      * Thread count
      