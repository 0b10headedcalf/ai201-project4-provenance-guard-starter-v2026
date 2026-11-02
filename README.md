# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

The service runs both signals, combines their scores, displays a label, and
records appeals for review.

1. `POST /submit` receives JSON, checks the text and creator ID, and assigns a
   unique `content_id` to an accepted submission.
2. `detector.model_signal` runs the local language model on the text and adds
   a 0–1 predictability-based AI score.
3. `stylometry.style_signal` measures sentence-length variation, vocabulary
   repetition, and punctuation, then adds a separate 0–1 style score.
4. `scoring.combine_signals` combines those scores into one AI-oriented score;
   disagreement pulls the result toward uncertainty.
5. `scoring.score_to_label` maps that score to a machine-readable guess and a
   sentence for the reader, using a deliberately wide unsure range.
6. `audit.log_decision` records the ID, creator, scores, guess, label, and
   `decided` status. The response returns the ID, scores, guess, and label.
7. A writer who disputes the result can send the ID and an explanation to
   `POST /appeal`; the audit log records the appeal and `under_review` status.
   A person must review it; the appeal does not automatically reverse a label.

---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:** The local model's perplexity: how predictable the first
400 model tokens are, converted to a 0–1 score where higher means more AI-like.

**Why that might differ between human and AI writing:** Generated text often
uses probable word sequences; human word choices can be less predictable.
These tendencies overlap and vary by text and model.

**What it can't see:** It cannot establish who wrote the text. Plain writing,
including writing by younger people or second-language writers, can be highly
predictable; edited or unusual AI text can be surprising. It also does not
inspect text beyond its token limit.

### Signal two — the style signal

**What it measures:** Text surface patterns: variation in sentence length,
vocabulary repetition, and punctuation density, combined into a 0–1 score
where higher means more AI-like.

**Why that might differ between human and AI writing:** Some generated prose
has more regular sentence lengths and repeated phrasing, while some human prose
varies more. Neither pattern belongs exclusively to either group.

**What it can't see:** It cannot understand the text or verify authorship.
Typos, sentence breaks, quotations, genre, and text length can change these
measurements without changing who wrote the work.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:** Take an equal weighted average of the two AI-oriented scores.
When they disagree, the average moves toward the middle rather than allowing
either signal alone to determine the label. These are initial design choices,
not measured accuracy or calibrated probabilities.

**The numbers:** `0.5 * model_score + 0.5 * style_score`.

The style score averages three normalized indicators and subtracts the result
from 1: sentence-length standard deviation divided by mean length, scaled by
0.5; unique words divided by total words, scaled by 0.8; and commas,
semicolons, colons, question marks, exclamation marks and dashes per character,
scaled by 0.03. Each is capped at 1. Text with fewer than 20 words or fewer
than two sentences receives the neutral style score 0.5. These heuristic
choices are starting points, not validated measures of authorship.

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:** If the
model gives 0.90 and style gives 0.10, the combined score is 0.50 and the
result is unsure. In the actual Dickens excerpt below, the model gives 0.9583
and the single-sentence style fallback gives 0.5000; their average is about
0.7292, in the proposed unsure band.

Standalone calibration with both signals (combined values rounded to four
places):

| Input | Model | Style | Combined |
|---|---:|---:|---:|
| Generated framework example from `detector.py` | 0.5924 | 0.6317 | 0.6120 |
| Human: Dickens, opening of *A Tale of Two Cities* | 0.9583 | 0.5000 | 0.7291 |
| Formal human: Lincoln, opening of the Gettysburg Address | 0.6896 | 0.3470 | 0.5183 |
| Lightly edited generated framework example | 0.4311 | 0.5049 | 0.4680 |
| Ambiguous-looking generated rain paragraph | 0.6990 | 0.2660 | 0.4825 |

The generated framework example has similar scores from both signals. The
Dickens and rain examples split substantially. Dickens scores higher overall
than the generated examples, illustrating why these scores cannot verify
authorship; the style fallback does not independently classify that sentence.
The edited example reads: “The implementation of this framework needs careful
consideration of several key factors. The system must handle many different
inputs and provide consistent results — even when conditions change.” The rain
example reads: “The rain stopped before lunch. I opened the window and listened
to the traffic while I tried to finish my work. It was an ordinary day, but I
felt strangely hopeful.”

---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00–0.35 inclusive | The text appears more consistent with human writing, but this is not proof of authorship. |
| unsure | Above 0.35 and below 0.80 | We cannot tell reliably who wrote this text. You can request a review. |
| high-confidence AI | 0.80–1.00 inclusive | The text appears more consistent with AI-generated writing, but this is not proof of authorship. You can request a review. |
The unsure label states the limit of the evidence without accusing the writer.
The 0.80 AI boundary is a starting point chosen to reduce false accusations;
it still needs testing on real human writing and the attack set. It will miss
some AI text as a result.

**False-positive trace:** A person writes a short, plain paragraph themselves.
Hypothetically, the model gives it 0.90 and style gives it 0.80, for a combined
score of 0.85. The reader sees the high-confidence AI sentence above, even
though the writer wrote it. The writer submits the returned `content_id` and
their explanation to `/appeal`; the service records `under_review` for a
person to review. There is no promised review deadline or automatic reversal in
this starter design. This example is illustrative, not an observed result.

### Proposed endpoint contract

Both endpoints accept a JSON object with `Content-Type: application/json` and
return JSON. Scores are numbers in [0, 1], with higher values more AI-like;
`confidence` is the combined score, not a probability or confidence in the
chosen label. Use the unrounded combined score to select a label, and return
scores rounded to four decimal places.

**`POST /submit`**

- Request: `text` (required string, 20–10,000 Unicode characters, containing
  non-whitespace text and at least three model tokens); `creator_id` (required
  nonblank string, at most 200 characters). Character limits count the original
  string; whitespace is not stripped from the text sent to the signals.
- Success: HTTP 200 with `content_id` (UUID string), `guess` (`human`, `unsure`,
  or `ai`), `confidence` (combined score), `label` (the exact sentence from the
  table), `model_score`, and `style_score` (both numbers). A decision is logged
  with status `decided` before returning success.
- Invalid JSON, a non-object body, missing or invalid fields, or too few model
  tokens: HTTP 400 with `error: "invalid_request"` and a human-readable
  `message`. An unavailable model: HTTP 503 with `error: "detector_unavailable"`
  and `message`. No decision is created on failure.

**`POST /appeal`**

- Request: `content_id` (required nonblank string identifying a previously
  decided submission) and `reasoning` (required nonblank string, at most 2,000
  characters).
- Success: HTTP 200 with `content_id`, `status: "under_review"`, and
  `message: "Your appeal has been recorded for review."` Each valid request
  appends an appeal to the audit log, including repeated appeals; it does not
  overwrite the original decision.
- Invalid JSON, a non-object body, or invalid fields: HTTP 400 with
  `error: "invalid_request"` and `message`. An unknown content ID: HTTP 404
  with `error: "not_found"` and `message`. Neither failure creates an appeal.
**Why I worded the "unsure" one this way:**
<!-- It's the hardest of the three. It has to admit uncertainty without
     sounding like an accusation, because the person reading it may well have
     written every word themselves. -->



---

## Sample Run

Port 51247 was used because port 5000 was already occupied. These are
synthetic coverage examples, not evidence that the guesses establish authorship.
The appeal explanation is a test claim, not a verified statement.

**A submission**

```bash
curl -X POST http://127.0.0.1:51247/submit -H 'Content-Type: application/json' -d '{"text":"The cat sat on the mat. The cat sat on the mat. The cat sat on the mat. The cat sat on the mat. The cat sat on the mat. The cat sat on the mat.","creator_id":"milestone5-ai"}'
```

```json
{"confidence":0.9657,"content_id":"d58d5a7f-041f-4759-af09-5c6e0a9e031e","guess":"ai","label":"The text appears more consistent with AI-generated writing, but this is not proof of authorship. You can request a review.","model_score":0.9893,"style_score":0.9421}
```

**Other label coverage**

```bash
curl -X POST http://127.0.0.1:51247/submit -H 'Content-Type: application/json' -d '{"text":"Beneath the glacial crevasse, phosphorescent tardigrades pirouetted; inexplicably, the cartographer catalogued each iridescent gyration. Squawk! Xenolithic fragments, turmeric-stained and bewilderingly asymmetrical, contradicted every geological supposition — especially mine. Why?","creator_id":"milestone5-human"}'
```

```json
{"confidence":0.0741,"content_id":"323f5b65-623d-468e-b02e-6f01c0043697","guess":"human","label":"The text appears more consistent with human writing, but this is not proof of authorship.","model_score":0.1482,"style_score":0.0}
```

```bash
curl -X POST http://127.0.0.1:51247/submit -H 'Content-Type: application/json' -d '{"text":"The rain stopped before lunch. I opened the window and listened to the traffic while I tried to finish my work. It was an ordinary day, but I felt strangely hopeful.","creator_id":"milestone5-unsure"}'
```

```json
{"confidence":0.4825,"content_id":"57103b51-1e89-4a5e-b9e9-cfadcdc44ff8","guess":"unsure","label":"We cannot tell reliably who wrote this text. You can request a review.","model_score":0.699,"style_score":0.266}
```

**An appeal**

```bash
curl -X POST http://127.0.0.1:51247/appeal -H 'Content-Type: application/json' -d '{"content_id":"d58d5a7f-041f-4759-af09-5c6e0a9e031e","reasoning":"I wrote this repeated sentence myself; please review the decision."}'
```

```json
{"content_id":"d58d5a7f-041f-4759-af09-5c6e0a9e031e","message":"Your appeal has been recorded for review.","status":"under_review"}
```

**What the log shows afterwards**

```bash
curl 'http://127.0.0.1:51247/log?limit=4'
```

```json
{"count":4,"entries":[{"combined_score":0.9657,"content_id":"d58d5a7f-041f-4759-af09-5c6e0a9e031e","creator_id":"milestone5-ai","guess":"ai","label":"The text appears more consistent with AI-generated writing, but this is not proof of authorship. You can request a review.","model_score":0.9893,"status":"decided","style_score":0.9421,"timestamp":"2026-11-02T22:40:59+00:00"},{"combined_score":0.0741,"content_id":"323f5b65-623d-468e-b02e-6f01c0043697","creator_id":"milestone5-human","guess":"human","label":"The text appears more consistent with human writing, but this is not proof of authorship.","model_score":0.1482,"status":"decided","style_score":0.0,"timestamp":"2026-11-02T22:41:05+00:00"},{"combined_score":0.4825,"content_id":"57103b51-1e89-4a5e-b9e9-cfadcdc44ff8","creator_id":"milestone5-unsure","guess":"unsure","label":"We cannot tell reliably who wrote this text. You can request a review.","model_score":0.699,"status":"decided","style_score":0.266,"timestamp":"2026-11-02T22:41:07+00:00"},{"content_id":"d58d5a7f-041f-4759-af09-5c6e0a9e031e","creator_id":"milestone5-ai","event":"appeal","reasoning":"I wrote this repeated sentence myself; please review the decision.","status":"under_review","timestamp":"2026-11-02T22:41:10+00:00"}]}
```

---

## How I Used AI

**Moment 1**

- *What I asked for:* A submission trace, two signal descriptions with blind
  spots, a false-positive example, and endpoint contracts for Milestone 1.
- *What came back:* An AI-written design with an equal-weight scoring rule,
  a wide unsure band, and a review path.
- *What changed:* The assistant added the design to this README; later work
  implemented the scoring rule, labels, and appeal path. The threshold is a
  starting choice, not an experimentally validated guarantee.

**Moment 2**

- *What I asked for:* Help with programming assistance when wiring up the API for the required functions
- *What came back:* Some neatly generated code that I reveiwed for correct and effective outputs.
- *What changed:* The bits to be replaced in the applciation's code.

<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
