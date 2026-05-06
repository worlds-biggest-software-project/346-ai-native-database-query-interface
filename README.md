# AI-Native Database Query Interface

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native natural-language-to-SQL layer with schema-aware query generation, plain-English explanations, and optimisation suggestions across major database dialects.

This project is a model-agnostic NL-to-SQL interface for developers, analysts, and data platform teams who need self-serve access to relational data without writing SQL by hand. It combines live schema introspection, semantic-layer-aware translation, and query explanation so non-experts can query data confidently and engineers can audit what the AI generates.

---

## Why AI-Native Database Query Interface?

- Standalone NL-to-SQL tools (AI2SQL, BlazeSQL, Text2SQL.ai) are proprietary SaaS at USD 15-29/user/month and lack a first-class semantic layer, so accuracy degrades on poorly named enterprise schemas.
- Hyperscaler offerings (Snowflake Cortex Analyst, BigQuery Gemini, Databricks Assistant, Microsoft Copilot, Oracle SQL AI) are locked to a single platform and cannot federate queries across heterogeneous databases.
- Open-source alternatives are fragmented: Vanna.ai requires cold-start training, DBHub has no semantic layer or multi-user isolation, and Wren AI's MDL semantic model must be authored manually.
- JetBrains DataGrip's AI Assistant is desktop-only and cannot be embedded as a service in other applications.
- Enterprise needs - per-user row-level security in the NL layer, query audit trails, and prompt-injection defence - are weak or absent across the surveyed tools.

---

## Key Features

### Natural Language to SQL Core

- Natural language input translated to valid SQL for PostgreSQL, MySQL, and Snowflake at minimum, with extensibility to BigQuery, SQL Server, Redshift, Oracle, SQLite, and DuckDB.
- Live schema introspection so generated queries reference real table and column names rather than guessed identifiers.
- Plain-English query explanation describing what the generated SQL does, clause by clause.
- Read-only safety mode that blocks INSERT, UPDATE, DELETE, and DDL until a user explicitly permits write operations.

### Semantic Layer and Accuracy

- YAML-based semantic layer for metric definitions, synonyms, and table relationships, compatible with the dbt MetricFlow format.
- Multi-turn conversational refinement so users can iteratively narrow results ("now just show Q1", "exclude internal users").
- Backlog goal of auto-generating the semantic layer from DDL, query history, and schema comments to reduce upfront curation cost.

### Enterprise Controls

- Per-user row-level security applied at the NL translation layer rather than delegated to the downstream database.
- Immutable query audit log capturing each natural-language question, the generated SQL, and the result for compliance and debugging.
- Streaming API responses for low-latency chat UX.
- Backlog goal of prompt-injection detection on NL inputs.

### Integration Surface

- MCP (Model Context Protocol) server interface so the tool works with any MCP-compatible AI assistant (Claude, GitHub Copilot, Cursor, VS Code, Codex).
- REST API for embedding NL-to-SQL into third-party applications.
- Backlog goal of cross-database federated NL queries that decompose a single question into sub-queries across Snowflake, BigQuery, and Postgres and join the results.
- Backlog goal of basic text-to-chart output alongside generated SQL.

---

## AI-Native Advantage

Rather than using AI only as a one-shot prompt-to-SQL translator, this project treats AI as the substrate of the whole query pipeline: auto-generating semantic layer artefacts from DDL and query history, explaining and rewriting slow queries, refining results across multi-turn dialogue, and routing federated queries across heterogeneous databases. Combined with row-level security applied inside the NL layer, this addresses the "messy enterprise database" gap where current tools drop to 50-70% accuracy and where audit, security, and federation are weak across incumbents.

---

## Tech Stack & Deployment

Deployment is intended to be self-hosted first (Docker) with an optional managed mode, exposing an MCP server for AI-assistant integration and a REST API for embedding. The project aligns with the SQL:2023 standard for dialect generation, MCP for AI client interoperability, JDBC/ODBC for schema introspection and execution, and the dbt semantic layer format for metric definitions. The architecture is LLM-agnostic, following the pattern set by Vanna.ai and Wren AI of pluggable model backends (OpenAI, Anthropic, Gemini, Ollama, Bedrock).

---

## Market Context

NL-to-SQL sits at the intersection of the broader BI market (USD 33 billion in 2024, growing at ~8% CAGR) and conversational AI, with Gartner projecting that 80% of organisations will adopt AI-driven data solutions by 2026. Standalone tools price at USD 15-29/user/month for professional tiers, while hyperscalers bundle NL-to-SQL into existing database subscriptions. Primary buyers are business analysts and product managers seeking self-serve data access, data engineers building semantic layers, enterprise data platform teams standardising NL across multiple databases, and CISOs requiring auditing and access control on natural-language data requests.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
