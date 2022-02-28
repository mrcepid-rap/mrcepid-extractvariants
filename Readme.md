# ExtractVariants (DNAnexus Platform App)

This is the source code for an app that runs on the DNAnexus Platform.
For more information about how to run or modify it, see
https://documentation.dnanexus.com/.

### Table of Contents

- [Introduction](#introduction)
    * [Background](#background)
    * [Dependencies](#dependencies)
        + [Docker](#docker)
        + [Resource Files](#resource-files)
- [Methodology](#methodology)
- [Running on DNANexus](#running-on-dnanexus)
    * [Inputs](#inputs)
        + [Gene ID](#gene-id)
        + [Association Tarball](#association-tarball)
    * [Outputs](#outputs)
    * [Command line example](#command-line-example)
        + [Batch Running](#batch-running)

## Introduction

This applet extracts individual level variant information and covariate data used to run burden analyses using the
[mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting) applet.

This README makes use of DNANexus file and project naming conventions. Where applicable, an object available on the DNANexus
platform has a hash ID like:

* file – `file-1234567890ABCDEFGHIJKLMN`
* project – `project-1234567890ABCDEFGHIJKLMN`

Information about files and projects can be queried using the `dx describe` tool native to the DNANexus SDK:

```commandline
dx describe file-1234567890ABCDEFGHIJKLMN
```

**Note:** This README pertains to data included as part of the DNANexus project "MRC - Variant Filtering" (project-G2XK5zjJXk83yZ598Z7BpGPk)

### Background

Individual-level data is NOT provided by the [mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting) applet
built for use on the RAP. Therefore, a separate applet was needed to extract individual level information to enable sensitivity
analyses on individual genes identified by that applet. This applet will extract:

1. The exact phenotype/covariate information in tabular format that was used as input for burden tests
2. Individual-level information describing genotype-sample pairs
3. Variant-level information for all variants included in a test
4. A basic linear model to confirm the correct mask/maf combination was used to generate the results output by this applet

### Dependencies

#### Docker

This applet uses [Docker](https://www.docker.com/) to supply dependencies to the underlying AWS instance
launched by DNANexus. The Dockerfile used to build dependencies is available as part of the MRCEpid organisation at:

https://github.com/mrcepid-rap/dockerimages/blob/main/associationtesting.Dockerfile

This Docker image is built off of the primary 20.04 Ubuntu distribution available via [dockerhub](https://hub.docker.com/layers/ubuntu/library/ubuntu/20.04/images/sha256-644e9b64bee38964c4d39b8f9f241b894c00d71a932b5a20e1e8ee8e06ca0fbd?context=explore).
This image is very light-weight and only provides basic OS installation. Other basic software (e.g. wget, make, and gcc) need
to be installed manually. For more details on how to build a Docker image for use on the UKBiobank RAP, please see:

https://github.com/mrcepid-rap#docker-images

In brief, the primary **bioinformatics software** dependencies required by this Applet (and provided in the associated Docker image)
are:

* [bcftools](https://samtools.github.io/bcftools/bcftools.html)
* [plink1.9](https://www.cog-genomics.org/plink2)
* [plink2](https://www.cog-genomics.org/plink/2.0/)
* [qctool](https://www.well.ox.ac.uk/~gav/qctool_v2/index.html)
* [bgenix](https://enkre.net/cgi-bin/code/bgen/doc/trunk/doc/wiki/bgenix.md)
    
This applet also uses the "exec depends" functionality provided as part of the DNANexus applet building. Please see
[DNANexus documentation on RunSpec](https://documentation.dnanexus.com/developer/apps/execution-environment#external-utilities)
for more information. This functionality allows one to specify python packages the applet needs via pip/apt. Here we install:

* [pandas](https://pandas.pydata.org/)
* [statsmodels](https://www.statsmodels.org/stable/index.html)

See `dxapp.json` for how this is implemented for this applet. These python modules are used to perform linear/logistic
models for confirmation of gene/phenotype associations.

This list is not exhaustive and does not include dependencies of dependencies and software needed
to acquire other resources (e.g. wget). See the referenced Dockerfile for more information.

#### Resource Files

This applet makes use of the filtered genotype data generated using the [mrcepid-buildgrms](https://github.com/mrcepid-rap/mrcepid-buildgrms)
applet included in this repository. Please see that repository for more information.

## Methodology

This applet uses the final output of [mrcepid-mergecollapsevariants](https://github.com/mrcepid-rap/mrcepid-mergecollapsevariants.git)
to perform rare variant burden testing. As such, the filtering for variants that the user provided beginning with [mrcepid-collapsevariants](https://github.com/mrcepid-rap/mrcepid-collapsevariants.git)
will determine what variants are processed by this applet.

This applet proceeds in the same basic steps as the [mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting)
applet. Please see that applet's documentation for more information on how covariates, individuals, and genotypes are selected
for inclusion/exclusion in the final output of this applet.

This applet uses an IDENTICAL methodology to run GLMs as in the [mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting)
applet. Please see the documentation for that applet on the basics of running GLM-based burden tests on the RAP.

## Running on DNANexus

### Inputs

There are a standard set of command-line inputs that may be useful to change for the typical user. Most of these inputs
are identical to those used for [mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting).
**Notable Differences** are for `gene_id` and `association_tarball`.

| input                   | description                                                                                                                                                                                      |
|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| gene_id                 | either an ENST or HGNC gene symbol to extract individual-level information for                                                                                                                   |
| association_tarball     | Hash ID of the output of ONE tarball from [mrcepid-collapsevariants](https://github.com/mrcepid-rap/mrcepid-collapsevariants) that you wish to use to extract individual-level information from. |
| phenofile               | Phenotypes file – the same format as that used for association testing                                                                                                                           |
| is_binary               | Is the given trait in the phenofile binary?                                                                                                                                                      |
| sex                     | Run only one sex or both sexes be run (0 = female, 1 = male, 2 = both) **[2]**?                                                                                                                  |
| inclusion_list          | List of samples (eids) to include in analysis **[None]**                                                                                                                                         |
| exclusion_list          | List of samples (eids) to exclude in analysis **[None]**                                                                                                                                         |
| output_prefix           | Prefix to use for naming output tar file of association statistics. Default is to use the file name 'assoc_stats.tar.gz'                                                                         |
| covarfile               | File containing additional covariates to correct for when running association tests                                                                                                              |
| categorical_covariates  | comma-delimited list of categorical covariates found in covarfile to include in this model                                                                                                       |
| quantitative_covariates | comma-delimited file of quantitative covariates found in covarfile to include in this model                                                                                                      |

There are also several command-line inputs that should not need to be changed if running from within application 9905. These
mostly have to do with the underlying inputs to models that are generated by other tools in this pipeline. We have set
sensible defaults for these files and only change them if running from a different set of filtered data.

| input             |description             | default file (all in `project-G6BJF50JJv8p4PjGB9yy7YQ2`) | 
|-------------------|------------------------| ------- |
| bgen_index        | index file with information on filtered and annotated UKBB variants      | `file-G86GJ3jJJv8fbXVB9PQ2pjz6` |
| transcript_index  | Tab-delimited file of information on transcripts expected by runassociationtesting output | `file-G7xyzF8JJv8kyV7q5z8VV3Vb` |
| base_covariates   | base covariates (age, sex, wes_batch, PC1..PC10) file for all WES UKBB participants       | `file-G7PzVbQJJv8kz6QvP41pvKVg` |
| fam_file          | corresponding .fam file for 'bed_file'                                   | `file-G6qXvg0J2vffF7Y44VFJb7jB` |

#### Gene ID

Gene ID can be either 

#### Association Tarball

`association_tarball` takes **ONE** DNANexus file-IDs that points to a single output from the 'mergecollapsevariants'. 
This file is one file-ID per line. **Please Note** that this is different from the `association_tarballs` input for [mrcepid-runassociationtesting](https://github.com/mrcepid-rap/mrcepid-runassociationtesting).

### Outputs

| output                 | description                                   |
|------------------------|-----------------------------------------------|
| output_tarball         | Output tarball containing summary information |

output_tarball is named `<gene_id>.assoc_stats.tar.gz` by default. If the parameter `output_prefix` is provided a file like (set `-ioutput_prefix="GIGYF1_MAF-01"`):

`GIGYF1_MAF-01.assoc_stats.tar.gz`

Would be created. This tar.gz file will contain files that contain various summary information:

1. `<output_prefix>.phenotypes_covariates.formatted.tsv`
   + a tab-delimited text file that contains the phenotype, individual ID (IID/FID) and all covariates for individuals *INCLUDED* in the model after applying exclusion/exclusion lists, missingness, and sex filtering.
2. `<output_prefix>.carriers_formated.tsv`
   + a tab-delimited text file with chrom/pos/varID/REF/ALT/IID/Genotype columns for ALL carriers included in the model
3. `<output_prefix>.association_stats.tsv`
   + Similar to the GLM output of mrcepid-runassociationtesting. Includes Beta/OR, std. error, and p. value for the linear model
4. `<output_prefix>.variant_table.tsv` 
   + All variants with annotation included in the model after filtering on carriers. This output is very similar to that 
   included in the filtered .bgen VEP annotations provided by (mrcepid-makebgen)[https://github.com/mrcepid-rap/mrcepid-makebgen]. 
   A noteable exception is columns with the suffix "tested" that list summary stats for AC/MAF of individuals included in 
   the model.

### Command line example

If this is your first time running this applet within a project other than "MRC - Extract Variants", please see our
organisational documentation on how to download and build this app on the DNANexus Research Access Platform:

https://github.com/mrcepid-rap

This applet can be run similar to the following:

```commandline
dx run mrcepid-extractvariants --destination results/gene_specific/ \
 --name GIGYF1.T2D --yes --brief \
 -iassociation_tarball=collapsed_variants_new/HC_PTV-MAF_01.tar.gz \ 
 -iphenofile=file-G5PFv00JXk80Qfx04p9X56X5 \
 -iis_binary=true \
 -ioutput_prefix=HC_PTV.GIGYF1.T2D \
 -igene_id=ENST00000678049 \
 -iinclusion_list=file-G6qXvjjJ2vfQGPp04ZGf6ygj 
```

**Note:** boolean parameters must be provided in json-compatible case ('true' or 'false').

Brief I/O information can also be retrieved on the command line:

```commandline
dx run mrcepid-extractvariants --help
```

I have set a sensible (and tested) default for compute resources on DNANexus for running BOLT and SAIGE. This is baked into the json used for building
the app (at `dxapp.json`) so setting an instance type when running either of those tools or all tools at once is unnecessary.
This current default is for a mem1_ssd1_v2_x4 instance (4 CPUs, 8 Gb RAM, 100Gb storage).

#### Batch Running

This applet is not compatible with batch running.

