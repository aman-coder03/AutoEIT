# AutoEIT — Automated EIT Scoring System

GSoC 2026 test submission for the AutoEIT project (University of Alabama + NIU).

This repo contains my solution for Test II: implement a reproducible script that applies the meaning-based EIT rubric to transcribed learner responses and outputs sentence-level scores.

---

## What's in here

- `autoeit_scoring.ipynb` — the main notebook, start here
- `autoeit_scoring_executed.ipynb` — same notebook with all outputs already run
- `autoeit_scoring_with_output.html` — HTML export of the executed notebook (easier to view without Jupyter)
- `AutoEIT_Sample_Transcriptions_for_Scoring.xlsx` — input file with transcriptions (blank scores)
- `AutoEIT_Sample_Transcriptions_for_Scoring_SCORED.xlsx` — output file with scores filled in

---

## How to run it

Put `AutoEIT_Sample_Transcriptions_for_Scoring.xlsx` in the same folder as the notebook, then:

```bash
pip install pandas openpyxl rapidfuzz scikit-learn matplotlib seaborn
jupyter notebook autoeit_scoring.ipynb
```

Run all cells top to bottom. The scored output file gets written automatically to the same folder.

Python 3.10+ recommended. No GPU needed, everything runs on CPU.

---

## Approach

The rubric (Ortega, 2000) is a meaning-based ordinal scale from 0 to 4. Rather than using an LLM (which the project description points out gives inconsistent scores), I went with a rule-based pipeline that encodes the rubric criteria directly. Same input always gives same output, and every score has a reason string explaining why it was assigned.

The main steps are:

**Preprocessing** strips false starts like `[word-]`, removes gap markers like `[...]`, flags non-Spanish responses (`[en inglés]` → score 0 immediately), strips the word-count annotations from stimulus strings like `(7)`, and applies the two rubric exceptions: `muy` insertions/deletions don't affect the score, and `y`/`pero` substitutions are treated as equivalent.

**Scoring** runs as a decision cascade. First check for score 0 conditions (empty, silence, only 1 word, only function words). Then check for score 4: the transcription needs fuzzy string similarity >= 0.95 AND fuzzy content word overlap >= 0.92 AND no new content words added. Then score 3 if content overlap >= 0.80, score 2 if >= 0.45, score 1 if >= 0.15 with at least 2 content words, and score 0 otherwise.

The key thing that makes this work better than simple word matching is **fuzzy word-level matching** for the content overlap calculation. Each word in the stimulus is matched against the best fuzzy match in the transcription (using character-level edit distance, threshold 80%). This handles things like `preseguido` matching `perseguido`, or `ayudara` matching `ayudaría`, which a human rater would score as a near-exact match but exact string comparison would miss entirely.

---

## Results

Evaluated against 1,560 human-rated utterances across 52 participants from the full EIT dataset:

| Metric | Result | Target |
|--------|--------|--------|
| Exact agreement | 79.6% | >= 90% |
| Adjacent agreement (within 1 point) | 98.0% | |
| Cohen's Kappa | 0.620 | |
| Mean total score difference per participant | 4.6 pts | < 10 pts |
| Participants within 10-point difference | 92.3% | |

The total score targets from the project spec are comfortably met. The gap to 90% exact agreement sits almost entirely at the 3/4 boundary (around 136 cases out of 1,560). These are cases where a grammar change is subtle enough that a human rater still scores it 4, but it's not a verbatim match either. Things like tense shifts (`fue` to `era`) or subject-verb agreement (`encantan` to `encanta`) fall in this grey area. Resolving these correctly needs semantic understanding of whether a grammar change actually shifts the meaning, which is beyond what lexical features alone can do. A multilingual sentence encoder like LaBSE or paraphrase-multilingual-MiniLM would be the natural next step, and that's essentially what the full GSoC project is asking for.

---

## Files generated

The notebook writes `AutoEIT_Sample_Transcriptions_for_Scoring_SCORED.xlsx` to the working directory. It's a copy of the input file with four new columns added to each participant sheet: `Auto_Score`, `Scoring_Reason`, `Fuzzy_Sim`, and `Content_Overlap`. The score cells are colour-coded (green for 4, yellow for 2, red for 0 etc.) to make it easy to spot patterns at a glance.

---

Aman Srivastava  
amansri345@gmail.com  