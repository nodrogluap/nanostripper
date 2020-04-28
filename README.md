# nanostripper
Strip reads from Oxford Nanopore FAST5 files if they meet certain criteria

## Getting Started

Clone this repository:

```console
git clone https://github.com/nodrogluap/nanostripper
cd nanostripper
```

This program is fairly simple but depends on [minimap2](https://github.com/lh3/minimap2) being in your path, and that you've 
got [Oxford Nanopore's FAST5 API](https://github.com/nanoporetech/ont_fast5_api) installed in Python.

If you have these requirements already, you are good to go.  If not, these requirements can be easily met with the 
included (conda)[https://docs.conda.io/projects/conda/en/latest/user-guide/getting-started.html] environment like so:

```console
conda env create -f nanostripper.yml
```

Then any time you want to run the program start with:

```console
conda activate nanostripper
```

Given contemporary Oxford Nanopore FAST5 multi-read files, the basic usage of this software is like so:

```console
./nanostripper thing_i_want_to_keep.fasta thing_i_want_to_exclude.fasta my_base_called_nanopore_data_file1.fast5 my_base_called_nanopore_data_file2.fast5 ...
```

The program will extract the basecalls from the FAST5, run them using minimap2 against the inclusion and exclusion (which overrides) reference FASTA file 
match criteria, and write the passed reads to ``stripped/my_base_called_nanopore_data_fileX.fast5``. For example to include only reads that map to the 
[SARS-CoV-2 genome](https://www.ncbi.nlm.nih.gov/nuccore/NC_045512), but not the [human genome](https://www.ncbi.nlm.nih.gov/assembly/GCF_000001405.26/) 
(e.g. a chimeric read from errant ligation during nanopre sample prep) at a bare minimum:


```console
./nanostripper SARS-CoV-2.fa hg38.fa my_nanopore_data_file.fast5 [list any other files you want to strip here]
```

This will produce a directory called ``stripped`` with the same files that you have given the program, but smaller as they have beenm stripped of the undesired reads.
There is also a file giving the minimap2 match information (in [PAF format](https://github.com/lh3/miniasm/blob/master/PAF.md)) for the stripped reads is furnished
as ``nanostripper_summary.txt`` in the same directory for informational and debugging purposes.  Reads that 
match neither the include nor exclude criteria are listed in the PAF file with their name and length, but the rest of the columns are dummy values (including 
the 12th and last column, mapping quality, as 255 indicating unknown as per the PAF spec).

## More options

The three main options that can be given are listed in the program help, which can be accessed using `nanostripper -h``:

> usage: nanostripper [-h] [-out OUT] [-t T] [-m M]
>                     pos_ref_file neg_ref_file fast5_file [fast5_file ...]
> 
> Strip reads from Oxford Nanopore FAST5 files if/unless they meet certain
> reference match criteria using minimap2.
> 
> positional arguments:
>   pos_ref_file  reference FastA DNA file for FAST5 inclusion using minimap2
>                 (or a pre-built .mmi index)
>   neg_ref_file  reference FastA DNA file for FAST5 exclusion using minimap2
>                 (or a pre-built .mmi index), override inclusion criterion
>   fast5_file    a FAST5 file to process (must have basecalls already included)
> 
> optional arguments:
>   -h, --help    show this help message and exit
>   -out OUT      directory where modified FAST5 files will be written (default:
>                 "stripped")
>   -t T          number of threads to use for minimap2 (default: 3)
>   -m M          minimum minimap2 mapping quality to consider an alignment as a
>                 match (default: 20)
> 
> real	0m0.371s
> user	0m2.735s
> sys	0m2.391s

