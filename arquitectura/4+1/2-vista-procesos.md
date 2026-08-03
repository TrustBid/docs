# Vista de Procesos — TrustBid

> **4+1 · Vista 2 de 5** · [← Lógica](./1-vista-logica.md) · [Índice](./README.md) · [Siguiente: Desarrollo →](./3-vista-desarrollo.md)
> Responde: *¿cómo se comporta el sistema en ejecución? ¿qué corre en paralelo?*
> Interesado: integrador, responsable de performance y disponibilidad.

---

## 1. Unidades de ejecución

```mermaid
graph TB
    subgraph EDGE["Cloudflare — edge, multi-región"]
        W1["Worker trustbid-dapp<br/>Next.js 16 vía OpenNext<br/>SSR + Route Handlers"]
        W2["Pages trustbid-landing<br/>estático prerenderizado"]
        W3["Pages trustbid-docs-site<br/>estático"]
    end

    subgraph RAILWAY["Railway — proceso único"]
        API["Proceso Node 22<br/>node apps/api/dist/main.js"]
        subgraph API_INT["Dentro del mismo proceso"]
            HTTP["Servidor HTTP Express<br/>(NestJS)"]
            WORKER["Worker BullMQ<br/>horizon-watch"]
            EV["EventEmitter2<br/>bus en memoria"]
            POOL["Pool pg (max 10)"]
        end
    end

    subgraph BROWSER["Navegador del usuario"]
        UI["React 19 · hooks<br/>Stellar Wallets Kit"]
        EXT["Extensión de wallet<br/>Freighter / Albedo"]
    end

    subgraph EXTERNOS["Servicios externos"]
        NEON["Neon PostgreSQL 16"]
        REDIS["Upstash Redis"]
        R2["Cloudflare R2"]
        SOROBAN["Soroban RPC"]
        HORIZON["Horizon API"]
        GEMINI["Google AI (Gemini)"]
        META["WhatsApp Cloud API"]
        TG["Telegram Bot API"]
        PRIVY["Privy"]
    end

    UI -->|fetch + Bearer JWT| HTTP
    UI --> EXT
    W1 -->|SSR fetch| HTTP
    HTTP --> POOL --> NEON
    HTTP --> REDIS
    WORKER --> REDIS
    WORKER --> HORIZON
    HTTP --> R2
    HTTP --> SOROBAN
    HTTP --> GEMINI
    HTTP --> PRIVY
    META -->|webhook| HTTP
    TG -->|webhook| HTTP
    HTTP --> META
    HTTP --> TG
    HTTP -.emit.-> EV -.OnEvent.-> HTTP

    style RAILWAY fill:#dcfce7,stroke:#166534
    style EDGE fill:#ffedd5,stroke:#9a3412
    style BROWSER fill:#dbeafe,stroke:#1e40af
```

**Observación estructural:** el backend es un **monolito modular de proceso único**. El worker
BullMQ (`HorizonWatcherProcessor`) corre *dentro* del mismo proceso que el servidor HTTP, no
en un servicio aparte. El bus de eventos (`EventEmitter2`) también es en memoria — no
sobrevive a un reinicio ni cruza instancias.

> Consecuencia operativa: escalar la API a más de una réplica duplicaría el consumo de la cola
> (BullMQ lo coordina bien) pero también duplicaría los listeners de `transaction.anchored`
> por instancia. Hoy corre una sola instancia en Railway.

## 2. Concurrencia y estado compartido

| Recurso | Tipo | Dónde vive | Vida útil |
|---|---|---|---|
| Nonce SEP-10 | Redis `auth:nonce:{G…}` | Upstash | 600 s, un solo uso (`DEL` al verificar) |
| Estado de conversación del bot | Redis `bot:conv:{canal}:{userId}` | Upstash | 1800 s |
| Cola `horizon-watch` | BullMQ sobre Redis | Upstash | hasta 60 intentos × 30 s |
| Sesión de usuario | JWT firmado (sin estado en servidor) | localStorage + cookie `tb_jwt` | `JWT_EXPIRES_IN` (8 h por defecto) |
| Pool de conexiones | `pg.Pool` | proceso API | `max: 10`, idle 30 s, connect timeout 5 s |
| Eventos de dominio | `EventEmitter2` | memoria del proceso | instantáneo, no persistido |

El **único estado en memoria del proceso** son los listeners de eventos y el pool. Todo lo
demás es externo, lo que hace al proceso API razonablemente *stateless*.

## 3. Escenario base — rendición y aprobación de un gasto (dashboard)

Es el flujo central del producto: muestra IA, storage, doble control y anclaje.

```mermaid
sequenceDiagram
    autonumber
    actor C as Contador
    participant UI as DApp (Worker CF)
    participant API as ProjectsController
    participant SVC as ProjectsService
    participant G as GeminiService
    participant S as StorageService (R2)
    participant DB as Neon PostgreSQL
    actor A as Admin/Auditor
    participant SOR as SorobanService
    participant RPC as Soroban RPC

    C->>UI: adjunta factura + monto declarado
    UI->>API: POST /my/projects/:id/transactions (multipart)
    API->>SVC: createTransaction(orgId, userId, role, dto, file)
    SVC->>DB: SELECT project WHERE id AND organization_id
    SVC->>SVC: sha256(file.buffer) → support_file_hash
    SVC->>S: putInvoice(buffer, sha256, mime)
    S-->>SVC: storage_key = invoices/&lt;sha256&gt;
    SVC->>G: extractInvoice(buffer, mime)
    G-->>SVC: {vendor, amount, invoiceNumber, taxId, confidence}
    SVC->>SVC: ai_match = abs(ai_amount − declarado) ≤ max(0.01, 1%)
    SVC->>DB: SELECT COUNT(*) año → memo_id PAY-YYYY-NNNN
    SVC->>DB: INSERT transactions (tx_status='pending', ai_*)
    Note over SVC: rol 'contador' ∉ APPROVER_ROLES → queda pendiente
    SVC-->>UI: 201 {id, memoId, status:'pending', ai:{…}}

    Note over A,DB: — Segundo actor, momento distinto —

    A->>UI: abre pendientes → ve monto, ai_match, factura
    UI->>API: GET /my/projects/:id/transactions/:txId
    API->>S: getSignedUrl(storage_key, 300 s)
    S-->>UI: invoiceUrl (expira en 5 min)
    A->>UI: Aprobar
    UI->>API: PATCH …/transactions/:txId/approve
    API->>SVC: approveTransaction(orgId, approverId, …)
    SVC->>DB: SELECT tx → valida tx_status='pending'
    alt created_by == approverUserId
        SVC-->>UI: 403 self_approval
    else aprobador distinto
        SVC->>DB: UPDATE tx_status='submitted', approved_by, approved_at
        SVC-->>UI: 200 {status:'submitted'}  ← respuesta inmediata
        par Anclaje asíncrono (fire-and-forget)
            SVC->>DB: SELECT organizations.wallet_address
            SVC->>SOR: anchorExpenseWithRetry({expenseId, projectId, amount, receiptHash})
            SOR->>RPC: expense-anchor.anchor() firmado por el servidor
            RPC-->>SOR: tx hash
            SOR-->>SVC: hash
            SVC->>DB: UPDATE tx_hash, tx_status='confirmed', confirmed_at
            SVC->>SVC: emit('transaction.anchored') si hay submitter_phone
        end
    end
```

**Puntos de diseño visibles en el diagrama:**

- El HTTP responde en el paso 24; el anclaje sigue después. La UI muestra `submitted` y luego
  `confirmed` cuando refresca.
- La URL de la factura es **firmada y efímera** (300 s): el comprobante nunca se expone público.
- El chequeo de auto-aprobación es explícito y devuelve `403 self_approval`.

## 4. Escenario bot — rendición desde el campo

```mermaid
sequenceDiagram
    autonumber
    actor V as Voluntario
    participant CH as WhatsApp / Telegram
    participant WH as WhatsappController / TelegramController
    participant BF as BotFlowService
    participant EN as EnrollmentService
    participant CV as ConversationService (Redis)
    participant G as GeminiService
    participant PS as ProjectsService
    participant EV as EventEmitter2
    participant BN as BotNotificationService

    Note over V,EN: Fase 1 — enrolamiento
    V->>CH: abre wa.me/…?text=ALTA-XY7K (o t.me/…?start=)
    CH->>WH: webhook (firma X-Hub-Signature-256 en WhatsApp)
    WH->>BF: handleMessage(channel, IncomingMessage)
    BF->>BF: regex /\bALTA-[A-Z0-9]{4,}\b/i
    BF->>EN: tryEnrollByCode(canal, userId, code, name)
    EN->>EN: valida vigencia y cupo (expires_at, uses < max_uses)
    EN->>EN: INSERT users(role='voluntario') + bot_enrollments + uses+1
    EN-->>BF: {enrolled, orgName, projectName}
    BF-->>V: "✅ Quedaste habilitado para el proyecto X"

    Note over V,PS: Fase 2 — rendición
    V->>CH: 📷 foto de la factura
    CH->>WH: webhook con media_id / file_id
    WH->>BF: handleMessage()
    BF-->>V: "📸 Recibí la factura, la estoy leyendo…"
    BF->>CH: downloadMedia(mediaId) → buffer + mime
    BF->>G: extractInvoice(buffer, mime)
    G-->>BF: {vendor, amount, invoiceDate, …}
    BF->>CV: set(state='awaiting_code', extraction, amount, imageBase64) TTL 30 min

    alt Invitación por-proyecto y monto detectado
        BF->>PS: createTransaction(orgId, userId, 'voluntario', defaultProjectId, …)
        Note over PS: 'voluntario' ∉ APPROVER_ROLES → tx_status='pending'
        PS-->>BF: {memoId}
        BF->>CV: clear()
        BF-->>V: "✅ Registrado (PAY-2026-0042). Pendiente de aprobación."
    else Falta monto o falta proyecto
        BF-->>V: "Escribí: monto 250" / "Enviá el CÓDIGO del proyecto"
        V->>CH: "monto 250" o "ESC01"
        BF->>CV: get() → recupera imagen desde base64
        BF->>PS: createTransaction(...)
    end

    Note over PS,BN: Fase 3 — el admin aprueba (flujo §3) y se ancla
    PS->>EV: emit('transaction.anchored', {txHash, submitterUserId, submitterChannel, memoId})
    EV->>BN: @OnEvent('transaction.anchored')
    BN->>CH: sendText("✅ Anclado on-chain. Hash: … stellar.expert/…")
    CH-->>V: notificación con el hash verificable
```

**Detalles de implementación relevantes:**

- La imagen se guarda **en base64 dentro de Redis** mientras dura la conversación
  (`ConversationService`). Una factura de ~2 MB ocupa ~2.7 MB en Redis por 30 minutos.
- WhatsApp valida `X-Hub-Signature-256` con HMAC-SHA256 y `timingSafeEqual`; **si
  `WHATSAPP_APP_SECRET` no está configurado, la verificación devuelve `true`** (decisión de MVP
  documentada en el código, `whatsapp.service.ts:52`).
- Telegram no tiene validación de firma equivalente en el código actual.
- El voluntario nunca elige el estado: su gasto siempre nace `pending`.

## 5. Anclaje on-chain (fire-and-forget)

```mermaid
flowchart TD
    START([approveTransaction / createTransaction auto-autorizada]) --> RESP[Responder HTTP 200 inmediatamente]
    START -.rama paralela.-> Q1[SELECT organizations.wallet_address]
    Q1 --> HASH{¿hay support_file_hash?}
    HASH -->|sí| USE[receiptHash = support_file_hash]
    HASH -->|no| GEN["receiptHash = sha256(txId:concept:amount)"]
    USE --> A1
    GEN --> A1
    A1[intento 1: expense-anchor.anchor] --> OK1{¿hash?}
    OK1 -->|sí| UPD[UPDATE tx_hash, tx_status='confirmed', confirmed_at]
    OK1 -->|no| W[esperar 1500 ms]
    W --> A2[intento 2] --> OK2{¿hash?}
    OK2 -->|sí| UPD
    OK2 -->|no| FAIL["UPDATE tx_status='failed'<br/>(log de error)"]
    UPD --> NOTIF{¿submitter_phone?}
    NOTIF -->|sí| EMIT["emit('transaction.anchored')"]
    NOTIF -->|no| END([fin])
    EMIT --> END
    FAIL --> END

    style FAIL fill:#fee2e2,stroke:#b91c1c
    style UPD fill:#dcfce7,stroke:#166534
```

### Firmante y atribución

Todas las invocaciones Soroban las firma **la keypair del servidor**
(`STELLAR_SERVER_SECRET`). El parámetro `callerPublicKey` que reciben los métodos de
`SorobanService` se lee de la organización pero **se descarta** (`void callerPublicKey`,
`soroban.service.ts:112` y `:146`), porque el contrato exige `caller.require_auth()` y sólo
quien firma puede autorizarse.

```mermaid
graph LR
    ORG["Wallet de la ONG<br/>(no custodial, SEP-10)"] -.no puede firmar server-side.-x SC
    SRV["Keypair del servidor<br/>STELLAR_SERVER_SECRET"] -->|caller + firma| SC["expense-anchor / fund-tracker"]
    SC --> LEDGER["Ledger Stellar"]
    ORG -.atribución fuera de cadena.-> DB["projects.organization_id"]
    DB -.join.-> LEDGER

    style SRV fill:#fef3c7,stroke:#a16207
```

> 🚩 **El log contradice al ledger.** `projects.service.ts:352` escribe
> `Allocation anchored project=… tx=… caller=${callerPublicKey}` con la **wallet de la ONG**,
> pero el `organization` que quedó grabado en `fund-tracker` es la dirección del servidor.
> Un auditor que confíe en el log llegará a una conclusión falsa. Esto además significa que el
> ítem **I-15 de `ISSUES.md`**, marcado como cerrado, está **neutralizado en la capa Soroban**.

> **Implicancia de confianza:** on-chain, el emisor de todos los anclajes es TrustBid. La
> vinculación gasto→ONG es verificable a través de la API pública, no del ledger solo. Cerrar
> esto requeriría autorización delegada de Soroban (firma de la ONG sobre la entrada de auth)
> o wallets custodiales con KMS — ambos previstos en el diseño, ninguno implementado.

## 6. Escenario donación pública — SEP-7 + vigilancia Horizon

```mermaid
sequenceDiagram
    autonumber
    actor D as Donante (sin cuenta)
    participant UI as Portal público (dapp)
    participant API as PublicController
    participant PS as PublicService
    participant DB as Neon
    participant HW as HorizonWatcherService
    participant Q as BullMQ (Upstash)
    participant WK as HorizonWatcherProcessor
    participant H as Horizon API
    participant WAL as Wallet del donante

    D->>UI: elige proyecto y monto
    UI->>API: POST /donations {projectId, amountUsd, walletAddress?, txHash?}
    API->>PS: createDonation(dto)
    PS->>DB: SELECT project + organizations.wallet_address
    PS->>PS: valida estado ≠ archived/completed
    PS->>DB: COUNT(*) del año → memo PAY-YYYY-NNNN
    PS->>DB: INSERT transactions (category='donation', tx_status)
    PS->>PS: arma link SEP-7 web+stellar:pay?destination…&memo=PAY-…

    alt Donación ya firmada en el cliente (txHash presente)
        PS-->>UI: {status:'submitted', sep7Link}
    else Flujo SEP-7 (sin txHash)
        PS->>HW: watchDonation({donationId, memoId, orgWallet, deadline: +30 min})
        HW->>Q: add('watch', job, delay 20 s, attempts 60, backoff fijo 30 s)
        PS-->>UI: {status:'pending', memoId, sep7Link}
        UI-->>D: QR / deep link
        D->>WAL: paga con memo = PAY-YYYY-NNNN
        WAL->>H: submit transaction
        loop cada 30 s, hasta 30 min
            Q->>WK: process(job)
            WK->>H: GET /accounts/{orgWallet}/transactions?order=desc&limit=50
            alt memo_type='text' y memo == memoId
                WK->>DB: UPDATE tx_status='confirmed', tx_hash, confirmed_at
            else no aparece aún
                WK->>WK: throw → BullMQ reintenta
            end
        end
        alt vencido el deadline
            WK->>DB: UPDATE tx_status='expired'
        end
    end
```

**Parámetros de la cola** (`horizon-watcher.service.ts:31-38`):

| Parámetro | Valor | Efecto |
|---|---|---|
| `delay` | 20 000 ms | primera comprobación 20 s después de crear la intención |
| `attempts` | 60 | tope de reintentos |
| `backoff` | `fixed` 30 000 ms | una consulta a Horizon cada 30 s |
| ventana total | 30 min | `DONATION_WATCH_WINDOW_MS` en `public.service.ts:14` |
| `removeOnComplete` | `true` | no acumula jobs exitosos |
| `removeOnFail` | 200 | conserva los últimos 200 fallidos para diagnóstico |

> Costo: hasta 60 llamadas a Horizon por donación no confirmada, cada una trayendo 50
> transacciones de la cuenta. Con muchas donaciones simultáneas conviene mover a *streaming*
> de Horizon o a un indexador.

## 7. Autenticación — dos rieles, un punto de convergencia

```mermaid
sequenceDiagram
    autonumber
    participant UI as DApp
    participant KIT as Stellar Wallets Kit
    participant EXT as Freighter/Albedo
    participant API as AuthController
    participant AS as AuthService
    participant R as Redis
    participant PV as PrivyService
    participant DB as Neon

    rect rgb(219, 234, 254)
    Note over UI,DB: Riel A — wallet nativa (SEP-10)
    UI->>KIT: authModal() → address G…
    UI->>API: GET /auth/challenge?account=G…
    API->>AS: generateChallenge(account)
    AS->>AS: valida StrKey ed25519
    AS->>R: SET auth:nonce:{G…} = randomBytes(48) EX 600
    AS->>AS: TransactionBuilder seq='-1', manageData(source=cliente,<br/>name='{HOME_DOMAIN} auth', value=nonce), timebounds ±300 s
    AS->>AS: firma con la keypair del servidor
    AS-->>UI: {transaction: XDR, network_passphrase}
    UI->>KIT: signTransaction(XDR)
    KIT->>EXT: solicita firma
    EXT-->>UI: signedTxXdr
    UI->>API: POST /auth/token {transaction, registration?}
    AS->>AS: verifica timebounds · formato manageData · firma servidor · firma cliente
    AS->>R: GET + DEL nonce (un solo uso)
    end

    rect rgb(250, 232, 255)
    Note over UI,DB: Riel B — Privy (usuario no cripto)
    UI->>UI: login email/OTP con @privy-io/react-auth
    UI->>API: POST /auth/privy {token, registration?}
    API->>PV: verifyAndEnsureStellarWallet(accessToken)
    PV->>PV: verifyAccessToken contra JWKS de Privy
    PV->>PV: busca linked_account chain_type='stellar'
    alt no existe
        PV->>PV: pregenerateWallets({chain_type:'stellar'}) server-side (Tier 2)
    end
    PV-->>API: stellarPublicKey
    end

    Note over AS,DB: Convergencia — bootstrapAndIssueToken()
    AS->>DB: SELECT user JOIN user_wallets WHERE public_key = G…
    alt primer login
        AS->>DB: BEGIN
        AS->>DB: INSERT organizations (name, slug, wallet_address, country)
        AS->>DB: INSERT users (role, email placeholder @wallet.stellar)
        AS->>DB: INSERT user_wallets (provider, public_key, is_primary=true)
        AS->>DB: COMMIT
    end
    AS-->>UI: {token} — JWT {sub, org, role}
    UI->>UI: localStorage['tb_jwt'] + cookie tb_jwt (30 días, SameSite=Lax)
```

**Detalles verificables:**

- El *bootstrap* de la organización es **transaccional** (`BEGIN`/`COMMIT`/`ROLLBACK`,
  `auth.service.ts:328-403`): o se crean org + usuario + wallet, o no se crea nada.
- El primer usuario de cada organización queda `admin` por defecto.
- El `email` de un usuario de wallet es un placeholder `{G…}@wallet.stellar`; el de un
  voluntario de bot, `{canal}-{userId}@bot.local`.
- La cookie `tb_jwt` existe **sólo** para que el middleware edge de Next pueda gatear
  `/dashboard`; la validación real la hace la API (`middleware.ts:8-10`).

### ⚠️ Asimetría entre los dos rieles: identidad sí, firma no

Los rieles convergen para **autenticar**, pero **no** para firmar transacciones Stellar:

| | Riel A — Wallet nativa | Riel B — Privy |
|---|---|---|
| Prueba de identidad | firma SEP-10 en la extensión | access token verificado contra JWKS |
| Origen de la wallet | del usuario, no custodial | embebida, pregenerada **server-side** |
| Firma de transacciones | `WalletKitSigner` (browser) — **en uso** | `PrivyStellarSigner` (`rawSign`) — **escrito pero nunca invocado** |
| Soporte del proveedor | nativo | **Stellar es Tier 2 en Privy**: sin SDK de alto nivel, hash firmado manualmente |

`PrivyStellarSigner` serializa el hash de la tx a hex con prefijo `0x`, llama
`privy.wallets().rawSign(walletId, {params:{hash}})` y adjunta la firma con `addSignature`.
Ningún flujo del código lo instancia hoy — el riel Privy termina en
`bootstrapAndIssueToken()` y las transacciones on-chain las sigue firmando la keypair del
servidor (§5).

> 🚩 **Bloqueante declarado en el propio código.** Tanto `privy.service.ts` como
> `privy-stellar-signer.ts` llevan el comentario `TODO(privy-stellar)`: *"Tier 2 — rawSign
> manual. VALIDAR EN SANDBOX/TESTNET que la firma producida es aceptada por Horizon antes de
> mover fondos reales… esta cuenta maneja fondos de ONGs."* Es el único punto donde el código
> se marca a sí mismo como no apto para producción. Mientras no se ejercite y valide contra
> Horizon, **las organizaciones que entren por Privy no pueden firmar nada por sí mismas**.

> ⚠️ Desalineación a tener presente: la cookie dura 30 días y el JWT 8 h. Pasadas las 8 h el
> middleware deja pasar y la API responde 401; la UI debe manejar ese 401 (los hooks lo hacen:
> `if (res.status === 401) setError('unauthenticated')`).

## 8. Perfil de latencia y modos de falla

| Operación | Camino crítico | Latencia dominante | Si el dependiente cae |
|---|---|---|---|
| `GET /projects` | API → Neon | query SQL | 500 (la dapp cae al *seed* local en SSR) |
| `POST /auth/token` | API → Redis + crypto | verificación de firmas | 500 si Redis no responde |
| `POST …/transactions` | API → R2 + Gemini + Neon | **Gemini (segundos)** | R2/Gemini nulos → sigue sin comprobante/IA |
| `PATCH …/approve` | API → Neon | ~1 query | responde igual; el anclaje falla aparte |
| Anclaje Soroban | fuera del request | 5–10 s + reintento | `tx_status='failed'`, requiere reproceso manual |
| Webhook del bot | canal → API → Gemini → Neon | Gemini | el usuario recibe mensaje de error amable |
| Donación SEP-7 | UI → API → Redis | encolado | sin Redis, `watchDonation` falla → 500 |

**El costo latente**: `extractInvoice` corre **en línea** dentro del `POST` de transacción
(`projects.service.ts:496`), así que la latencia percibida al cargar un gasto con factura es
la de Gemini. Moverlo a la cola BullMQ existente sería la optimización de mayor impacto.

### Degradación por servicio

```mermaid
graph TD
    subgraph OBLIG["Obligatorios — sin esto la API no arranca o falla"]
        D1["DATABASE_URL (Neon)"]
        D2["REDIS_URL (AuthModule y HorizonWatcher lo exigen)"]
        D3["JWT_SECRET"]
        D4["STELLAR_SERVER_SECRET"]
        D5["*_CONTRACT_ID o caatinga.artifacts.json"]
    end
    subgraph OPT["Opcionales — enabled=false y siguen"]
        O1["R2_* → sin comprobantes persistidos"]
        O2["GOOGLE_API_KEY → sin OCR ni ai_match"]
        O3["WHATSAPP_* → bot WA inerte"]
        O4["TELEGRAM_BOT_TOKEN → bot TG inerte"]
        O5["PRIVY_* → sólo login SEP-10"]
    end
    style OBLIG fill:#fee2e2,stroke:#b91c1c
    style OPT fill:#dcfce7,stroke:#166534
```

`SorobanService` usa `getOrThrow` sobre `STELLAR_SERVER_SECRET` y resuelve los contract IDs
desde env o, en su defecto, desde `caatinga.artifacts.json`; si ninguno existe, **el módulo
falla al construirse y la API no levanta**.

## 9. Idempotencia y consistencia

| Punto | Garantía actual | Riesgo |
|---|---|---|
| `memo_id` `PAY-YYYY-NNNN` | `COUNT(*)` del año + 1, sin lock | **carrera**: dos inserts concurrentes pueden calcular el mismo `n`; la unicidad la salva el índice único, pero el segundo insert falla |
| Nonce SEP-10 | `SET` + `DEL` explícito | correcto (un solo uso) |
| Anclaje de gasto | ninguna llave de idempotencia | un reintento manual re-ancla y sobrescribe en el contrato |
| `tryEnrollByCode` | `SELECT` + `INSERT` sin transacción | dos mensajes simultáneos del mismo usuario podrían duplicar el enrolamiento |
| `uses` de invitación | `UPDATE … uses = uses + 1` | atómico a nivel fila ✅ |
| Bootstrap de organización | transacción explícita | correcto ✅ |
| `approveTransaction` | `SELECT` y luego `UPDATE`, sin `FOR UPDATE` | doble clic podría disparar dos anclajes |

## 10. Observabilidad

- **Logs**: `Logger` de NestJS por servicio; los mensajes clave incluyen `tx=…`,
  `project=…`, `expense=…`, lo que permite correlacionar un anclaje de punta a punta.
- **Health**: `GET /health` (uptime) y `GET /` (red, versión) — ambos públicos.
- **Worker CF**: `[observability] enabled = true` en `apps/dapp/wrangler.toml`.
- **CI**: `soroban-integration.yml` sube los logs de la corrida de integración como artefacto.
- **No hay**: tracing distribuido, métricas agregadas, ni alertas sobre `tx_status='failed'`.
  Una transacción que agota reintentos queda en `failed` sin que nadie se entere
  automáticamente — es la brecha operativa más relevante de esta vista.

---

## Referencias al código

| Proceso | Archivo |
|---|---|
| Anclaje asíncrono + reintentos | `apps/api/src/modules/projects/projects.service.ts:601-661` · `modules/soroban/soroban.service.ts:169-193` |
| Cola y worker Horizon | `apps/api/src/modules/horizon/horizon-watcher.{service,processor}.ts` |
| Bus de eventos y notificación | `apps/api/src/modules/whatsapp/bot-notification.service.ts` |
| Flujo del bot | `apps/api/src/modules/whatsapp/bot-flow.service.ts` |
| SEP-10 | `apps/api/src/modules/auth/auth.service.ts:70-210` |
| Estado efímero en Redis | `apps/api/src/modules/whatsapp/conversation.service.ts` |
