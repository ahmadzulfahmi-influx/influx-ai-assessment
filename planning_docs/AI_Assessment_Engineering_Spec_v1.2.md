# AI Assessment System — Engineering Specification v1.2

> **Audience:** Engineering team only.
> **Note:** This document intentionally omits HR scoring rubrics, psychological trait definitions, and grading rationale. Those are in the HR planning document (restricted) and embedded inside the system prompts. Engineers work from this spec only.
>
> **Changelog from v1.1 → v1.2:**
> - Added §13: Voice-First Extension — four-layer signal model, ASR adapter interface (in-browser Whisper via transformers.js), Layer 2 language metrics, Layers 3–4 capture diagnostics
> - Extended §5.1 `getCandidateInput()` to be modality-aware (voice returns `{text, asr, media_ref}`)
> - Extended §5.3 transcript format with sidecar note (transcript string itself unchanged)
> - Extended §7.2 `AssessmentReport` with `language_assessment` and `capture_confidence` fields
> - Extended §8 Guardrail Reference Table with voice-specific guardrails V1–V5
> - Extended §12.5 `CandidateSession` with voice fields (`media_ref`, `asr_metadata`, `language_metrics`, `capture_diagnostics`)
> - Added OD-14 (voice consent / audio retention) and OD-15 (Layer 2 adverse-impact review)
>
> **Changelog from v1.0 → v1.1:**
> - Corrected probe caps to match Prompt A v1.1 (max 1 per trait, not 2)
> - Corrected Prompt B output format (markdown narrative, not JSON)
> - Added Prompt C (JSON Extractor) to bridge narrative → structured data
> - Added session-end detection options
> - Added open decisions section

---

## 1. System Overview

Three-call architecture. Prompt A interviews, Prompt B scores narratively, Prompt C extracts structured data.

| Step | Prompt | Role | Input | Output |
|------|--------|------|-------|--------|
| Iterative (N turns) | **Prompt A** | Interviewer | Conversation history | Next question or probe or closing |
| Single call | **Prompt B** | Scorer | Full transcript + `debug_mode` flag | Markdown narrative report |
| Single call | **Prompt C** | Extractor | Prompt B report | Structured JSON |

> **Why three calls?**
> Prompt B was designed to produce a human-readable report for HR reviewers. The engineering system needs structured data (scores, field states) to store in a database. Prompt C handles this conversion cleanly without touching the HR rubric logic in Prompt B.

Benefits of this separation:
- Candidates never see scoring output.
- Transcripts can be re-scored with updated rubrics without re-interviewing.
- The HR report (Prompt B output) can be reviewed directly by humans without parsing.
- Prompt C is a thin, stateless transformer — cheap to call and easy to test.

---

## 2. Data Model

### 2.1 Signal Field Registry

These are the 18 mandatory signal fields the system tracks. Field names are canonical — use exactly as written in all database schemas, API responses, and code.

```typescript
type FieldState = "filled" | "partially_filled" | "empty" | "negative_fill";

interface SignalField {
  field: string;
  state: FieldState;
  evidence: string | null; // Direct quote from transcript, or null
}
```

**Q1 — Commitment** (3 fields)
- `motivation_type`
- `role_specificity`
- `career_direction`

**Q1 — Self Awareness** (3 fields)
- `self_knowledge`
- `gap_acknowledgment`
- `authenticity`

**Q2 — Learning Agility** (3 fields)
- `learning_method`
- `self_direction`
- `validation`

**Q3 — Intelligence** (3 fields)
- `hypothesis_formation`
- `systematic_approach`
- `root_cause_vs_symptom`

**Q4 — Conscientiousness** (3 fields)
- `pressure_response`
- `process_orientation`
- `execution_follow_through`

**Q4 — Self Awareness** (3 fields)
- `communication_timing`
- `emotional_ownership`
- `stakeholder_empathy`

**Q5 — Agency** (3 fields)
- `resourcefulness_depth`
- `experimentation`
- `self_sufficiency`

### 2.2 Trait-to-Slot Map

```typescript
// The primary unit of assessment is the SLOT, not the trait.
// A slot assesses 1 or 2 traits within a specific behavioral context.
// self_awareness in Q1 (motivation/self-knowledge context) and
// self_awareness in Q4 (pressure/ownership context) are the same trait
// assessed under two distinct lenses — not duplication.

const TRAIT_SLOT_MAP = {
  Q1: ["commitment", "self_awareness"],         // 2 traits, 6 fields
  Q2: ["learning_agility"],                     // 1 trait, 3 fields
  Q3: ["intelligence"],                         // 1 trait, 3 fields
  Q4: ["conscientiousness", "self_awareness"],  // 2 traits, 6 fields
  Q5: ["agency"],                               // 1 trait, 3 fields
};

// Total trait scores: 7 (confirmed)
// Q1 → commitment, self_awareness_q1
// Q2 → learning_agility
// Q3 → intelligence
// Q4 → conscientiousness, self_awareness_q4
// Q5 → agency
//
// ✅ OD-3 RESOLVED: Prompt B says "8 total" — this is a typo. Correct count is 7.
// Use 7 scores in mean calculation and DB schema.
// Recommended DB keys: store slot context in the key name to avoid ambiguity.
```

### 2.3 Probe Cap Rules (Corrected — matches Prompt A v1.1)

```typescript
// ✅ OD-1 RESOLVED: max 1 probe per trait. Confirmed.
// Probe cap follows directly from slot structure:
//   multi-trait slot (2 traits) → max 2 probes per slot
//   single-trait slot (1 trait) → max 1 probe per slot

const PROBE_CAPS = {
  Q1: { max_per_trait: 1, max_total: 2 }, // 2 traits → 2 probes max
  Q2: { max_per_trait: 1, max_total: 1 }, // 1 trait  → 1 probe max
  Q3: { max_per_trait: 1, max_total: 1 }, // 1 trait  → 1 probe max
  Q4: { max_per_trait: 1, max_total: 2 }, // 2 traits → 2 probes max
  Q5: { max_per_trait: 1, max_total: 1 }, // 1 trait  → 1 probe max
};
// Max probes across the full interview session: 7 (2+1+1+2+1)
```

> **Note for HR team:** The planning doc specified max 2 probes per trait. Prompt A v1.1 was written with max 1 per trait. Before going live, confirm which version is intended — this affects assessment depth for candidates who give thin initial answers.

---

## 3. Layer 1 — Field State Logic (Engineering Rules)

These are the four states Prompt B assigns to every signal field. **The scoring implications are structural rules only — the actual quality calibration lives inside Prompt B's Layer 2 section (HR-restricted).**

```
FILLED
  Definition : Candidate gave specific, concrete evidence. Quoteable.
  Scoring    : Counts toward score. Quality level (5 vs 4) determined
               by Layer 2 inside Prompt B.

PARTIALLY_FILLED
  Definition : Candidate touched the topic vaguely or generically.
               Summarizable but not strongly quoteable.
  Scoring    : Contributes to score. Caps trait at 3 unless a probe
               upgrades it to FILLED.

EMPTY
  Definition : Not addressed at all.
  Scoring    : Pulls score down. Still EMPTY after probe = scored absent.

NEGATIVE_FILL
  Definition : Candidate addressed it but evidence contradicts the trait.
  Scoring    : Single → trait capped at 2.
               Multiple → trait capped at 1. Non-negotiable.
```

---

## 4. Probing Loop — State Machine

### 4.1 Per-Slot Flow

```
START SLOT (Qn)
│
├─ Generate question (from template + constraints)
│   └─ If ANY constraint is violated → use fallback verbatim
│
├─ Receive candidate response
│
├─ [Internal] Assess each signal field → assign Layer 1 state
│
├─ PROBE LOOP
│   │
│   ├─ Count EMPTY fields per trait
│   ├─ Select ONE field per trait to probe (priority: most critical)
│   ├─ Deliver probe (one at a time — wait for response)
│   ├─ Re-assess affected field after response
│   └─ Cap: 1 probe per trait, max [2 or 1] total (see §2.3)
│
└─ TRANSITION → next slot (neutral language, no evaluation signal)
```

### 4.2 Probe Constraints (Enforced Inside Prompt A)

```
- One probe per turn. Never batch.
- Never name the signal field or what you are looking for.
- Never rephrase the candidate's answer back at them.
- Neutral acknowledgment before probing: "Thank you for sharing that."
- After cap reached: move on. No fishing.
```

### 4.3 Special Case — `authenticity` Field (Q1 Self Awareness)

`authenticity` has **no probe**. It is assessed holistically across the full Q1 answer. Prompt A does not trigger a probe for this field even if it's empty.

---

## 5. API Loop — Prompt A (Interview Call)

### 5.1 Call Pattern

```typescript
// Pseudocode — adapt to your HTTP client

const PROMPT_A = /* load from secure config, never hardcode */;
const SESSION_END_PATTERNS = [
  "That's everything I wanted to ask",
  "Thank you so much for taking the time",
  "We'll be in touch about next steps",
];

const conversationHistory: Message[] = [];
let sessionActive = true;

// Step 1: Initial greeting (no user message yet)
const greeting = await callClaude({
  system: PROMPT_A,
  messages: [],
  maxTokens: 256,
});
conversationHistory.push({ role: "assistant", content: greeting });
renderToChat(greeting);

// Step 2: Wait for candidate "Ready" confirmation
const readyInput = await getCandidateInput();
conversationHistory.push({ role: "user", content: readyInput });

// Step 3: Main interview loop
while (sessionActive) {
  const response = await callClaude({
    system: PROMPT_A,
    messages: conversationHistory,
    maxTokens: 512,
    temperature: 0.3,
  });

  const aiTurn = response.content[0].text;
  renderToChat(aiTurn);

  // Detect session end
  if (SESSION_END_PATTERNS.some(p => aiTurn.includes(p))) {
    sessionActive = false;
    break;
  }

  // getCandidateInput() is modality-aware (see §13.2 for voice shape).
  // For text modality it returns a plain string (backward compatible).
  // For voice modality it returns { text, asr, media_ref }.
  // Only the .text field is pushed into conversationHistory — Prompt A always sees plain text.
  const candidateInputRaw = await getCandidateInput();
  const candidateInput = typeof candidateInputRaw === "string"
    ? candidateInputRaw
    : candidateInputRaw.text;
  // Side-channel data (asr, media_ref) is accumulated separately for Layer 2 + storage.
  // See §13 for the full accumulation pattern.
  conversationHistory.push({ role: "assistant", content: aiTurn });
  conversationHistory.push({ role: "user", content: candidateInput });
}

// Step 4: Build transcript for scoring
const transcript = buildTranscript(conversationHistory, candidateMeta);
```

### 5.2 Session End Detection

Prompt A v1.1 does **not** emit a structured sentinel. Use phrase matching against the closing script. Recommended approach — match on the most unique phrase:

```typescript
const SESSION_END_TRIGGER = "We'll be in touch about next steps";
```

> **⚠️ Open Decision:** Ask the HR/AI team to add `[INTERVIEW_COMPLETE]` to the end of Prompt A's closing script. Phrase matching is fragile — a one-word edit to the closing in a future prompt version would silently break the loop. See §10.

### 5.3 Transcript Format

Build the transcript as a structured string before passing to Prompt B:

```
[CANDIDATE_ID: {id}]
[SESSION_START: {ISO8601 timestamp}]
[FLAGS: {comma-separated anti-gaming flags detected by Prompt A, if any}]

Interviewer: {turn 1}
Candidate: {turn 2}
Interviewer: {turn 3}
...
```

> **Voice modality — transcript is unchanged.** The flat string above remains the sole input to Prompt B (§8 S1: "score only what's in the transcript"). Paralinguistic signal (speaking rate, filler words, background noise) must **never** appear inside this string — it would corrupt the S1 invariant. Instead, `buildTranscript()` computes a separate **sidecar** (`language_metrics`, `capture_diagnostics`) alongside the transcript, stored in `CandidateSession` and surfaced to the recruiter dashboard. See §13 for schema and accumulation pattern.

### 5.4 Token Budget

| Parameter | Prompt A | Prompt B | Prompt C |
|-----------|----------|----------|----------|
| Model | `claude-sonnet-4-6` | `claude-sonnet-4-6` | `claude-haiku-4-5-20251001` |
| `max_tokens` | 512/turn | 2048 (debug: 4096) | 1024 |
| `temperature` | 0.3 | 0.1 | 0.0 |

> Use Haiku for Prompt C — it's a deterministic extraction task on structured text. No reasoning required.

---

## 6. Scoring Call — Prompt B

### 6.1 Call Pattern

```typescript
const PROMPT_B = /* load from secure config */;

const scoringResponse = await callClaude({
  system: PROMPT_B,
  messages: [{
    role: "user",
    content: `<transcript>\n${transcript}\n</transcript>\n\ndebug_mode: ${debugMode}`,
  }],
  maxTokens: debugMode ? 4096 : 2048,
  temperature: 0.1,
});

const narrativeReport: string = scoringResponse.content[0].text;
// This is a markdown string. Do NOT attempt to JSON.parse() it.
// Pass it to Prompt C for structured extraction (§7).
// Optionally store the raw markdown for HR reviewers.
```

### 6.2 Output Format (Markdown Narrative)

Prompt B produces a human-readable markdown report structured as:

```markdown
# CANDIDATE ASSESSMENT REPORT

## Candidate: [Name]
## Date: [Date]
## Overall Score: [X.X] / 5.0
## Recommendation: [Strong Yes / Yes / Neutral / No / Strong No]

---

### Q1: Commitment
**Score: [X] / 5 — [Label]**
[Evidence rationale — 2-4 sentences]

### Q1: Self Awareness
...
### Q2: Learning Agility
...
### Q3: Intelligence
...
### Q4: Conscientiousness
...
### Q4: Self Awareness
...
### Q5: Agency
...

---

### Holistic Notes
### Integrity Notes
### Summary
```

When `debug_mode: true`, the full internal reasoning block (field state analysis, Layer 2 quality, score reasoning) is prepended before the report section.

---

## 7. Extraction Call — Prompt C (JSON Extractor)

Prompt C is a thin extraction layer. It reads the narrative report from Prompt B and outputs structured JSON. It contains **no scoring logic** — it only parses what Prompt B already decided.

### 7.1 Call Pattern

```typescript
const PROMPT_C = `
You are a data extraction assistant. You will receive a structured assessment report
in markdown format. Extract the data into the JSON schema provided. Do not infer,
interpret, or modify the data. Extract only what is explicitly stated in the report.
Output valid JSON only. No prose, no explanation.
`;

const extractionResponse = await callClaude({
  system: PROMPT_C,
  messages: [{
    role: "user",
    content: `Extract the following report into JSON:\n\n${narrativeReport}\n\nSchema:\n${JSON.stringify(ASSESSMENT_SCHEMA, null, 2)}`,
  }],
  maxTokens: 1024,
  temperature: 0.0,
});

const assessmentData: AssessmentReport = JSON.parse(
  extractionResponse.content[0].text
);
```

### 7.2 JSON Schema (Prompt C Output)

```typescript
interface AssessmentReport {
  candidate_id: string;
  assessed_at: string;           // ISO 8601
  debug_mode: boolean;
  overall_score: number;         // e.g. 3.75 — formula UNCHANGED by voice modality
  recommendation: RecommendationLabel;

  trait_scores: TraitScore[];    // 7 entries — see §2.2 for slot/trait breakdown

  holistic_notes: string | null;
  integrity_notes: string | null;
  summary: string;
  flags: AssessmentFlag[];       // populated from "Integrity Notes" section

  // Integrity confidence fields — populated by Prompt C from the "Integrity Notes" section.
  // lowered_confidence is true when Prompt B's Integrity Notes section contains the phrase
  // "Lowered confidence" (i.e. ≥2 flags fired). Used for routing short-circuit (see §12.3).
  // integrity_flag_count is the raw count of flags listed in the Integrity Notes section.
  // NEVER adjusts overall_score or recommendation label — annotation only.
  lowered_confidence: boolean;
  integrity_flag_count: number;

  // Voice modality fields — null when interview was text-mode
  language_assessment: LanguageAssessment | null;
  capture_confidence: "high" | "reduced" | null;
  // "reduced" = one or more Layer 3/4 diagnostics flagged poor conditions.
  // Annotation only — NEVER adjusts overall_score or recommendation.
}

interface TraitScore {
  trait: string;
  // Use these exact keys:
  // "commitment" | "self_awareness_q1" | "learning_agility" | "intelligence"
  // | "conscientiousness" | "self_awareness_q4" | "agency"
  slot: "Q1" | "Q2" | "Q3" | "Q4" | "Q5";
  score: 1 | 2 | 3 | 4 | 5;
  label: string;                 // e.g. "Intentional Pivot", "Sandbox/Simulation"
  rationale: string;
}

type RecommendationLabel =
  | "strong_yes" | "yes" | "neutral" | "no" | "strong_no";

interface AssessmentFlag {
  type: "anti_gaming" | "off_topic" | "no_example" | "other";
  slot: string;
  note: string;
}

// Voice modality only — computed from ASR word timings (Layer 2).
// This is a SEPARATE scoring axis; it never folds into overall_score.
// Populated only when job_relevance_scoped = true (English fluency is a stated job requirement).
// See §13.3 and guardrail V4 for the full fairness contract.
interface LanguageAssessment {
  words_per_minute: number | null;
  filler_rate_per_min: number | null;     // um / uh / like / you know
  mean_pause_ms: number | null;
  vocabulary_ratio: number | null;        // type-token ratio (unique / total words)
  grammar_score: number | null;           // 0–1; null unless grammar pass run
  job_relevance_scoped: boolean;          // must be true for any score to be surfaced to recruiter
}
```

---

## 8. Guardrail Reference Table (for QA / Test Cases)

| Code | Type | Rule | Where Enforced |
|------|------|------|---------------|
| C1 | Consistency | Same structural pattern per slot | Prompt A §QUESTION TEMPLATES |
| C2 | Consistency | No difficulty variation across candidates | Prompt A §QUESTION TEMPLATES |
| C3 | Consistency | No role-specific language | Prompt A §QUESTION TEMPLATES |
| C4 | Consistency | No trait name or hint in question | Prompt A §QUESTION TEMPLATES |
| C5 | Consistency | Fallback verbatim if constraint violated | Prompt A §QUESTION TEMPLATES |
| B1 | Behavioral | Neutral acknowledgments only | Prompt A §YOUR IDENTITY |
| B2 | Behavioral | No coaching or structuring candidate's answer | Prompt A §GUARDRAILS |
| B3 | Behavioral | Neutral slot transitions | Prompt A §TRANSITIONS |
| B4 | Behavioral | No answer recycling across slots | Prompt A §GUARDRAILS |
| B5 | Behavioral | One redirect on "no example", then absent | Prompt A §GUARDRAILS |
| P1 | Probe | Max 1 per trait (Prompt A v1.1) | Prompt A §PROBE RULES |
| P2 | Probe | Sequential — one at a time | Prompt A §PROBE RULES |
| P3 | Probe | Targets specific empty field | Prompt A §PROBE GUIDES |
| P4 | Probe | Non-leading, no rubric vocabulary | Prompt A §PROBE GUIDES |
| P5 | Probe | Priority selection when cap exceeded | Prompt A §PROBE RULES |
| G1 | Conversation | Off-topic: max 2 redirects, score what's there | Prompt A §GUARDRAILS |
| G2 | Conversation | No example: one redirect, then absent | Prompt A §GUARDRAILS |
| G3 | Conversation | Anti-gaming: flag internally only | Prompt A §GUARDRAILS |
| G4 | Conversation | Never reveal scoring/field names/traits | Prompt A §GUARDRAILS |
| G5 | Conversation | Natural interview length | Prompt A §GUARDRAILS |
| S1 | Scoring | Score only what's in the transcript — no inference | Prompt B §CRITICAL RULES |
| S2 | Scoring | Empty after probing = absent, pulls score down | Prompt B §CRITICAL RULES |
| S3 | Scoring | Negative fill always caps — non-negotiable | Prompt B §CRITICAL RULES |
| S4 | Scoring | Cross-references are narrative only, never change numbers | Prompt B §HOLISTIC CROSS-REFERENCE |
| S5 | Scoring | Same transcript = same score (determinism) | temperature: 0.1 |
| V1 | Voice | `getCandidateInput()` exposes `{text, asr, media_ref}`; only `.text` enters `conversationHistory` | §13.2 ASR adapter |
| V2 | Voice | `language_metrics` sidecar must NOT appear inside the transcript string fed to Prompt B | `buildTranscript()` |
| V3 | Voice | Layers 3–4 (`capture_diagnostics`, `capture_confidence`) are excluded from ALL scoring math | `routeCandidate()` |
| V4 | Voice | Layer 2 (`language_assessment`) scores on a separate axis only; requires `job_relevance_scoped = true` | §13.3, OD-15 |
| V5 | Voice | Typed-answer fallback must remain available when device cannot transcribe | §13.2 ASR adapter |

---

## 9. Suggested Build Order (MVP)

```
Phase 1 — Loop plumbing (no real prompts needed yet)
  [ ] Scaffold API loop with a stub system prompt
  [ ] Confirm turn-taking, history accumulation, session end detection
  [ ] Build transcript formatter (§5.3)

Phase 2 — Scoring pipeline
  [ ] Wire Prompt B call with debug_mode: true
  [ ] Confirm markdown report is generated correctly
  [ ] Wire Prompt C call
  [ ] Validate JSON schema output
  [ ] Write unit tests: parse score, recommendation, trait_scores array

Phase 3 — Integration
  [ ] Replace stub with real Prompt A (coordinate with HR/AI team for prompt file)
  [ ] Replace stub Prompt B with real file
  [ ] End-to-end test: full 5-slot run → scoring → extraction → DB insert

Phase 4 — Hardening
  [ ] Add session end sentinel to Prompt A (pending open decision)
  [ ] Add anti-gaming flag propagation from Prompt A transcript header → Prompt C → DB
  [ ] Set debug_mode: false for production
  [ ] Load test: concurrent sessions
```

---

## 10. Open Decisions (Needs Resolution Before v1.0 Launch)

| # | Issue | Status | Resolution / Options | Owner |
|---|-------|--------|---------------------|-------|
| OD-1 | **Probe cap** | ✅ Resolved | Max 1 per trait. Multi-trait slots = 2 probes, single-trait = 1. Max 7 across full session. | — |
| OD-2 | **Session end sentinel** | ✅ Resolved | `[INTERVIEW_COMPLETE]` added to Prompt A v1.1 closing. API loop detects this token and strips it before display. | — |
| OD-3 | **Trait count** | ✅ Resolved | 7 scores total. Prompt B saying "8" is a typo. Scoring unit is the slot — Self Awareness scored once per slot context (Q1 and Q4). | — |
| OD-4 | **`authenticity` field — no probe** | 🟡 Needs test | Holistically assessed in Prompt A. Confirm Prompt B scores it correctly vs. always marking empty. | AI team to test |
| OD-5 | **Prompt file storage** | 🔴 Open | Prompt A and B contain HR-sensitive Layer 2 rubrics. Must not live in repo. (Env var? Secret manager?) | Eng + Security |
| OD-6 | **candidate_id injection** | 🔴 Open | Who generates and injects candidate_id into transcript header — ATS or assessment web app? | Eng + Product |

---

## 11. What's in This Spec vs. What's in the Prompts

| Concern | Where It Lives | Who Touches It |
|---------|---------------|----------------|
| Signal field names (canonical) | This spec | Eng |
| Layer 1 state logic and caps | This spec | Eng |
| API loop and call patterns | This spec | Eng |
| JSON schema | This spec | Eng |
| Fast-track routing and calendar integration | This spec §12 | Eng + HR |
| Question templates and fallbacks | Prompt A | HR/AI team |
| Signal field detection cues | Prompt A | HR/AI team |
| Probe guides | Prompt A | HR/AI team |
| Layer 2 quality descriptors | Prompt B | HR/AI team |
| Scoring matrix | Prompt B | HR/AI team |
| Cross-reference rules | Prompt B | HR/AI team |

---

## 12. Fast-Track Routing & Calendar Integration

### 12.1 Overview

After the scoring pipeline completes (Prompt B → Prompt C → `AssessmentReport` JSON), the app evaluates the candidate's `mean_score` against two configurable thresholds. There are three routing paths:

```
mean_score >= FAST_TRACK_THRESHOLD  (pass bar — "book recruiter interview")
  → Fast-track path: show inline scheduling panel to candidate
  → Candidate selects a recruiter and time slot
  → Booking confirmed, calendar invite sent to both parties
  → Assessment report stored and flagged as fast-tracked

AUTO_REJECT_FLOOR <= mean_score < FAST_TRACK_THRESHOLD  (middle band)
  → Human-review queue (or auto-reject — ops choice, see OD-16b)
  → Assessment report stored in review queue
  → Recruiter reviews 5-min report manually and decides to advance or reject

mean_score < AUTO_REJECT_FLOOR  (1.9, strictly)
  → Auto-reject: show "we'll be in touch" message; report stored but not surfaced for manual review
  → This band is 100% recruiter-confirmed reject on the behavioral dataset (v1.3 calibration)
  → Eustace (1.9 exactly) is a confirmed recruiter-Yes — the floor is STRICTLY LESS THAN 1.9
```

> **Passing = fast-track (one gate, not two).** `FAST_TRACK_THRESHOLD` is the single meaningful gate — clearing it means the candidate may book a recruiter interview. There is no separate "pass threshold" distinct from the fast-track threshold. The earlier default of 4.5 (`strong_yes`) was a placeholder.
>
> **The bar is `3.5` — a deliberate, fixed qualifier line, not an empirical fit-to-cohort value (HR decision, Fahmi, 2026-06-13).** 3.5 is the semantic boundary between qualifiers and non-qualifiers. It is held fixed *on purpose*: a stable "pass" meaning is the precondition for raising the bar later (e.g. to 4.0) with confidence that a higher score means a genuinely better candidate. A bar bent to fit the current distribution cannot be raised later — it carries no quality signal.
>
> **Why not the empirical 2.71?** 2.71 was the value that maximizes *fast-track recall* on the v1.3 18-candidate cohort (10/16 Yes pass). It was **considered and rejected** (Addendum 6/7): optimizing for fast-track recall is not the goal, and a 2.71 "pass" tells you nothing about candidate quality — it forecloses the raise-the-bar-later strategy. Critically, candidates below 3.5 are **not rejected** — they route to human review (the middle band), where a recruiter reads the 5-minute report. Low fast-track recall ≠ low acceptance recall; the human-review catch-net is exactly what makes a high, selective bar safe.
>
> **Open calibration item (not a reason to lower the bar):** after v1.4 + Phase 8 scorer calibration, only 2/16 recruiter-Yes candidates currently reach 3.5. That gap measures how much scorer/probe calibration and data volume remain — and may partly reflect transcript-only ground-truth limits (Addendum 7 §8.6). The response is to keep calibrating the prompt and grow the sample, **not** to move the line. See OD-16.

### 12.2 Configurable Thresholds

Both thresholds **must not be hardcoded**. HR needs to adjust them without a deployment.

```typescript
// Stored in environment config or HR admin settings — never in source code

// Pass bar: candidates at or above this score may book a recruiter interview (fast-track).
// Fixed qualifier line — the semantic boundary between qualifiers and non-qualifiers (HR
// decision, 2026-06-13). Held constant so it can be RAISED later (e.g. 4.0) with confidence;
// NOT tuned to fit the current cohort. The empirical 2.71 (max fast-track recall) was rejected
// (Addendum 6/7). Below-bar candidates route to human_review, not auto-reject. See OD-16.
const FAST_TRACK_THRESHOLD: number = parseFloat(
  process.env.FAST_TRACK_THRESHOLD ?? "3.5"  // fixed qualifier line; see OD-16
);

// Auto-reject floor: candidates strictly below this score are rejected without human review.
// STRICTLY less than 1.9 — the value 1.9 itself is in false-negative territory
// (Eustace at exactly 1.9 is a confirmed recruiter-Yes in the v1.3 calibration dataset).
const AUTO_REJECT_FLOOR: number = parseFloat(
  process.env.AUTO_REJECT_FLOOR ?? "1.9"
);

// Routing logic — three paths + integrity short-circuit
type RoutingDecision = "fast_track" | "human_review" | "auto_reject";

function routeCandidate(report: AssessmentReport): RoutingDecision {
  // Integrity short-circuit (Option A — RECOMMENDED, see §12.3):
  // Candidates with ≥2 integrity flags go to human review regardless of score.
  // The score is reported honestly; routing does the protecting, not score suppression.
  // A future high-scoring prepared-genuine candidate with 2 flags goes to human review (not
  // auto-reject) — the human reviewer can clear them. This is the preferred design.
  if (report.lowered_confidence) return "human_review";

  if (report.overall_score >= FAST_TRACK_THRESHOLD) return "fast_track";
  if (report.overall_score < AUTO_REJECT_FLOOR)     return "auto_reject";
  return "human_review";  // middle band: [AUTO_REJECT_FLOOR, FAST_TRACK_THRESHOLD)
}
```

**FAST_TRACK_THRESHOLD default:** `3.5` — the fixed qualifier line (HR decision, 2026-06-13). This is a *semantic* bar (qualifier vs. non-qualifier), deliberately held constant rather than fitted to the cohort, so it can be raised later with confidence that a higher score = a better candidate. The empirical value 2.71 (which maximizes fast-track recall at 10/16 Yes) was **considered and rejected** — see Addendum 6/7 and OD-16. Candidates below 3.5 route to `human_review`, not auto-reject.

**AUTO_REJECT_FLOOR default:** `1.9`. This band (`< 1.9`) is 100% recruiter-confirmed reject on the behavioral dataset. Do not raise it without re-running the calibration dataset. Do not lower it — Eustace at 1.9 is already in potential false-negative territory.

> **⚠️ Demo discrepancy:** The demo HTML uses `>= 4.0` as the threshold (set for demo purposes so it triggers reliably in a presentation). The production bar is `3.5` (the fixed qualifier line, OD-16); the demo still uses `4.0` for presentation triggering.

### 12.3 Integrity Routing (v1.3 addition)

The behavioral calibration (v1.3, 2026-06-12) identified a structural false-positive that scoring alone cannot resolve: a candidate (Asha Nikita) whose Intelligence and Agency evidence is genuinely strong but whose Commitment and Self Awareness profile indicate a behavioral fit problem flagged by recruiter review. The rubric correctly fires `lowered_confidence` via ≥2 integrity flags (100% paste, mirroring, STAR-structure, no natural hesitation). The integrity routing short-circuit addresses this at the routing layer, keeping the score honest.

**Trigger condition:** `report.lowered_confidence === true` — which is set by Prompt C when Prompt B's Integrity Notes section contains the phrase "Lowered confidence" (≥2 flags fired). The ≥2 threshold is already embedded in the Prompt B rubric; Prompt C reads the prose and sets the boolean.

**Safety constraint:** `lowered_confidence` is NEVER set on a single paste flag — one benign paste (e.g., pasting a prepared opening sentence) fires on too many genuine candidates (Nurul Aini, Khairunnisa) and would cause false negatives. The ≥2 threshold is the gate.

**Two containment designs — both implemented here for operator choice:**

**Option A — Routing short-circuit (RECOMMENDED, already in `routeCandidate` above):**
- `lowered_confidence → human_review` unconditionally (before score check)
- Score remains honest in the report and in the database
- The recruiter sees the score + the integrity note in the 5-min review
- A future high-scoring prepared-genuine candidate with 2 flags is not auto-rejected — a human reviewer can clear them
- Matches the flag-not-deduct philosophy of the Integrity Notes section

**Option B — Score cap (saved option, NOT active):**
```typescript
// Apply ONLY if operator prefers routing-by-score-adjustment over routing-by-flag
// Trigger: lowered_confidence = true (i.e. ≥2 flags — never single-paste)
// Cap: min(overall_score, FAST_TRACK_THRESHOLD - 0.1)  // = 3.4 at the 3.5 bar
// On the current cohort (v1.4 + Phase 8), this changes nothing Option A doesn't already do:
// Asha (3.29 post-amendment, lowered_confidence=true) is already below the 3.5 bar and routed
// to human_review by Option A regardless. Nurul Aini (4.14, 1 benign paste,
// lowered_confidence=false) is untouched by both.
// Caveat: if future scorer changes lift a prepared-genuine candidate to FAST_TRACK_THRESHOLD+
// AND they have ≥2 flags (e.g. a polished-but-genuine bilingual candidate),
// Option B suppresses their score while Option A routes them to human review.
// Option A is safer for that future case.
if (report.lowered_confidence) {
  report.overall_score = Math.min(
    report.overall_score,
    FAST_TRACK_THRESHOLD - 0.1
  );
}
```

**Where to store these:**
- MVP: environment variables (`FAST_TRACK_THRESHOLD`, `AUTO_REJECT_FLOOR`)
- Production: HR admin settings panel (editable by HR Ops without Eng involvement)
- Both values must be logged at session start so scoring audits can reconstruct which thresholds were active at the time of assessment (`CandidateSession.threshold_at_time` — extend to store both)

**Middle-band ops choice (OD-16b):** The band `[1.9, FAST_TRACK_THRESHOLD)` currently routes to `human_review`. Whether this stays as a human-review queue or collapses to `auto_reject` is an ops decision HR must confirm. Recommendation: keep as human-review until the pass-bar is validated and stable — this band contains genuine borderline candidates and English-driver false-positives that will be re-assessed via voice/video (OD-15).

### 12.3 Calendar Integration — Options

The demo references Calendly as the integration target. Three implementation options, ordered by MVP cost:

| Option | How It Works | Pros | Cons |
|--------|-------------|------|------|
| **A — Calendly Embed (MVP)** | Generate a personalised Calendly link per candidate, embed or redirect after fast-track threshold is met | Zero backend, free tier available, fastest to ship | Less control over UI, recruiter assignments done in Calendly manually |
| **B — Calendly API** | Fetch recruiter availability programmatically, render custom slot picker (as shown in demo), POST booking via API | Custom UI, matches demo exactly, recruiter assignment automated | Requires Calendly Pro ($10/mo per recruiter), more backend work |
| **C — Google Calendar API** | Direct Google Calendar integration — fetch free/busy, create events | Native to Influx's stack if on Google Workspace | Most complex, requires OAuth per recruiter |

**Recommended for MVP:** Option A. Ship a Calendly link redirect first. Upgrade to Option B in Phase 2 once the rest of the pipeline is stable.

### 12.4 Candidate-Facing Scheduling Flow

This is what the candidate sees immediately after `[INTERVIEW_COMPLETE]` is detected, **only if fast-tracked**:

```
State 1 — Scheduling Panel (shown in chat input area)
  Header : "You've been fast-tracked!" (with bolt icon)
  Copy   : "Your responses met our fast-track threshold. Select a time
            to speak with a recruiter — your assessment report will be
            reviewed by them before the call."
  Content: List of available recruiters, each with 3 time slots
  Action : Candidate selects one slot → "Confirm Selected Slot" button enables
  Footer : Lock icon + "Your assessment report is for recruiters only
           and will not be shared with you."

State 2 — Confirmed (replaces panel after confirmation)
  Icon   : Calendar check (green)
  Header : "Interview Scheduled!"
  Copy   : "You'll meet with [Recruiter Name] on [Date · Time]"
  Note   : "A confirmation will be sent to you. The recruiter will review
            your assessment before the call."
```

For standard queue (not fast-tracked):
```
  Header : "Interview Complete"
  Copy   : "Thank you for your time — we'll be in touch about the next steps soon."
```

### 12.5 Data Models

```typescript
// Region grouping — HR provides availability organised by region
type Region = "APAC" | "EU_AF" | "Americas";

// Recruiter availability — provided by HR, grouped by region
interface RecruiterSlot {
  recruiter_id: string;
  name: string;
  initials: string;        // e.g. "SL"
  role: string;            // e.g. "Senior Talent Recruiter"
  region: Region;          // for grouping and default display
  timezone: string;        // IANA tz string, e.g. "Asia/Makassar", "Europe/London"
  available_slots: TimeSlot[];
}

interface TimeSlot {
  slot_id: string;         // Calendly event URI or internal UUID
  datetime_iso: string;    // ISO 8601 in UTC — always store in UTC
  display_label: string;   // rendered in candidate's local timezone at display time
                           // e.g. "Tue 15 Apr · 10:00 (your time)"
}

// Timezone display rule:
// - Store all datetimes in UTC
// - Detect candidate's timezone via Intl.DateTimeFormat().resolvedOptions().timeZone
// - Render slot labels converted to candidate's local time
// - Show recruiter's region as a secondary label so candidates know who they're booking with

// Candidate session state — persisted to DB
type SessionStatus =
  | "in_progress"          // interview not yet complete
  | "completed_unbooked"   // interview done, fast-tracked, slot not yet selected
  | "completed_booked"     // interview done, slot confirmed
  | "completed_standard";  // interview done, routed to standard queue (no booking)

interface CandidateSession {
  session_id: string;
  candidate_id: string;
  status: SessionStatus;
  overall_score: number | null;       // null until scoring completes
  recommendation: string | null;
  fast_tracked: boolean;
  threshold_at_time: number | null;   // threshold value active at time of scoring
  transcript: string | null;          // raw transcript text (flat string, voice unchanged)
  report_markdown: string | null;     // Prompt B narrative output
  assessment_json: AssessmentReport | null; // Prompt C structured output
  created_at: string;                 // ISO 8601
  completed_at: string | null;        // set when [INTERVIEW_COMPLETE] detected
  booking_confirmed_at: string | null;

  // Voice modality fields — null when interview was text-mode (see §13)
  media_ref: string | null;                  // audio blob pointer (storage path/URI); retention governed by OD-14
  asr_metadata: AsrMetadata | null;          // engine, model, confidence, latency
  language_metrics: LanguageMetrics | null;  // Layer 2 computed from ASR word timings
  capture_diagnostics: CaptureDiagnostics | null; // Layers 3–4 browser telemetry (diagnostic only)
}

// Written to DB after candidate confirms a slot
interface SchedulingConfirmation {
  candidate_id: string;
  session_id: string;
  recruiter_id: string;
  recruiter_name: string;
  recruiter_region: Region;
  slot_datetime_utc: string;   // ISO 8601 UTC
  confirmed_at: string;        // ISO 8601
  calendly_event_uri: string | null;
  threshold_at_time: number;
  overall_score: number;
}
```

### 12.6 Session State & Re-open Redirect

Candidate session state is persisted to DB as soon as the interview completes. This allows the app to handle returning candidates correctly regardless of how they re-access the link.

```
Candidate opens assessment link
│
├─ session NOT found → start new interview (in_progress)
│
└─ session found → check status
    ├─ in_progress          → resume interview (or restart, TBD — see OD-10b)
    ├─ completed_unbooked   → skip interview entirely, render scheduling panel directly
    ├─ completed_booked     → show confirmation screen ("You're all set — see you on [date]")
    └─ completed_standard   → show "we'll be in touch" screen (no re-entry to interview)
```

`completed_unbooked` is set immediately when `[INTERVIEW_COMPLETE]` is detected AND `fast_tracked = true`. It persists until either:
- The candidate selects and confirms a slot → transitions to `completed_booked`
- HR manually overrides the status (e.g. candidate books externally)

The scheduling panel shown on re-open is identical to the one shown post-interview — same recruiter list, same slot data, refreshed from the availability source so stale slots don't appear.

### 12.7 Post-Confirmation Actions

When a candidate confirms a slot, the system must:

1. **Create the calendar event** — via Calendly API (Option B/C) or treat as booked via Calendly link redirect (Option A)
2. **Send confirmation email to candidate** — recruiter name, date/time in candidate's local timezone, note that the assessment report won't be shared with them
3. **Notify the recruiter** — calendar invite + direct link to the assessment report in PA
4. **Write `SchedulingConfirmation` to DB** — for audit trail and PA sync
5. **Update `CandidateSession.status`** → `completed_booked`, set `booking_confirmed_at`
6. **Update candidate record in PA** — `fast_tracked: true`, `interview_scheduled_at` timestamp

### 12.8 Pre-Screen Pipeline

A lightweight conversational phase that runs **before** the behavioral interview (Prompt A). It collects logistics/eligibility answers, runs deterministic knockout evaluation, and surfaces results to recruiters in People App — without blocking any candidate.

#### 12.8.1 Pipeline flow

```
Candidate opens assessment link
  ↓
Prompt P (Pre-Screen Interviewer) — asks only items enabled in the template
  ↓ pre-screen transcript
Prompt PE (Pre-Screen Extractor) — single call → PrescreenResponses JSON
  ↓
Deterministic knockout evaluation (plain Ruby, no LLM)
  ↓
  ├─ all gates pass        → proceed to Prompt A (behavioral interview, unchanged)
  └─ one+ knockout flagged → set prescreen_flagged_reasons, STILL proceed to Prompt A
                             (no hard block — recruiter decides)
```

- **Prompt P** system prompt hardcoded in `ChatAssessment::Prescreener::SYSTEM_PROMPT_BASE`. Template-editable checklist in `chat_assessment_template.prescreen_prompt`. Appends `[PRESCREEN_COMPLETE]` sentinel to trigger the `:prescreening → :interviewing` state transition.
- **Prompt PE** is fully hardcoded (`ChatAssessment::PrescreenExtractor::SYSTEM_PROMPT`). Target model: `claude-haiku-4-5-20251001`. Receives transcript + `device_adequacy_criteria` + `ITEMS ASKED` list. Outputs `PrescreenResponses` JSON. Makes no decisions.
- **Knockout evaluation** compares `PrescreenResponses` fields against the template config knockout rules in plain Ruby. An item flags only when `enabled && is_knockout && extracted-value-fails-rule`.

#### 12.8.2 Pre-screen items

| Item key | What is captured | Knockout-eligible? |
|---|---|---|
| `device_specs` | Device description, internet speed, adequacy classification | Yes — if `adequacy == "inadequate"` |
| `competing_processes` | Whether candidate has active applications elsewhere | Rarely (informational by default) |
| `shift_preference` | Ranked array of all 5 shifts (most → least preferred) | Yes — if none of candidate's top-N ranked choices (default: top 2) match `required_shifts` |
| `voice_support` | Comfortable with voice support (client-facing roles only) | Yes |
| `ic_status` | Accepts independent-contractor terms | Usually yes |
| `salary` | Acknowledges entry-level salary range | Yes — if candidate explicitly declines |

#### 12.8.3 CandidateSession additions

Add these fields alongside the existing behavioral fields (do not modify existing fields):

```typescript
prescreen_status: PrescreenStatus;           // see enum below
prescreen_transcript: string | null;         // raw Prompt P transcript
prescreen_responses: PrescreenResponses | null; // Prompt PE output (JSONB)
prescreen_flagged_reasons: string[];          // e.g. ["ic_status_declined"]
prescreen_completed_at: string | null;        // ISO 8601, set on [PRESCREEN_COMPLETE]

type PrescreenStatus =
  | "not_started"
  | "in_progress"
  | "completed_clear"     // all enabled gates pass
  | "completed_flagged";  // one+ knockout flagged; candidate still continued
```

Migration default: existing rows get `prescreen_status = "not_started"`.

Note: `SessionStatus` (behavioral/scheduling lifecycle) is **not modified**. Pre-screen state lives entirely in the new `prescreen_status` field.

#### 12.8.4 PrescreenResponses schema (Prompt PE output contract)

Field names are canonical — they are the contract with the knockout evaluator, exactly as signal field names are the contract with Prompt C.

```typescript
interface PrescreenResponses {
  device_specs:        { value: string | null; internet_speed: string | null;
                         adequacy: "adequate" | "inadequate" | "unclear" | null;
                         evidence: string | null };
  competing_processes: { value: string | null; evidence: string | null };
  shift_preference:    { ranking: Array<"regular"|"morning"|"afternoon"|"graveyard"|"weekend"> | null;
                                    // ordered most→least preferred; null if not asked or ambiguous
                         evidence: string | null };
  voice_support:       { comfortable: boolean | null; evidence: string | null };
  ic_status:           { accepts: boolean | null; evidence: string | null };
  salary:              { acknowledged: boolean | null; evidence: string | null };
}
```

Fields for items not asked are `null` (all sub-fields null). The `ITEMS ASKED` list passed to Prompt PE tells it which fields were covered.

#### 12.8.5 Template config additions

New fields on `chat_assessment_template`:

| Field | Type | Purpose |
|---|---|---|
| `prescreen_enabled` | boolean | Master toggle |
| `device_specs.enabled / is_knockout` | boolean | Ask + flag device |
| `device_adequacy_criteria` | string | Criteria for Prompt PE classification (e.g. "Min 8GB RAM, 10 Mbps") |
| `competing_processes.enabled / is_knockout` | boolean | Usually enabled, rarely a knockout |
| `shift_preference.enabled / is_knockout / required_shifts` | boolean + string[] | Knockout only if required shift unavailable |
| `voice_support.enabled / is_knockout` | boolean | Client-facing roles only |
| `ic_status.enabled / is_knockout` | boolean | Usually enabled + is_knockout |
| `salary.enabled / is_knockout` | boolean | Flag only if candidate explicitly declines |
| `salary_range` | string | Injected into `{{salary_range}}` in Prompt P at runtime |

#### 12.8.6 People App integration

- **`chat_assessments` results page:** add Pre-Screen summary block above behavioral results. Shows each item's captured value + clear/flagged badge + `prescreen_flagged_reasons` list.
- **`chat_assessment_templates` page:** add Pre-Screen config section (master toggle, per-item enabled/knockout, salary-range field, device criteria).
- **Candidate traits:** add `chat-assessment-prescreen-flagged` trait (alongside existing `chat-assessment-passed/booked/failed`), set when `prescreen_status == "completed_flagged"`.

#### 12.8.7 Prompt files

| File | Owner | Location |
|---|---|---|
| `chat-assessment-prescreen-system-prompt-v1.2.md` | Eng | `planning_docs/prompt-engineering/` |
| `chat-assessment-prescreen-template-prompt-v1.2.md` | HR | `planning_docs/prompt-engineering/` |
| `chat-assessment-prescreen-extractor-prompt-v1.2.md` | Eng | `planning_docs/prompt-engineering/` |

Pre-screen prompts contain no Layer 2 behavioral rubric and are safe to commit. Verify no salary figures appear before committing the template prompt (salary must stay as `{{salary_range}}`).

---

### 12.9 Open Decisions

| # | Issue | Status | Resolution / Options | Owner |
|---|-------|--------|---------------------|-------|
| OD-7 | **Threshold storage location** | 🔴 Open | (a) Env var — simple, requires redeploy to change. (b) HR admin panel — HR self-service, no Eng needed. Recommend (b) for production. | Eng + HR Ops |
| OD-8 | **Calendly option (A vs B)** | 🔴 Open | Option A for MVP, Option B for production. Confirm which tier of Calendly Influx already has. | Eng + HR Ops |
| OD-9 | **Recruiter availability source** | ✅ Resolved | HR provides availability. Organised by region: APAC, EU_AF, Americas. HR Ops maintains the list. | — |
| OD-10 | **Candidate no-show on scheduling panel** | ✅ Resolved | Session stored as `completed_unbooked`. Re-opening the assessment link redirects directly to the scheduling panel — interview is not re-run. | — |
| OD-10b | **Resume vs. restart for `in_progress` sessions** | 🔴 Open | If a candidate drops mid-interview, should re-opening resume from where they left off, or restart from Q1? Affects transcript continuity and scoring validity. | Product + HR |
| OD-11 | **PA integration scope** | 🟡 Partial | Behavioral data: score, recommendation, scheduling status, full report, `fast_tracked` flag → PA on completion. Pre-screen data (per §12.8.6): `prescreen_status`, `prescreen_flagged_reasons`, and the `chat-assessment-prescreen-flagged` trait → PA after pre-screen completes. Exact trigger events and field mapping TBD. | Eng + HR Ops |
| OD-12 | **Threshold audit log** | ✅ Resolved | `CandidateSession.threshold_at_time` logs the active threshold value at time of scoring. Ensures fairness audits can reconstruct decisions historically. | — |
| OD-13 | **Pre-screen resume vs. restart** | 🔴 Open | If a candidate drops mid-pre-screen, should re-opening resume or restart from Item 1? Simpler to restart (pre-screen is short); resuming requires transcript stitching. | Product + HR |
| OD-14 | **Voice consent / audio blob retention** | 🔴 Open | Recording candidate voice is sensitive PII. Spec must define: (a) explicit pre-capture consent gate (candidate must accept before mic is opened), (b) retention window for `media_ref` audio blob (delete after scoring? after N days?), (c) storage location and access controls. Affects §13 implementation and any data-residency obligations. | Eng + Legal/HR |
| OD-15 | **Layer 2 adverse-impact review** | 🔴 Open | Grammar / filler-rate / speaking-rate scoring is a proxy for accent and non-native-English status — acute for multilingual Indonesian candidates. Layer 2 must NOT go live until: (a) English fluency is documented as a stated job requirement for the specific role (`job_relevance_scoped = true`), and (b) the Layer 2 rubric undergoes a bias / adverse-impact review by HR. Blocking for any production deployment of Layer 2. | HR + Legal |
| OD-16 | **Pass-bar (`FAST_TRACK_THRESHOLD`) value** | 🟢 Decided (calibration ongoing) | **Bar = `3.5`** — the fixed qualifier line (HR decision, Fahmi, 2026-06-13). This is a *semantic* boundary (qualifier vs. non-qualifier), deliberately held constant rather than fitted to the cohort, so it can be RAISED later (e.g. to 4.0) with confidence that higher score = better candidate. The empirical alternative `2.71` (max fast-track recall, 10/16 Yes on the v1.3 cohort) was **considered and rejected** (Addendum 6/7): a cohort-fitted bar carries no quality signal and forecloses raising it later. The old `4.5` placeholder matched `strong_yes` and was never intended as the bar. **Calibration is the ongoing work, not the bar:** only 2/16 recruiter-Yes currently reach 3.5 — the job is to keep calibrating the scorer/probes and grow the sample so genuine qualifiers reach the line (and to account for transcript-only ground-truth limits, Addendum 7 §8.6), NOT to move the line. Decision file: `memory-banks/ai-assessment-demo/context/decisions/2026-06-12 - AI-assessment-demo behavioral assessment is a recruiter-agreement selection gate.md` | HR + Eng |
| OD-16b | **Middle-band routing: human-review vs auto-reject** | 🔴 Open | Candidates scoring `[AUTO_REJECT_FLOOR, FAST_TRACK_THRESHOLD)` currently route to `human_review`. HR must confirm whether this band should remain a manual-review queue or collapse to auto-reject. Recommendation: keep as human-review until OD-16 is resolved and the pass-bar is stable — this band currently contains genuine borderline candidates and English-driver false-positives (OD-15). | HR Ops |

---

---

## 13. Voice-First Extension

### 13.1 Overview and Four-Layer Signal Model

The voice modality adds candidate audio capture. The existing content-scoring engine (Prompt A/B/C) is **untouched** — it only ever sees a plain-text transcript. Voice adds three new layers computed outside the LLM pipeline.

| Layer | Signal | Engine | Cost | Scored? |
|---|---|---|---|---|
| **1. Content** | What the candidate says — existing 18 fields, 7 traits | Prompt A/B/C (unchanged) | existing | Yes (unchanged) |
| **2. Language** | Grammar, filler words, speaking rate, pauses, vocabulary | ASR word timings → arithmetic | ~$0 | Conditional — separate axis; only when `job_relevance_scoped = true` (OD-15) |
| **3. Conditions** | Background noise / SNR | Web Audio API analyser at capture | $0, client-side | No — diagnostic only |
| **4. Hygiene** | Connection, internet speed, A/V quality, device | Browser telemetry (`navigator.connection`, WebRTC `getStats`) | $0, no AI | No — diagnostic only |

**Why ASR is the linchpin:** one transcription pass feeds both Layer 1 (the text string) and Layer 2 (the word-timing array). Layer 2 metrics are pure arithmetic over the ASR word list — no extra LLM calls.

**Claude cannot ingest audio or video** (confirmed: Claude API supports text, images, and PDFs only). An ASR step converting voice → text is mandatory before content reaches any Claude model.

### 13.2 ASR Adapter Interface

ASR runs **in-browser via transformers.js** (Whisper-class model). Audio never leaves the device — the strongest consent/PII fit for HR voice data and $0 transcription cost.

> **Residual risk:** weak/low-end devices may transcribe slowly or fail. Mitigate with: (a) capability pre-check before opening mic, (b) graceful typed-answer fallback (guardrail V5), (c) transcription latency recorded in `asr_metadata` and surfaced in Layer 4 diagnostics.

All ASR code must sit behind an adapter interface so a server-side engine can be swapped in later without touching the pipeline:

```typescript
// Adapter interface — both in-browser and server-side engines must implement this
interface AsrAdapter {
  transcribe(audioBlob: Blob): Promise<AsrResult>;
}

interface AsrResult {
  text: string;                           // plain transcript string
  words: Array<{ w: string; start: number; end: number }>; // word timings in seconds
  confidence: number | null;              // mean word-level confidence 0–1
  latency_ms: number;                     // wall-clock transcription time
}

// getCandidateInput() return type when voice modality is active
interface VoiceCandidateInput {
  text: string;                           // the string pushed into conversationHistory
  asr: AsrResult;
  media_ref: string | null;              // audio blob storage pointer (if retained per OD-14)
}
```

`AsrMetadata` persisted to `CandidateSession`:

```typescript
interface AsrMetadata {
  engine: "transformers.js-whisper" | "server-whisper" | "other";
  model_id: string;              // e.g. "Xenova/whisper-small"
  confidence_mean: number | null;
  transcription_latency_ms: number | null;
  device_fallback_used: boolean; // true if typed answer was used (weak-device path)
}
```

### 13.3 Layer 2 — Language Metrics

Computed from the ASR `words[]` array after all turns complete. No LLM call.

```typescript
interface LanguageMetrics {
  words_per_minute: number | null;
  filler_rate_per_min: number | null;   // count of: um, uh, like, you know, basically, literally
  pause_profile: {
    mean_pause_ms: number | null;
    max_pause_ms: number | null;        // inter-word gaps > 500 ms count as pauses
  } | null;
  vocabulary_ratio: number | null;      // unique word types / total word tokens (TTR)
  grammar_score: number | null;         // 0–1; null unless optional grammar pass is run
  job_relevance_scoped: boolean;        // MUST be true for any Layer 2 data to surface to recruiter
}
```

**Fairness contract (see also guardrail V4 and OD-15):**
- Layer 2 data is collected regardless of `job_relevance_scoped`. The flag gates *surfacing* it to the recruiter and any downstream scoring.
- When `job_relevance_scoped = false`: store the raw metrics in `CandidateSession.language_metrics` but set `AssessmentReport.language_assessment = null`. Do not render any language score in the recruiter dashboard.
- `job_relevance_scoped` is set by the role template (HR-managed field), not derived at runtime.
- Layer 2 scores on a **completely separate axis** — they are never included in the `overall_score` mean, never adjust `recommendation`, never trigger fast-track routing.
- OD-15 must be resolved (adverse-impact review complete) before Layer 2 is live in any production role.

### 13.4 Layer 3 — Conditions Diagnostics

Background noise estimate computed by the Web Audio API `AnalyserNode` during recording:

```typescript
// Collect at capture time, before ASR
function estimateSnr(stream: MediaStream): number | null {
  // Use AnalyserNode to compute RMS signal level; compare silent-period RMS to active-speech RMS.
  // Returns estimated SNR in dB, or null if browser does not support Web Audio API.
  // Threshold for "reduced" flag: < 10 dB SNR (configurable).
}
```

### 13.5 Layer 4 — Hygiene Diagnostics

All browser telemetry, no network calls:

```typescript
interface CaptureDiagnostics {
  snr_estimate_db: number | null;        // from Layer 3
  connection_type: string | null;        // navigator.connection.effectiveType ("4g", "3g", etc.)
  downlink_mbps: number | null;          // navigator.connection.downlink
  audio_bitrate_kbps: number | null;     // MediaRecorder effective bitrate (if queryable)
  device_label: string | null;           // MediaDeviceInfo.label for selected mic
  transcription_latency_ms: number | null; // from AsrMetadata — slow transcription = weak device
  confidence_flag: "high" | "reduced";
}
```

`confidence_flag` derivation rule (implement as a pure function, no LLM):

```
"reduced" if ANY of:
  - snr_estimate_db < 10
  - connection_type in ["slow-2g", "2g"]
  - downlink_mbps < 1.0
  - transcription_latency_ms > 30_000  (> 30 s transcription = weak device)
Otherwise: "high"
```

`confidence_flag` maps directly to `AssessmentReport.capture_confidence`. It is informational context for the recruiter — **it does not change `overall_score`, `recommendation`, or fast-track routing** (guardrail V3).

### 13.6 Consent Gate (see also OD-14)

Before `getUserMedia()` is called, the UI must show an explicit consent screen:

```
Required copy (exact wording TBD per OD-14 / Legal review):
  "This interview uses voice recording. Your audio is processed on your device
   and [retained / deleted after scoring — TBD]. By continuing, you consent to
   this recording for the purpose of this assessment."
  
  [Continue with voice] / [Use text instead]
```

"Use text instead" routes to the existing typed-answer flow. It must always be offered — rejecting voice consent must not block the interview (guardrail V5).

---

*This document is engineering-only. For HR scoring rubrics and trait definitions, see AI Assessment Planning Document v1.4 and the system prompts (restricted).*

*Spec v1.1 — updated after review of Prompt A v1.1 and Prompt B v1.1, and demo page (index.html).*

*Spec v1.1 amended (2026-05-26) — added §12.8 Pre-Screen Pipeline.*

*Spec v1.2 amended (2026-06-11) — added §13 Voice-First Extension (four-layer model, transformers.js ASR adapter, language metrics, capture diagnostics, fairness guardrails V1–V5, OD-14/OD-15); extended §5.1 input seam, §5.3 sidecar note, §7.2 AssessmentReport, §8 guardrails, §12.5 CandidateSession.*

*Spec v1.2 amended (2026-06-12) — §12.1 updated from two-path to three-path routing model (fast-track / human-review / auto-reject). §12.2 renamed to "Configurable Thresholds" (plural); added `AUTO_REJECT_FLOOR = 1.9` (strictly below 1.9 = auto-reject, 100% recruiter-confirmed on v1.3 calibration dataset — Eustace at 1.9 is recruiter-Yes so floor is strict); updated `routeCandidate` to return three outcomes; noted `FAST_TRACK_THRESHOLD` default `4.5` is a placeholder pending v1.3 re-score validation; documented middle-band ops choice. Added OD-16 (pass-bar value — TBD from v1.3 re-score) and OD-16b (middle-band routing decision). Context: passing = fast-track = book recruiter interview (one gate); goal is agreement with recruiter-advance set on behavioral-decided candidates, not a fixed quota. Decision record in `memory-banks/ai-assessment-demo/context/decisions/2026-06-12 - AI-assessment-demo behavioral assessment is a recruiter-agreement selection gate.md`.*

*Spec v1.2 amended (2026-06-12, Phase 6.3) — added integrity routing to §7.2 `AssessmentReport` (`lowered_confidence: boolean`, `integrity_flag_count: number`); added §12.3 "Integrity Routing" documenting the two containment designs (Option A: routing short-circuit — RECOMMENDED, active in `routeCandidate`; Option B: score cap — saved option, not active); updated `routeCandidate` to short-circuit to `human_review` when `lowered_confidence`. Motivation: Phase 6.2 blind re-score confirmed the scoring layer alone cannot separate behavioral-No candidates (Asha Nikita, 3.29) from behavioral-Yes candidates (Pogi 3.29, Khairunnisa 3.00) — integrity routing is the primary containment. Results in `…/feedbacks/transcripts/v1.3-test-results.md` Addendum 4.*
