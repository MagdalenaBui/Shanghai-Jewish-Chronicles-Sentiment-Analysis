# Shanghai Jewish Chronicle – OCR & Sentiment Analysis Pipeline

This repository contains two Jupyter notebooks that together turn scanned
issues of the *Shanghai Jewish Chronicle* into a document-level sentiment
and topic analysis, as part of the Hausarbeit "Shanghai als Zufluchtsort
1930–1949".

## Contents

- `OCR.ipynb` — OCR & article extraction
- `sentiment_analyse.ipynb` — Sentiment analysis & topic modelling


## Pipeline overview

```
raw PDF  →  [1] OCR.ipynb  →  articles_extracted.csv
                                                              ↓
                                            [2] sentiment_analyse.ipynb
                                                              ↓
                              yearly sentiment trends, ranked articles,
                                topic–emotion correlations, plots, CSVs
```

---

## 1. `OCR.ipynb` – OCR & Article Extraction

**What it does:** Takes the scanned newspaper PDF and produces a clean CSV
of individual articles (no advertisements), one row per article.

**Steps:**
1. Renders each PDF page as a 300-DPI image (PyMuPDF).
2. Runs Tesseract OCR with a combined German+English language model,
   producing word-level output with position/font-size data.
3. Groups words into lines/paragraphs and detects headlines via
   font-size/line-spacing heuristics to segment the page into articles.
4. Cleans text (rejoins hyphenated line breaks, etc.).
5. Filters out advertisements (regex heuristics: pricing, "Chiffre",
   phone numbers, high digit ratio) and flags wire-agency content
   (Reuter, Transocean, Havas, D.N.B.).
6. Extracts the publication year from page headers, tolerant of common
   OCR misreads (e.g. "1" read as "i"/"l").
7. Processes pages in parallel with per-page timeouts and checkpointing,
   so a run can be resumed/monitored on large PDFs (~200+ pages).

**Input:** the raw newspaper PDF (path set near the top of the notebook).

**Output:** `articles_extracted.csv` with columns `source_file`,
`pdf_page`, `year`, `month`, `lang`, `has_agency_marker`, `word_count`,
`text`.
---

## 2. `sentiment_analyse.ipynb` – Sentiment Analysis & Topic Modelling

**What it does:** Scores each article for emotion/sentiment using the
NRC Emotion Lexicon, aggregates results by year, and explores how
emotions relate to thematic content.

**Steps:**
1. Loads `articles_extracted.csv` and cleans it (year filter, junk-text
   filter, removes leading OCR digit artifacts).
2. Detects article language (German/English).
3. Loads the multilingual NRC Emotion Lexicon and transliterates German
   entries to the corpus's historical spelling (ä→ae, ö→oe, ü→ue, ß→ss).
4. Tokenizes and lemmatizes text with spaCy (language-specific model),
   keeping only content words (nouns, proper nouns, verbs, adjectives,
   adverbs), with a cross-language-safe stopword filter that protects
   real lexicon words (e.g. English "war"/"die" are not stripped just
   because they're also German stopwords).
5. Improves lexicon coverage via:
   - fuzzy matching (RapidFuzz) for OCR spelling variants,
   - German compound-word splitting (nouns only, high-confidence splits).
6. Scores each article per emotion, both as a **raw score** (emotion
   words / all words) and a **coverage-normalized score** (emotion
   words / matched words only), to separate real sentiment trends from
   declining OCR/lexicon coverage over time.
7. Aggregates scores by year and plots trends.
8. Ranks articles within each emotion category (top Fear, Joy, etc.).
9. Reports NRC coverage diagnostics by year/language.
10. Compares agency (wire) vs. editorial content.
11. **Topic modelling, approach 1:** TF-IDF on top-scoring articles per
    emotion to extract characteristic vocabulary (exploratory; found
    less thematically coherent).
12. **Topic modelling, approach 2:** manually defined bilingual keyword
    categories (diaspora groups, occupying administrations, aid
    organizations, core themes), tested for correlation with emotion
    scores via Mann-Whitney U tests, rank-biserial effect size, and
    Benjamini-Hochberg FDR correction (categories with <30 matches are
    excluded from testing).

**Input:** `articles_extracted.csv` (from notebook 1) and the NRC
Emotion Lexicon file (`NRC-Emotion-Lexicon-v0.92-InManyLanguages-web.xlsx`).

**Output:** yearly sentiment plots, `articles_with_sentiment.csv`,
top-articles-per-emotion CSVs, `emotion_topic_words.csv`,
`topic_keyword_counts.csv`, a topic–emotion correlation heatmap.

---

## Setup

```bash
pip install pymupdf pytesseract pandas spacy rapidfuzz openpyxl \
    scikit-learn statsmodels matplotlib
python -m spacy download de_core_news_sm
python -m spacy download en_core_web_sm
```
Tesseract OCR must also be installed system-side with German+English
language data (`deu`, `eng`).


## Notes / limitations

See the Hausarbeit's "Discussion of Methods" section for a full
discussion of limitations (OCR quality/ad filtering, NRC lexicon vs.
a trained ML sentiment model, keyword-based topic correlation vs. full
LDA topic modelling).
