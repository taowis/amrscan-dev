# E. coli research cohort configuration

`cohort_config.json` is version 1.0.0 of the phenotype-only E. coli cohort
rules. It requires a versioned Pathogen Detection accession, identity-consistent
BioSample and GenBank assembly links for overall eligibility, and one
unambiguous R or S endpoint per recognized antimicrobial. Repeated consistent
AST observations are collapsed only for the derived endpoint; every observation
is retained. Discordance and nonbinary values remain explicit and are excluded
from the binary denominator.

Study-drug panel thresholds are intentionally unset pending review of a complete
population census. See [the cohort methods and current census status](../../../docs/ecoli-research-cohort.md).
