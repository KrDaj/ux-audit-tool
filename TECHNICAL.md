# Technical Documentation — UX Audit Tool

Internal reference for architecture decisions, prompt design, and implementation details.  
Not a changelog — for version history see `CHANGELOG.md`.  
Last updated: v0.85

---

## Architecture Overview

Single-file browser application (`index.html`, ~175kb). No backend, no build step, no dependencies.  
Deployed via GitHub Pages at `https://krdaj.github.io/ux-audit-tool`.

All API calls are made directly from the browser to AI provider endpoints. Data never passes through an intermediate server.

---

## Model Configuration

```js
const CLAUDE_MODEL  = 'claude-sonnet-4-20250514'
const GROQ_MODEL    = 'llama-3.3-70b-versatile'
const GPT_MODEL     = 'gpt-4o'
const GEMINI_MODEL  = 'gemini-2.5-flash'
const APERTUS_MODEL = 'swiss-ai/Apertus-8B-Instruct-2509'
const GITHUB_MODEL  = 'meta/Llama-3.3-70B-Instruct'
const HF_BASE       = 'https://router.huggingface.co/v1/chat/completions'
const GITHUB_BASE   = 'https://models.github.ai/inference/chat/completions'
```

### Model Roles

| Model | Vision | Evaluator | Best Practice | Fix Generator | Weight |
|-------|--------|-----------|---------------|---------------|--------|
| Claude Sonnet 4 | ✅ | ❌ | ✅ web search | ✅ fallback | ×1.0 |
| Gemini 2.5 Flash | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| GPT-4o | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| GitHub Models (Llama 3.3 70B) | ❌ | ✅ | ❌ | ❌ | ×0.75 |
| Apertus 8B (Swiss AI) | ❌ | ✅ | ❌ | ❌ | ×0.65 |
| Groq Llama 3.3 70B | ❌ | ✅ | ❌ | ✅ primary | ×0.6 |

---

## Pipeline Execution

```
Upload → Compress → [Vision calls parallel + retry] → Deduplicate → [Evaluator calls parallel] → Weighted avg → Render
```

1. Image compressed to max 1280px, JPEG 85%
2. Vision models run in parallel; each retries once after 3s on failure
3. Claude used as last resort if all vision models fail
4. Findings deduplicated (>60% word overlap dropped)
5. Evaluators run in parallel on text summary of findings
6. Scores coerced via `parseFloat`, padded to 10, clamped 0–10
7. Failed models shown as warnings in Nielsen Scores tab

---

## Token Optimisation

### Prompt Caching (Claude)
`anthropic-beta: prompt-caching-2024-07-31` on system prompts in `callClaudeVision()`, `callClaudeEval()`, `loadBestPractice()`.

### max_tokens

| Call | Model | max_tokens |
|------|-------|-----------|
| Vision | Claude | 2000 |
| Vision | GPT-4o | 2000 |
| Vision | Gemini | 8192 |
| Evaluator | all | 200 |
| Evaluator | Claude | 400 |
| Best Practice | Claude | 800 |
| Fix Generator | Groq/Apertus/Claude | 900 |

### _reasoning field removed (v0.85)
Was causing silent JSON truncation at max_tokens:1500 — Claude dropped from scores. Removed from schema; CoT is now a mental instruction only.

---

## CORS Strategy

| Provider | GitHub Pages | Solution |
|----------|-------------|----------|
| Anthropic | ✅ | `anthropic-dangerous-direct-browser-access: true` |
| Gemini | ✅ | Native CORS |
| Groq | ❌ | corsproxy.io fallback |
| Apertus (HF) | ❌ | corsproxy.io fallback |
| GitHub Models | ❌ | corsproxy.io fallback |
| OpenAI | ❌ | No fallback (local use only) |

**GitHub Models:** Classic token (`ghp_...`) required. Fine-grained tokens rejected even with Models: Read permission.

---

## Prompt Architecture

### Context Injection Order

```
[Role framing]
[PAGE CONTEXT — product, flow, known issues, do not audit]
[REGIONAL CONTEXT — country standards]
[INDUSTRY CONTEXT — oblique/ecommerce/saas etc.]
[USER PERSONA — age, frequency, tech affinity, device]
[Quality checklist + scoring rules + JSON schema]
```

### Country / Region Context (v0.85)

`buildCountryContext()` per country:

| Country | Key instructions |
|---------|-----------------|
| CH | CHF, DD.MM.YYYY, BehiG, nDSG, DE/FR/IT, flags US patterns |
| DE | EUR comma, BITV 2.0, DSGVO |
| AT | EUR, WZG, DSGVO |
| EU | EAA, GDPR, multilingual |
| UK | GBP, DD/MM/YYYY, GDS patterns |
| US | USD, MM/DD/YYYY, ADA/508 |
| Other | ISO standards only |

Country context also injected into Best Practice search prompt.

### Vision Prompt Techniques

1. Reputation reward signal
2. Mental scratchpad (STEP 1/2/3)
3. 4-question self-check per finding
4. Confidence calibration with anchored examples
5. Negative examples (5 bad findings to never produce)
6. Persona impact explicit

### Evaluator Prompt
"Be sceptical: challenge severity." Unmentioned heuristics scored 7.

---

## Coordinate System

Zone-based 3×3 grid. Model names element + zone, sets x/y within zone ranges.

**Collision detection (v0.85):** pins closer than `2.4 × radius` are spread apart before rendering. Canvas bounds respected.

---

## PDF Structure

| Page | Content |
|------|---------|
| 1 | Cover |
| 2 | Annotated screenshot |
| 3 | Score summary + legends |
| 4 | WCAG Contrast Analysis |
| 5–14 | One per heuristic (Issues / Recommendations) |
| Last | Best Practices & Quick Fixes (own page, only if loaded) |

### stripBP()
Global function strips `<cite index="...">` tags and HTML from Best Practice API responses before jsPDF rendering.

```js
function stripBP(s) {
  return String(s)
    .replace(/<cite[^>]*>([\s\S]*?)<\/cite>/gi, '$1')
    .replace(/<[^>]+>/g, '')
    .replace(/&amp;/g, '&')...trim();
}
```

---

## localStorage Keys

| Key | Content |
|-----|---------|
| `uxaudit_keys` | `{ claude, groq, gemini, openai, apertus, github }` |

---

## Known Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| GPT-4o CORS on GitHub Pages | Unusable in production | Local use only |
| Groq/Apertus/GitHub via corsproxy.io | Third-party proxy | Acceptable for evaluators |
| Apertus HF free quota (402) | Monthly limit exhausted | huggingface.co/settings/billing |
| GitHub Classic token only | Fine-grained rejected | Tokens (classic), no scopes |
| Pin accuracy ±15% dense UIs | Pins may miss element | Move Pin manually |
| Gemini free tier 20 req/min | Rate limiting | Paid tier or Claude only |
| Single HTML ~175kb | Growing size | Acceptable for GitHub Pages |
