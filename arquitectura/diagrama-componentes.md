# TrustBid — Diagrama de Componentes

> Vista estructural del sistema en dos niveles del modelo **C4**: Contenedores
> (C4-L2) y Componentes de la API (C4-L3).
> **Fuente de verdad:** [casos-de-uso.md](./casos-de-uso.md) (módulos M1–M7) ·
> [diagrama-clases.md](./diagrama-clases.md) · [diagrama-secuencia.md](./diagrama-secuencia.md).

---

## 🤖 Contexto para agentes / IA

- El sistema es un **monolito modular** (NestJS) + **workers** desacrochados por
  cola, no microservicios. No separar en servicios físicos sin una razón de
  escalabilidad real (ver regla 10 de la skill de arquitectura).
- Cada **módulo de la API mapea 1:1 a un módulo de casos de uso** (M1–M7).
- Las piezas externas (SDP, Privy, Anchor, Stellar) **se integran, no se
  reimplementan**. TrustBid las orquesta.
- Toda dependencia va **hacia interfaces** (puertos), no hacia implementaciones
  concretas — habilita reemplazar el anchor (BlindPay↔Abroad) sin tocar el dominio.

---

## Nivel 2 (C4) — Contenedores

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'13px'}}}%%
graph TB
    classDef person fill:#dbeafe,stroke:#3b82f6,stroke-width:2px,color:#1e3a5f,font-weight:bold
    classDef container fill:#f8fafc,stroke:#475569,stroke-width:2px,color:#0f172a
    classDef data fill:#ecfeff,stroke:#0891b2,stroke-width:2px,color:#083344
    classDef ext fill:#f3e8ff,stroke:#9333ea,stroke-width:2px,color:#581c87

    U1["👤 Usuarios ONG<br/>(admin, responsable,<br/>contador, regional)"]:::person
    U2["👤 Donante /<br/>Público"]:::person
    U3["👨‍💻 Desarrollador /<br/>Partner"]:::person

    subgraph TB_SYS ["Sistema TrustBid"]
        WEB["🖥️ Web App<br/>Next.js (App Router)<br/>portal interno + público"]:::container
        DOCS["📚 Docs Site<br/>Next.js (Static)<br/>Cloudflare Pages"]:::container
        API["⚙️ API<br/>NestJS REST<br/>dominio + orquestación"]:::container
        WRK["🧰 Workers<br/>indexador · OCR/IA ·<br/>PDF · reconciliación"]:::container
        PKG["📦 packages/*<br/>stellar-sdk+KMS · ocr-pipeline ·<br/>reports · tenancy · ui"]:::container
        PG[("🛢️ PostgreSQL / Neon<br/>RLS por tenant")]:::data
        RDS[("⚡ Redis / Upstash<br/>BullMQ (colas) + cache")]:::data
        R2[("🗄️ Cloudflare R2<br/>evidencias / PDFs")]:::data
    end

    subgraph EXT ["Ecosistema externo"]
        SDP["💸 SDP<br/>(servicio aparte)"]:::ext
        PRIVY["👛 Privy<br/>wallets invisibles"]:::ext
        ANCHOR["🏦 Anchor SEP<br/>BlindPay / Abroad"]:::ext
        KMS["🔐 AWS KMS"]:::ext
        STELLAR["⛓️ Stellar + Soroban<br/>+ Horizon / Stellar Expert"]:::ext
        CLERK["🪪 Clerk<br/>auth multi-tenant"]:::ext
        EXTSYS["🔗 ERP/CRM donante<br/>NetSuite / Salesforce"]:::ext
    end

    U1 --> WEB
    U2 --> WEB
    U3 --> DOCS
    WEB --> API
    API --> PKG
    API --> PG
    API --> RDS
    API --> R2
    WRK --> RDS
    WRK --> PG
    WRK --> R2
    API --> CLERK
    API --> PRIVY
    API --> SDP
    API --> EXTSYS
    PKG --> KMS
    PKG --> STELLAR
    SDP --> STELLAR
    WRK --> STELLAR
    WRK --> ANCHOR
    SDP --> ANCHOR
    U2 -.verifica.-> STELLAR
```

---

## Nivel 3 (C4) — Componentes internos de la API NestJS

> Cada módulo corresponde a un módulo de casos de uso (M1–M7).

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'12px'}}}%%
graph TB
    classDef mod fill:#eff6ff,stroke:#3b82f6,stroke-width:2px,color:#1e3a5f,font-weight:bold
    classDef cross fill:#fef9c3,stroke:#ca8a04,stroke-width:2px,color:#422006
    classDef port fill:#f5f3ff,stroke:#7c3aed,stroke-width:1.5px,color:#4c1d95
    classDef data fill:#ecfeff,stroke:#0891b2,stroke-width:2px,color:#083344

    subgraph CROSS ["Capa transversal (cross-cutting)"]
        AUTH["🔐 AuthGuard +<br/>TenantContext (org_id)"]:::cross
        RLS["🧱 Repositorio con RLS<br/>(filtra por org_id)"]:::cross
        AUDIT["📝 ActivityEvents<br/>(auditoría append-only)"]:::cross
        IDEM["♻️ Idempotencia<br/>(UUID + memo)"]:::cross
    end

    subgraph MODS ["Módulos de dominio"]
        M1["M1 · Acceso y<br/>Configuración"]:::mod
        M2["M2 · Proyectos"]:::mod
        M3["M3 · Operaciones<br/>y Gastos"]:::mod
        M4["M4 · Reportes y<br/>Auditoría"]:::mod
        M5["M5 · Transparencia<br/>y Donaciones"]:::mod
        M6["M6 · Blockchain<br/>(orquestación)"]:::mod
        M7["M7 · Integraciones"]:::mod
    end

    subgraph PORTS ["Puertos (interfaces) → adaptadores externos"]
        P1["IDisbursementPort → SDP"]:::port
        P2["IWalletPort → Privy"]:::port
        P3["IOffRampPort → Anchor SEP"]:::port
        P4["ISignerPort → KMS"]:::port
        P5["ILedgerPort → Stellar/Horizon"]:::port
        P6["IExternalSysPort → ERP/CRM"]:::port
    end

    PG[("🛢️ PostgreSQL")]:::data

    AUTH --> RLS --> PG
    M1 --> AUTH
    M2 --> RLS
    M3 --> RLS
    M3 --> IDEM
    M3 --> AUDIT
    M4 --> RLS
    M5 --> RLS
    M6 --> IDEM
    M6 --> AUDIT

    M3 --> P4
    M3 --> P5
    M6 --> P1
    M6 --> P3
    M6 --> P5
    M1 --> P2
    M7 --> P6
    M4 --> P5
```

---

## Responsabilidad de cada componente (Alta Cohesión)

| Componente | Responsabilidad única | UCs |
|---|---|---|
| **M1 Acceso/Config** | Org, usuarios, roles, jerarquía, wallets | UC01–04 |
| **M2 Proyectos** | Plan→Programa→Proyecto, presupuesto, fuentes, pipeline | UC05–11 |
| **M3 Operaciones** | Gastos, OCR, splits, evidencia, impacto, beneficiarios | UC12–18 |
| **M4 Reportes** | Templates, conciliación, import, export por donante | UC19–28 |
| **M5 Transparencia** | Portal público, donaciones, trazabilidad | UC29–32 |
| **M6 Blockchain** | Orquesta firma/anclaje/desembolso, idempotencia | UC33–35 |
| **M7 Integraciones** | API hacia/desde ERP/CRM del donante | UC36–37 |
| **Transversal** | Tenant, RLS, auditoría, idempotencia (todos los módulos) | — |

## Decisiones de diseño (consecuencias)

- **Puertos y adaptadores (Hexagonal):** el dominio no conoce a SDP/Privy/anchor
  concretos → se puede cambiar BlindPay por Abroad sin tocar M2–M6. *(Costo: más
  interfaces que un acoplamiento directo.)*
- **Workers vía cola (BullMQ):** OCR, PDF, indexado y reconciliación no bloquean la
  API → resiliencia y respuesta rápida en 3G. *(Costo: consistencia eventual.)*
- **RLS + `org_id` transversal:** doble capa de aislamiento multi-tenant. *(Costo:
  cada repositorio debe propagar el contexto de tenant sin excepción.)*
