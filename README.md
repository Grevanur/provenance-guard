# Provenance Guard

**Loom Video Link** - https://www.loom.com/share/4addcd61ad284f349d19430fe8c603fd

A backend API for AI-content attribution on creative platforms. Accepts submitted text or structured metadata, classifies it using a three-signal ensemble detection pipeline, returns a confidence score and transparency label, supports creator appeals and provenance certificates, provides an analytics dashboard, and logs every decision in a structured audit log.

**Stack:** Flask · Flask-Limiter · Groq (llama-3.3-70b-versatile) · SQLite · Python 3.11+

---

## Setup

```bash
git clone <your-repo-url>
cd provenance-guard
python -m venv .venv
source .venv/bin/activate        # Mac/Linux
pip install -r requirements.txt
```

Create a `.env` file in the project root (never commit this):

```
GROQ_API_KEY=your_key_here
```

Run the server:

```bash
python app.py
```

The API runs at `http://localhost:5001`.

> **Note:** Port 5000 is reserved by macOS AirPlay Receiver. The app defaults to 5001. Override with `PORT=5000 python app.py` if needed on non-macOS systems.

---

## Architecture Overview

A submitted piece of text travels through the following path:

1. **`POST /submit`** receives `{text, creator_id}` and checks the rate limit.
2. **Signal 1 — LLM classification:** Text is sent to Groq's `llama-3.3-70b-versatile` with a structured prompt asking for an `ai_probability` score (0.0–1.0).
3. **Signal 2 — Stylometric heuristics:** Three structural metrics are computed in pure Python: sentence-length variance, type-token ratio, and period-ending ratio. Combined into `stylo_score` (0.0–1.0).
4. **Signal 3 — Lexical burstiness:** Word-length coefficient of variation and unique bigram ratio are computed, capturing a lexical distribution dimension independent of signals 1 and 2. Combined into `burstiness_score` (0.0–1.0).
5. **Confidence scoring:** `0.70 × llm_score + 0.20 × stylo_score + 0.10 × burstiness_score`.
6. **Attribution:** The confidence score maps to one of three categories — `likely_ai`, `uncertain`, or `likely_human` — using asymmetric thresholds.
7. **Transparency label:** A plain-language label is selected based on the attribution and formatted with the confidence percentage.
8. **Audit log:** The full decision (all three signal scores, combined confidence, label, status) is written to SQLite.
9. **Response:** JSON containing `content_id`, `attribution`, `confidence`, `signals` (all three), `transparency_label`, and `status`.

For appeals: `POST /appeal` accepts `content_id` + `creator_reasoning`, updates the audit log entry to `under_review`, and returns a confirmation. `GET /log` surfaces all entries for review.

```
POST /submit ──► rate check ──► llm_signal() + stylo_signal() + burstiness_signal()
                                                    │
                                             combine_signals()
                                                    │
                                           get_attribution() ──► generate_label()
                                                    │
                                              log_submission() ──► JSON response

POST /appeal  ──► fetch_entry() ──► update_appeal()  ──► JSON response
POST /certify ──► fetch_entry() ──► store_certificate() ──► JSON response
GET  /content/<id> ──► fetch_entry() + get_certificate() ──► JSON response
GET  /analytics ──► aggregate audit_log ──► JSON response
GET  /log       ──► fetch_log() ──► JSON response
```

---

## Detection Signals

### Signal 1: LLM Classification (Groq)

**What it measures:** Holistic semantic and stylistic coherence — whether the writing reads like it came from a language model. The LLM assesses hedging language, generic framing, absence of personal voice, and phrasing patterns typical of AI output.

**Why it differs:** LLMs produce text that is statistically "average" — grammatically clean and coherent but lacking the idiosyncrasy of a real person. A strong LLM can recognize the very patterns it generates.

**What it misses:** Very short texts give the model too little to analyze. Polished formal human writing (academic papers, journalism) can trigger false positives. An LLM instructed to write "casually" can fool this signal.

### Signal 2: Stylometric Heuristics (pure Python)

**What it measures:** Three structural properties:
- **Sentence-length variance:** AI text is more uniform; human text is more irregular.
- **Type-token ratio (TTR):** Ratio of unique words to total words. AI text reuses vocabulary at a slightly higher rate.
- **Period-ending ratio:** Fraction of sentences ending with a period. AI prose is grammatically "complete" at a more consistent rate.

Combined as: `0.4 × variance_score + 0.4 × ttr_score + 0.2 × punct_score`

**Why it differs:** AI models are trained to produce clean, readable prose. That cleanliness shows up as measurable uniformity across these structural features.

**What it misses:** Very short texts produce unreliable variance estimates. Formal human genres (legal, academic) score high on period-ending ratio and low on TTR — they can look AI-like on this signal. Intentional repetition in poetry (anaphora, litany) also inflates the score.

### Signal 3: Linguistic Pattern Density (pure Python — ensemble stretch)

**What it measures:** Two linguistic fingerprinting properties independent of both semantic coherence (Signal 1) and sentence-level structure (Signal 2):
- **Sentence-starter repetition:** AI text frequently reuses a small set of sentence-opening words ("It", "The", "This", "Furthermore", "Moreover"). Lower variety of first-words → more AI-like.
- **Boilerplate phrase density:** AI formal prose uses hedging/transitional phrases ("it is important to", "furthermore", "it is essential", "in conclusion", "this enables") at a measurably higher rate than human writing. Measured as occurrences per 100 words. Higher density → more AI-like.

Combined as: `0.5 × starter_score + 0.5 × density_score`

**Why it differs:** This captures *lexical fingerprinting* — the specific patterns of language choice that LLMs use disproportionately. A text can have varied sentence lengths (fooling Signal 2) and even partially confuse the LLM (Signal 1) while still exhibiting the formulaic opener/phrase patterns that characterize AI output.

**What it misses:** Academic human writing may use some transitional phrases. Texts with fewer than 3 sentences return neutral (0.5) — the signal needs enough structure to measure.

### Ensemble weighting and conflict resolution

```
confidence = 0.70 × llm_score + 0.20 × stylo_score + 0.10 × burstiness_score
```

The LLM carries the most weight because it captures holistic semantic coherence. Stylometric is second — reliable at scale but degrades on short texts. The linguistic pattern signal is a corroborating layer that requires ≥ 3 sentences to activate (returns 0.5 otherwise).

**When signals conflict:** The LLM's judgment dominates. A split (LLM=0.8, stylo=0.3, burstiness=0.4) produces `0.8×0.70 + 0.3×0.20 + 0.4×0.10 = 0.660` — landing in the uncertain zone rather than forcing a binary flip. Signal disagreement is itself evidence of uncertainty.

---

## Confidence Scoring

```
confidence = 0.70 × llm_score + 0.20 × stylo_score + 0.10 × burstiness_score
```

**Threshold design (asymmetric by intent):**

| Confidence | Attribution | Label shown |
|---|---|---|
| ≥ 0.75 | `likely_ai` | AI-Detected label |
| 0.36 – 0.74 | `uncertain` | Uncertainty notice |
| ≤ 0.35 | `likely_human` | Human-authorship confirmation |

The asymmetry is deliberate: a false positive (calling a human's work AI-generated) harms a creator's reputation and trust in the platform. A false negative (missing an AI submission) is a lighter cost. Raising the AI threshold to 0.75 means the system must see strong evidence before applying the AI label.

**Example submissions with different confidence scores (real API output):**

| Input type | LLM score | Stylo score | Burstiness score | Combined confidence | Attribution |
|---|---|---|---|---|---|
| "Artificial intelligence represents a transformative paradigm shift in how humanity approaches complex problems. It is important to note that while the benefits of AI are numerous, it is essential to consider the ethical implications. Furthermore, stakeholders across various sectors must collaborate..." | 0.90 | 0.37 | 0.50 | **0.754** | `likely_ai` |
| "ok so i finally tried that ramen place downtown and honestly? underwhelming. broth was fine but WAY too much sodium, i was thirsty for like three hours after. my friend got the spicy version said it was better. probably wont go back unless someone drags me there lol" | 0.20 | 0.39 | 0.00 | **0.218** | `likely_human` |

The 0.754 vs 0.218 spread (real API output) confirms the scoring produces meaningful variation. The human text's burstiness score of 0.0 reflects zero AI-style boilerplate phrases; the AI text scores 0.50 (neutral, because its sentence starters happen to be diverse, but its phrase density contributes).

---

## Transparency Labels

All three variants as they appear in API responses:

### High-confidence AI (`attribution: "likely_ai"`, confidence ≥ 0.75)

> "AI-Assisted Content Detected — High Confidence ({pct}%)
> Our system found strong signals of AI-generated writing in this submission, including uniform sentence structure and phrasing patterns typical of language models. This label does not prevent the content from being shared, but is shown to readers for context. If you are the human author of this work and believe this is incorrect, you may submit an appeal and a human reviewer will examine your case."

### High-confidence Human (`attribution: "likely_human"`, confidence ≤ 0.35)

> "Original Human Authorship — High Confidence ({pct}%)
> Our system found strong signals of human authorship in this submission: natural variation in sentence rhythm, vocabulary diversity, and stylistic irregularities consistent with original human writing. No AI-generation label is shown to readers."

### Uncertain (`attribution: "uncertain"`, 0.35 < confidence < 0.75)

> "Origin Unclear — Further Review May Be Needed ({pct}% confidence)
> Our system detected a mixed signal and could not confidently determine whether this content was written by a human or generated by AI. It may be human-written, AI-assisted, or AI-generated with heavy human editing. Out of caution, this content is not labeled as AI-generated. If you feel the ambiguity reflects poorly on your work, you may submit an appeal to request human review."

---

## Provenance Certificate (Stretch)

A creator can earn a **Verified Human Author** certificate by completing a human-attestation step.

**Verification step:** The creator calls `POST /certify` with their `content_id` and a `human_statement` — a first-person signed statement (minimum 20 characters) attesting they are the human author of the specific work.

**Certificate design:** The `verified_label` is visually and textually distinct from the standard `transparency_label`:
- Standard label describes the automated classification result only.
- Verified label begins with `[VERIFIED HUMAN AUTHOR — Certificate <short-id>]`, includes the creator's signed statement verbatim, the issuance date, and notes the automated classification on file alongside it.

The certificate does not override the automated classification — both are shown to readers via `GET /content/<content_id>`, letting them see the system's verdict and the creator's attestation together.

**Example verified label:**

> "[VERIFIED HUMAN AUTHOR — Certificate abc12345]
> The creator has completed a human-attestation verification step for this submission. Creator's signed statement: "I wrote every word of this poem myself over three drafts in November."
> Issued: 2026-06-27 to creator 'user-42'.
> Automated classification on file: uncertain (confidence 58%). This certificate is an additional layer of context shown to readers alongside the automated result — it does not override the classification."

---

## Analytics Dashboard (Stretch)

`GET /analytics` returns three metrics across all submissions:

Real output from `GET /analytics` after 4 test submissions and 1 appeal:

```json
{
  "total_submissions": 4,
  "detection_pattern": {
    "likely_ai":    { "count": 2, "percentage": 50.0 },
    "likely_human": { "count": 1, "percentage": 25.0 },
    "uncertain":    { "count": 1, "percentage": 25.0 }
  },
  "appeal_rate": {
    "appealed_count": 1,
    "total": 4,
    "rate_percent": 25.0
  },
  "avg_confidence_by_attribution": {
    "likely_ai":    0.8171,
    "likely_human": 0.218,
    "uncertain":    0.65
  }
}
```

**Metric 1 — Detection pattern:** ratio of AI vs human vs uncertain verdicts. Useful for monitoring platform-wide trends and detecting drift.

**Metric 2 — Appeal rate:** fraction of submissions that have been appealed. A high appeal rate signals either poor calibration or that the confident-zone threshold is too low.

**Metric 3 — Average confidence by attribution:** how certain the system is within each category. If `avg_confidence` for `likely_ai` drops below 0.80, the system may be overcalling AI at the margin.

---

## Multi-Modal Support (Stretch)

`POST /submit` accepts a second content type — **structured metadata** — in addition to plain text.

**Request format:**
```json
{
  "content_type": "metadata",
  "creator_id": "user-123",
  "metadata": {
    "title": "Echoes of Tomorrow",
    "description": "A journey through the intersection of memory and identity...",
    "tags": ["fiction", "literary", "identity", "memory", "drama", "contemporary", "debut"],
    "word_count": 4200
  }
}
```

**Signals for metadata:**
- **LLM signal (`metadata_llm_signal`):** Groq evaluates whether the metadata reads as AI-generated — focusing on formulaic titles, generic phrases in descriptions ("a journey through the intersection of"), and systematically exhaustive tag lists.
- **Structural signal (`metadata_structural_signal`):** Three heuristics:
  1. *Personal pronoun presence in description* — human creators say "I wrote this" or "my story"; AI descriptions use impersonal third person.
  2. *Generic AI phrase detection* — a curated list of phrases LLMs disproportionately use in content summaries.
  3. *Tag count* — AI produces exhaustive systematic tags; humans pick a small specific set (1–3 tags is typical).

**Combining:** `confidence = 0.60 × llm_score + 0.40 × structural_score` (burstiness is not applicable to structured metadata fields).

**Response:** Identical structure to text submissions — same `content_id`, `attribution`, `confidence`, `transparency_label`, `status` fields. The `signals` object contains `llm_score` and `structural_score` instead of three text-specific scores.

---

## API Reference

### `POST /submit`

Submit text or metadata for attribution analysis.

**Text request:**
```json
{
  "text": "Your poem, story excerpt, or blog post here...",
  "creator_id": "user-123"
}
```

**Metadata request:**
```json
{
  "content_type": "metadata",
  "creator_id": "user-123",
  "metadata": {
    "title": "My Story Title",
    "description": "A brief description written by the creator...",
    "tags": ["fiction", "drama"]
  }
}
```

**Response (text):**
```json
{
  "content_id": "3f7a2b1e-...",
  "content_type": "text",
  "attribution": "likely_ai",
  "confidence": 0.8121,
  "signals": {
    "llm_score": 0.90,
    "stylo_score": 0.57,
    "burstiness_score": 0.61
  },
  "transparency_label": "AI-Assisted Content Detected — High Confidence (81%)...",
  "status": "classified"
}
```

**Rate limits:** 10 per minute, 100 per day (see Rate Limiting section).

---

### `POST /appeal`

Contest a classification.

**Request:**
```json
{
  "content_id": "3f7a2b1e-...",
  "creator_reasoning": "I wrote this myself. The formal style is because it's a cover letter."
}
```

**Response:**
```json
{
  "message": "Your appeal has been received. This content is now under human review.",
  "content_id": "3f7a2b1e-...",
  "original_attribution": "likely_ai",
  "status": "under_review"
}
```

---

### `POST /certify`

Earn a Verified Human Author provenance certificate.

**Request:**
```json
{
  "content_id": "3f7a2b1e-...",
  "creator_id": "user-123",
  "human_statement": "I wrote every word of this poem myself across three drafts in November 2025."
}
```

**Response:**
```json
{
  "certificate_id": "abc12345-...",
  "content_id": "3f7a2b1e-...",
  "creator_id": "user-123",
  "issued_at": "2026-06-27T10:00:00+00:00",
  "verified_label": "[VERIFIED HUMAN AUTHOR — Certificate abc12345]...",
  "message": "Provenance certificate issued. Your verified human-authorship label is now attached to this content."
}
```

---

### `GET /content/<content_id>`

Returns the full audit entry plus certificate status for a given submission.

**Response:**
```json
{
  "entry": { "content_id": "...", "attribution": "uncertain", "confidence": 0.58, ... },
  "certificate": {
    "certificate_id": "abc12345-...",
    "verified_label": "[VERIFIED HUMAN AUTHOR — Certificate abc12345]...",
    "issued_at": "2026-06-27T10:00:00+00:00"
  }
}
```

`certificate` is `null` if no certificate has been issued.

---

### `GET /analytics`

Returns 3 metrics across all submissions.

**Response:** See Analytics Dashboard section above.

---

### `GET /log`

Returns audit log entries (most recent first).

**Query params:** `?limit=20` (default 20)

**Response:**
```json
{
  "entries": [
    {
      "content_id": "3f7a2b1e-...",
      "creator_id": "user-123",
      "timestamp": "2026-06-27T04:08:35+00:00",
      "text_snippet": "Artificial intelligence represents...",
      "content_type": "text",
      "attribution": "likely_ai",
      "confidence": 0.8121,
      "llm_score": 0.90,
      "stylo_score": 0.57,
      "burstiness_score": 0.61,
      "transparency_label": "AI-Assisted Content Detected...",
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    }
  ],
  "count": 1
}
```

---

## Rate Limiting

**Limits:** 10 requests per minute · 100 requests per day (applied to `POST /submit`)

**Reasoning:** A real creative writer submitting their own work won't need more than a few submissions per minute — even with iterative drafts, 10/minute is generous. The 100/day cap covers a very productive day of writing while making bulk automated abuse (scraping, mass-submission attacks) expensive without a new IP. These values reflect the use case: a person, not a script.

**Evidence (rate limit test output):**

Run this while the server is running to verify:

```bash
for i in $(seq 1 12); do
  curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:5001/submit \
    -H "Content-Type: application/json" \
    -d '{"text": "This is a test submission for rate limit testing purposes only.", "creator_id": "test"}'
done
```

Actual output (the number of initial 200s reflects how many requests remain in the current minute's budget; once the per-minute counter hits 10, all subsequent requests return 429):
```
200
200
200
200
200
200
429
429
429
429
429
429
```

The above run started after 4 earlier test submissions had already consumed 4 of the 10/minute slots — so 6 more 200s appeared before the limiter triggered. If run on a fresh minute the first 10 return 200 and the last 2 return 429. Either way, 429 responses confirm the limiter is active.

---

## Audit Log Sample

Real output from `GET /log` after 3 text submissions, 1 metadata submission, and 1 appeal (most recent first):

```json
{
  "count": 4,
  "entries": [
    {
      "id": 4,
      "content_id": "1310c15d-d94f-429f-a86c-6ebbc339c80a",
      "creator_id": "user-4",
      "timestamp": "2026-06-27T04:39:08.622208+00:00",
      "text_snippet": "[metadata] title='A Journey Through the Intersection of Loss and Identity'",
      "content_type": "metadata",
      "attribution": "likely_ai",
      "confidence": 0.88,
      "llm_score": 0.8,
      "stylo_score": 1.0,
      "burstiness_score": null,
      "transparency_label": "AI-Assisted Content Detected — High Confidence (88%)...",
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "id": 3,
      "content_id": "f06c98c7-3232-4c3f-a9d2-9186635a4c45",
      "creator_id": "user-3",
      "timestamp": "2026-06-27T04:38:09.776735+00:00",
      "text_snippet": "The relationship between monetary policy and asset price inflation has been extensively studied in the literature. Central banks face a fundamental tension...",
      "content_type": "text",
      "attribution": "uncertain",
      "confidence": 0.65,
      "llm_score": 0.8,
      "stylo_score": 0.2,
      "burstiness_score": 0.5,
      "transparency_label": "Origin Unclear — Further Review May Be Needed (65% confidence)...",
      "status": "under_review",
      "appeal_reasoning": "I am a finance professor and this is an excerpt from my published research paper. The formal style is required by the journal.",
      "appeal_timestamp": "2026-06-27T04:38:20.191249+00:00"
    },
    {
      "id": 2,
      "content_id": "c9414ef0-e981-4d73-8d48-376232ddad0e",
      "creator_id": "user-2",
      "timestamp": "2026-06-27T04:38:01.177112+00:00",
      "text_snippet": "ok so i finally tried that ramen place downtown and honestly? underwhelming. broth was fine but WAY too much sodium...",
      "content_type": "text",
      "attribution": "likely_human",
      "confidence": 0.218,
      "llm_score": 0.2,
      "stylo_score": 0.39,
      "burstiness_score": 0.0,
      "transparency_label": "Original Human Authorship — High Confidence (22%)...",
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    },
    {
      "id": 1,
      "content_id": "4aab2dc2-49bb-4502-a068-99416ffc7b1f",
      "creator_id": "user-1",
      "timestamp": "2026-06-27T04:37:47.313708+00:00",
      "text_snippet": "Artificial intelligence represents a transformative paradigm shift in how humanity approaches complex problems. It is important to note that while the benefits of AI are numerous...",
      "content_type": "text",
      "attribution": "likely_ai",
      "confidence": 0.7541,
      "llm_score": 0.9,
      "stylo_score": 0.3707,
      "burstiness_score": 0.5,
      "transparency_label": "AI-Assisted Content Detected — High Confidence (75%)...",
      "status": "classified",
      "appeal_reasoning": null,
      "appeal_timestamp": null
    }
  ]
}
```

---

## Known Limitations

**Formal human writing (academic, legal, journalistic) triggers false positives.** The stylometric signal is calibrated on everyday human writing. Academic prose has low type-token ratio (domain jargon repeats), consistent sentence termination, and moderate sentence-length variance — all features that score as AI-like on this signal. A constitutional law review article by a human expert will likely land in the uncertain zone or, combined with an LLM signal that notices formal hedging language, could cross the 0.75 threshold incorrectly. This is a property of the signal itself, not a calibration issue: structural uniformity is genuinely shared between formal human writing and AI output, and no purely structural metric can cleanly separate them.

**Very short texts (< ~80 words) are unreliable.** Sentence-length variance computed over 2–3 sentences has high variance itself — a single unusually short sentence can swing the stylo score 0.2 points. The LLM signal also degrades because there is less text to analyze semantically. The burstiness signal requires at least 15 words before it produces a meaningful estimate. The system returns a result for any text over 20 characters, but results under 80 words should be treated as low-confidence by any consumer.

---

## Spec Reflection

**One way the spec helped:** Writing the transparency label variants in `planning.md` before touching the implementation forced a decision that would have otherwise been deferred until too late: the asymmetric thresholds. Once the labels were written out, it was obvious that the "uncertain" label — which explicitly says "we are not labeling this as AI-generated out of caution" — required the uncertain zone to be wide enough to matter. That pushed the AI threshold up to 0.75 instead of the intuitive 0.5.

**One way implementation diverged from the spec:** The spec described the stylometric signal as computing "sentence length variance, type-token ratio, punctuation density." During implementation, "punctuation density" (count of commas, semicolons, etc.) turned out to be nearly useless — AI and human text at similar lengths have nearly identical punctuation counts. It was replaced with "period-ending ratio" (fraction of sentences that are grammatically complete), which showed much clearer separation between formal AI prose and casual human writing.

---

## AI Usage

**Instance 1: Flask app skeleton + LLM signal function**
I provided Claude with the Detection Signals section of `planning.md` plus the Architecture ASCII diagram and asked it to generate: (1) the Flask app skeleton with the `POST /submit` route stub returning a hardcoded response, and (2) the `llm_signal()` function that calls Groq and parses a JSON response. The generated Groq call used an older API style (`openai`-compatible wrapper) that didn't match the `groq` SDK. I revised the import and the `client.chat.completions.create()` call to use the current Groq Python SDK directly.

**Instance 2: Stylometric signal + confidence scoring**
I provided the Uncertainty Representation section (thresholds table) and asked for `stylo_signal()` using the three metrics I had specified. The generated function combined the three metrics with equal weights (0.33 each). I overrode the weighting to 0.4/0.4/0.2 because the period-ending ratio is a weaker signal than sentence variance or TTR, and equal weighting would have given it too much influence on the combined score.

**Instance 3: Ensemble signal + stretch features**
I provided the Stretch Features section of `planning.md` and asked for (1) a 3rd detection signal, (2) the `POST /certify` endpoint with SQLite certificate storage, and (3) `GET /analytics`. The generated 3rd signal used word-length coefficient of variation (CV) and unique bigram ratio. During testing I found that English prose — AI and human alike — has high CV because it always mixes short function words ("a", "it", "the") with longer content words. The CV signal scored nearly all text as human-like (near 0.0), failing to differentiate. I replaced it with a boilerplate phrase density signal (counting AI-typical phrases like "it is important to", "furthermore", "it is essential to" per 100 words) combined with sentence-starter variety — both of which genuinely differ between AI formal prose and human writing. I also added the conflict-resolution docstring to `combine_signals()` manually, as the generated version had no explanation for how disagreeing signals were handled.