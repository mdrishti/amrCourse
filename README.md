# Clone this repository

If you don't already have Git installed:

```bash
# macOS
brew install git

# Ubuntu/Debian
sudo apt-get install git

```

Then clone this repo:

```bash
git clone <this-repo-url>
cd amrCourse
```

---

# Overview of the notebook
1. Query for papers about AMR on PubMed
2. Triage whether a paper has relevant information to be curated - using language models.

# Hands-on exercise: LLM triage in Colab

There are three notebooks, meant to be run in order:
1. `colab_01_query_corpus.ipynb` — build and clean the PMID corpus.
2. `colab_02_triage.ipynb` — triage a sample of papers from that corpus with an LLM.
3. `colab_03_curation.ipynb` — structured extraction (strains/genes/phenotypes) from papers that pass triage.

Open each notebook in Colab and make your own copy (File > Save a copy in Drive) before starting, so your edits don't collide with anyone else's.

### Running it locally instead (optional)

If you'd rather run the notebooks on your own laptop than in Colab (e.g. to use the local-model
section's MLX backend on Apple Silicon), this repo ships a `Pipfile` with everything the notebooks need:

```bash
pip install pipenv          # if you don't already have it
cd amrCourse
pipenv install              # creates a venv with biopython, pandas, llama-cpp-python, etc.
                             # -- mlx-lm is pulled in automatically only on Apple Silicon
pipenv run jupyter notebook colab_01_query_corpus.ipynb
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
3. Get a free Gemini API key from Google AI Studio: https://aistudio.google.com/api-keys. The notebook calls the Gemini API directly (no router/middleman), so this one key is all you need for the language-model side.

### Initial steps

#### Option A: Google Colab
1. Open `colab_01_query_corpus.ipynb` in Google Colab. Click the key icon in the left sidebar (Secrets) and add secrets named `GEMINI_API_KEY`, `NCBI_API_KEY` and `ENTREZ_EMAIL` with your keys and email as the values. This keeps them out of the notebook file and out of your clipboard history.
2. Run the setup cell at the top (installs the needed packages, then reads your keys and email).

#### Option B: Jupyter on your own laptop
1. Follow "Running it locally instead" above to install the dependencies and create your `.env` file with `GEMINI_API_KEY`, `NCBI_API_KEY` and `ENTREZ_EMAIL`.
2. Open `colab_01_query_corpus.ipynb` with `pipenv run jupyter notebook colab_01_query_corpus.ipynb` and run the setup cell at the top. It installs the needed packages, then reads your keys and email from `.env`.

### Query PMC to get initial corpus of papers.
3. The next cell builds the search query used to find the paper corpus. Look at the search terms, and add or remove terms to make the list of PMIDs wider or narrower. For instance, you can add a genus name to focus on specific bacterial species or strains.
4. The query also excludes some article types from the results: review, case reports, letter, comment, address, autobiography, biography, conference proceedings, editorial, interactive tutorial, interview, introductory journal article, personal narrative, portrait, news.
5. Think about how far back in time you want to search for papers. The example uses the range 2015–2023. You can change this range to widen or narrow your search.
6. A later cell in this notebook removes more papers from the list: retraction notices and expression-of-concern notices. These are notices about papers that were later found to have problems, so you would not want to pull information from them. These notices also point to the PMID of the original paper being retracted, so that original paper is removed too.

### Triage using language models
Open `colab_02_triage.ipynb` next.
7. One cell defines the triage questions directly in the notebook, as plain lists you can edit. These questions guide the language model: Is this paper about bacteria? Is a gene or genotype named? Does it report a result about antibiotic resistance?
8. The next cells set up the call to the Gemini model and fetch the title and abstract for a small set of sample papers. Run the fetch cell before triaging anything — the triage step needs the paper text it collects.
9. Use the questions to run triage on one or more papers ("Run triage on one paper").

You should see each question's answer and reasoning print as it runs, then a final result like:
```
{'is_bacteria': True, 'has_gene_genotype': True, 'has_amr_phenotype': True}
```

### Run triage across all sample papers and tabulate ("Run triage across several papers and tabulate")

10. Run that section. It loops over all 9 sample papers and builds a table. Record your own read of each result in a table like this one, filling in the PMIDs printed in the notebook:

| PMID | is_bacteria | has_gene_genotype | has_amr_phenotype | Your gut check — do you agree? |
|---|---|---|---|---|
| | | | | |

**Question to think about while you do this:** for any paper where you disagree with the model, look at the printed reasoning from "Run triage on one paper" (re-run that paper individually with the verbose option if you skipped straight to the tabulated section) — is the model wrong about the biology, or is the *question wording* ambiguous for this paper?

### Edit a question and re-run

11. Find the question about antibiotic resistance in the cell that defines the triage questions, and edit its wording — narrower or broader, your choice. Re-run that cell, then re-run "Run triage on one paper" or the tabulated section on the same paper(s). Does the answer or the reasoning change? (This only changes the question for the rest of your current session. If you run the notebook again from the top, it goes back to the original wording.)

**Watch the free-tier cap**: Gemini's free tier limits how many requests you can make per minute and per day (check current limits at https://ai.google.dev/gemini-api/docs/rate-limits). Each triage run on one paper uses a few requests, so a handful of papers plus a re-run or two is plenty for one session.

### Triage a full paper, using the right section each time

12. So far every question has seen the same title and abstract text. There's a later cell in the same notebook that instead gives each question only the part of the paper it actually needs — for example, a question about methods only sees the methods section — instead of hoping the answer happens to be mentioned in the abstract.

This fetches the paper's full text straight from PubMed Central — no files to download or upload, just an online lookup:
1. First it checks whether a paper has a linked full-text record. Not every paper does — if a paper doesn't have one, try a different paper.
2. Then it fetches that paper's full text and splits it into sections (title, abstract, methods, results, and so on).
3. Then it runs the same set of questions, but each one only sees its own relevant section(s) instead of one abstract-only blob of text.

Try this on the first sample paper, then try it on each of your other sample papers too — some will have no full-text record available (closed access or metadata-only), which is expected. For any paper that works, compare its result here to what you got from the abstract-only triage on the same paper. Did seeing the full text change any answer?

### Discuss

Bring back to the group:
- Any paper where your triage answer surprised you.
- Whether you hit the Gemini rate limit.
- One triage question you'd want to add or reword for your own research area — in the notebook this just means editing the questions directly.
- Whether abstract-only and full-text triage agreed, for anyone who tried the full-text step.
