# Technical Documentation — UX Audit Tool

Internal reference for architecture decisions, prompt design, and implementation details.  
Not a changelog — for version history see `CHANGELOG.md`.  
Last updated: v0.95

---

## Architecture Overview

Single-file browser application (`index.html`, ~215kb). No backend, no build step, no dependencies.  
Deployed via GitHub Pages at `https://krdaj.github.io/ux-audit-tool`.

All API calls made directly from browser to AI provider endpoints. Data never passes through an intermediate server.

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

| Model | Vision | Evaluator | Best Practice | Fix | Weight |
|-------|--------|-----------|---------------|-----|--------|
| Claude Sonnet 4 | ✅ | ❌ | ✅ web search | ✅ fallback | ×1.0 |
| Gemini 2.5 Flash | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| GPT-4o | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| GitHub Models (Llama 3.3 70B) | ❌ | ✅ | ❌ | ❌ | ×0.75 |
| Apertus 8B (Swiss AI) | ❌ | ✅ | ❌ | ❌ | ×0.65 |
| Groq Llama 3.3 70B | ❌ | ✅ | ❌ | ✅ primary | ×0.6 |

---

## Pipeline Execution

### UX/UI Audit
```
Upload → Compress → [Vision calls parallel + retry] → Deduplicate → [Evaluator calls parallel] → Weighted avg → Render
```

### AX Audit (runs alongside if HTML provided)
```
HTML Input → DOMParser (instant, no API) → Rule checks → [Optional: Vision AI AX pass] → Render AX tab
```

AX runs automatically after main audit if HTML textarea is filled. No separate trigger needed.

---

## AX Audit — Rule Engine

`parseHTMLForAX(htmlStr)` runs entirely in-browser via `DOMParser`. No API call, no tokens.

| Check | WCAG | Level | Method |
|-------|------|-------|--------|
| Images without alt | 1.1.1 | A | `querySelectorAll('img')` |
| Alt text = filename | 1.1.1 | A | regex on alt value |
| Heading hierarchy skipped | 1.3.1 | A | ordered traversal |
| Multiple H1 | 1.3.1 | A | count |
| Input without label | 1.3.1 | A | cross-reference id/for |
| Button without name | 4.1.2 | A | text + aria-label check |
| Link without name | 2.4.4 | A | text + aria + img[alt] |
| Non-descriptive link text | 2.4.6 | AA | string match |
| Missing lang attribute | 3.1.1 | A | html[lang] |
| Positive tabindex | 2.4.3 | A | tabindex > 0 |
| ARIA role misuse | 4.1.2 | A | role/tag combos |
| Missing landmarks | 1.3.6 | AAA | main/nav presence |

**Visual AX pass** (if screenshot + Claude/Gemini key): separate vision call with AX-specific prompt checking focus indicators, touch targets, visual contrast, alt text quality. Results merged into same issues array with category "Visual (AI)".

---

## CORS Strategy

```js
async function fetchCORS(url, opts) {
  try { return await fetch(url, opts); }
  catch(e) { return fetch('https://corsproxy.io/?url=' + encodeURIComponent(url), opts); }
}
```

| Provider | GitHub Pages | Strategy |
|----------|-------------|----------|
| Anthropic | ✅ | `anthropic-dangerous-direct-browser-access: true` |
| Gemini | ✅ | Native CORS |
| Groq | ❌ | `fetchCORS()` fallback |
| Apertus (HF) | ❌ | `fetchCORS()` fallback |
| GitHub Models | ❌ | `fetchCORS()` fallback |
| OpenAI | ❌ | No fallback (local only) |

**GitHub Models:** Classic token (`ghp_...`) required. Fine-grained tokens rejected.

---

## Eval Function Architecture

Single generic function replaces 4 near-identical implementations:

```js
async function callOpenAIEval(key, endpoint, model, findingsSummary) {
  const r = await fetchCORS(endpoint, { method:'POST',
    headers: { 'Content-Type':'application/json', 'Authorization':'Bearer '+key },
    body: JSON.stringify({ model, messages:[...], max_tokens:200, temperature:0 })
  });
  if(!r.ok) throw new Error(EVAL_ERRORS[r.status] || 'Eval error');
  return parseJSON((await r.json()).choices[0].message.content);
}

// One-liners:
const callGroqEval   = (k,f) => callOpenAIEval(k, GROQ_BASE,   GROQ_MODEL,   f);
const callApertusEval = (k,f) => callOpenAIEval(k, HF_BASE,     APERTUS_MODEL, f);
const callGitHubEval  = (k,f) => callOpenAIEval(k, GITHUB_BASE, GITHUB_MODEL,  f);
```

`EVAL_ERRORS` map handles 401/402/403/429 with user-friendly messages.

---

## Prompt Architecture

### Context Injection Order (visionSystem)
```
[Role framing]
[PAGE CONTEXT — product, flow, known issues, do not audit]
[REGIONAL CONTEXT — country standards]
[INDUSTRY CONTEXT — oblique/ecommerce/saas etc.]
[USER PERSONA — age, frequency, tech affinity, device]
[Quality checklist + scoring rules + JSON schema]
```

### Country Context (buildCountryContext)
Per-country instructions covering: currency format, date format, accessibility law, privacy law, preferred sources, patterns to flag.

| Country | Key |
|---------|-----|
| CH | CHF, DD.MM.YYYY, BehiG, nDSG, DE/FR/IT |
| DE | EUR comma, BITV 2.0, DSGVO |
| AT | EUR, WZG, DSGVO |
| EU | EAA, GDPR, multilingual |
| UK | GBP, DD/MM/YYYY, GDS |
| US | USD, MM/DD/YYYY, ADA/508 |

### Vision Prompt Techniques
1. Reputation reward signal
2. Mental STEP 1/2/3 scratchpad
3. 4-question self-check per finding
4. Confidence anchors (95–100 / 80–94 / 70–79)
5. 5 concrete bad-finding examples
6. Persona impact explicit

---

## Finding Lifecycle

```
AI generates finding
  → shown in sidebar with Pending status (no badge)
  → user can: Edit / Delete / Move Pin / Comment / Challenge
    → Comment: freetext + URL → stored as ann._comment / ann._commentUrl
    → Challenge: argument + URL → Claude re-evaluates → verdict:
        remove    → ann._removed=true → greyed out
        downgrade → ann.priority updated
        confirmed → ann._confirmed=true
        keep      → no change
  → PDF export: filters by confidence + _removed; shows status badges
```

---

## PDF Structure

| Page | Content | Condition |
|------|---------|-----------|
| 1 | Cover — title adapts to report type | Always |
| 2 | Annotated screenshot | inclUXUI |
| 3 | Score summary + legends | inclUXUI |
| 4 | WCAG Contrast Analysis | inclAX |
| 5 | AX Findings — impact summary + issues by category | inclAX + HTML provided |
| 6–15 | One per Nielsen heuristic | inclUXUI |
| Last | Best Practices & Quick Fixes | inclUXUI + BP loaded |

Report types: `full` (all), `uxui` (inclUXUI only), `ax` (inclAX only).

### stripBP()
Global function strips `<cite index="...">` and HTML tags from Best Practice API responses before jsPDF rendering.

### pdfSet(size, weight, color)
PDF style shorthand inside exportPDF — avoids repeating 3 `doc.set*` calls.

---

## localStorage

| Key | Content |
|-----|---------|
| `uxaudit_keys` | `{ claude, groq, gemini, openai, apertus, github }` |

---

## Known Limitations

| Limitation | Workaround |
|-----------|------------|
| GPT-4o CORS on GitHub Pages | Local use only |
| Groq/Apertus/GitHub via corsproxy.io | Acceptable for evaluators |
| Apertus HF quota (402) | huggingface.co/settings/billing |
| GitHub: Classic token only | Tokens (classic), no scopes |
| AX: only structural checks automated | Manual screenreader testing required |
| Pin accuracy ±15% | Move Pin manually |
| Gemini free tier 20 req/min | Paid tier or Claude only |
| Single HTML ~215kb | Acceptable for GitHub Pages |
