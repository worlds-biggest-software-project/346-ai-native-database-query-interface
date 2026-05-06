# Standards & API Reference

> Project: AI-Native Database Query Interface · Generated: 2026-05-04

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 9075:2023 (SQL:2023)**
- URL: https://www.iso.org/standard/76583.html
- The ninth edition of the ANSI/ISO SQL standard, formally adopted June 2023. Any NL-to-SQL tool must generate output valid under the target database's SQL dialect, which in turn must be traceable to SQL:2023 core compliance. Part 16 (SQL/PGQ — Property Graph Queries) is newly standardised, enabling graph-style traversal within SQL databases. Dialect-specific extensions (PostgreSQL, MySQL, BigQuery, Snowflake, Redshift) are vendor deviations from this baseline.

**ISO/IEC 9075-3:2023 (SQL/CLI — Call-Level Interface)**
- URL: https://www.iso.org/standard/76585.html
- Defines the standard call-level interface for executing SQL from application code. JDBC and ODBC are direct implementations of this standard; any AI query interface that connects to a live database uses these connection abstractions for schema introspection and query execution.

---

### W3C & IETF Standards

**RFC 7230–7235 (HTTP/1.1) and RFC 9110–9114 (HTTP Semantics, HTTP/2, HTTP/3)**
- URLs: https://www.rfc-editor.org/rfc/rfc9110, https://www.rfc-editor.org/rfc/rfc9113, https://www.rfc-editor.org/rfc/rfc9114
- NL-to-SQL REST APIs (Cortex Analyst, Vanna managed, AI2SQL, Wren AI) operate over HTTP. API design must comply with HTTP semantics for request methods, status codes, and content negotiation.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorisation framework for protecting REST API endpoints. All production NL-to-SQL APIs with multi-user access must implement OAuth 2.0 flows (client credentials for server-to-server; authorization code for user-facing apps).

**RFC 6750 — OAuth 2.0 Bearer Token Usage**
- URL: https://www.rfc-editor.org/rfc/rfc6750
- Defines how Bearer tokens are included in HTTP requests. Required alongside RFC 6749 for any token-secured NL query API.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- JWT is the standard encoding for OAuth 2.0 access tokens in database API contexts; also used in Snowflake key-pair authentication and Databricks personal access tokens.

**W3C Server-Sent Events (SSE)**
- URL: https://www.w3.org/TR/eventsource/
- Used by MCP remote transport (HTTP + SSE) and streaming NL-to-SQL APIs (Snowflake Cortex Analyst streaming mode, Vanna 2.0 streaming). Enables real-time progressive token output in chat interfaces.

---

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1 / 3.2**
- URLs: https://spec.openapis.org/oas/v3.1.1.html · https://spec.openapis.org/oas/v3.2.0.html
- The standard format for documenting REST APIs in YAML/JSON. OAS 3.2 (2026) adds native QUERY HTTP method support, streaming media type support (SSE, JSON Lines), and OAuth 2.0 Device Authorization Flow. NL-to-SQL APIs exposed as REST services should publish an OpenAPI document to enable automatic SDK generation and contract testing.

**JSON Schema (draft 2020-12)**
- URL: https://json-schema.org/specification
- Used to validate request/response bodies in OpenAPI 3.1+ and in MCP message payloads. Relevant for validating semantic model YAML structures and API response schemas in NL-to-SQL services.

**GraphQL Specification (2021)**
- URL: https://spec.graphql.org/October2021/
- The dbt Semantic Layer and Wren Engine expose MetricFlow query APIs via GraphQL. Some NL-to-SQL tools (Wren AI) use GraphQL as an alternative to REST for structured metric queries.

**Model Context Protocol (MCP) Specification 2025-11-25**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- An open standard (JSON-RPC 2.0 over stdio or HTTP/SSE) for connecting AI assistants to external data systems as tools, resources, and prompts. Now governed by the Agentic AI Foundation (Linux Foundation, co-founded by Anthropic, Block, OpenAI). DBHub and Wren Engine implement MCP servers for database access. DataGrip 2026.1 ships a database-specific MCP server. The 2026 roadmap includes a stateless HTTP transport variant and a Tasks primitive for async long-running operations.

**dbt MetricFlow Specification (YAML)**
- URL: https://docs.getdbt.com/docs/build/about-metricflow
- The semantic model DSL used by dbt Cloud's Semantic Layer. Defines metrics, dimensions, entities, and measures in YAML. NL-to-SQL tools that integrate with dbt (Wren AI, Cortex Analyst) or target enterprise data teams should be compatible with MetricFlow definitions. Queries are served via JDBC or GraphQL endpoints.

**JDBC 4.3 (JSR 221)**
- URL: https://jcp.org/en/jsr/detail?id=221
- The Java standard for relational database connectivity. Most enterprise database drivers implement JDBC; schema introspection in Java-hosted NL-to-SQL tools uses `DatabaseMetaData` APIs defined here.

**ODBC 3.8 (Microsoft / Open Group)**
- URL: https://learn.microsoft.com/en-us/sql/odbc/reference/odbc-programmer-s-reference
- The platform-neutral C-level database connectivity standard. Required for NL-to-SQL tools targeting Windows-centric enterprise environments or SQL Server deployments.

---

### Security & Authentication Standards

**OWASP Top 10 for LLM Applications (2025 edition)**
- URL: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- The authoritative checklist for LLM security risks directly applicable to NL-to-SQL systems. Key risks include: LLM01 Prompt Injection (malicious NL input causing destructive SQL generation), LLM02 Insecure Output Handling (generated SQL executed without validation), LLM08 Excessive Agency (the query interface executing schema-altering statements it should not), and LLM09 Misinformation (returning plausible but incorrect query results). Mitigation: treat LLM-generated SQL as untrusted; use parameterised queries; enforce read-only mode by default.

**OWASP SQL Injection Prevention Cheat Sheet**
- URL: https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html
- Parameterised queries and prepared statements are the primary defence. NL2SQL tools must not construct SQL by string interpolation from user-controlled natural language input. Prompt-to-SQL (P2SQL) injection — where a user's data value contains LLM instructions — is an active research area with no fully automated mitigation as of 2026.

**NIST SP 800-207 — Zero Trust Architecture**
- URL: https://csrc.nist.gov/publications/detail/sp/800-207/final
- Governs access control design for enterprise data systems. An NL query gateway operating as a proxy to sensitive databases should implement Zero Trust principles: authenticate every request, authorise at the query layer (not only the database layer), and log all access.

**EU AI Act (Regulation (EU) 2024/1689)**
- URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32024R1689
- Fully applicable from August 2, 2026. An NL-to-SQL system deployed in a high-risk context (e.g., querying HR, financial, or health data) may be subject to transparency and human oversight requirements under Article 13 and 14. The query audit log is a direct compliance artefact for this regulation.

**GDPR Article 32 — Security of Processing**
- URL: https://gdpr-info.eu/art-32-gdpr/
- Requires appropriate technical measures to ensure a level of security appropriate to the risk. For NL-to-SQL tools processing personal data, this mandates encryption in transit (TLS 1.3), access controls, and audit logging of all query activity.

**OAuth 2.0 + OpenID Connect (OIDC)**
- URLs: https://oauth.net/2/ · https://openid.net/connect/
- The standard identity layer for user authentication in NL query APIs. OIDC ID tokens carry user identity claims used for per-user row-level security enforcement in Vanna 2.0's user-aware architecture.

---

### MCP Server Specifications

**Model Context Protocol — Tools, Resources, and Prompts Primitives**
- URL: https://modelcontextprotocol.io/docs/concepts/tools
- MCP tools are the primary primitive for exposing database operations (execute_sql, search_objects, generate_sql) to AI agents. Resources expose schema metadata as browsable, citable context. Prompts expose canned workflows (e.g., "explain this database") as reusable templates.

**MCP Apps (SEP-1865) — UI Extension Specification**
- URL: https://modelcontextprotocol.io (specification link to be confirmed once finalised)
- Formalized in early 2026, MCP Apps enables MCP servers to render rich UI components (e.g., interactive result tables, chart previews) back to compatible AI clients. Relevant for NL-to-SQL interfaces that want to display query results inline in the chat.

**Microsoft SQL MCP Server (Data API Builder)**
- URL: https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview
- Microsoft's reference implementation of an MCP server over SQL databases, part of the Azure Data API Builder product. Documents the MCP tool surface for SQL query execution, schema navigation, and safe execution patterns.

---

## Similar Products — Developer Documentation & APIs

### Snowflake Cortex Analyst
- **Description:** Fully-managed text-to-SQL service within Snowflake that converts natural language questions to SQL using a YAML semantic model, achieving 90%+ accuracy on real-world BI workloads.
- **API Documentation:** https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst/rest-api
- **SDKs/Libraries:** Snowflake Python Connector, Snowflake Connector for Java, Node.js driver; Streamlit reference app: https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-analyst/
- **Developer Guide:** https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-analyst
- **Standards:** REST/JSON; HTTP + SSE streaming; OpenAPI-compliant endpoint design
- **Authentication:** Snowflake session token (key-pair JWT or OAuth 2.0); RBAC via `SNOWFLAKE.CORTEX_ANALYST_USER` role

### Vanna.ai
- **Description:** Open-source (MIT) RAG-based text-to-SQL Python library and managed SaaS. LLM-agnostic; trains on DDL, query examples, and schema documentation.
- **API Documentation:** https://vanna.ai/docs (managed SaaS REST API); Python library: https://github.com/vanna-ai/vanna
- **SDKs/Libraries:** Python package (`pip install vanna`); supports OpenAI, Anthropic, Gemini, Ollama, AWS Bedrock, and pluggable vector stores (ChromaDB, Pinecone, Qdrant)
- **Developer Guide:** https://vanna.ai/docs/getting-started
- **Standards:** Python library; managed SaaS REST; no published OpenAPI spec
- **Authentication:** LLM API keys per provider; managed SaaS uses API key auth

### DBHub (Bytebase)
- **Description:** Zero-dependency, token-efficient MCP server for PostgreSQL, MySQL, MariaDB, SQL Server, and SQLite. Bridges any MCP-compatible AI client to live databases.
- **API Documentation:** https://dbhub.ai · https://github.com/bytebase/dbhub
- **SDKs/Libraries:** None — communicates via MCP protocol (JSON-RPC over stdio or HTTP/SSE); compatible with Claude, Cursor, VS Code, GitHub Copilot
- **Developer Guide:** https://www.deployhq.com/blog/how-to-generate-sql-queries-with-ai-step-by-step-guide-using-claude-code-and-dbhub
- **Standards:** MCP specification 2025-11-25 (JSON-RPC 2.0); HTTP + SSE transport
- **Authentication:** Database-level credentials; MCP transport authentication per client configuration

### Wren AI
- **Description:** Open-source (Apache 2.0) GenBI agent with semantic layer (MDL). Text-to-SQL and text-to-chart, supporting 12+ databases and any LLM. Wren Engine exposes a semantic MCP server.
- **API Documentation:** https://www.getwren.ai/oss · REST API: https://medium.com/@qdrddr/wrenai-text-to-sql-api-the-good-stuff-e4d57c0c181c
- **SDKs/Libraries:** Docker Compose deployment; Python and TypeScript community clients; dbt integration
- **Developer Guide:** https://github.com/Canner/WrenAI · https://github.com/Canner/wren-engine
- **Standards:** REST/JSON; GraphQL (Wren Engine MetricFlow queries); MCP (Wren Engine server); JDBC (Wren Engine data source connections)
- **Authentication:** API key; database-level credentials

### dbt Semantic Layer (MetricFlow)
- **Description:** A semantic layer specification and query engine that encodes business metric definitions in YAML (dbt models). Provides JDBC and GraphQL endpoints for consistent metric queries; increasingly used as a business vocabulary layer for NL-to-SQL tools.
- **API Documentation:** https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-sl
- **SDKs/Libraries:** Python SDK: `dbt-cloud-cli`; JDBC driver for BI tool connections; GraphQL API
- **Developer Guide:** https://docs.getdbt.com/docs/build/about-metricflow
- **Standards:** YAML semantic model spec (MetricFlow); JDBC 4.3; GraphQL 2021; REST/JSON
- **Authentication:** dbt Cloud service tokens; database credentials for data platform connections

### Google BigQuery (Gemini in BigQuery)
- **Description:** Native NL-to-SQL and AI SQL functions (AI.IF, AI.GENERATE, AI.EMBED, AI.SIMILARITY) embedded in BigQuery, with cross-service federation to Cloud SQL and AlloyDB.
- **API Documentation:** https://docs.cloud.google.com/bigquery/docs/gemini-overview · https://docs.cloud.google.com/bigquery/docs/write-sql-gemini
- **SDKs/Libraries:** Google Cloud BigQuery client libraries: Python (`google-cloud-bigquery`), Java, Go, Node.js, C#
- **Developer Guide:** https://cloud.google.com/blog/products/data-analytics/nl2sql-with-bigquery-and-gemini
- **Standards:** BigQuery REST API (JSON/HTTP); BigQuery Storage API (gRPC/Arrow); OpenAPI-described endpoints
- **Authentication:** Google Cloud IAM; OAuth 2.0; Application Default Credentials (ADC)

### Databricks Assistant (Genie / ai_query)
- **Description:** Context-aware AI assistant embedded in Databricks Notebooks and SQL Editor, powered by Unity Catalog metadata. Includes ai_query, ai_parse_document, and ai_prep_search SQL functions for AI-enriched pipelines.
- **API Documentation:** https://docs.databricks.com/aws/en/large-language-models/ai-functions · https://docs.databricks.com/aws/en/sql/language-manual/functions/ai_query
- **SDKs/Libraries:** Databricks Python SDK (`databricks-sdk`); Databricks Connect; JDBC driver for SQL Editor
- **Developer Guide:** https://www.databricks.com/product/databricks-assistant
- **Standards:** REST/JSON (Databricks REST API 2.0); Delta Sharing protocol; Unity Catalog REST API
- **Authentication:** Databricks personal access tokens; OAuth M2M (Machine-to-Machine); Microsoft Entra ID integration

### Microsoft Copilot in Azure SQL / Fabric
- **Description:** Natural language to T-SQL in the Azure portal, VS Code MSSQL Extension, and Microsoft Fabric. Skills include schema explanation, query generation, performance diagnostics, and DML/DDL authoring.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/azure-sql/copilot/copilot-azure-sql-overview · https://learn.microsoft.com/en-us/fabric/database/sql/copilot-sql-database
- **SDKs/Libraries:** Microsoft.Data.SqlClient (C#/.NET); pyodbc / sqlalchemy (Python); mssql VS Code extension
- **Developer Guide:** https://devblogs.microsoft.com/azure-sql/vscode-mssql-march-2026/
- **Standards:** T-SQL (Microsoft dialect of SQL:2023); REST/JSON; Azure REST API design guidelines
- **Authentication:** Microsoft Entra ID (Azure AD); Azure RBAC; connection string with Entra token

### JetBrains DataGrip AI Assistant
- **Description:** Embedded NL-to-SQL within the DataGrip IDE, with full live-schema context from the connected database. Exposes a database-specific MCP server (2026.1) for use by external AI agents.
- **API Documentation:** https://www.jetbrains.com/datagrip/features/ai/ · https://www.jetbrains.com/help/ai-assistant/use-ai-assistant.html
- **SDKs/Libraries:** No external API; IDE plugin only. MCP server available for external AI client consumption from DataGrip 2026.1
- **Developer Guide:** https://blog.jetbrains.com/datagrip/2026/03/26/datagrip-2026-1-redesigned-query-files-data-source-templates-in-your-jetbrains-account-ai-agents-in-the-ai-chat-explain-plan-flow-enhancements-and-more/
- **Standards:** MCP 2025-11-25 (exposed MCP server); JDBC (database connections); JetBrains AI Assistant API (internal)
- **Authentication:** JetBrains account; AI Assistant subscription; database credentials

---

## Notes

**Emerging: MCP as the universal database tool protocol.** By mid-2026, MCP servers for databases (DBHub, Wren Engine, DataGrip, Microsoft SQL MCP Server, Google Cloud SQL managed MCP) have converged on a shared tool surface: execute_sql, search_objects/list_tables, generate_sql, and explain_db. A new NL-to-SQL project should implement and document an MCP server as a primary delivery artefact rather than treating it as an add-on.

**Semantic layer YAML has no universal standard yet.** Snowflake (YAML semantic models), dbt (MetricFlow YAML), and Wren AI (MDL YAML) each define their own schema. An AI-native project that auto-generates semantic layers should target MetricFlow format as the broadest-reach option (dbt Cloud + Wren AI support), while providing an export path to Snowflake semantic view format.

**SQL injection via prompt remains an open research problem.** OWASP and academic researchers (ACM PODS 2025) have documented P2SQL injection attacks but no standardised mitigation exists beyond read-only mode enforcement and treating LLM output as untrusted input. Building a safe-execution wrapper (parameterised query re-writer, query AST validation) is a differentiating security feature with no established off-the-shelf standard to follow.

**EU AI Act compliance (August 2026 deadline).** Systems querying sensitive data (HR, finance, health) must implement audit logging, human oversight mechanisms, and transparency disclosures. The query audit log is both a compliance requirement and a training data source for semantic layer improvement.
