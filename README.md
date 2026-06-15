# Influx AI Assessment — Voice Capability Lab

Standalone test harness for the in-browser voice path of the Influx AI candidate assessment system. Tests in-browser Whisper (transformers.js) + all four signal layers against real hardware — before any production integration decision.

---

## Files

| File | Purpose |
|------|---------|
| `voice-lab/index.html` | The voice lab — single self-contained file, no build step |
| `planning_docs/AI_Assessment_Engineering_Overview.md` | Engineering reference — pipeline architecture, voice-first extension, four signal layers, ASR adapter interface, data schema, guardrails, open decisions |

---

## Running the voice lab

Must be served over `http://` — ES module imports and model downloads don't work from `file://`.

```bash
cd voice-lab
npx serve .
# or: python -m http.server 8080
```

Open in **Chrome**. Try the **WebGPU toggle** first (faster). First run downloads the Whisper model (~40MB, cached after). Record a ~15s clip — all four panels populate after transcription.

> **Do not add `Cross-Origin-Embedder-Policy: require-corp`** — it blocks the CDN import and Hugging Face model download. Serve plainly.

---

## What the lab tests

**Core question:** can in-browser Whisper deliver per-word timestamps at acceptable latency on this hardware? The ASR Diagnostic panel answers this for the actual device, not as an assumption.

| Layer | Signal | Engine | Scored? |
|-------|--------|--------|---------|
| **1. Content** | Transcript text | Whisper → text | Display only |
| **2. Language** | WPM, filler rate, pauses, vocabulary TTR | Word timings → arithmetic | Proficiency verdict |
| **3. Conditions** | Background noise (SNR) | Web Audio AnalyserNode | Diagnostic only |
| **4. Hygiene** | Connection, device, transcription latency | Browser telemetry | Diagnostic only |

Layers 3–4 are **diagnostic only** — they set a `capture_confidence` flag (high / reduced) and never alter the Layer 2 proficiency verdict. This mirrors the production fairness contract (Spec §13.6 / OD-15).

---

## ASR backends

| Backend | Speed | Word timestamps | Notes |
|---------|-------|----------------|-------|
| WASM (CPU) | Slow (~30–60s for 15s clip) | ✓ Yes — word-level | Reliable; single-threaded without COEP |
| WebGPU | Fast | Maybe — chunk-level only | Lab confirms empirically; `Xenova/*` models are q8 and may error on WebGPU |

If WebGPU errors on model load, switch to WASM. The lab catches and reports it.

---

## Model options

| Model | Size | Notes |
|-------|------|-------|
| `Xenova/whisper-tiny.en` | ~40MB | Default — fastest |
| `Xenova/whisper-base.en` | ~75MB | More accurate |

Both are English-only `.en` variants, matching the CSA English assessment context.

---

## Layer 2 proficiency thresholds

| Verdict | Criteria |
|---------|---------|
| `strong` | WPM 100–170 AND filler < 2/min AND TTR > 0.55 |
| `adequate` | WPM 80–200 AND filler < 4/min AND TTR > 0.40 |
| `limited` | anything else |

Thresholds are shown in the UI — the verdict is not a black box.

---

## Engineering reference

See [`planning_docs/AI_Assessment_Engineering_Overview.md`](planning_docs/AI_Assessment_Engineering_Overview.md) for:
- Full pipeline architecture (pre-screen → behavioral → scoring → routing)
- ASR adapter interface and pipeline code
- Complete Layer 2–4 metric definitions and formulas
- Data schema extensions (`CandidateSession`, `AssessmentReport`, `LanguageMetrics`, `CaptureDiagnostics`)
- Guardrails (V1–V5)
- Open decisions OD-14 and OD-15
