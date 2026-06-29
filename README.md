# TrustBid — Docs

Documentación de producto y estrategia de **TrustBid**: la capa de transparencia y
trazabilidad de fondos para ONGs sobre Stellar.

## Índice

| Documento | Qué es |
|---|---|
| [integraciones-stellar.md](./integraciones-stellar.md) | **Mapa técnico**: SEPs y herramientas de Stellar (SEP‑1/7/8/9/10/12/24, Stellar Wallets Kit, SDP, Soroban SBT/ZK, MPP) mapeadas a los procesos de TrustBid —reputación, auditoría, gestión financiera, donaciones y anti‑suplantación—, con roadmap por fases. |
| [ecosistema-stellar.md](./ecosistema-stellar.md) | **Estrategia de composabilidad**: cómo TrustBid pasa de app a infraestructura que otros proyectos, protocolos y developers de Stellar consumen. |

## Arquitectura (`arquitectura/`)

> Artefactos UML/C4 coherentes y trazables entre sí. Orden de lectura recomendado:
> casos de uso → DER → clases → secuencia → componentes → despliegue.

| Documento | Qué es |
|---|---|
| [arquitectura/casos-de-uso.md](./arquitectura/casos-de-uso.md) | 7 actores · 37 casos de uso · 7 módulos (M1–M7). |
| [arquitectura/der.md](./arquitectura/der.md) | Modelo entidad-relación (23 tablas) en 3 vistas por dominio. |
| [arquitectura/diagrama-clases.md](./arquitectura/diagrama-clases.md) | Visión OOP completa, multi-tenant con RLS. |
| [arquitectura/diagrama-secuencia.md](./arquitectura/diagrama-secuencia.md) | 4 flujos dinámicos (gasto+OCR+anclaje, desembolso+off-ramp, verificación donante, reporte) con control interno e idempotencia. |
| [arquitectura/diagrama-componentes.md](./arquitectura/diagrama-componentes.md) | Modelo C4 (contenedores + componentes de la API), puertos/adaptadores. |
| [arquitectura/diagrama-despliegue.md](./arquitectura/diagrama-despliegue.md) | Vista física: Vercel, Railway, SDP, Neon, Upstash, R2, KMS, Stellar testnet. |
| [arquitectura/checklist-integraciones-stellar.md](./arquitectura/checklist-integraciones-stellar.md) | Checklist de implementación por fase + 5 flujos de secuencia (identidad/login, donación, KYC/KYB, desembolso SDP, auditoría) para las piezas de [integraciones-stellar.md](./integraciones-stellar.md). |

> Verificado contra la documentación oficial de Stellar. Las fuentes están enlazadas
> al final de cada documento.
