# TrustBid — Diagrama Entidad-Relación (DER)

> 23 tablas — Sprint 1+2 (`init-db.sql`) + Sprint 3 (`sprint3-schema.sql`).
> Dividido en 3 vistas por dominio. El diagrama de clases (`diagrama-clases.md`) muestra la visión OOP.

---

## Vista 1 — Identidad y Jerarquía

Entidades: `organizations`, `users`, `user_wallets`, `plans`, `programs`, `areas`, `projects`

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#dbeafe', 'primaryTextColor': '#1e3a5f', 'primaryBorderColor': '#3b82f6', 'lineColor': '#475569', 'fontSize': '11px'}}}%%
erDiagram

    organizations {
        uuid id PK
        varchar name
        varchar slug UK
        char country
        varchar wallet_address
        varchar stellar_network
        jsonb settings
        timestamptz created_at
        timestamptz updated_at
    }

    users {
        uuid id PK
        uuid organization_id FK
        citext email UK
        varchar name
        varchar phone
        user_role role
        boolean is_active
        timestamptz last_login_at
        timestamptz created_at
    }

    user_wallets {
        uuid id PK
        uuid user_id FK
        uuid organization_id FK
        wallet_provider provider
        varchar public_key
        boolean is_primary
        timestamptz connected_at
        timestamptz disconnected_at
    }

    plans {
        uuid id PK
        uuid organization_id FK
        varchar name
        text description
        date start_date
        date end_date
        uuid created_by FK
        timestamptz created_at
    }

    programs {
        uuid id PK
        uuid organization_id FK
        uuid plan_id FK
        varchar name
        text description
        date start_date
        date end_date
        uuid created_by FK
        timestamptz created_at
    }

    areas {
        uuid id PK
        uuid organization_id FK
        uuid parent_area_id FK
        varchar name
        smallint level
        uuid responsable_id FK
        timestamptz created_at
    }

    projects {
        uuid id PK
        uuid organization_id FK
        uuid program_id FK
        varchar name
        text description
        project_category category
        project_status status
        numeric budget_amount
        asset_code budget_asset
        numeric spent_amount
        date start_date
        date end_date
        boolean blockchain_enabled
        varchar stellar_contract_id
        uuid created_by FK
        uuid current_stage_id FK
        timestamptz created_at
    }

    organizations ||--o{ users : "has"
    organizations ||--o{ plans : "has"
    organizations ||--o{ areas : "owns"
    organizations ||--o{ projects : "manages"

    users ||--o{ user_wallets : "connects"
    users }o--o{ areas : "responsable"

    plans ||--o{ programs : "contains"
    programs ||--o{ projects : "groups"

    areas ||--o{ areas : "parent of"
```

---

## Vista 2 — Operaciones Financieras

Entidades: `projects` (ref), `areas` (ref), `accounts`, `custodian_keys`, `funding_sources`, `transactions`, `expense_splits`, `indexer_state`

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#d1fae5', 'primaryTextColor': '#064e3b', 'primaryBorderColor': '#10b981', 'lineColor': '#475569', 'fontSize': '11px'}}}%%
erDiagram

    projects {
        uuid id PK
        varchar name
        project_status status
        numeric budget_amount
        asset_code budget_asset
        uuid program_id FK
        uuid current_stage_id FK
    }

    areas {
        uuid id PK
        uuid organization_id FK
        uuid parent_area_id FK
        varchar name
        smallint level
        uuid responsable_id FK
    }

    accounts {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        uuid area_id FK
        varchar name
        varchar wallet_address
        numeric budget_amount
        numeric spent_amount
        asset_code asset_code
        jsonb categories
        timestamptz created_at
    }

    custodian_keys {
        uuid id PK
        uuid organization_id FK
        uuid account_id FK
        varchar public_key
        varchar kms_key_id
        varchar kms_key_version
        varchar key_type
        timestamptz created_at
    }

    funding_sources {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        varchar name
        funding_source_type funder_type
        numeric amount
        asset_code asset_code
        date received_at
        text notes
        timestamptz created_at
    }

    transactions {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        uuid account_id FK
        uuid area_id FK
        varchar beneficiary
        varchar concept
        varchar category
        numeric amount
        asset_code asset_code
        varchar memo_id UK
        varchar tx_hash
        tx_status tx_status
        bigint stellar_ledger
        varchar support_file_path
        uuid created_by FK
        timestamptz created_at
        timestamptz confirmed_at
    }

    expense_splits {
        uuid id PK
        uuid transaction_id FK
        uuid project_id FK
        uuid funding_source_id FK
        numeric amount
        numeric percentage
        text notes
    }

    indexer_state {
        smallint id PK
        bigint last_ledger
        timestamptz last_sync
        varchar status
        text error_message
    }

    projects ||--o{ accounts : "allocates"
    projects ||--o{ funding_sources : "funded by"
    projects ||--o{ transactions : "tracks"
    projects ||--o{ expense_splits : "receives"

    areas }o--o{ accounts : "associated"
    areas }o--o{ transactions : "associated"

    accounts ||--o{ transactions : "records"
    accounts ||--o| custodian_keys : "managed by"

    funding_sources ||--o{ expense_splits : "sourced by"
    transactions ||--o{ expense_splits : "split into"

    indexer_state }o..o{ transactions : "indexes"
```

---

## Vista 3 — Pipeline, Impacto y Reportes

Entidades: `projects` (ref), `users` (ref), `pipeline_templates`, `pipeline_template_stages`, `pipeline_stages`, `pipeline_transitions`, `impact_indicators`, `beneficiaries`, `report_templates`, `reports`, `report_attachments`, `activity_events`

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'primaryColor': '#fef3c7', 'primaryTextColor': '#451a03', 'primaryBorderColor': '#f59e0b', 'lineColor': '#475569', 'fontSize': '11px'}}}%%
erDiagram

    projects {
        uuid id PK
        varchar name
        project_status status
        uuid current_stage_id FK
    }

    users {
        uuid id PK
        varchar name
        user_role role
    }

    pipeline_templates {
        uuid id PK
        uuid organization_id FK
        varchar name
        boolean is_default
        uuid created_by FK
        timestamptz created_at
    }

    pipeline_template_stages {
        uuid id PK
        uuid template_id FK
        varchar name
        smallint order_index
        boolean notify_donor
        text notification_msg
    }

    pipeline_stages {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        uuid template_stage_id FK
        varchar name
        smallint order_index
        boolean notify_donor
        text notification_msg
        timestamptz created_at
    }

    pipeline_transitions {
        uuid id PK
        uuid project_id FK
        uuid from_stage_id FK
        uuid to_stage_id FK
        uuid transitioned_by FK
        text notes
        timestamptz created_at
    }

    impact_indicators {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        varchar name
        varchar unit
        numeric target_value
        numeric actual_value
        date recorded_at
        text evidence_url
        uuid recorded_by FK
        timestamptz created_at
    }

    beneficiaries {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        integer count
        text description
        text evidence_url
        date recorded_at
        uuid recorded_by FK
        timestamptz created_at
    }

    report_templates {
        uuid id PK
        uuid organization_id FK
        varchar name
        template_format format
        jsonb schema_definition
        boolean is_default
        uuid created_by FK
        timestamptz created_at
    }

    reports {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        uuid template_id FK
        report_type report_type
        report_status status
        varchar title
        date period_start
        date period_end
        numeric funds_used_amount
        numeric milestone_progress
        uuid submitted_by FK
        timestamptz created_at
    }

    report_attachments {
        uuid id PK
        uuid report_id FK
        uuid organization_id FK
        varchar file_name
        varchar file_path
        varchar mime_type
        bigint size_bytes
        uuid uploaded_by FK
        timestamptz uploaded_at
    }

    activity_events {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        activity_type event_type
        text description
        varchar tx_hash
        varchar reference_table
        uuid reference_id
        uuid created_by FK
        timestamptz created_at
    }

    pipeline_templates ||--o{ pipeline_template_stages : "defines"
    pipeline_template_stages ||--o{ pipeline_stages : "cloned to"

    projects ||--o{ pipeline_stages : "has"
    projects }o--o| pipeline_stages : "current stage"
    projects ||--o{ pipeline_transitions : "logs"
    projects ||--o{ impact_indicators : "measures"
    projects ||--o{ beneficiaries : "serves"
    projects ||--o{ reports : "generates"
    projects ||--o{ activity_events : "emits"

    pipeline_stages ||--o{ pipeline_transitions : "from"
    pipeline_stages ||--o{ pipeline_transitions : "to"
    users }o--o{ pipeline_transitions : "transitions"

    report_templates ||--o{ reports : "used by"
    reports ||--o{ report_attachments : "includes"
    users }o--o{ reports : "submits"
    users }o--o{ activity_events : "generates"
```

---

## Enums del schema

| Enum | Valores |
|---|---|
| `user_role` | admin, responsable, donante |
| `wallet_provider` | freighter, albedo, custodial |
| `project_status` | draft, active, paused, completed, archived |
| `project_category` | infrastructure, education, health, technology, environment, social, other |
| `asset_code` | XLM, USDC |
| `tx_status` | pending, submitted, confirmed, failed |
| `report_type` | financial, milestone, audit |
| `report_status` | draft, submitted, approved, rejected |
| `activity_type` | verification, disbursement, expense, report, project |
| `funding_source_type` | international_org, government, corporate, individual, event, other |
| `template_format` | eu, usaid, idb, custom |

## Convenciones

| Notación | Significado |
|---|---|
| `PK` | Primary key |
| `FK` | Foreign key |
| `UK` | Unique constraint |
| `||--o{` | Uno a muchos (obligatorio → opcional) |
| `}o--o|` | Muchos opcional a uno opcional |
| `}o..o{` | Relación conceptual sin FK directa |
