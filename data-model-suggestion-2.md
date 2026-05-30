# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: AI-Native Database Query Interface · Created: 2026-05-25

## Philosophy

This model collapses the 15-table normalized schema into 6 core tables by embedding related data as JSONB documents within their parent entities. Organisations embed their users, API keys, and security policies. Database connections embed the full introspected schema (tables and columns) and semantic layer (metrics, dimensions, synonyms) as JSONB arrays. Conversations embed their message history. The principle: loading everything needed to generate a SQL query from a natural language input should require at most two rows — the organisation (for auth and policies) and the database connection (for schema, semantics, and context).

Queries remain as a standalone partitioned table because they arrive at high volume and serve as the immutable audit trail required by EU AI Act and GDPR. Cost records remain relational for billing aggregation.

**Best for:** Early-stage NL-to-SQL products prioritising development speed, single-query context loading for LLM prompts, and rapid iteration on the semantic layer format. Ideal for single-org or small-team deployments where schema introspection results and semantic definitions change together.

**Trade-offs:**
- (+) 6 tables — minimal migration surface as the product evolves
- (+) Single-row fetch loads complete connection context (schema + semantic layer + security policies)
- (+) Schema and semantic layer evolve together in one document — no cross-table coordination
- (+) Conversation history in one row for multi-turn refinement context
- (-) Schema introspection for databases with 500+ tables creates large JSONB documents
- (-) No foreign-key enforcement between semantic entity references and introspected columns
- (-) Cross-connection schema search requires JSONB extraction
- (-) Synonym lookup requires JSONB path queries instead of GIN-indexed arrays

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SQL:2023 (ISO/IEC 9075:2023) | `database_connections.dialect` tracks target dialect; `queries.target_dialect` in query records |
| MCP 2025-11-25 | Platform exposes MCP tools; `organisations.api_keys_json[].scopes` gate MCP tool access |
| dbt MetricFlow | `database_connections.semantic_layer_json` stores MetricFlow-compatible metric definitions |
| OAuth 2.0 / OIDC | API keys in `organisations.api_keys_json[]`; OIDC subjects in `organisations.users_json[]` |
| OWASP Top 10 for LLM Apps | `queries.injection_json` flags P2SQL injection attempts |
| OpenAPI 3.1 | REST API documented as OpenAPI; scopes in `api_keys_json[]` map to OpenAPI security schemes |
| GDPR Article 32 | `audit_log` captures all data access; `database_connections.credential_ref` uses vault references |
| EU AI Act | `queries` table serves as transparency audit trail for Articles 13-14 |
| NIST SP 800-207 | Zero Trust: security policies embedded in org, applied per-query, logged in audit_log |
| W3C SSE | `queries.is_streaming` tracks streaming mode for progressive response delivery |

---

## Core Tables

### organisations

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT UNIQUE NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'free' CHECK (plan IN ('free','pro','enterprise')),
    settings_json   JSONB NOT NULL DEFAULT '{}',
    monthly_budget_cents BIGINT,

    users_json      JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","email":"analyst@example.com","full_name":"Jane Analyst",
    --   "oidc_subject":"google|12345","role":"analyst","is_active":true}]

    api_keys_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"mcp-prod","key_prefix":"nlsql_",
    --   "key_hash":"sha256...","scopes":["chat","execute","generate_sql"],
    --   "rate_limit_rpm":100,"is_active":true,"expires_at":"2027-01-01T00:00:00Z"}]

    security_policies_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"id":"uuid","name":"Hide PII Columns","connection_slug":"prod-pg",
    --   "policy_type":"column_mask","target_table":"users","target_column":"email",
    --   "mask_function":"redact","applies_to_role":"analyst","priority":10},
    --  {"id":"uuid","name":"Filter by Region","connection_slug":"prod-pg",
    --   "policy_type":"row_filter","target_table":"orders",
    --   "filter_expression":"region = current_user_region()","applies_to_role":"analyst"}]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
CREATE INDEX idx_orgs_keys ON organisations USING GIN (api_keys_json);
```

### database_connections

Each connection embeds its introspected schema and semantic layer. Loading one row provides everything the LLM needs to generate accurate SQL.

```sql
CREATE TABLE database_connections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    dialect         TEXT NOT NULL CHECK (dialect IN ('postgresql','mysql','snowflake','bigquery',
                                                     'sqlserver','redshift','oracle','sqlite',
                                                     'duckdb','clickhouse')),
    host            TEXT,
    port            INTEGER,
    database_name   TEXT,
    credential_ref  TEXT NOT NULL,
    connection_options_json JSONB NOT NULL DEFAULT '{}',
    safety_mode     TEXT NOT NULL DEFAULT 'read_only'
                    CHECK (safety_mode IN ('read_only','read_write')),

    schema_json     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"version":3,"introspected_at":"2026-05-25T10:00:00Z",
    --   "table_count":42,"column_count":287,
    --   "tables":[
    --     {"schema":"public","name":"orders","type":"table","row_count_estimate":1500000,
    --       "description":"Customer orders",
    --       "columns":[
    --         {"name":"id","type":"uuid","nullable":false,"pk":true},
    --         {"name":"customer_id","type":"uuid","nullable":false,"fk":"public.customers.id"},
    --         {"name":"total_cents","type":"bigint","nullable":false},
    --         {"name":"status","type":"text","nullable":false},
    --         {"name":"created_at","type":"timestamptz","nullable":false}
    --       ]},
    --     {"schema":"public","name":"customers","type":"table",
    --       "columns":[
    --         {"name":"id","type":"uuid","nullable":false,"pk":true},
    --         {"name":"email","type":"text","nullable":false},
    --         {"name":"region","type":"text","nullable":true}
    --       ]}
    --   ],
    --   "drift":{"tables_added":[],"tables_removed":[],"columns_changed":[]}}

    semantic_layer_json JSONB NOT NULL DEFAULT '{}',
    -- Example: {"name":"Sales Metrics v2","source":"manual","version":2,
    --   "entities":[
    --     {"type":"metric","name":"total_revenue","display_name":"Total Revenue",
    --       "sql":"SUM(orders.total_cents)","agg":"sum","table":"orders","column":"total_cents",
    --       "synonyms":["revenue","sales","income"]},
    --     {"type":"dimension","name":"order_status","display_name":"Order Status",
    --       "table":"orders","column":"status",
    --       "synonyms":["status","state"]},
    --     {"type":"relationship","name":"order_customer",
    --       "from_table":"orders","from_column":"customer_id",
    --       "to_table":"customers","to_column":"id","type":"many_to_one"}
    --   ]}

    schema_history_json JSONB NOT NULL DEFAULT '[]',
    -- Example: [{"version":2,"introspected_at":"2026-05-20T10:00:00Z",
    --   "drift":{"tables_added":["returns"],"columns_changed":[]}},
    --  {"version":1,"introspected_at":"2026-05-15T10:00:00Z","drift":null}]

    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_connections_org ON database_connections(org_id);
CREATE INDEX idx_connections_schema ON database_connections USING GIN (schema_json);
CREATE INDEX idx_connections_semantic ON database_connections USING GIN (semantic_layer_json);
```

### conversations

Multi-turn chat sessions with embedded message history for context-aware refinement.

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID,
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    title           TEXT,

    messages_json   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"role":"user","content":"Show me total revenue by region this quarter",
    --    "timestamp":"2026-05-25T10:00:00Z"},
    --   {"role":"assistant","content":"Here's the revenue by region...",
    --    "query_id":"uuid","sql":"SELECT region, SUM(total_cents)...",
    --    "timestamp":"2026-05-25T10:00:02Z"},
    --   {"role":"user","content":"Now exclude internal users",
    --    "timestamp":"2026-05-25T10:01:00Z"},
    --   {"role":"assistant","content":"Filtered out internal users...",
    --    "query_id":"uuid","sql":"SELECT region, SUM(total_cents)... WHERE NOT is_internal",
    --    "timestamp":"2026-05-25T10:01:03Z"}
    -- ]

    message_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_convos_org ON conversations(org_id, created_at DESC);
CREATE INDEX idx_convos_user ON conversations(user_id, created_at DESC);
```

### queries

Standalone audit trail for every NL→SQL query. Partitioned by time for compliance retention.

```sql
CREATE TABLE queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    conversation_id UUID,
    user_id         UUID,
    api_key_prefix  TEXT,
    connection_id   UUID NOT NULL,

    nl_input        TEXT NOT NULL,
    generated_sql   TEXT,
    target_dialect  TEXT NOT NULL,
    model_used      TEXT NOT NULL,

    safety_mode     TEXT NOT NULL DEFAULT 'read_only'
                    CHECK (safety_mode IN ('read_only','read_write')),
    is_streaming    BOOLEAN NOT NULL DEFAULT false,

    injection_json  JSONB,
    -- Example: {"detected":true,"category":"data_exfiltration","confidence":0.92,
    --   "flagged_phrase":"ignore previous instructions","action":"blocked"}

    security_json   JSONB,
    -- Example: {"policies_applied":["Hide PII Columns","Filter by Region"],
    --   "rows_filtered":true,"columns_masked":["email"],
    --   "original_sql":"SELECT * FROM users","rewritten_sql":"SELECT id, '***' AS email FROM users WHERE region = 'US'"}

    explanation_json JSONB,
    -- Example: [{"clause_type":"select","sql":"SELECT region, SUM(total_cents) AS revenue",
    --   "explanation":"Selects the region and calculates total revenue by summing order amounts"},
    --  {"clause_type":"from","sql":"FROM orders JOIN customers ON ...",
    --   "explanation":"Joins orders with customers to access region data"},
    --  {"clause_type":"where","sql":"WHERE created_at >= '2026-04-01'",
    --   "explanation":"Filters to orders created in Q2 2026 (this quarter)"}]

    execution_status TEXT NOT NULL DEFAULT 'pending'
                    CHECK (execution_status IN ('pending','generating','generated','executing',
                                                 'completed','error','blocked','injection_blocked')),
    row_count       INTEGER,
    execution_ms    INTEGER,
    ttft_ms         INTEGER,
    error_message   TEXT,

    tokens_input    INTEGER,
    tokens_output   INTEGER,
    llm_cost_cents  BIGINT NOT NULL DEFAULT 0,

    quality_score   NUMERIC(5,4),
    user_feedback   TEXT CHECK (user_feedback IS NULL OR user_feedback IN (
                        'thumbs_up','thumbs_down','edited','rerun')),
    edited_sql      TEXT,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_queries_org ON queries(org_id, created_at DESC);
CREATE INDEX idx_queries_convo ON queries(conversation_id, created_at);
CREATE INDEX idx_queries_conn ON queries(connection_id, created_at DESC);
CREATE INDEX idx_queries_injection ON queries USING GIN (injection_json)
    WHERE injection_json IS NOT NULL;
```

### cost_records

```sql
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    connection_slug TEXT,
    period          DATE NOT NULL,
    period_type     TEXT NOT NULL CHECK (period_type IN ('daily','monthly')),
    model_used      TEXT,
    query_count     INTEGER NOT NULL DEFAULT 0,
    tokens_input    BIGINT NOT NULL DEFAULT 0,
    tokens_output   BIGINT NOT NULL DEFAULT 0,
    total_cost_cents BIGINT NOT NULL DEFAULT 0,
    injection_blocked_count INTEGER NOT NULL DEFAULT 0,
    avg_execution_ms INTEGER,
    avg_quality_score NUMERIC(5,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, connection_slug, period, period_type, model_used)
);

CREATE INDEX idx_cost_org ON cost_records(org_id, period_type, period DESC);
```

### audit_log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','system','api_key','ai')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    changes_json    JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

---

## Example Queries

### Load complete context for query generation (single query)

```sql
SELECT dc.dialect, dc.safety_mode, dc.schema_json, dc.semantic_layer_json,
       o.security_policies_json
FROM database_connections dc
JOIN organisations o ON o.id = dc.org_id
WHERE dc.id = $1;
```

### API key validation and org lookup

```sql
SELECT o.id, o.slug, o.plan, o.security_policies_json
FROM organisations o
WHERE o.api_keys_json @> $1::jsonb;
-- $1 = '[{"key_prefix":"nlsql_abc","is_active":true}]'
```

### Query audit trail with security details

```sql
SELECT q.nl_input, q.generated_sql, q.target_dialect, q.execution_status,
       q.injection_json, q.security_json, q.explanation_json,
       q.row_count, q.llm_cost_cents, q.user_feedback
FROM queries q
WHERE q.org_id = $1 AND q.created_at BETWEEN $2 AND $3
ORDER BY q.created_at DESC;
```

### Conversation with full message history

```sql
SELECT c.title, c.messages_json, c.message_count,
       dc.name AS connection_name, dc.dialect
FROM conversations c
JOIN database_connections dc ON dc.id = c.connection_id
WHERE c.id = $1;
```

### Cost by connection this month

```sql
SELECT cr.connection_slug, cr.model_used,
       SUM(cr.query_count) AS queries,
       SUM(cr.total_cost_cents) AS cost_cents,
       SUM(cr.injection_blocked_count) AS blocked
FROM cost_records cr
WHERE cr.org_id = $1 AND cr.period_type = 'daily'
  AND cr.period >= date_trunc('month', now())
GROUP BY cr.connection_slug, cr.model_used
ORDER BY cost_cents DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation (with users, API keys, security policies) | 1 | organisations |
| Database Connectivity (with schema, semantic layer) | 1 | database_connections |
| Conversations (with message history) | 1 | conversations |
| Query Audit Trail (with explanations, security, injection) | 1 | queries (partitioned) |
| Cost Records | 1 | cost_records |
| Audit | 1 | audit_log (partitioned) |
| **Total** | **6** | |

---

## Key Design Decisions

1. **Connection as complete context document** — `database_connections` embeds `schema_json` (full introspected schema with tables, columns, types, foreign keys) and `semantic_layer_json` (metrics, dimensions, synonyms). Loading one row provides everything the LLM needs to generate accurate SQL. This minimises database round-trips on the query-generation hot path.

2. **Schema history embedded in connection** — `schema_history_json` stores previous schema versions with drift summaries. Combined with `schema_json.version`, this enables schema drift tracking without a separate snapshots table. The trade-off: very large schemas with frequent changes will grow the JSONB document.

3. **Security policies embedded in organisation** — `security_policies_json` stores row-level filters and column masks with `connection_slug` references. Loaded alongside the connection context, they're applied before SQL generation. The trade-off: no FK enforcement between policy targets and actual schema columns.

4. **Query explanations embedded in query rows** — `explanation_json` stores the clause-by-clause breakdown as a JSONB array on the query row itself. This avoids a separate table and ensures the explanation is always loaded with the query for debugging.

5. **Injection detection embedded in query rows** — `injection_json` captures P2SQL injection detection results (category, confidence, flagged phrase, action taken) directly on the query record. Queries with `execution_status = 'injection_blocked'` are findable via the GIN index.

6. **Security rewriting embedded in query rows** — `security_json` records which policies were applied, which columns were masked, and the original vs. rewritten SQL. This provides a complete audit trail of how security transformed the generated query.

7. **Conversation messages as JSONB array** — `messages_json` stores the multi-turn chat history as an ordered array of user/assistant messages with timestamps. Each assistant message references its `query_id` in the queries table. This keeps the conversation browsable in a single row while maintaining the audit trail in the queries table.

8. **API key validation via GIN index** — `api_keys_json` has a GIN index enabling `@>` containment queries for key prefix lookup. The gateway extracts the key prefix from the Bearer token and finds the matching org in one query.

9. **Semantic layer versioned with schema** — By co-locating `semantic_layer_json` with `schema_json` in the same row, schema introspection and semantic layer updates can be committed atomically. When the schema changes, the semantic layer can be validated against the new schema in the same transaction.

10. **Credentials as vault references** — `database_connections.credential_ref` stores a vault path rather than the actual connection password. Credentials are resolved at runtime from the vault, not stored in PostgreSQL.
