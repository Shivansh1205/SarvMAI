
# Fast Rule-Based Word Normalizer & Matcher

## Overview

This project maps noisy / misspelled / dialectal romanized input words (e.g., from Hindi, English, South Indian transliterations) to their canonical correct forms from a reference dictionary.  
It does **not** use pretrained models—matching is based on normalization, edit distance (Levenshtein), a **BK-tree** for fast nearest neighbors, and a **k-gram fallback** for robustness.

Example:  
`Raaam`, `ramm`, `shakttii` → `Ram`, `Shakti` etc.

## Features

- Language-aware normalization (vowel collapsing, transliteration tweaks like `w→v`, `ph→f`, etc.)
- Efficient approximate string matching using a **BK-tree** (edit distance metric)
- Fallback candidate seeding via **k-gram overlap** when BK-tree misses
- Deterministic, thresholdable matching (no ML/fine-tuning)
- Easy to evaluate: outputs gold vs predicted for accuracy proof
- Lightweight: per-word lookup is fast (sub-millisecond on typical dictionaries)

## File Structure

- `reference.txt`  
  Canonical vocabulary (one word per line). The source dictionary to match against.

- `errors.txt` *(optional for basic usage)*  
  Noisy inputs / test inputs to correct (can be used in batch scripts).

- `matcher.py`  
  Core implementation: builds BK-tree and k-gram index, normalizes inputs, and returns best match.

- `evaluation_script.py`  
  (Or the cell in Colab) Generates synthetic test pairs, runs matching, computes:
  - Exact-match accuracy  
  - Top-3 recall  
  - Average normalized edit distance  
  - Timing (efficiency)  
  - Saves `gold_*.tsv` and `predicted_*.tsv` for proof.

- `corrected_output.tsv`  
  (Generated) Mapping of input → predicted canonical word, with scores/confidence if enabled.

## Installation / Setup

No heavy dependencies required for the core BK-tree version. Just standard Python 3.

```bash
# Optional: create virtualenv
python -m venv venv
source venv/bin/activate

# If using fuzzy/phonetic extended version:
pip install rapidfuzz jellyfish
````

## Usage

### 1. Prepare reference

Populate `reference.txt` with your canonical words (verbs, names, tokens, etc.), one per line.

### 2. Match a single word

Import and use the matching logic (example from `matcher.py`):

```python
from matcher import build_structure, match_best

reference = open("reference.txt").read().splitlines()
structure = build_structure(reference, bk_radius=2, k=3)  # tune radius/k as needed

input_word = "Raaam"
best_match, distance = match_best(input_word, structure)
print(f"{input_word} → {best_match} (edit distance: {distance})")
```

### 3. Batch evaluation (proof of accuracy)

Run the evaluation script to:

* Generate synthetic noisy variants from the reference
* Match them back
* Compute metrics and save:

  * `gold_bktest.tsv`
  * `predicted_bktest.tsv`

Example metrics output:

```
Exact-match accuracy: 93.5%
Top-3 recall: 98.7%
Avg normalized edit distance: 0.05
Per-word latency: ~2ms
```

## How It Works (in brief)

1. **Normalize**: Input and reference words are cleaned (lowercased, repeated vowels collapsed, dialectal replacements applied, non-alphanumerics stripped).
2. **BK-tree**: Built over normalized reference forms for fast approximate-match retrieval using Levenshtein distance.
3. **Query**:

   * First attempt: search BK-tree within a small radius.
   * Fallback: if no close BK-tree hits, use k-gram overlap to seed candidates, then compute edit distances.
4. **Selection**: Pick the candidate with the smallest edit distance as the best match.
5. **Optional fallback**: If both fail, do a brute-force scan (rare).

## Tuning

* `bk_radius`: how tolerant the BK-tree is to edit differences. Larger radius finds more distant matches but may slow queries.
* `k` (k-gram size): controls granularity of fallback overlap. Commonly 2 or 3.
* Thresholding (if added): you can reject or flag matches with edit distance above a cutoff for manual review.

## Efficiency

* Preprocessing (once): normalization + BK-tree / k-gram index build.
* Query time: expected sublinear via BK-tree; common cases resolved in constant/few comparisons.
* Empirical per-word latency is typically **1–5ms** on dictionaries of size 10k–100k in Python.

## Accuracy Proof

To prove accuracy to stakeholders:

1. Provide `gold_*.tsv` (ground truth noisy→correct pairs).
2. Provide `predicted_*.tsv` (system output).
3. Show metrics:

   * Exact-match accuracy (how often top prediction equals correct word)
   * Top-K recall (correct word appears among top-k candidates)
   * Average normalized edit distance (distance error magnitude)
   * Sample mismatches with alternatives
4. Show timing to demonstrate efficiency.

## Example

Input: `shakttii`
Normalized → `shakti`
Matched: `Shakti` (edit dist = 1 or 2 depending on corruption)

## Extension Ideas

* Add confidence scoring / labeling (HIGH/MEDIUM/LOW) based on distance thresholds.
* Integrate into a REST API for real-time correction.
* Log low-confidence or unmatched inputs to expand reference dictionary (bootstrapping).
* Combine with lightweight phonetic boosting (optional) for improved recall on sound-alikes.

## License

Specify your license here (e.g., MIT, proprietary, etc.)

## Contact / Demo

Include a short demo script or link if needed.
For detailed evaluation report generation, run the provided evaluation script and share the TSVs.



