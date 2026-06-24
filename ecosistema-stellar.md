# TrustBid en el ecosistema Stellar — Estrategia de composabilidad

> Cómo TrustBid pasa de ser **una app que verifica ONGs** a ser **infraestructura de
> confianza** que otros proyectos, protocolos y developers de Stellar consumen.
> Documento de estrategia. Última revisión: 2026‑06.
> Complementa a [`integraciones-stellar.md`](./integraciones-stellar.md).

---

## 1. Tesis: de app a protocolo

El tagline ya lo dice: *"transparency infrastructure"*. La infraestructura no se usa sola —
**otros construyen encima**. El cambio de mentalidad:

> TrustBid no es el destino final del dato. Es la **fuente de verdad que otros consultan**:
> registro on‑chain + esquema estándar + contratos/SDK/API abiertos.

Eso convierte a TrustBid de *app* en *protocolo*, y ahí está el foso defensivo.

**Analogía:** en Ethereum, **EAS / Verax** (Ethereum Attestation Service) son infraestructura
que cientos de apps consultan. **TrustBid puede ser el registro de atestaciones de impacto y
trazabilidad de fondos de Stellar.** Quien define el esquema, define el estándar.

---

## 2. El modelo — capa de atestaciones

Las insignias/ACTAs de TrustBid (Soulbound Tokens en Soroban) y sus pruebas viven **on‑chain,
con un esquema público y estándar**. Eso las vuelve **consultables por cualquiera, sin pedir
permiso a TrustBid**. Ese es el núcleo de todo lo que sigue.

```
        ┌─────────────────────────────────────────────┐
        │   TrustBid: emite atestaciones verificables   │
        │   (KYB, hitos cumplidos, fondos ejecutados)   │
        └───────────────────────┬─────────────────────-─┘
                                │  on-chain, esquema estándar
        ┌───────────────────────┴───────────────────────┐
        │              Capa de lectura pública            │
        └───┬──────────┬───────────┬──────────┬────────-─┘
         wallets    anchors      DeFi/      grants/      watchdogs
                                lending      DAOs        auditores
```

---

## 3. Superficies de integración

### 3.1 Capa de lectura — atestaciones como bien público

Cualquiera lee el estado de verificación de una cuenta y actúa en consecuencia:

| Consumidor | Cómo usa TrustBid |
|---|---|
| **Wallets** (Freighter, Lobstr) | Muestran un badge *"TrustBid Verified"* nativo junto a la cuenta. |
| **Anchors** (LatAm) | Reconocen el **KYB** de TrustBid para sus rampas → KYC reutilizable (dolor real del ecosistema). |
| **DeFi / lending (Soroban)** | Usan la atestación como *colateral de confianza* → préstamos sub‑colateralizados a ONGs verificadas. |
| **Grants / DAOs** | Restringen el desembolso a orgs con el sello de TrustBid. |
| **Watchdogs / auditores / prensa** | Consultan el historial de fondos verificable de forma independiente. |

### 3.2 Primitivas componibles en Soroban

Contratos que otros **llaman desde sus propios contratos**:

- **Registro de atestaciones** (lectura pública): `verify_org(address) -> Status`, `get_badges(address)`.
- **Escrow por hitos:** libera fondos cuando una atestación de TrustBid confirma un *milestone*.
  Cualquier crowdfunding/grant se enchufa sin construir su propia lógica de verificación.
- **Verificador ZK reutilizable** (`groth16_verifier`): otros prueban cumplimiento sin exponer
  datos, reutilizando la primitiva de TrustBid.

### 3.3 SDK + API para developers

- **JS SDK + REST/GraphQL** → un dev añade *"TrustBid Verified"* a su app en pocas líneas.
- **Webhooks** cuando se emite o revoca una atestación.
- **Monetización con MPP** (Machine Payments Protocol): la API se cobra **por consulta**
  (pay‑per‑verification), liquidada en USDC sobre Soroban. Este es el encaje real de MPP en
  el modelo de negocio.

### 3.4 Accountability layer sobre la SDP

La **Stellar Disbursement Platform mueve el dinero hacia afuera**; nadie certifica que
*llegó y funcionó*. **TrustBid es esa capa de prueba.**

> Posicionamiento de una línea: *"el accountability layer encima de la SDP"*.
> Es un encaje narrativo que la Stellar Development Foundation entiende al instante.

---

## 4. Estrategia de estándar

La composabilidad exige **esquemas compartidos**. Si TrustBid define el esquema, TrustBid es
el estándar de facto.

- **Proponer un SEP / esquema** de "atestaciones de impacto y trazabilidad de fondos".
  Ser la implementación de referencia = ser el estándar.
- **Open‑source** de los contratos de atestación → adopción de devs → el tooling de TrustBid
  se vuelve el *default* del ecosistema.

---

## 5. Go‑to‑ecosystem

- **Stellar Community Fund (SCF):** construir un bien público de transparencia financiera para
  LatAm es exactamente lo que la SDF financia y promociona. Aporta fondos + distribución +
  credibilidad.
- **Partnerships de lectura:** integrar el badge en 1–2 wallets y 1 anchor regional crea el
  primer efecto de red.
- **Complemento, no competencia:** posicionarse junto a la SDP (no contra ella) abre la puerta
  a colaboración directa con la SDF.

---

## 6. Posición defensiva (moat)

1. **Estándar:** quien define el esquema de atestaciones gana lock‑in de ecosistema.
2. **Efecto de red:** cada wallet/anchor/protocolo que lee TrustBid aumenta el valor de estar
   verificado por TrustBid.
3. **Datos:** el historial auditado de fondos es un activo acumulativo difícil de replicar.

---

## 7. Próximos pasos

1. Definir y publicar el **esquema de atestación** (campos, versión, revocación).
2. Desplegar el **registro de atestaciones** en Soroban (Testnet) + contrato de lectura pública.
3. Publicar **SDK mínimo** (`verify_org`, `get_badges`) y documentación para devs.
4. Conseguir **1 wallet + 1 anchor** que muestren/consuman el badge (prueba de red).
5. Aplicar a **SCF** con el framing de "accountability layer sobre la SDP".

---

## Relación con otros documentos

- [`integraciones-stellar.md`](./integraciones-stellar.md) — el **qué** técnico: SEPs y
  herramientas de Stellar mapeadas a los procesos de TrustBid.
- Este documento — el **cómo** estratégico: convertir esas integraciones en una capa que el
  ecosistema consume.
