# TrustBid — Flujos de Integraciones Stellar

> Flujos de secuencia para las piezas de [`integraciones-stellar.md`](../integraciones-stellar.md)
> (el mapa técnico de SEPs/herramientas). **Fuente de verdad del mapeo:** ese
> documento — si algo aquí lo contradice, gana `integraciones-stellar.md`.
>
> Coherente con [diagrama-secuencia.md](./diagrama-secuencia.md) (mismas
> convenciones de actores/notación) y [casos-de-uso.md](./casos-de-uso.md) (UCxx).

---

## 🤖 Contexto para agentes / IA

- Si un flujo requiere una tabla/columna que no existe en [der.md](./der.md) o
  [diagrama-clases.md](./diagrama-clases.md), **márcalo como pendiente de schema**,
  no asumas que ya existe (mismo criterio aplicado en `diagrama-secuencia.md` con el
  flujo de desembolso).
- **Testnet primero, siempre.** Nunca mainnet ni claves con fondos reales.
- **Blockchain invisible:** ningún paso de UI debe filtrar terminología cripto al
  usuario final (ver CLAUDE.md sección 1).
- Respeta el **roadmap por fases** de `integraciones-stellar.md` sección 8.

---

## Flujo A — Identidad y login de la organización (SEP-1 + SEP-10/45)

> SEP-1, SEP-10, SEP-45 · UC01, UC03 · Prerrequisito de todo lo demás
> (KYB, SBT, desembolsos).

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor A as 👤 Admin (ONG)
    participant W as 🖥️ Web
    participant API as ⚙️ API NestJS<br/>(módulo Acceso)
    participant TOML as 🌐 stellar.toml<br/>(dominio de la ONG)
    participant STL as ⛓️ Stellar testnet

    A->>W: Registra organización + dominio
    API->>TOML: Verifica stellar.toml publicado en el dominio
    TOML-->>API: Cuentas/activos/KYC_SERVER declarados
    Note over API,TOML: SEP-1: dominio controlado por la ONG =<br/>primera barrera anti-suplantación
    A->>W: Conectar wallet (login)
    W->>API: Solicita challenge SEP-10
    API->>STL: Genera tx de reto (no se envía a la red)
    API-->>W: Tx de reto para firmar
    W->>A: Pide firma en la wallet
    A->>W: Firma con clave privada (nunca sale del cliente)
    W->>API: Envía tx firmada
    API->>API: Verifica firma == cuenta declarada
    API-->>W: JWT de sesión
    Note over API: Prueba control de cuenta,<br/>no solo conocimiento de contraseña
```

---

## Flujo B — Donación con wallet (SAK + SEP-7 + USDC)

> Stellar Wallets Kit, SEP-7, USDC · UC29, UC30.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#d1fae5','primaryTextColor':'#064e3b','primaryBorderColor':'#10b981','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor D as 👤 Donante
    participant W as 🌐 Portal público
    participant SAK as 👛 Stellar Wallets Kit
    participant WAL as 🔑 Wallet del donante<br/>(Freighter/xBull/Albedo/...)
    participant API as ⚙️ API NestJS<br/>(módulo Transparencia)
    participant STL as ⛓️ Stellar testnet

    D->>W: Abre proyecto → "Donar" / escanea QR (SEP-7)
    W->>SAK: Conectar wallet
    SAK->>WAL: Solicita conexión
    WAL-->>SAK: Dirección pública
    alt Donante sin trustline a USDC
        SAK->>WAL: Solicita crear trustline USDC
        WAL-->>STL: Trustline creada
    end
    W->>API: Pide monto + memo (proyecto)
    API-->>W: Tx de pago (destino, monto, memo)
    SAK->>WAL: Solicita firma de la tx
    WAL-->>D: Confirmación en wallet
    D->>WAL: Firma
    WAL->>STL: Envía tx (USDC + memo del proyecto)
    STL-->>API: Confirmación (vía indexador, async)
    API->>API: Asocia donación al proyecto por memo
    Note over W: UI sin jerga cripto:<br/>"fondos disponibles", no "USDC/wallet"
```

---

## Flujo C — KYC/KYB de la organización (SEP-9 + SEP-12)

> SEP-9, SEP-12 · UC01 · Requiere SEP-10 previo (Flujo A). Alimenta el SBT de reputación.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#fef3c7','primaryTextColor':'#451a03','primaryBorderColor':'#f59e0b','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor A as 👤 Admin (ONG)
    participant W as 🖥️ Web
    participant API as ⚙️ API NestJS<br/>(módulo Acceso)
    participant KYB as 🔍 Proveedor KYB<br/>(Sumsub/Persona) o manual
    participant DB as 🛢️ PostgreSQL
    participant M6 as ⛓️ Módulo Blockchain<br/>(SBT)

    A->>W: Completa campos SEP-9 (organización)
    W->>API: PUT /customer (SEP-12, JWT de SEP-10)
    API->>DB: Guarda campos KYB (pendiente)
    API->>KYB: Envía a validación
    KYB-->>API: Resultado (aprobado/rechazado)
    API->>DB: Actualiza estado KYB
    alt KYB aprobado
        API->>M6: Solicita emisión de insignia/ACTA inicial
        M6->>M6: Emite SBT (no transferible) a la cuenta de la ONG
        Note over M6: SEP-12 transporta el dato<br/>El SBT es la prueba on-chain del resultado
    else KYB rechazado
        API-->>W: Notifica motivo, solicita corrección
    end
```

---

## Flujo D — Desembolso masivo vía SDP

> SDP · UC11 · Mismo flujo de fondo que **Flujo 2** de
> [diagrama-secuencia.md](./diagrama-secuencia.md) — éste detalla la integración
> SDP en sí (registro de wallet del receptor vía deeplink/OTP), no la reconciliación
> dual completa. Ver ese documento para idempotencia, Saga y reversa append-only.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#f3e8ff','primaryTextColor':'#581c87','primaryBorderColor':'#9333ea','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor AR as 👤 Admin Regional
    actor B as 👤 Beneficiario/Responsable
    participant API as ⚙️ API NestJS
    participant SDP as 💸 SDP<br/>(servicio aparte)
    participant SMS as 📱 SMS/WhatsApp<br/>(invitación SDP)

    AR->>API: Aprobar distribución (lote de receptores)
    API->>SDP: POST /disbursements (tenant_id, UUID, lote)
    SDP-->>API: 201 (ingestado)
    SDP->>SMS: send_receiver_wallets_invitation_job
    SMS->>B: Deeplink/OTP de registro
    B->>SDP: Confirma datos (teléfono, fecha nac.) directo con SDP
    Note over B,SDP: Datos NO se comparten con la wallet —<br/>los recibe solo la SDP
    SDP->>SDP: payment_to_submitter_job → TSS → on-chain
    Note over API,SDP: Ver Flujo 2 de diagrama-secuencia.md<br/>para reconciliación dual + idempotencia
```

---

## Flujo E — Auditoría desde el ledger (Horizon)

> Horizon API · UC27, UC34 · No requiere SEP — es lectura directa del ledger.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#ecfeff','primaryTextColor':'#083344','primaryBorderColor':'#0891b2','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor C as 👤 Contador
    participant W as 🖥️ Web
    participant API as ⚙️ API NestJS<br/>(módulo Reportes)
    participant IDX as 🔎 Worker Indexador
    participant HZ as 🌅 Horizon API
    participant DB as 🛢️ PostgreSQL

    IDX->>HZ: Polling de cuentas/proyectos (last_ledger)
    HZ-->>IDX: Historial de transacciones
    IDX->>DB: Reconcilia movimientos (activity_events)
    C->>W: Solicita informe financiero del período
    W->>API: GET /reports?project&period
    API->>DB: Lee movimientos indexados
    API-->>W: Informe + links a Stellar Expert por tx
    Note over W: Cualquier tercero puede abrir el link<br/>sin cuenta y verificar de forma independiente
```

---

## Flujo F — Emisión de SBT de reputación (Soroban)

> Soroban SBT · Tablas: `organization_badges`, `activity_events` ·
> Prerrequisito: **Flujo C** (KYB aprobado) para `badge_type = kyb_verified`.
> Los badges de transparencia (bronze/silver/gold) los dispara el módulo Blockchain
> automáticamente al detectar umbrales de meses de cumplimiento en DB.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#f3e8ff','primaryTextColor':'#3b0764','primaryBorderColor':'#9333ea','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    participant API as ⚙️ API NestJS<br/>(módulo Blockchain)
    participant DB as 🛢️ PostgreSQL
    participant KMS as 🔐 AWS KMS
    participant SRN as 🔷 Soroban RPC
    participant IDX as 🔎 Worker Indexador
    participant ACT as 📋 activity_events

    Note over API: Trigger: KYB aprobado (Flujo C)<br/>o umbral de meses cumplido (cron)
    API->>DB: INSERT organization_badges (status=pending, badge_type, contract_id)
    API->>KMS: @caatinga/client → invoke(mint_badge)<br/>CaatingaWalletAdapter backed by AWS KMS
    KMS-->>API: XDR firmado (clave privada nunca sale de KMS)
    API->>SRN: Envía XDR al Soroban RPC (fn: mint_badge)
    SRN-->>API: Hash de tx (no confirmado aún)
    API->>DB: UPDATE organization_badges (status=pending, anchor_tx_hash)
    Note over SRN: Contrato Soroban valida:<br/>1. Llamante autorizado (contrato owner)<br/>2. Badge no transferible (Soulbound)<br/>3. No duplicado para esta org
    IDX->>SRN: Polling del ledger (last_ledger)
    SRN-->>IDX: Invocación confirmada en ledger N
    IDX->>DB: UPDATE organization_badges<br/>(status=issued, token_id, stellar_ledger=N, issued_at)
    IDX->>ACT: INSERT activity_events (type=verification,<br/>reference_table=organization_badges)
    Note over DB: Append-only: revocación posterior<br/>= nuevo registro status=revoked,<br/>nunca UPDATE del badge original
```

**Invariantes del contrato SBT:**
- `mint_badge` solo puede llamarlo el owner del contrato (AWS KMS, server-side).
- La función `transfer` está deshabilitada — el token es no transferible.
- `revoke_badge` emite una nueva entrada en el ledger; el badge original queda inmutable.
- El `token_id` lo asigna el contrato y se recupera del evento de la tx confirmada.

---

## Flujo G — Prueba ZK de compliance presupuestario (Soroban)

> ZK (zk-SNARK offchain + anclaje Soroban) · Tablas: `zk_proofs`, `activity_events` ·
> UC27, UC34 — un auditor externo verifica que el gasto ≤ presupuesto sin ver montos reales.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#ecfeff','primaryTextColor':'#083344','primaryBorderColor':'#0891b2','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor C as 👤 Contador
    actor AUD as 🔍 Auditor externo
    participant W as 🖥️ Web
    participant API as ⚙️ API NestJS<br/>(módulo Blockchain)
    participant DB as 🛢️ PostgreSQL
    participant WZK as ⚡ Worker ZK<br/>(@caatinga/zk)
    participant R2 as 🗄️ R2 (artefactos)
    participant KMS as 🔐 AWS KMS
    participant SRN as 🔷 Soroban RPC

    C->>W: Solicita generar prueba de compliance<br/>(proyecto, período)
    W->>API: POST /zk-proofs (credential_type=budget_compliance)
    API->>DB: SELECT transactions + budget del proyecto (datos privados)
    API->>API: Calcula commitment_hash<br/>= hash(monto_total, presupuesto, salt)
    API->>DB: INSERT zk_proofs (status=computing,<br/>commitment_hash, public_inputs)
    API->>WZK: Encola job ZK (inputs privados + circuito Circom)
    Note over WZK: caatinga zk prove (Groth16):<br/>prueba que gasto ≤ presupuesto<br/>SIN revelar montos reales
    WZK->>R2: Sube proof_artifact.json<br/>(inputs públicos + prueba Groth16 completa)
    WZK->>DB: UPDATE zk_proofs (status=anchoring, proof_artifact_url)
    WZK->>API: Notifica proof lista
    API->>KMS: caatinga zk invoke → CaatingaWalletAdapter/KMS<br/>(fn: anchor_commitment, args: hash, credential_type)
    KMS-->>API: XDR firmado
    API->>SRN: Envía XDR al Soroban RPC
    Note over SRN: ⚠️ Mainnet: requiere --allow-dev-ceremony<br/>en caatinga.config.ts (bloqueado por defecto)
    SRN-->>API: Hash confirmado en ledger N
    API->>DB: UPDATE zk_proofs (status=anchored,<br/>anchor_tx_hash, stellar_ledger=N, generated_at)
    API-->>W: Prueba lista (anchor_tx_hash + proof_artifact_url)
    Note over W: UI muestra "código de verificación"<br/>(anchor_tx_hash), nunca jerga ZK
    AUD->>SRN: Consulta contrato con commitment_hash
    SRN-->>AUD: Confirmado en ledger N (tipo + timestamp)
    AUD->>R2: Descarga proof_artifact.json
    AUD->>AUD: Verifica la prueba localmente<br/>(sin ver los montos reales)
```

**Garantías de privacidad:**
- El `commitment_hash` en Soroban no revela montos ni fechas.
- Los `public_inputs` en DB y en el artefacto solo incluyen el threshold (ej. `compliance: true`) y el período, no los valores exactos.
- El auditor puede verificar matemáticamente la prueba con herramientas estándar (snarkjs) sin necesitar acceso a DB ni a R2 privado si se le comparte el artefacto.

---

## Flujo H — Despliegue inicial de contratos Soroban (one-time, DevOps)

> Caatinga CLI · Prerrequisito de F y G · Se ejecuta una vez por entorno (testnet → mainnet).
> Resuelve la pregunta: *¿de dónde viene el `contract_id` que guardan `organization_badges`
> y `zk_proofs`?* Sin este flujo, F y G tienen una referencia huérfana.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#fef9c3','primaryTextColor':'#713f12','primaryBorderColor':'#ca8a04','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor OPS as 🔧 DevOps / Backend Admin
    participant CLI as 💻 caatinga CLI
    participant RUS as 🦀 Rust Toolchain<br/>(wasm32 target)
    participant SRN as 🔷 Soroban RPC<br/>(testnet / mainnet)
    participant ART as 📄 caatinga.artifacts.json
    participant ENV as ⚙️ sync-env / .env
    participant DB as 🛢️ PostgreSQL<br/>(config / secrets)

    OPS->>CLI: caatinga build --contracts sbt_badge,zk_verifier
    CLI->>RUS: cargo build --target wasm32-unknown-unknown --release
    RUS-->>CLI: sbt_badge.wasm + zk_verifier.wasm + WASM hashes
    CLI-->>OPS: Build OK
    OPS->>CLI: caatinga deploy --network testnet --source trustbid-ops
    CLI->>SRN: Upload WASM (InstallContractCode)
    SRN-->>CLI: sbt_contract_id + zk_contract_id (confirmados en ledger)
    CLI->>ART: Escribe contract_ids, wasm_hash, deploy_timestamp por red
    Note over ART: caatinga.artifacts.json = fuente de verdad<br/>versionada en git del estado on-chain
    CLI->>CLI: caatinga wire (resuelve ${contracts.sbt.contractId}<br/>en postDeploy hooks entre contratos)
    CLI->>ENV: caatinga sync-env → actualiza vars de entorno<br/>(NEXT_PUBLIC_SBT_CONTRACT_ID, ZK_CONTRACT_ID…)
    OPS->>CLI: caatinga doctor
    CLI->>SRN: Verifica contratos accesibles y responden
    SRN-->>CLI: OK
    OPS->>DB: INSERT config (sbt_contract_id, zk_contract_id,<br/>network, wasm_hash, deployed_at)
    Note over DB: A partir de aquí organization_badges.contract_id<br/>y zk_proofs.contract_id tienen fuente de verdad.<br/>Repite el flujo para mainnet cuando sea momento.
```

**Notas operativas:**
- `--source trustbid-ops` es un alias de Stellar CLI que apunta a la cuenta pagadora (no usa clave privada raw — ver restricción Caatinga).
- El `caatinga.artifacts.json` se commitea al repo de `backend/` para que CI/CD pueda reproducir el estado exacto.
- Para mainnet: mismos pasos con `--network mainnet` + añadir `allowDevCeremony: false` en `caatinga.config.ts` para el módulo ZK.

---

## Flujo I — Verificación pública de reputación ONG (SEP-1 + on-chain)

> SEP-1, Soroban RPC, Horizon · Tablas: `organization_badges` (solo lectura) ·
> **Diferenciador clave:** cualquier tercero (donante, auditor, ERP, regulador) puede
> verificar la reputación y los objetivos on-chain de una ONG **sin depender de TrustBid**.
> La fuente de verdad es el dominio propio de la ONG (SEP-1) + el ledger Soroban (inmutable).

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dcfce7','primaryTextColor':'#14532d','primaryBorderColor':'#16a34a','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor V as 🔍 Verificador externo<br/>(donante / auditor / ERP / regulador)
    participant DOM as 🌐 Dominio ONG<br/>(.well-known/stellar.toml)
    participant SRN as 🔷 Soroban RPC<br/>(caatinga read — sin firma)
    participant HZ as 🌅 Horizon API
    participant SE as 🔗 Stellar Expert

    V->>DOM: GET https://ong.org/.well-known/stellar.toml (SEP-1)
    DOM-->>V: ACCOUNTS=[wallet_address]<br/>TRUST_SBT_CONTRACT=[sbt_contract_id]<br/>ORG_NAME, ORG_URL, ORG_DESCRIPTION
    Note over DOM,V: SEP-1: el dominio controlado por la ONG es<br/>la declaración pública sin intermediarios.<br/>TrustBid no es el custodio de esta información.
    V->>SRN: caatinga read → query_contract(sbt_contract_id,<br/>fn: get_badges, args: wallet_address)
    SRN-->>V: [{badge_type, issued_at, token_id, anchor_tx_hash}]
    Note over SRN,V: Lectura sin wallet ni firma —<br/>CAATINGA_MULTI_AUTH_REQUIRED no aplica en read().<br/>El verificador no necesita cuenta Stellar.
    V->>HZ: GET /transactions/:anchor_tx_hash
    HZ-->>V: Tx confirmada — ledger N, timestamp, fuente on-chain
    V->>SE: Abre Stellar Expert con anchor_tx_hash
    SE-->>V: Verificación independiente de terceros<br/>(inmutable, archivada en el ledger para siempre)
    Note over V,SE: El verificador obtiene prueba criptográfica<br/>de reputación sin que TrustBid esté disponible.<br/>Funciona aunque TrustBid tenga downtime.
```

**Por qué esto es el diferenciador:**
- **Sin custodia:** la ONG publica su propio `stellar.toml` en su dominio — TrustBid no puede modificarlo.
- **Inmutabilidad:** los badges en el ledger Soroban no pueden borrarse ni editarse retroactivamente.
- **Integración nativa con ERPs y flujos de caja:** un ERP puede hacer polling automatizado de `stellar.toml` + Soroban RPC para validar automáticamente si una ONG tiene `kyb_verified` antes de procesar una transferencia. No requiere API key ni contrato con TrustBid.
- **Reguladores y auditores:** un ente regulador puede verificar el historial completo de badges a través de Horizon sin solicitar datos a TrustBid.

---

## Trazabilidad

| Flujo | Piezas Stellar / Toolchain | UCs | Documento relacionado |
|---|---|---|---|
| A · Identidad/login | SEP-1, SEP-10/45 | UC01, UC03 | — |
| B · Donación con wallet | SAK, SEP-7, USDC | UC29, UC30 | diagrama-secuencia.md Flujo 3 |
| C · KYC/KYB organización | SEP-9, SEP-12 | UC01 | — |
| D · Desembolso masivo | SDP | UC11 | diagrama-secuencia.md **Flujo 2** |
| E · Auditoría Horizon | Horizon API | UC27, UC34 | diagrama-secuencia.md Flujo 4 |
| F · Emisión SBT | Soroban + @caatinga/client + KMS | — | der.md Vista 4, sprint4-sbt-zk.sql |
| G · Prueba ZK compliance | Soroban + @caatinga/zk (Groth16) | UC27, UC34 | der.md Vista 4, sprint4-sbt-zk.sql |
| H · Despliegue contratos | caatinga CLI (build/deploy/wire) | — | sprint4-sbt-zk.sql, caatinga.artifacts.json |
| I · Verificación pública ONG | SEP-1 + Soroban RPC + Horizon | UC29, UC31, UC32 | der.md Vista 4 |
