# TrustBid — Documentación

**TrustBid** es la capa de transparencia y trazabilidad de fondos para ONGs, construida sobre Stellar.  
Cada peso donado queda registrado on-chain. Cada gasto, verificado.

---

## Para desarrolladores y socios

| Documento | Descripción |
|---|---|
| [stellar-integrations-public.md](./stellar-integrations-public.md) | Cómo TrustBid usa el ecosistema Stellar — SEPs, Soroban, SDP, wallets. Lectura recomendada para integradores. |
| [public-api-spec.md](./public-api-spec.md) | **TrustBid Public API** — API abierta para que otros proyectos y devs de Stellar consulten trazabilidad, identidad de ONGs y cumplimiento de objetivos. |

## Arquitectura — modelo 4+1 (estado actual, verificado contra el código)

> **Punto de entrada recomendado.** Documenta la implementación tal como está hoy
> (relevamiento del 2026-08-03), no el diseño previsto.

| Documento | Descripción |
|---|---|
| [arquitectura/4+1/README.md](./arquitectura/4+1/README.md) | Índice del modelo 4+1 · inventario de lo implementado vs. lo diseñado · contratos desplegados |
| [arquitectura/4+1/1-vista-logica.md](./arquitectura/4+1/1-vista-logica.md) | Dominio, roles, servicios, contratos Soroban, máquinas de estado, modelo de datos y multi-tenancy |
| [arquitectura/4+1/2-vista-procesos.md](./arquitectura/4+1/2-vista-procesos.md) | Runtime: proceso único + BullMQ + EventEmitter, anclaje asíncrono, latencias, idempotencia |
| [arquitectura/4+1/3-vista-desarrollo.md](./arquitectura/4+1/3-vista-desarrollo.md) | Monorepo, dependencias entre paquetes, cadena Caatinga, CI/CD, testing, deuda técnica |
| [arquitectura/4+1/4-vista-fisica.md](./arquitectura/4+1/4-vista-fisica.md) | Nodos (Cloudflare · Railway · Neon · Upstash · R2 · Stellar), variables, seguridad, camino a mainnet |
| [arquitectura/4+1/5-escenarios.md](./arquitectura/4+1/5-escenarios.md) | 8 escenarios end-to-end con criterios de aceptación + matriz de cobertura |

## Arquitectura interna — artefactos UML/C4

> Detalle por artefacto. Trazables entre sí y referenciados desde las vistas 4+1.  
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
| [arquitectura/backend-spec.md](./arquitectura/backend-spec.md) | Especificación técnica del backend NestJS: módulos, endpoints, contratos Soroban ⚠️ *documento de diseño — incluye módulos aún no implementados* |
| [arquitectura/informe-pruebas-contratos.md](./arquitectura/informe-pruebas-contratos.md) | Evidencia de las 32 pruebas unitarias de los contratos Soroban + deploy en testnet |

---

## Stack

| Capa | Tecnología |
|---|---|
| Frontend DApp | Next.js 16 · React 19 · TypeScript · shadcn/radix · Tailwind 4 |
| Backend | NestJS 11 · TypeScript · BullMQ · class-validator · `pg` (sin ORM) |
| Base de datos | Neon (PostgreSQL 16) · aislamiento por `organization_id` en cada query |
| Auth | SEP-10 (wallet challenge-response) · Privy (email/OTP + wallet embebida) · JWT |
| Wallets | Stellar Wallets Kit v2 · Freighter · Albedo · Privy embedded |
| IA | Google Gemini (OCR y extracción estructurada de facturas) |
| Mensajería | WhatsApp Cloud API · Telegram Bot API (interfaz `BotChannel` común) |
| Storage | Cloudflare R2 (comprobantes *content-addressed* por SHA-256) |
| Colas | Upstash Redis · BullMQ (vigilancia de donaciones en Horizon) |
| Blockchain | Stellar Testnet · Soroban RPC · Horizon API |
| Contratos | Rust / soroban-sdk · orquestación con Caatinga 3.7 |
| Hosting | Cloudflare Worker (dapp, vía OpenNext) · Cloudflare Pages (landing + docs) · Railway (API) |

---

> Documentación verificada contra la documentación oficial de Stellar.  
> Fuentes enlazadas al final de cada documento.
