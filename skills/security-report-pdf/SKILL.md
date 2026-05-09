---
name: security-report-pdf
description: >-
  Convert security review findings into a professional pentest report Markdown file and render it
  to PDF using Pandoc, XeLaTeX, and the Eisvogel LaTeX template. Use when: generating a security
  assessment report, exporting penetration test findings to PDF, formatting vulnerability findings
  into a pentest report, producing a CVSS-scored findings document, creating a security audit
  report with a table of contents and numbered sections.
argument-hint: "Optional: path to save the .md report (default: report.md)"
---

# Security Report PDF

Converts security review findings into a professionally formatted pentest report Markdown file,
then renders it to PDF using Pandoc + XeLaTeX + the Eisvogel template.

This skill also produces a Snyk dependency report (JSON + HTML + Markdown + PDF).

## When to Use

- After running a security scan or receiving findings from the Security Review Agent
- When the user asks to "generate a report", "export findings to PDF", or "write up the security review"
- When findings need to be formatted consistently with severity ratings, CVSS, evidence, and remediation
- When the user wants both a pentest report and a Snyk report in one run

## Output Location (Required)

Always write artifacts to `./vulnerabilities`.

- Create it if it does not exist.
- Do not leave final report artifacts in the workspace root.
- Default output set:
  - `vulnerabilities/pentest-report.md`
  - `vulnerabilities/pentest-report.pdf`
  - `vulnerabilities/snyk-report.json`
  - `vulnerabilities/snyk-report.html`
  - `vulnerabilities/snyk-report.md`
  - `vulnerabilities/snyk-report.pdf`

## Prerequisites

Verify these tools are available before rendering:

```bash
pandoc --version
xelatex --version          # usually: tlmgr install collection-xetex
ls ./templates/eisvogel.latex  # download from https://github.com/Wandmalfarbe/pandoc-latex-template
snyk --version
snyk-to-html --version
```

If `eisvogel.latex` is missing, download it:
```bash
mkdir -p templates
curl -Lo templates/eisvogel.latex \
  https://raw.githubusercontent.com/Wandmalfarbe/pandoc-latex-template/master/eisvogel.latex
```

## Procedure

### Step 1 — Collect Findings

Read the security findings from:
- The current conversation
- The output of the Security Review Agent
- Any existing notes or scan results in the workspace

Then run Snyk and capture dependency findings.

Primary command requested by users:

```bash
snyk test --json | snyk-to-html -o vulnerabilities/snyk-report.html
```

For monorepos/workspaces, run:

```bash
snyk test --all-projects --json > vulnerabilities/snyk-report.json
snyk-to-html -i vulnerabilities/snyk-report.json -o vulnerabilities/snyk-report.html
```

If Snyk is not installed globally, use:

```bash
pnpm dlx snyk test --all-projects --json > vulnerabilities/snyk-report.json
pnpm dlx snyk-to-html -i vulnerabilities/snyk-report.json -o vulnerabilities/snyk-report.html
```

Identify for each finding:
- Title and numbered identifier (e.g. `1`, `2`, `3`)
- Severity: **Critical / High / Medium / Low / Informational**
- Render severity labels using the shared LaTeX macros from frontmatter:
  - `\\sevcritical` = purple
  - `\\sevhigh` = red
  - `\\sevmedium` = yellow
  - `\\sevlow` = blue
  - `\\sevinfo` = gray
- CVSS score (if calculable or provided)
- Affected asset (file path, endpoint, component)
- Description of the vulnerability
- Evidence (code snippet, log line, request/response)
- Impact (what an attacker can do)
- Remediation (specific fix, not generic advice)
- Retest status: `Open / Remediated / Accepted Risk`

### Step 2 — Build the Markdown Report

Create the report file with the YAML frontmatter from [./assets/frontmatter-template.yaml](./assets/frontmatter-template.yaml).

Structure the document:

```
1. Executive Summary
2. Scope and Methodology
3. Risk Rating Summary (table)
4. Findings
  4.1  Finding 1: <Title>
  4.2  Finding 2: <Title>
   ...
5. Remediation Roadmap
6. Appendices (if needed)
```

**Finding block template** (repeat for each finding):

```markdown
## Finding 1: Title Here {#finding-1}

| Field            | Value                      |
|------------------|----------------------------|
| **Severity**     | \sevhigh                   |
| **CVSS Score**   | 8.1 (AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:N) |
| **Affected Asset** | `apps/api/src/routes/rewards.ts:214` |
| **Retest Status** | Open                      |

### Description

Clear explanation of the vulnerability class and why it exists in this codebase.

### Evidence

```ts
// Paste the exact vulnerable code or request/response here
rewardsRouter.post("/:id/request", async (req, res) => {
  // no auth middleware
});
```

### Impact

What an attacker can do if this is exploited. Be specific to the application context.

### Remediation

Concrete code-level fix. Reference the exact file and line. Prefer showing a before/after diff.

```ts
// Before
rewardsRouter.post("/:id/request", async (req, res) => {

// After
rewardsRouter.post("/:id/request", requireMember, async (req, res) => {
```
```

Use a unique anchor for every finding heading: `{#finding-<number>}`.

**Risk Rating Summary table** (place after Executive Summary):

```markdown
| Number | Title                              | Severity | Status |
|--------|------------------------------------|----------|--------|
| [1](#finding-1) | Unauthenticated reward request | \sevhigh | Open   |
| [2](#finding-2) | SSRF in gallery from-url       | \sevcritical | Open   |
```

**Severity color convention for Eisvogel** (already defined in frontmatter):
- Critical: purple
- High: red
- Medium: yellow
- Low: blue
- Informational: gray

**Code block style (required):**
- Use a dark background for code blocks in PDF output.
- Frontmatter `header-includes` must define `lstlisting` colors and `\\lstset`.
- Use this palette:
  - Background: `#111827`
  - Foreground: `#E5E7EB`
  - Keywords: `#93C5FD`
  - Strings: `#FDE68A`
  - Comments: `#9CA3AF`

### Step 3 — Save the Markdown File

Save pentest markdown to `vulnerabilities/pentest-report.md` unless the user specifies a different
path under `vulnerabilities/`.

Verify:
- YAML frontmatter is valid (no unescaped colons, no tabs)
- All code fences are properly closed
- Image paths are relative (e.g. `./images/screenshot.png`)
- No HTML tags that XeLaTeX cannot process (`<br>` → blank line, `<b>` → `**`)
- Code blocks render with dark-mode background and readable text contrast

### Step 4 — Render to PDF

Run the Pandoc command:

```bash
pandoc vulnerabilities/pentest-report.md \
  -o vulnerabilities/pentest-report.pdf \
  --pdf-engine=xelatex \
  --template=./templates/eisvogel.latex \
  --table-of-contents \
  --toc-depth=3 \
  --number-sections \
  --syntax-highlighting=zenburn \
  --resource-path=".:./images:./vulnerabilities"
```

### Step 5 — Build Snyk Markdown and PDF

Create `vulnerabilities/snyk-report.md` from `vulnerabilities/snyk-report.json` with:

- Scan summary: projects, total vulnerabilities, severity counts
- Per-project summary table
- Detailed findings table (top issues)

Render Snyk PDF:

```bash
pandoc vulnerabilities/snyk-report.md \
  -o vulnerabilities/snyk-report.pdf \
  --pdf-engine=xelatex \
  --template=./templates/eisvogel.latex \
  --table-of-contents \
  --toc-depth=3 \
  --number-sections \
  --syntax-highlighting=zenburn \
  --resource-path=".:./images:./vulnerabilities"
```

If the render fails:
- **Missing package**: Run `tlmgr install <package>` for the named LaTeX package
- **Unicode errors**: Add `--pdf-engine-opt=-shell-escape` or replace problematic characters
- **Long code lines overflowing**: Add `listings: true` to YAML frontmatter
- **Template not found**: Ensure `./templates/eisvogel.latex` exists relative to the working directory

### Step 6 — Verify Output

Open or confirm outputs contain:
- Title page with date, author, subtitle
- Table of contents on its own page
- Numbered sections (1, 2, 3...)
- Syntax-highlighted code blocks
- All findings present with no truncation
- Two final PDFs exist in `vulnerabilities/`:
  - `pentest-report.pdf`
  - `snyk-report.pdf`

## Quality Checklist

- [ ] `vulnerabilities/` exists and contains all generated artifacts
- [ ] Both reports were generated (pentest + Snyk)
- [ ] Snyk JSON and HTML artifacts were generated
- [ ] Every finding has Severity, Affected Asset, Evidence, Impact, Remediation
- [ ] Risk summary number column links to each finding anchor (`#finding-<n>`)
- [ ] Risk rating summary table is present and accurate
- [ ] Executive Summary is 1–3 paragraphs (non-technical audience)
- [ ] YAML frontmatter date is set (use `\today` for auto-date)
- [ ] No broken fenced code blocks (every ` ``` ` has a matching close)
- [ ] PDF renders without errors
- [ ] TOC page numbers are correct


## Windows PowerShell Pandoc Command

Use semicolons in `--resource-path` on Windows.

pandoc "input.md" `
  -o "output.pdf" `
  --pdf-engine=xelatex `
  --template=".\templates\eisvogel.latex" `
  --table-of-contents `
  --toc-depth 6 `
  --number-sections `
  --syntax-highlighting=kate `
  --resource-path=".;.\images"
