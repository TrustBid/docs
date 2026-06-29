# TrustBid — Diagrama de Despliegue

> Vista física: dónde corre cada contenedor, sobre qué nodos y con qué protocolos.
> **Fuente de verdad:** [diagrama-componentes.md](./diagrama-componentes.md) ·
> [diagrama-secuencia.md](./diagrama-secuencia.md) · CLAUDE.md (stack e infra).

---

## 🤖 Contexto para agentes / IA

- Entorno del hackathon = **Stellar testnet**. Nunca apuntar a mainnet ni usar
  claves con fondos reales.
- Las llaves viven **solo** en variables de entorno del nodo; en el repo solo
  `.env.example`. KMS firma server-side: la clave privada nunca llega al cliente.
- **Regla de oro de despliegue:** Privy + off-ramp se despliegan PRIMERO (red de
  seguridad). SDP se monta encima. Si SDP no llega, deben quedar 2 integraciones
  reales desplegadas.
- SDP corre como **servicio aparte** (su propio host/contenedor + su propia BD), no
  embebido en la API.

---

## Diagrama de despliegue

```mermaid
%%{init: {'theme':'base','themeVariables':{'primaryColor':'#dbeafe','primaryTextColor':'#1e3a5f','primaryBorderColor':'#3b82f6','lineColor':'#64748b','fontSize':'12px'}}}%%
graph TB
    classDef device fill:#dbeafe,stroke:#3b82f6,stroke-width:2px,color:#1e3a5f,font-weight:bold
    classDef node fill:#f8fafc,stroke:#475569,stroke-width:2px,color:#0f172a
    classDef data fill:#ecfeff,stroke:#0891b2,stroke-width:2px,color:#083344
    classDef ext fill:#f3e8ff,stroke:#9333ea,stroke-width:2px,color:#581c87
    classDef artifact fill:#ffffff,stroke:#94a3b8,stroke-width:1px,color:#334155

    subgraph CLIENT ["📱 Dispositivos cliente"]
        BR["🖥️ Navegador desktop<br/>(admin, contador, regional)"]:::device
        MOB["📱 Móvil 3G<br/>(responsable en campo)"]:::device
    end

    subgraph VERCEL ["☁️ Vercel (edge/CDN)"]
        WEBN["«artifact» Web App<br/>Next.js SSR/Static"]:::artifact
    end

    subgraph CFPAGES ["☁️ Cloudflare Pages"]
        DOCSN["«artifact» Docs Site<br/>Next.js (Static Export)"]:::artifact
    end

    subgraph RAILWAY ["☁️ Railway"]
        APIN["«artifact» API<br/>NestJS (Docker)"]:::artifact
        WRKN["«artifact» Workers<br/>indexador·OCR·PDF·reconcil."]:::artifact
    end

    subgraph SDPHOST ["☁️ Host SDP (servicio aparte)"]
        SDPN["«artifact» Stellar Disbursement<br/>Platform (Docker)"]:::node
        SDPDB[("🛢️ PostgreSQL SDP<br/>(esquemas por tenant)")]:::data
    end

    subgraph MANAGED ["☁️ Servicios gestionados"]
        NEON[("🛢️ Neon<br/>PostgreSQL + RLS")]:::data
        UPS[("⚡ Upstash<br/>Redis / BullMQ")]:::data
        R2N[("🗄️ Cloudflare R2<br/>object storage")]:::data
    end

    subgraph CLOUD3 ["🔐 Servicios externos"]
        CLERKN["🪪 Clerk"]:::ext
        KMSN["🔐 AWS KMS"]:::ext
        PRIVYN["👛 Privy"]:::ext
        ANCHORN["🏦 Anchor SEP<br/>(sandbox)"]:::ext
    end

    subgraph STELLARNET ["⛓️ Stellar testnet"]
        HORIZON["🌅 Horizon API"]:::ext
        SOROBAN["📜 Soroban RPC"]:::ext
        EXPERT["🔗 Stellar Expert"]:::ext
    end

    BR -->|HTTPS| WEBN
    MOB -->|HTTPS 3G| WEBN
    BR -.docs.-> DOCSN
    WEBN -->|HTTPS REST| APIN
    APIN -->|TLS| NEON
    APIN -->|TLS| UPS
    APIN -->|HTTPS| R2N
    APIN -->|OIDC| CLERKN
    APIN -->|HTTPS API| PRIVYN
    APIN -->|HTTPS API + tenant| SDPN
    WRKN -->|consume cola| UPS
    WRKN -->|TLS| NEON
    WRKN -->|HTTPS| R2N
    WRKN -->|SEP-10/12/38/24| ANCHORN
    WRKN -->|polling| HORIZON
    APIN -->|sign| KMSN
    SDPN --> SDPDB
    SDPN -->|submit tx| HORIZON
    APIN -->|invoke| SOROBAN
    BR -.verifica sin cuenta.-> EXPERT
```

---

## Nodos y artefactos

| Nodo | Artefacto desplegado | Protocolo entrante | Notas |
|---|---|---|---|
| Vercel | Web Next.js | HTTPS | Edge/CDN; portal interno + público |
| Cloudflare Pages | Docs Site Next.js (static) | HTTPS | Documentación pública para desarrolladores/partners |
| Railway | API NestJS (Docker) | HTTPS REST | Dominio + orquestación |
| Railway | Workers (Docker) | cola (BullMQ) | Indexador, OCR, PDF, reconciliación |
| Host SDP | SDP + su PostgreSQL | HTTPS API | Servicio aparte, esquemas por tenant |
| Neon | PostgreSQL + RLS | TLS 5432 | BD principal multi-tenant |
| Upstash | Redis / BullMQ | TLS | Colas + cache |
| Cloudflare R2 | Object storage | HTTPS S3 | Evidencias y PDFs |
| Clerk | Auth gestionada | OIDC/JWT | Multi-tenant, magic link |
| AWS KMS | Custodia de claves | API firmada | Firma server-side, no-custodial técnico |
| Privy | Wallets embebidas | HTTPS API | Una wallet invisible por área |
| Anchor SEP | Off-ramp fiat (sandbox) | SEP-10/12/38/24 | BlindPay / Abroad |
| Stellar testnet | Horizon · Soroban · Expert | HTTPS | Confirmación y verificación on-chain |

## Atributos de calidad cubiertos

| Atributo | Cómo se logra en el despliegue |
|---|---|
| **Disponibilidad** | Servicios gestionados (Neon/Upstash/R2/Vercel) con SLA; workers desacoplados |
| **Seguridad** | KMS server-side, RLS en Neon, secretos solo en env, testnet |
| **Escalabilidad** | API y workers escalan por separado; colas absorben picos |
| **Resiliencia** | Cola con reintentos + idempotencia; SDP aislado (fallback Privy+anchor) |
| **Verificabilidad** | Cualquiera abre Stellar Expert sin cuenta (control externo) |
| **Aislamiento multi-tenant** | RLS (Neon) + `org_id` en API + esquemas por tenant (SDP) |
