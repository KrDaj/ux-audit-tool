# Changelog

All notable changes to the UX / UI Audit Tool are documented here.  
Format: [Semantic Versioning](https://semver.org) — `MAJOR.MINOR.PATCH`

---

## v1.4.0 — April 2026

### Added
- **Persona-based audit** — optional fields for age, usage frequency, tech affinity and device; findings are prioritised for the specified user group
- **Industry context dropdown** — 7 options including Generic, Swiss Government (Bund/Oblique-konform), E-Commerce, SaaS, Banking, Healthcare, Marketing; each injects domain-specific evaluation criteria into the prompt
- **Nielsen severity scale 0–4** — formal alternative to High / Medium / Low; badges in the sidebar and PDF update dynamically
- **Swiss Government mode** — evaluates against BIT Oblique, eCH standards, WCAG 2.1 AA, and multilingual conventions (DE / FR / IT)

### Changed
- Input section reorganised into three numbered cards: URL & Screenshot → Audit Context → AI Models

---

## v1.3.0 — April 2026

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

## v1.2.0 — April 2026

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

## v1.1.0 — April 2026

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

## v1.0.0 — April 2026

### Added
- Initial release
- URL input + screenshot upload
- Multi-model audit: Claude Sonnet 4, GPT-4o, Gemini 1.5 Flash, Groq Llama 3.3 70B
- 7-dimension scoring (Navigation, Visual Hierarchy, Accessibility, Conversion, Mobile UX, Content Clarity, Performance)
- Scores tab with per-model and combined scores
- Basic PDF export
- GitHub Pages deployment (`index.html` — no build step required)
- API keys stored in browser only — never transmitted to any server other than the respective AI provider
