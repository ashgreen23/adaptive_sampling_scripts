# adaptive_sampling_scripts
Scripts and data for adaptive sampling analysis

NOTE: An example of how to run a simulation on BEEG is depicted on my MRes project page, but further details about scripts/commands and what they are actually doing is available here, as I did not write this code. analyse_unblocks.py is used for alignment. analyse_RU.py is used for enrichment. See below for SH's descriptions of these scripts.

## Data

- `data/cps/Clusters` directory: contains CBL sequences clusterd by scheme in [Mavroidi et al.] (https://pubmed.ncbi.nlm.nih.gov/17766424/).
- `data/cps/Sequences` directory: genomes for pnuemococcal strains used frequently during analysis.
- `data/cps/split_cps` directory: reference sequences for CBL.
- `data/cps/updated_cps.fasta`: file containing up-to-date sequences for all known CBL (as of Decemeber 19th 2022)
- `data/cps/README.md`: Details of sources for all CBL sequences within `data/cps/updated_cps.fasta`.

## Analysing adaptive sampling performance

### analyse_RU.py

`analyse_RU.py` is an edited version of the script available [here](https://github.com/SR-Martin/Adaptive-Sequencing-Analysis-Scripts).
This edited script allows removal of multimapping reads from [Minimap2](https://github.com/lh3/minimap2) and filtering based on % identity to the aligning reference.

To use analyse_RU.py, direct it to a read directory containing `fastq_pass` and `fastq_fail` folders containing `fastq.gz` files generated by a basecaller:
```
python analyse_RU.py -f /path/to/read/directory/ -p 0.84 -o /output/prefix -c 1-256 -i /path/to/reference.fasta -t "23F,19A,19F"
```

To specify which sequence(s) the `reference.fasta` to align to, use the `-t` flag using a comman separated list as above. The names must match fasta headers in the `reference.fasta` file.

To specify which channels to separate (e.g. if you have used adaptive sampling on some channels), use `-c`, followed by a hyphen separated pair of integers between 1 and 512.

To change the proportion of an ALIGNMENT (NOTE: not the full read) that must match a reference to be deemed as correct, use the `-p` flag. As default, a multimapping read will assigned to the sequence it has the greatest aligning proportion to (turn this off using `-r`).

By default, both pass and fail reads are used. To use pass only, specify `-b`

This will generate a `_summary.txt` file containing statistics about read mapping and yield, and `_bootstrap.txt`, which contains bootstrap enrichment calculations (change number of bootstrap samples with `-bs`)

### analyse_overhang.py

Works in a similar way to `analyse_RU.py`. Determines the length of the overhang of each aligned read outside of the target locus.

To use `analyse_overhang.py`, direct it to a read directory containing `fastq_pass` and `fastq_fail` folders containing `fastq.gz` files generated by a basecaller:
```
python analyse_overhang.py -f /path/to/read/directory/ -p 0.84 -o /output/prefix -c 1-256 -i /path/to/reference.fasta -t "23F,19A,19F"
```

This will generate a summary file, which details the largest overhang for each barcode, and a dataframe describing the amount of overhang for each read in the dataset.

### analyse_coverage.py

`analyse_coverage.py` determines the proportion of a reference covered by reads. 

Run with:
```
python analyse_coverage.py -f /path/to/read/directory/ -p 0.84 -o output/prefix -i /path/to/reference.fasta -t "23F,19A,19F"
```

`-p`, `-c`, `-b`, `-r` and `-t` behave the same as with `analyse_RU.py`.

This will output a `X_adaptive_hist.csv` (for channels specified by `-c`) and `X_control_hist.csv` (for all other channels) for each barcode, which details the coverage at each position in the reference. `X` specifies which reference was been aligned to. Additionally, a `_summary.txt` file detailing total bases aligning and the average coverage to each reference sequence is generated.

### analyse_read_lengths.py

`analyse_read_lengths.py` generates an output file summarising read lengths from a sequencing run. It can be used to analyse simulated runs from bulk recordings.

Run with:
```
python analyse_read_lengths.py --indir /path/to/read/directory/ --out /output/filename"
```

This generates a file detailing the read length of all reads along with their barcode, channel, and whether they passed quality filtering.

## Parsing sequences

### split_by_channel.py

`split_by_channel.py` can be used to split reads into two separated files depending on specified channel. For use when adaptive sampling has been used on some but not all channels.

Run with:
```
python split_by_channel.py --infile /path/to/reads.fastq --out /output/prefix --channels 1-256
```

This will output `target.fastq` for reads from the stipulated channels, and `nontarget.fastq` for all others.

### parse_reads_by_time.py

`parse_reads_by_time.py` splits reads into different fasta files based on the time since sequencing began. This is useful when simulating time-series analysis of run performance e.g. real-time assembly.

Run with:
```
python parse_reads_by_time.py --dir /path/to/reads/directory --output /output/prefix --start 2023-04-20T14:29:36.956327+01:00 --end 2023-04-21T14:30:40.938064+01:00 --breaks 6
```

`--dir` is a directory containing unzipped `fastq` files. If analysing pass and fail reads together, these must be combined into a single `fastq` file for each individual barcode.

`--start` and `--end` are required, and detail the time and date of the run's start and end. These can be pasted directly from `final_summary.txt` file present in a sequencing run's read output directory generated by MinKNOW.

`--breaks` describes the number of time breaks to separate the run into. For example, a 24 hour run separated into 6 breaks will generate 6 `fastq` files per barcode, one for every 4 hour period of sequencing.

This will generate `_time_X.fastq` for each file in the `--dir`, where X defines the time break number.

### split_by_alignment.py

`split_by_alignment.py` parses reads based on their alignment to a given reference, placing them in separate files.

To run
```
python split_by_alignment.py --indir /path/to/read/directory/ --pid 0.84 --outdir /output/directory --channels 1-256 --ref /path/to/reference.fasta --target "23F,19A,19F"
```

To specify which sequence(s) the `reference.fasta` to align to, use the `--target` flag using a comman separated list as above. The names must match fasta headers in the `reference.fasta` file.

To specify which channels to separate (e.g. if you have used adaptive sampling on some channels), use `--channels`, followed by a hyphen separated pair of integers between 1 and 512.

To change the proportion of an ALIGNMENT (NOTE: not the full read) that must match a reference to be deemed as correct, use the `--pid` flag. As default, a multimapping read will assigned to the sequence it has the greatest aligning proportion to (turn this off using `--remove`).

By default, both pass and fail reads are used. To use pass only, specify `--pass-only`

This will generate `fastq` files for each of the references specified to `--target` if any read align. Reads not aligning will be placed in `_unaligned.fastq`. Reads generated by channels specified in `--channels` will be placed into `_adaptive_*.fastq`, all others will be placed into `_control_*.fastq`

### loci_cutter.py

`loci_cutter.py` aligns a database containing loci of interest to individual genomes and 'cuts' it out, generating `fasta` files containing the locus and the remainder of the genome.

Run with:
```
python loci_cutter.py --infile /path/to/infile.txt --query /path/to/locus.fasta --cutoff 0.7 --outpref /output/prefix --separate --count
```

`--infile` is a list of file paths to `fasta` files containing sequences from which to cut loci from (one path per line). 

`--query` is a `fasta` file containing the loci to align and cut.

`--cutoff` defines the minimum identity between the input sequence and an aligned locus to cut.

`--separate` places the cut loci and remaining sequence in different files, otherwise placed in the same file.

`--count` generates an output file detailing the number of each loci in `--query` found in the `fasta` within `--infile`.


This will generate `_cut.fa` which contains the locus, and `_rem.fa` which is the remainder of the sequence. If no locus is found, the sequence will be generated as `_nocut.fa` 

## Simulation

### kmer_simulation.R

`kmer_simulation.R` is an R script set up to simulate error-prone reads from a target and non-target genome and determine the sensitity and specificity of pseudoalignment using different read lengths (`read.lengths` variable), mutation rates (`mu.rates` variable) and k-mer lengths (`kmer.sizes` variable). 

This script will automatically generate plots describing:
- pseudoalignment sensitivity (`kmer_mu_comp_readlen` plot)
- pseudoalignment specificity (`kmer_random_match_readlen` plot)
- effect of identity cut-off (`kmer_cutoff_mu` plot)
- Reciever operator curve (`ROC_len` plot)
- Concordance between estimated and real read identity (`mash_mu_concordance` plot)
- RMSE between estimate and real read identity (`RMSE_mash` plot)

To avoid rerunning simulations, k-mer sequences are placed in `all_simulations.csv` and `all_cutoffs.csv` for re-generating plots.

### analyse_unblocks.py

`analyse_unblocks.py` aligns reads to a given reference and parses their length. It can be used to analyse simulated runs from bulk recordings.

Run with:
```
python analyse_unblocks.py --indir /path/to/read/directory/ --out /output/filename --ref /path/to/reference.fasta --target "23F" --loci /path/to/locus.fasta --mux-period 480
```

To specify which sequence(s) the `reference.fasta` to align to, use the `--target` flag using a comman separated list as above. The names must match fasta headers in the `reference.fasta` file.

If you have loci of interest within the `reference.fasta`, you can separately supply it to `--loci` in fasta format. These will be aligned to the reference, and the coordinates used in read mapping to assign reads at correctly mapping to or missing the locus.

To ignore reads from the start of the run e.g. during the mux period, specify using `--mux-period` (in seconds).

### split_simulated.py

`split_simulated.py` is used to parse simulated reads based on their place of origin for use in `simulate_readuntil.py`.

First generate simulated reads using [NanoSim-H](https://github.com/karel-brinda/NanoSim-H)

```
nanosim-h-train -i /path/to/training.fasta /path/to/reference/genome.fasta output_dir
nanosim-h -n 500000 -p output_dir /path/to/reference_genome.fasta
```

This will generated `simulated.fa`, which details where each read was generated from in `reference_genome.fasta` in terms of locus

Then parse reads based on their position with:
```
python split_simulated.py --infile /path/to/simulated.fa --names "Genome_X" --pos 0-1000 --min-overlap 50 --out parsed_simulated
```

`--names` defines the header of the specific sequence in `reference_genome.fasta` to be used as a reference. `--pos` defines the locus to split reads generated from that region. `--min-overlap` defines how small of an overlap is required between the locus defined in `--pos` and the read for the read to be parsed.

This will output `parsed_simulated_target.fasta` (true positive reads) containing reads generated from the locus of interest, and `parsed_simulated_nontarget.fasta` (true negative reads) for all others.

### simulate_readuntil.py

`simulate_readuntil.py` compares the time taken to align reads to a [Bifrost](https://github.com/pmelsted/bifrost) and [Minimap2](https://github.com/lh3/minimap2) index.

First install Minimap2 and the [graph alignment](https://github.com/samhorsfield96/readfish/releases/tag/GraphAlignment%2Fv0.0.1) branch of readfish.

Then, generate Bifrost and Minimap2 indexes:

```
minimap2 -d index.mmi /path/to/input.fasta
ru_generate_graph --refs /path/to/reference/input.txt --kmer 19 --out index
```

Finally run with:
```
python simulate_readuntil.py --infile /path/to/reads.fasta --mappy-index index.mmi --graph-index index.gfa --id 0.75 --min-len 50 --out output_file.txt --avg-poi 180
```

`--infile` is a path to the reads to be aligned. Usually they would have been simulated and parsed using `split_simulated.py` above.

`--mappy-index` and `--graph-index` are the indexes generated above. `--id` and `--min-len` control the cutoff of to graph pseudoalignment identity and minimum read length respectively. `--avg-poi` defines the average length taken from the start of each read during alignment to the index to mimic the process of NAS.

This will output `output_file.txt`, which has for columns:
- The tool used (Graph or Mappy)
- Time taken for alignment
- Fragment length aligned
- Whether the read was rejected (1) or not (0)
