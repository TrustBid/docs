# TrustBid — Documentación

**TrustBid** es la capa de transparencia y trazabilidad de fondos para ONGs, construida sobre Stellar.  
Cada peso donado queda registrado on-chain. Cada gasto, verificado.

---

## Para desarrolladores y socios

| Documento | Descripción |
|---|---|
| [stellar-integrations-public.md](./stellar-integrations-public.md) | Cómo TrustBid usa el ecosistema Stellar — SEPs, Soroban, SDP, wallets. Lectura recomendada para integradores. |
| [public-api-spec.md](./public-api-spec.md) | **TrustBid Public API** — API abierta para que otros proyectos y devs de Stellar consulten trazabilidad, identidad de ONGs y cumplimiento de objetivos. |

## Arquitectura interna

> Artefactos UML/C4 trazables entre sí.  
> Orden de lectura: casos de uso → DER → clases → secuencia → componentes → despliegue.

| Documento | Descripción |
|---|---|
| [arquitectura/casos-de-uso.md](./arquitectura/casos-de-uso.md) | 7 actores · 37 casos de uso · 7 módulos (M1–M7) |
| [arquitectura/der.md](./arquitectura/der.md) | Modelo entidad-relación (26 tablas) en 4 vistas por dominio |
| [arquitectura/diagrama-clases.md](./arquitectura/diagrama-clases.md) | Dominio OOP completo, multi-tenant con RLS por `organization_id` |
| [arquitectura/diagrama-secuencia.md](./arquitectura/diagrama-secuencia.md) | 4 flujos dinámicos: gasto+OCR+anclaje, desembolso+off-ramp, verificación donante, reporte |
| [arquitectura/diagrama-componentes.md](./arquitectura/diagrama-componentes.md) | Modelo C4: contenedores + componentes de la API, puertos/adaptadores |
| [arquitectura/diagrama-despliegue.md](./arquitectura/diagrama-despliegue.md) | Vista física: Railway, Neon, Upstash, Cloudflare R2, AWS KMS, Stellar Testnet |
| [arquitectura/flujos-integraciones-stellar.md](./arquitectura/flujos-integraciones-stellar.md) | Diagramas de secuencia de los 9 flujos Stellar (A–I): login SEP-10, KYB, donación, SDP, auditoría |
| [arquitectura/backend-spec.md](./arquitectura/backend-spec.md) | Especificación técnica del backend NestJS: módulos, endpoints, contratos Soroban |

---

## Stack

| Capa | Tecnología |
|---|---|
| Frontend DApp | Next.js 15 · React 19 · TypeScript · shadcn/ui · Tailwind |
| Backend | NestJS 11 · TypeScript · BullMQ |
| Base de datos | Neon (PostgreSQL 16) · RLS por `organization_id` |
| Auth | SEP-10 (wallet challenge-response) · JWT |
| Wallets | Stellar Wallets Kit v2 · Freighter · Albedo |
| Storage | Cloudflare R2 |
| Blockchain | Stellar Testnet · Soroban RPC · SDP · Horizon API |
| Hosting | Railway (API + DApp) · Cloudflare Pages (landing + docs) |

---

> Documentación verificada contra la documentación oficial de Stellar.  
> Fuentes enlazadas al final de cada documento.
