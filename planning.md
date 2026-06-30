# Provenance Guard — Planning Document

## Architecture

### Submission Flow Narrative

A piece of text enters the system through `POST /submit` along with a `creator_id`. The request is validated (non-empty text, present creator ID), then passed in parallel to two independent detection signals. Signal 1 sends the text to Groq's `llama-3.3-70b-versatile` model with a structured prompt that returns an AI-likelihood score between 0 and 1. Signal 2 runs three stylometric heuristics directly in Python — sentence-length variance, type-token ratio, and punctuation density — and combines them into a second AI-likelihood score between 0 and 1. Both scores feed the confidence scoring component, which combines them using a weighted average (60% LLM, 40% stylometric) and maps the result to an attribution verdict and one of three transparency label variants. The full record — content ID, creator ID, timestamp, both signal scores, combined confidence, attribution verdict, label text, and status — is written to a structured JSON audit log. The complete response is returned to the client.

If a creator disputes the result, `POST /appeal` accepts their `content_id` and `creator_reasoning`. The system looks up the original record, updates its status to `"under_review"`, appends the appeal reasoning to the audit log entry, and returns a confirmation. No automated re-classification occurs; a human reviewer would inspect the log.

### Architecture Diagram

```
SUBMISSION FLOW

  client ──POST /submit {text, creator_id}──▶ [validate input]
                                                      │
                          ┌───────────────────────────┼───────────────────────────┐
                          │ raw text                                    raw text   │
                          ▼                                                        ▼
          ┌─────────────────────────────┐                  ┌──────────────────────────────────┐
          │ Signal 1: Groq LLM          │                  │ Signal 2: Stylometric Heuristics │
          │ llama-3.3-70b-versatile     │                  │ - sentence length variance        │
          │ → ai_score_1  (0.0 – 1.0)  │                  │ - type-token ratio                │
          └─────────────────────────────┘                  │ - punctuation density             │
                          │                                │ → ai_score_2  (0.0 – 1.0)        │
                          │                                └──────────────────────────────────┘
                          └──────────── ai_score_1 ──┐  ┌── ai_score_2 ──────────────────────┘
                                                     ▼  ▼
                                         ┌───────────────────────────┐
                                         │ Confidence Scoring        │
                                         │ combined = 0.6*s1 + 0.4*s2│
                                         │ → confidence (0.0 – 1.0)  │
                                         │ → attribution verdict      │
                                         └───────────────────────────┘
                                                     │ confidence + verdict
                                                     ▼
                                         ┌───────────────────────────┐
                                         │ Label Generator           │
                                         │ → label text (1 of 3)     │
                                         └───────────────────────────┘
                                                     │ full record
                                                     ▼
                                         ┌───────────────────────────┐
                                         │ Audit Log (JSON write)    │
                                         └───────────────────────────┘
                                                     │
                                                     ▼
            response: {content_id, attribution, confidence, signals, label}


APPEAL FLOW

  client ──POST /appeal {content_id, creator_reasoning}──▶ [look up content_id]
                                                                    │ original record
                                                                    ▼
                                                        [update status → "under_review"]
                                                                    │ record + appeal text
                                                                    ▼
                                                        [audit log: append appeal fields]
                                                                    │
                                                                    ▼
            response: {content_id, status: "under_review", message}
```

---

## Detection Signals

### Signal 1 — Groq LLM Classifier (Semantic)

**What it measures:** Holistic semantic and stylistic patterns. The LLM evaluates whether the writing reads as human or AI-generated based on phrasing register, tonal consistency, structural predictability, and vocabulary choices.

**Why this differs between human and AI writing:** AI-generated text tends toward even hedging language ("it is important to note that"), smooth transitions, and consistent formal register. Human writing is messier — register shifts, personal asides, unexpected word choices, and emotional inconsistency.

**Output format:** A float between 0.0 and 1.0 representing AI likelihood. Returned as a structured JSON field from the Groq API response, parsed from the model's output.

**Blind spots:**
- Non-native English speakers often write in a formal, uniform register that the LLM flags as AI-like — this is the primary false-positive risk.
- Lightly edited AI output can fool the detector if enough human irregularity is introduced.
- Non-deterministic: the same input may produce slightly different scores across calls.
- Short texts (under ~50 words) give the model too little signal; scores become unreliable.

---

### Signal 2 — Stylometric Heuristics (Structural)

**What it measures:** Three statistical properties of the text computed in pure Python:

1. **Sentence-length variance:** The standard deviation of word counts across sentences. AI text tends to be uniform; human writing is bursty (mix of short punchy sentences and long complex ones).
2. **Type-token ratio (TTR):** Unique words divided by total words. AI text reuses vocabulary more than humans do, especially in longer passages.
3. **Punctuation density:** Punctuation characters divided by total characters. Human writing tends to use more varied punctuation (em-dashes, ellipses, exclamation marks); AI output is more conservative.

**Why this differs between human and AI writing:** AI models are trained to minimize perplexity — they produce statistically "safe," predictable text. This shows up as measurable uniformity across all three metrics.

**Output format:** Each metric is normalized to a 0–1 range and combined into a single `ai_score_2` float via a simple average. A high score means the text looks statistically AI-like.

**Blind spots:**
- Short texts produce unreliable variance and TTR statistics (< 3 sentences or < 30 words).
- Formal human writing (academic papers, legal documents) naturally scores high — similar uniformity to AI.
- Poetry with heavy repetition (anaphora, refrains) will score AI-like due to low TTR and uniform line lengths.
- Human writers who naturally use simple, short sentences (casual blog posts) may score low even if other signals are high.

---

## Uncertainty Representation

### What confidence scores mean

The combined confidence score represents **AI likelihood** on a 0.0–1.0 scale. It is not a probability in the strict statistical sense — it is a calibrated estimate based on two independent signals.

| Score range | Meaning |
|---|---|
| 0.00 – 0.35 | Strong evidence of human authorship |
| 0.36 – 0.64 | Ambiguous — neither signal is conclusive |
| 0.65 – 1.00 | Strong evidence of AI generation |

**Asymmetric thresholds:** The "likely AI" threshold is set higher (0.65) than the midpoint on purpose. A false positive — labeling a human's work as AI — is worse than a false negative on a writing platform. The system should require stronger evidence to make the AI accusation than to clear someone.

### How scores are combined

```
combined = (0.6 × llm_score) + (0.4 × stylometric_score)
```

The LLM signal gets more weight (60%) because it captures semantic coherence holistically. The stylometric signal (40%) adds structural grounding but is more susceptible to content-type false positives. Neither signal alone determines the verdict.

### What 0.6 means specifically

A score of 0.60 falls in the uncertain band (0.36–0.64). The system would return an "Uncertain" label. This is intentional — 0.60 is not strong enough evidence to accuse anyone of AI generation. It reflects a system that genuinely doesn't know and says so.

---

## Transparency Label Design

Three variants, mapped to confidence score ranges:

### High-confidence AI (confidence ≥ 0.65)

```
⚠️ AI-Generated Content Likely

Our system has analyzed this submission and found strong indicators that it
may have been generated with AI assistance (confidence: {score}%).

This label does not mean the content has no value — it's here to help readers
understand how it was created. If you are the creator and believe this is
incorrect, you can submit an appeal below.
```

### Uncertain (0.36 ≤ confidence < 0.65)

```
🔍 Authorship Uncertain

Our system analyzed this submission but could not confidently determine
whether it was written by a person or generated with AI assistance
(confidence: {score}%).

Some writing styles, non-native English, or formal prose can be difficult
to classify. If you are the creator, you may submit an appeal to add context.
```

### High-confidence Human (confidence < 0.36)

```
✅ Likely Human-Written

Our system found strong indicators that this content was written by a person
(confidence: {score}%).

This label reflects our best analysis and is not a guarantee. Our detection
system is not perfect.
```

*Note: `{score}%` is replaced at runtime with `round((1 - confidence) * 100)` for the human label and `round(confidence * 100)` for the AI label, so the percentage shown always reflects the direction of the verdict.*

---

## Appeals Workflow

### Who can submit an appeal
Any creator who has received a `content_id` from a prior `/submit` call. No authentication is implemented in this version — the content ID acts as the access token.

### What they provide
- `content_id` — the ID returned by `/submit`
- `creator_reasoning` — a free-text explanation (e.g., "I am a non-native English speaker and my formal writing style may have triggered the detector")

### What the system does
1. Looks up the original audit log entry by `content_id`
2. Updates the entry's `status` field from `"classified"` to `"under_review"`
3. Appends `appeal_reasoning` and `appeal_timestamp` to the log entry
4. Returns a confirmation response: `{content_id, status: "under_review", message: "Your appeal has been received and will be reviewed."}`

### What a human reviewer would see
When a reviewer calls `GET /log`, they see entries with `"status": "under_review"` alongside the original `attribution`, `confidence`, `llm_score`, `stylometric_score`, and the creator's `appeal_reasoning`. This gives them everything needed to make a manual determination.

### What the system does NOT do
- No automated re-classification on appeal
- No email notification
- No auth layer (out of scope for this project)

---

## Anticipated Edge Cases

### Edge case 1: Non-native English formal prose
A creator who writes in careful, formal English as a second language produces text with uniform sentence structure, conservative vocabulary, and minimal colloquialisms. Signal 1 (LLM) flags it as AI-like due to register. Signal 2 (stylometric) flags it as AI-like due to low variance. Combined score lands in the 0.60–0.75 range — a false positive. **Mitigation:** The asymmetric threshold (0.65 for AI verdict) gives some buffer, and the uncertain band (0.36–0.64) catches borderline cases. The appeal workflow is the real escape hatch here.

### Edge case 2: Very short submissions (< 3 sentences)
A haiku, a two-sentence bio, or a single paragraph gives Signal 2 almost no data — sentence-length variance is undefined or meaningless with fewer than 3 sentences, and TTR is artificially high (short texts use more unique words by default). The stylometric score becomes noise. **Mitigation:** Add a minimum-length check; if the text is under 30 words, set `stylometric_score` to 0.5 (neutral) and flag `low_data_warning: true` in the response so the label can acknowledge the uncertainty.

### Edge case 3: Lightly edited AI output
A user generates text with an AI tool, then manually edits 20–30% of it — changing phrasing, adding personal details, breaking up uniform sentences. Signal 2 may now score it as human-like (variance is introduced), while Signal 1 may still catch the underlying AI register. The combined score could land in the uncertain band even though the content is substantially AI-generated. **Mitigation:** This is an acknowledged limitation of the current two-signal approach. The label in the uncertain band explicitly communicates that the system cannot make a confident determination.

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1

**Spec sections to provide:** Detection Signals (Signal 1 only) + Architecture Diagram (submission flow)

**What to ask for:**
- Flask app skeleton with `POST /submit` route stub that accepts `{text, creator_id}` and returns a hardcoded JSON response
- A standalone `groq_signal(text)` function that calls `llama-3.3-70b-versatile` with a structured prompt and returns a float between 0 and 1
- A `write_log(entry)` helper that appends a JSON object to `audit_log.json`
- A `GET /log` route that reads and returns the log

**How to verify:**
- Call `groq_signal()` directly with the four test inputs from Milestone 4 and print the raw scores before wiring into the endpoint
- Run `curl` against `POST /submit` and confirm the response includes `content_id`, `attribution`, `confidence` (placeholder), and `label` (placeholder)
- Call `GET /log` and confirm the entry appears

---

### Milestone 4 — Signal 2 + confidence scoring

**Spec sections to provide:** Detection Signals (Signal 2) + Uncertainty Representation + Architecture Diagram

**What to ask for:**
- A standalone `stylometric_signal(text)` function that computes sentence-length variance, TTR, and punctuation density, normalizes each to 0–1, and returns a combined float
- A `score_confidence(llm_score, stylometric_score)` function that applies the weighted average and returns `{confidence, attribution}` according to the thresholds in this spec
- Integration of both signals into the `/submit` endpoint

**How to verify:**
- Test `stylometric_signal()` on the four Milestone 4 test inputs independently
- Compare both signal scores side by side — do they agree on clear cases and diverge on borderline ones?
- Confirm that clearly AI-generated text scores above 0.65 and clearly human text scores below 0.36

---

### Milestone 5 — Production layer

**Spec sections to provide:** Transparency Label Design + Appeals Workflow + Architecture Diagram (both flows)

**What to ask for:**
- A `generate_label(confidence)` function that maps the score to one of the three label variants defined in this spec
- `POST /appeal` endpoint that accepts `{content_id, creator_reasoning}`, updates the log entry status, and returns a confirmation
- Flask-Limiter applied to `/submit` with `storage_uri="memory://"` and limits of `10 per minute; 100 per day`

**How to verify:**
- Submit inputs that produce all three confidence ranges and confirm all three label variants are reachable
- Submit an appeal with a known `content_id`, then call `GET /log` and confirm `status` is `"under_review"` and `appeal_reasoning` is populated
- Run the 12-request rate limit test from the spec and confirm requests 11–12 return HTTP 429