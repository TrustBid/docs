# TrustBid Public API — Especificación

> **Estado:** Propuesta · Sprint 5  
> **Audiencia:** Desarrolladores externos, protocolos Stellar, ONGs integradoras, auditores.

---

## Propósito

La TrustBid Public API expone datos de trazabilidad, identidad y cumplimiento de las ONGs que operan en TrustBid, **sin autenticación**, para que cualquier developer, protocolo o solución del ecosistema Stellar pueda:

- Verificar que una ONG existe y está validada.
- Consultar el destino de fondos de un proyecto específico.
- Confirmar si un objetivo fue cumplido y tiene evidencia on-chain.
- Componer TrustBid como fuente de verdad de transparencia dentro de sus propias aplicaciones.

---

## Base URL

```
https://api.trustbid.io/v1/public
```

> Todos los endpoints son `GET`, sin autenticación, con rate limit de **60 req/min por IP**.  
> Respuestas en JSON. Devolución de campos sensibles nunca incluye datos personales de beneficiarios.

---

## Recursos

### 1. Organizaciones (NGO Identity)

#### `GET /v1/public/orgs`

Lista las ONGs registradas y verificadas en TrustBid.

**Query params:**

| Param | Tipo | Descripción |
|---|---|---|
| `verified` | `boolean` | Filtrar solo ONGs con SBT de verificación emitido |
| `country` | `string` | ISO-3166 de 2 letras (ej. `CO`, `MX`, `AR`) |
| `page` | `number` | Paginación (default 1) |
| `limit` | `number` | Máx. 50 (default 20) |

**Respuesta:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "LATIR ONG",
      "slug": "latir-ong",
      "country": "CO",
      "stellar_address": "G...",
      "verified": true,
      "sbt_contract": "C...",
      "active_projects": 3,
      "total_disbursed_usdc": "12500.00",
      "created_at": "2025-01-15T00:00:00Z"
    }
  ],
  "meta": { "total": 12, "page": 1, "limit": 20 }
}
```

---

#### `GET /v1/public/orgs/:slug`

Perfil completo de una ONG por su slug.

**Respuesta:**
```json
{
  "id": "uuid",
  "name": "LATIR ONG",
  "slug": "latir-ong",
  "country": "CO",
  "stellar_address": "G...",
  "stellar_toml": "https://trustbid.io/.well-known/latir-ong.toml",
  "verified": true,
  "sbt_contract": "C...",
  "sbt_issued_at": "2025-03-01T00:00:00Z",
  "kyb_status": "approved",
  "projects_summary": {
    "total": 5,
    "active": 3,
    "completed": 2
  },
  "financials": {
    "total_received_usdc": "45000.00",
    "total_disbursed_usdc": "38500.00",
    "total_anchored_txs": 147
  }
}
```

---

#### `GET /v1/public/orgs/:slug/verify`

Verificación rápida de identidad — ideal para integraciones en tiempo real.

**Respuesta:**
```json
{
  "slug": "latir-ong",
  "stellar_address": "G...",
  "verified": true,
  "sbt_contract": "C...",
  "verification_hash": "0x...",
  "checked_at": "2026-06-30T12:00:00Z"
}
```

> El `verification_hash` es el hash SHA-256 del SBT on-chain — puede verificarse directamente en Soroban sin pasar por TrustBid.

---

### 2. Proyectos (Fund Traceability)

#### `GET /v1/public/projects`

Lista proyectos públicos con trazabilidad habilitada.

**Query params:**

| Param | Tipo | Descripción |
|---|---|---|
| `org` | `string` | Slug de la ONG |
| `category` | `string` | `education`, `health`, `infrastructure`, `technology`, `environment`, `social` |
| `status` | `string` | `active`, `completed` |
| `country` | `string` | ISO-3166 |
| `page` / `limit` | `number` | Paginación |

**Respuesta:**
```json
{
  "data": [
    {
      "id": "uuid",
      "name": "Escuela San Pedro — Fase 1",
      "org_slug": "latir-ong",
      "category": "education",
      "status": "active",
      "budget_usdc": "25000.00",
      "spent_usdc": "18200.00",
      "completion_pct": 73,
      "blockchain_enabled": true,
      "current_stage": "Ejecución",
      "beneficiary_count": 240,
      "start_date": "2025-02-01",
      "end_date": "2025-12-31"
    }
  ],
  "meta": { "total": 8, "page": 1, "limit": 20 }
}
```

---

#### `GET /v1/public/projects/:id`

Detalle completo de un proyecto.

**Respuesta:**
```json
{
  "id": "uuid",
  "name": "Escuela San Pedro — Fase 1",
  "description": "Construcción de aulas y equipamiento básico...",
  "org": {
    "slug": "latir-ong",
    "name": "LATIR ONG",
    "verified": true
  },
  "category": "education",
  "status": "active",
  "budget_usdc": "25000.00",
  "spent_usdc": "18200.00",
  "completion_pct": 73,
  "blockchain_enabled": true,
  "pipeline": {
    "current_stage": "Ejecución",
    "stages": [
      { "name": "Planificación", "completed": true, "anchored_at": "2025-02-10T..." },
      { "name": "Fondeo",        "completed": true, "anchored_at": "2025-02-20T..." },
      { "name": "Ejecución",     "completed": false, "anchored_at": null },
      { "name": "Verificación",  "completed": false, "anchored_at": null },
      { "name": "Cierre",        "completed": false, "anchored_at": null }
    ]
  },
  "impact": {
    "beneficiary_count": 240,
    "indicators": [
      { "name": "Aulas construidas", "target": 4, "achieved": 3 },
      { "name": "Niños beneficiados", "target": 240, "achieved": 180 }
    ]
  }
}
```

---

#### `GET /v1/public/projects/:id/transactions`

Lista de transacciones on-chain asociadas al proyecto, verificables en Stellar Explorer.

**Respuesta:**
```json
{
  "data": [
    {
      "memo_id": "PAY-2025-0012",
      "amount_usdc": "3200.00",
      "status": "confirmed",
      "tx_hash": "a1b2c3...",
      "stellar_explorer_url": "https://stellar.expert/explorer/testnet/tx/a1b2c3...",
      "executed_at": "2025-04-15T10:23:00Z",
      "description": "Pago contratista — cimientos Bloque B"
    }
  ],
  "meta": { "total": 14, "page": 1, "limit": 50 }
}
```

---

### 3. Cumplimiento y objetivos (Compliance)

#### `GET /v1/public/projects/:id/compliance`

Resumen de cumplimiento verificable on-chain: si la ONG cumplió los objetivos comprometidos.

**Respuesta:**
```json
{
  "project_id": "uuid",
  "project_name": "Escuela San Pedro — Fase 1",
  "org_verified": true,
  "budget_compliance": {
    "approved_usdc": "25000.00",
    "spent_usdc": "18200.00",
    "variance_pct": -1.2,
    "within_tolerance": true
  },
  "milestone_compliance": {
    "total": 5,
    "completed": 2,
    "completion_pct": 40,
    "all_anchored_on_chain": true
  },
  "document_integrity": {
    "total_documents": 8,
    "documents_with_hash": 8,
    "hashes_verifiable_on_chain": true
  },
  "overall_status": "on_track",
  "last_verified_at": "2026-06-30T12:00:00Z"
}
```

**Valores de `overall_status`:** `on_track` · `at_risk` · `completed` · `failed`

---

### 4. Webhook (push en lugar de pull)

Para integraciones que necesitan reaccionar a eventos sin polling:

#### `POST /v1/public/webhooks`

Requiere API key (gratuita, se solicita en el portal de TrustBid).

**Body:**
```json
{
  "url": "https://mi-app.com/trustbid-events",
  "events": ["project.milestone_completed", "project.transaction_confirmed", "org.verified"],
  "org_slug": "latir-ong"
}
```

**Eventos disponibles:**

| Evento | Descripción |
|---|---|
| `org.verified` | Una ONG recibió su SBT de verificación |
| `org.kyb_updated` | El estado KYB de una ONG cambió |
| `project.created` | Nuevo proyecto con blockchain habilitado |
| `project.milestone_completed` | Una etapa del pipeline fue completada y anclada |
| `project.transaction_confirmed` | Una transacción fue confirmada en Stellar |
| `project.completed` | El proyecto cerró con todos los hitos cumplidos |

---

## Casos de uso para el ecosistema Stellar

### Para otros proyectos / protocolos

```
// Verificar antes de permitir una donación
GET /v1/public/orgs/latir-ong/verify
→ { verified: true, sbt_contract: "C..." }

// Mostrar progreso en tiempo real en un portal de donantes
GET /v1/public/projects/:id
→ pipeline actual + indicadores de impacto

// Auditoría automática mensual
GET /v1/public/projects/:id/compliance
→ budget_compliance + milestone_compliance
```

### Para billeteras Stellar

Una wallet puede mostrar un badge "ONG Verificada por TrustBid" junto a la dirección de destino antes de confirmar una transacción, consultando `GET /v1/public/orgs/:slug/verify` con la dirección Stellar.

### Para plataformas de crowdfunding

Integrar TrustBid como capa de reporting: las donaciones se procesan externamente, pero la trazabilidad del destino se consulta directamente de la Public API y se muestra al donante.

### Para auditores y organismos reguladores

Acceso programático al historial completo de transacciones on-chain de una ONG, con hashes verificables independientemente en Stellar Explorer.

---

## Implementación técnica

### Stack propuesto

- **Framework:** NestJS (mismo monorepo `platform/apps/api`, nuevo módulo `PublicApiModule`)
- **Rate limiting:** `@nestjs/throttler` — 60 req/min por IP, configurable por API key
- **Caché:** Upstash Redis — TTL de 60 segundos en endpoints de consulta frecuente
- **Versionado:** `/v1/` en la URL, `X-TrustBid-API-Version` header en respuestas
- **Documentación interactiva:** Swagger UI en `api.trustbid.io/v1/public/docs`

### Módulos NestJS a crear

```
apps/api/src/modules/
└── public-api/
    ├── public-api.module.ts
    ├── orgs.controller.ts        # /v1/public/orgs
    ├── projects.controller.ts    # /v1/public/projects
    ├── compliance.controller.ts  # /v1/public/projects/:id/compliance
    ├── webhooks.controller.ts    # /v1/public/webhooks
    └── public-api.service.ts
```

### Consideraciones de privacidad

- Los datos de **beneficiarios individuales** nunca se exponen — solo conteos agregados.
- Los **nombres de transacciones** pueden ocultarse con ZK proofs (Sprint 5) manteniendo la verificabilidad del monto y hash.
- Las ONGs pueden marcar proyectos como `public: false` — no aparecen en la API pero sí en auditorías internas.
- Cumplimiento con GDPR/LGPD: ningún dato personal se almacena junto a los hashes on-chain.

---

## Roadmap de implementación

| Sprint | Entregable |
|---|---|
| Sprint 5 | Módulo `public-api` · endpoints `/orgs` y `/projects` · rate limiting · Swagger |
| Sprint 6 | Endpoint `/compliance` · webhook system · portal de API keys |
| Sprint 7 | ZK proofs en transacciones sensibles · SDK JavaScript/TypeScript |
| Sprint 8 | SDK Python · integración piloto con wallet o plataforma de donaciones |

---

## Contacto para integraciones

Para acceder a webhooks o hablar sobre una integración específica:  
**teamtrustbid@gmail.com** · [trustbid.pages.dev](https://trustbid.pages.dev)
