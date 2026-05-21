# GEMINI.md

Agent instructions for this repository (Gemini CLI and Antigravity CLI). Antigravity reads this file as its highest-priority override layer on top of AGENTS.md.

## Purpose

Data engineering research repository. Topics are investigated across multiple components (e.g. kafka, spark, dbt, airflow, flink), with downloaded resources stored and summarized locally.

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
- `{component}` is free-form. Create the folder if it does not exist.
- `{resource-title}` is a short, descriptive, kebab-case name.
- **Do not save raw HTML files.** Save raw files only for PDFs and code/repos.

---

## Research Workflow

Follow these phases in order. Do not fetch or write files until Phase 3.

### Phase 1 — Scope and disambiguate

1. **Identify all components** involved in the topic.
2. **Disambiguate component names** — if a name could match multiple products, resolve it with a targeted search first.
3. **Classify each provided source** by evidence tier:
   - `vendor` — published by the company selling the product
   - `independent` — analyst, trade press, or practitioner with no financial stake
   - `academic` — peer-reviewed or university research
   - `official` — project/foundation documentation
4. **Plan three search angles**:
   - (a) Neutral overview
   - (b) Limitations/critiques — use terms: `limitations`, `cons`, `alternatives`, `issues`
   - (c) Competitive comparison — use terms: `vs`, `compared to`, `benchmark`

### Phase 2 — Parallel resource collection

Fire all searches simultaneously. Source priority:
1. Official project documentation
2. Independent analyst reports and peer-reviewed papers
3. Trade press (The New Stack, Blocks & Files, HPCwire)
4. Practitioner blogs with methodology shown
5. Vendor marketing and whitepapers

For every vendor source, find at least one independent source covering the same claim.

### Phase 3 — Content retrieval

- Fetch resources in parallel.
- **If 403 or gated**: search for secondary coverage of the same document. Note `accessible: false` in summary.
- **If a PDF is gated**: note landing page claims, mark as indirect, search for independent analyses.
- Perform security scan during this phase.

### Phase 4 — Write summaries

One `.summary.md` per resource immediately after retrieval.

### Phase 5 — Write the topic report

Lead with the verdict in Overview. Do not bury conclusions in Findings.

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

<2–5 sentence summary. State explicitly if not directly accessible.>

## Key Points

- ...

## Security Notes

<"No issues detected." if clean, otherwise describe findings.>
Checks performed:
- Malicious or obfuscated code: ...
- Suspicious URLs or redirects: ...
- Content quality / AI-generated: ...
```

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

**Verdict: <one or two sentence conclusion — required here, not only in Findings.>**

## Evidence Quality

| Source | Type | Tier |
|---|---|---|
| Resource A | article | independent |
| Resource B | pdf | vendor |

<Note any gaps: missing independent benchmarks, gated sources, vendor-only coverage.>

## Components

### `component-a`

<Role, key concepts, constraints.>

### `component-b`

<Role, key concepts.>

## Findings

<Synthesized conclusions with trade-offs. State the evidence and confidence for each major finding.>

## References

- [Resource Title](../resources/component-a/resource-title.summary.md)
- [Resource Title](../resources/component-b/resource-title.summary.md)
```
