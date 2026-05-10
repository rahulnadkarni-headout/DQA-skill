---
name: dqa-skill
description: >
  Performs a structured design accuracy audit comparing a live page against a Figma design
  reference and the project's active design system. Accepts either (a) codebase source files
  or (b) a live URL such as an ODE (On-Demand Environment) or staging link — no codebase
  access required for URL mode. Use whenever the user asks for a "design audit", "design
  accuracy check", "Figma vs code comparison", "ODE review", "design QA", or provides a
  Figma link alongside a page URL or route. The output is always a structured Markdown audit
  report saved as a file.
---

# Design Accuracy Audit Skill

Produces a structured Markdown audit report comparing a live page or source code against its
Figma design reference and the project's current design system.

Two operating modes are supported:

| Mode | When to use | What you analyse |
|---|---|---|
| **URL mode** | Designer has a staging/ODE link but no codebase access | Fetched HTML + linked CSS files |
| **Codebase mode** | Engineer has shared source files | Source file imports + styles |

If the user provides a URL, default to **URL mode**. If they share source files only, use
**Codebase mode**. If both are available, prefer URL mode for visual properties and codebase
mode for library/migration analysis; combine findings.

---

## Prerequisites — Gather Before Starting

Do NOT begin the audit until you have all required inputs for the chosen mode.
If any required input is missing, ask explicitly before proceeding.

### URL mode (designer / ODE workflow)

| Input | Required? | What to ask if missing |
|---|---|---|
| **Figma link** | Required | "Please share the Figma file URL and, if possible, the specific frame or node link." |
| **Live URL** (ODE, staging, or production link) | Required | "Please share the link to the rendered page you want audited." |
| **Page or section name** | Required | "What is the name of this page or feature so I can title the report correctly?" |
| **Design system token prefix or CSS variable convention** | Recommended | "What prefix do your design tokens use? E.g. `--color-*`, `--spacing-*`, `ds-*`." |
| **Known deprecated libraries or legacy patterns** | Optional | "Are there any libraries or class patterns I should flag as legacy?" |

### Codebase mode (engineer workflow)

| Input | Required? | What to ask if missing |
|---|---|---|
| **Figma link** | Required | "Please share the Figma file URL and, if possible, the specific frame/node link." |
| **Page route(s) or component path(s)** | Required | "Which page routes or file paths should I audit?" |
| **Codebase files** | Required | "Please share the relevant source files — paste contents or use your editor's 'Add to chat' feature." |
| **Design system package names** | Recommended | "What are the names of your design system packages (e.g. @acme/ui-core, @acme/tokens)?" |

Optional for either mode:
- Screenshots of the rendered page (desktop + mobile)
- Known migration status ("we're migrating from X to Y")
- Names of deprecated libraries to flag

---

## Step 1 — Fetch Figma Design Context

Use the Figma MCP tools in this order. Try each; note failures explicitly in the report.

```
1. Figma:get_design_context  — primary: layout, spacing, typography, color tokens
2. Figma:get_screenshot       — visual reference for the frame
3. Figma:get_variable_defs    — token/variable definitions used in the design
4. Figma:get_metadata         — fallback: node name and type tree
```

**If ALL Figma MCP calls fail or time out:**
- Note this under "Figma MCP Status" in the report
- Continue the audit using user-provided screenshots, Figma URL metadata, and page/code analysis
- Do NOT abort — a partial audit with honest caveats is more useful than nothing

**What to extract from Figma if accessible:**
- Layout: width, max-width, min-width, padding, gap, margin values (in px)
- Typography: font size, weight, line height, letter spacing, font family
- Color: hex or token values for backgrounds, text, borders, icons
- Component names: Figma component names that should map to design-system components
- Spacing tokens: Figma variable names that correspond to design tokens
- Breakpoints: responsive variants in the design

Record all extracted Figma values in a working reference table before moving to Step 2.
You will cross-reference these values against the live page or source code.

---

## Step 2A — Fetch and Analyse the Live URL (URL mode)

_Skip this step if in Codebase mode — proceed directly to Step 2B._

### 2A-1. Fetch the page HTML

Use WebFetch to retrieve the HTML of the provided URL. From the HTML, extract:

- All `<link rel="stylesheet" href="...">` URLs (CSS files to fetch next)
- All inline `<style>` blocks (copy in full)
- All `data-*` attributes that may identify design-system components
- Tailwind or utility class names on key elements (hero, CTA, nav, cards, typography)
- Any `class` attribute values that match design-token patterns (e.g. `ds-*`, `ui-*`, `token-*`)

If the page returns empty body content (JavaScript-rendered SPA), note this and work with
whatever HTML is present. Partial analysis with caveats is still useful.

### 2A-2. Fetch all linked CSS files

For each stylesheet URL found in the HTML:
- Use WebFetch to retrieve the full CSS content
- If a CSS file is minified, still analyse it — look for token patterns even in minified output

From each CSS file, extract and record:

**Design token declarations:**
```
CSS custom property definitions:
  --color-brand-primary: #FF6B35;
  --spacing-md: 16px;
  → Record: token name → resolved value
```

**Color values used:**
```
background: var(--color-brand-primary)     → token reference ✅
background: #FF6B35                        → hardcoded value ⚠️
background: rgb(255, 107, 53)              → hardcoded value ⚠️
```

**Spacing values used:**
```
padding: var(--spacing-md)                 → token reference ✅
padding: 16px                              → hardcoded value ⚠️
gap: 1.5rem                                → hardcoded value ⚠️
margin: 0 24px                             → hardcoded value ⚠️
```

**Typography values:**
```
font-size: var(--text-heading-xl)          → token reference ✅
font-size: 2.25rem                         → hardcoded value ⚠️
font-family: 'Inter', sans-serif           → note for Figma comparison
font-weight: 700                           → note for Figma comparison
line-height: 1.5                           → note for Figma comparison
```

**Border and radius values:**
```
border-radius: var(--radius-md)            → token reference ✅
border-radius: 8px                         → hardcoded value ⚠️
border: 1px solid #E5E7EB                  → hardcoded color ⚠️
```

### 2A-3. Decode Tailwind class names

If the page uses Tailwind CSS, decode class names on key elements into their pixel/hex equivalents
using standard Tailwind v3 defaults (or v4 if evident). Build a mapping table:

| Class | Property | Value |
|---|---|---|
| `px-6` | padding-left + padding-right | 24px |
| `py-12` | padding-top + padding-bottom | 48px |
| `text-4xl` | font-size | 36px / 2.25rem |
| `bg-blue-600` | background-color | #2563EB |
| `gap-4` | gap | 16px |
| `rounded-lg` | border-radius | 8px |
| `text-gray-900` | color | #111827 |

Use this table in Step 3 when comparing against Figma values.

If custom Tailwind values appear (e.g. `bg-[#1a1a2e]`, `px-[28px]`), flag these as
hardcoded arbitrary values — they bypass the token system.

### 2A-4. Identify design system component fingerprints

Look for class naming patterns that reveal which design system (if any) the page uses:

- **Headless/Radix**: `data-radix-*`, `[data-state="open"]`
- **Shadcn/ui**: class patterns like `cn(...)` conventions, Radix primitives
- **Chakra UI**: `chakra-*` classes
- **Material UI**: `MuiButton-root`, `MuiTypography-*`
- **Custom DS**: `ds-*`, `hds-*`, `ui-*` prefixes, or project-specific patterns
- **No DS / bespoke**: generic class names, no recognisable system fingerprint

Also look for `<link>` tags pointing to a known design system CDN, or `<script>` tags
importing a DS bundle.

### 2A-5. Map extracted values to Figma reference

For each Figma design property recorded in Step 1, find the corresponding CSS value on the page.
Build a comparison table:

| Property | Figma value | Page value | Source | Match? |
|---|---|---|---|---|
| Hero background | `#1A1A2E` | `var(--color-brand-midnight)` → `#1A1A2E` | `.hero` rule | ✅ token |
| Hero padding (vertical) | `48px` | `py-12` → `48px` | Tailwind class | ✅ |
| CTA button bg | `var(--color-interactive-primary)` = `#2563EB` | `bg-blue-600` → `#2563EB` | Tailwind class | ✅ value matches, but uses Tailwind not token |
| Section gap | `24px` | `gap-[28px]` | Arbitrary Tailwind | ❌ 28px vs 24px |
| Heading font-size | `40px` | `2.25rem` = `36px` | CSS rule | ❌ 4px off |
| Card border-radius | `12px` | `8px` | CSS rule | ❌ |

This table becomes the evidence base for all findings. Do not write a finding without a row in
this table to back it up.

---

## Step 2B — Analyse the Codebase (Codebase mode)

_Skip this step if in URL mode — Step 2A covers live-page analysis._

Read every file shared. For each file, extract:

### Design system usage
Flag every import from a design system package. Categorise as:
- ✅ **Current stack** — imports from the new/target design system packages
- ⚠️ **Partial / override** — uses a current package but wraps it in heavy local overrides
- ❌ **Legacy / deprecated** — imports from old libraries that should have been replaced

### Layout constants
Find and list every hardcoded layout value:
- Arbitrary widths/heights: `[75rem]`, `[600px]`, `570px` etc.
- Hardcoded gaps, paddings, margins not from a token
- z-index magic numbers
- Placeholder/skeleton heights

### Token usage
- Which spacing/color/typography tokens are used correctly (e.g. `space.16`, `color.brand.primary`)
- Which values are raw CSS instead of tokens

### Component mapping
For each UI section visible in the design, identify:
- Which design-system component is used in code (if any)
- Whether that component is the correct/current one per the design system
- Whether a bespoke wrapper or override exists around it

### Responsive behaviour
- Is there a separate mobile container? (e.g. separate dweb/mweb files)
- Are mobile breakpoints handled via design-system primitives or bespoke CSS?

---

## Step 3 — Write the Audit Report

Follow the report schema below exactly, in order. Do not skip sections.
Use "N/A — [reason]" if a section genuinely does not apply.

**Output file naming:** `design-audit-[page-name]-[YYYY-MM-DD].md`

---

## Report Schema

```markdown
# [Page Name] Design Accuracy Audit

**Date:** YYYY-MM-DD
**Audit mode:** URL mode / Codebase mode
**Design reference:** Figma file `[File Name]`, [desktop/mobile] node `[node-id]`
**Page audited:** `[URL or route/path]`

---

## Scope

[2–4 sentences: which page, which sections, desktop vs mobile, what is in scope.
What is explicitly NOT in scope.]

What is in scope:
- [bullet list of specific areas checked]

What is not fully in scope:
- [bullet list — global nav, other routes, dynamic states not visible in static HTML, etc.]

---

## Figma MCP Status

[Describe what Figma data was successfully retrieved, what failed, and what was used instead.
"get_design_context succeeded; get_screenshot succeeded; get_variable_defs returned no variables"
is more useful than "Figma was partially available".]

---

## Data Source Status (URL mode only)

[Describe what was retrieved from the URL. Note:
- How many CSS files were fetched and their total approximate size
- Whether the page appeared to be server-rendered (HTML present) or JS-rendered (empty body)
- Whether design tokens (CSS custom properties) were found and how many
- Any fetch failures (blocked CSS, CORS, authentication walls)
If page required authentication and returned a login page, stop and report this clearly.]

---

## Overall Assessment

[2–4 sentence executive summary. State design alignment status clearly and directly.
Do not hedge with "seems mostly fine". Make a call: aligned / minor gaps / significant gaps /
not assessable due to data limitations.]

---

## Design vs Page: Property Comparison Table

| Property | Figma value | Page value | Source | Status |
|---|---|---|---|---|
| [property] | [figma] | [page] | [CSS rule / class / inline] | ✅ / ⚠️ / ❌ |

Status legend:
- ✅ Match — value and token usage both correct
- ⚠️ Value match but concern — correct value reached via hardcoded means (bypasses token system)
- ❌ Mismatch — value does not match Figma

---

## Token Usage Summary (URL mode)

| Token type | Token refs found | Hardcoded values found | Notes |
|---|---|---|---|
| Color | [N] | [N] | [e.g. "all hardcoded in hero section"] |
| Spacing | [N] | [N] | [e.g. "Tailwind defaults used, not custom tokens"] |
| Typography | [N] | [N] | |
| Border radius | [N] | [N] | |

---

## Design System Usage Summary (Codebase mode)

| Area | Status | Notes |
|---|---|---|
| [Section/component name] | ✅ Current / ⚠️ Partial / ❌ Legacy / 🔲 Bespoke | [one-line note] |

Status definitions:
- ✅ Current stack — uses the current design-system library with minimal local overrides
- ⚠️ Partial — uses current library but has non-trivial override styles
- ❌ Legacy — imports from a deprecated library that should have been replaced
- 🔲 Bespoke — entirely custom; no design-system component used where one should exist

---

## Migration Status by Area (Codebase mode)

### Clearly migrated
[Bullet list with file paths.]

### Partially migrated / still bespoke
[Bullet list. State specifically what is bespoke.]

### Not migrated / still on legacy stack
[Bullet list with file paths and the deprecated import found.
If empty: "No legacy library usage found in audited files."]

---

## Findings

[Numbered. Order: High severity first, then Medium, then Low.
Within the same severity, above-the-fold before below-the-fold.]

### [N]. [Finding title — specific, not vague]

**Severity:** High / Medium / Low
**Area:** [which section or component]
**Evidence source:** [CSS file URL / Tailwind class / inline style / source file path]

[2–5 sentences: what was found, why it matters, what the design specifies, what the page
shows instead. Be precise about values.]

**Evidence:**
[Paste the specific CSS rule, class value, or code pattern]

**Figma reference:** [The exact value the design specifies]
**Page value:** [The exact value found on the page]

**Recommended fix:** [1–2 sentence concrete remediation direction. For URL mode: what the
CSS or token should be changed to. For codebase mode: which file and what to change.]

---

## Concrete Flags

### High confidence issues
[Bullet list — issues you are certain about from design + page comparison.
Include specific values: "Hero padding is 32px on page vs 48px in Figma."]

### Not currently true
[Bullet list — assumptions that might be made about this page that the audit found to be WRONG.
Required section. State as quoted claims + why incorrect.
Example: "The page still uses the old color #FF0000 for CTAs" — found #2563EB, matching the
updated design spec.]

---

## Where the Page Still Needs Work

[Ordered list of specific, actionable remediation steps. Each item must be specific enough
that a developer can act on it without further clarification.]

1. [Action item — e.g. "Replace hardcoded `gap: 28px` in `.card-grid` with `var(--spacing-7)` (28px token) to match Figma's 28px gap spec."]
2. [Action item]

---

## Bottom Line

[3–5 sentences. Restate overall alignment. Identify the most impactful gap. State which
issues are blocking (visible to users) vs. cleanup (token hygiene). End with a clear
recommendation: ship as-is / fix critical gaps before launch / needs design-system work.]
```

---

## Severity Definitions

| Severity | Definition |
|---|---|
| **High** | Visible design deviation a user or designer would notice: wrong color, wrong spacing by more than 4px, wrong font size, wrong component used. |
| **Medium** | Correct visual outcome but achieved via hardcoded values instead of tokens, or a partial override that creates maintenance risk. Users probably don't notice it. |
| **Low** | Minor token inconsistency, arbitrary constant that could be a token, or cleanup opportunity. No visible impact. |

---

## What Makes a Good Finding (URL mode)

A finding must answer: **"What specific CSS rule or class on the live page does not match the
Figma design, and what are the exact values on each side?"**

If you cannot state both the Figma value and the page value with specifics, it is not a finding.

### ❌ Weak finding (URL mode)

> **Section padding looks off**
> The hero section appears to have less padding than the design.

Bad because: no CSS source cited, no Figma value, no page value, "appears to" is not audit language.

### ✅ Strong finding (URL mode)

> **2. Hero section vertical padding is 32px on the page vs 48px in Figma**
>
> **Severity:** High
> **Area:** Hero section
> **Evidence source:** `.hero-section` rule in `main.abc123.css`
>
> The hero section top and bottom padding is 32px on the live page, but the Figma frame specifies
> 48px (equivalent to `py-12` in Tailwind or `var(--spacing-12)`). This is a visible gap —
> the section will appear more cramped than designed, especially on desktop widths above 1200px.
>
> **Evidence:**
> ```css
> .hero-section { padding: 32px 24px; }
> ```
>
> **Figma reference:** 48px vertical padding
> **Page value:** 32px vertical padding (`padding-top` + `padding-bottom`)
>
> **Recommended fix:** Update `.hero-section` padding-top and padding-bottom to `48px` or
> `var(--spacing-12)` to match the Figma spec.

---

## What NOT to Write Findings For

1. **Things that match** — correct values belong in the Property Comparison Table with ✅
2. **Values you cannot verify** — if neither Figma nor the page gave you a concrete value, don't guess
3. **JavaScript-dynamic states** — hover, focus, open/closed states not visible in fetched HTML
4. **Things outside the audited section** — global nav, site footer, other pages

---

## How to Handle Missing Figma Data

**What you can still audit without Figma access:**

- Token vs. hardcoded value usage (from CSS custom properties and class names)
- Design system fingerprint (from class naming conventions and data attributes)
- Spacing and color consistency within the page (internal consistency, not Figma parity)
- Tailwind arbitrary value usage (flags like `px-[28px]`, `bg-[#1a2b3c]`)
- CSS custom property coverage (how much of the page uses `var(--*)` vs raw values)

**What you cannot audit without Figma access:**

- Whether a specific value (e.g. 24px padding) matches the design intent
- Color accuracy — you can describe what is there, but not whether it's correct
- Typography spec parity
- Spacing token correctness

When a finding is page-only due to missing Figma data, say so:
> **Note:** Figma `get_design_context` timed out. This finding is based on page CSS analysis only.
> The value `gap: 28px` is hardcoded — a designer should verify whether 28px matches the spec.

---

## URL Mode Limitations

Always note the following caveats where applicable:

1. **JavaScript-rendered content**: If the page is a React/Vue/Angular SPA and the HTML body
   was empty, the audit covers only what was present in the initial HTML shell. Interactive
   states, modals, and dynamic content are not audited.

2. **Computed styles vs. authored styles**: WebFetch retrieves authored CSS (what the developer
   wrote), not computed styles (what the browser resolves after cascade). Overrides from later
   rules or specificity battles may not be apparent. If a property appears in CSS but looks
   different in the browser, this could be the reason.

3. **Authentication-gated pages**: If the URL redirects to a login page, report this immediately
   and stop. Do not audit the login page as if it were the target.

4. **CSS-in-JS (styled-components, Emotion)**: Styles injected by CSS-in-JS libraries are not
   present in CSS files — they appear only in the browser runtime. If a page uses CSS-in-JS,
   this will look like an unstyled page with no meaningful CSS. Report this and switch to
   Codebase mode if possible.

5. **Design tokens not declared in CSS**: If tokens are defined in a JS theme object (e.g.
   Chakra UI, MUI ThemeProvider) rather than CSS custom properties, they will not appear in
   the fetched CSS. Note this limitation and focus on what is auditable.

---

## Quality Rules

1. Every finding must cite both a Figma value and a page value. One without the other is not
   a finding — it is a hypothesis.

2. In URL mode, always note whether a matched value is via token or hardcoded. Token match
   is ✅; value-only match (correct number, wrong method) is ⚠️.

3. Distinguish visual gaps (High) from token hygiene issues (Medium/Low). They have different
   remediation paths and urgency.

4. Always note when Figma data was unavailable and what you used instead.

5. "Not currently true" claims are as important as findings. Explicitly state what the audit
   did NOT find — this prevents false assumptions from spreading.

6. If the fetched CSS is minified, say so. Extract what you can; note what was unreadable.

7. Mobile and desktop are separate audit surfaces. If the Figma frame covers mobile, check
   for `@media` breakpoints in CSS or a separate mobile URL.

8. In codebase mode: never claim a page "uses the old design system" unless you found an
   actual import from the deprecated library. Visual impression is not evidence.
