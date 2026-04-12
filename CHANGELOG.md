# Changelog

All notable changes to the UX / UI Audit Tool are documented here.  
Format: [Semantic Versioning](https://semver.org) — `MAJOR.MINOR.PATCH`

---

## v0.50 — April 2026

### Added
- **Confidence score per finding** — model returns `confidence: 0–100` per annotation; displayed as colour-coded badge (green ≥80%, amber ≥60%, red <60%); helps distinguish reliable findings from uncertain ones
- **Edit findings inline** — hover any finding to reveal ✎ Edit button; opens inline form to correct title and description; Save / Cancel
- **Delete findings** — ✕ button removes a finding from sidebar and canvas instantly
- **Move pin** — "📍 Move pin" in edit form activates crosshair mode; next click on the screenshot repositions the pin to the correct element
- **Element description in prompt** — model must now name the UI element before giving coordinates (e.g. "price range slider track in top-left sidebar"); coordinate and description are cross-checked for consistency
- **20×20 reference grid** — finer grid overlay (up from 10×10) gives model 2-decimal coordinate precision (±2.5% vs ±5%)
- **Gemini 2.5 Flash** — upgraded from deprecated `gemini-1.5-flash-latest`; handles Thinking block response format correctly
- **How it works** — three-panel strip below hero: 01 Upload → 02 Analyse → 03 Export
- **Sticky Run Audit button** — appears at bottom of screen when main button scrolls out of view; synced framework dropdown
- **Results section separator** — visual divider between input and results areas
- **Favicon** — red SVG square with "UX" in browser tab
- **Meta description** — for browser and sharing previews

### Fixed
- Gemini 2.5 Flash response parsing — model returns Thinking blocks before JSON answer; now correctly extracts last non-thought text part
- Footer cleaned up — removed redundant "UX Audit Tool · Keys stay in your browser" text; now shows only source references and version

---

## v0.45 — April 2026

### Added
- **API key persistence** — keys saved to `localStorage` on input, restored on page load; no more re-entering after refresh
- **Onboarding panel** — collapsible "How to get an API key" in the Models card with links to Anthropic, Groq, Gemini, OpenAI consoles
- **Finding deduplication** — when multiple vision models run in parallel, near-duplicate findings (>60% word overlap) are automatically merged
- **Code fix language selector** — dropdown next to "Run Audit →"; set once, applies to all "Generate fix →" outputs; options: HTML/CSS, Angular, React, Vue 3, Svelte
- **PDF severity legend** — Score Summary page now includes a full severity legend (SEV 0–4 with descriptions, or High/Medium/Low) and score rating legend (Good / Needs work / Poor with ranges)

### Improved
- **Angular fix prompt** — generates targeted, copy-paste-ready snippets with file comments (`// product.component.html`), `export class`, explicit return types, typed interfaces; no fake `@Component` boilerplate
- **PDF finding blocks** — badge label respects selected severity scale (Nielsen vs simple); tighter spacing; vertical divider matches exact block height; recommendation text reframed as action ("Ensure that…") not copy-paste
- **Best practice JSON parsing** — extracts last text block from Claude web search response (avoiding "Now let me search…" prose); system prompt enforces JSON-only output
- **`parseJSON` helper** — falls back to regex extraction (`/\{[\s\S]*\}/`) when direct parse fails; handles all model response formats
- **Labels cleaned** — removed pricing hints from onboarding; "Code fix language" replaces "Frontend framework"

### Fixed
- Best practice panel showing raw JSON when Claude web search returned prose before the JSON block

---

## v0.40 — April 2026

### Added
- **Results fade-in** — findings appear with a 0.4s opacity transition instead of appearing abruptly
- **Score card tooltips** — hovering a heuristic score card shows the full heuristic name (e.g. "Visibility of System Status") with an arrow tooltip
- **Run button pulse animation** — button glows red during analysis, text changes to "Analyzing…" and resets to "Run Audit →" on completion
- **Keyboard shortcut** — `⌘ Enter` (Mac) / `Ctrl+Enter` (Windows) triggers the audit; hint shown next to the run button
- **Findings counter in tab** — tab label shows "Annotated screenshot (6)" so issue count is immediately visible
- **Sidebar empty state** — when no issues are found, a green "✓ No issues detected" block is shown instead of an empty list
- **Color band on score cards** — overall score cards get a coloured top border (green / amber / red by score); model cards get the model's brand colour
- **PDF green empty state** — heuristic pages with no findings show a green "✓ No issues detected for this heuristic" block instead of grey text
- **Version number in footer** — `v1.8.0` shown discretely next to "Keys stay in your browser"
- **Animated loading ticker** — progress bar + rotating messages during audit showing real research sources (NNG, Baymard, ACM CHI, WCAG); industry-aware phrasing
- **Value proposition in hero** — sourced statistics below the description: NNG 1,000+ articles, Baymard 130,000+ hours, WCAG 78 criteria; footnotes with links in footer
- **Redundant eyebrow removed** — "UX / UI Audit Tool" label below logo was duplicate; removed

### Fixed
- `willReadFrequently` canvas warning — added `{willReadFrequently:true}` to `sampleColors()` and `runWCAGCheck()` contexts
- Groq `Failed to fetch` now fails silently (CORS on GitHub Pages); proxy fallback handles retry automatically

---

## v0.35 — April 2026

### Added
- **Frontend framework dropdown** in Audit Context card — Angular, React, Vue 3, Svelte, or Generic HTML/CSS; "Generate fix →" produces framework-specific code
- **Angular fix prompt** — generates real Angular 17+ code: `@Component` template syntax, TypeScript class, SCSS; Before/After labelled as `component.html` / `component.ts` / `component.scss`
- **Groq CORS proxy fallback** — all Groq calls (eval, fix, best practice) automatically retry via `corsproxy.io` when blocked by GitHub Pages CORS; no warning shown to user

### Fixed
- `AbortSignal object could not be cloned` — replaced `fetchWithTimeout` spread `{...opts, signal}` with plain `fetch()` for all vision calls; removed AbortController entirely from large-payload requests
- Groq `Failed to fetch` warning suppressed — CORS failures fail silently, Claude covers scoring
- `Canvas2D: willReadFrequently` warning — added `{willReadFrequently:true}` to both `sampleColors()` and `runWCAGCheck()` canvas contexts

### Improved — PDF Export
- **Score Summary table** — fixed column positions (`COL_NUM`, `COL_NAME`, `COL_BAR`, `COL_SCORE`, `COL_RATING`); score and rating now on the same baseline; rating no longer overlaps score number
- **Heuristic pages** — large number and title correctly baseline-aligned; two-column divider drawn per finding instead of fixed 100mm; Recommendations column shows real actionable text instead of copy-pasting the issue description
- **Block height** pre-calculated from actual line counts — no more text overflow or overlapping between findings
- **Heuristic matching** rewritten with keyword list per heuristic — findings reliably appear under the correct heuristic page

### Improved — Best Practice panel
- New structured layout: "Why it matters" with source citation and clickable link; company examples with coloured initials badge and ↗ source link; Quick Fix in green accent block
- Prompt updated to request `source`, `source_url` fields for verifiable citations
- Raw JSON no longer shown on parse failure — falls back to clean text rendering

---

## v0.30 — April 2026

### Fixed
- Constants self-reference bug (`CLAUDE_MODEL = CLAUDE_MODEL`) causing `ReferenceError` on load

### Optimised — Code Quality
- Extracted API model names into constants (`CLAUDE_MODEL`, `GROQ_MODEL`, `GPT_MODEL`, `GEMINI_MODEL`) — single place to update on model changes
- Extracted `claudeHeaders(key)` helper — replaces 5× repeated Claude header objects
- GPT-4o and Gemini vision functions now correctly use `cssCtx` and `sys` parameters — no more inline `el('cssInput').value` reads inside API calls

### Optimised — Token Savings
- Groq evaluator: compact prompt (~150 tokens vs ~530), `max_tokens` reduced from 400 → 200 — ~60% cheaper per Groq call
- Best practice lookup: `max_tokens` reduced from 1000 → 700
- Region audit: now injects CSS context correctly
- Gemini URL uses `GEMINI_MODEL` constant

---

## v0.25 — April 2026

### Optimised — Performance
- **Screenshot compression** — images resized to max 1280 px, JPEG 85% quality before sending; saves ~75% vision tokens; size shown in preview
- **Claude Prompt Caching** — system prompt cached with `cache_control: ephemeral`; ~90% cheaper on repeated audits within 5 minutes
- **Parallel model execution** — all vision and evaluator models run concurrently via `Promise.all`; ~50% faster with multiple models
- **`visionSystem()` cached once per audit** — built once before `Promise.all`, passed as parameter to all model calls
- **CSS input debounced** — `parseCSSLive()` fires 300ms after last keystroke instead of on every keypress
- **Font-display: swap** — Google Fonts no longer blocks first render; fallback font shown immediately
- **Canvas redraw guard** — annotations only redrawn when active index changes
- **Upload listener guard** — event listeners attached only once via `_listenersAttached` flag

### Fixed
- NaN combined score when evaluator models return fewer than 10 scores
- `temperature: 0` set on all 8 API calls
- All prompts updated from 7 to 10 Nielsen heuristic scores
- Anti-hallucination instructions added to vision and evaluator prompts

---

## v0.20 — April 2026

### Added
- **Persona-based audit** — optional fields for age, usage frequency, tech affinity and device; findings are prioritised for the specified user group
- **Industry context dropdown** — 7 options including Generic, Swiss Government (Bund/Oblique-konform), E-Commerce, SaaS, Banking, Healthcare, Marketing; each injects domain-specific evaluation criteria into the prompt
- **Nielsen severity scale 0–4** — formal alternative to High / Medium / Low; badges in the sidebar and PDF update dynamically
- **Swiss Government mode** — evaluates against BIT Oblique, eCH standards, WCAG 2.1 AA, and multilingual conventions (DE / FR / IT)

### Changed
- Input section reorganised into three numbered cards: URL & Screenshot → Audit Context → AI Models

---

## v0.15 — April 2026

### Added
- **Screenshot compression** — images are resized to max 1280 px width and compressed to JPEG 85 % before sending; saves ~75 % of vision tokens with negligible quality loss; size info shown in preview
- **Claude Prompt Caching** — system prompt is cached with `cache_control: ephemeral`; subsequent audits within 5 minutes cost ~10 % of normal prompt price
- **Parallel model execution** — vision models (Claude, GPT-4o, Gemini) and evaluator models (Groq) now run concurrently via `Promise.all`; reduces wait time by ~50 % when using multiple models

### Fixed
- **NaN combined score** — undefined scores from evaluator models (which return fewer dimensions) are now filtered before averaging
- **temperature: 0 on all API calls** — ensures consistent, reproducible results across repeated audits
- **Anti-hallucination prompts** — vision prompt now requires direct visual evidence for every finding; evaluator prompt restricts scoring to listed findings only
- **All prompts updated from 7 to 10 heuristic scores** — Groq evaluator was returning 7 scores causing index mismatches

---

## v0.10 — April 2026

### Added
- **PDF export — NNG Heuristic Evaluation Workbook format** — structured as a formal workbook inspired by the Nielsen Norman Group template
  - Cover page with large typography, URL, date, framework reference
  - Annotated screenshot page (full width, correct aspect ratio)
  - Score summary as table with progress bars and Good / Needs improvement / Poor ratings
  - One page per Nielsen heuristic with Issues and Recommendations columns
  - Dark footer with page numbers on every page
- **Nielsen's 10 Usability Heuristics** replace the previous 7 custom dimensions across all scoring, prompts and PDF

### Changed
- Scores grid updated to 5-column layout (2 rows of 5) to accommodate 10 heuristics
- All API prompts updated to request 10 scores in the correct heuristic order

---

## v0.05 — April 2026

### Added
- **WCAG 2.1 contrast checker tab** — samples colours from the screenshot using the relative luminance formula; reports Fail / AA / AAA with hex codes and contrast ratios
- **CSS / Figma Dev Mode input** — optional collapsible field; when pasted, WCAG uses exact hex values instead of pixel sampling and the AI receives design token context
- **Before / After code fix generator** — "Generate fix →" button per finding; produces a side-by-side CSS / HTML snippet with explanation and impact rating (uses Groq for cost efficiency, Claude as fallback)
- **Best practice lookup with web search** — "Best practice →" button per finding; uses Claude with live web search to surface 2–3 real-world examples and a quick-fix sentence
- **Region selection audit** — draw a bounding box on the screenshot to audit a specific component; findings are added as a separate green-labelled layer
- **Annotated screenshot** — numbered pins drawn directly on the screenshot via Canvas; clickable and cross-linked with the sidebar
- **Multi-model pipeline** — vision-capable models (Claude, GPT-4o, Gemini) analyse the screenshot; text-only models (Groq) evaluate findings and contribute scores; all results averaged
- **Screenshot upload + drag & drop** — PNG, JPG, WebP supported; image passed as base64 to vision models
- **Model key management** — any combination of keys can be entered; only configured models run

### Changed
- Redesigned UI inspired by Swiss Federal / Bund Design System — Source Sans 3 typeface, Bundesrot `#DC0018` accent, sharp corners, structured input cards
- Together AI removed (no longer offers a free tier)
- Gemini model corrected to `gemini-1.5-flash-latest`

### Fixed
- Hard requirement for Claude key removed; any vision model key is sufficient
- GPT-4o and Gemini noted as CORS-restricted on GitHub Pages (Claude recommended for hosted deployments)
- Canvas memory leak — old event listeners are removed before new ones are added on each audit

---

## v0.01 — April 2026

### Added
- Initial release
- URL input + screenshot upload
- Multi-model audit: Claude Sonnet 4, GPT-4o, Gemini 1.5 Flash, Groq Llama 3.3 70B
- 7-dimension scoring (Navigation, Visual Hierarchy, Accessibility, Conversion, Mobile UX, Content Clarity, Performance)
- Scores tab with per-model and combined scores
- Basic PDF export
- GitHub Pages deployment (`index.html` — no build step required)
- API keys stored in browser only — never transmitted to any server other than the respective AI provider
