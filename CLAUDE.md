# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is for data engineering research. Topics are investigated across multiple components (e.g. kafka, spark, dbt, airflow, flink), with downloaded resources stored and summarized locally.

---

## Repository Structure

```
resources/
  {component}/
    {resource-title}.summary.md   ← required for every resource
    {resource-title}.pdf          ← save raw file only for PDFs and code; skip HTML
topics/
  {topic-title}.report.md
```

- All file and folder names use **kebab-case**.
- `{component}` is free-form (e.g. `kafka`, `spark`, `dbt`). Create the folder if it does not exist.
- `{resource-title}` is a short, descriptive, kebab-case name.
- **Do not save raw HTML files** — they add no value over a summary. Save raw files only for PDFs and code/repos.

---

## Research Workflow

Follow these phases in order. Phases 1–2 are planning; do not fetch or write files until Phase 3.

### Phase 1 — Scope and disambiguate

Before any searches or fetches:

1. **Identify all components** involved in the topic. Create a working list.
2. **Disambiguate component names.** If a name could match multiple products (e.g. "Weka" = Java ML tool AND weka.io storage), resolve it first with a targeted search before collecting resources.
3. **Classify each provided source** by type:
   - `vendor` — published by the company selling the product
   - `independent` — analyst, trade press, or practitioner with no financial stake
   - `academic` — peer-reviewed or university research
   - `official` — project/foundation documentation (e.g. spark.apache.org)
4. **Plan three search angles** for every topic:
   - (a) Neutral overview — what it is and how it works
   - (b) Limitations and critiques — search terms: `limitations`, `cons`, `alternatives`, `issues`, `problems`
   - (c) Competitive comparison — search terms: `vs`, `compared to`, `alternative to`, `benchmark`

### Phase 2 — Parallel resource collection

Fire all searches simultaneously. Do not fetch pages one at a time sequentially.

Source priority (highest to lowest):
1. Official project documentation
2. Independent analyst reports (Gartner, IDC) and peer-reviewed papers
3. Trade press (The New Stack, Blocks & Files, HPCwire, SiliconANGLE)
4. Practitioner blogs with methodology shown
5. Vendor marketing and whitepapers

For every vendor source provided, find at least one independent source covering the same claim.

### Phase 3 — Content retrieval

- Fetch resources in parallel where possible.
- **If a page returns 403 or is behind a form/paywall**: do not abandon it. Instead:
  1. Search for `"{resource title}" summary` or `"{author} {topic}"` to find secondary coverage.
  2. Note in the summary that content was not directly accessed and mark `accessible: false`.
- **If a PDF is gated**: note the claims described on the landing page, mark as indirect, and search for independent analyses of the same document.
- Perform the security scan during this phase.

### Phase 4 — Write summaries

Write one `.summary.md` per resource immediately after retrieval. See template below.

### Phase 5 — Write the topic report

Synthesize across all summaries. See template below. Lead with the verdict — do not bury it in findings.

---

## Summary File Template

```markdown
---
source: <URL or origin>
component: <component>
type: article | pdf | github-repo
evidence-tier: vendor | independent | academic | official
accessible: true | false (gated/403 — content derived from secondary sources)
date-retrieved: YYYY-MM-DD
security:
  malicious-code: none | flagged
  suspicious-urls: none | flagged
  quality: high | low | ai-generated
---

## Summary

<2–5 sentence summary. If not directly accessible, state that explicitly.>

## Key Points

- ...

## Security Notes

<Write "No issues detected." if clean. Otherwise describe findings.>
Checks performed:
- Malicious or obfuscated code: ...
- Suspicious URLs or redirects: ...
- Content quality / AI-generated: ...
```

### Security checks (required for every summary)

- **Malicious code**: obfuscated scripts, encoded eval/exec, exfiltration patterns, unusual payloads.
- **Suspicious URLs**: uncommon domains, URL shorteners, redirect chains.
- **Quality**: flag AI-generated, padded, or low-signal content.

---

## Topic Report Template

```markdown
---
title: <Topic Title>
date: YYYY-MM-DD
status: draft | in-progress | complete
components: [component-a, component-b]
---

## Overview

<What this topic is about and why it matters for data engineering.>

**Verdict: <one or two sentence conclusion here — do not bury it.>**

## Evidence Quality

| Source | Type | Tier |
|---|---|---|
| Resource A | article | independent |
| Resource B | pdf | vendor |

<Note any gaps: missing independent benchmarks, gated sources, vendor-only coverage.>

## Components

### `component-a`

<Role of this component in the topic. Key concepts, constraints, and caveats.>

### `component-b`

<Role of this component. Key concepts.>

## Findings

<Synthesized conclusions: comparisons, trade-offs, practical implications.>
<For each major finding, state what evidence supports it and its confidence level.>

## References

- [Resource Title](../resources/component-a/resource-title.summary.md)
- [Resource Title](../resources/component-b/resource-title.summary.md)
```

Rules:
- The `components` frontmatter field lists every component tag for the topic.
- References link to `.summary.md` files, not raw downloads.
- The verdict must appear in the **Overview** section, not only in Findings.
- If vendor sources dominate the evidence base, state this as a limitation explicitly in Evidence Quality.
