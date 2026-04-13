# Technical Documentation — UX Audit Tool

Internal reference for architecture decisions, prompt design, and implementation details.  
Not a changelog — for version history see `CHANGELOG.md`.  
Last updated: v0.70

---

## Architecture Overview

Single-file browser application (`index.html`, ~146kb). No backend, no build step, no dependencies.  
Deployed via GitHub Pages at `https://krdaj.github.io/ux-audit-tool`.

All API calls are made directly from the browser to AI provider endpoints. Data never passes through an intermediate server.

---

## Model Configuration

```js
const CLAUDE_MODEL  = 'claude-sonnet-4-20250514'   // Vision + scoring + best practice + fix
const GROQ_MODEL    = 'llama-3.3-70b-versatile'    // Evaluator + fix (text only, no vision)
const GPT_MODEL     = 'gpt-4o'                     // Vision + scoring (CORS issues on GitHub Pages)
const GEMINI_MODEL  = 'gemini-2.5-flash'            // Vision + scoring (free tier available)
```

### Model Roles

| Model | Vision | Evaluator | Best Practice | Fix Generator | Score Weight |
|-------|--------|-----------|---------------|---------------|--------------|
| Claude Sonnet 4 | ✅ | ✅ | ✅ (web search) | ✅ fallback | ×1.0 |
| Gemini 2.5 Flash | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| GPT-4o | ✅ | ❌ | ❌ | ❌ | ×0.9 |
| Groq Llama 3.3 70B | ❌ | ✅ | ❌ | ✅ primary | ×0.6 |

**Score weights** reflect role and quality: vision models score based on what they see; text evaluators (Groq) score from a text summary only — their scores are down-weighted accordingly.

**Vision models** analyze the screenshot and return annotations + scores.  
**Evaluator models** receive the text summary of findings and return calibrated scores only — no image needed.

---

## Pipeline Execution

```
Upload → Compress → [Vision calls in parallel] → Deduplicate → [Evaluator calls in parallel] → Average scores → Render
```

### Step by step

1. **Image compression** — resized to max 1280px, JPEG at 85% quality. Reduces token cost significantly (vision tokens are charged per image tile).
2. **Parallel vision calls** — `Promise.all(visionIds.map(...))` — Claude, Gemini, GPT-4o run simultaneously.
3. **Finding deduplication** — findings with >60% word overlap in title are dropped. Prevents duplicate pins when multiple models find the same issue.
4. **Parallel evaluator calls** — `Promise.all(evalCalls)` — Groq + Claude evaluator run simultaneously on the text summary.
5. **Score averaging** — all model scores are collected into `allScores[]`, filtered for `NaN`, averaged per heuristic dimension.

---

## Token Optimisation

### Prompt Caching (Claude only)
Claude supports `anthropic-beta: prompt-caching-2024-07-31`. The system prompt is marked with `cache_control: { type: "ephemeral" }`. On repeated calls within 5 minutes the cached tokens cost ~10% of normal input price.

Applied to:
- `callClaudeVision()` — system prompt cached
- `callClaudeEval()` — static evaluator instructions cached
- `loadBestPractice()` — search system prompt cached

### temperature: 0 everywhere
All 9 API calls use `temperature: 0`. This eliminates randomness, improves consistency ("same screenshot → same findings"), and reduces hallucination risk.

### max_tokens per call

| Call | Model | max_tokens | Reason |
|------|-------|-----------|--------|
| Vision | Claude | 1500 | JSON with 5–7 annotations + scores |
| Vision | GPT-4o | 1500 | Same |
| Vision | Gemini | 8192 | Thinking blocks consume extra tokens |
| Evaluator | Claude | 400 | Scores + notes only |
| Evaluator | Groq | 200 | Scores only, no notes |
| Best Practice | Claude | 1500 | Search + structured JSON |
| Fix Generator | Groq | 900 | Before/after code + explanation |
| Fix Generator | Claude | 900 | Fallback if no Groq key |
| WCAG | Claude | 800 | Colour analysis JSON |

### CSS input truncation
CSS pasted from Figma Dev Mode is truncated to 1500 characters before being sent to the model. Only the first 1500 chars typically contain colour tokens and typography — the rest is layout.

### visionSystem() built once per audit
`const _sys = visionSystem()` is called once and passed to all vision model calls. Avoids redundant JS string building.

---

## CORS Strategy

| Provider | GitHub Pages | Solution |
|----------|-------------|----------|
| Anthropic | ✅ Works | Native — `anthropic-dangerous-direct-browser-access: true` header |
| Google Gemini | ✅ Works | Native CORS support |
| Groq | ❌ Blocked | Retry via `corsproxy.io` proxy — silent fallback |
| OpenAI | ❌ Blocked | No fallback — GPT-4o only works locally |

Groq fallback pattern:
```js
try {
  r = await fetch('https://api.groq.com/...');
} catch(e) {
  r = await fetch('https://corsproxy.io/?url=https://api.groq.com/...');
}
```

---

## Prompt Architecture

### Vision System Prompt Structure

```
[Role framing — 15 years experience, precise, evidence-based]
[Industry context — optional, injected if selected]
[Persona context — optional, injected if filled]
[Quality checklist — ✓ what a good finding looks like]
[Avoid list — ✗ what not to report]
[Critical rules — confidence ≥70, no quota filling, consistency]
[JSON schema — exact field names, types, examples]
[Positioning rules — zone-based 3×3 grid with x/y ranges]
[Scoring rules — Nielsen heuristic order, calibration]
```

### Positive Prompt Techniques Used

- **Role framing** — "15 years of experience... known for being precise, evidence-based, and actionable" primes the model to self-regulate quality
- **Concrete examples in schema** — `"the red '20% OFF' badge on the Corporate Mid Back chair"` shows the model exactly how specific to be
- **Checklist format** — ✓/✗ lists are processed more reliably than paragraphs
- **Step-by-step positioning** — STEP 1 / STEP 2 / STEP 3 forces a reasoning sequence before outputting coordinates
- **Persona calibration** — "65-year-old low-tech user has different pain points than a power user" gives the model a concrete mental model

### Negative Prompt Techniques Used

- **Explicit avoid list** — `✗ Generic issues ("navigation could be clearer")` with examples of what NOT to write
- **Quota warning** — "Do NOT invent issues to fill a quota. 5 strong findings beat 8 weak ones."
- **Confidence gate** — "Only include findings with confidence ≥ 70"
- **Fabrication warning** — "never fabricate problems for low scores" on the scoring section
- **Consistency requirement** — "Same screenshot analyzed twice → same findings"

### Evaluator Prompt
Text-only models (Groq, Claude Eval) receive a plain-text summary of findings — no image. They score only based on what the vision model found. This prevents them from inventing additional issues.

Key rule: `"If a heuristic is not mentioned in the findings, score it 7"` — avoids artificially low scores on unmentioned dimensions.

---

## Coordinate System

### Why Zone-Based (not pixel-grid)

Earlier versions sent a 10×10 and 20×20 reference grid drawn onto the JPEG before sending to the model. Labels like `"0.35,0.40"` were printed at grid intersections.

**Problem:** JPEG compression made the small labels unreadable. Models were guessing coordinates rather than reading the grid, giving no improvement over the baseline.

**Solution — Zone-based 3×3 grid:**
1. Model names the element and its location in text ("bottom of left sidebar")
2. Model assigns one of 9 zones: `top-left | top-center | ... | bottom-right`
3. Model sets x/y within that zone's enforced range
4. Prompt enforces consistency: "bottom-left" means `x < 0.34 AND y > 0.66` — no exceptions

This uses the model's language understanding rather than its unreliable pixel-reading.

### Known Limitation
Even with zones, pins can be off by ~15–20% on dense UIs. The **Edit → Move Pin** feature allows manual repositioning. Pixel-perfect accuracy would require DOM access via a backend (Puppeteer + `getBoundingClientRect()`).

---

## Finding Deduplication

```js
function deduplicateAnnotations(anns) {
  // >60% word overlap in title → drop the duplicate
}
```

Applied after merging findings from multiple vision models. Prevents the same issue (e.g. "low contrast button") from appearing twice with slightly different wording.

---

## localStorage Keys

| Key | Content | Set by |
|-----|---------|--------|
| `uxaudit_keys` | `{ claude, groq, gemini, openai }` API keys | Model card inputs |

Keys are saved on every `input` event and restored on `DOMContentLoaded`. Never sent to any server other than the respective AI provider.

---

## Gemini 2.5 Flash — Thinking Block Handling

Gemini 2.5 Flash returns Thinking blocks before the actual response:

```json
parts: [
  { thought: true, text: "Let me analyze..." },  // thinking block — skip
  { text: "{\"annotations\": [...]}" }             // actual answer — use this
]
```

Parser:
```js
const textPart = responseParts
  .filter(p => p.text && !p.thought).pop()   // exclude thought blocks
  || responseParts.filter(p => p.text).pop(); // fallback: last text part
```

`maxOutputTokens: 8192` is required because thinking consumes tokens before the output.

---

## parseJSON Helper

All model responses go through `parseJSON()` which:
1. Tries `JSON.parse()` directly
2. If that fails, extracts the first `{...}` block via regex
3. Strips markdown fences (` ```json `) if present

Handles all edge cases: models that add preamble, models that wrap in markdown, models that return trailing commas.

---

## Swiss Gov / Oblique Context

When `industrySelect = "ch-gov"`, the audit prompt injects a detailed checklist covering:

- **Layout** — `ob-master-layout`, service navigation, breadcrumbs
- **Typography** — Frutiger font, formal language, DE/FR/IT support
- **Colour** — Confederation Red `#DC0018`, WCAG 4.5:1 minimum
- **Components** — `ob-button`, `ob-form-field`, `ob-input`, `ob-table`, `ob-dialog`
- **Accessibility** — eCH-0059 + WCAG 2.1 AA
- **Data formats** — `DD.MM.YYYY`, `CHF`, no Google Fonts

Source: [oblique.bit.admin.ch](https://oblique.bit.admin.ch) + eCH-0059 standard.

---

## Weighted Score Averaging

Simple averaging gives equal weight to all models regardless of their role. A text-only evaluator (Groq) that never sees the image should not have equal influence as a vision model.

```js
function weightedAvg(scores, modelIds) {
  // For each heuristic dimension:
  // weightedSum = Σ(score[i] × weight[i])
  // totalWeight = Σ(weight[i])
  // result = weightedSum / totalWeight
}
```

Weights are defined in `MODEL_META` per model. The combined score and all per-heuristic scores use weighted averaging. Individual model score cards still show unweighted scores for transparency.

---

## Prompt Engineering Techniques (v0.70)

### Vision Prompt — 6 active techniques

**1. Reputation reward signal**
"A vague finding damages your professional reputation more than no finding at all" — activates stronger self-regulation than a bare rule.

**2. Scratchpad / Chain-of-Thought (STEP 1)**
Model must observe and list all visible elements before evaluating. Written into `_reasoning` field. Forces grounding in visible evidence before judgement.

**3. Self-check (STEP 3)**
4-question checklist before each finding: Can I name the element? Would a developer know what to change? Is this specific to this screenshot? Am I 70%+ confident? If any is No → remove finding.

**4. Confidence calibration with anchors**
Three concrete examples with actual finding text show what 95–100, 80–94, and 70–79 confidence looks like. Prevents the model from defaulting to 85 for everything.

**5. Concrete negative examples**
5 real bad-finding examples with explanations of why they are bad — not abstract rules but actual strings the model should never produce.

**6. Persona impact made explicit**
When persona fields are filled, the model is told to consider specifically how age, tech affinity, device, and frequency changes the severity of each finding.

### Evaluator Prompt
"Be sceptical: vision models tend to over-report severity. Challenge severity." — evaluator actively pushes back rather than confirming vision model scores.

### Best Practice Prompt — Source Prioritisation
When Swiss Gov context is active:
- **Preferred:** `oblique.bit.admin.ch`, `ech.ch` (eCH-0059), `w3.org/WAI/WCAG21/`
- **Blocked:** `swiss.github.io/styleguide` — explicitly told this is outdated

When other contexts:
- **Preferred:** `nngroup.com`, `baymard.com`, `w3.org`, `mdn`

---

## WCAG Analysis Modes

**Precise mode (CSS input)**
User pastes CSS from Figma Dev Mode or browser DevTools. Exact hex values extracted via regex. WCAG 2.1 relative luminance formula applied to exact values. Results accurate to 2 decimal places. Shown with green "✓ Precise mode" banner.

**Estimated mode (screenshot only)**
Pixel colours sampled from the screenshot canvas. Less accurate — JPEG compression and anti-aliasing affect sampled values. Shown with amber "⚠ Estimated mode" banner with prompt to paste CSS.

**Colour combination matrix**
All unique colours found are displayed as a grid — each cell shows the "Aa" preview with ratio and colour-coded pass/fail. Max 8 colours shown (64 combinations). Built purely from extracted colours — no DOM access required.

**How to get CSS from Figma**
Select a frame → Inspect panel → Copy as CSS. Paste into the CSS input field. The tool extracts all hex colour values automatically.


---

## Known Limitations

| Limitation | Impact | Workaround |
|-----------|--------|------------|
| GPT-4o blocked by CORS on GitHub Pages | GPT-4o unusable in production | Use locally / open index.html directly |
| Groq via corsproxy.io | Slight latency, third-party proxy | Acceptable for evaluator role |
| Pin accuracy ~±15% on dense UIs | Pins may miss exact element | Manual move via Edit → Move Pin |
| No audit history | Can't compare scores over time | PDF export per audit |
| Gemini Free Tier: 10 req/min, 500/day | Rate limiting on heavy use | Upgrade to paid or use Claude only |
| Gemini Free Tier: possible data training | Privacy concern for sensitive screens | Use Claude only for confidential UIs |
| Single HTML file — ~146kb | Growing size, harder to maintain | Acceptable for GitHub Pages, no build needed |
