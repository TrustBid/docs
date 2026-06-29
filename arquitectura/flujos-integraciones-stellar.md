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
        Note over M6: SEP-12 transporta el dato;<br/>el SBT es la prueba on-chain del resultado
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

## Trazabilidad

| Flujo | Piezas Stellar | UCs | Documento relacionado |
|---|---|---|---|
| A · Identidad/login | SEP-1, SEP-10/45 | UC01, UC03 | — |
| B · Donación | SAK, SEP-7, USDC | UC29, UC30 | diagrama-secuencia.md Flujo 3 |
| C · KYC/KYB | SEP-9, SEP-12 | UC01 | — |
| D · Desembolso SDP | SDP | UC11 | diagrama-secuencia.md **Flujo 2** |
| E · Auditoría Horizon | Horizon API | UC27, UC34 | diagrama-secuencia.md Flujo 4 |

> Los flujos de **SBT/reputación** y **ZK** (Soroban) no tienen diagrama aún porque
> su modelo de datos no está en `der.md` / `diagrama-clases.md` — agregar ahí primero
> (tabla de insignias/credenciales) antes de dibujar el flujo, para no repetir el
> error corregido en el flujo de desembolso.
