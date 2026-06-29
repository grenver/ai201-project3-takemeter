# TakeMeter — NBA Discourse Quality Classifier

A fine-tuned text classifier that evaluates discourse quality in r/nba, distinguishing structured analysis from bold unsupported takes from pure emotional reaction.

---

## Community Choice

**Community:** r/nba

**Why this community:** NBA subreddits produce an unusually wide rhetorical spectrum within single threads. An OKC elimination thread and a Kawhi Leonard trade rumor thread both contain everything from detailed salary cap arguments to one-word meme responses — sometimes in adjacent comments. That variance across a shared topic makes r/nba a strong fit for a discourse-quality classifier: the signal isn't which team someone supports, it's how they argue.

**Data sources:** Two threads were used.
- The OKC Thunder elimination thread (`r/nba`, June 2025) — a high-volume celebration/pile-on thread, heavy in `reaction` content
- The Kawhi Leonard / Plaschke article thread (`r/nba`, June 2026) — an opinion-and-argument thread with strong `analysis` and `hot_take` representation

Using two threads was necessary to balance the dataset. Elimination threads structurally over-produce `reaction` content; argument threads produce the minority classes. Both are standard r/nba discourse.

---

## Label Taxonomy

Three mutually exclusive labels capture the meaningful distinctions in r/nba discourse quality.

### `analysis`
A post makes a structured or logical argument backed by tactical observations, statistical evidence, financial data (e.g., salary cap rules), or historical comparisons where evidence is explicit or strongly implied.

- *"In Hartenstein + Chet minutes the Spurs would typically put a way smaller defender on Chet and he never took advantage."*
- *"Sorry, but you can't retroactively call the kawhi signing or the PG trade bad with the benefit of hindsight. PG was PG, and he was balling out at the time, and nobody really knew what SGA would turn into."*

### `hot_take`
A bold, confident, sweeping, or accusatory opinion regarding a team, player, or coach stated without substantive supporting evidence or reasoning.

- *"CHET WORST GAME 7 PERFORMANCE IN HISTORY"*
- *"Spurs fans never realized it at the time but they were actually lucky he left."*

### `reaction`
An immediate emotional response, meme, inside joke, short exclamation, or meta-commentary reflecting feelings in the moment with no attempt at analytical argument or hot-take generation.

- *"COUNTER-TERRORISTS WIN"*
- *"BYE BYE 👋"*

---

## Data Collection

**Source:** Public Reddit comments scraped using RedScraper from two r/nba threads. All content is publicly accessible with no authentication required.

**Labeling process:** Comments were labeled manually using the definitions and edge case rules documented in `planning.md`. An LLM (Claude) was used to pre-label batches of 50–100 comments at a time using the explicit label definitions as a prompt. Every pre-assigned label was reviewed and corrected by hand before being accepted into the dataset. Approximately 15–20% of pre-labels were overridden during review.

**Final distribution (230 total):**

| Label | Count | % |
|---|---|---|
| `reaction` | 138 | 60.0% |
| `hot_take` | 46 | 20.0% |
| `analysis` | 46 | 20.0% |

**Three examples that were genuinely difficult to label:**

1. *"We might be witnessing history. Wemby is unlike anything we've ever seen already. If he wins a chip at 22 years old as the best player on his team. Man. With a team that went from the bottom of the league when he was drafted just a short few years ago."*
   This reads as excited hyperbole (`reaction`) but actually builds its argument on a specific, verifiable framework: age bracket, roster trajectory since the draft, and comparative team position. Decision rule: if the post grounds its claim in a structural, verifiable framing (even if enthusiastically stated), it is `analysis`. → **`analysis`**

2. *"There are no more dynasties. Denver, nope. Boston, nope. OKC, nope. Warriors were the last one to do back to back. But we're in the finals 5 straight years. We will not see that again with the 2nd apron."*
   Aggressive assertion that could read as `hot_take`, but it explicitly invokes the NBA's 2nd apron luxury tax restriction as a structural mechanic. That specific verifiable rationale elevates it. → **`analysis`**

3. *"CHET IS FUCKING GARBAGE, COUNTER TERRORISTS WIN"*
   Contains an aggressive player evaluation (`hot_take` candidate) paired immediately with a Counter-Strike meme phrase. The meme framing signals that this is primarily emotional venting, not a sincere claim meant to be debated. → **`reaction`**

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` (HuggingFace)

**Training setup:** Fine-tuned on Google Colab with a T4 GPU. The notebook handled the train/validation/test split automatically (70% / 15% / 15%), producing approximately 161 training examples, 34 validation examples, and 35 test examples.

**Key hyperparameter decision:** The default learning rate of `2e-5` was kept rather than increasing it. With only 161 training examples, a higher learning rate risks overshooting the loss minimum and producing an unstable model. 3 epochs was also kept as the default; given the small dataset size, additional epochs would likely cause overfitting rather than improved generalization.

---

## Baseline

**Model:** Groq `llama-3.3-70b-versatile` (zero-shot)

**Prompt used:**
```
You are classifying Reddit comments from r/nba into one of three discourse quality categories.

Labels:
- analysis: A structured or logical argument backed by stats, tactical observations, financial data, or historical comparisons. Evidence is explicit or strongly implied.
- hot_take: A bold, confident opinion stated without supporting evidence. The claim might be true but the post asserts rather than argues.
- reaction: An emotional response, meme, exclamation, or meta-commentary with no attempt at argument.

Classify the following comment. Respond with ONLY the label name — one of: analysis, hot_take, reaction

Comment: {text}
```

**Result:** 80.0% overall accuracy on the 35-example test set.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy |
|---|---|
| Zero-shot baseline (llama-3.3-70b-versatile) | **80.0%** |
| Fine-tuned DistilBERT | **60.0%** |
| Difference | -20.0% |

The fine-tuned model underperforms the baseline by 20 percentage points. This is a significant regression and is explained in detail in the failure analysis below.

### Per-Class Metrics — Both Models

| Label | Baseline Precision | Baseline Recall | Baseline F1 | Fine-tuned Precision | Fine-tuned Recall | Fine-tuned F1 |
|---|---|---|---|---|---|---|
| `analysis` | 1.00 | 0.43 | 0.60 | — | 0.00 | 0.00 |
| `hot_take` | 0.62 | 0.71 | 0.67 | — | 0.00 | 0.00 |
| `reaction` | 0.83 | 0.95 | 0.89 | 0.60 | 1.00 | 0.75 |
| **Macro avg** | **0.82** | **0.70** | **0.72** | — | — | **0.25** |

*Fine-tuned `analysis` and `hot_take` precision are undefined — the model predicted only `reaction` across all 35 test examples.*

### Confusion Matrix — Fine-Tuned Model

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 0 | 0 | 7 |
| **True: hot_take** | 0 | 0 | 7 |
| **True: reaction** | 0 | 0 | 21 |

### Failure Analysis

**14 of 35 test examples were wrong. The model predicted `reaction` for every single test example — 35 out of 35.**

This is a complete collapse into a single-class predictor. The fine-tuned model learned nothing about `analysis` or `hot_take` — it defaulted to `reaction` with 100% frequency, which is the majority class at 60% of the training data. The confidence scores on all wrong predictions ranged from 0.39–0.46, barely above the 0.33 random-chance floor for a 3-class problem. The model was not confidently wrong; it was uncertain every time and resolved that uncertainty the same way: predict `reaction`.

**Failure type 1: `hot_take` complete miss (7 wrong)**

The model predicted `hot_take` zero times. All 7 `hot_take` examples in the test set were classified as `reaction`. `hot_take` and `reaction` share nearly every surface feature — both are short, assertive, and lack evidence. The distinction in my taxonomy (sincere debatable claim vs. emotional venting) is pragmatic and depends on speaker intent, not word choice. With ~32 `hot_take` training examples, the model never learned this signal.

**Failure type 2: `analysis` complete miss (7 wrong)**

The model predicted `analysis` zero times. All 7 `analysis` examples were classified as `reaction`. The model did not learn that structured arguments embedded in casual or profane language are still `analysis`. The 60% training share of `reaction` created an overwhelming pull toward that label.

**Three specific wrong predictions:**

**Wrong prediction 1 (hot_take → reaction):**
*"perhaps we shouldnt be calling teams a dynasty after 1 chip lol"*
True: `hot_take` | Predicted: `reaction` | Confidence: 0.46

This is a clear debatable claim — plenty of r/nba users would push back on it. But it ends with "lol," uses no capitalization, and is structurally identical to a dismissive reaction comment. The model cannot distinguish between "a take delivered casually" and "pure emotional noise." This is a labeling boundary problem: the `hot_take`/`reaction` line depends on whether the speaker intends the claim to be argued with, which no surface feature reliably signals.

**Wrong prediction 2 (analysis → reaction):**
*"In Hartenstein + Chet minutes the Spurs would typically put a way smaller defender on Chet and he never took advantage."*
True: `analysis` | Predicted: `reaction` | Confidence: 0.39

This is one of the clearest `analysis` examples in the dataset — it names a specific lineup combination, describes a tactical matchup, and makes a verifiable observation. The model predicted `reaction` with the lowest confidence in the entire wrong-predictions list (0.39). It wasn't even close to `analysis`. This reveals that 32 training examples of `analysis` was not enough to learn the pattern reliably, especially for shorter analytical observations that don't look like formal writing.

**Wrong prediction 3 (analysis → reaction):**
*"r/NBA 12/25 EDIT: The Spurs averaged 29 wins over the last 6 years, and just eliminated OKC. What the actual fuck"*
True: `analysis` | Predicted: `reaction` | Confidence: 0.46

This post cites a specific statistic (29 wins over 6 years) grounded in a verifiable claim about team trajectory. Under my decision rules it is `analysis`. But it ends with "What the actual fuck" — a phrase that appears throughout `reaction` posts in the training set. The model likely learned that expletive-exclamation endings signal `reaction` and overrode the statistical content. This is a data problem: the training set needed more examples of analysis written in a casual/profane register to teach the model that emotional language doesn't disqualify a post from being analytical.

### Sample Classifications

The following are drawn from the model's actual test set predictions:

| Post | True Label | Predicted | Confidence | Correct? |
|---|---|---|---|---|
| *"COUNTER-TERRORISTS WIN"* | `reaction` | `reaction` | 0.91 | ✅ |
| *"They literally don't win without him"* | `hot_take` | `reaction` | 0.46 | ❌ |
| *"CHET WORST GAME 7 PERFORMANCE IN HISTORY"* | `hot_take` | `reaction` | 0.41 | ❌ |
| *"In Hartenstein + Chet minutes the Spurs would typically put a way smaller defender on Chet and he never took advantage."* | `analysis` | `reaction` | 0.39 | ❌ |
| *"morey 🤝 divac being stupid as fuck"* | `hot_take` | `reaction` | 0.45 | ❌ |

**On the correct prediction:** "COUNTER-TERRORISTS WIN" is correctly labeled `reaction` with high confidence. It is a pure CS:GO meme with no basketball content — the model reliably learned that meme phrases and exclamatory non-arguments belong in `reaction`. This class was well-represented in training and the model performs well on it (0.95 recall).

**On the wrong predictions:** Notice that all four wrong predictions have confidence scores between 0.39–0.46 — barely above the 0.33 random-chance floor for a 3-class problem. The model is not confidently wrong; it is uncertain and defaults to `reaction` every time. This pattern holds across all 14 wrong predictions in the test set. A well-calibrated model would express this uncertainty rather than committing to the majority class.

---

## Reflection: What the Model Learned vs. What I Intended

I intended the model to learn the difference between *how* someone makes a claim (with or without evidence, sincerely or performatively). What it actually learned was a proxy: **post length and topic familiarity**.

The `reaction` class is dominated by very short posts (1–10 words), memes, and game-specific references. The model learned "short and exclamatory → reaction." The `analysis` class is dominated by longer posts with player names and specific game situations. The model learned "long with named entities → analysis." The `hot_take` class is short like `reaction` but doesn't use meme phrases — and there weren't enough training examples for the model to learn the subtle remaining distinction.

The result is a classifier that is roughly a length and topic filter, not a discourse quality evaluator. It would likely predict `reaction` for any short opinion regardless of its debatability, and `analysis` for any long comment regardless of whether it actually argues anything.

This gap between intention and behavior is expected at 161 training examples, but it also reveals that my label definitions required pragmatic inference (speaker intent) that is genuinely hard for a small model. A larger model, or more carefully curated examples that directly contrast same-length `hot_take` vs. `reaction` pairs, might learn a better signal.

---

## Spec Reflection

**One way the spec helped:** The instruction to define success criteria in advance (Milestone 2) forced me to commit to a 75% accuracy / 0.70 Macro F1 target before seeing any results. When the fine-tuned model came in at 65.7% accuracy and 0.44 Macro F1, I could not retroactively lower the bar — the spec made the failure unambiguous and directed me toward genuine analysis rather than post-hoc rationalization.

**One way implementation diverged:** The spec assumes data collection precedes annotation as a clean two-step process. In practice, the initial dataset was heavily imbalanced (~69% reaction) because a single elimination thread structurally over-produces emotional content. I had to return mid-project to collect from a second thread type (a trade rumor/opinion thread) to reach the 20% minimum for minority classes. The spec's mitigation advice (collect longer comments to find analysis) didn't apply to my domain — the issue wasn't comment length, it was thread type. Scraping from a structurally different thread was the necessary fix.

---

## AI Usage

**Instance 1 — Label stress-testing (planning phase):**
I provided Claude with my three label definitions and edge case descriptions and asked it to generate 10 synthetic comments that would sit at the boundary between `hot_take` and `reaction`. It produced several short opinion statements like "This team is cooked" and "He's done." These were genuinely ambiguous under my initial definition. I used them to sharpen my decision rule: `hot_take` requires a claim about a player or team that a community member would *disagree with and argue against*, not just an emotional reaction that people would *share*. That distinction didn't exist in my original spec and was added because of this exercise.

**Instance 2 — Annotation assistance:**
For both the OKC elimination thread (200 comments) and the Kawhi thread (~60 comments), I used Claude to pre-label batches using the label definitions from `planning.md` as a prompt. Claude's pre-labels were accepted roughly 80–85% of the time. The most common overrides were:
- Claude over-predicted `hot_take` for short aggressive comments I labeled `reaction` (it couldn't distinguish sincere claim from performative aggression)
- Claude under-predicted `analysis` for longer comments that used casual language — it associated analytical content with formal writing style

**Instance 3 — Failure pattern analysis:**
After seeing the confusion matrix, I pasted the misclassified test examples into Claude and asked it to identify common themes. It identified: (1) that all `hot_take` misclassifications were short declarative sentences, and (2) that the `analysis` misclassifications all involved casual or profane language despite structural arguments. I verified both patterns by re-reading the examples — both held. I discarded a third suggested pattern (that cross-thread examples were more likely to be misclassified) because I couldn't verify it without knowing which test examples came from which thread.
