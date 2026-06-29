# TrustBid — Diagramas de Secuencia

> Flujos dinámicos del sistema, trazables a los artefactos estáticos.
> **Fuente de verdad:** [casos-de-uso.md](./casos-de-uso.md) (UCxx) ·
> [der.md](./der.md) (tablas) · [diagrama-clases.md](./diagrama-clases.md) (clases).
> Stack: Next.js (web) · NestJS (API) · Workers · PostgreSQL/Neon (RLS) · R2 ·
> AWS KMS · SDP · Privy · Anchor SEP · Stellar/Soroban.

---

## 🤖 Contexto para agentes / IA (leer antes de modificar)

Estos diagramas son **canónicos**. Cualquier código o flujo nuevo debe respetar:

1. **Multi-tenant:** toda operación lleva `organization_id` (= `fundacion_id`). Se
   propaga también a SDP como `tenant_id` 1:1, nunca se resuelve por separado.
2. **Append-only:** un movimiento/desembolso confirmado **no se edita ni borra**.
   Una corrección es un nuevo registro (reversa contable).
3. **Idempotencia:** toda escritura externa (SDP/anchor) usa un `UUID` generado por
   TrustBid *antes* de la llamada + `memo` Stellar. Un reintento nunca duplica.
4. **Async ≠ confirmado:** un `201` de SDP confirma *ingestión*, no *ejecución*. El
   estado `confirmed` solo se marca tras confirmación on-chain (webhook/polling).
5. **Blockchain invisible:** en strings de usuario nunca aparece wallet, hash, USDC,
   Stellar. Se usa "cuenta del área", "código de verificación", "fondos".
6. **Control interno:** desembolsos exigen separación de funciones
   (Initiator ≠ Approver). Ver flujo 2.

---

## Flujo 1 — Registrar gasto con OCR + anclaje on-chain

> UC13, UC15, UC16, UC24, UC33, UC34 · Tablas: `transactions`, `expense_splits`,
> `activity_events`, `indexer_state` · Actor: Responsable de Área (móvil, 3G).

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor R as 👤 Responsable
    participant W as 🖥️ Web (Next.js)
    participant API as ⚙️ API NestJS<br/>(módulo Operaciones)
    participant OCR as 🧠 Worker OCR/IA
    participant R2 as 🗄️ R2 (evidencia)
    participant DB as 🛢️ PostgreSQL (RLS)
    participant KMS as 🔐 AWS KMS
    participant STL as ⛓️ Stellar/Horizon
    participant IDX as 🔎 Worker Indexador

    R->>W: Foto de factura + monto
    W->>API: POST /transactions (JWT + org_id)
    API->>API: Valida tenant + rol (Guard)
    API->>R2: Sube imagen
    API->>DB: INSERT transactions (status=pending, memo_id=UUID)
    API-->>W: 201 (borrador en revisión)
    API->>OCR: Encola job OCR (tx_id)
    OCR->>R2: Lee imagen
    OCR->>DB: Actualiza campos extraídos (pendiente de validar)
    Note over OCR,DB: UC24 — la IA NO autoconfirma:<br/>requiere validación humana
    API->>KMS: Firma tx Stellar (server-side, memo=memo_id)
    KMS-->>API: Tx firmada
    API->>STL: Envía tx con MEMO de anclaje
    STL-->>API: Hash (no confirmado aún)
    IDX->>STL: Polling del ledger (last_ledger)
    STL-->>IDX: Tx confirmada en ledger N
    IDX->>DB: UPDATE status=confirmed, ledger=N
    IDX->>DB: INSERT activity_events (type=expense, tx_hash)
    Note over DB: Append-only — el confirmado<br/>ya no se modifica
```

**Control interno:** el registro nace `pending`; solo el indexador (proceso de
sistema, no un humano) lo pasa a `confirmed` al verlo en el ledger. La evidencia
queda en R2 y el anclaje on-chain es verificable por terceros.

---

## Flujo 2 — Desembolso donante → área → off-ramp (SDP + reconciliación dual)

> UC11, UC30 + desembolso · Tablas: `funding_sources`, `transactions`,
> `expense_splits`, `custodian_keys` · **Separación de funciones obligatoria.**

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#d1fae5','primaryTextColor':'#064e3b','primaryBorderColor':'#10b981','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor I as 👤 Admin (Initiator)
    actor AP as 👤 Admin Regional (Approver)
    participant API as ⚙️ API NestJS<br/>(módulo Disbursements)
    participant DB as 🛢️ PostgreSQL
    participant SDP as 💸 Stellar Disbursement<br/>Platform
    participant ANC as 🏦 Anchor SEP (off-ramp)
    participant REC as ♻️ Worker Reconciliación

    I->>API: Crea solicitud desembolso (UUID idempotencia)
    API->>DB: INSERT desembolso (status=draft, requested_by=I)
    Note over API,DB: Control interno: quien crea ≠ quien aprueba
    AP->>API: Aprueba desembolso
    API->>API: Verifica Approver ≠ Initiator
    API->>DB: status=ready, approved_by=AP
    API->>SDP: POST /disbursements (tenant_id, UUID)
    SDP-->>API: 201 (ingestado, NO ejecutado)
    API->>DB: status=submitted
    Note over SDP: Jobs internos SDP envían<br/>al TSS y luego on-chain
    loop Polling / webhook
        REC->>SDP: GET /statistics
        SDP-->>REC: on-chain SUCCESS
        REC->>ANC: GET /transaction (estado fiat)
        ANC-->>REC: success | failed
    end
    alt on-chain SUCCESS + anchor success
        REC->>DB: status=confirmed (cierre idempotente)
    else on-chain SUCCESS + anchor failed
        REC->>DB: INSERT reversa (append-only)
        Note over REC,DB: Saga: compensación,<br/>nunca editar el original
    end
```

**Control interno (clave para auditoría/jurado):**
- **Initiator ≠ Approver** — la API rechaza que el mismo usuario cree y apruebe.
- **Idempotencia** — el `UUID` evita doble desembolso ante reintentos.
- **Reconciliación dual (Saga)** — si el riel fiat falla tras el éxito on-chain, se
  registra una **reversa append-only**, no se borra el movimiento.

---

## Flujo 3 — Donante verifica trazabilidad (sin cuenta)

> UC29, UC31, UC32 · Tablas: `projects`, `transactions`, `pipeline_stages`,
> `activity_events` · Actor: Donante (lectura pública).

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor D as 👤 Donante
    participant W as 🌐 Portal público (Next.js)
    participant API as ⚙️ API NestJS<br/>(módulo Transparencia)
    participant DB as 🛢️ PostgreSQL
    participant SE as 🔗 Stellar Expert

    D->>W: Abre proyecto financiado
    W->>API: GET /public/projects/:id/trazabilidad
    API->>DB: SELECT movimientos + pipeline + impacto
    DB-->>API: Datos del proyecto
    API-->>W: Trazabilidad + "código de verificación"
    Note over W: UI sin terminología cripto:<br/>"código de verificación", no "hash"
    D->>SE: Clic en código → verificación independiente
    SE-->>D: Tx confirmada on-chain (3ros, sin cuenta)
```

**Credibilidad:** el donante verifica en una fuente externa (Stellar Expert), no
solo en la palabra de TrustBid. Cumple "todo verificable" del proyecto.

---

## Flujo 4 — Generar y exportar reporte por template de donante

> UC21, UC27, UC28 · Tablas: `report_templates`, `reports`, `report_attachments`.

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#fef3c7','primaryTextColor':'#451a03','primaryBorderColor':'#f59e0b','lineColor':'#64748b','fontSize':'13px'}}}%%
sequenceDiagram
    autonumber
    actor C as 👤 Contador
    participant W as 🖥️ Web
    participant API as ⚙️ API NestJS<br/>(módulo Reportes)
    participant DB as 🛢️ PostgreSQL
    participant PDF as 📄 Worker Reportes
    participant R2 as 🗄️ R2

    C->>W: Generar reporte (proyecto, período, donante)
    W->>API: POST /reports (template_id=eu|usaid|idb|custom)
    API->>DB: SELECT template + movimientos del período
    API->>PDF: Encola render con schema del template
    PDF->>DB: Lee datos
    PDF->>R2: Guarda PDF/Excel generado
    PDF->>DB: INSERT report_attachments
    API-->>W: Reporte listo (formato del donante)
    Note over API,PDF: Motor de plantillas: mismos datos,<br/>formato según destinatario (UC28)
```

---

## Trazabilidad de los flujos

| Flujo | UCs | Tablas principales | Patrón de diseño |
|---|---|---|---|
| 1 · Gasto + OCR + anclaje | UC13,15,16,24,33,34 | transactions, activity_events, indexer_state | Pipeline · Append-only |
| 2 · Desembolso + off-ramp | UC11,30 | funding_sources, transactions, custodian_keys | **Saga** · Idempotencia · Segregación de funciones |
| 3 · Verificación donante | UC29,31,32 | projects, transactions, pipeline_stages | CQRS lectura · verificación externa |
| 4 · Reporte por template | UC21,27,28 | report_templates, reports, report_attachments | Strategy (template) · Worker async |
