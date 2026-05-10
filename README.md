# DQA-skill (Design Quality Audit)

A skill for Claude that produces structured Markdown audit reports comparing a live page or
source code against its Figma design reference and the project's active design system.

Two modes are supported — no codebase access required for URL mode:

| Mode | Who uses it | What is analysed |
|---|---|---|
| **URL mode** | Designer with a staging/ODE link | Fetched HTML + linked CSS files |
| **Codebase mode** | Engineer with source files | Source file imports + styles |

---

## What it does

Given a Figma link and either a live URL or source files, the skill:

1. Fetches design context from Figma (layout, typography, spacing, color tokens, component names)
2. **URL mode:** Fetches the live page HTML and all linked CSS files; decodes Tailwind classes,
   extracts CSS custom properties (design tokens), and identifies hardcoded vs token-based values
3. **Codebase mode:** Analyses source files for design system usage, hardcoded constants,
   legacy imports, and component mapping
4. Builds a property comparison table: Figma value vs page/code value, with match status
5. Outputs a structured `.md` audit report with severitised findings, token usage summary,
   and concrete remediation steps

The report is always saved as `design-audit-[page-name]-[YYYY-MM-DD].md`.

---

## When Claude triggers this skill

- "Run a design audit on this page"
- "Check how closely our implementation matches the Figma"
- "Design accuracy check for the checkout page"
- "Figma vs code comparison"
- "ODE review" / "design QA on this link"
- "Check this staging URL against the Figma design"
- "Migration audit" (e.g. old design system → new)
- Any request pairing a Figma link with a page URL or file paths and asking for a written report

---

## Inputs required

### URL mode (designer / ODE workflow)

| Input | Required? |
|---|---|
| Figma link (file URL + ideally specific frame/node) | Required |
| Live URL (ODE, staging, or production link) | Required |
| Page or section name | Required |
| Design system token prefix / CSS variable convention | Recommended |
| Known deprecated libraries or patterns to flag | Optional |

### Codebase mode (engineer workflow)

| Input | Required? |
|---|---|
| Figma link (file URL + ideally specific frame/node) | Required |
| Page route(s) or component path(s) | Required |
| Source files (paste or "Add to chat") | Required |
| Design system package names (e.g. `@acme/ui-core`) | Recommended |

Claude will ask for any missing required inputs before starting.

---

## Report structure

```
# [Page Name] Design Accuracy Audit
├── Scope
├── Figma MCP Status
├── Data Source Status              ← URL mode only: CSS files fetched, tokens found
├── Overall Assessment
├── Design vs Page: Property Comparison Table   ← Figma value vs page value, per property
├── Token Usage Summary             ← URL mode: token refs vs hardcoded values by type
├── Design System Usage Summary     ← Codebase mode: ✅ / ⚠️ / ❌ / 🔲 per section
├── Migration Status by Area        ← Codebase mode
├── Findings                        ← numbered, ordered High → Medium → Low
├── Concrete Flags
│   ├── High confidence issues
│   └── Not currently true          ← explicit list of false assumptions the audit disproved
├── Where the Page Still Needs Work
└── Bottom Line
```

---

## Severity levels

| Level | Meaning |
|---|---|
| **High** | Visible design deviation a user or designer would notice: wrong color, spacing off by >4px, wrong font size |
| **Medium** | Correct visual value but hardcoded instead of using a token, or bespoke override creating drift risk |
| **Low** | Cleanup opportunity: arbitrary constants, minor token inconsistency, no visible impact |

---

## What the audit covers

**URL mode — always audited:**
- Color values on key elements vs Figma color spec
- Spacing (padding, gap, margin) vs Figma layout spec
- Typography (size, weight, line height) vs Figma typography spec
- Token usage: CSS custom properties (`var(--*)`) vs hardcoded values
- Tailwind arbitrary value detection (`px-[28px]`, `bg-[#1a1a2e]`)
- Design system fingerprint (Radix, MUI, Chakra, custom DS, or none)
- Border radius, border color accuracy

**Codebase mode — always audited:**
- Design system library imports — current vs legacy vs bespoke
- Hardcoded constants vs token usage (arbitrary widths, gaps, heights, z-indices)
- Component mapping per section
- Mobile vs desktop container separation

**Both modes — when Figma data is available:**
- Exact spacing, typography, and color parity against the Figma frame
- Component name mapping (Figma component → code component)
- Responsive breakpoint alignment

---

## URL mode limitations

- **JS-rendered SPAs**: If the page renders via JavaScript, the fetched HTML may be an empty shell. The audit covers what is present in the initial HTML and CSS files only.
- **CSS-in-JS**: Styles injected by styled-components or Emotion are not in CSS files — only in the browser runtime. URL mode cannot see them; switch to Codebase mode.
- **Authentication-gated pages**: If the URL redirects to a login page, the audit stops and reports this.
- **Computed vs authored styles**: CSS cascade overrides are not always visible from authored CSS alone.

---

## Figma MCP tools used

```
Figma:get_design_context   ← primary
Figma:get_screenshot        ← visual reference
Figma:get_variable_defs     ← token/variable definitions
Figma:get_metadata          ← fallback
```

If all Figma calls fail, the audit continues with page/code-only analysis and notes exactly
what was and wasn't available.

---

## Example finding (URL mode)

**Weak** — do not write findings like this:
> The hero section appears to have less padding than the design.

No CSS source, no Figma value, no page value. Not audit language.

**Strong** — findings must look like this:
> **2. Hero section vertical padding is 32px on the page vs 48px in Figma**
> **Severity:** High | **Area:** Hero section
> **Evidence source:** `.hero-section` rule in `main.abc123.css`
>
> The Figma frame specifies 48px vertical padding; the live page applies 32px.
> This is a visible gap — the section appears more cramped than designed on desktop.
>
> ```css
> .hero-section { padding: 32px 24px; }
> ```
> **Figma reference:** 48px | **Page value:** 32px
> **Fix:** Update to `padding: 48px 24px` or `var(--spacing-12)`.

Every URL-mode finding must cite both the Figma value and the CSS source with the page value.
