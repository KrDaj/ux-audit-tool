# Auditly - Interpretation & Usage Guide

v0.74 - May 2026

This guide explains what the AI audit does, what it does not do, and how to interpret the results of each tab. It is intended for auditors using the tool and for stakeholders receiving the exported PDF report.

---

## 1. Audit Modes

The tool has three modes selected at the top of Card 1:

| Mode | Button Label | Purpose |
|------|-------------|---------|
| `uxui` | UX / UI / AX Audit | Standard audit - finds usability issues, scores heuristics, checks WCAG contrast, runs AX structural checks |
| `benchmark` | UX/UI Pattern Research | Identifies UI patterns in the screenshot and searches how reference products solve the same design challenges. Visual only - not a code review. |
| `feedback` | Test Analysis | Paste usability test feedback - Claude cross-references observations with the screenshot and finds reference solutions. |

**What runs per mode:**

| Step | UX/UI/AX | Pattern Research | Test Analysis |
|------|----------|-----------------|---------------|
| Vision AI (findings) | Yes | Yes (pattern prompt) | Yes (feedback prompt) |
| Nielsen evaluator scoring | Yes | No | No |
| WCAG contrast check | Yes | No | No |
| AX structural checks (HTML) | Yes | No | No |
| UX Patterns tab | No | Yes | No |
| Feedback Analysis tab | No | No | Yes |

**Audit Context (Card 2)** adapts per mode:
- UX/UI/AX: all fields (Industry, Country, Product, Flow, Known issues, Persona)
- Pattern Research: Product + Flow only (brands define scope)
- Test Analysis: Product + Flow + Persona (feedback replaces Known issues)

---

## 3. What the Audit Does

### UX/UI Audit - Vision AI Analysis
The tool sends a screenshot to one or more vision AI models (Claude, Gemini, GPT-4o). Each model analyses the interface image and identifies usability issues against **Nielsen's 10 Usability Heuristics**:

1. Visibility of system status
2. Match between system and the real world
3. User control and freedom
4. Consistency and standards
5. Error prevention
6. Recognition rather than recall
7. Flexibility and efficiency of use
8. Aesthetic and minimalist design
9. Help users recognise, diagnose and recover from errors
10. Help and documentation

Each finding is assigned a **severity (1–4)**, a **confidence score (0–100%)** and a **zone location** on the screenshot. Multiple models vote on severity - scores are weighted by model reliability.

### AX Audit - Rule-Based Accessibility Analysis
When HTML source is provided, the tool runs **structural WCAG 2.1 checks directly in the browser** - no API call, no tokens consumed:

| Check | WCAG Criterion | Level |
|-------|---------------|-------|
| Images without alt attribute | 1.1.1 | A |
| Alt text = filename | 1.1.1 | A |
| Heading hierarchy skipped | 1.3.1 | A |
| Multiple H1 elements | 1.3.1 | A |
| Inputs without label | 1.3.1 | A |
| Buttons without accessible name | 4.1.2 | A |
| Links without accessible name | 2.4.4 | A |
| Non-descriptive link text ("click here") | 2.4.6 | AA |
| Missing lang attribute | 3.1.1 | A |
| Positive tabindex | 2.4.3 | A |
| ARIA role misuse | 4.1.2 | A |
| Missing landmark regions | 1.3.6 | AAA |

If a screenshot is also provided, a **Visual AX pass** runs via Claude or Gemini - checking focus indicators, touch target sizes, contrast and alt text quality against the visible image.

### Quality Mode - Multi-Model Iteration
When enabled, findings go through a specialist review pipeline before being shown:

- **Groq** checks specificity (is the UI element named?)
- **Apertus** checks relevance (is this a real visible problem?)
- **GitHub Models** checks fixability (is the recommendation actionable?)
- **Claude** synthesises all feedback and revises flagged findings

Findings removed by the review are not shown. Findings revised are marked **✓ Revised**.

---

### UX/UI Pattern Research
Upload a prototype screenshot. Claude identifies the UI patterns visible on screen (navigation, forms, CTAs, card layouts etc.) and searches how reference products (Amazon, Galaxus, GOV.UK etc.) solve the same visual design challenges. Includes research citations.

- Input: screenshot + optional reference brands
- Output: Pattern Research tab with brand comparisons and citations
- Not a code review - focuses on visual UI patterns only

### Test Analysis
Upload a screenshot and paste raw usability test feedback (observations, quotes, task completion times). Claude cross-references what users reported with what is visible on screen and finds how reference products solve each issue.

- Input: screenshot + feedback textarea (observations, quotes, metrics)
- Output: Feedback Analysis tab with root causes and reference solutions
- Persona context from Card 2 helps Claude interpret who the test participants were

---

## 4. What the Audit Does NOT Do

| Limitation | Why | What to do instead |
|-----------|-----|-------------------|
| Cannot test keyboard navigation | Requires a running browser session | Manual tab-through test |
| Cannot run a screenreader | Requires NVDA, JAWS or VoiceOver | Manual screenreader test |
| Cannot test dynamic interactions | Screenshot is static | Record interaction video, test manually |
| Cannot guarantee WCAG conformance | Automated checks cover structural criteria only (~30–40% of WCAG) | Commission a formal accessibility audit |
| Cannot issue a legal accessibility declaration | Not a certified auditor | Contact your accessibility officer |
| Cannot detect all colour contrast issues | Pixel sampling is approximate without CSS | Paste CSS from Figma Dev Mode for precise ratios |
| Cannot assess cognitive load in context | Vision AI sees layout, not user behaviour | Conduct usability testing |
| May produce hallucinated findings | AI models occasionally flag non-existent issues | Use Challenge function; apply 80%+ confidence filter |

---

## 5. How to Interpret Each Tab

### Annotated Screenshot
Numbered pins mark where each finding is located on the interface. Click a pin or finding in the sidebar to select it.

- **Pin colour** reflects severity: red = severity 4 (critical), orange = 3, yellow = 2, grey = 1
- **Pin number** matches the finding number in the sidebar and PDF
- Pins are placed using a zone system (top-left, centre, bottom-right etc.) - exact pixel position may vary ±15%

### Nielsen Scores
Radar chart and table showing scores for each of the 10 heuristics (1–10 scale):

| Score | Interpretation |
|-------|---------------|
| 1–4 | Serious usability problem - prioritise immediately |
| 5–6 | Minor issue - fix in next sprint |
| 7–9 | Acceptable - room for improvement |
| 10 | Exemplary |

Scores are weighted averages across all active AI models. Each model contributes according to its reliability weight (Claude ×1.0, Gemini ×0.9, Groq evaluator ×0.6 etc.).

**Important:** Scores reflect findings in the screenshot only. If no finding is identified for a heuristic, the model defaults to 7 (acceptable) - this does not mean the heuristic is perfectly implemented.

### WCAG Contrast
Colour pairs extracted from the screenshot (or from pasted CSS) are tested against WCAG 2.1 contrast ratios:

| Result | Ratio | Meaning |
|--------|-------|---------|
| ✓ Pass | ≥ 4.5:1 (text) / ≥ 3:1 (large text/UI) | Meets WCAG AA |
| ⚠ Borderline | 3.5:1–4.4:1 | Marginal - test with actual users |
| ✗ Fail | < 3.5:1 | Does not meet WCAG AA - fix required |

**Without CSS:** Colours are sampled from screenshot pixels - ratios are approximate. Paste CSS from Figma Dev Mode for exact hex values and precise ratios (shown to 2 decimal places).

### AX Audit
Issues grouped by category with impact levels:

| Impact | Meaning | Action |
|--------|---------|--------|
| Critical | Blocks access entirely for affected users | Fix before release |
| Serious | Significantly impairs access | Fix in current sprint |
| Moderate | Creates friction but workaround exists | Fix in next sprint |
| Minor | Best practice - low impact | Address when time allows |

**Visual (AI) category:** Issues found by vision AI that are not detectable from HTML alone (focus indicators, contrast, touch targets). These require manual verification.

**Coverage note:** Automated checks cover structural WCAG Level A criteria. Cognitive, contextual and interaction-based criteria require manual testing.

---

## 6. How to Interpret Findings

### Confidence Score
The percentage shown on each finding indicates how confident the AI model is that the issue is real and significant:

| Confidence | Meaning |
|-----------|---------|
| 90–100% | High confidence - likely a real issue |
| 80–89% | Good confidence - review before acting |
| 70–79% | Moderate - verify manually |
| < 70% | Low - treat as hypothesis only |

Use the **Min. confidence filter** (≥ 80% recommended) before exporting to PDF to exclude speculative findings.

### Severity
Severity 1–4 reflects the impact on usability:

| Severity | Meaning |
|---------|---------|
| 4 | Catastrophic - prevents task completion |
| 3 | Major - significantly impairs task completion |
| 2 | Minor - causes friction but task completable |
| 1 | Cosmetic - does not affect task completion |

### Status Badges
| Badge | Meaning |
|-------|---------|
| ✓ Confirmed | You manually confirmed this finding is valid |
| Challenged | You submitted a counter-argument; AI re-evaluated |
| ✓ QM | Finding passed Quality Mode specialist review |
| ✓ Revised | Finding was revised by Claude after specialist feedback |

### Challenge Function
Use Challenge when you believe a finding is incorrect or exaggerated. Provide a counter-argument and optionally a reference URL. Claude fetches the URL (up to 1500 chars) and re-evaluates:

- **Remove** - finding deleted from results
- **Downgrade** - severity reduced
- **Keep** - finding retained as-is
- **Confirmed** - finding strengthened

---

## 7. For Swiss Federal Government Context

### Applicable Standards
| Standard | Scope | Reference |
|---------|-------|-----------|
| eCH-0059 | Accessibility for federal web applications | https://www.ech.ch |
| WCAG 2.1 AA | Web Content Accessibility Guidelines | https://www.w3.org/TR/WCAG21/ |
| BehiG | Behindertengleichstellungsgesetz | https://www.fedlex.admin.ch |
| Oblique Design System | BIT component library | https://oblique.bit.admin.ch |

### Oblique Components
When auditing applications using the Oblique Design System, the following components are checked against Oblique specifications:

- `ob-master-layout` - overall page structure
- `ob-form-field` - form inputs and labels
- `ob-button` - button styles and states
- `ob-dialog` - modal dialogs

### Data Privacy
The tool sends screenshot images and HTML to third-party AI providers (Anthropic, Google, Groq). For public-facing pages without personal data this is acceptable. **Do not use this tool with screenshots containing personal data or internal confidential information without legal review.**

API keys are stored locally in your browser and are never sent to any server other than the AI provider.

---

## 8. Recommended Workflow

```
1. Prepare
   Upload screenshot of the page to audit
   Optional: paste CSS from Figma Dev Mode (for precise contrast)
   Optional: paste HTML source (for AX structural checks)
   Set Audit Context: product name, step in flow, known issues

2. Configure
   Select region (CH / DE / EU etc.)
   Select industry (Swiss Gov / E-Commerce / SaaS etc.)
   Enable Quality Mode for higher accuracy (slower, more API calls)

3. Run
   Click Run Audit →
   Wait for vision analysis + evaluator scoring
   AX audit runs automatically if HTML was provided

4. Review
   Filter findings by confidence (≥ 80% recommended)
   Review each finding - use Challenge for false positives
   Add comments with Jira/Confluence links

5. Export
   Choose report type (Full / UX/UI / AX)
   Export PDF → share with team or attach to Jira epic
```

---

## 9. Limitations Summary

This tool is a **decision-support tool**, not a certification system. Results should be:

- **Reviewed** by a UX professional before acting
- **Verified** manually for high-severity findings
- **Supplemented** with usability testing for cognitive and interaction issues
- **Not used alone** as evidence for WCAG conformance declarations

For formal accessibility audits, engage a certified accessibility auditor.
