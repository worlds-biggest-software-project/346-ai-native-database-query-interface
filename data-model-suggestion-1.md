# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: AI-Native Database Query Interface · Created: 2026-05-25

## Philosophy

This model gives every concept its own table: organisations, users, API keys, database connections, introspected schema elements, semantic layer definitions, security policies, conversations, queries, and cost records. Foreign keys enforce referential integrity across the full lifecycle — from connecting to a database, through schema introspection and semantic layer definition, to natural language query generation, explanation, and audit.

The schema introspection subsystem is modelled as a versioned snapshot chain: each `schema_snapshot` captures a point-in-time view of a connected database's structure, with `introspected_tables` and `introspected_columns` recording the discovered schema elements. This enables drift detection (comparing snapshots over time) and ensures the LLM always has an accurate, timestamped view of the target schema. The semantic layer tables mirror the dbt MetricFlow format — metrics, dimensions, measures, and synonyms — stored relationally so they can be queried, versioned, and validated independently.

Queries are first-class entities with their own table, partitioned by time for audit trail compliance. Each query records the natural language input, generated SQL, target dialect, execution status, and cost. Query explanations are stored separately to support clause-by-clause breakdown without bloating the query row.

**Best for:** Teams building a production NL-to-SQL gateway where schema accuracy, audit compliance (EU AI Act, GDPR), and multi-user row-level security are primary requirements. Ideal when the semantic layer will be curated by data engineers and the query audit trail must be queryable for compliance reporting.

**Trade-offs:**
- (+) Full referential integrity across connections, schemas, semantic layers, and queries
- (+) Schema drift detection via snapshot versioning
- (+) Semantic layer entities independently queryable and validatable
- (+) Row-level security policies enforced at the data model level
- (+) Query audit trail partitioned for compliance retention
- (-) 15 tables — more migrations as the product evolves
- (-) Loading a complete connection context (schema + semantic layer + policies) requires multiple joins
- (-) Schema introspection creates many rows in introspected_tables/columns for large databases
- (-) Semantic layer updates require coordinated inserts across semantic_layers and semantic_entities

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SQL:2023 (ISO/IEC 9075:2023) | `queries.target_dialect` tracks which SQL dialect was generated; `database_connections.dialect` references SQL:2023 compliance level |
| MCP 2025-11-25 | The platform exposes MCP tools (execute_sql, generate_sql); `api_keys.scopes` gate MCP tool access |
| dbt MetricFlow | `semantic_entities` stores metrics, dimensions, measures in a structure compatible with MetricFlow YAML export |
| OAuth 2.0 / OIDC / JWT | `api_keys` for service-to-service; `users` carry OIDC subject identifiers for SSO |
| OWASP Top 10 for LLM Apps | `queries.injection_detected` flags P2SQL injection attempts; `queries.safety_mode` enforces read-only |
| OpenAPI 3.1 | REST API published as OpenAPI document; `api_keys.scopes` map to OpenAPI security schemes |
| GDPR Article 32 | `audit_log` captures all data access; `database_connections.credential_ref` uses vault references |
| EU AI Act | `queries` table serves as the transparency audit trail required by Articles 13-14 |
| NIST SP 800-207 | Zero Trust: every query authenticated via API key, authorised via security policies, logged in audit_log |
| W3C SSE | Streaming query responses; `queries.is_streaming` tracks streaming mode |

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_orgs_slug ON organisations(slug);
```

### users

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    email           TEXT NOT NULL,
    full_name       TEXT,
    oidc_subject    TEXT,
    role            TEXT NOT NULL DEFAULT 'analyst'
                    CHECK (role IN ('owner','admin','data_engineer','analyst','viewer')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, email)
);

CREATE INDEX idx_users_org ON users(org_id);
CREATE INDEX idx_users_oidc ON users(oidc_subject) WHERE oidc_subject IS NOT NULL;
```

### api_keys

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    name            TEXT NOT NULL,
    key_prefix      TEXT NOT NULL,
    key_hash        TEXT NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{chat,execute}',
    rate_limit_rpm  INTEGER,
    rate_limit_tpm  INTEGER,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_keys_org ON api_keys(org_id);
CREATE INDEX idx_keys_prefix ON api_keys(key_prefix);
```

---

## Database Connectivity

### database_connections

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
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_introspected_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (org_id, slug)
);

CREATE INDEX idx_connections_org ON database_connections(org_id);
```

### schema_snapshots

Versioned point-in-time snapshots of introspected schema. Enables drift detection and ensures LLM context accuracy.

```sql
CREATE TABLE schema_snapshots (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    version         INTEGER NOT NULL,
    table_count     INTEGER NOT NULL DEFAULT 0,
    column_count    INTEGER NOT NULL DEFAULT 0,
    status          TEXT NOT NULL DEFAULT 'in_progress'
                    CHECK (status IN ('in_progress','completed','failed')),
    introspection_duration_ms INTEGER,
    drift_summary_json JSONB,
    -- Example: {"tables_added":["orders"],"tables_removed":[],"columns_changed":[
    --   {"table":"users","column":"email","change":"type_changed","from":"VARCHAR(100)","to":"TEXT"}]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (connection_id, version)
);

CREATE INDEX idx_snapshots_conn ON schema_snapshots(connection_id, version DESC);
```

### introspected_tables

```sql
CREATE TABLE introspected_tables (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_id     UUID NOT NULL REFERENCES schema_snapshots(id) ON DELETE CASCADE,
    schema_name     TEXT NOT NULL DEFAULT 'public',
    table_name      TEXT NOT NULL,
    table_type      TEXT NOT NULL DEFAULT 'table'
                    CHECK (table_type IN ('table','view','materialized_view','foreign_table')),
    row_count_estimate BIGINT,
    description     TEXT,
    UNIQUE (snapshot_id, schema_name, table_name)
);

CREATE INDEX idx_itables_snapshot ON introspected_tables(snapshot_id);
```

### introspected_columns

```sql
CREATE TABLE introspected_columns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_id        UUID NOT NULL REFERENCES introspected_tables(id) ON DELETE CASCADE,
    column_name     TEXT NOT NULL,
    ordinal_position INTEGER NOT NULL,
    data_type       TEXT NOT NULL,
    is_nullable     BOOLEAN NOT NULL DEFAULT true,
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    is_foreign_key  BOOLEAN NOT NULL DEFAULT false,
    fk_references   TEXT,
    default_value   TEXT,
    description     TEXT,
    UNIQUE (table_id, column_name)
);

CREATE INDEX idx_icolumns_table ON introspected_columns(table_id);
```

---

## Semantic Layer

### semantic_layers

Named semantic layer configurations per database connection. Compatible with dbt MetricFlow export.

```sql
CREATE TABLE semantic_layers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    version         INTEGER NOT NULL DEFAULT 1,
    source          TEXT NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual','auto_generated','dbt_import')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (connection_id, slug, version)
);

CREATE INDEX idx_semantic_conn ON semantic_layers(connection_id);
```

### semantic_entities

Metrics, dimensions, measures, synonyms, and relationships within a semantic layer.

```sql
CREATE TABLE semantic_entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    layer_id        UUID NOT NULL REFERENCES semantic_layers(id) ON DELETE CASCADE,
    entity_type     TEXT NOT NULL CHECK (entity_type IN ('metric','dimension','measure',
                                                          'synonym','relationship','entity')),
    name            TEXT NOT NULL,
    display_name    TEXT,
    description     TEXT,
    sql_expression  TEXT,
    data_type       TEXT,
    table_ref       TEXT,
    column_ref      TEXT,
    agg_function    TEXT CHECK (agg_function IS NULL OR agg_function IN (
                        'sum','count','count_distinct','avg','min','max','median')),
    synonyms        TEXT[] NOT NULL DEFAULT '{}',
    related_entity_id UUID REFERENCES semantic_entities(id),
    relationship_type TEXT CHECK (relationship_type IS NULL OR relationship_type IN (
                        'one_to_one','one_to_many','many_to_many')),
    config_json     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sentities_layer ON semantic_entities(layer_id);
CREATE INDEX idx_sentities_type ON semantic_entities(layer_id, entity_type);
CREATE INDEX idx_sentities_synonyms ON semantic_entities USING GIN (synonyms);
```

---

## Security

### security_policies

Row-level security and column masking rules applied at the NL translation layer.

```sql
CREATE TABLE security_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    name            TEXT NOT NULL,
    policy_type     TEXT NOT NULL CHECK (policy_type IN ('row_filter','column_mask','table_deny',
                                                          'column_deny')),
    applies_to_role TEXT,
    applies_to_user_id UUID REFERENCES users(id),
    target_schema   TEXT,
    target_table    TEXT NOT NULL,
    target_column   TEXT,
    filter_expression TEXT,
    mask_function   TEXT CHECK (mask_function IS NULL OR mask_function IN (
                        'redact','hash','partial','null_out')),
    priority        INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_secpol_conn ON security_policies(connection_id);
CREATE INDEX idx_secpol_org ON security_policies(org_id);
```

---

## Query Pipeline

### conversations

Multi-turn chat sessions for iterative query refinement.

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    user_id         UUID REFERENCES users(id),
    connection_id   UUID NOT NULL REFERENCES database_connections(id),
    title           TEXT,
    message_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_convos_org ON conversations(org_id, created_at DESC);
CREATE INDEX idx_convos_user ON conversations(user_id, created_at DESC);
```

### queries

Each NL→SQL query is an immutable audit record. Partitioned by time for compliance retention.

```sql
CREATE TABLE queries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL,
    conversation_id UUID,
    user_id         UUID,
    api_key_id      UUID,
    connection_id   UUID NOT NULL,

    nl_input        TEXT NOT NULL,
    generated_sql   TEXT,
    target_dialect  TEXT NOT NULL,
    model_used      TEXT NOT NULL,

    safety_mode     TEXT NOT NULL DEFAULT 'read_only'
                    CHECK (safety_mode IN ('read_only','read_write')),
    is_streaming    BOOLEAN NOT NULL DEFAULT false,
    injection_detected BOOLEAN NOT NULL DEFAULT false,
    injection_details_json JSONB,
    -- Example: {"category":"data_exfiltration","confidence":0.92,
    --   "flagged_phrase":"ignore previous instructions"}

    security_policies_applied TEXT[] NOT NULL DEFAULT '{}',
    sql_rewritten   BOOLEAN NOT NULL DEFAULT false,
    original_sql    TEXT,

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
CREATE INDEX idx_queries_status ON queries(execution_status) WHERE execution_status != 'completed';
```

### query_explanations

Clause-by-clause plain-English explanations of generated SQL.

```sql
CREATE TABLE query_explanations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    query_id        UUID NOT NULL,
    clause_order    INTEGER NOT NULL,
    sql_clause      TEXT NOT NULL,
    clause_type     TEXT NOT NULL CHECK (clause_type IN ('select','from','join','where','group_by',
                                                          'having','order_by','limit','cte','subquery',
                                                          'window','union','insert','update','delete')),
    explanation     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_explanations_query ON query_explanations(query_id, clause_order);
```

---

## Operations

### cost_records

```sql
CREATE TABLE cost_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_id          UUID NOT NULL REFERENCES organisations(id),
    connection_id   UUID,
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
    UNIQUE (org_id, connection_id, period, period_type, model_used)
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

### Load connection context for query generation

```sql
SELECT dc.*, ss.id AS snapshot_id, ss.version AS schema_version,
       sl.id AS layer_id, sl.name AS layer_name
FROM database_connections dc
LEFT JOIN schema_snapshots ss ON ss.connection_id = dc.id
    AND ss.version = (SELECT MAX(version) FROM schema_snapshots WHERE connection_id = dc.id)
LEFT JOIN semantic_layers sl ON sl.connection_id = dc.id AND sl.is_active
WHERE dc.id = $1;
```

### Get introspected schema for LLM context

```sql
SELECT it.schema_name, it.table_name, it.table_type, it.description,
       ic.column_name, ic.data_type, ic.is_nullable, ic.is_primary_key,
       ic.is_foreign_key, ic.fk_references, ic.description AS column_description
FROM introspected_tables it
JOIN introspected_columns ic ON ic.table_id = it.id
WHERE it.snapshot_id = $1
ORDER BY it.schema_name, it.table_name, ic.ordinal_position;
```

### Apply security policies to a generated query

```sql
SELECT sp.policy_type, sp.target_table, sp.target_column,
       sp.filter_expression, sp.mask_function
FROM security_policies sp
WHERE sp.connection_id = $1 AND sp.is_active
  AND (sp.applies_to_user_id = $2 OR sp.applies_to_role = $3)
ORDER BY sp.priority DESC;
```

### Query audit trail for compliance

```sql
SELECT q.nl_input, q.generated_sql, q.target_dialect, q.execution_status,
       q.injection_detected, q.security_policies_applied,
       q.row_count, q.llm_cost_cents, q.user_feedback,
       u.email AS user_email, dc.name AS connection_name
FROM queries q
LEFT JOIN users u ON u.id = q.user_id
LEFT JOIN database_connections dc ON dc.id = q.connection_id
WHERE q.org_id = $1 AND q.created_at BETWEEN $2 AND $3
ORDER BY q.created_at DESC;
```

### Schema drift detection

```sql
SELECT s1.version AS old_version, s2.version AS new_version,
       s2.drift_summary_json
FROM schema_snapshots s1
JOIN schema_snapshots s2 ON s2.connection_id = s1.connection_id
    AND s2.version = s1.version + 1
WHERE s1.connection_id = $1
ORDER BY s1.version DESC
LIMIT 10;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-Tenant Identity | 3 | organisations, users, api_keys |
| Database Connectivity | 3 | database_connections, schema_snapshots, introspected_tables + introspected_columns (counted as 2) |
| Introspected Schema | (included above) | introspected_tables, introspected_columns |
| Semantic Layer | 2 | semantic_layers, semantic_entities |
| Security | 1 | security_policies |
| Query Pipeline | 3 | conversations, queries (partitioned), query_explanations |
| Operations | 2 | cost_records, audit_log (partitioned) |
| **Total** | **15** | |

---

## Key Design Decisions

1. **Versioned schema snapshots** — `schema_snapshots` captures point-in-time views of the connected database's structure. Each introspection creates a new snapshot version with `introspected_tables` and `introspected_columns` linked to it. This enables drift detection (comparing snapshots) and ensures the LLM always references an accurate, timestamped schema rather than stale metadata.

2. **Semantic layer as relational entities** — `semantic_entities` stores metrics, dimensions, measures, and synonyms as individual rows rather than a single YAML blob. This enables SQL-based validation (orphaned references, duplicate synonyms), per-entity versioning, and export to dbt MetricFlow YAML format without parsing.

3. **Security policies as first-class entities** — `security_policies` defines row-level filters and column masks that are applied at the NL translation layer, not delegated to the downstream database. This follows NIST SP 800-207 Zero Trust: the query gateway enforces access control regardless of the target database's own RBAC.

4. **Queries as immutable audit records** — The `queries` table is partitioned by `created_at` and serves as the compliance audit trail required by EU AI Act Articles 13-14 and GDPR Article 32. Each row captures the complete lifecycle: NL input, generated SQL, security policies applied, injection detection, execution status, and user feedback.

5. **Injection detection on queries** — `injection_detected` and `injection_details_json` flag P2SQL (prompt-to-SQL) injection attempts per OWASP LLM01. Blocked queries are recorded with `execution_status = 'injection_blocked'` for forensic analysis.

6. **User feedback loop** — `user_feedback` and `edited_sql` on queries capture whether the user accepted, rejected, or edited the generated SQL. This enables accuracy measurement and training data collection for semantic layer auto-generation.

7. **Clause-by-clause explanations** — `query_explanations` breaks down generated SQL into individual clauses with plain-English descriptions. Stored separately from queries to avoid row bloat and to support UI rendering of step-by-step explanations.

8. **Database credentials as vault references** — `database_connections.credential_ref` stores a vault path (e.g., `vault://prod-postgres`) rather than the actual connection password. Credentials are resolved at runtime from the vault.

9. **Synonym GIN index** — `semantic_entities.synonyms` uses a TEXT[] array with a GIN index, enabling fast synonym lookup when the NL input uses business vocabulary that doesn't match column names directly.

10. **Multi-dialect support** — `database_connections.dialect` and `queries.target_dialect` track which SQL dialect is being targeted. The LLM prompt is conditioned on the dialect to generate valid SQL for the specific database engine.
