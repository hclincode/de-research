# AGENTS.md

Agent instructions for this repository (OpenAI Codex CLI, Google Antigravity CLI, and any other agent that reads AGENTS.md).

## Purpose

Data engineering research repository. Topics are investigated across multiple components (e.g. kafka, spark, dbt, airflow, flink), with downloaded resources stored and summarized locally.

---

## Repository Structure

```
resources/
  {component}/
    {resource-title}.summary.md   ← required for every resource (one URL per file)
    {resource-title}.pdf          ← save raw file only for PDFs and code; skip HTML
topics/
  {topic-title}.report.md
```

- All file and folder names use **kebab-case**.
- `{component}` is free-form. Create the folder if it does not exist.
- **One URL per summary file.** Never aggregate multiple sources into one summary.
- **Do not save raw HTML files.**

---

## Research Workflow

Follow these phases in order. Do not fetch or write files until Phase 3.

### Phase 1 — Scope and disambiguate

1. **Identify all components** in the topic.
2. **Disambiguate names** that could match multiple products. After finding sources, verify they are about the right product before using them.
3. **Classify each provided source** using the five evidence tiers (see below).
4. **List expected conclusions.** You will search against these in Phase 2.
5. **Apply constraint filters.** Eliminate non-qualifying candidates immediately and document why.

### Phase 1.5 — Clarifying questions (5-minute timeout)

After scoping, identify questions whose answers would materially change the research direction. Ask at most **3–5 questions**. Do not ask about anything already in the request.

High-value question categories: scope layer (storage service vs table format vs full stack), deployment context (greenfield vs migration), workload profile (batch / streaming / interactive), performance priority, existing constraints.

**Procedure:**
1. State each question as plain text with an explicit default, e.g. _"Q: Which layer? Default if no reply: full stack."_
2. Wait approximately 2 minutes. If the user replies, use their answers. If not, proceed with the stated defaults.
3. Document any defaults used in the report's Evidence Quality section under "User-default assumptions."

**Skip Phase 1.5** if the request already specifies scope layer, deployment context, and workload profile.

### Phase 2 — Plan and collect in parallel

Four search angles, fired simultaneously:

- **(a) Neutral overview**
- **(b) Limitations/critiques** — terms: `limitations`, `cons`, `alternatives`, `issues`, `problems`, `criticism`
- **(c) Competitive comparison** — terms: `vs`, `compared to`, `benchmark`
- **(d) Counter-evidence** — for each expected conclusion, search `"why [X] is wrong"`, `"[X] failure"`, `"[X] not recommended"`. **Mandatory.**

Source priority: official → analyst → press → vendor-adjacent → vendor.
For every vendor/vendor-adjacent source, find at least one independent corroboration.

**Source balance check** (before Phase 3): For the primary recommendation, confirm at least 2 non-vendor sources are planned. If not, add a targeted search.

**Staleness gap rule**: If the only benchmark for a key claim is >2 years old, run one targeted recency search before Phase 3. Document whether recent data was found.

**Scope lock** (write before Phase 3): One sentence — what this research covers and does not cover. Use the Phase 1.5 answer if provided; otherwise derive from the request.

### Phase 3 — Content retrieval

- Fetch in parallel.
- **If 403 or gated:** find secondary coverage, mark `accessible: false`.
- **Flag benchmarks >2 years old** explicitly in the summary.
- Run security analysis.

### Phase 4 — Write summaries

One `.summary.md` per URL immediately after retrieval.

### Phase 5 — Write the topic report

**Pre-write checklist:**
- [ ] At least 1 official or press source for each recommended component.
- [ ] Every eliminated option has a sourced reason.
- [ ] All LOW-confidence findings call out what would raise confidence.
- [ ] Stale benchmarks (>2 years) are flagged with a note that no recent data was found.
- [ ] Phase 1.5 defaults (if used) documented in Evidence Quality.

- Verdict in **Overview**.
- Each finding: **inline citations** + **confidence level** (HIGH / MEDIUM / LOW).
- Findings on vendor/vendor-adjacent sources only: label LOW confidence.
- Evidence Quality must name uncorroborated conclusions and stale benchmarks.

---

## Evidence Tiers

| Tier | Definition |
|---|---|
| `official` | Published by the project or standards body |
| `analyst` | Financially neutral analyst or peer-reviewed academic |
| `press` | Independent trade press with no financial stake |
| `vendor-adjacent` | Commercial company with stake in a related/competing product; technically neutral in presentation |
| `vendor` | Published by the company selling the evaluated product |

**`vendor-adjacent` is not independent.** Examples: Red Hat on Ceph, Cloudera on Iceberg+Ozone, Dremio on Nessie, Onehouse on Hudi.

---

## Summary File Template

```markdown
---
source: <single URL>
component: <component>
type: article | pdf | github-repo
evidence-tier: official | analyst | press | vendor-adjacent | vendor
accessible: true | false
benchmark-age: YYYY | n/a
date-retrieved: YYYY-MM-DD
security:
  malicious-code: none | flagged
  suspicious-urls: none | flagged
  quality: high | low | ai-generated
---

## Summary

<2–5 sentences. Note if inaccessible or benchmark is stale (>2 years old).>

## Key Points

- ...

## Security Notes

Checks performed:
- Malicious or obfuscated code: ...
- Suspicious URLs or redirects: ...
- Content quality / AI-generated: ...
```

---

## Topic Report Template

```markdown
---
title: <Title>
date: YYYY-MM-DD
status: draft | in-progress | complete
components: [component-a]
constraints:
  - open-source: true
  - deployment: on-premise
---

## Overview

<Context.>

**Verdict: <conclusion — required here.>**

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|

**Gaps and confidence limits:** <uncorroborated claims, stale benchmarks, inaccessible sources.>

## Components

### `component-a`

<Role, concepts, caveats. Cite inline: [Source](../resources/...).>

## Findings

### N. Finding title

<Evidence: [Source A](../resources/...). Confidence: HIGH | MEDIUM | LOW.>

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|

## References

- [Title](../resources/component/file.summary.md)
```

**Confidence:** HIGH = ≥2 independent sources, no contradiction. MEDIUM = 1 independent source or multiple vendor-adjacent agree. LOW = vendor/vendor-adjacent only, inaccessible primary source, or benchmark >2 years old.
