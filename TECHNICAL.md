# Technical Documentation - Auditly

Internal reference for architecture decisions, prompt design, and implementation details.
Not a changelog - for version history see `CHANGELOG.md`.
Last updated: v0.73

---

## Version Scheme
`vX.XX` - increments by 0.01 per push. Current: v0.73.

Single-file browser application (`index.html`, ~323kb). No backend, no build step, no dependencies.
Deployed via GitHub Pages at `https://krdaj.github.io/ux-audit-tool`.

All API calls made directly from browser to AI provider endpoints. Data never passes through an intermediate server.

---

## Audit Modes

| Internal | Label | Vision | Evaluators | WCAG | AX | Action Tabs |
|----------|-------|--------|------------|------|----|-------------|
| `uxui` | UX / UI / AX Audit | All models parallel | Yes | Yes | Yes (if HTML) | Yes |
| `benchmark` | Pattern Research | 1 model only, 600px/0.5 JPEG | No | No | No | No |
| `feedback` | Test Analysis | 1 model only, 600px/0.5 JPEG | No | No | No | No |

---

## Model Configuration

```js
const CLAUDE_MODEL  = 'claude-sonnet-4-20250514'
const GROQ_MODEL    = 'llama-3.3-70b-versatile'
const GPT_MODEL     = 'gpt-4o'
const GEMINI_MODEL  = 'gemini-2.5-flash'
const APERTUS_MODEL = 'swiss-ai/Apertus-8B-Instruct-2509'
const GITHUB_MODEL  = 'meta/Llama-3.3-70B-Instruct'
const BEDROCK_MODEL = 'anthropic.claude-sonnet-4-6'
const HF_BASE       = 'https://router.huggingface.co/v1/chat/completions'
const GITHUB_BASE   = 'https://models.github.ai/inference/chat/completions'
const BEDROCK_BASE  = 'https://k1wa0n6ns6.execute-api.us-east-1.amazonaws.com/invoke'
const BEDROCK_MODEL = 'arn:aws:bedrock:us-east-1:869425948913:inference-profile/global.anthropic.claude-sonnet-4-6'
```

### Model Roles

| Model | Vision | Evaluator | Best Practice | Fix | Weight |
|-------|--------|-----------|---------------|-----|--------|
| Claude Sonnet 4 | Yes | No | Yes (web search) | Yes fallback | x1.0 |
| AWS Bedrock (Sonnet 4.6) | Yes | Yes | Yes fallback | Yes fallback | x1.0 |
| Gemini 2.5 Flash | Yes | No | No | No | x0.9 |
| GPT-4o | Yes | No | No | No | x0.9 |
| GitHub Models (Llama 3.3 70B) | No | Yes | No | No | x0.75 |
| Apertus 8B (Swiss AI) | No | Yes | No | No | x0.65 |
| Groq Llama 3.3 70B | No | Yes | No | Yes primary | x0.6 |

### KEY_IDS
`['key-claude','key-groq','key-gemini','key-openai','key-apertus','key-github','key-bedrock']`

### Bedrock Auth
No auth header needed - Lambda IAM role handles AWS auth internally.
The `key-bedrock` field value is used only as an on/off switch (any non-empty value activates Bedrock).

**Architecture:**
```
Browser -> API Gateway (k1wa0n6ns6.execute-api.us-east-1.amazonaws.com/invoke)
        -> Lambda (bedrock-proxy-handler, us-east-1)
        -> Bedrock Runtime (anthropic.claude-sonnet-4-6)
```

Lambda has `AmazonBedrockFullAccess` IAM policy. CORS configured: `*` origin, `POST/OPTIONS`, `content-type`.
No web_search tool (not supported on Bedrock). All `claudeKey` reads fall back to `key-bedrock`.

---

## Pipeline Execution

### UX/UI/AX Audit
```
Upload -> Compress (1280px/0.85) -> [Vision calls parallel + retry]
  -> Deduplicate -> Option D scoring:
     applyConfidenceWeighting() -> applyWCAGCorrective() -> applyAXCorrective()
  -> [Evaluator calls parallel] -> Weighted avg -> Render
  -> WCAG check (CSS or image sampling)
  -> AX audit (DOMParser, no API)
```

### Pattern Research (benchmark)
```
Upload -> Compress (600px/0.5 JPEG) -> 1 vision model only (short prompt ~200 tokens)
  -> 8s delay -> renderBenchmarkResults() (Claude + web_search OR Bedrock)
  -> UX Patterns tab
```

### Test Analysis (feedback)
```
Upload -> Compress (600px/0.5 JPEG) -> 1 vision model only
  -> renderFeedbackResults() -> Feedback Analysis tab
```

---

## Option D Scoring

Three correctives chained after `weightedAvg()`:

```js
const rawCombined = weightedAvg(allScores, activeModelIds);
const corrected1  = applyConfidenceWeighting(rawCombined, annotations);
const corrected2  = applyWCAGCorrective(corrected1);
const combinedScores = applyAXCorrective(corrected2);
```

| Corrective | Effect |
|-----------|--------|
| `applyConfidenceWeighting` | High SEV + high confidence findings pull heuristic score down |
| `applyWCAGCorrective` | WCAG fails lower H1 (Visibility) and H8 (Aesthetic) |
| `applyAXCorrective` | AX Critical issues lower H1 and H5 (Error Prevention) |

---

## AX Audit - Rule Engine

`parseHTMLForAX(htmlStr)` runs entirely in-browser via `DOMParser`. No API call, no tokens.

| Check | WCAG | Level |
|-------|------|-------|
| Images without alt | 1.1.1 | A |
| Alt text = filename | 1.1.1 | A |
| Heading hierarchy skipped | 1.3.1 | A |
| Multiple H1 | 1.3.1 | A |
| Input without label | 1.3.1 | A |
| Button without name | 4.1.2 | A |
| Link without name | 2.4.4 | A |
| Non-descriptive link text | 2.4.6 | AA |
| Missing lang attribute | 3.1.1 | A |
| Positive tabindex | 2.4.3 | A |
| ARIA role misuse | 4.1.2 | A |
| Missing landmarks | 1.3.6 | AAA |

**Visual AX pass** (if screenshot + Claude/Bedrock/Gemini key): separate vision call checking focus indicators, touch targets, contrast, alt text quality. Results merged into issues array with category "Visual (AI)".

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
| Anthropic | Yes | `anthropic-dangerous-direct-browser-access: true` |
| AWS Bedrock | Yes | Direct fetch via API Gateway (CORS enabled) |
| Gemini | Yes | Native CORS |
| Groq | No | `fetchCORS()` fallback |
| Apertus (HF) | No | `fetchCORS()` fallback |
| GitHub Models | No | `fetchCORS()` fallback |
| OpenAI | No | No fallback (local only) |

---

## Prompt Architecture

### visionSystem() - Mode-aware

```
benchmark mode -> return early with short ~200 token prompt (pattern-focused)
feedback mode  -> inject feedback text + persona
uxui mode      -> full prompt:
  [Role + reputation reward]
  [PAGE CONTEXT: product, flow, known issues, do not audit]
  [REGIONAL CONTEXT: country standards, currency, date format, law]
  [INDUSTRY CONTEXT: oblique/ecommerce/saas etc.]
  [USER PERSONA: age, frequency, tech affinity, device]
  [Quality checklist + avoid-list + scoring rules + JSON schema]
```

### Country Context
| Country | Currency | Date | A11y Law | Privacy |
|---------|----------|------|----------|---------|
| CH | CHF | DD.MM.YYYY | BehiG | nDSG |
| DE | EUR comma | DD.MM.YYYY | BITV 2.0 | DSGVO |
| AT | EUR | DD.MM.YYYY | WZG | DSGVO |
| EU | EUR | varies | EAA | GDPR |
| UK | GBP | DD/MM/YYYY | GDS | UK GDPR |
| US | USD | MM/DD/YYYY | ADA/508 | varies |

---

## Finding Lifecycle

```
AI generates finding (_model tagged with source model ID)
  -> shown in sidebar with count badge + filter bar (dim / SEV / Fixed)
  -> user can: Edit / Delete (undo buffer, 5s toast) / Move Pin / Comment / Challenge / Mark Fixed
  -> Export filters by confidence + _removed + _fixed
  -> PDF: findings sorted by severity
```

### Annotation Properties
| Property | Type | Purpose |
|----------|------|---------|
| `_model` | string | Which vision model found this |
| `_removed` | bool | Soft-delete with undo |
| `_fixed` | bool | Mark as resolved |
| `_confirmed` | bool | Challenge confirmed it |
| `_challenged` | bool | Challenge in progress |
| `_comment` | string | User note |
| `_commentUrl` | string | Jira/Confluence link |
| `_matrixNote` | string | Effort note in Priority Matrix |
| `_effortOverride` | float | Manual effort 0-1 |
| `_qualityOk` | bool | Quality Mode approved |
| `_qualityRevised` | bool | Quality Mode revised |

---

## PDF Structure

| Page | Content | Condition |
|------|---------|-----------|
| 1 | Cover - title adapts to report type | Always |
| 1b | Executive Summary - score, metrics, top 3 findings, next steps | Always |
| 2 | Annotated screenshot | inclUXUI |
| 3 | Score summary + legends | inclUXUI |
| 4 | WCAG Contrast Analysis | inclAX |
| 5 | AX Findings | inclAX + HTML |
| 6-15 | One per Nielsen heuristic | inclUXUI |
| 16 | Best Practices & Quick Fixes | inclUXUI |
| 17 | Priority Matrix - quadrant plot + legend | inclUXUI |
| 18 | QA Test Cases | inclUXUI + test cases generated |
| 19 | Jira Tickets appendix | full report only |
| Last | Methodology & Transparency - tool, models, audit ID, disclaimer | Always |

Report types: `full` (all), `uxui` (inclUXUI only), `ax` (inclAX only).

---

## localStorage

| Key | Content |
|-----|---------|
| `uxaudit_keys` | `{ claude, groq, gemini, openai, apertus, github, bedrock }` |
| `auditly_theme` | `'dark'` or `'light'` |
| `auditly_onboarded` | `'1'` after first visit |

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+Enter` | Run Audit |
| `Cmd+E` | Export PDF |
| `Cmd+K` | Open Guide |
| `Cmd+D` | Toggle Dark Mode |
| `1-9` | Navigate to finding #N |

---

## Known Limitations

| Limitation | Workaround |
|-----------|------------|
| GPT-4o CORS on GitHub Pages | Local use only |
| Groq/Apertus/GitHub via corsproxy.io | Acceptable for evaluators |
| Apertus HF quota (402) | huggingface.co/settings/billing |
| GitHub: Classic token only | Tokens (classic), no scopes |
| Bedrock: no web_search tool | Pattern Research uses knowledge only |
| Bedrock requires API Gateway proxy | Direct browser calls blocked by AWS CORS |
| AX: only structural checks automated | Manual screenreader testing required |
| Claude Tier 1: 30k TPM | 8s delay for benchmark; upgrade to Tier 2 |
| Lambda timeout | Set to 60s for vision calls with large screenshots |
| Single HTML ~323kb | Acceptable for GitHub Pages |
| Pin accuracy ~15% | Move Pin manually |
