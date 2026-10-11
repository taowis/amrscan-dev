# E. coli research cohort configuration

`cohort_config.json` is version 1.0.0 of the phenotype-only E. coli cohort
rules. It requires a versioned Pathogen Detection accession, identity-consistent
BioSample and GenBank assembly links for overall eligibility, and one
unambiguous R or S endpoint per recognized antimicrobial. Repeated consistent
AST observations are collapsed only for the derived endpoint; every observation
is retained. Discordance and nonbinary values remain explicit and are excluded
from the binary denominator.

Study-drug panel thresholds are intentionally unset pending scientific review of
the population census. See [the cohort methods and current census status](../../../docs/ecoli-research-cohort.md).

`population_config.json` is version 1.0.0 of the population-structure rules used
by `bin/population_structure.py`: genome QC bounds, the MLST scheme, the
phylogroup method, the lineage definition, holdout eligibility, the provisional
holdout screening rules and the review-flag thresholds. None of these values
were chosen with reference to phenotype or model results. See
[the population structure methods and census](../../../docs/ecoli-population-structure.md).
