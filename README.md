# de-research

A personal research repository for data engineering topics, structured for AI-assisted investigation.

AI agents (Claude, Codex, Antigravity) can autonomously download resources, generate summaries, and produce research reports — no confirmation needed for fetching content.

---

## Structure

```
resources/
  {component}/
    {resource-title}            ← raw downloaded content (html, pdf, or folder for repos)
    {resource-title}.summary.md ← agent-generated summary with security analysis

topics/
  {topic-title}.report.md       ← research report spanning one or more components
```

Names use **kebab-case** throughout. Components are free-form (e.g. `kafka`, `spark`, `dbt`, `airflow`, `flink`).

---

## How It Works

**Resources** are source material — articles, PDFs, GitHub repos — downloaded under a component folder. Each resource gets a `.summary.md` with key points and a mandatory security scan (malicious code, suspicious URLs, content quality).

**Topic reports** synthesize findings across components. They use YAML frontmatter to tag which components are involved and link out to summary files as references.

---

## AI Agent Instructions

| Agent | File |
|---|---|
| Claude Code | `CLAUDE.md` |
| OpenAI Codex CLI | `AGENTS.md` |
| Google Antigravity / Gemini CLI | `GEMINI.md` + `AGENTS.md` |
