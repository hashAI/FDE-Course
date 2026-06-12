# FDE Course

Forward Deployment Engineer curriculum — technical depth for integration work, plus interview and client-facing skills.

**Bookmark this page:** [github.com/hashAI/FDE-Course](https://github.com/hashAI/FDE-Course)

---

## Reading on mobile

Open the repo link in the **GitHub mobile app** or mobile browser. Tap any module below — GitHub renders markdown with readable typography. Avoid the **Raw** button.

| Approach | Mobile experience | Setup |
|----------|-------------------|-------|
| **This README (recommended now)** | Tap-to-navigate index on repo homepage | None |
| **GitHub Pages + MkDocs** | Sidebar, search, best long-form reading | ~30 min one-time setup |
| **Single merged file** | One scroll, no file switching | Hard to maintain |

> **Tip:** Long modules (1,500+ lines) are easier to skim on mobile if you use GitHub's heading outline — tap the **⋯** menu (browser) or scroll to section headers.

---

## Part 1 — Technical

### [A — Python](Part1-Technical/A-Python/)

| # | Module |
|---|--------|
| 01 | [Async/Await in Python](Part1-Technical/A-Python/01-async-await.md) |
| 02 | [Pydantic v2 & Type Hints](Part1-Technical/A-Python/02-pydantic-type-hints.md) |
| 03 | [pytest for Integration Code](Part1-Technical/A-Python/03-pytest-testing.md) |
| 04 | [Error Handling](Part1-Technical/A-Python/04-error-handling.md) |
| 05 | [Packaging with uv & Poetry](Part1-Technical/A-Python/05-packaging-uv-poetry.md) |

### [C — Integration](Part1-Technical/C-Integration/)

| # | Module |
|---|--------|
| 01 | [REST, GraphQL & Webhooks](Part1-Technical/C-Integration/01-rest-graphql-webhooks.md) |
| 02 | [Auth Flows — OAuth2, API Keys, mTLS, JWT](Part1-Technical/C-Integration/02-auth-flows.md) |
| 03 | [Resilience Patterns](Part1-Technical/C-Integration/03-resilience-patterns.md) |
| 04 | [Data Mapping & Transformation](Part1-Technical/C-Integration/04-data-mapping-transformation.md) |

### [D — DevOps](Part1-Technical/D-DevOps/)

| # | Module |
|---|--------|
| 01 | [Docker](Part1-Technical/D-DevOps/01-docker.md) |
| 02 | [CI/CD with GitHub Actions](Part1-Technical/D-DevOps/02-cicd-github-actions.md) |
| 03 | [Environment Separation](Part1-Technical/D-DevOps/03-env-separation.md) |
| 04 | [Secrets & Configuration](Part1-Technical/D-DevOps/04-secrets-config.md) |

### [F — Observability](Part1-Technical/F-Observability/)

| # | Module |
|---|--------|
| 01 | [Structured Logging](Part1-Technical/F-Observability/01-structured-logging.md) |
| 02 | [Distributed Tracing (OpenTelemetry)](Part1-Technical/F-Observability/02-distributed-tracing-otel.md) |
| 03 | [Metrics, Prometheus & Grafana](Part1-Technical/F-Observability/03-metrics-prometheus-grafana.md) |

### [H — AI / LLM](Part1-Technical/H-AI-LLM/)

| # | Module |
|---|--------|
| 01 | [LLM API & Prompt Engineering](Part1-Technical/H-AI-LLM/01-llm-api-prompt-engineering.md) |
| 02 | [Tool Calling (Function Calling)](Part1-Technical/H-AI-LLM/02-tool-calling.md) |

### [I — Data](Part1-Technical/I-Data/)

| # | Module |
|---|--------|
| 01 | [SQL Fluency](Part1-Technical/I-Data/01-sql-fluency.md) |
| 02 | [ETL & ELT Patterns](Part1-Technical/I-Data/02-etl-elt.md) |

---

## Part 2 — Project

*Coming soon.*

---

## Part 3 — Non-Technical

| # | Module |
|---|--------|
| 01 | [Problem Decomposition](Part3-Non-Technical/01-decomposition.md) |
| 02 | [Story Bank — 6 Frameworks + Examples](Part3-Non-Technical/02-story-bank.md) |

---

## Suggested reading order

<details>
<summary><strong>New to FDE work — start here</strong></summary>

1. [Problem Decomposition](Part3-Non-Technical/01-decomposition.md)
2. [REST, GraphQL & Webhooks](Part1-Technical/C-Integration/01-rest-graphql-webhooks.md)
3. [Async/Await](Part1-Technical/A-Python/01-async-await.md)
4. [Auth Flows](Part1-Technical/C-Integration/02-auth-flows.md)
5. [Resilience Patterns](Part1-Technical/C-Integration/03-resilience-patterns.md)
6. [Story Bank](Part3-Non-Technical/02-story-bank.md)

</details>

<details>
<summary><strong>AI / LLM integration track</strong></summary>

1. [LLM API & Prompt Engineering](Part1-Technical/H-AI-LLM/01-llm-api-prompt-engineering.md)
2. [Tool Calling](Part1-Technical/H-AI-LLM/02-tool-calling.md)
3. [Structured Logging](Part1-Technical/F-Observability/01-structured-logging.md)
4. [Error Handling](Part1-Technical/A-Python/04-error-handling.md)

</details>
