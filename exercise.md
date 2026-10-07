# Overview of the notebook
1. Query for papers about AMR on PubMed
2. Triage whether a paper has relevant information to be curated - using language models.

# Hands-on exercise: LLM triage in Colab

Open the notebook in Colab and make your own copy (File > Save a copy in Drive) before starting, so your edits don't collide with anyone else's.

### Running it locally instead (optional)

If you'd rather run the notebook on your own laptop than in Colab (e.g. to use section (i)'s MLX
backend on Apple Silicon), this repo ships a `Pipfile` with everything the notebook needs:

```bash
pip install pipenv          # if you don't already have it
cd amrCourse
pipenv install              # creates a venv with biopython, pandas, llama-cpp-python, etc.
                             # -- mlx-lm is pulled in automatically only on Apple Silicon
pipenv run jupyter notebook colab_triage_workshop.ipynb
```

Secrets work differently outside Colab — there's no Secrets manager, so the notebook falls back to
a `.env` file in this directory (loaded via `python-dotenv`, already in the `Pipfile`). Create one
with:
```
GEMINI_API_KEY=...
ENTREZ_EMAIL=...
NCBI_API_KEY=...
```
and make sure `.env` is gitignored before you commit anything.

## Setup

### PubMed
1. Make an account on NCBI here https://account.ncbi.nlm.nih.gov/signup/?back_url=https%3A%2F%2Fwww.ncbi.nlm.nih.gov%2F
2. Go to the settings on your NCBI account https://account.ncbi.nlm.nih.gov/settings/ and create an API key.

### Language models
3. Get a free OpenRouter API key: https://openrouter.ai/keys. Openrouter is a service provider that operates a platform for accessing and routing requests to large language models. For some models accessible through their free router (https://openrouter.ai/openrouter/free). The list of these models changes frequently. For this workshop, you can use gemini models - google/gemma-4-31b-it:free and google/gemma-4-26b-a4b-it. 

If you would like to use API keys of other language model platforms, you can check here (https://openrouter.ai/models) whether that model is accessible through Openrouter and plug-in the key here (https://openrouter.ai/workspaces/default/byok) in your Openrouter account to access it programatically.
For this workshop, it is worth getting an API key for gemini on Google AI studio here (https://aistudio.google.com/api-keys) and plugging that in Openruter account settings to access gemini models. While you can also use gemini models without Openrouter, the latter makes it easier to access any model you want and not just the ones provided by gemini.


### Initial steps
4. Open the notebook in Google Colab. Click the key icon in the left sidebar (Secrets) and add secrets named `OPENROUTER_API_KEY`, `NCBI_API_KEY` and `ENTREZ_EMAIL` with your repective keys and email-id as the value. This keeps it out of the notebook file and out of your clipboard history. 
5. Run the notebook's setup cells in section (a) (`pip install biopython requests pandas`, then the API key / email cell).
6. Run section (b) — this downloads the triage flowchart/prompts and reference-data CSVs from a web link and checks each one's checksum before using it. You should see `✓ <filename> downloaded and checksum-verified` for all 4 files.

### Query PMC to get initial corpus of papers.
7. Section (c) constructs the AMR search query used to build the paper corpus. Think about the terms, add/substract terms you think can be useful to expand/narrow the list of PMIDs. For instance, you can add a genus name to focus on specific bacterial species/strains. 
You can also play with a set of random 200 papers, analyze what phrases were reported the most, and finally include the most reported terms in the query to improve you PMC search.
8. Exclusion of papers - part 1: the search query's `PUBTYPE_EXCLUSIONS` term (section c) excludes these article types from the result: review, case reports, letter, comment, address, autobiography, biography, conference proceedings, editorial, interactive tutorial, interview, introductory journal article, personal narrative, portrait, news.
9. Think about how long would you like to go back in time to get the relevant papers. The current example has the range of 2015-2023. You can change the range to expand/narrow your search.
10. Section (d) lists a flow to exclude more papers from the initial cohort. There are papers of type retraction/expression-of-concern notices. You would not want to curate information from such papers where the scientific integrity is not optimal. These notices also contain metadata (e.g.: PMID of the orginal paper that is retracted). This second pass excludes the PMIDs of the notices and the concerned original papers.

### Triage using language models
11. Section (e) loads the triage question set from the downloaded JSON files into `TRIAGE_PROMPTS` (the questions) and `TRIAGE_FLOWCHART` (the order/branching). These together guide the language model through the questions - 
Is this paper about bacteria? Is a gene/genotype named? Does it report a phenotype? is that phenotype AMR-specific?
12. Section (f) sets up the OpenRouter call and the flowchart walker (`call_llm`/`run_triage`). Section (g) fetches title+abstract for a small set of sample PMIDs via `Bio.Entrez` — run this before triaging anything, since `run_triage` needs the `papers` dict it builds.
13. Use the schema to run triage on one or more papers ("Run triage on one paper").


You should see each question's answer and reasoning print as it runs, then a final dict like:
```
{'is_bacteria': True, 'has_gene_genotype': True, 'has_phenotype': True, 'has_amr_phenotype': True}
```

## 3. Run triage across all sample papers and tabulate ("Run triage across several papers and tabulate")

Run that section. It loops `run_triage` over all 4 sample PMIDs and builds a pandas table. Record your own read of each result:

| PMID | is_bacteria | has_gene_genotype | has_phenotype | has_amr_phenotype | Your gut check — do you agree? |
|---|---|---|---|---|---|
| 12737743 | | | | | |
| 12800222 | | | | | |
| 12927083 | | | | | |
| 12059906 | | | | | |

**Question to think about while you do this:** for any paper where you disagree with the model, look at the printed reasoning from "Run triage on one paper" (re-run that paper individually with `verbose=True` if you skipped straight to the tabulated section) — is the model wrong about the biology, or is the *question wording* ambiguous for this paper?

## 4. Edit a prompt and re-run

After section (e) has run (so `TRIAGE_PROMPTS` is loaded), edit `TRIAGE_PROMPTS["has_amr_phenotype"]["prompt"]` directly in a new cell — narrower or broader, your choice — then re-run "Run triage on one paper" or the tabulated section on the same paper(s). Does the answer or reasoning change? (This only changes the in-memory dict for your session, not the published file on disk — to make the change stick for everyone, it would need to go back into `workshop/release_data/prompts_triage_amr_workshop.json` and get re-published.)

**Watch the free-tier cap**: OpenRouter's free tier allows 50 requests/day *total on your key*, shared across every free model — each `run_triage` call uses up to 4 requests, so a handful of papers plus a re-run or two is plenty for one session.

## 5. Triage a full paper, section-aware (Section h)

So far every triage question has seen the same title+abstract text. Section (h) instead routes each
question to only the paper section(s) listed in its `target_section` (the IAO ontology codes already
present in `TRIAGE_PROMPTS`, loaded back in section (e)) — giving each question the actual section of
the paper it needs, instead of hoping the answer happens to be mentioned in the abstract.

This fetches the paper's full text straight from PMC — no files to download or upload, just NCBI:
1. `pmid_to_pmcid(pmid)` looks up whether a PMID has a linked PMC full-text record. Not every paper
   does — a `None` result means try a different PMID.
2. `fetch_sections_from_pmc(pmcid)` fetches that record's full-text XML and splits it into sections.
3. `run_triage_sectioned(sections)` runs the same flowchart, but each question only sees its own
   relevant section(s) instead of one abstract-only blob.

Run the "Try it" cell on `sample_pmids[0]`, then try it on each of your other sample PMIDs too (just
change the `pmid = ...` line) — some will have no PMC full-text record (closed access or
metadata-only), which is expected. For any PMID that works, compare its result here to what you got
from the abstract-only `run_triage` on the same paper. Did seeing the full text change any answer?

## 6. Discuss

Bring back to the group:
- Any paper where your triage answer surprised you.
- Whether you hit the OpenRouter rate limit.
- One triage question you'd want to add or reword for your own research area — in the notebook this is a `TRIAGE_PROMPTS`/`TRIAGE_FLOWCHART` dict edit.
- Whether abstract-only and section-aware full-text triage agreed, for anyone who tried Section (h).
