# Vista Lógica — TrustBid

> **4+1 · Vista 1 de 5** · [← Índice](./README.md) · [Siguiente: Procesos →](./2-vista-procesos.md)
> Responde: *¿qué hace el sistema y cuál es su dominio?*
> Interesado: analista funcional, usuario final, auditor.

---

## 1. Descomposición funcional de alto nivel

```mermaid
graph TB
    subgraph PUB["🌐 Dominio Público (sin cuenta)"]
        P1[Explorar proyectos]
        P2[Ver trazabilidad de fondos]
        P3[Donar con wallet / SEP-7]
        P4[Verificar reputación on-chain]
    end

    subgraph ORG["🏢 Dominio Organización (autenticado)"]
        O1[Gestión de proyectos]
        O2[Rendición de gastos]
        O3[Aprobación · doble control]
        O4[Reportes a donantes]
        O5[Perfil e integraciones]
        O6[Invitaciones de voluntarios]
    end

    subgraph CAMPO["📱 Dominio Campo (bot)"]
        C1[Enrolarse con código ALTA-XXXX]
        C2[Enviar foto de factura]
        C3[Confirmar monto]
        C4[Recibir hash on-chain]
    end

    subgraph CHAIN["⛓️ Dominio On-Chain"]
        B1[Asignación presupuestaria]
        B2[Anclaje de gasto/reporte]
        B3[SBT de reputación]
    end

    O1 --> B1
    O3 --> B2
    C2 --> O3
    C4 -.notificación.- B2
    P2 --> B2
    P4 --> B3
    O6 --> C1

    style PUB fill:#dbeafe,stroke:#1e40af
    style ORG fill:#dcfce7,stroke:#166534
    style CAMPO fill:#fae8ff,stroke:#86198f
    style CHAIN fill:#fef3c7,stroke:#a16207
```

## 2. Actores y roles

Los roles viven en la columna `users.role` y se aplican con `@Roles(...)` +
`RolesGuard` (`apps/api/src/common/guards/roles.guard.ts`).

| Rol | Origen | Puede | Etiqueta en UI |
|---|---|---|---|
| `admin` | bootstrap de la org al primer login | todo; crear/aprobar gastos (auto-anclaje); emitir invitaciones; mint/revoke de SBT | Administrador |
| `admin_regional` | asignación manual | aprueba gastos; emite invitaciones de bot | — |
| `auditor` | asignación manual | aprueba/rechaza gastos; lectura amplia | Auditor (solo lectura en UI) |
| `contador` | asignación manual | carga gastos (quedan `pending`) | Contador |
| `responsable` | asignación manual | carga gastos de su área | Resp. de Área |
| `voluntario` | auto-creado al enrolarse por bot | carga gastos desde WhatsApp/Telegram (siempre `pending`) | — |
| `donante` | enum del schema | rol previsto, sin flujo autenticado propio hoy | Donante |
| Público anónimo | — | consulta proyectos, trazabilidad, badges; inicia donación | — |

> ⚠️ **Trampa de rol en el registro.** `RegistrationDto` acepta `role` del cliente y
> `findOrCreateUser` lo persiste tal cual (`auth.service.ts:361`). Quien cree su organización
> eligiendo `responsable` o `donante` **queda sin permisos de `admin` en su propia ONG**
> —no puede editar la organización, emitir invitaciones de bot ni badges— y no existe flujo
> para corregirlo. Además el enum del DTO tiene 3 valores (`admin`, `responsable`, `donante`)
> mientras el de Postgres ya tiene 6: registrarse como `contador` o `auditor` es rechazado por
> la validación aunque la base los admita.

**Regla estructural del dominio** (`projects.service.ts:14`):
`APPROVER_ROLES = ['admin', 'admin_regional', 'auditor']`. Un gasto cargado por uno de estos
roles se **auto-autoriza** y ancla al instante; cargado por cualquier otro queda `pending` y
requiere un aprobador **distinto** del creador.

## 3. Modelo de dominio (clases de servicio)

```mermaid
classDiagram
    direction LR

    class AuthService {
        +generateChallenge(account) ChallengeXDR
        +verifyAndIssueToken(xdr, registration) JWT
        +loginWithPrivy(privyToken, registration) JWT
        +refresh(bearer) JWT
        +getMe(userId) / updateMe(userId, dto)
        -findOrCreateUser() bootstrap org+user+wallet
    }
    class PrivyService {
        +verifyAndEnsureStellarWallet(token)
        -findStellarAddress(linkedAccounts)
    }
    class ProjectsService {
        +listByOrg / getById / create / update
        +getTransactions / getTransactionDetail
        +extractInvoice(file) InvoiceExtraction
        +createTransaction(...) TxResult
        +approveTransaction(...) doble control
        +rejectTransaction(...)
        +getPipelineStages / getRecentActivity
        -anchorTransactionAsync() fire-and-forget
    }
    class ReportsService {
        +listByOrg / create
        +getOnChainExpense
        -anchorReportOnChain()
    }
    class BadgesService {
        +listByOrganization(orgId)
        +mint(dto, issuedBy)
        +revoke(tokenId, orgId)
    }
    class PublicService {
        +getNgo / getProjects / getProject
        +getCategories
        +createDonation(dto) memo + SEP-7
        +getDonation(id)
    }
    class OrganizationsService {
        +getOrg / updateOrg / listUsers
        +getSettingsIntegrations
        +getOrganization / updateOrganization
        +getLookups() áreas·poblaciones·ODS
    }
    class SorobanService {
        +allocateFunds(projectId, xlm, caller)
        +anchorExpense(opts) / anchorExpenseWithRetry
        +mintBadge / revokeBadge / readBadges
        +readAllocation / readExpense
    }
    class StorageService {
        +enabled bool
        +putInvoice(buffer, sha256, mime)
        +getSignedUrl(key, ttl)
    }
    class GeminiService {
        +enabled bool
        +extractInvoice(buffer, mime)
    }
    class BotFlowService {
        +handleMessage(channel, msg)
        -handleImage / handleText
        -createPending()
        -resolveEnrollment / resolveProject
    }
    class EnrollmentService {
        +createInvite / listInvites / revokeInvite
        +tryEnrollByCode(channel, userId, code)
    }
    class HorizonWatcherService {
        +watchDonation(job)
    }

    AuthService --> PrivyService
    ProjectsService --> SorobanService
    ProjectsService --> StorageService
    ProjectsService --> GeminiService
    ReportsService --> SorobanService
    BadgesService --> SorobanService
    PublicService --> HorizonWatcherService
    BotFlowService --> ProjectsService
    BotFlowService --> GeminiService
    BotFlowService --> EnrollmentService
```

### Servicios de soporte

| Servicio | Rol | Degradación |
|---|---|---|
| `ConversationService` | estado efímero del voluntario en Redis (`bot:conv:{canal}:{id}`, TTL 30 min) | requiere Redis |
| `BotNotificationService` | escucha `transaction.anchored` y avisa el hash por el canal correcto | silencioso si el canal está deshabilitado |
| `WhatsappService` / `TelegramService` | implementan la interfaz `BotChannel` (`sendText`, `downloadMedia`) | `enabled=false` sin credenciales |
| `HorizonWatcherProcessor` | worker BullMQ que busca el memo de la donación en Horizon | requiere Redis |
| `AreasService` | áreas presupuestarias de la organización (`areas`, con `budget_amount`); las transacciones se imputan por `area_id` | — |
| `PipelineTemplatesService` | plantillas de pipeline reutilizables: crear, editar, **duplicar** y borrar | — |
| `BillingService` | planes, suscripción vigente, historial de pagos y uso (proyectos/usuarios/tx/reportes) | **sin pasarela de pago** — ver abajo |

### La abstracción `BotChannel`

El punto de diseño más limpio del bot (`modules/whatsapp/bot-channel.ts`): un único
`BotFlowService` sirve a los dos canales porque ambos implementan el mismo contrato.

```mermaid
classDiagram
    class BotChannel {
        <<interface>>
        +kind : whatsapp o telegram
        +enabled : boolean
        +sendText(userId, body)
        +downloadMedia(mediaId) Buffer y mime
    }
    class IncomingMessage {
        <<type>>
        +channel · userId
        +type : image, text u other
        +mediaId? · text? · name?
    }
    BotChannel <|.. WhatsappService
    BotChannel <|.. TelegramService
    BotFlowService ..> BotChannel : usa
    BotFlowService ..> IncomingMessage : normaliza
```

Simétricamente, `StellarSigner` (`packages/stellar-sdk/src/signer.ts`) unifica los dos
rieles de firma: `WalletKitSigner` (browser, Freighter/Albedo) y `PrivyStellarSigner`
(server, `rawSign` Tier 2). La lógica de negocio no distingue el riel.

## 4. Contratos Soroban — modelo lógico on-chain

```mermaid
classDiagram
    class FundTracker {
        <<contract>>
        +initialize(admin)
        +allocate(caller, project_id, amount_xlm)
        +get_allocation(project_id) Option~FundAllocation~
    }
    class FundAllocation {
        +project_id: Symbol
        +organization: Address
        +amount_xlm: i128
        +allocated_at: u64
    }
    class ExpenseAnchor {
        <<contract>>
        +initialize(admin)
        +anchor(caller, expense_id, project_id, amount_xlm, receipt_hash)
        +get_expense(expense_id) Option~AnchoredExpense~
    }
    class AnchoredExpense {
        +expense_id: Symbol
        +project_id: Symbol
        +submitted_by: Address
        +amount_xlm: i128
        +receipt_hash: Bytes  SHA-256 del comprobante en R2
        +anchored_at: u64
    }
    class SbtBadge {
        <<contract>>
        +initialize(admin) once
        +mint_badge(organization, badge_type) u64
        +revoke_badge(token_id)
        +get_badge(token_id) Option~Badge~
        +get_badges(organization) Vec~Badge~
        +get_active_badges(organization) Vec~Badge~
    }
    class Badge {
        +token_id: u64
        +badge_type: Symbol
        +organization: Address
        +status: BadgeStatus
        +issued_at: u64
        +revoked_at: u64
    }
    FundTracker --> FundAllocation : persistent
    ExpenseAnchor --> AnchoredExpense : persistent + event
    SbtBadge --> Badge : persistent + index org→ids
```

**Invariantes verificadas por el código Rust:**

| Contrato | Invariante | Implementación |
|---|---|---|
| `fund-tracker` | re-asignar sobrescribe (última asignación gana) | clave `DataKey::Allocation(project_id)` |
| `fund-tracker` | `initialize` es idempotente (no panica) | no chequea existencia previa |
| `expense-anchor` | cada anclaje emite el evento `expense_anchored` | `env.events().publish(...)` |
| `expense-anchor` | re-anclar el mismo `expense_id` sobrescribe | clave `DataKey::Expense(expense_id)` |
| `sbt-badge` | `initialize` **sólo una vez** (panica `already_initialized`) | chequeo de `DataKey::Admin` |
| `sbt-badge` | `token_id` autoincremental desde 1 | `DataKey::NextTokenId` |
| `sbt-badge` | mint/revoke sólo el admin (`require_admin`) | `admin.require_auth()` |
| `sbt-badge` | tipos válidos: `kyb_verified`, `transparency_{bronze,silver,gold}` | `validate_badge_type`, panica si no |
| `sbt-badge` | **soulbound**: no existe función de transferencia | ausencia deliberada de `transfer` |
| `sbt-badge` | revocar no borra: `status=Revoked` + `revoked_at`, `issued_at` intacto | append-only de hecho |

> ⚠️ **Limitación conocida**: `fund-tracker` acepta `amount_xlm` negativo (hay un test que lo
> documenta: `test_negative_amount_accepted`). La validación de rango vive hoy sólo en el DTO
> de la API, no en el contrato.

### Codificación de identificadores

Los IDs de PostgreSQL son UUID (36 chars); los `Symbol` de Soroban admiten hasta 32
caracteres y en la práctica se usan cortos. `SorobanService` los reduce así
(`soroban.service.ts:107`):

```ts
const symId = projectId.replace(/-/g, '').slice(-12);   // últimos 12 hex del UUID
const amountRaw = BigInt(Math.round(amountXlm * 1e7));  // stroops (7 decimales)
```

> Consecuencia: la clave on-chain es un **sufijo** del UUID, no el UUID. La probabilidad de
> colisión es baja (48 bits) pero no nula, y no hay detección de colisión.

## 5. Máquinas de estado del dominio

### 5.1 Transacción (gasto/donación) — `transactions.tx_status`

```mermaid
stateDiagram-v2
    [*] --> pending: createTransaction()<br/>rol NO aprobador o bot
    [*] --> submitted: createTransaction()<br/>rol aprobador (auto-autorizado)
    [*] --> pending2: createDonation() sin txHash
    [*] --> submitted2: createDonation() con txHash

    pending --> submitted: approveTransaction()<br/>aprobador ≠ creador
    pending --> failed: rejectTransaction()

    submitted --> confirmed: anclaje Soroban OK<br/>(tx_hash + confirmed_at)
    submitted --> failed: anclaje agotó reintentos

    pending2 --> confirmed: memo hallado en Horizon
    pending2 --> expired: 30 min sin aparecer

    confirmed --> [*]
    failed --> [*]
    expired --> [*]

    note right of pending
        Doble control: el aprobador
        NO puede ser el creador
        (ForbiddenException self_approval)
    end note
```

`pending2`/`submitted2` son el mismo estado `pending`/`submitted` del enum; se separan en el
diagrama para distinguir el camino donación (vigilado por Horizon) del camino gasto
(anclado en Soroban).

### 5.2 Proyecto y reporte — `blockchain_status`

```mermaid
stateDiagram-v2
    [*] --> sin_estado: creación sin blockchain
    [*] --> anchored: allocate() devolvió hash
    [*] --> failed: allocate() falló
    anchored --> anchored: cambio de presupuesto → re-allocate
    anchored --> failed: re-allocate falló
    failed --> anchored: reintento exitoso
```

Reportes agregan un estado `pending` explícito, escrito antes de invocar el contrato
(`reports.service.ts:120`).

### 5.3 SBT de reputación

```mermaid
stateDiagram-v2
    [*] --> pending: INSERT organization_badges
    pending --> issued: mint_badge() OK<br/>token_id + anchor_tx_hash
    pending --> revoked: mint_badge() falló
    issued --> revoked: revoke_badge() OK
    revoked --> [*]
```

> Nota de dominio: cuando el mint falla, la fila se marca `revoked` (no existe estado
> `failed` en el enum `badge_status`), lo que mezcla "nunca se emitió" con "se revocó".

### 5.4 Conversación del bot

```mermaid
stateDiagram-v2
    [*] --> sin_enrolar
    sin_enrolar --> enrolado: mensaje con ALTA-XXXX válido
    enrolado --> awaiting_code: llega imagen → OCR Gemini<br/>(estado en Redis, TTL 30 min)
    awaiting_code --> awaiting_code: "monto 250" corrige el monto
    awaiting_code --> registrado: código de proyecto válido
    enrolado --> registrado: proyecto por defecto + monto detectado<br/>(atajo de invitación por-proyecto)
    registrado --> enrolado: conv.clear()
```

## 6. Modelo de datos y multi-tenancy

### 6.1 Núcleo del modelo

```mermaid
erDiagram
    ORGANIZATIONS ||--o{ USERS : emplea
    ORGANIZATIONS ||--o{ USER_WALLETS : registra
    ORGANIZATIONS ||--o{ PROJECTS : ejecuta
    ORGANIZATIONS ||--o{ TRANSACTIONS : mueve
    ORGANIZATIONS ||--o{ REPORTS : emite
    ORGANIZATIONS ||--o{ ORGANIZATION_BADGES : acredita
    ORGANIZATIONS ||--o{ BOT_INVITES : genera
    ORGANIZATIONS ||--o{ BOT_ENROLLMENTS : habilita

    PROJECTS ||--o{ TRANSACTIONS : imputa
    PROJECTS ||--o{ REPORTS : documenta
    PROJECTS ||--o{ PIPELINE_STAGES : recorre
    PROJECTS ||--o{ IMPACT_INDICATORS : mide
    PROJECTS ||--o{ BENEFICIARIES : alcanza
    PROJECTS ||--o{ FUNDING_SOURCES : financia

    USERS ||--o{ TRANSACTIONS : "created_by"
    USERS ||--o{ TRANSACTIONS : "approved_by"
    USERS ||--o| BOT_ENROLLMENTS : "usuario voluntario"
    BOT_INVITES }o--o| PROJECTS : "default_project_id"

    ORGANIZATIONS {
        uuid id PK
        text name
        text slug UK
        text wallet_address
        text stellar_network
        char country
        jsonb settings
        text legal_name "sprint6"
        text fiscal_id "sprint6"
    }
    PROJECTS {
        uuid id PK
        uuid organization_id FK
        text name
        varchar code "sprint11 · código para el bot"
        enum category
        enum status
        numeric budget_amount
        numeric spent_amount
        bool blockchain_enabled
        text allocation_tx_hash "sprint7"
        varchar blockchain_status "sprint-review"
        text image_url "sprint8"
    }
    TRANSACTIONS {
        uuid id PK
        uuid organization_id FK
        uuid project_id FK
        text memo_id UK "PAY-YYYY-NNNN"
        numeric amount
        enum tx_status
        text tx_hash
        text support_file_hash "SHA-256"
        text storage_key "clave R2"
        text settlement_type "sprint10"
        jsonb ai_extracted "sprint10"
        numeric ai_amount "sprint10"
        bool ai_match "sprint10"
        numeric ai_confidence "sprint10"
        uuid approved_by FK "sprint10"
        varchar submitter_phone "sprint11"
        varchar submitter_channel "sprint13"
    }
    BOT_ENROLLMENTS {
        uuid id PK
        varchar channel "whatsapp o telegram"
        varchar channel_user_id
        uuid organization_id FK
        uuid user_id FK
        uuid default_project_id FK "sprint13"
        varchar status
    }
    BOT_INVITES {
        uuid id PK
        varchar code UK "ALTA-XXXX"
        uuid organization_id FK
        uuid project_id FK "sprint13"
        int max_uses
        int uses
        timestamp expires_at
        varchar status
    }
```

El DER completo (25+ tablas en 4 vistas de dominio) está en [der.md](../der.md).

### 6.2 Evolución del schema

| Migración | Aporte |
|---|---|
| `init-db.sql` | núcleo: organizations, users, user_wallets, projects, accounts, transactions, reports, report_attachments, activity_events, custodian_keys, indexer_state + RLS |
| `sprint3-schema.sql` | planes, programas, áreas, fuentes de fondeo, splits, pipeline (templates/stages/transitions), indicadores de impacto, beneficiarios, OCR, templates de reporte |
| `sprint4-sbt-zk.sql` | `organization_badges`, `zk_proofs` + enums de badge y ZK |
| `sprint5-wallet-auth.sql` | ampliación del enum `wallet_provider` (xbull, rabet, lobstr, hana, hot-wallet, privy) |
| `sprint6-org-profile.sql` | perfil extendido de ONG + catálogos `intervention_areas`, `target_populations`, `ods_goals` y sus tablas puente |
| `sprint7-anchor-txhash.sql` | `reports.anchor_tx_hash`, `projects.allocation_tx_hash` |
| `sprint8-project-image.sql` | `projects.image_url` |
| `sprint9-transaction-invoice.sql` | `invoice_number`, `tax_id`, `invoice_date` |
| `sprint10-invoice-veracity.sql` | veracidad IA + doble control: `settlement_type`, `storage_key`, `ai_*`, `approved_by/at` |
| `sprint11-whatsapp-bot.sql` | `projects.code`, `transactions.submitter_phone`, `bot_enrollments` |
| `sprint12-bot-invites.sql` | `bot_invites` con código, cupo y vencimiento |
| `sprint13-project-invites-channels.sql` | invitación por proyecto + canal (`telegram`) |
| `sprint-review-blockchain-status.sql` | `blockchain_status` en `projects` y `reports` |
| `sprint14-settings-areas-notifications.sql` | `areas` gana `description` y `budget_amount`; `transactions.area_id` (+ índice); tabla `notification_preferences` |
| `sprint15-settings-profile-invites-billing.sql` | `organizations`: `mission`, `logo_url`, `timezone` (default `America/Bogota`), `language`; tablas `user_invites`, `subscription_plans`, `organization_subscriptions`, `subscription_payments` + seed de planes |

### 6.3 Multi-tenancy: lo declarado vs. lo aplicado

```mermaid
graph LR
    subgraph DECLARADO["Declarado en init-db.sql"]
        R1["ALTER TABLE … ENABLE ROW LEVEL SECURITY<br/>(10 tablas)"]
        R2["CREATE POLICY … USING<br/>organization_id = current_setting('app.current_organization_id')"]
    end
    subgraph APLICADO["Aplicado en runtime"]
        A1["JWT payload: sub · org · role"]
        A2["@Org() decorator → orgId"]
        A3["WHERE organization_id = $1<br/>en cada query"]
    end
    subgraph FALTANTE["No conectado"]
        F1["SET LOCAL app.current_organization_id<br/>❌ no se ejecuta en ningún lado"]
        F2["RlsInterceptor<br/>❌ no registrado en ningún módulo"]
    end

    A1 --> A2 --> A3
    R2 -.espera.-> F1
    F2 -.debería.-> F1

    style FALTANTE fill:#fee2e2,stroke:#b91c1c
    style APLICADO fill:#dcfce7,stroke:#166534
    style DECLARADO fill:#fef3c7,stroke:#a16207
```

**Estado real:** el aislamiento funciona, pero por disciplina en la capa de servicio, no por
la base. Dos consecuencias:

1. Un `WHERE organization_id` omitido en un service nuevo no lo atrapa nada.
2. Las políticas RLS, tal como están escritas, **bloquearían todas las lecturas** si se
   activara la conexión con un rol no-superusuario sin setear la variable de sesión.

Camino para cerrarlo: registrar `RlsInterceptor` globalmente y hacer que ejecute
`SET LOCAL app.current_organization_id = $orgId` dentro de una transacción por request, con
un cliente tomado del pool en vez de `pool.query` directo.

## 7. Superficie de API (contratos lógicos)

### 7.1 Público (sin JWT — decorador `@Public()`)

| Método | Ruta | Devuelve |
|---|---|---|
| `GET` | `/` | metadata de la API (red, versión, docs) |
| `GET` | `/health` | `status`, `uptime` |
| `GET` | `/.well-known/stellar.toml` | SEP-1: `SIGNING_KEY`, `WEB_AUTH_ENDPOINT`, USDC |
| `GET` | `/ngo` | perfil de la ONG + totales + uso de fondos por categoría |
| `GET` | `/projects?q=&category=` | listado público de proyectos |
| `GET` | `/projects/:id` | detalle + pipeline + trazabilidad + impacto |
| `GET` | `/categories` | categorías con proyectos activos |
| `POST` | `/donations` | crea intención → `memoId` + link SEP-7 |
| `GET` | `/donations/:id` | estado de la donación (`verificationCode` = tx hash) |
| `GET` | `/organizations/:id/badges` | SBT en DB + lectura on-chain |
| `GET` | `/auth/challenge?account=G…` | XDR del challenge SEP-10 |
| `POST` | `/auth/token` | verifica XDR firmado → JWT |
| `POST` | `/auth/privy` | verifica token Privy → JWT |
| `GET/POST` | `/webhooks/whatsapp` | verificación Meta + recepción de mensajes |
| `POST` | `/webhooks/telegram` | recepción de updates |
| `GET` | `/my/org/lookups` | catálogos (áreas, poblaciones, ODS) — público pese al prefijo |

### 7.2 Autenticado (JWT `Bearer`)

| Método | Ruta | Roles | Nota |
|---|---|---|---|
| `GET/PATCH` | `/auth/me` | cualquiera | perfil + wallet primaria |
| `POST` | `/auth/refresh` | cualquiera | reemite el JWT |
| `GET` | `/my/projects` · `/my/projects/:id` | cualquiera | scope por `org` del JWT |
| `POST` | `/my/projects` | cualquiera | dispara `allocate()` si `blockchainEnabled` |
| `PATCH` | `/my/projects/:id` | cualquiera | re-ancla si cambia el presupuesto |
| `GET` | `/my/projects/recent-activity` | cualquiera | últimas transacciones de la org |
| `GET` | `/my/projects/:id/on-chain` | cualquiera | lectura directa de `fund-tracker` |
| `GET` | `/my/projects/:id/pipeline-stages` | cualquiera | etapas con estado calculado |
| `GET` | `/my/projects/:id/transactions[/:txId]` | cualquiera | detalle incluye URL firmada de R2 |
| `POST` | `/my/projects/:id/transactions/ocr` | admin·contador·responsable | OCR sin persistir |
| `POST` | `/my/projects/:id/transactions` | admin·contador·responsable | multipart, crea gasto |
| `PATCH` | `/my/projects/:id/transactions/:txId/approve` | admin·auditor | doble control + anclaje |
| `PATCH` | `/my/projects/:id/transactions/:txId/reject` | admin·auditor | marca `failed` |
| `GET/POST` | `/my/reports` | cualquiera | crear ancla el reporte |
| `GET` | `/my/reports/:id/on-chain` | cualquiera | lectura de `expense-anchor` |
| `GET/PATCH` | `/my/org` · `/my/org/profile` | PATCH: admin | perfil básico y extendido |
| `GET` | `/my/org/users` · `/my/org/settings/integrations` | cualquiera | — |
| `PATCH` | `/my/org/users/:id` | admin | cambia rol / estado de un usuario |
| `GET/POST/DELETE` | `/my/org/invites[/:id]` | admin | invitaciones **por email** — ver ⚠️ abajo |
| `GET/PUT` | `/my/org/settings/notifications` | PUT: admin | preferencias de notificación |
| `POST` | `/my/org/logo` · `/my/org/avatar` | logo: admin | multipart → R2 `avatars/{org,user}/<id>` |
| `GET/POST/PATCH/DELETE` | `/my/areas[/:id]` | escritura: admin | áreas presupuestarias |
| `GET/POST/PATCH/DELETE` | `/my/pipeline-templates[/:id]` | escritura: admin | plantillas de pipeline |
| `POST` | `/my/pipeline-templates/:id/duplicate` | admin | clona una plantilla |
| `GET` | `/my/billing` · `/my/billing/plans` | cualquiera | suscripción, uso y catálogo de planes |
| `POST` | `/my/billing/change-plan` · `/my/billing/cancel` | admin | ⚠️ sin cobro real |
| `POST/GET/DELETE` | `/my/bot/invites[/:id]` | admin·admin_regional | invitaciones del bot |
| `POST` | `/admin/badges/mint` · `/admin/badges/:tokenId/revoke` | admin | SBT |

> ⚠️ **Dos superficies incompletas.**
> **Invitaciones de usuario**: se crean, listan y revocan, pero no hay SMTP conectado ni
> endpoint que canjee el token — el estado `accepted` de `user_invites` es inalcanzable.
> **Billing**: `change-plan` y `cancel` sólo escriben en `organization_subscriptions`; no hay
> pasarela de pago ni webhook, y los límites del plan (`max_projects`, `max_users`) **no se
> aplican** en ningún guard.
>
> Ojo con la homonimia: `/my/org/invites` (usuarios, por email) y `/my/bot/invites`
> (voluntarios, por código `ALTA-XXXX`) son mecanismos distintos y sin relación entre sí.

> Observación: los guards son globales (`APP_GUARD`), así que **toda ruta sin `@Public()`
> exige JWT**; y las rutas `/my/*` sin `@Roles` quedan abiertas a cualquier rol autenticado
> de la organización, incluido `voluntario`.

## 8. Trazabilidad: dato → evidencia

La cadena de custodia que hace verificable un gasto:

```mermaid
graph LR
    F["📄 Factura<br/>(imagen/PDF)"] -->|SHA-256| H["🔑 support_file_hash"]
    F -->|PutObject| R2["☁️ R2<br/>invoices/&lt;sha256&gt;"]
    F -->|Gemini| AI["🤖 ai_extracted<br/>ai_amount · ai_confidence"]
    AI -->|comparar ±1%| M["✔️ ai_match"]
    H -->|receipt_hash: Bytes| SC["⛓️ expense-anchor.anchor()"]
    SC --> TX["#️⃣ tx_hash"]
    TX --> DB["🗄️ transactions.tx_hash<br/>tx_status = confirmed"]
    DB --> PUB["🌐 /projects/:id<br/>verificationCode"]
    R2 -.URL firmada 5 min.-> AUD["🔍 Auditor re-calcula SHA-256"]
    AUD -.compara.-> SC

    style SC fill:#fef3c7,stroke:#a16207
    style R2 fill:#ffedd5,stroke:#9a3412
```

La clave en R2 **es** el hash del contenido: si alguien sustituye el archivo, la clave deja
de coincidir y el hash anclado on-chain no valida. Ese es el mecanismo de integridad.

---

## Referencias al código

| Concepto | Archivo |
|---|---|
| Reglas de aprobación y anclaje | `platform/apps/api/src/modules/projects/projects.service.ts` |
| SEP-10 (challenge, verificación, bootstrap) | `platform/apps/api/src/modules/auth/auth.service.ts` |
| Riel Privy | `platform/apps/api/src/modules/auth/privy.service.ts` |
| Clientes de contrato + conversión de tipos | `platform/apps/api/src/modules/soroban/soroban.service.ts` |
| Contratos Rust | `platform/contracts/contracts/{fund-tracker,expense-anchor,sbt-badge}/src/lib.rs` |
| Schema y migraciones | `platform/apps/api/db/*.sql` |
| Roles en UI | `platform/apps/dapp/src/components/settings/roles.ts` |
