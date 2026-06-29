# TakeMeter Project Planning Specification (`planning.md`)

## 1. Community Selection & Rationalization
* **Chosen Community:** `r/nba` (The National Basketball Association Subreddit)
* **Target Data Source:** Post-Game Elimination Thread: *"The Oklahoma City Thunder have been eliminated"*
* **Why this community matches the task:** Online sports forums are hyper-reactive ecosystems where community discourse ranges from deeply analytical post-mortems to unhinged, emotional flame wars. An elimination thread functions as a pressure cooker, compressing the community's entire rhetorical spectrum into a single thread. The discourse is highly varied, containing everything from detailed salary cap and tactical breakdowns to reactive memes and reactionary, unbacked hot takes. This makes it an ideal domain for evaluation using text classification.

---

## 2. Label Taxonomy & Baseline Examples
The classification system utilizes three mutually exclusive and exhaustive labels to parse the community's discourse quality.

### Label 1: `analysis`
* **Definition:** The post makes a structured or logical argument backed by tactical observations, statistical evidence, financial data (e.g., salary cap rules), or historical comparisons where evidence is explicit or strongly implied.
* **Example 1:** *"In Hartenstein + Chet minutes the Spurs would typically put a way smaller defender on Chet and he never took advantage."*
* **Example 2:** *"There are no more dynasties. Denver, nope. Boston, nope. OKC, nope. Warriors were the last one to do back to back. But we’re in the finals 5 straight years. We will not see that again with the 2nd apron."*

### Label 2: `hot_take`
* **Definition:** A bold, confident, sweeping, or accusatory opinion regarding a team, player, or coach stated entirely without substantive supporting evidence or reasoning. 
* **Example 1:** *"CHET WORST GAME 7 PERFORMANCE IN HISTORY"*
* **Example 2:** *"Wemby is going to singlehandedly alter the course of Chets career. I've never seen a player get sonned so hard by one player and it visibly crater their impact that way."*

### Label 3: `reaction`
* **Definition:** An immediate emotional response, meme, inside joke, short exclamation, or meta-commentary about the thread itself, reflecting feelings in the moment with no attempt at analytical argument or hot-take generation.
* **Example 1:** *"COUNTER-TERRORISTS WIN"*
* **Example 2:** *"BYE BYE 👋"*

---

## 3. Hard Edge Cases & Decision Boundaries
Discourse boundaries naturally blur when fans blend humor with analysis or voice aggressive opinions utilizing loose facts. To keep annotation consistent, the following specific decision rules resolve boundaries:

* **Edge Case 1: Hype/Emotion vs. Historical Comparison (`reaction` vs. `analysis`)**
  * *Example Post:* *"We might be witnessing history. Wemby is unlike anything we've ever seen already. If he wins a chip at 22 years old as the best player on his team. Man. With a team that went from the bottom of the league when he was drafted just a short few years ago."*
  * *Decision Rule:* If a post uses hyperbole but builds its argument on a specific, verifiable framework (e.g., team trajectory since being drafted, age brackets, roster position), it is classified as `analysis`. Pure excitement without structural backing defaults to `reaction`. Therefore, this example is **`analysis`**.
* **Edge Case 2: Sweeping Assertion vs. Structural Argument (`hot_take` vs. `analysis`)**
  * *Example Post:* *"There are no more dynasties... We will not see that again with the 2nd apron."*
  * *Decision Rule:* Aggressive assertions that might seem like hot takes are elevated to `analysis` if they explicitly ground their premise in a structural rule or technical mechanic of the game (such as the NBA's collective bargaining agreement "2nd apron" luxury tax restriction). This provides a verifiable rationale, rendering it **`analysis`**.
* **Edge Case 3: Insult/Meme vs. Serious Take (`reaction` vs. `hot_take`)**
  * *Example Post:* *"CHET IS FUCKING GARBAGE, COUNTER TERRORISTS WIN"*
  * *Decision Rule:* If a post features an aggressive evaluation but pairs it with an immediate video game meme phrase or casual slang meant to mock rather than debate, it functions primarily as an emotional celebration/vent. It is classified as a **`reaction`**.

---

## 4. Data Collection & Curation Plan
* **Source Platform:** Public Reddit data scraped from the `r/nba` elimination thread using RedScraper.
* **Volume Target:** Exactly 200 clean, deduplicated human comments after filtering out automoderator bots, deleted/removed content placeholders, and entirely blank submissions.
* **Distribution Target:** Aim for balanced representation, aiming for at least 20% minimum per class to prevent severe majority-class bias.
* **Mitigation for Underrepresented Labels:** If a category (such as `analysis`) falls below the 20% margin (40 comments) after analyzing the initial scrape pool, the collection will be padded by extracting an extra 30 comments from the same thread specifically targeted by comment length (sorting or filtering for longer character counts) to capture substantive arguments.

---

## 5. Evaluation Metrics
To comprehensively judge the model's value beyond simple overall accuracy, evaluation relies on a combination of the following indicators:
* **Overall Accuracy:** Measures global correctness across all test rows.
* **Per-Class F1-Score:** The metric used to judge the performance of individual classes. Because `reaction` text tends to dominate online spaces numerically, standard accuracy could hide a model that fails entirely at parsing technical `analysis`. Evaluating the harmonic mean of precision and recall per category ensures the boundaries are solid across all classes.
* **Confusion Matrix:** Essential for identifying directional confusion (e.g., tracking whether the model systematically misclassifies nuanced `analysis` text as a flat `hot_take`).

---

## 6. Definition of Success
* **Deployable Threshold:** For this model to be genuinely useful as a community moderator or feed-filtering utility, it must achieve an **overall accuracy of $\ge$ 75%** on the test set.
* **Class-Specific Benchmark:** The system must achieve a **Macro F1-Score of $\ge$ 0.70** across all three distinct buckets.
* **Justification:** Online text patterns contain high amounts of sarcasm and sports slang that present intrinsic difficulty. A baseline accuracy of 75% demonstrates a strong signal that cleanly outperforms trivial zero-shot models and simple keyword lookups while effectively managing severe real-world noise.

---

## 7. AI Tool Plan

### Label Stress-Testing
Prior to completing the annotation framework, the label structures will be provided to an LLM (Claude/ChatGPT) to generate synthetic text blocks matching the specific class boundaries. If the generated boundary items expose logical holes where a post matches multiple categories, the classification instructions will be narrowed further before processing the core dataset.

### Annotation Assistance
A semi-automated pipeline approach is utilized to maintain speed. An LLM will handle initial programmatic text processing by pre-labeling comments in batches of 100 rows using explicit instructions. To ensure total alignment with the project specifications, a complete human review will be executed over every single predicted label in Excel/Google Sheets, explicitly modifying any errors or misaligned classifications prior to final tokenization.

### Failure Analysis
Following evaluation runs, all misclassified test lines along with their true labels and incorrect predictions will be compiled and processed using an LLM. The model will search for systematic traits (such as comment length, sentence count, use of punctuation, or presence of sarcasm) to accelerate error grouping. These extracted patterns will be audited by hand against the generated confusion matrix to construct the final project summary report.