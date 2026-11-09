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

### Running it locally instead 

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

### Run triage across all sample papers and tabulate

10. Run that section. It loops over all 9 sample papers and builds a table. Record your own read of each result in a table like this one, filling in the PMIDs printed in the notebook:

| PMID | is_bacteria | has_gene_genotype | has_amr_phenotype | How does it compare against manual triage? |
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

### Trying a local, open weights model for triage

There's an optional section near the end of this notebook for running an open-weights model on
your own computer instead of calling Gemini online — useful for comparing results, or for running
triage without an internet connection. It's off by default: nothing downloads or uses your
computer's memory unless you deliberately switch it on in that section. If you try it, you can
also turn on `ENABLE_LOCAL_THINKING` there to see the model's step-by-step reasoning for each
question, which can help explain a surprising answer.

### Discuss

Bring back to the group:
- Any paper where your triage answer surprised you.
- Whether you hit the Gemini rate limit.
- One triage question you'd want to add or reword for your own research area — in the notebook this just means editing the questions directly.
- Whether abstract-only and full-text triage agreed, for anyone who tried the full-text step.

### Curation: pulling out the actual facts

Open `colab_03_curation.ipynb` next.

13. Triage only answers yes/no questions. This notebook goes further: for a paper that already passed triage, it pulls out the real facts — which bacteria were studied, which genes were found, and what resistance result was measured. It does this in three steps, one after another: strains, then genes, then phenotypes. A last step checks all three together and flags anything that looks off.
14. Like the triage questions, the instructions for each step are written directly in the notebook as plain text, so you can edit them the same way.
15. Run curation on one sample paper first ("Try it"). You'll see a small table for strains, a table for genes, and a table for phenotypes.
16. Then run it across all sample papers ("Run curation across all sample papers"). Look at how many rows end up in each table. It's normal for the phenotypes table to have fewer rows than the genes table — not every gene has a measured resistance result reported in the paper.

### Antibiotic names get looked up, not guessed

17. Each gene and phenotype row tries to record which antibiotic it relates to, along with a standard reference code for that antibiotic. This code is not a guess from the language model — it's looked up from a reference file that comes with this repo, so it's accurate.

### Extra data hiding in supplementary files

18. Some papers put their real results in a separate supplementary file instead of the main text — an extra table or spreadsheet attached to the paper. Further down in the notebook ("Supplementary table mining") there's a step that checks a paper for these extra files, fetches them, and curates them the same way as the main text. Turn it on, then re-run the full pipeline to see if it picks up anything new.

### Trying a local, open weights model

19. Like the triage notebook, there's an optional section near the end for running an AI model on your own computer instead of calling Gemini online. This is **off by default** — the notebook will not download anything or use your computer's memory unless you deliberately switch it on in that section. Only try this if you want to compare it with Gemini, or want to run curation without an internet connection.

### Discuss

Bring back to the group:
- How many strains, genes, and phenotypes you ended up with for your sample papers.
- Any gene marked as causing resistance that had no matching phenotype row — what do you think happened there?
- Whether turning on supplementary file mining changed any of your results.
