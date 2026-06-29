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
| Fine-tuned DistilBERT | **65.7%** |
| Difference | -14.3% |

The fine-tuned model underperforms the baseline by 14 percentage points. This is a significant regression and is explained in detail in the failure analysis below.

### Per-Class Metrics — Both Models

| Label | Baseline Precision | Baseline Recall | Baseline F1 | Fine-tuned Precision | Fine-tuned Recall | Fine-tuned F1 |
|---|---|---|---|---|---|---|
| `analysis` | 1.00 | 0.43 | 0.60 | 0.75 | 0.43 | 0.55 |
| `hot_take` | 0.62 | 0.71 | 0.67 | — | 0.00 | 0.00 |
| `reaction` | 0.83 | 0.95 | 0.89 | 0.65 | 0.95 | 0.77 |
| **Macro avg** | **0.82** | **0.70** | **0.72** | — | — | **0.44** |

*Fine-tuned `hot_take` precision is undefined — the model predicted this label zero times across the entire test set.*

### Confusion Matrix — Fine-Tuned Model

|  | Predicted: analysis | Predicted: hot_take | Predicted: reaction |
|---|---|---|---|
| **True: analysis** | 3 | 0 | 4 |
| **True: hot_take** | 0 | 0 | 7 |
| **True: reaction** | 1 | 0 | 20 |

### Failure Analysis

**The dominant failure mode: `hot_take` total collapse.**

The fine-tuned model predicted `hot_take` zero times. Every `hot_take` in the test set (7 of 7) was classified as `reaction`. This is the most severe failure in the results.

**Why this boundary is hard:** `hot_take` and `reaction` share nearly all surface features. Both are often short. Both frequently use all-caps, profanity, and exclamation. Neither cites evidence. The distinction my taxonomy draws — *sincere claim meant to be debated* vs. *emotional expression not meant to be debated* — is pragmatic and contextual, not lexical. A model trained on 161 examples cannot learn a distinction that depends on speaker intent rather than word choice.

**Secondary failure: `analysis` leaking into `reaction`.**

4 of 7 `analysis` examples in the test set were predicted as `reaction`. The model learned that `reaction` is the safe default — when in doubt, predict `reaction`. This makes structural sense given that `reaction` was 60% of training data, but it means the model essentially learned a majority-class bias rather than the underlying discourse distinctions.

**Three specific wrong predictions:**

**Wrong prediction 1 (hot_take → reaction):**
*"No one feels sorry for Steve Ballmer."*
True label: `hot_take` | Predicted: `reaction`

This is a six-word declarative opinion with no evidence. It reads identically to a pure reaction comment (`"FRAUD"`, `"BYE BYE 👋"`) from a surface-feature standpoint. The model had no way to learn that the Ballmer comment is making a sincere debatable claim while `"FRAUD"` is an emotional exclamation — both are short, both lack evidence, both are negative. This is a genuine labeling boundary problem: the `hot_take`/`reaction` distinction requires understanding communicative intent that a small DistilBERT model cannot infer from text alone.

**Wrong prediction 2 (hot_take → reaction):**
*"Spurs fans never realized it at the time but they were actually lucky he left."*
True label: `hot_take` | Predicted: `reaction`

This is a more substantive `hot_take` with a clear, challengeable claim. The failure here reveals a different problem: the Kawhi thread's `hot_take` examples (about Ballmer, the Clippers organization, legacy claims) are linguistically distinct from the OKC thread's `hot_take` examples (about Chet, SGA). The model likely never learned a cross-domain `hot_take` pattern — it learned "things people say about Chet" and "things people say in celebration" and classified everything else as `reaction` by default.

**Wrong prediction 3 (analysis → reaction):**
*"I won't defend Kawhi cause some of Plaschke's complaints are like deja vu to my team, but he should be more upset at the org for what they already knew about Kawhi needing load management and bending over to get the PG trade done. Kawhi doesn't do all of this in a vacuum by himself."*
True label: `analysis` | Predicted: `reaction`

This is the clearest evidence of the majority-class bias. This comment makes a structural argument (the org's culpability, the known risk of load management) that should be recognizable as `analysis`. The model predicted `reaction`. The training signal for `analysis` was simply too weak — 46 examples, roughly 32 in training, split across two different topic domains — for the model to reliably identify this pattern.

### Sample Classifications

The following examples were run through the fine-tuned model:

| Post | True Label | Predicted Label | Confidence |
|---|---|---|---|
| *"COUNTER-TERRORISTS WIN"* | `reaction` | `reaction` | ~0.97 |
| *"BYE BYE 👋"* | `reaction` | `reaction` | ~0.95 |
| *"In Hartenstein + Chet minutes the Spurs would typically put a way smaller defender on Chet and he never took advantage."* | `analysis` | `analysis` | ~0.71 |
| *"CHET WORST GAME 7 PERFORMANCE IN HISTORY"* | `hot_take` | `reaction` | ~0.89 |
| *"Chet sucks"* | `hot_take` | `reaction` | ~0.94 |

**On the correct `analysis` prediction:** The Hartenstein/Chet comment is correctly identified as `analysis` because it describes a specific, observable tactical situation with named players and a concrete matchup. The model appears to have learned that naming specific game situations and player matchups is a reliable signal for `analysis` — which is true in the training data.

**On the `hot_take` failures:** Both hot takes are confidently predicted as `reaction` (0.89 and 0.94). The model is not uncertain about these — it has learned the wrong boundary with high confidence. This is worse than being uncertain; it means more data alone may not fix the problem without also addressing the label definition.

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
