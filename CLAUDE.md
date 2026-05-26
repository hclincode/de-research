# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This repository is for data engineering research. Topics are investigated across multiple components (e.g. kafka, spark, dbt, airflow, flink), with downloaded resources stored and summarized locally.

---

## Repository Structure

```
resources/
  {component}/
    {resource-title}.summary.md   ← required for every resource (one URL per file)
    {resource-title}.pdf          ← save raw file only for PDFs and code; skip HTML
topics/
  {YYYYMMDD}-{topic-title}.report.md   ← date prefix is the file's last-modified date
```

- All file and folder names use **kebab-case**.
- `{component}` is free-form (e.g. `kafka`, `spark`, `dbt`). Create the folder if it does not exist.
- `{resource-title}` is a short, descriptive, kebab-case name.
- **One URL per summary file.** Do not aggregate multiple sources into one summary — create separate files. Multiple-source summaries make claims untraceable.
- **Do not save raw HTML files.** Save raw files only for PDFs and code/repos.

---

## Research Workflow

Follow these phases in order. Do not fetch or write files until Phase 3.

### Phase 1 — Scope and disambiguate

Before any searches or fetches:

1. **Identify all components** in the topic. Create a working list.
2. **Disambiguate names.** If a name matches multiple products, resolve with a targeted search first. After finding sources, verify they are actually about the right product (e.g. "Weka" is both a Java ML toolkit and a storage vendor — confirm before using).
3. **Classify each provided source** using the five evidence tiers (see below). Do this before fetching — bias is easier to spot when you know who published the source.
4. **List expected conclusions.** Write down what answer you expect to reach. You will explicitly search against these in Phase 2.
5. **Apply constraint filters.** If constraints exist (e.g. open-source only, on-premise only), eliminate non-qualifying candidates immediately and document why.

### Phase 1.5 — Clarifying questions (5-minute timeout)

After scoping, identify questions whose answers would materially change the research direction. Ask at most **3–5 questions**. Do not ask about anything already specified in the request.

**High-value question categories:**
- **Scope layer**: Which layer to focus on — storage service (Ceph, Ozone, MinIO), table format (Iceberg, Hudi, Delta), catalog, or full stack?
- **Deployment context**: New greenfield deployment, or migrating existing infrastructure?
- **Workload profile**: Batch-heavy, CDC/streaming, interactive analytics, or mixed?
- **Performance priority**: Throughput, latency, storage-cost efficiency, or operational simplicity?
- **Existing constraints**: Components already mandated, ruled out, or already deployed?

**Procedure (Claude Code):**
1. State each question as plain text with an explicit default: e.g. _"Q: Which layer should this cover? Default if no reply: full stack."_
2. In the same response, call `ScheduleWakeup(300s)` with the prompt: `"Clarifying questions were asked. Proceed with stated defaults: [list each default]."` This fires in 5 minutes if the user does not respond.
3. If the user replies before the wakeup fires: use their answers. When the wakeup fires, check the conversation — if the user responded, proceed with their answers; otherwise proceed with the defaults.
4. If no user response by wakeup: proceed with the stated defaults and document them in the report's Evidence Quality section under "User-default assumptions."

**Skip Phase 1.5** if the request already specifies all three: scope layer, deployment context, and workload profile.

### Phase 2 — Plan and collect in parallel

Plan four search angles, then fire all searches simultaneously:

- **(a) Neutral overview** — what it is and how it works
- **(b) Limitations and critiques** — terms: `limitations`, `cons`, `alternatives`, `issues`, `problems`, `criticism`
- **(c) Competitive comparison** — terms: `vs`, `compared to`, `benchmark`, `alternative to`
- **(d) Counter-evidence** — for each expected conclusion from Phase 1, search `"why [X] is wrong"` or `"[X] failure"` or `"[X] not recommended"`. This step is mandatory.

Source priority:
1. Official project/foundation documentation
2. Financially neutral analysts (Gartner, IDC) and peer-reviewed papers
3. Independent trade press (The New Stack, InfoQ, HPCwire, Blocks & Files)
4. Practitioner blogs with disclosed methodology
5. Vendor-adjacent sources (companies that contribute to or commercially support the evaluated product)
6. Vendor marketing (published by the company selling the product)

For every vendor or vendor-adjacent source, find at least one independent or official source covering the same claim before treating the claim as established.

**Source balance check** (mandatory before moving to Phase 3): For the primary recommendation, confirm at least 2 non-vendor/non-vendor-adjacent sources are in your plan. If not, add a targeted search for independent coverage of that claim before proceeding.

**Staleness gap rule**: If the only benchmark found for a key performance claim is >2 years old, run one additional search — `"[component] benchmark [current year]"` or `"[component] performance [current year-1]"` — before Phase 3. Document whether recent data was found or not.

**Scope lock** (write before Phase 3): State in one sentence what this research covers and what it does not. Example: _"This research covers physical storage services (Ozone, Ceph, HDFS) and does not cover table formats or catalogs."_ Use the user's Phase 1.5 answer if provided; otherwise derive from the request. This prevents scope drift during retrieval.

### Phase 3 — Content retrieval

- Fetch resources in parallel.
- **If 403 or gated:** search for secondary coverage of the same document. Mark `accessible: false`. Do not silently drop the source.
- **Check benchmark age.** If a benchmark is >2 years old, note it explicitly in the summary.
- Perform security analysis during this phase.

### Phase 4 — Write summaries

One `.summary.md` per URL, immediately after retrieval. See template below.

### Phase 5 — Write the topic report

**Pre-write checklist** (verify before writing the report):
- [ ] At least 1 official or press source exists for each recommended component.
- [ ] Every eliminated option has a stated reason backed by at least one source.
- [ ] All LOW-confidence findings call out the gap and note what would be needed to raise confidence.
- [ ] Any benchmark cited is either ≤2 years old, or is explicitly flagged as stale with a note that no recent data was found.
- [ ] If Phase 1.5 defaults were used (no user response), document them in Evidence Quality under "User-default assumptions."

Synthesize across summaries. Rules:
- Verdict must appear in **Overview**, not buried in Findings.
- Each finding must include **inline citations** to specific summary files and a **confidence level** (HIGH / MEDIUM / LOW).
- Findings that rest only on vendor or vendor-adjacent sources must be labelled LOW confidence regardless of content quality.
- The Evidence Quality section must note gaps and name any expected conclusions that could not be independently corroborated.

---

## Evidence Tiers

Use exactly these five tiers — do not invent others:

| Tier | Definition | Examples |
|---|---|---|
| `official` | Published by the project or standards body itself | spark.apache.org, ozone.apache.org, ASF announcements |
| `analyst` | Financially neutral third-party analyst or peer-reviewed academic | Gartner, IDC, ACM/IEEE papers, university research |
| `press` | Independent trade press with no financial stake in the outcome | InfoQ, The New Stack, HPCwire, Blocks & Files, SiliconANGLE |
| `vendor-adjacent` | Commercial company with a stake in a related or competing product; technically neutral in presentation | Red Hat (Ceph contributor) on Ceph, Cloudera (sells CDP) on Iceberg+Ozone, Dremio (Nessie backer) on Nessie, Onehouse (Hudi backer) on lakehouse formats |
| `vendor` | Published by the company selling the evaluated product | weka.io on Weka, databricks.com on Delta Lake, hudi.apache.org on Hudi |

**Key rule:** `vendor-adjacent` is not `independent`. A company that contributes to, commercially supports, or competes against an evaluated product has a stake in the outcome. Label it `vendor-adjacent` and find corroboration.

---

## Summary File Template

```markdown
---
source: <single URL>
component: <component>
type: article | pdf | github-repo
evidence-tier: official | analyst | press | vendor-adjacent | vendor
accessible: true | false (gated/403 — content derived from secondary sources)
benchmark-age: YYYY | n/a          ← year of the benchmark data, not the publication date
date-retrieved: YYYY-MM-DD
security:
  malicious-code: none | flagged
  suspicious-urls: none | flagged
  quality: high | low | ai-generated
---

## Summary

<2–5 sentences. If not directly accessible, state that explicitly. If benchmark data is stale (>2 years old relative to the research date), state that.>

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
constraints:                        ← list hard constraints that eliminate candidates
  - open-source: true
  - deployment: on-premise
clarification-defaults:             ← omit if user answered Phase 1.5 questions; include if defaults were used
  scope-layer: full stack           ← example default
  deployment-context: greenfield    ← example default
---

## Overview

<What this topic is about and why it matters.>

**Verdict: <one or two sentence conclusion — required here.>**

## Evidence Quality

| Source | File | Tier | Accessible |
|---|---|---|---|
| Source title | [link](../resources/...) | press | Yes |

**Gaps and confidence limits:** <List: (1) claims that rest only on vendor/vendor-adjacent sources, (2) benchmarks that are stale, (3) expected conclusions that could not be independently corroborated, (4) sources that were inaccessible.>

## Components

### `component-a`

<Role, key concepts, constraints, caveats. Cite inline: [Source](../resources/...).>

## Findings

### N. Finding title

<Evidence: [Source A](../resources/...), [Source B](../resources/...). Confidence: HIGH | MEDIUM | LOW.>

<Finding body. If confidence is LOW or MEDIUM, state why.>

## Decision Matrix

| Profile | Recommendation | Eliminated options and why |
|---|---|---|

## References

- [Resource Title](../resources/component-a/resource-title.summary.md)
```

**Confidence levels:**
- **HIGH**: ≥2 independent sources (official, analyst, or press) corroborate the claim. No contradicting evidence found.
- **MEDIUM**: 1 independent source, or multiple vendor-adjacent sources agree. Contradicting evidence is minor or unverified.
- **LOW**: Evidence rests only on vendor or vendor-adjacent sources, or the primary source was inaccessible, or benchmark data is >2 years old with no recent corroboration.
