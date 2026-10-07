# Biology pointers for the workshop

For framing Section 1 (why these specific triage questions) and answering questions during the hands-on section. Not a slide-by-slide script — background for you.

## Why these five questions, in this order

`data/flowchart_triage_amr.json` asks, in sequence: is_about_bacteria → has_genetic_variants → has_gene_genotype → has_phenotype_data → has_amr_phenotype. That order mirrors how a domain expert would actually skim a paper: organism first (wrong kingdom kills relevance immediately), then whether there's any genotype signal at all, then whether that genotype signal is phenotypically linked, then whether the phenotype is specifically resistance-related. It's worth saying this explicitly — the flowchart isn't arbitrary, it encodes a triage heuristic a curator already uses.

The workshop's Colab notebook uses a trimmed 4-question version of this (is_about_bacteria → has_gene_genotype → has_phenotype_data → has_amr_phenotype), dropping only `has_genetic_variants` — the variant-path check. `has_gene_genotype` is kept because it's the AMR-path genotype check, not a variant check: it asks whether a gene/genotype is *named* at all, regardless of whether a specific mutation/variant is described. Worth mentioning explicitly if students ask why the notebook's flowchart is shorter than the repo's 5-question default.

## Genotype-phenotype linkage, briefly

The full mining pipeline (not triage) is trying to extract statements like "strain X carries variant Y in gene Z, and is resistant to antibiotic W." Triage's job is only to filter papers where such a statement is *plausible* — it doesn't try to extract or link the actual (gene, variant, phenotype) triple yet. If students ask "why doesn't triage just extract the answer directly," that's the right question — it's a cost tradeoff: triage questions are single boolean calls per section, string extraction/linking is multiple structured calls per candidate entity.

## What counts as an AMR phenotype (for has_amr_phenotype)

Talking points if students ask what should/shouldn't trigger "yes":
- MIC (minimum inhibitory concentration) values, disk diffusion zone sizes, resistant/intermediate/susceptible (R/I/S) calls — yes.
- A gene *named* as a resistance gene (e.g. *blaTEM*, *mecA*, *gyrA* mutation) without any reported susceptibility testing — this is the ambiguous case worth discussing; the current prompt wording is what determines whether it's yes or no, which is exactly the kind of edge case that motivates Section 5 (editable prompts).
- Resistance in a host organism or an unrelated control strain, not the study bacterium — no (the prompt for `has_genetic_variants` already explicitly excludes this case for variants; worth checking whether `has_amr_phenotype` handles it the same way, and if not, that's a good candidate edit for the group to try in the hands-on section).

## Why section-aware triage (Section h) can disagree with abstract-only triage

Each question in `TRIAGE_PROMPTS` carries a `target_section` list (IAO ontology codes for TITLE,
ABSTRACT, METHODS, RESULTS, ...). Section (h) of the workshop notebook uses this to show each question
only the section(s) it's actually about, instead of the flat title+abstract blob used everywhere else
— `is_about_bacteria` can be answered from the title/abstract alone, but `has_amr_phenotype` often
can't: MIC values, R/I/S calls, and resistance-gene expression data are reported in Results/Methods and
routinely left out of the abstract for space.

This is a good biology talking point in its own right: a paper can be a genuine AMR-phenotype paper
and still answer "no" under abstract-only triage simply because the abstract doesn't mention
resistance explicitly, even though Results does. If students compare the same paper's abstract-only
and section-aware answers in Section (h) and get different results, that's expected, not a bug — it's
the gap abstract-only triage accepts as a speed/cost tradeoff (see "Genotype-phenotype linkage,
briefly" above for the same tradeoff reasoning at the extraction level).

## Picking sample papers with known variety

The notebook's `sample_pmids` list (section g) fetches title+abstract live from PubMed via `Bio.Entrez` — there's no local full-text corpus involved in the Colab version. When choosing/updating that PMID list, aim for a spread rather than all-positive:
- A clear salmonella AMR paper (positive on all four triage questions).
- A paper on a different, off-topic-for-AMR organism — good for checking is_about_bacteria still says yes (it's still bacteria) even though it's off-topic for a Salmonella-AMR-specific downstream use — this distinguishes "is this about bacteria" from "is this about the AMR biology we care about," which is a useful distinction to draw out live.
- If you can find one, a paper that mentions a resistance gene by name but reports no susceptibility phenotype (tests has_amr_phenotype boundary above).

## Field-specific terms students may ask about

- **MIC / disk diffusion / R-I-S**: standard antimicrobial susceptibility testing readouts — the "phenotype" side of genotype-phenotype.
- **Efflux pump gene**: a common non-point-mutation resistance mechanism (e.g. AcrAB-TolC in Enterobacteriaceae) — worth a one-line mention since `has_gene_genotype`'s prompt explicitly calls out efflux pump genes as an example.
- **Lineage/genotype designation** (e.g. sequence type, clonal complex): counts under `has_gene_genotype` even without a specific point mutation — a strain-typing label is still genotype information for this pipeline's purposes.
