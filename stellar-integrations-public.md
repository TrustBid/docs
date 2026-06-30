# TrustBid × Stellar — Integraciones

> Documento público. Describe cómo TrustBid usa el ecosistema Stellar para
> garantizar transparencia, trazabilidad e identidad verificable en la gestión
> de fondos de ONGs.

---

## Por qué Stellar

Stellar es la infraestructura ideal para TrustBid por tres razones:

1. **Finalidad en ~5 segundos** — cada gasto queda registrado on-chain casi en tiempo real.
2. **Costo mínimo** — fees de fracciones de centavo permiten anclar hasta la transacción más pequeña.
3. **Ecosistema de estándares abiertos** — los SEPs definen protocolos interoperables para identidad, pagos y cumplimiento que TrustBid implementa directamente.

---

## Mapa de integraciones

### Identidad y acceso — SEP-10 + SEP-1

Toda sesión en TrustBid se abre mediante **SEP-10 (Stellar Web Authentication)**:

1. El servidor genera un challenge XDR firmado con su keypair.
2. La wallet del usuario lo firma localmente (Freighter, Albedo u otras via Stellar Wallets Kit v2).
3. El servidor verifica la firma y emite un JWT de sesión.

La clave pública de la organización queda registrada en la DB y se usa como identificador permanente. El servidor publica un `stellar.toml` (SEP-1) con metadata de la plataforma.

**Wallets soportadas:** Freighter · Albedo · xBull · Rabet · cualquier wallet compatible con Stellar Wallets Kit v2.

---

### Donaciones — SDP (Stellar Disbursement Platform)

Los donantes envían fondos en XLM o USDC a la dirección de la organización en Stellar Testnet. TrustBid usa **SDP** para:

- Emitir pagos batch a múltiples beneficiarios.
- Asociar cada desembolso a un proyecto específico mediante `memo_id` con formato `PAY-YYYY-NNNN`.
- Registrar el `tx_hash` en la DB para auditoría.

El proceso es invisible para el donante: solo ve un link de donación y un recibo con el hash de la transacción.

---

### Trazabilidad on-chain — Soroban (contratos inteligentes)

Cada gasto registrado en TrustBid puede anclarse opcionalmente en **Soroban** mediante dos contratos:

| Contrato | Función |
|---|---|
| `fund-tracker` | Registra el hash SHA-256 del gasto, el monto, la etapa del pipeline y el timestamp. |
| `expense-anchor` | Ancla el hash de documentos de respaldo (facturas, fotos) almacenados en R2. |

Los hashes son inmutables una vez escritos. Cualquiera puede verificarlos en Stellar Explorer sin acceder a TrustBid.

---

### KYB organizacional — SEP-12

Las ONGs que quieran recibir fondos de donantes internacionales pueden completar un proceso de **KYB (Know Your Business)** a través de SEP-12:

- La organización sube documentación (acta constitutiva, RUT, etc.) via la dApp.
- Un anchor regulado verifica los documentos.
- Al aprobar, se emite un **SBT (Soulbound Token)** en Soroban que acredita la identidad verificada de la ONG.

El SBT es visible públicamente y no transferible — es prueba criptográfica de que la ONG existe y fue validada.

---

### Firma de transacciones institucionales — AWS KMS

Para organizaciones con billetera custodial (sin wallet propia), TrustBid firma las transacciones Stellar usando **AWS KMS**:

- La clave privada nunca sale del HSM de AWS.
- Cada firma requiere autorización del administrador de la org.
- El log de firmas queda en CloudTrail para auditoría.

---

### Comunicación de estándares — SEP-7 + SEP-24

- **SEP-7**: Los links de donación de TrustBid generan URIs SEP-7 compatibles con cualquier wallet Stellar. El donante solo hace clic y su wallet pre-rellena todos los campos.
- **SEP-24**: Para on-ramp/off-ramp de USDC (entrada de fondos bancarios y retiro a cuenta local), TrustBid usa SEP-24 con anchors regulados.

---

## Roadmap por fases

| Fase | Integraciones |
|---|---|
| **Sprint 1–2** (actual) | SEP-10 (auth) · Horizon API (transacciones) · SDP (desembolsos) |
| **Sprint 3** | SEP-1 (stellar.toml) · SEP-7 (donation links) · SEP-12 (KYB básico) |
| **Sprint 4** | Soroban fund-tracker · SBT de identidad · AWS KMS custodial |
| **Sprint 5** | SEP-24 (off-ramp USDC) · ZK proofs (privacidad de beneficiarios) · Public API |

---

## Recursos

- [Stellar Developer Docs](https://developers.stellar.org)
- [SEP-10 — Web Authentication](https://github.com/stellar/stellar-protocol/blob/master/ecosystem/sep-0010.md)
- [Stellar Wallets Kit](https://github.com/creit-tech/stellar-wallets-kit)
- [Stellar Disbursement Platform](https://github.com/stellar/stellar-disbursement-platform-backend)
- [Soroban Docs](https://soroban.stellar.org)
- [Stellar Expert Explorer (Testnet)](https://stellar.expert/explorer/testnet)
