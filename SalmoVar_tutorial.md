# Tutorial for salmonella_varcall v2.1

## Getting the list of Salmonella SRAs
I downloaded new SRA accessions using edirect/eutilities. This was the command I used to filter for salmonella full genome sequences in the SRA archive. The list is saved as `SraAccList.txt` in the project directory.
```bash!
esearch -db sra -query "txid28901[Organism:exp] AND (cluster_public[prop] AND 'biomol dna'[Properties] AND 'library layout paired'[Properties] AND 'platform illumina'[Properties] AND 'strategy wgs'[Properties] OR 'strategy wga'[Properties] OR 'strategy wcs'[Properties] OR 'strategy clone'[Properties] OR 'strategy finishing'[Properties] OR 'strategy validation'[Properties])" | efetch -format runinfo -mode xml | xtract -pattern Row -element Run > SraAccList.txt
```
In 2023, Lei's efetch command downloaded 543713 SRA sequences and took a few hours to finish processing. On 2024-09-09, Annette re-ran esearch and efetch and downloaded 685477 SRA sequences. 

## Metadata for SRA accessisons

Using the BigQuery browser interface, I uploaded the accession list SraAccList.txt and obtained the metadata using the SQL query below.  The resulting table was too large to download at once, so I searched for SRR (NCBI), ERR (European Bioinformatics Institute), and DRR (DNA Data Bank of Japan) accessions seperately and downloaded as .json files in Google Drive.  

```SQL!
SELECT * FROM `nih-sra-datastore.sra.metadata` 
where consent = "public" and acc like "SRR%" and acc in (SELECT string_field_0 FROM `salmonella-sra-2024.SRA_all.sra_670k`);
```

| Prefix | Accessions | Metadata entries |
| ------ | ---------: | ----------------: |
| SRR | 606195 | 605491 |
| ERR | 78004 | 78004 |
| DRR | 1278 | 1270 |
| Total | 685477 | 684765 |

## Nextflow pipeline

Based on the file [main.nf](https://github.com/USDA-ARS-GBRU/salmonella_varcall/blob/SciNet-Atlas/main.nf). Note that `Singularity` has been transitioned to `Apptainer`.

Processes in workflow:

* `sra_n_ch` 

* `sra_ch` 

* `fetch_SRA`: Uses fastq-dump to download the raw fastq files at 2,000,000 read depth. From `ghcr.io/usda-ars-gbru/salmonella-varcal/sra-tools-bash:latest`. 
* `megahit`: assembles raw reads into a `contigs.fa` file.
* `sistr`: Uses the [SISTR](https://github.com/phac-nml/sistr_cmd) command line tool to perform in silico typing on the assembled genome. Generates a one line `.tab` tab delimited file with SISTR's output
* `bwa_indexing`: Burrow-Wheeler Aligner for pairwise alignment between DNA sequences. Indexes the reference genome `GCF_000006945.2.fasta.gz`. From `biocontainers/bwa:v0.7.17_cv1`.

* `gatk_createdict`: Create dictionary for reference FASTA file. From `broadinstitute/picard:2.27.4`.
* `gatk_vcf_indexer`: From `broadinstitute/picard:2.27.4`.

* `map_reads`: bwa mem to map forward and reverse reads to the reference genome, generating a `.sam` file. From `biocontainers/bwa:v0.7.17_cv1`.

* `sort_and_index`: Uses samtools to sort and index the sam files into `.bam` format.From `biocontainers/samtools:v1.9-4-deb_cv1`.

* `mpileup`: Uses bcftools mpileup to call variants based on the reference genome. Generates tabix indexed and bgzipped `.mpileup.vcf.gz` file.

* `add_read_groups`: From `broadinstitute/picard:2.27.4`.

* `vg_pack` : Uses [vg](https://github.com/vgteam/vg) toolkit's giraffe, filter, and pack functions to generate a `pack.edge.table.gz` file which describes paths through each node of the graph alignment.

* `gvg_call`: Uses inhouse caller gfa_var_genotyper.pl to call variants based on the pack edge table and the reference graph. Generates a gzipped (note: NOT bgzipped) `vg.vcf.gz` file in the output directory

* `gatk_haplotypecaller`: From `broadinstitute/gatk:4.2.6.1`.

![Pipeline flowchart][def]

[def]: ~/Documents/Clone/salmonella_varcall/flowchart.png


* cleanup (sra_done_signal and mpileup_done_signal)
    * For every SRA accession, once it's passed through both variant calling pipelines, replaces the raw fastqs with decoy files using [tricking_nextflow_cache](https://github.com/pirl-unc/tricking_nextflow_cache)
    * For every SRA accession, once it's passed through the mpileup variant calling pipeline, replaces all intermediate files from map_reads, sort_and_index, and mpileup with decoy files using same tools as above
    * Files not tracked by nextflow are deleted with `rm` in the bash shell commands of each process

## List of output files generated
These output files can be found in the output.tar archive on Juno. There should be 1 of each of the files for each salmonella SRA accession. 
* mpileup VCF: variants called using mpileup. Accompanied by a .tbi index file in the same folder
* gvg VCF: variants called using gvg
* sistr: output of sistr in silico serotyping
* pack: pack edge tables for the gvg VCFs

## Seqera Platform

Formerly known as Nextflow Tower, Seqera helps you manage and monitor your workflow.  To create a token and use it with your Nextflow pipeline, follow their [tutorial](https://training.nextflow.io/basic_training/seqera_platform/#cli).

## Troubleshooting

## Authors
Annette M. Hynes and Lei Ma