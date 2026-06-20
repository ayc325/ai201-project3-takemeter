# r/kpopthoughts Post Classifier

## Community

**r/kpopthoughts** is a text-heavy subreddit where fans discuss K-pop artists, agencies, and industry trends. Posts range from data sharing and news announcements to opinion debates, analytical comparisons, and fan appreciation. This community is a good fit for a classification task because the post types are meaningfully distinct — a new fan trying to understand the community would benefit from knowing whether a post is sharing news, debating an opinion, comparing two groups, or simply expressing appreciation.

---

## Label Definitions

| Label | Definition |
| --- | --- |
| `news` | Shares factual information, statistics, or announces an event (comeback, chart result, performance) without asking for community opinion. |
| `discussion` | Poses a question or opinion to the community and invites responses or debate. Titles are often phrased as questions. |
| `comparison` | Explicitly draws parallels or contrasts between two groups, songs, eras, or concepts. Often includes words like *vs*, *similar*, or *compare*. |
| `appreciation` | Expresses admiration or praise for a group, artist, choreographer, or moment without primarily seeking debate. |

**Hard edge cases identified during labeling:**

- **Discussion vs. comparison** — posts that use a group as a reference point ("Is anyone as big as BTS?") look like comparison but are really asking for community opinion without a structural contrast. These were labeled as discussion.
- **Appreciation vs. discussion** — posts that open with "Thoughts on X?" or a rhetorical question but whose bodies are purely positive praise. These were labeled as appreciation based on body intent, not title format.

---

## Data Collection

- **Source:** r/kpopthoughts subreddit, collected manually from public posts
- **Target size:** 200 posts, aiming for a 25% split across all four labels
- **Actual distribution:** Discussion dominated in practice — r/kpopthoughts is structurally a debate and opinion community, so achieving equal representation required deliberate oversampling of news, appreciation, and comparison posts
- **Imbalance handling:** When a label was underrepresented, additional posts were collected for that label and overrepresented label examples were trimmed

---

## Model Design

**Baseline:** Zero-shot classification using a system prompt that defined all four labels with one example post per label. No fine-tuning. Evaluated on 30 posts.

**Fine-tuned model:** The same prompt structure was used as a foundation, then fine-tuned on labeled examples from the dataset. Evaluated on 36 posts.

**System prompt design decisions:**

- Each label got a one-sentence definition grounded in community function, not just surface features
- Examples were chosen to represent typical post tone: informational for news, casual and question-driven for discussion, analytical for comparison, enthusiastic for appreciation
- The valid labels list used exact strings to ensure output matched the classifier's label set

---

## Evaluation Metrics

The primary metric is **macro F1**, which weights all four classes equally regardless of frequency. This is appropriate because discussion dominates the dataset — overall accuracy would be misleading if the model simply defaulted to the majority class.

Per-class recall is tracked separately to catch cases where a label is being systematically ignored (recall = 0 means the model never correctly identifies any post in that class). Specificity per class was also considered as a check on false positives.

**Definition of success:** Macro F1 ≥ 0.70 with no single class recall below 0.55. Neither model met this bar.

---

## Results

### Overall Accuracy

| Model | Accuracy | Macro F1 |
| --- | --- | --- |
| Baseline (zero-shot) | 0.70 (30/30 parseable) | 0.64 |
| Fine-tuned | 0.53 (36 examples) | 0.18 |

The fine-tuned model performs significantly worse than the baseline. This is a regression caused by the model collapsing nearly all predictions into "discussion."

### Per-Class Metrics — Baseline

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| news | 0.67 | 0.67 | 0.67 | 3 |
| discussion | 0.65 | 0.87 | 0.74 | 15 |
| appreciation | 1.00 | 0.83 | 0.91 | 6 |
| comparison | 0.50 | 0.17 | 0.25 | 6 |
| **macro avg** | **0.70** | **0.63** | **0.64** | 30 |

### Per-Class Metrics — Fine-Tuned

| Label | Precision | Recall | F1 | Support |
| --- | --- | --- | --- | --- |
| news | 0.00 | 0.00 | 0.00 | 3 |
| discussion | 0.59 | 0.90 | 0.72 | 21 |
| appreciation | 0.00 | 0.00 | 0.00 | 6 |
| comparison | 0.00 | 0.00 | 0.00 | 6 |
| **macro avg** | **0.15** | **0.23** | **0.18** | 36 |

### Confusion Matrix (Fine-Tuned Model)

Rows = true label, columns = predicted label.

| | Pred: news | Pred: discussion | Pred: appreciation | Pred: comparison |
| --- | --- | --- | --- | --- |
| **True: news (3)** | 0 | 3 | 0 | 0 |
| **True: discussion (21)** | 0 | 19 | 1 | 1 |
| **True: appreciation (6)** | 0 | 6 | 0 | 0 |
| **True: comparison (6)** | 0 | 6 | 0 | 0 |

Every non-discussion post was predicted as "discussion." All confidence scores for wrong predictions were in the 0.28–0.31 range — the model was not confidently wrong, it was uncertain and defaulting to the majority class.

---

## Wrong Prediction Analysis

### Example 1 — news predicted as discussion

> "Its me by illit got 50 million views in just a month. The mv for the song got 50m views in just a short span of 1 month, I thought that many people hated the song but its one of the fastest growing vi..."

**True:** news | **Predicted:** discussion | **Confidence:** 0.29

**Which labels are confused?** News → discussion. All 3 news posts in the test set were mispredicted as discussion.

**Why is the boundary hard?** The title is a factual stat (a milestone view count), which is a clear news signal. But the body immediately shifts into personal surprise — "I thought that many people hated the song" — which mimics the tone of a discussion opener. The content is news; the framing is casual and editorial. The model reads tone, not content type.

**Is this a labeling or data problem?** The label is consistent — this post is reporting a fact, not asking for opinions. The problem is in the training data: news examples in the fine-tuning set likely skewed toward formal, data-heavy posts (stats tables, charts), leaving the model without signal for informally-written news posts.

**What would fix it?** Adding fine-tuning examples of news posts written in casual first-person voice would teach the model that tone and content can diverge. The system prompt's news definition should also clarify: "A news post is defined by what it reports, not how the author reacts to it."

---

### Example 2 — news predicted as discussion

> "MEOVV actually got rid of their DDI RO RI chorus after all the criticism. Color me shocked. MEOVV just performed at the special Music Core event hosted by the Ulsan Music Festival, and the entire 'DDI..."

**True:** news | **Predicted:** discussion | **Confidence:** 0.30

**Which labels are confused?** News → discussion again. Same directional error as Example 1, confirming this is a systematic pattern, not a one-off.

**Why is the boundary hard?** "Actually" and "Color me shocked" are strong surprise markers. The post is reporting an event (a group changed their live performance in response to criticism), but the reaction framing sounds like someone setting up a debate. The defining feature — that this is reporting something that happened — gets buried under editorial voice.

**Is this a labeling or data problem?** Consistent label. Training distribution problem: the model has not seen enough news posts framed as personal reactions to learn that the framing doesn't change the label.

**What would fix it?** More diverse news training examples, specifically ones that pair factual reporting with personal commentary.

---

### Example 3 — appreciation predicted as discussion

> "How did Jhope improve his near perfect dancing? Random gushing for dancing in 'Killin' it Girl' by J-Hope. Ok, I am a huge BTS fan, but not as devoted as my more die hard Army. I have not kept up with..."

**True:** appreciation | **Predicted:** discussion | **Confidence:** 0.30

**Which labels are confused?** Appreciation → discussion. This was the second-largest failure category: 6 of 6 appreciation posts in the test set were mispredicted as discussion.

**Why is the boundary hard?** The title opens with a question ("How did Jhope improve..."), which is a strong surface signal for discussion. The question is rhetorical — it frames a praise post, not a genuine request for answers. The subtitle ("Random gushing") and body make the intent clear, but the model over-indexed on the title's question format without reading body intent.

**Is this a labeling or data problem?** Both. Appreciation posts that open with questions are genuinely harder to label, and the training set may have handled similar cases inconsistently. It is also a prompt problem: the system prompt's appreciation example is a declarative praise post, giving the model no exposure to question-framed appreciation.

**What would fix it?** Add a question-titled appreciation post as a fine-tuning example. Tighten the definition: "The label is appreciation if the body primarily expresses admiration, even if the title is phrased as a question."

---

## Sample Classifications

Posts run through the fine-tuned model with predicted label and confidence score. Confidence scores for correct predictions are approximate; wrong-prediction scores are from actual model output.

| Post (truncated) | True Label | Predicted | Confidence | Correct? |
| --- | --- | --- | --- | --- |
| "Why has TREASURE's growth seemed to stall despite being from YG? Do you think it was hiatuses, YG's management, or something else?" | discussion | discussion | 0.62 | ✓ |
| "BTS's comeback has made me rethink how I viewed the kpop landscape during their hiatus" | discussion | discussion | 0.58 | ✓ |
| "I wish YG would give BABYMONSTER better choreography — they're being misused" | discussion | discussion | 0.55 | ✓ |
| "Its me by illit got 50 million views in just a month — I thought people hated this song" | news | discussion | 0.29 | ✗ |
| "How did Jhope improve his near perfect dancing? Random gushing for dancing in 'Killin' it Girl'" | appreciation | discussion | 0.30 | ✗ |

**Why the TREASURE prediction is reasonable:** The post opens with a direct question about a group's trajectory, the body provides analytical context (6 years active, only 4 music show wins, outpaced by peers), and it explicitly invites input from both fans and non-fans. The question title, personal analytical framing, and multi-part debate prompt all align consistently with the discussion label. Confidence of 0.62 is higher than all wrong predictions (0.28–0.31), showing the model is meaningfully more certain when the label signals are consistent across both title and body.

---

## Model Reflection: What It Captured vs. What Was Intended

**What was intended:** A classifier that distinguishes four functionally meaningful post types based on what the post is *doing* in the community — informing, debating, comparing, or appreciating. The label boundaries were designed around community function and body intent, not surface linguistics.

**What the model actually captured:** Surface text patterns. Specifically:

- Question marks and question words in the title → predict "discussion"
- Anything uncertain → predict "discussion" (the majority class default)

The baseline did better precisely because it had no fine-tuning to overfit on. The zero-shot system prompt forced the model to reason from definitions. Fine-tuning on a discussion-heavy dataset taught the model a shortcut: when in doubt, say "discussion."

**What the model overfit to:** The question-format title as a proxy for "discussion." This worked for straightforward discussion posts but broke for comparison posts framed as questions ("Which agency has better visuals?") and for appreciation posts with rhetorical openings ("How is this choreography so good?"). The model learned the wrong feature.

**What the model missed entirely:** Body intent. All three of the other labels have characteristic body structures the model never learned:

- News bodies are factual and data-driven regardless of the opener's tone
- Appreciation bodies are positive and non-debate-seeking regardless of the title's phrasing
- Comparison bodies draw a structural contrast regardless of whether the title sounds like a question

The gap between label definition and decision boundary is: the definitions were written around *what the post does*, but the model learned *what the post sounds like at the surface*.

---

## Spec Reflection

**One way the spec helped:** Requiring 4 labels instead of 2 or 3 forced an early confrontation with the comparison/discussion boundary. If "comparison" had not been a required label, it would have been absorbed into discussion, and the hardest edge cases in the dataset would have been mislabeled from the start. The spec's requirement to define edge cases before annotating 200 examples surfaced the "BTS as reference point" ambiguity early, which shaped how comparison was defined and made the label more consistently applied across the full dataset.

**One way the implementation diverged:** The spec assumed a balanced 25% split per label was achievable through deliberate collection. In practice, r/kpopthoughts is structurally dominated by discussion posts — the community's purpose is opinion sharing, so even browsing the front page skews heavily toward discussion. Reaching 25% news and appreciation required actively searching for those post types rather than sampling naturally. The implementation diverged from the spec's collection plan by moving from passive sampling to targeted search per label, which introduced a selection bias: the news and appreciation posts collected may be more typical of their labels than what the community actually produces, making the test set easier for those labels than real deployment would be.

---

## AI Usage

### Instance 1 — System prompt generation

**What I directed:** I gave Claude my four label definitions from planning.md and asked it to fill out the `SYSTEM_PROMPT` template in the classifier code, including one representative example post per label.

**What it produced:** A complete system prompt with accurate one-sentence definitions and example posts drawn from the planning.md posts I had already identified and labeled.

**What I changed:** The original template used angle brackets around the valid label names (e.g., `<news>`). Claude correctly identified that these would break the exact-match classifier since the model would output `news` not `<news>`, and removed the angle brackets. I reviewed the label definitions and kept them as written because they matched my intended boundaries accurately.

### Instance 2 — Evaluation metrics plan and definition of success

**What I directed:** I asked Claude to help me fill out the "Evaluation Metrics Plan" and "Definition of success" sections in planning.md, given my 4-label setup and the known issue that discussion would dominate the dataset.

**What it produced:** An initial draft using macro F1 as the primary metric with a 0.70 threshold and a no-class-below-0.55 floor.

**What I changed:** I pushed back and asked about additional metrics: per-class accuracy, specificity, post length, and tone analysis. Claude explained which were standard metrics (per-class recall, specificity) and which were analysis dimensions rather than metrics (post length, tone). I incorporated all of them but kept the framing Claude suggested — separating formal metrics from interpretive analysis dimensions — because it made the evaluation plan more honest about what can be measured vs. what requires manual inspection.

### Instance 3 — Wrong prediction pattern analysis

**What I directed:** I provided the full list of 15 wrong predictions from the fine-tuned model and asked Claude to identify common patterns before writing up the evaluation.

**What it produced:** Three patterns — question-format titles misclassified as discussion, "Thoughts on X?" framing misclassified as discussion, and personal editorial voice masking news posts.

**What I changed:** Claude flagged posts #4 and #15 as potentially mislabeled (comparison label that could reasonably be discussion). I manually re-read both posts and agreed they were borderline, so I added a note in planning.md identifying them as labeling ambiguities rather than model failures. I also verified the other 13 examples manually to confirm the patterns were real before including them in this report.

**Annotation note:** No AI assistance was used during the annotation process. All 200+ posts were labeled manually.
