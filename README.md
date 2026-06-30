# Provenance Guard

A backend classification system that detects AI-generated text content and surfaces transparency labels to readers. Built for CodePath AI201 Project 4.

---

## Architecture Overview

A piece of text enters through `POST /submit` with a `creator_id`. It is validated, then passed to two independent detection signals running in sequence. **Signal 1** sends the text to Groq's `llama-3.3-70b-versatile` model, which returns an AI-likelihood score based on semantic and stylistic patterns. **Signal 2** computes three stylometric heuristics in pure Python — sentence-length variance, type-token ratio, and punctuation density — and combines them into a structural AI-likelihood score. Both scores feed a weighted confidence scoring function (60% LLM, 40% stylometric) that produces a single calibrated confidence score and an attribution verdict. That score maps to one of three transparency label variants. The full record is written to a structured JSON audit log and returned to the client with a `content_id`.

If a creator disputes a classification, `POST /appeal` accepts their `content_id` and reasoning, updates the log entry's status to `under_review`, and appends the appeal context for human review. No automated re-classification occurs.

```
SUBMISSION FLOW

  client ──POST /submit {text, creator_id}──▶ [validate input]
                                                      │
                    ┌─────────────────────────────────┼─────────────────────────────────┐
                    │ raw text                                                raw text   │
                    ▼                                                                    ▼
    ┌─────────────────────────────┐                      ┌──────────────────────────────────┐
    │ Signal 1: Groq LLM          │                      │ Signal 2: Stylometric Heuristics │
    │ llama-3.3-70b-versatile     │                      │ - sentence length variance        │
    │ → llm_score  (0.0 – 1.0)   │                      │ - type-token ratio                │
    └─────────────────────────────┘                      │ - punctuation density             │
                    │                                    │ → stylometric_score (0.0 – 1.0)  │
                    └──────── llm_score ──┐  ┌── stylometric_score ──────────────────────┘
                                         ▼  ▼
                             ┌───────────────────────────┐
                             │ Confidence Scoring        │
                             │ 0.6*llm + 0.4*stylo       │
                             │ → confidence (0.0 – 1.0)  │
                             │ → attribution verdict      │
                             └───────────────────────────┘
                                         │
                                         ▼
                             ┌───────────────────────────┐
                             │ Label Generator           │
                             │ → label text (1 of 3)     │
                             └───────────────────────────┘
                                         │
                                         ▼
                             ┌───────────────────────────┐
                             │ Audit Log (JSON write)    │
                             └───────────────────────────┘
                                         │
                                         ▼
        response: {content_id, attribution, confidence, signals, label}


APPEAL FLOW

  client ──POST /appeal {content_id, creator_reasoning}──▶ [look up content_id]
                                                                  │
                                                                  ▼
                                                    [update status → "under_review"]
                                                                  │
                                                                  ▼
                                                    [audit log: append appeal fields]
                                                                  │
                                                                  ▼
          response: {content_id, status: "under_review", message}
```

---

## Detection Signals

### Signal 1 — Groq LLM Classifier (Semantic)

**What it measures:** Holistic semantic and stylistic patterns. The model evaluates phrasing register, tonal consistency, structural predictability, and vocabulary choices — the kinds of patterns that differ between human and AI writing at the level of meaning, not just surface statistics.

**Why it differs between human and AI writing:** AI-generated text tends toward even hedging language ("it is important to note that"), smooth transitions, and consistent formal register. Human writing is messier — register shifts, personal asides, unexpected word choices.

**What it misses:** Non-native English speakers often write in a formal, uniform register that the LLM flags as AI-like. Lightly edited AI output can fool the detector if enough human irregularity is introduced. The model is also non-deterministic — the same input may produce slightly different scores across calls.

### Signal 2 — Stylometric Heuristics (Structural)

**What it measures:** Three statistical properties computed in pure Python:
- **Sentence-length variance:** Standard deviation of word counts across sentences. AI text is uniform; human writing is bursty.
- **Type-token ratio (TTR):** Unique words divided by total words. AI text reuses vocabulary more predictably.
- **Punctuation density:** Density of expressive punctuation (`!?—…;:()`). Human writing uses more varied punctuation; AI output is conservative.

**Why it differs between human and AI writing:** AI models are trained to minimize perplexity, producing statistically "safe," predictable text. This shows up as measurable uniformity across all three metrics.

**What it misses:** Short texts (under 30 words / 2 sentences) produce unreliable statistics — the system returns a neutral 0.5 and flags `low_data_warning: true`. Formal human writing (academic prose, legal documents) naturally scores AI-like. Poetry with heavy repetition will score high due to low TTR.

---

## Confidence Scoring

### How signals are combined

```
confidence = (0.6 × llm_score) + (0.4 × stylometric_score)
```

The LLM signal receives more weight (60%) because it captures semantic coherence holistically. The stylometric signal (40%) adds structural grounding but is more susceptible to content-type false positives.

### Thresholds

| Score range | Attribution |
|---|---|
| ≥ 0.65 | `likely_ai` |
| 0.35 – 0.64 | `uncertain` |
| < 0.35 | `likely_human` |

The "likely AI" threshold is intentionally set above the midpoint (0.65 rather than 0.50). A false positive — labeling a human's work as AI — is worse than a false negative on a writing platform. The system requires stronger evidence to make the AI accusation than to clear someone.

### Validation: two example submissions with noticeably different scores

**High-confidence AI (confidence: 0.69):**
```
Input: "Artificial intelligence represents a transformative paradigm shift in modern
society. It is important to note that while the benefits of AI are numerous, it is
equally essential to consider the ethical implications. Furthermore, stakeholders
across various sectors must collaborate to ensure responsible deployment."

llm_score: 0.8  |  stylometric_score: 0.524  |  confidence: 0.69  |  attribution: likely_ai
```

**Low-confidence human (confidence: 0.291):**
```
Input: "ok so i finally tried that new ramen place downtown and honestly?
underwhelming. the broth was fine but they put WAY too much sodium in it and i was
thirsty for like three hours after. my friend got the spicy version and said it was
better. probably wont go back unless someone drags me there"

llm_score: 0.2  |  stylometric_score: 0.428  |  confidence: 0.291  |  attribution: likely_human
```

The 0.399 gap between these two scores demonstrates meaningful variation — the system is not clustering near a constant.

---

## Transparency Label

Three variants, shown verbatim as returned by the API:

### High-confidence AI (`confidence ≥ 0.65`)

```
⚠️ AI-Generated Content Likely

Our system has analyzed this submission and found strong indicators that it
may have been generated with AI assistance (confidence: 69%).

This label does not mean the content has no value — it's here to help readers
understand how it was created. If you are the creator and believe this is
incorrect, you can submit an appeal.
```

### Uncertain (`0.35 ≤ confidence < 0.65`)

```
🔍 Authorship Uncertain

Our system analyzed this submission but could not confidently determine
whether it was written by a person or generated with AI assistance
(confidence: 63%).

Some writing styles, non-native English, or formal prose can be difficult
to classify. If you are the creator, you may submit an appeal to add context.
```

### Likely Human (`confidence < 0.35`)

```
✅ Likely Human-Written

Our system found strong indicators that this content was written by a person
(confidence: 80%).

This label reflects our best analysis and is not a guarantee. Our detection
system is not perfect.
```

*Note: for the human label, the percentage shown is `round((1 - confidence) * 100)` — it reflects how confident the system is that the author is human, not the AI score.*

---

## Rate Limiting

**Limits applied to `POST /submit`:** 10 requests per minute, 100 requests per day.

**Reasoning:** A real writer submitting their own work would realistically submit once or twice per session — 10 per minute is already generous for legitimate use. The per-minute limit prevents automated flooding (a script hitting the endpoint in a loop). The per-day limit of 100 allows heavy legitimate users while making bulk abuse expensive. The `/appeal` and `/log` endpoints are not rate-limited since appeals are infrequent by nature and `/log` is an internal monitoring route.

**Rate limit test — status codes from 12 rapid requests:**

```
200
200
200
200
200
200
200
200
200
429 Too Many Requests — 10 per 1 minute
429 Too Many Requests — 10 per 1 minute
429 Too Many Requests — 10 per 1 minute
```

Requests 1–9 succeeded. Requests 10–12 were blocked with HTTP 429, confirming the limit is enforced correctly.

---

## Audit Log

Every attribution decision is written to `audit_log.json`. Each entry captures: timestamp, content ID, creator ID, attribution verdict, combined confidence score, individual signal scores, low-data warning flag, and status. Appeals append `appeal_reasoning` and `appeal_timestamp` to the original entry and flip `status` to `under_review`.

Sample log output from `GET /log` (3 representative entries):

```json
[
  {
    "content_id": "ec45a9a4-cbc4-4c07-bf5c-d8d901710349",
    "creator_id": "test-ai",
    "timestamp": "2026-06-30T22:04:36.561434+00:00",
    "attribution": "likely_ai",
    "confidence": 0.69,
    "llm_score": 0.8,
    "stylometric_score": 0.5242,
    "low_data_warning": false,
    "status": "classified"
  },
  {
    "content_id": "3329531c-9582-4557-bb1d-e63698c9f27e",
    "creator_id": "test-human",
    "timestamp": "2026-06-30T21:52:24.475961+00:00",
    "attribution": "likely_human",
    "confidence": 0.291,
    "llm_score": 0.2,
    "stylometric_score": 0.4279,
    "low_data_warning": false,
    "status": "classified"
  },
  {
    "content_id": "54e3d1d6-da94-4879-aef9-39cfbeb60912",
    "creator_id": "test-user-1",
    "timestamp": "2026-06-30T21:47:43.782086+00:00",
    "attribution": "likely_human",
    "confidence": 0.2,
    "llm_score": 0.2,
    "stylometric_score": null,
    "low_data_warning": null,
    "status": "under_review",
    "appeal_reasoning": "I wrote this myself from personal experience. I am a non-native English speaker and my writing style may appear more formal than typical.",
    "appeal_timestamp": "2026-06-30T22:04:48.979935+00:00"
  }
]
```

---

## Known Limitations

**Non-native English writers are the primary false-positive risk.** Formal, careful prose written by a non-native speaker tends to score high on both signals — the LLM reads the register as AI-like, and the stylometrics read the uniformity as AI-like. The asymmetric threshold (0.65) provides some buffer, but this is an inherent limitation of both signal types. The appeals workflow is the real mitigation.

**Short submissions produce unreliable stylometric scores.** Sentence-length variance is mathematically unstable with fewer than 3 sentences, and TTR is artificially inflated in short texts. The system detects this (`low_data_warning: true`) and falls back to a neutral stylometric score of 0.5, but this means short content is classified almost entirely on the LLM signal alone.

---

## Spec Reflection

**One way the spec helped:** Writing out the three label variants in `planning.md` before any code forced a concrete decision about what "uncertain" actually communicates to a non-technical reader. Without that, the label would have been an afterthought bolted onto the confidence score. Having the exact text already written made the `generate_label()` function a straightforward translation, not a design problem.

**One way implementation diverged from the spec:** The planning doc specified that the stylometric signal would combine sentence-length variance, TTR, and punctuation density as a simple average. In practice, punctuation density turned out to be a weaker signal than expected — most text, human or AI, falls in a narrow range. The normalization constant (dividing by 0.02) required tuning against real test inputs before it contributed meaningfully rather than pushing every score toward 1.0. The spec said "simple average"; the implementation required calibrating the normalization first.

---

## AI Usage

**Instance 1 — Flask app skeleton and Signal 1:** I directed Claude to generate the Flask app skeleton with `POST /submit` and `GET /log` routes, plus a standalone `groq_signal()` function, using my planning.md detection signals section and architecture diagram as context. The generated function returned a raw string from the model rather than parsed JSON. I revised it to strip markdown code fences before `json.loads()` and added a try/except with a 0.5 fallback so a malformed model response wouldn't crash the endpoint.

**Instance 2 — Stylometric signal and confidence scoring:** I directed Claude to generate `stylometric_signal()` with the three sub-metrics and a `score_confidence()` weighting function, providing my uncertainty representation section from planning.md. The generated scoring function used a symmetric 0.5 threshold rather than the asymmetric 0.65/0.35 split I had specified. I corrected the thresholds to match the spec before integrating, since the whole point of the asymmetric design was to reduce false positives.

---

## Stack

- **Framework:** Flask 3.x
- **LLM:** Groq `llama-3.3-70b-versatile`
- **Rate limiting:** Flask-Limiter 4.x (in-memory storage)
- **Audit log:** JSON flat file (`audit_log.json`)
- **Stylometrics:** Pure Python (no external libraries)
- **Environment:** Python 3.12, Windows / PowerShell