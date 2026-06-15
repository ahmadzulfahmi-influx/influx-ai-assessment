# Influx AI Assessment — Engineering Overview

_Version 1.0 · Public · Last updated: 2026-06-15_

This document describes the technical architecture of the Influx AI-powered candidate assessment system. It covers the conversation pipeline, the voice-first extension, and the data contracts that connect each component.

Assessment criteria, trait definitions, scoring rubrics, and signal field schemas are defined separately in an internal spec and are not included here.

---

## 1. System Overview

The assessment system conducts a structured behavioral interview through a chat interface. Questions are delivered as text. Candidate responses are captured (typed or voice), scored by Claude, and routed to a recruiter dashboard.

The system is split into two sequential phases:

```
[Pre-Screen Phase]
  Conversational pre-screen — logistics and eligibility questions
    ↓ transcript
  Extractor call — structured JSON output (PrescreenResponses)
    ↓ deterministic knockout evaluation (server-side)
    ↓ prescreen_status + flagged reasons

[Behavioral Phase]
  Interviewer (Prompt A) — iterative, N turns
    ↓ full transcript
  Scorer (Prompt B) — single call, markdown narrative
    ↓
  Extractor (Prompt C) — single call, AssessmentReport JSON
    ↓ stored to DB
    ↓ fast-track routing logic
```

### Model targets

| Prompt | Role | Model |
|--------|------|-------|
| Prompt A | Behavioral interviewer | `claude-sonnet-4-6` |
| Prompt B | Scorer | `claude-sonnet-4-6` |
| Prompt C | Extractor (JSON) | `claude-haiku-4-5` |
| Prompt PE | Pre-screen extractor | `claude-haiku-4-5` |

---

## 2. Input Seam

The candidate input entry point (`getCandidateInput()`) is modality-aware:

```typescript
// Typed answer
{ text: string }

// Voice answer (after ASR)
{
  text: string,                      // transcript — the only thing that enters the prompt
  asr: {
    words: Array<{ w: string, start: number, end: number }>,
    confidence: number
  },
  media_ref: string | null           // pointer to audio blob (if retained)
}
```

Only `.text` is passed into `conversationHistory` and from there into Prompt A/B. All paralinguistic data (word timings, audio reference) rides as a sidecar and is never injected into the transcript string. This preserves the invariant that the content engine scores only what the candidate said, not how they said it.

---

## 3. Conversation Loop

```
while interview not complete:
  send system_prompt + conversationHistory → Claude (Prompt A)
  receive assistant_turn
  if assistant_turn contains [INTERVIEW_COMPLETE]:
    strip token, close loop
  else:
    append to conversationHistory
    getCandidateInput() → append to conversationHistory
```

The session sentinel `[INTERVIEW_COMPLETE]` is appended by Prompt A on a new line after the closing message. The application loop detects and strips it before display.

---

## 4. Fast-Track Routing

After Prompt C produces the `AssessmentReport`:

- `overall_score >= threshold` → inline scheduling panel (fast-track)
- `overall_score < threshold` → standard human review queue

The threshold is a fixed qualifier line set by HR — not a cohort-fitted value. Below-threshold candidates route to human review, not auto-reject.

---

## 5. Voice-First Extension

The system is designed for voice-first candidate answers (audio-only MVP; video deferred). Questions remain text. In-browser transcription converts voice to text before it enters the content pipeline — so Prompt A/B/C are completely unchanged by the voice modality.

### 5.1 Why in-browser ASR

Claude's API accepts text, images, and PDFs — not audio. An ASR step is mandatory regardless of where it runs. We chose **in-browser Whisper via [transformers.js](https://huggingface.co/docs/transformers.js)** for the following reasons:

- Audio never leaves the candidate's device — strongest privacy story for sensitive HR voice data
- $0 transcription cost per candidate
- Lightest consent/retention burden (no audio upload, no server-side storage required)
- One-time model download (~40–75MB), cached in the browser after the first session

Residual risk: weak or low-end devices may transcribe slowly or fail. Mitigations: device pre-check, graceful typed-answer fallback, transcription latency surfaced as a Layer 4 diagnostic.

### 5.2 ASR pipeline

```
getUserMedia({ audio: true })
  → MediaRecorder → Blob (audio/webm)
  → AudioContext({ sampleRate: 16000 }).decodeAudioData()
  → Float32Array (mono, 16kHz)
  → pipeline('automatic-speech-recognition', model, { return_timestamps: 'word' })
  → { text: string, chunks: [{ text, timestamp: [start, end] }] }
```

Word-level timestamps (`return_timestamps: 'word'`) are the linchpin for Layer 2. They require the WASM backend (ONNX Runtime Web CPU); the WebGPU backend may return only chunk-level timestamps. The voice lab (`voice-lab/index.html`) empirically confirms which backend and timestamp resolution the target hardware supports.

Model options:

| Model | Size | Notes |
|-------|------|-------|
| `Xenova/whisper-tiny.en` | ~40MB | Fast, English-only |
| `Xenova/whisper-base.en` | ~75MB | More accurate, English-only |

### 5.3 ASR adapter interface

The ASR engine is wrapped in a thin adapter so the backend (in-browser vs. server-side) can be swapped without touching the pipeline:

```typescript
interface AsrAdapter {
  transcribe(audioBlob: Blob): Promise<AsrResult>;
}

interface AsrResult {
  text: string;
  words: Array<{ w: string; start: number; end: number }>;
  confidence: number;
  latency_ms: number;
  engine: string;       // e.g. "whisper-tiny.en@wasm"
}
```

---

## 6. Four-Layer Signal Model

Each layer routes to the cheapest engine that can produce its signal. Only Layer 1 uses an LLM.

| Layer | Signal | Engine | Cost | Scored? |
|-------|--------|--------|------|---------|
| **1. Content** | What the candidate says | Prompt A/B/C — unchanged | existing | **Yes** |
| **2. Language** | Speaking rate, filler words, pauses, vocabulary breadth | ASR word timings → arithmetic | ~$0 | **Conditional** (see §6.2) |
| **3. Conditions** | Background noise / SNR | Web Audio AnalyserNode (client-side) | $0 | **No — diagnostic only** |
| **4. Hygiene** | Connection quality, device, transcription latency | Browser telemetry | $0 | **No — diagnostic only** |

### 6.1 Layer 2 — Language metrics (arithmetic, no LLM)

All metrics are derived from the word list produced by ASR. No additional model calls.

**Speaking rate (WPM)**
```
wpm = word_count / ((last_word.end - first_word.start) / 60)
```

**Filler word rate**
```
fillers = tokens matching /^(um+|uh+|er+|ah+|like|basically|literally|so|actually|right)$/i
filler_rate = filler_count / duration_minutes
```

**Pause count**
```
pauses = inter-word gaps where gap > 0.5s
         (requires word-level timestamps)
```

**Vocabulary breadth — Type-Token Ratio (TTR)**
```
ttr = unique_lemmas / total_tokens
      (lowercase, punctuation stripped)
```

**Proficiency verdict** (display label, separate axis from content score):

| Label | Criteria |
|-------|---------|
| `strong` | WPM 100–170 AND filler < 2/min AND TTR > 0.55 |
| `adequate` | WPM 80–200 AND filler < 4/min AND TTR > 0.40 |
| `limited` | anything else |

### 6.2 Layer 2 fairness constraints

Layer 2 is the highest adverse-impact surface — speaking rate, filler words, and vocabulary are proxies for accent and non-native English status. Two hard constraints apply before Layer 2 can go live in production:

1. **Job-relevance scope** — Layer 2 may only score when English fluency is a documented requirement of the specific role (`job_relevance_scoped = true`).
2. **Adverse-impact review** — the Layer 2 rubric must pass a bias/adverse-impact review (OD-15, open). **This blocks Layer 2 from production until complete.**

Layer 2 is a **separate axis** and must never fold into the content `overall_score`.

### 6.3 Layer 3 — SNR estimation

Background noise is estimated from the live microphone signal during recording using a Web Audio `AnalyserNode`. RMS amplitude is sampled every 250ms:

```javascript
// noise floor = 10th percentile RMS; speech level = 90th percentile RMS
snr_db = 20 * log10(speech_rms / noise_rms)
```

Flagged as noisy if SNR < 10 dB.

### 6.4 Layer 4 — Capture hygiene

Collected from browser APIs after mic permission is granted:

| Field | Source | Notes |
|-------|--------|-------|
| `connection_type` | `navigator.connection.effectiveType` | 4g / 3g / 2g / slow-2g |
| `downlink_mbps` | `navigator.connection.downlink` | Estimated throughput |
| `rtt_ms` | `navigator.connection.rtt` | Round-trip time |
| `device_label` | `MediaStreamTrack.label` | Mic device name |
| `capture_sample_rate` | `MediaStreamTrack.getSettings().sampleRate` | Native capture rate |
| `transcription_latency_ms` | Wall clock | ASR wall time — weak-device signal |

### 6.5 `capture_confidence` flag

A derived diagnostic flag summarising Layers 3–4:

```
capture_confidence = "reduced" if any of:
  - snr_db < 10
  - downlink_mbps < 1
  - connection_type in ["slow-2g", "2g"]
  - transcription_latency_ms > 30000
else "high"
```

`capture_confidence` is surfaced to the recruiter as context. It **never alters the Layer 2 proficiency verdict or the content `overall_score`**.

---

## 7. Guardrails

| ID | Rule |
|----|------|
| V1 | The content engine (Prompt A/B/C) receives only the transcript string — never audio, never paralinguistic metadata. |
| V2 | `overall_score` is computed solely from Prompt C's output. Layers 3–4 have no code path to modify it. |
| V3 | Layer 2 metrics are stored on a separate `language_assessment` object, never merged into `AssessmentReport.overall_score`. |
| V4 | `capture_confidence` annotates the session — it has no scoring weight. |
| V5 | If ASR produces an empty transcript, the turn is treated as a non-answer — not as a low-score input. |

---

## 8. Data Schema Extensions (voice)

### CandidateSession additions

```typescript
interface CandidateSession {
  // ... existing fields ...
  media_ref:          string | null;   // audio blob pointer (if retained)
  asr_metadata:       AsrMetadata | null;
  language_metrics:   LanguageMetrics | null;
  capture_diagnostics: CaptureDiagnostics | null;
}

interface AsrMetadata {
  engine:     string;   // e.g. "whisper-tiny.en@wasm"
  confidence: number;
  latency_ms: number;
}

interface LanguageMetrics {
  words_per_minute:  number | null;
  filler_rate:       number | null;   // per minute
  filler_count:      number;
  pause_count:       number | null;   // gaps > 0.5s
  mean_pause_sec:    number | null;
  max_pause_sec:     number | null;
  vocabulary_ttr:    number | null;
  proficiency:       "strong" | "adequate" | "limited" | null;
  word_timestamps:   boolean;         // whether per-word ts were available
}

interface CaptureDiagnostics {
  snr_db:               number | null;
  connection_type:      string | null;
  downlink_mbps:        number | null;
  rtt_ms:               number | null;
  device_label:         string | null;
  capture_sample_rate:  number | null;
  transcription_latency_ms: number;
  capture_confidence:   "high" | "reduced";
}
```

### AssessmentReport addition

```typescript
interface AssessmentReport {
  // ... existing content scoring fields ...
  language_assessment:  LanguageAssessment | null;  // null until OD-15 resolved
  capture_confidence:   "high" | "reduced" | null;  // null for typed sessions
}

interface LanguageAssessment {
  proficiency:    "strong" | "adequate" | "limited";
  metrics:        LanguageMetrics;
  job_relevance_scoped: boolean;
}
```

---

## 9. Open Decisions (voice)

| ID | Decision | Status | Blocks |
|----|----------|--------|--------|
| OD-14 | Voice consent gate copy, audio blob retention window, storage location and access controls | Open — Legal | Audio retention |
| OD-15 | Layer 2 adverse-impact review: rubric bias review before any production deployment | Open — HR/Legal | **Layer 2 going live** |

---

## 10. Voice Lab (`voice-lab/index.html`)

A standalone browser-based test harness that validates the full ASR path — in-browser Whisper + all four signal layers — against real hardware, before any integration decision.

**What it tests:**
- Whether per-word timestamps actually come back on the chosen backend (WASM vs WebGPU) — the B0 spike
- Transcription latency on this specific device
- All four signal layers end-to-end

**How to run:**

```bash
cd voice-lab
npx serve .
# open in Chrome, not via file://
```

Try WebGPU first (faster). First run downloads the model (~40MB, cached). Record a ~15s clip and read the ASR Diagnostic panel — it reports the truth for the hardware, not assumptions.

**Note on serving:** do not add `Cross-Origin-Embedder-Policy: require-corp`. It blocks the CDN import and Hugging Face model download. Serve plainly.

---

## 11. Pre-Screen Form

The pre-screen phase is implemented as a static HTML form (`pre-screening/csa/prescreen-form.html`). It applies hard gate logic for availability, salary, and English self-assessment before the candidate reaches the behavioral chat. The form outputs a `PrescreenResponses` JSON blob consumed by the knockout evaluator.
