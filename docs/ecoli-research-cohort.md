# E. coli AST research cohort builder

`bin/ecoli_cohort.py` builds an auditable, phenotype-only isolate and
isolate–antimicrobial census from a validated raw NCBI Pathogen Detection AST
snapshot. It reuses `bin/ast_dataset.py` for archive validation, normalization,
and source joins. It does not call resistance genes, infer phenotype from
genotype, or fit prediction models.

## Current census status

The archived metadata candidate inventory retrieved on 2026-10-11 contains
21,042 rows but only 10,521 distinct versioned PDT accessions: exactly two
metadata rows per PDT from consecutive Pathogen Detection pipeline runs. These
are isolate metadata candidates, not AST observations. The export query was
`scientific_name:"Escherichia coli" AND AST_phenotypes:* AND asm_acc:GCA_*`;
the archived source file is
`datasets/ncbi_ast_manual/ecoli_ast_assembly_candidate_list_20261011/isolates_candidates.tsv`
with SHA-256
`66eee1964b4777086cc32e5d0aeb914cbfa5bcf9f8f324593deae5d28ae3f9e1`.
The export metadata and helper outputs are local and Git-ignored.

The matching full AST observation export for these candidates could not be
retrieved because NCBI requests failed DNS resolution in the current shell.
Therefore antimicrobial, country, year, source, conflict, missingness, and
eligible-cohort population summaries have **not** been computed. The five
isolates in the earlier AMRFinder validation are a software smoke dataset, not
this research census. No population size or study-drug panel is inferred from
the candidate inventory. The configuration leaves publication thresholds at
`HUMAN_REVIEW_REQUIRED`.

## Rebuild from a complete archived AST snapshot

First archive and validate the full AST and metadata exports using the data
builder guide and this source-side query (or a documented, narrower E. coli
query with identical archived provenance):

```bash
python3 bin/ast_dataset.py download \
  --query 'scientific_name:"Escherichia coli" AND AST_phenotypes:* AND asm_acc:GCA_*' \
  --outdir datasets/ecoli_research_v1/raw
python3 bin/ast_dataset.py validate \
  --snapshot datasets/ecoli_research_v1/raw
python3 bin/ecoli_cohort.py build \
  --snapshot datasets/ecoli_research_v1/raw \
  --config benchmark/cohorts/ecoli_research_v1/cohort_config.json \
  --outdir datasets/ecoli_research_v1/processed
```

Output directories are immutable: choose a new path for each source snapshot or
configuration version. The output manifest records source URLs, timestamps,
hashes, parser and schema versions, configuration hash, builder version and the
code commit. `SHA256SUMS` covers the generated tables and manifests. Rebuilding
the same snapshot with the same code and configuration yields deterministic
tables; the recorded time is the latest source retrieval time, not build wall
clock time.

## Observation and eligibility rules

All normalized AST observations for E. coli are preserved in
`ast_observations.tsv`, including repeated tests, discordant results, nonbinary
categories, missing results, and unknown antimicrobials. A repeated set of the
same recognized R or S result becomes one derived isolate–drug endpoint with
`repeated_consistent` status; the original rows remain available. R/S discordance
is `CONFLICT`; mixtures of binary and nonbinary values are
`NON_BINARY_OR_CONFLICT`. Neither enters the binary endpoint denominator.
Intermediate, SDD, NS, HLAR, ND, missing, and unrecognized values are retained
and counted separately; none is silently converted to R or S.

The versioned config defines a binary endpoint as a recognized antimicrobial
with one unambiguous R or S result on a versioned PDT accession. Overall
eligibility additionally requires a valid BioSample, a versioned GenBank GCA
assembly, E. coli identity, and no identity disagreement across linked source
records. Country, collection year, and known source category are separate
analysis-specific eligibility dimensions. The output reports explicit reason
codes instead of filling unknown metadata from genotype or nearby records.
Study-drug sufficiency thresholds require human review and remain unset.

## Outputs

The builder emits normalized AST observations, candidate isolates,
isolate–drug endpoints, eligible isolates, analysis-specific exclusions,
antimicrobial/country/year/source summaries, missingness and conflict tables,
a reconciled cohort flow, and a provenance manifest. Counts distinguish
observations, isolates, and isolate–drug endpoints. Summary denominators are
documented in column names; repeated observations never increase the number of
isolates.

The separate `inventory` command can summarize an archived metadata-only
candidate list. Its result explicitly sets the AST observation count to null
and labels the endpoint census unavailable. It must not be used as a substitute
for the full `build` census.

See [the schema contract](../schemas/ecoli-research-cohort-schema.md),
[configuration](../benchmark/cohorts/ecoli_research_v1/cohort_config.json),
and [tests](../tests/README.md#e-coli-research-cohort-builder).
