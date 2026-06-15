# Influx AI Assessment

AI-powered chat-based candidate assessment system for Influx. This repo contains the scripted presentation demo, the voice capability lab, and the engineering spec.

---

## Files

| File | Purpose |
|------|---------|
| `index.html` | Scripted presentation demo — walkthrough of the full assessment flow (pre-screen → behavioral chat → dashboard). Open in browser, password `influx2025`. Arrow keys navigate stages. |
| `voice-lab/index.html` | Standalone voice capability lab — tests in-browser Whisper (transformers.js) against the four signal layers. No build step needed. |
| `planning_docs/AI_Assessment_Engineering_Spec_v1.2.md` | Primary engineering reference — data model, API call patterns, JSON schemas, prompt architecture, fast-track routing, voice-first extension (§13), open decisions. |
| `planning_docs/chat_assessment_guide_v2.md` | Candidate-facing welcome guide (shown before the assessment starts). |

---

## Running the demo

No build step — open directly or serve:

```bash
npx serve .
# or
python -m http.server 8080
```

Demo password: `influx2025`

---

## Running the voice lab

Serve the `voice-lab/` subfolder (or the root) the same way — must be served over `http://`, not opened as a `file://` URL (ES module + model download require it).

```bash
cd voice-lab
npx serve .
```

Open in Chrome. Try **WebGPU** first (faster). First run downloads the Whisper model (~40MB, cached after). Record a ~15s clip to see all four signal layers populate.

---

## Architecture overview

```
[Pre-Screen Phase]
Prompt P (Pre-Screen Interviewer) — conversational, logistics/eligibility
  ↓ transcript
Prompt PE (Pre-Screen Extractor) — single call → PrescreenResponses JSON
  ↓ deterministic knockout (Ruby)

[Behavioral Phase]
Prompt A (Interviewer) — iterative, N turns
  ↓ full transcript
Prompt B (Scorer) — single call → markdown narrative
  ↓
Prompt C (Extractor) — single call → AssessmentReport JSON → DB → fast-track routing
```

Model targets: Prompts A & B → `claude-sonnet-4-6` · Prompts C & PE → `claude-haiku-4-5`

---

## Voice signal layers (Spec §13)

| Layer | Signal | Engine | Scored? |
|-------|--------|--------|---------|
| 1. Content | What they say (21 fields, 7 traits) | Prompt A/B/C — unchanged | Yes |
| 2. Language | WPM, filler rate, pauses, vocabulary TTR | ASR word timings → arithmetic | Conditional (OD-15) |
| 3. Conditions | Background noise / SNR | Web Audio AnalyserNode | No — diagnostic |
| 4. Hygiene | Connection, device, transcription latency | Browser telemetry | No — diagnostic |

ASR: in-browser Whisper via [transformers.js](https://huggingface.co/docs/transformers.js). Audio never leaves the device.

---

## Open decisions blocking production

- **OD-14** — Voice consent / audio blob retention
- **OD-15** — Layer 2 adverse-impact review (blocks Layer 2 going live)
- **OD-5** — Prompt file storage (layer 2 rubrics must not live in the app repo)

See `planning_docs/AI_Assessment_Engineering_Spec_v1.2.md §10` for the full list.
