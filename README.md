# nanostripper
Strip reads from Oxford Nanopore FAST5 files if they meet certain criteria

## Getting Started

Clone this repository if you have not done so already:

```console
git clone https://github.com/nodrogluap/nanostripper
cd nanostripper
```

This program is fairly simple but depends on [minimap2](https://github.com/lh3/minimap2) being in your path, and that you've 
got [Oxford Nanopore's FAST5 API](https://github.com/nanoporetech/ont_fast5_api) installed in Python.

If you have these requirements already, you are good to go.  If not, these requirements can be easily met with the 
included [conda](https://docs.conda.io/projects/conda/en/latest/user-guide/getting-started.html) environment, with no administrator privileges required, like so:

```console
conda env create -f nanostripper.yml
```

Then any time you want to run the program start with:

```console
conda activate nanostripper
```

Not using conda yet? Trust me, you should really try it.

Given contemporary Oxford Nanopore FAST5 multi-read files, the basic usage of this software is like so:

```console
./nanostripper thing_i_want_to_keep.fasta thing_i_want_to_exclude.fasta my_base_called_nanopore_data_file1.fast5 my_base_called_nanopore_data_file2.fast5 ...
```

The program will extract the basecalls from the FAST5, run them using 
minimap2 against the inclusion and exclusion (which overrides) reference FASTA file 
match criteria, and write the passed reads to ``stripped/my_base_called_nanopore_data_fileX.fast5``. For example to include only reads that map to the 
[SARS-CoV-2 genome](https://www.ncbi.nlm.nih.gov/nuccore/NC_045512), but not the [human genome](https://www.ncbi.nlm.nih.gov/assembly/GCF_000001405.26/) 
(e.g. a chimeric read from errant ligation during nanopore sample prep) at a bare minimum:


```console
./nanostripper SARS-CoV-2.fa hg38.fa my_nanopore_data_file.fast5 [list any other files you want to strip here]
```

This will produce a directory called ``stripped`` with the same files that you have given the program, but smaller as they have been stripped of the undesired reads. Optionally, a directory name can be provided on input instead of individual FAST5s. 
There is also a file giving the minimap2 match information (in [PAF format](https://github.com/lh3/miniasm/blob/master/PAF.md)) for the stripped reads, furnished
as ``nanostripper_summary.txt`` in the same directory for informational and debugging purposes.  Reads that 
match neither the include nor exclude criteria are listed in the PAF file with their name and length, but the rest of the columns are dummy values (including 
the 12th and last column, mapping quality, as 255 indicating unknown as per the PAF spec).

For a FAST5 file of 4000 reads against SARS-CoV-2 and human it takes about 18 seconds on a modern CPU when using 20 threads against a pre-built human minimap2 reference file (see below for threading and other options).

## Demultiplexing

Nanostripper can also perform sample-pool data demultiplexing for you (via [qcat](https://github.com/nanoporetech/qcat)), so that for each sample you get completely independent output FAST5 and ``sequencing_summary.txt`` files (e.g. for later use in [nanopolish](https://github.com/jts/nanopolish)). Simply add ``-demux`` to the command. 

```console
# Basecall the reads if you have not already, with FAST5 output enabled (you'll need to download Guppy separately from ONT's Website)
/path/to/ont-guppy/bin/guppy_basecaller --input_path my_test --save_path my_test/basecalled --num_callers 4 --config dna_r9.4.1_450bps_modbases_dam-dcm-cpg_hac.cfg --device cuda:all:100% --fast5_out

# Demultiplex and strip human data in one go with nanostripper
$ python ./nanostripper my_target_pathogen_genome.fna hg19.mmi my_test/basecalled -demux
Adapters detected in 3081 of 3963 reads
  NBD104/NBD114   3081: |      ############### |  77.73 %
           none    882: |                 #### |  22.27 %
Barcodes detected in 3081 of 3963 adapters
      barcode07    508: |                   ## |  12.82 %
      barcode08    804: |                 #### |  20.29 %
      barcode09    481: |                   ## |  12.14 %
      barcode10    480: |                   ## |  12.11 %
      barcode11    552: |                   ## |  13.93 %
      barcode12    256: |                    # |   6.46 %
           none    882: |                 #### |  22.27 %
Demultiplexing finished in 14.06s
[M::mm_idx_gen::0.119*1.02] collected minimizers
[M::mm_idx_gen::0.144*1.36] sorted minimizers
[M::main::0.144*1.36] loaded/built the index for 1 target sequence(s)
[M::mm_mapopt_update::0.158*1.33] mid_occ = 13
[M::mm_idx_stat] kmer size: 15; skip: 10; is_hpc: 0; #seq: 1
[M::mm_idx_stat::0.164*1.32] distinct minimizers: 499684 (97.41% are singletons); average occurrences: 1.042; average spacing: 5.348
[M::worker_pipeline::0.908*2.40] mapped 3963 sequences
[M::main] Version: 2.17-r941
[M::main] CMD: minimap2 -x map-ont -t 3 staph /tmp/tmpt9gwq73x
[M::main] Real time: 0.916 sec; CPU: 2.186 sec; Peak RSS: 0.066 GB
[M::main::7.131*1.00] loaded/built the index for 93 target sequence(s)
[M::mm_mapopt_update::8.797*1.00] mid_occ = 610
[M::mm_idx_stat] kmer size: 15; skip: 10; is_hpc: 0; #seq: 93
[M::mm_idx_stat::9.755*1.00] distinct minimizers: 100029959 (38.72% are singletons); average occurrences: 5.458; average spacing: 5.746
[M::worker_pipeline::12.299*1.39] mapped 3963 sequences
[M::main] Version: 2.17-r941
[M::main] CMD: minimap2 -x map-ont -t 3 hg19.mmi /tmp/tmpt9gwq73x
[M::main] Real time: 12.573 sec; CPU: 17.391 sec; Peak RSS: 7.680 GB
my_test/basecalled/workspace/FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5: stripped 587/3963 reads

$ ls -l stripped/basecalled/*
stripped/basecalled/10:
total 98868
-rw-rw-r-- 1 gordonp gordonp 101115083 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    118500 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/11:
total 130716
-rw-rw-r-- 1 gordonp gordonp 133706452 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    136519 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/12:
total 67924
-rw-rw-r-- 1 gordonp gordonp 69479939 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    66581 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/7:
total 123492
-rw-rw-r-- 1 gordonp gordonp 126322848 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    126466 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/8:
total 156376
-rw-rw-r-- 1 gordonp gordonp 159920440 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    200310 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/9:
total 117148
-rw-rw-r-- 1 gordonp gordonp 119829221 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    120312 Jul  9 11:02 sequencing_summary.txt

stripped/basecalled/none:
total 172216
-rw-rw-r-- 1 gordonp gordonp 176186320 Jul  9 11:02 FAL08081_741cb7c06ae4795368d785667f1304f554bfb4f8_53.fast5
-rw-rw-r-- 1 gordonp gordonp    152772 Jul  9 11:02 sequencing_summary.txt
```

## More options

To use nanostripper to just demultiplex FAST5 or FASTQ files, you can use /dev/null (Linux) or nul (Windows) as the strip criteria instead of FastA files.

In addition to exporting the pass filter FAST5 records, nanostripper can also be asked to output the pass filter FASTQ data for your convenience (rather than needing to use poretools or another program to do the FAST5->FASTQ extraction). Simply add ``-fastq`` to the command invocation. A ``fastq`` directory will be created under each FAST5 output directory.


The three other options that can be given are listed in the program help, which can be accessed using ``nanostripper -h``:

```
usage: nanostripper [-h] [-out OUT] [-t T] [-fastq [FASTQ]] [-demux [DEMUX]]
                    [-m M]
                    pos_ref_file neg_ref_file fast5_dir_name
                    [fast5_dir_name ...]

Strip reads from Oxford Nanopore FAST5 files if/unless they meet certain
reference match criteria using minimap2.

positional arguments:
  pos_ref_file    reference FastA DNA file for FAST5 inclusion using minimap2
                  (or a pre-built .mmi index)
  neg_ref_file    reference FastA DNA file for FAST5 exclusion using minimap2
                  (or a pre-built .mmi index), overrides inclusion criterion
  fast5_dir_name  a directory of FAST5 files to recursively process (must have
                  basecalls already included)

optional arguments:
  -h, --help      show this help message and exit
  -out OUT        directory where modified FAST5 files will be written
                  (default: "stripped")
  -t T            number of threads to use for minimap2 (default: 3)
  -fastq [FASTQ]  output stripped FASTQ files as well (default: False)
  -demux [DEMUX]  demultiplex the data based on ONT barcodes using qcat
                  (default: False)
  -m M            minimum minimap2 mapping quality to consider an alignment as
                  a match (default: 20)
 ``` 
 
Finally, the processing can be made faster by providing the minimap index for a FASTA file rather than the FASTA file itself.  The minimap2 index can be generated like so just once:

```console
minimap2 -d hg38.mmi hg38.fa 
```

Then invoke ``nanostripper`` with this index as many times as you like:

```console
time python nanostripper -m 30 -t 12 SARS-CoV-2.fa hg19.mmi /path/to/FAL90649_dda1537ffc4910b351360f5c24d80ad08ffe7950_129.fast5

[M::mm_idx_gen::0.004*1.93] collected minimizers
[M::mm_idx_gen::0.006*3.74] sorted minimizers
[M::main::0.006*3.74] loaded/built the index for 1 target sequence(s)
[M::mm_mapopt_update::0.006*3.46] mid_occ = 3
[M::mm_idx_stat] kmer size: 15; skip: 10; is_hpc: 0; #seq: 1
[M::mm_idx_stat::0.007*3.26] distinct minimizers: 5587 (99.93% are singletons); average occurrences: 1.004; average spacing: 5.332
[M::worker_pipeline::0.087*4.61] mapped 4000 sequences
[M::main] Version: 2.17-r941
[M::main] CMD: minimap2 -x map-ont -t 12 SARS-CoV-2.fa /tmp/tmpkg0rwio1
[M::main] Real time: 0.089 sec; CPU: 0.404 sec; Peak RSS: 0.068 GB
[M::main::7.230*1.00] loaded/built the index for 93 target sequence(s)
[M::mm_mapopt_update::8.898*1.00] mid_occ = 610
[M::mm_idx_stat] kmer size: 15; skip: 10; is_hpc: 0; #seq: 93
[M::mm_idx_stat::9.857*1.00] distinct minimizers: 100029959 (38.72% are singletons); average occurrences: 5.458; average spacing: 5.746
[M::worker_pipeline::10.036*1.13] mapped 4000 sequences
[M::main] Version: 2.17-r941
[M::main] CMD: minimap2 -x map-ont -t 12 hg19.mmi /tmp/tmpkg0rwio1
[M::main] Real time: 10.367 sec; CPU: 11.630 sec; Peak RSS: 7.680 GB
/home/gordonp/FAL90649_dda1537ffc4910b351360f5c24d80ad08ffe7950_129.fast5: stripped 2857/4000 reads

real	0m17.561s
user	0m16.039s
sys	0m7.402s

```
