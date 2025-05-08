

# NCDHHS-Enterprise/sabr-jgt

## Introduction

**sabr-jgt** is a referenced-based genotyping pipeline. A user provides a reference genome and samplesheet of paired-end Illumina reads. Reads are mapped to the reference with BWA and genotypes called with GATK. Some read quality statisitics are reported along the way and basic hard-filtering performed at the end, but users need to check that filtering is sufficient and appropriate. 

### Some simplifying assumtions
1. Only one read pair per individual are allowed
   * Sometimes multiple libraries are sequenced for the same individual, which requires assigning different read groups then merging bams after alignment with BWA. This scenario is highly unlikely for small genomes and not accounted for here. Technical replicates of the same library should be able to be concatendated prior to genotyping without incident.
2. There are no intervals
   * Intervals can be defined for large genomes, these would be specific known regions of the reference genome with some biologial relevance, such as individual chromosomes. These intervals are used to *scatter-gather* genotyping operations such that each chromosome is an independent compute job. One could define mulitple intervals on a genome comprised of a single haploid chromosome, but the computational benefits deiminish with many small intervals as opposed to a few large ones. Namely, this creates a lot of extra IO and extra work on recombining the resulting *vcfs* into a single genome-wide *vcf*. Thus, intervals are ignored in favor of not spinning up many trivial compute requests.

### An important note
GATK is run with its default ploidy setting, diploid. While this seemed unintuitive to me at first, but the Datapult *Mycobacterium tuberculosis* pipeline uses the diploid genoypes to help screen for chimeras and contamination. I decided to continue this approach by letting GATK call 0/0, 0/1, or 1/1 genotypes and leaving it to the investigator to interpret the relevance as appropriate.

Some filtering of the multisample vcf is done based on the GATK best practices. This might not always be a good idea, so the vcfs at various stages are provided. No more filtering is done within **sabr-jgt** because using vcftools is very easy and coming up with depth and missing data cutoffs is a bit subjective requiring data exploration and judgement calls. Thus, the goal of this pipeline is to get the computational load of genotyping done and dealing with all of the GATK steps in a containerized envrionment. Steps beyond that are the time for science.


Development of the **sabr-jgt** pipeline borrowed heavily from [sarek](https://github.com/nf-core/sarek) to learn how to finesse various nf-core modules. Many of the GATK modules were copied over to `modules/local` and modified to simplify the genotyping for small single-chromosome organisms.

## Usage
The input data is a comma-separated sample sheet that looks as follows:

`samplesheet.csv`:

```csv
sample,fastq_1,fastq_2
SRR1574573,jgt_data/fastq/SRR1574573_1.fastq.gz,jgt_data/fastq/SRR1574573_2.fastq.gz
SRR1574576,jgt_data/fastq/SRR1574576_1.fastq.gz,jgt_data/fastq/SRR1574576_2.fastq.gz
SRR30888765,jgt_data/fastq/SRR30888765_1.fastq.gz,jgt_data/fastq/SRR30888765_2.fastq.gz
SRR31075530,jgt_data/fastq/SRR31075530_1.fastq.gz,jgt_data/fastq/SRR31075530_2.fastq.gz
SRR31075531,jgt_data/fastq/SRR31075531_1.fastq.gz,jgt_data/fastq/SRR31075531_2.fastq.gz
SRR31075532,jgt_data/fastq/SRR31075532_1.fastq.gz,jgt_data/fastq/SRR31075532_2.fastq.gz
```
Relative paths are acceptable too provided the root is the nextflow launch directory. Here is a real example where the sample sheet is in `data` and the reads are in `data/fastq`


A reference genome is expected. This is for reference alignment and genotyping. An interval file is also needed for GATK. The parameters can be defined from the command line with `reference` and `intervals`, respectively. Here is an example:
```bash
nextflow run sabr-jgt --input jgt_data/samplesheet.csv --outdir jgt_result --intervals jgt_data/ref/intervals.list --reference jgt_data/ref/ref.fa --phix jgt_data/db/bbduk/phix174_ill.ref.fa -profile docker
```
The reference fasta file is also has a default in the config and json as `data/ref/ref.fa` but this can be overwritten at the command line if need be. Similar to the samplesheet and fastq availability, one might be able to provide a relative path or need to provide a full path depending on the resource.

A script for preparing the *intervals* file is available at `bin/prepare_intervals.py`. It might be automated in the pipeline to create the intervals file from the reference genome, but for now it is simply an isolated step the user needs to do and think about.

Some notable options have been pre-configure in `conf\modules.config`:
 * Only properly paired reads with a mapping score of at least 20 are retained for genotyping with '-b -q 20 -f 2' -F 4' applied by samtools view
 * Some cpu allocations need to be changed here rather than editing the profiles in the module main.nf scripts because the linting will complain
 * HaplotypeCaller from GATK runs in GVCF mode. This would need to be changed to *BP_RESOLUTION* if genotype information for all sites, such as differentiating between invariant and data-deficient, is needed.

## Test Data

Some test data is available via [onedrive](https://ncconnect-my.sharepoint.com/:f:/r/personal/george_tiley_dhhs_nc_gov/Documents/pipeline_test_data/jgt_data?csf=1&web=1&e=APc9Zl). This includes a *Streptococcus pyogenes* rference genome[^1] and a sample of whole-genome illumina data from six isoaltes[^2]. Data are publically available. The phiX sequence used by bbduk is included as well. 

## Development-notes
Sevaral tasks are still needed for the joint-genotyping pipeline:
* mark duplicates prior to haplotype caller
* Build database of gvcfs
* Joint genotypeing
* hardfiltering with vcftools

## Credits

**sabr-jgt** was originally written by George P. Tiley.


## Citations

<!-- TODO nf-core: Add citation for pipeline after first release. Uncomment lines below and update Zenodo doi and badge at the top of this file. -->
<!-- If you use sabr/jgt for your analysis, please cite it using the following doi: [10.5281/zenodo.XXXXXX](https://doi.org/10.5281/zenodo.XXXXXX) -->

<!-- TODO nf-core: Add bibliography of tools and data used in your pipeline -->

An extensive list of references for the tools used by the pipeline can be found in the [`CITATIONS.md`](CITATIONS.md) file.

This pipeline uses code and infrastructure developed and maintained by the [nf-core](https://nf-co.re) community, reused here under the [MIT license](https://github.com/nf-core/tools/blob/main/LICENSE).

> **The nf-core framework for community-curated bioinformatics pipelines.**
>
> Philip Ewels, Alexander Peltzer, Sven Fillinger, Harshil Patel, Johannes Alneberg, Andreas Wilm, Maxime Ulysse Garcia, Paolo Di Tommaso & Sven Nahnsen.
>
> _Nat Biotechnol._ 2020 Feb 13. doi: [10.1038/s41587-020-0439-x](https://dx.doi.org/10.1038/s41587-020-0439-x).


[^1]: Salva-Serra F, Jaen-Luchoro D, Jakobsson HE, Gonzales-Siles L, Karlsson R, Busquets A, Gomila M, Bennasar-Figuera A, Russell JE, Fazal MA, Alexander S, Moore ERB. 2020. Complete genome sequences of Streptococcus pyogenes type strain reveal 100%-match between PacBio-solo and Illumina-Oxford Nanopore hybrid assemblies. Scientific Reports. 10:11656. DOI: https://doi.org/10.1038/s41598-020-68249-y

[^2]: Chochua S, Metcalf BJ, Li Z, Rivers J, Mathis S, Jackson D, Gertz Jr RE, Srinivasan V, Lynfield R, Van Beneden C, McGee L, Beall B. 2017. Population and whole genome sequence based characterization of invasive group A streptococci recovered in the united states during 2015. DOI: https://doi.org/10.1128/mbio.01422-17