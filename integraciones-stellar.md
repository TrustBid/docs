# Integraciones Stellar — TrustBid

> Documento técnico de integración. Mapea cada pieza del ecosistema Stellar a los
> procesos de valor de TrustBid: **reputación**, **auditoría** y **gestión financiera**,
> más **donaciones/crowdfunding** y **anti‑suplantación**.
> Última revisión: 2026‑06. Verificado contra docs oficiales (ver Fuentes).

---

## 1. Cómo encaja Stellar con TrustBid

TrustBid necesita tres cosas que Stellar resuelve de forma nativa:

1. **Un libro inmutable y en vivo** donde cada fondo deja rastro → el *ledger* de Stellar (rápido, ~5 s, comisiones ~$0.00001).
2. **Un activo estable** para mover dinero real sin volatilidad → **USDC en Stellar** (emitido por Circle).
3. **Estándares de identidad y cumplimiento** ya escritos (los SEP) → no reinventamos KYC/KYB, auth ni rampas.

La capa de "confianza" (insignias, pruebas, reputación) se construye sobre **Soroban** (los smart contracts de Stellar).

### Mapa rápido

| Necesidad de TrustBid | Pieza Stellar | Tipo | Prioridad |
|---|---|---|---|
| Identidad de la organización + anti‑suplantación | **SEP‑1** (stellar.toml) | Estándar | Fase 1 |
| Login que prueba control de la cuenta | **SEP‑10** (Web Auth) / **SEP‑45** (Soroban) | Estándar | Fase 1 |
| KYC de personas y **KYB** de organizaciones | **SEP‑12 + SEP‑9** | Estándar | Fase 1–2 |
| Donar desde la plataforma (conectar wallet) | **Stellar Wallets Kit (SAK)** | Librería | Fase 1 |
| Links/QR de donación | **SEP‑7** (URI scheme) | Estándar | Fase 1 |
| Donar con tarjeta/fiat → USDC | **SEP‑24 / SEP‑6** (rampas) | Estándar | Fase 2 |
| Reputación: insignias/ACTAs verificables | **Soroban SBT** (patrón Chaincerts) | Smart contract | Fase 2 |
| Reputación: probar sin exponer datos | **Soroban ZK** (BLS12‑381 / Groth16) | Smart contract | Fase 3 |
| Desembolso masivo a beneficiarios | **Stellar Disbursement Platform (SDP)** | Plataforma | Fase 2 |
| Aprobar/regular cada transferencia | **SEP‑8** (regulated assets) | Estándar | Opcional |
| Pagos por API / agentes IA (futuro) | **MPP** (Machine Payments Protocol) | Protocolo | Exploratorio |
| Auditoría y reportes | **Horizon API + datos del ledger** | API | Fase 1 |

---

## 2. Identidad y anti‑suplantación

El objetivo: que un donante o auditor pueda comprobar que "esta organización es quien dice ser" sin confiar a ciegas.

### SEP‑1 — `stellar.toml`
La organización publica un archivo en `https://sudominio.org/.well-known/stellar.toml` que declara su identidad, sus cuentas/activos, claves de firma y la URL de su servidor KYC.
- **En TrustBid:** cada ONG verificada tiene su `stellar.toml`. Vincula el **dominio real** (controlado por la org) con sus **cuentas Stellar**. Suplantar implicaría controlar el dominio — primera barrera anti‑fraude.

### SEP‑10 — Stellar Web Authentication
Login por reto‑respuesta: el servidor manda una transacción que solo el dueño de la cuenta puede firmar; al firmarla, recibe un token de sesión (JWT). Prueba **control de la cuenta**, no solo conocimiento de una contraseña.
- **En TrustBid:** autentica a ONGs y wallets antes de cualquier acción sensible (ver fondos, emitir insignias, pedir KYB). Es el prerrequisito de SEP‑12.
- **SEP‑45** es la variante equivalente para cuentas‑contrato de Soroban; usarla si la cuenta es un smart wallet.

### SEP‑12 + SEP‑9 — KYC / **KYB**
- **SEP‑9** define el catálogo estándar de campos (persona y **organización**: `organization.name`, registro, dirección, etc.).
- **SEP‑12** es la API (`PUT/GET /customer`) para enviar y consultar esa información de forma estructurada. Requiere SEP‑10.
- **En TrustBid:** corremos (o integramos) un `KYC_SERVER` declarado en el `stellar.toml`. Para **KYB**, recogemos los campos de organización de SEP‑9 y los validamos contra un proveedor de verificación de negocios (Sumsub, Persona, etc.) o manualmente en fase 1. El resultado del KYB alimenta directamente la **insignia de reputación** (sección 3).

> **Resumen anti‑suplantación:** SEP‑1 (dominio) + SEP‑10 (control de cuenta) + SEP‑12/KYB (negocio validado) + SBT de reputación (sección 3). Cuatro capas, no una.

---

## 3. Reputación verificable (el diferenciador)

Aquí vive el valor único de TrustBid. Dos tecnologías de Soroban lo hacen real:

### Insignias / "ACTAs" como Soulbound Tokens (SBT)
Un **SBT** es un token **no transferible** ligado a una cuenta — perfecto para credenciales: no se pueden vender ni falsificar, y se verifican leyendo la cadena sin llamar a la entidad emisora.
- **Patrón de referencia:** **Chaincerts** ya emite certificaciones verificables no transferibles vía contratos Soroban.
- **En TrustBid:** cada vez que una ONG **cumple un objetivo** (hito de un proyecto, ejecución correcta de fondos, auditoría aprobada), TrustBid emite una **insignia/ACTA** como SBT a su cuenta. Un donante ve la cuenta y comprueba, on‑chain, qué objetivos cumplió — *ve, no cree*.
- Anti‑sybil opcional: **Human Passport** registra verificaciones como SBT en Stellar; sirve para reforzar que detrás hay una entidad real.

### Pruebas de conocimiento cero (ZK) sobre Soroban
Desde **Protocol 22 (CAP‑0059)**, Soroban incorpora funciones de host para la curva **BLS12‑381**, lo que permite **verificar zk‑SNARKs (Groth16) on‑chain**. Existe un contrato de ejemplo (`groth16_verifier`) y un prototipo de *Privacy Pools* de la propia Stellar.
- **En TrustBid:** probar afirmaciones **sin exponer datos privados**. Ej.: "este proyecto gastó dentro del presupuesto y cumplió los criterios del financiador" se puede **demostrar y verificar en cadena** sin publicar el detalle financiero sensible. Es el sustento técnico real de tu mensaje *"verificable sin exponer tus datos"*.
- **Madurez:** es lo más avanzado del stack; va en Fase 3, tras tener SBT y auditoría funcionando.

---

## 4. Donaciones / crowdfunding desde la plataforma

El objetivo: que cualquiera done desde TrustBid, que sea verídico y trazable.

### Stellar Wallets Kit (SAK)
Librería (`@creit.tech/stellar-wallets-kit`) que conecta **todas** las wallets de Stellar con una sola API: **Freighter, xBull, Albedo, Rabet y WalletConnect**. Maneja la conexión y la firma; la UI la controlamos nosotros.
- **En TrustBid:** botón "Donar" → el donante conecta su wallet → firma la transferencia de USDC al proyecto. La donación queda firmada por el donante (verídica) y en el ledger (trazable). Es la pieza central del crowdfunding.

### SEP‑7 — URI scheme de pagos
Genera links `web+stellar:pay?...` y **códigos QR** con destino, monto y memo predefinidos.
- **En TrustBid:** cada campaña/proyecto tiene su link y QR de donación (compartible en redes, email, físico). El `memo` etiqueta la donación al proyecto correcto → trazabilidad automática.

### USDC en Stellar
Activo estable para donar/mover valor sin volatilidad. El donante necesita una *trustline* a USDC (la wallet la crea).

### SEP‑24 / SEP‑6 — rampas (fiat ↔ USDC)
Permiten **donar con tarjeta o transferencia** (fiat) y que llegue como USDC, vía un *anchor*. SEP‑24 es flujo interactivo (hosted); SEP‑6 es programático.
- **En TrustBid:** Fase 2, para donantes sin cripto. Integramos un anchor regional (LatAm) que ofrezca on‑ramp a USDC.

### MPP — Machine Payments Protocol *(exploratorio)*
Protocolo de **pagos por‑request sobre HTTP** (extiende el código 402) pensado para **agentes IA y APIs**; liquida con Soroban SAC, soporta USDC y comisiones patrocinadas. Dos modos: *charge* (pago por petición) y *session* (canal con micro‑pagos off‑chain).
- **Relevancia para TrustBid (honesta):** **no es necesario** para las donaciones v1. Encaja a futuro en dos casos: (a) **monetizar nuestra API** de datos/verificación por consulta, y (b) **donaciones iniciadas por agentes IA** (un agente que asigna fondos a proyectos automáticamente). Mantener en radar, no en el camino crítico.

---

## 5. Desembolso de fondos a beneficiarios

### Stellar Disbursement Platform (SDP)
Plataforma open‑source de la Stellar Development Foundation para **pagos masivos**: se sube una lista de receptores y montos (hasta **10.000 pagos por lote**) y se desembolsa. Ya integra **SEP‑10 y SEP‑24 de forma nativa** (sin Anchor Platform externo). El receptor **registra su wallet** mediante un *deeplink*/OTP y confirma sus datos (teléfono, fecha de nacimiento) directamente con la SDP, **sin compartirlos con la wallet**.
- **En TrustBid:** cuando una ONG distribuye ayuda a muchos beneficiarios (subsidios, becas, pagos de campo), usamos la SDP. Cada pago queda en el ledger → entra directo a **auditoría y reportes**. Es el complemento natural del crowdfunding: entra dinero (sección 4), sale a beneficiarios (sección 5), todo trazado.

### SEP‑8 — Regulated Assets *(opcional)*
El emisor del activo aprueba (o rechaza) **cada transferencia** vía un *approval server*.
- **En TrustBid:** si un financiador exige que los fondos solo lleguen a destinatarios pre‑aprobados, SEP‑8 hace cumplir esa regla a nivel de protocolo.

---

## 6. Auditoría y trazabilidad

No es un SEP: es leer el ledger.
- **Horizon API** (y/o datos de Soroban) expone todo el historial de transacciones de cada cuenta/proyecto, en vivo.
- **En TrustBid:** indexamos los movimientos de cada proyecto y generamos los **informes financieros certificados, segmentados por periodo, y la rendición de cuentas** que las ONGs deben presentar. Como la fuente es el ledger inmutable, el informe es **verificable de forma independiente** — el auditor puede contrastar contra la cadena. Esto materializa el pilar *"Auditoría sin sustos"* y *"Finanzas listas para presentar"*.

---

## 7. Stack y herramientas

| Capa | Herramienta |
|---|---|
| SDK base (JS/TS) | `@stellar/stellar-sdk` |
| Conexión de wallets | `@creit.tech/stellar-wallets-kit` |
| Smart contracts (SBT, ZK, lógica) | **Soroban** (Rust) + `soroban-examples` (`groth16_verifier`) |
| Auth/KYC server | **Anchor Platform** (implementa SEP‑10/12/24/6) o propio |
| Desembolsos | **stellar-disbursement-platform-backend** |
| Pagos agénticos (futuro) | `@stellar/mpp` / `stellar-mpp-sdk` |
| Datos/auditoría | **Horizon API** |

---

## 8. Roadmap sugerido por fases

- **Fase 1 — Base de confianza y donar.** SEP‑1 + SEP‑10 + Stellar Wallets Kit + SEP‑7 + USDC + lectura de ledger para auditoría básica. *(Permite: identidad, donar desde la plataforma, trazabilidad.)*
- **Fase 2 — Cumplimiento y escala.** SEP‑12/KYB + SDP (desembolsos) + rampas SEP‑24 + insignias SBT (ACTAs). *(Permite: KYB, repartir fondos, reputación verificable, donar con fiat.)*
- **Fase 3 — Privacidad avanzada.** Pruebas ZK (Soroban BLS12‑381) + SEP‑8 si algún financiador lo exige. *(Permite: probar cumplimiento sin exponer datos.)*
- **Exploratorio:** MPP para API monetizada / agentes IA.

---

## 9. Consideraciones

- **Custodia de claves:** definir si TrustBid es no‑custodial (el usuario firma con su wallet, recomendado para donaciones) o custodial (TrustBid administra cuentas, implica más responsabilidad regulatoria). Probablemente **híbrido**: no‑custodial para donantes, gestionado para desembolsos vía SDP.
- **KYB real:** SEP‑9/12 transportan los datos, pero la **validación** del negocio la hace un proveedor (Sumsub/Persona) o un proceso manual en fase 1. SEP‑12 es el "cómo se mueven", no el "quién valida".
- **Regulación:** mover USDC y hacer rampas fiat puede requerir licencias según jurisdicción; coordinar con el anchor regional.
- **Testnet primero:** todo lo anterior tiene red de pruebas (Testnet/Futurenet) — construir y demostrar ahí antes de mainnet.

---

## Fuentes

- [MPP on Stellar — Stellar Docs](https://developers.stellar.org/docs/build/agentic-payments/mpp)
- [Agentic Payments — Stellar Docs](https://developers.stellar.org/docs/build/agentic-payments)
- [Stellar Disbursement Platform — Stellar.org](https://stellar.org/products-and-tools/disbursement-platform)
- [SDP Architecture — Stellar Docs](https://developers.stellar.org/docs/platforms/stellar-disbursement-platform/admin-guide/design-and-architecture)
- [Stellar Wallets Kit — GitHub (Creit‑Tech)](https://github.com/Creit-Tech/Stellar-Wallets-Kit) · [npm](https://www.npmjs.com/package/@creit.tech/stellar-wallets-kit)
- [SEP‑0012 (KYC API) — stellar‑protocol](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0012.md)
- [SEP‑0009 (Standard KYC Fields) — stellar‑protocol](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0009.md)
- [Announcing Protocol 22 (BLS12‑381 / CAP‑0059) — Stellar.org](https://stellar.org/blog/developers/announcing-protocol-22)
- [Prototyping Privacy Pools on Stellar — Stellar.org](https://stellar.org/blog/ecosystem/prototyping-privacy-pools-on-stellar)
- [groth16_verifier — soroban-examples](https://github.com/stellar/soroban-examples/tree/main/groth16_verifier)
- [Building on Soroban / Chaincerts (SBT) — Stellar.org](https://stellar.org/blog/developers/building-on-soroban-three-teams-journeys-smart-contracts)
- [Stellar Integration — Human Passport (SBT)](https://docs.passport.xyz/building-with-passport/individual-verifications/supported-chains/stellar)
