# AMRFinderPlus assembly validation

This validation layer keeps source AST observations separate from genomic
caller evidence. It does not infer phenotype from genes, predict R/S, apply
CLSI/EUCAST breakpoints or calculate genotype–phenotype performance. A caller
no-hit result does not establish susceptibility.

## Cohort selection and retrieval

`benchmark/cohorts/ecoli_amrfinder_validation.tsv` defines the intended small
cohort schema. It remains header-only until each exact versioned PDT accession,
BioSample and Assembly relationship is verified and the Assembly report
confirms BioSample and *Escherichia coli*. Candidate AST rows come from the
PR #3 evidence layer and are retained at observation level, including
conflicting and nonbinary categories. Inclusion is based on source availability
and identity, never on AMRFinderPlus calls or agreement with AST. Missing or
ambiguous mappings are excluded with an explicit reason, not guessed.

PR #3's documented BioSample `SAMN05170351` had AST linked to SRA
`SRR4065706` but no Assembly. During this change, two small official Pathogen
Detection `action=retrieve` lookups briefly succeeded on 2026-10-11 UTC and
showed candidate assembly-linked records:

| PDT | BioSample | Assembly | Observation |
| --- | --- | --- | --- |
| `PDT000026734.2` | `SAMN02581401` | `GCA_000633675.2` | First retrieved AST page contained ND observations. |
| `PDT000041778.1` | `SAMN03075588` | `GCA_000770275.1` | First retrieved AST page contained conflicting submitter observations. |

The candidate `GCA_000770275.1` has since been checked against an official NCBI
Datasets genome package retrieved on 2026-10-11. Its report identifies
BioSample `SAMN03075588`, *Escherichia coli*, strain MRSN17749, Assembly
`ASM77027v1`, 92 sequences and 5,046,460 bases. The package's NCBI MD5 list and
recorded SHA-256 hashes were verified. The package remains locally under the
Git-ignored `datasets/ncbi_manual/` directory. The candidate is still not
admitted: its complete, source-preserved Pathogen Detection AST export has not
been archived and linked.
The AST request used the official [Pathogen Detection service](https://www.ncbi.nlm.nih.gov/pathogens/pathogens-srv/)
with `collection=ast`, filter `scientific_name:"Escherichia coli"`, and
`limit=20`; the metadata lookup used `collection=isolates` with the two exact
PDT accessions and projected `target_acc,biosample_acc,asm_acc,scientific_name`.
The full table projections and export contract are in
[`schemas/ast_raw_schema.md`](../schemas/ast_raw_schema.md).

The downloader retrieves only explicitly listed accessions from NCBI Datasets,
then checks the returned Assembly accession, BioSample, exact organism, FASTA
structure and SHA-256. Its provenance records the manifest hash, source URL,
retrieval time, package member and FASTA hash. It refuses to overwrite prior
outputs. Genome files and provenance are ignored by Git.

If the command-line environment cannot reach NCBI, retrieve the complete
official NCBI Datasets ZIP in a browser or another network-enabled environment.
Keep `md5sum.txt`, the Assembly report and FASTA in the package. Create a
tab-separated package manifest with columns `assembly_accession`,
`package_path`, `source_url` and timezone-qualified `retrieved_at`; paths may
be absolute or relative to that table. The offline importer checks every
member listed in `md5sum.txt`, current and legacy NCBI report identity fields,
FASTA structure, package hash and cohort mapping before publishing files.

```sh
python3 bin/genome_dataset.py import-packages \
  --manifest benchmark/cohorts/ecoli_amrfinder_validation.tsv \
  --packages-tsv /path/to/ncbi-packages.tsv \
  --outdir datasets/ecoli_validation/genomes
python3 bin/genome_dataset.py validate \
  --manifest benchmark/cohorts/ecoli_amrfinder_validation.tsv \
  --genomes datasets/ecoli_validation/genomes
```

```sh
python3 bin/genome_dataset.py download \
  --manifest benchmark/cohorts/ecoli_amrfinder_validation.tsv \
  --outdir datasets/ecoli_validation/genomes
python3 bin/genome_dataset.py validate \
  --manifest benchmark/cohorts/ecoli_amrfinder_validation.tsv \
  --genomes datasets/ecoli_validation/genomes
```

Once candidate accessions are selected, archive official AST evidence through
the existing builder. The query is the live NCBI browser export interface and
may return a large, changing table; inspect the resulting PDT/BioSample/
Assembly mappings before editing the cohort manifest.

```sh
python3 bin/ast_dataset.py download \
  --outdir datasets/ecoli_validation/ast_raw \
  --query 'scientific_name:"Escherichia coli"' --max-bytes 536870912
python3 bin/ast_dataset.py build \
  --snapshot datasets/ecoli_validation/ast_raw \
  --outdir datasets/ecoli_validation/ast_processed
```

## Caller environment, database and control

`environment/amrfinderplus.yml` pins AMRFinderPlus 4.2.7 and Python 3.12.
AMRFinderPlus 4.2.7 was installed in a temporary macOS ARM64 Conda environment
using conda-forge and bioconda, without accepting additional channel terms.
Database snapshot `2026-08-07.1` was downloaded from NCBI FTP; all 162 files in
the recorded SHA-256 manifest pass. The real binary reports software 4.2.7,
database 2026-08-07.1, and lists `Escherichia` as a supported organism option.
Package URLs and hashes, database provenance and file hashes remain in
Git-ignored `datasets/amrfinderplus-db/`; the environment is under
`/private/tmp` and may not persist across restarts.

On a network-enabled host, create the pinned environment and download a
separate database snapshot. Keep the generated dated release directory
immutable; pass that exact directory to `--database` for controls and cohort
runs. `amrfinder --database_version` reports its software and database
versions, while the call provenance records every database file hash.

```sh
conda env create -f environment/amrfinderplus.yml
database_root="$PWD/datasets/amrfinderplus-db"
mkdir -p "$database_root"
conda run -n amrscan-amrfinder-validation amrfinder_update -d "$database_root"
find "$database_root" -mindepth 1 -maxdepth 1 -type d -print
database="<choose-the-dated-release-directory-printed-above>"
conda run -n amrscan-amrfinder-validation amrfinder --database "$database" \
  --database_version
conda run -n amrscan-amrfinder-validation amrfinder -l
```

NCBI's AMRFinderPlus installation test is a reproducible positive **software
control** from the upstream `ncbi/amr` repository at tag `amrfinder_v4.2.7`:
`test_dna.fa`, `test_prot.fa`, `test_prot.gff` and `test_both.expected`. The
expected output includes known AMR elements such as `blaTEM-156`, `blaPDC-114`
and `vanG`. NCBI identifies the software/database repository as public domain.
This is not an E. coli isolate, cohort member or AST observation. Retrieve
The pinned upstream files are [`test_dna.fa`](https://raw.githubusercontent.com/ncbi/amr/amrfinder_v4.2.7/test_dna.fa),
[`test_prot.fa`](https://raw.githubusercontent.com/ncbi/amr/amrfinder_v4.2.7/test_prot.fa),
[`test_prot.gff`](https://raw.githubusercontent.com/ncbi/amr/amrfinder_v4.2.7/test_prot.gff)
and [`test_both.expected`](https://raw.githubusercontent.com/ncbi/amr/amrfinder_v4.2.7/test_both.expected).
The [NCBI usage guide](https://github.com/ncbi/amr/wiki/Running-AMRFinderPlus)
documents this installation test, and the [NCBI AMRFinderPlus repository](https://github.com/ncbi/amr)
contains its public-domain notice. Retrieve those four upstream files into an
ignored scratch directory, preserve each source URL and SHA-256, run the
installation test with the selected immutable database, and compare output
with the expected file. Record `amrfinder --database_version` and the database
file fingerprint. This control was run with software 4.2.7 and database
2026-08-07.1. AMRFinderPlus exited successfully and 25 of 26 expected rows
match. The sole difference is the `blaEC` closest-reference annotation:
expected `WP_063610930.1` (EC-15), observed `WP_498365732.1` (EC-28), with the
same `blaEC` symbol and class. A database-content difference is plausible but
has not been confirmed; this is a partial control result, not an exact-output
pass. The tracked v4.0.23 expected-report fixture remains parser input only.
The synthetic `tests/fixtures/tiny.fna` also completed a real caller no-hit
smoke with zero data rows; it is not a biological negative control, and no-hit
must not be reported as susceptibility.

On a host with the pinned environment and database, the upstream control can
be retrieved and checked without adding sequence data to Git:

```sh
control_dir=/private/tmp/amrfinderplus-control-4.2.7
mkdir -p "$control_dir"
for name in test_dna.fa test_prot.fa test_prot.gff test_both.expected; do
  curl -fLsS "https://raw.githubusercontent.com/ncbi/amr/amrfinder_v4.2.7/$name" \
    -o "$control_dir/$name"
done
shasum -a 256 "$control_dir"/* > "$control_dir/SHA256SUMS"
amrfinder --database "$database" --plus -n "$control_dir/test_dna.fa" \
  -p "$control_dir/test_prot.fa" -g "$control_dir/test_prot.gff" \
  -O Escherichia --print_node -o "$control_dir/test_both.actual"
diff -u "$control_dir/test_both.expected" "$control_dir/test_both.actual"

# Software no-hit smoke only; this synthetic sequence says nothing about AST.
amrfinder --database "$database" -n tests/fixtures/tiny.fna \
  -O Escherichia -o "$control_dir/no_hit.tsv"
test "$(wc -l < "$control_dir/no_hit.tsv" | tr -d ' ')" = 1
```

For every verified cohort genome, run and harmonize the caller separately;
use the exact file names below, with `pdt`, `fasta` and `database` set from the
manifest, retrieval provenance and immutable database location.

```sh
amrfinder --database_version
python3 bin/amrscan.py call --sample "$pdt" --input "$fasta" \
  --database "$database" --organism Escherichia \
  --log "datasets/ecoli_validation/calls/$pdt.log" \
  --raw "datasets/ecoli_validation/calls/$pdt.amrfinder.tsv" \
  --provenance "datasets/ecoli_validation/calls/$pdt.provenance.json"
python3 bin/amrscan.py harmonize --sample "$pdt" \
  --raw "datasets/ecoli_validation/calls/$pdt.amrfinder.tsv" \
  --provenance "datasets/ecoli_validation/calls/$pdt.provenance.json" \
  --output "datasets/ecoli_validation/calls/$pdt.harmonized.tsv"
```

## Evidence linkage and summary

`bin/amrfinder_validation.py summarize` links exact PDT → BioSample → verified
Assembly → caller result → harmonized determinant evidence → AST observations.
It verifies the downloaded assembly hashes, caller input hash, raw report hash,
sample ID, caller version and database fingerprint. AST rows are joined only
by exact versioned PDT accession, then checked against the cohort BioSample
and organism. Assembly conflicts fail closed. Genotype and AST are stored in
separate evidence tables with shared isolate keys; every AST record is kept.

The deterministic summary reports selected cohort size, valid assemblies,
successfully genotyped isolates, isolates with and without AMR determinant
calls, AST-linked isolates, exclusions and reasons, AMR determinant/gene counts,
caller element types, and AST observations grouped by the normalized submitted
phenotype, including ND/I/SDD/NS/HLAR/NA where present. No sensitivity,
specificity, F1, PPV, NPV, prediction or concordance is calculated. The
summary's provenance JSON fingerprints its manifest, genome provenance, AST
table and caller reports. Results are published to a new directory and never
overwrite an existing summary.

```sh
python3 bin/amrfinder_validation.py summarize \
  --manifest benchmark/cohorts/ecoli_amrfinder_validation.tsv \
  --genomes datasets/ecoli_validation/genomes \
  --ast datasets/ecoli_validation/ast_processed/ecoli_ast.tsv \
  --calls datasets/ecoli_validation/calls \
  --exclusions benchmark/cohorts/ecoli_amrfinder_exclusions.tsv \
  --outdir datasets/ecoli_validation/summary
```

## Validation status

| Stage | Status |
| --- | --- |
| Retrieval, linkage and summary implementation | Implemented. Identity/hash mismatches fail closed. |
| Offline tests | Passed using mocked NCBI packages, synthetic FASTA, upstream report rows and AST fixtures. These validate software behavior only. |
| Official Pathogen Detection lookup | Two preliminary candidate mappings were observed; complete AST rows for the candidate are not archived. |
| NCBI assembly package | `GCA_000770275.1` report identity, 92-sequence FASTA, NCBI MD5s and recorded SHA-256s verified. |
| AMRFinderPlus and database | Real binary 4.2.7 with database 2026-08-07.1; 162-file SHA-256 manifest passes. |
| Real genome call and provenance | `GCA_000770275.1` / `PDT000041778.1`: real call succeeded with 18 reported rows (17 AMR and 1 STRESS). Call provenance records the exact FASTA SHA-256, raw report SHA-256, caller version and database file hashes; harmonization completed. Database fingerprint: `sha256:748425a026512fda44008b9d106a4546518cc57dac59eb6432232eb3ac2c978b`. The candidate is not counted in the cohort without archived AST evidence. |
| Caller controls | NCBI upstream positive software control: 25/26 rows match, with one unresolved `blaEC` reference annotation difference. Synthetic no-hit smoke: zero data rows. No biological E. coli negative control is claimed. |
| AST snapshot, linked cohort and summary | Not validated. The tracked cohort manifest remains header-only; there are zero AST-linked isolates and no real cohort summary. |

Keep PR #4 in Draft until the archived AST observations can be linked to
accession-verified assemblies and the intended 3–10 isolate cohort can be
summarized. One real accession-to-caller-to-harmonized-result chain has now
been demonstrated, but it does not yet establish the AST-linked study.
