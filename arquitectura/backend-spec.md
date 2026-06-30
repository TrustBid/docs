# TrustBid — Especificación Preliminar del Backend

> **Estado:** Diseño aprobado — pendiente de implementación.
> **Ubicación en el repo:** `platform/apps/api/` (monorepo `TrustBid/platform`)
> **Stack:** NestJS 11 (monolito modular) · PostgreSQL/Neon (RLS) · BullMQ · Stellar/Soroban · AWS KMS · Cloudflare R2
> **Schema DB:** `platform/apps/api/db/` — `init-db.sql` + `sprint3-schema.sql` + `sprint4-sbt-zk.sql`
> **Coherencia:** diseñado contra [diagrama-secuencia.md](./diagrama-secuencia.md) ·
> [flujos-integraciones-stellar.md](./flujos-integraciones-stellar.md) ·
> [der.md](./der.md) · [diagrama-clases.md](./diagrama-clases.md) ·
> [casos-de-uso.md](./casos-de-uso.md)

---

## 🤖 Contexto para agentes / IA (leer antes de modificar)

1. **Multi-tenant obligatorio:** toda operación autenticada lleva `organization_id` extraído
   del JWT. El interceptor RLS lo propaga al contexto de DB antes de cualquier query.
   Nunca se resuelve `organization_id` por separado ni se acepta en el body del request.
2. **Error format global:** todos los endpoints, sin excepción, serializan errores como
   `{ error: { code: string, message: string } }`. El frontend lo asume en `body?.error?.message`.
3. **Append-only:** transacciones y badges confirmados no se editan ni borran. Una corrección
   es un nuevo registro. Ver principios en diagrama-secuencia.md sección "Contexto para agentes".
4. **Async ≠ confirmado:** un `201` de este backend confirma _ingestión_, no _ejecución_
   on-chain. El estado `confirmed` solo lo escribe el Worker Indexador tras ver la tx en el ledger.
5. **Idempotencia:** toda escritura externa (SDP, Soroban) usa un `memo_id = UUID` generado
   por TrustBid _antes_ de la llamada. Un reintento nunca duplica.
6. **Blockchain invisible:** ninguna respuesta del backend filtra terminología cripto al usuario
   final (`wallet`, `hash`, `USDC`, `Stellar`). Se usa `verificationCode`, `account`, `funds`.

---

## 1. Contexto y decisiones de arquitectura

### 1.1 Relación con el frontend

El frontend (Next.js 15 en `TrustBid-DApp/`) tiene dos rutas de datos hacia este backend:

| Ruta | Quién la usa | Cómo llega al backend |
|---|---|---|
| **Server Components** | Next.js SSR | `repository.ts` → `BACKEND_URL` directo |
| **Client Components** | Navegador | `/api/public/*` (Route Handlers Next.js) → `BACKEND_URL` |

Sin `BACKEND_URL` el frontend cae a seed automáticamente — el portal funciona con datos demo.
El contrato de respuesta es el mismo en ambos casos.

### 1.2 Variables de entorno requeridas

```env
# Backend
DATABASE_URL=postgresql://...@neon.tech/trustbid        # Neon PostgreSQL
REDIS_URL=rediss://...@upstash.io:6379                  # Upstash Redis (BullMQ)
STELLAR_NETWORK=testnet                                  # "testnet" | "public"
STELLAR_HORIZON_URL=https://horizon-testnet.stellar.org
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
AWS_KMS_KEY_ID=arn:aws:kms:...                          # Firma server-side
R2_BUCKET=trustbid-evidencias
R2_ACCOUNT_ID=...
R2_ACCESS_KEY_ID=...
R2_SECRET_ACCESS_KEY=...
SDP_BASE_URL=https://sdp.trustbid.com
JWT_SECRET=...
JWT_EXPIRES_IN=8h

# Compartidas con el frontend
NEXT_PUBLIC_STELLAR_NETWORK=testnet
BACKEND_URL=https://api.trustbid.com
```

### 1.3 NestJS vs Next.js

El backend vive en `backend/` como servicio NestJS independiente, desplegado en Railway
(Docker). **No** es Next.js. Los Route Handlers de Next.js (`/api/public/*`) son proxies
ligeros hacia este servicio — no contienen lógica de negocio.

---

## 2. Estructura de módulos

```
platform/apps/api/src/
├── main.ts
├── app.module.ts
│
├── common/                          # Transversales — sin lógica de negocio
│   ├── filters/
│   │   └── http-exception.filter.ts # Serializa TODA excepción → { error: { code, message } }
│   ├── guards/
│   │   ├── jwt-auth.guard.ts        # Verifica JWT en Authorization: Bearer
│   │   └── roles.guard.ts           # @Roles('admin','contador','responsable',...)
│   ├── decorators/
│   │   ├── roles.decorator.ts
│   │   └── org.decorator.ts         # @CurrentOrg() extrae organization_id del JWT
│   └── interceptors/
│       └── rls.interceptor.ts       # SET app.current_organization_id = $1 antes de cada query
│
├── database/
│   ├── database.module.ts           # pg-pool + Neon serverless driver
│   └── migrations/                  # init-db.sql + sprint3-schema.sql + sprint4-sbt-zk.sql
│
├── modules/
│   ├── public/                      # Flujos 3, B, I — sin auth
│   ├── auth/                        # Flujo A — SEP-10 + JWT ← MERGE BLOCKER
│   ├── organizations/               # Flujo C — KYB, onboarding
│   ├── users/                       # Gestión interna de usuarios
│   ├── projects/                    # Dashboard interno
│   ├── transactions/                # Flujo 1 — gastos + OCR
│   ├── disbursements/               # Flujos 2, D — SDP + reconciliación
│   ├── reports/                     # Flujos 4, E — templates + Horizon
│   └── blockchain/                  # Flujos F, G, H — SBT + ZK + Caatinga
│
├── workers/                         # BullMQ — corren como procesos separados
│   ├── ocr.worker.ts                # Flujo 1
│   ├── indexer.worker.ts            # Flujos 1, 2, B
│   ├── reconciliation.worker.ts     # Flujo 2
│   ├── zk.worker.ts                 # Flujo G
│   └── pdf.worker.ts                # Flujo 4
│
└── config/
    ├── stellar.config.ts
    ├── queue.config.ts
    └── jwt.config.ts
```

---

## 3. Módulos — Contratos de API

### 3.1 `public` — Portal público sin autenticación

> Flujos: 3 (verificación donante), B (donación), I (reputación SEP-1 + on-chain)
> Tablas: `projects`, `transactions`, `pipeline_stages`, `impact_indicators`,
> `beneficiaries`, `organization_badges`
> Auth: ninguna

#### `GET /ngo`

```typescript
// Response 200
{
  name: string
  tagline: string
  mission: string
  totals: {
    projects: number
    raisedUsd: number
    spentUsd: number
    beneficiaries: number
  }
  fundUsage: Array<{
    category: string       // project_category enum
    amountUsd: number
  }>
}
```

#### `GET /projects`

```typescript
// Query params opcionales
?q=string          // búsqueda por name o description
?category=string   // project_category

// Response 200
Array<{
  id: string
  name: string
  category: string
  status: 'active' | 'completed' | 'paused'
  summary: string
  budgetTotalUsd: number
  budgetSpentUsd: number
  beneficiariesTarget: number
  beneficiariesReached: number
  currentStage: string          // label de pipeline_stages WHERE id = current_stage_id
}>
```

#### `GET /projects/:id`

```typescript
// Response 200 — detalle completo
{
  // ...todos los campos de ProjectSummary, más:
  description: string
  currency: string               // "USDC"
  pipeline: Array<{
    key: string                  // pipeline_stages.id
    label: string                // pipeline_stages.name
    date: string | null          // pipeline_transitions.created_at | null si pending
    status: 'done' | 'current' | 'pending'
  }>
  traceability: Array<{
    id: string                   // transactions.id
    date: string                 // transactions.confirmed_at (ISO-8601 UTC)
    concept: string              // transactions.concept
    amount: number
    currency: string
    verificationCode: string     // transactions.tx_hash — nunca exponer como "hash"
    status: 'verified' | 'pending'  // tx_status = 'confirmed' → 'verified'
  }>
  impact: Array<{
    label: string                // impact_indicators.name
    target: number               // impact_indicators.target_value
    actual: number               // impact_indicators.actual_value
    unit: string                 // impact_indicators.unit
  }>
}

// Response 404
{ error: { code: 'not_found', message: string } }
```

#### `GET /categories`

```typescript
// Response 200
string[]    // DISTINCT category FROM projects WHERE status != 'archived'
```

#### `POST /donations`

Crea una intención de donación. El `verificationCode` queda `null` hasta que el Worker
Indexador confirma la tx on-chain (flujo async — el frontend ya lo contempla).

```typescript
// Request body
{
  projectId: string          // requerido — UUID
  amountUsd: number          // requerido, positivo, máx 1_000_000
  walletAddress?: string     // dirección G... del donante
  walletProvider?: string    // 'freighter' | 'albedo'
}

// Response 201
{
  id: string                 // UUID — transactions.id
  projectId: string
  amountUsd: number
  status: 'pending'          // siempre pending al crear
  verificationCode: null     // null hasta confirmación on-chain
  createdAt: string          // ISO-8601 UTC
}

// Response 422
{ error: { code: 'validation_error', message: string } }
```

---

### 3.2 `auth` — SEP-10 + JWT ← MERGE BLOCKER

> Flujo: A (identidad y login ONG)
> Tablas: `users`, `user_wallets`, `organizations`
> Auth: ninguna en challenge/token; JWT en los demás

#### `GET /auth/challenge?account=G...`

Genera una transacción Stellar sin firmar (SEP-10 challenge). No se envía a la red.

```typescript
// Query params
account: string    // dirección G... de la cuenta que quiere autenticarse

// Response 200
{
  transaction: string          // XDR base64 — tx con manage_data + time_bounds ±5 min
  network_passphrase: string   // 'Test SDF Network ; September 2015' | 'Public Global...'
}

// Response 400
{ error: { code: 'invalid_account', message: string } }
```

**Implementación del challenge:**
1. Construir `Transaction` con `sequence = 0` (no consume sequence on-chain)
2. `operation: ManageData('trustbid_auth', randomNonce)`
3. `time_bounds: { minTime: now - 5min, maxTime: now + 5min }`
4. Guardar nonce en Redis con TTL de 10 min (`auth:nonce:{account}`)
5. Devolver XDR base64 — el frontend firma con la wallet del usuario

#### `POST /auth/token`

Verifica la firma y emite JWT de sesión.

```typescript
// Request body
{
  transaction: string    // XDR base64 firmado por el usuario
}

// Response 200
{
  token: string          // JWT — payload: { sub: userId, org: organizationId, role: user_role }
}

// Response 400
{ error: { code: 'invalid_transaction', message: string } }
// Response 401
{ error: { code: 'expired_challenge', message: string } }
```

**Implementación de verificación:**
1. Deserializar XDR → extraer `source_account` y firmas
2. Verificar que `time_bounds` no expiraron
3. Verificar que el nonce en Redis existe y coincide (luego borrarlo — single use)
4. Verificar firma criptográfica con la clave pública del `source_account`
5. Buscar o crear `user` + `user_wallet` para esta cuenta
6. Emitir JWT con `{ sub: user.id, org: user.organization_id, role: user.role }`

#### `POST /auth/refresh`

```typescript
// Response 200
{ token: string }
// Response 401
{ error: { code: 'token_expired', message: string } }
```

#### `GET /auth/me`

```typescript
// Response 200
{
  id: string
  name: string
  email: string
  role: user_role
  organizationId: string
  walletAddress: string | null
}
```

---

### 3.3 `organizations` — KYB y onboarding

> Flujo: C (KYC/KYB SEP-9/12)
> Tablas: `organizations`, `users`, `organization_badges`

```
POST   /organizations              → Crea org + user admin (registro inicial)
GET    /organizations/:id          → Perfil org + estado KYB + badges SBT
PUT    /organizations/:id/kyb      → Sube campos SEP-9, dispara proveedor KYB
GET    /organizations/:id/badges   → Lista organization_badges con estado on-chain
```

**`PUT /organizations/:id/kyb` — campos SEP-9:**
```typescript
{
  legalName: string
  country: string              // ISO 3166-1 alpha-2
  registrationNumber: string
  taxId?: string
  website: string
  address: string
  contactName: string
  contactEmail: string
  documents?: string[]         // URLs firmadas R2 de documentos subidos
}
```

---

### 3.4 `transactions` — Gasto + OCR + validación Contador

> Flujo 1 (registro gasto + OCR + anclaje) · UC13, UC15, UC16, UC24
> Tablas: `transactions`, `invoice_ocr`, `activity_events`
> Auth: JWT requerido · Roles: responsable (POST), contador (PATCH invoice-ocr)

```
POST   /transactions               → Responsable registra gasto + sube imagen
GET    /transactions               → Lista filtrada por org/proyecto/estado
GET    /transactions/:id           → Detalle + invoice_ocr asociado

GET    /invoice-ocr                → ?status=extracted — bandeja del Contador
PATCH  /invoice-ocr/:id            → { action: 'validate' | 'reject', rejectionReason? }
```

**`POST /transactions` — flujo completo:**
1. Sube imagen a R2
2. `INSERT transactions (tx_status=pending, memo_id=UUID)`
3. `INSERT invoice_ocr (ocr_status=pending, transaction_id)`
4. Responde `201` al cliente
5. Encola job en `ocr-jobs` (async)

**`PATCH /invoice-ocr/:id` — path `validate`:**
1. `UPDATE invoice_ocr (ocr_status=validated, validated_by, validated_at)`
2. Firma tx Stellar vía AWS KMS (memo = `memo_id`)
3. `UPDATE transactions (tx_status=submitted, tx_hash)`
4. El Worker Indexador completa a `confirmed` (async)

---

### 3.5 `disbursements` — Desembolso + SDP + Saga

> Flujos 2, D · UC11, UC30
> Tablas: `transactions`, `expense_splits`, `funding_sources`, `activity_events`
> Auth: JWT · Roles: admin_regional

```
POST   /disbursements              → Admin Regional aprueba lote → SDP
GET    /disbursements              → Lista por org + estado de reconciliación
GET    /disbursements/:id          → Detalle + estado SDP + estado anchor
```

**`POST /disbursements` — flujo completo:**
1. `INSERT transactions (tx_status=pending, memo_id=UUID, created_by)`
2. `INSERT expense_splits` si aplica distribución multi-proyecto
3. `INSERT activity_events (type=disbursement)`
4. Firma KMS → `POST SDP /disbursements (tenant_id, memo_id)`
5. `UPDATE transactions (tx_status=submitted)`
6. El Worker Reconciliación monitorea SDP + Anchor (async, Saga)

---

### 3.6 `reports` — Templates + Horizon

> Flujos 4, E · UC21, UC27, UC28
> Tablas: `reports`, `report_templates`, `report_attachments`
> Auth: JWT · Roles: contador

```
POST   /reports                    → Genera reporte (template_id, project_id, period)
GET    /reports                    → Lista por org
GET    /reports/:id/download       → URL firmada R2 del PDF/Excel generado
GET    /horizon/transactions       → Auditoría directa desde ledger Horizon (Flujo E)
```

**`POST /reports` — request:**
```typescript
{
  templateId: string             // report_templates.id (eu | usaid | idb | custom)
  projectId: string
  periodStart: string            // YYYY-MM-DD
  periodEnd: string
  title: string
}
```

---

### 3.7 `blockchain` — SBT + ZK + Caatinga

> Flujos F, G, H
> Tablas: `organization_badges`, `zk_proofs`, `activity_events`
> Auth: JWT · Roles: admin (badges/contratos), contador (zk-proofs)

```
# SBT — Flujo F (server-side; también triggereado por cron o evento KYB)
POST   /badges/emit                → Mint SBT vía @caatinga/client + KMS
GET    /badges/:orgId              → Lista organization_badges con estado on-chain

# ZK — Flujo G
POST   /zk-proofs                  → Contador solicita prueba de compliance presupuestario
GET    /zk-proofs/:id              → Estado + proof_artifact_url cuando esté lista

# Contratos — Flujo H (solo admin de plataforma / DevOps)
GET    /contracts/status           → Estado de contratos deployados (caatinga doctor)
```

**`POST /zk-proofs` — request:**
```typescript
{
  credentialType: 'budget_compliance' | 'donor_privacy' | 'impact_threshold' | 'audit_trail'
  projectId: string
  periodStart: string
  periodEnd: string
}
```

---

## 4. Workers BullMQ

Corren como procesos separados. En Railway: un servicio por worker o todos en un mismo
contenedor con `--worker` flag. Comparten la misma DB y Redis que el servidor principal.

| Worker | Cola | Flujo | Responsabilidad |
|---|---|---|---|
| `ocr.worker` | `ocr-jobs` | Flujo 1 | Lee imagen R2 → extrae campos con IA → `UPDATE invoice_ocr (ocr_status=extracted)` |
| `indexer.worker` | polling cron (15s) | Flujos 1, 2, B | Horizon polling por `last_ledger` → confirma tx → `UPDATE transactions (tx_status=confirmed)` + `INSERT activity_events` |
| `reconciliation.worker` | `reconciliation-jobs` | Flujo 2 | Polling SDP `/statistics` + Anchor → Saga: si fiat falla post on-chain, INSERT reversa append-only |
| `zk.worker` | `zk-jobs` | Flujo G | `caatinga zk prove` (Groth16) → sube artefacto a R2 → `caatinga zk invoke` → `UPDATE zk_proofs (status=anchored)` |
| `pdf.worker` | `pdf-jobs` | Flujo 4 | Genera PDF/Excel con template del donante → sube a R2 → `INSERT report_attachments` → `UPDATE reports (status=submitted)` |

---

## 5. Contratos transversales

### 5.1 Error format global

El `HttpExceptionFilter` transforma toda excepción — incluyendo las de validación de class-validator,
las de NestJS y las propias — al formato que el frontend consume:

```typescript
// Formato único — el frontend lee body?.error?.message
{ error: { code: string, message: string } }

// Ejemplos por flujo
{ error: { code: 'not_found',           message: 'Project not found' } }
{ error: { code: 'validation_error',    message: 'amountUsd must be positive' } }
{ error: { code: 'invalid_transaction', message: 'SEP-10 signature mismatch' } }
{ error: { code: 'expired_challenge',   message: 'Challenge expired — request a new one' } }
{ error: { code: 'forbidden',           message: 'Role contador required' } }
{ error: { code: 'conflict',            message: 'Badge kyb_verified already issued' } }
```

### 5.2 Multi-tenancy y RLS

El `RlsInterceptor` se ejecuta en cada request autenticado:

```sql
-- Antes de cualquier query en el request
SET app.current_organization_id = '<organization_id del JWT>';
```

Las políticas RLS en PostgreSQL ya filtran por esta variable (ver `sprint3-schema.sql`).
Ningún endpoint acepta `organization_id` en el body ni en query params — siempre viene del JWT.

### 5.3 Idempotencia

Toda operación que escribe en Stellar genera un `memo_id = UUID` **antes** de llamar
a Horizon/SDP/Soroban. El campo tiene restricción `UNIQUE` en `transactions.memo_id`.
Un reintento de red no genera doble registro.

### 5.4 Principio append-only

```
transactions (tx_status = confirmed)   → nunca UPDATE ni DELETE
organization_badges (status = issued)  → revocación = nuevo registro status=revoked
zk_proofs (status = anchored)          → inmutable
activity_events                        → solo INSERT, nunca UPDATE
```

### 5.5 SEP-10 — consideraciones de seguridad

- El challenge XDR nunca se envía a la red Stellar — solo se firma localmente.
- El nonce se almacena en Redis con TTL de 10 min y se invalida tras un uso (single-use).
- Los `time_bounds` del challenge son ±5 min — el servidor los verifica en `/auth/token`.
- La clave privada del usuario **nunca llega al backend** — solo la tx firmada.

---

## 6. Trazabilidad flujos → módulos

| Flujo | Módulo(s) backend | Workers | Tablas principales |
|---|---|---|---|
| **1** Gasto + OCR + anclaje | `transactions` | `ocr`, `indexer` | transactions, invoice_ocr, activity_events |
| **2** Desembolso + Saga | `disbursements` | `reconciliation`, `indexer` | transactions, expense_splits, activity_events |
| **3** Verificación donante | `public` | — | projects, transactions, pipeline_stages |
| **4** Reporte template | `reports` | `pdf` | reports, report_templates, report_attachments |
| **A** Login SEP-10 | `auth` | — | users, user_wallets |
| **B** Donación wallet | `public` | `indexer` | transactions |
| **C** KYC/KYB | `organizations` | — | organizations, organization_badges |
| **D** Desembolso SDP | `disbursements` | `reconciliation` | transactions, expense_splits |
| **E** Auditoría Horizon | `reports` | — | (Horizon API directo) |
| **F** Emisión SBT | `blockchain` | — | organization_badges, activity_events |
| **G** Prueba ZK | `blockchain` | `zk` | zk_proofs |
| **H** Deploy contratos | `blockchain` (admin) | — | config (contract_ids) |
| **I** Verificación pública | `public` | — | organization_badges |

---

## 7. Orden de construcción (sprints)

| Sprint | Módulos / Workers | Desbloquea |
|---|---|---|
| **0 — ahora** | `common/filters` · `auth` (SEP-10 completo) | Merge blocker — login real con wallet |
| **1** | `public` completo (todos los endpoints del merge spec) | Portal público con datos reales; cierra el merge |
| **2** | `transactions` · `ocr.worker` · `indexer.worker` | Flujo 1 — volumen diario más alto; conecta el dashboard de gastos |
| **3** | `disbursements` · `reconciliation.worker` | Flujos 2 y D — desembolsos SDP |
| **4** | `reports` · `pdf.worker` | Flujo 4 — la UI ya existe, falta el endpoint |
| **5** | `organizations` (KYB completo) · `blockchain` (SBT + ZK) | Flujos C, F, G — reputación on-chain |

---

## 8. Lo que el backend NO implementa en el merge

- Endpoints de dashboard interno (proyectos, reportes, settings) — usan datos mock.
- Procesamiento real de tx Stellar en `POST /donations` — `verificationCode` puede ser `null`;
  el Worker Indexador lo actualiza when confirma on-chain.
- KYC/KYB — Fase 2 (Sprint 5).
- Gestión de usuarios interna — Sprint 2+.
