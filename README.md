# Microbiome QC report

I built a bacterial DNA quality report because I wanted to know when a microbiome comparison has enough usable data to trust. A sequencing file can contain errors, artificial mixed sequences and contamination. A tidy chart can hide all three.

I start with public raw reads, check their quality, recover distinct marker-gene sequences, attach organism names and compare diversity. The report shows where reads were lost, whether more sequencing would still find new variants, and which samples are too shallow to compare. It also checks a real extraction blank, a sample prepared without the target material, for contamination warnings.

**Try it without installing anything:** open the [saved report](reports/demo/report.md) on GitHub. For a portable version, download [report.html](reports/demo/report.html) and open it in a browser. Its charts are embedded, so it works offline.

The example is a public mock-community dilution series, a mixture with decreasing amounts of bacterial DNA. It demonstrates the analysis and its limits. I do not treat these dilution samples as independent biological replicates or use them to draw health conclusions.

## Run it myself

I use [micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) to install the pinned environment. On Linux:

```sh
micromamba create -y -n microbiome-qc -f environment-linux-64.lock
micromamba activate microbiome-qc
make demo
```

On an Apple Silicon Mac, use `environment-osx-arm64.lock` in the first command. Other platforms can solve `environment.yml`, but those are not the exact tested package builds. The environment includes R, Python and the analysis packages; `make` is supplied by the operating system's development tools.

`make demo` downloads and verifies the public inputs, keeps a small matching subset of each read pair and writes `results/demo/report.html` and `results/demo/report.md`. Internet access is needed for the initial read and reference downloads. Use a fresh output directory for every run:

```sh
make demo PAIRS=20000 DEPTH=5000 OUT=results/larger-subset
```

`PAIRS` caps raw pairs per sample in archive order. `DEPTH` is the minimum number of usable pairs required for comparison. Changing either changes the analysis; the report records the actual counts and decisions. I use a fixed seed for the equal-depth subsampling so I can repeat the comparison.

## What is inside

DADA2 learns the sequencing error rates, infers amplicon sequence variants (ASVs, distinct marker-gene sequences), joins overlapping read pairs and removes chimeras, which are artificial mixed sequences. It assigns taxonomy, or organism names, against the full RDP reference database. Uncertain names stay unassigned.

The pipeline then plots rarefaction, which asks how many variants I would see with fewer reads. Samples below the requested depth are excluded. It calculates alpha diversity, the variety within each sample, and beta diversity, differences between samples, at the same read depth. The report includes read survival, quality profiles, fitted error diagnostics, organism names and a distance matrix. The underlying tables and sequence FASTA are saved alongside it.

The blank screen flags variants enriched in the control for manual review. One blank provides limited evidence, and a flag is not proof of contamination. I keep flagged variants visible and do not silently delete them. Missing or empty controls are marked unavailable. See [the analysis choices and limitations](docs/methods.md) before using this on another dataset.

## Public inputs and references

I use SRA study [SRP155048](https://www.ncbi.nlm.nih.gov/sra/?term=SRP155048): runs [SRR7588938](https://www.ebi.ac.uk/ena/browser/view/SRR7588938) (D0), [SRR7588942](https://www.ebi.ac.uk/ena/browser/view/SRR7588942) (D4), [SRR7588945](https://www.ebi.ac.uk/ena/browser/view/SRR7588945) (D8) and [SRR10898399](https://www.ebi.ac.uk/ena/browser/view/SRR10898399) (extraction blank). The reads come from the public ENA mirror of the SRA deposit. No data source was replaced with synthetic data.

- Study: Caruso, Song, Asquith and Karstens (2019), [Performance of Microbiome Sequence Inference Methods in Environments with Varying Biomass](https://doi.org/10.1128/mSystems.00163-18).
- Raw-read locations and original checksums: [ENA file report](https://www.ebi.ac.uk/ena/portal/api/filereport?accession=SRP155048&result=read_run&fields=run_accession,sample_alias,fastq_ftp,fastq_md5&format=tsv), frozen in [data/sources.json](data/sources.json).
- Sequence inference: [DADA2 paper](https://doi.org/10.1038/nmeth.3869) and [the authors' tutorial](https://benjjneb.github.io/dada2/tutorial.html).
- Names: Callahan's [DADA2-formatted RDP trainset 18, release 11.5](https://doi.org/10.5281/zenodo.4310151), CC BY 4.0, and the [RDP classifier paper](https://doi.org/10.1128/AEM.00062-07).
- Diversity calculations: [vegan documentation](https://vegandevs.github.io/vegan/reference/index.html).

The saved report uses a technical subset of the real reads and the full taxonomy reference. It is not a reanalysis of every sample in the paper. The offline CI fixture has a smaller, explicitly labelled reference subset; its purpose is to test execution. [Fixture provenance and attribution](tests/fixture/README.md) explain the difference.

## Reproducibility and checks

The main tools are pinned in [environment.yml](environment.yml), with complete package URLs and hashes in the platform locks. Each run records its input subset hashes, parameters, package versions, count table and outputs. The implementation is in [scripts](scripts); detailed reasoning is in [docs/methods.md](docs/methods.md).

```sh
make fixture-test
python -m json.tool reports/demo/summary.json
```

CI runs the actual denoising and taxonomy pipeline on checked-in public FASTQ pairs, then validates both the new fixture report and the saved demonstration. Unit tests cover damaged or mismatched reads, failed checksums, missing and empty controls, shallow-sample exclusion and known diversity calculations. The fixture deliberately reduces one sample to exercise exclusion; that is not a claim about its original archive depth. CI uploads its report for inspection. The [saved fixture report](reports/fixture/report.md) also shows the shallow-sample warning without running anything. To repeat a fixture run locally, use a new directory, for example `make fixture-test FIXTURE_OUT=results/fixture-again`.

The saved public run processed 48,000 raw read pairs, retained 32,907 non-chimeric pairs and inferred 266 variants. 3 biological samples qualified for comparison at 5,000 pairs per sample. The saved CI fixture processed 6,080 raw pairs and excluded 1 deliberately shallow technical subset. These numbers come from `python scripts/metrics.py`; the summaries and supporting tables are committed with the reports.
