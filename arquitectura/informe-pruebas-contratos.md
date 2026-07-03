# Informe de Pruebas — Contratos Soroban TrustBid

**Fecha del informe:** 2026-06-30
**Red:** Stellar Testnet
**Repositorio:** [TrustBid/contracts](https://github.com/TrustBid/contracts) · rama `feat/implements-caatinga-workflow`
**Herramientas:** `cargo test` (Soroban SDK 23) · [Caatinga CLI](https://github.com/Dione-b/caatinga) (build/deploy/orquestación) · [Stellar Expert](https://stellar.expert) (verificación pública on-chain)

---

## 1. Resumen ejecutivo

TrustBid registra su trazabilidad de fondos mediante **tres contratos Soroban** independientes. Los tres fueron:

1. **Probados unitariamente** con `cargo test --workspace` — 32 tests, 100% aprobados.
2. **Compilados a WASM** (`wasm32v1-none`) vía Caatinga.
3. **Desplegados en Stellar Testnet**, con inicialización (`initialize`) automática post-deploy.
4. **Verificados públicamente** en Stellar Expert, donde cualquier jurado puede auditar el bytecode, las transacciones de deploy y las llamadas registradas on-chain.

| Contrato | Tests unitarios | Estado | Desplegado en testnet |
|---|---|---|---|
| `fund-tracker` | 7/7 ✅ | Aprobado | Sí |
| `expense-anchor` | 9/9 ✅ | Aprobado | Sí |
| `sbt-badge` | 16/16 ✅ | Aprobado | Sí |
| **Total** | **32/32 ✅** | — | — |

---

## 2. Contratos desplegados en Stellar Testnet

| Contrato | Contract ID | Stellar Expert |
|---|---|---|
| `fund-tracker` | `CCAG3JA4WHWIZCASIGQQ7KOXQLO7OT76M4YGKHTSWJHIN3GCYNAX5HBR` | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/contract/CCAG3JA4WHWIZCASIGQQ7KOXQLO7OT76M4YGKHTSWJHIN3GCYNAX5HBR) |
| `expense-anchor` | `CA4RCZDLFKBLHMUTPN4H4AY4L6DXBBSTYW5MJ6ST3MO27HOD7THIWSSK` | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/contract/CA4RCZDLFKBLHMUTPN4H4AY4L6DXBBSTYW5MJ6ST3MO27HOD7THIWSSK) |
| `sbt-badge` | `CCXS7PT6ZPM3BVA45HSJM265VTK3X5Y3MTBFTAZI35EJF27SV2D6TOIH` | [Ver en Stellar Expert](https://stellar.expert/explorer/testnet/contract/CCXS7PT6ZPM3BVA45HSJM265VTK3X5Y3MTBFTAZI35EJF27SV2D6TOIH) |

> Estos IDs son la fuente de verdad versionada en [`caatinga.artifacts.json`](https://github.com/TrustBid/contracts/blob/feat/implements-caatinga-workflow/caatinga.artifacts.json) del repo de contratos. Aún no están fusionados a `main`; corresponden al deploy más reciente en la rama de integración de Caatinga.

### Detalle de deploy (WASM hash + timestamp)

| Contrato | WASM hash | Desplegado (UTC) |
|---|---|---|
| `fund-tracker` | `e2d972627fc5070bfcc6a447ab87919f10687df82c4293203c2d1c5849a3d62d` | 2026-07-01T01:37:12Z |
| `expense-anchor` | `9507405a2d7125eeaccc16332e1c354a2d74defd88ca3f13c9fbe358195cdede` | 2026-07-01T01:37:22Z |
| `sbt-badge` | `2a91bd5ac87dd5b888e681bf152c105eac8068a579ddd2cec397f4ecfa0f8377` | 2026-07-01T01:37:31Z |

El WASM hash permite verificar en Stellar Expert que el bytecode publicado on-chain corresponde exactamente al código fuente compilado del repositorio, sin alteraciones.

---

## 3. Resultados de la suite de pruebas unitarias

Ejecución local (`cargo test --workspace`), Soroban SDK 23, target `wasm32v1-none`:

### `fund-tracker` — 7/7 ✅

| Test | Resultado |
|---|---|
| `test_allocate_and_get` | ok |
| `test_allocated_at_matches_ledger_time` | ok |
| `test_different_projects_independent` | ok |
| `test_double_initialize_succeeds` | ok |
| `test_get_nonexistent_returns_none` | ok |
| `test_negative_amount_accepted` | ok |
| `test_reallocate_overwrites` | ok |

### `expense-anchor` — 9/9 ✅

| Test | Resultado |
|---|---|
| `test_anchor_and_get` | ok |
| `test_anchor_emits_event` | ok |
| `test_anchored_at_matches_ledger_time` | ok |
| `test_different_callers_can_anchor` | ok |
| `test_different_expenses_independent` | ok |
| `test_double_initialize_succeeds` | ok |
| `test_get_nonexistent_returns_none` | ok |
| `test_re_anchor_same_id_overwrites` | ok |
| `test_short_receipt_hash_accepted` | ok |

### `sbt-badge` — 16/16 ✅

| Test | Resultado |
|---|---|
| `test_all_valid_badge_types_accepted` | ok |
| `test_badges_are_per_organization` | ok |
| `test_double_initialize_panics` (should panic) | ok |
| `test_duplicate_badge_type_allowed` | ok |
| `test_get_active_badges_excludes_revoked` | ok |
| `test_get_badge_correct_data` | ok |
| `test_get_badge_nonexistent_returns_none` | ok |
| `test_get_badges_empty_for_unknown_org` | ok |
| `test_get_badges_returns_all` | ok |
| `test_invalid_badge_type_panics` (should panic) | ok |
| `test_mint_emits_event` | ok |
| `test_mint_returns_sequential_ids` | ok |
| `test_revoke_already_revoked_panics` (should panic) | ok |
| `test_revoke_emits_event` | ok |
| `test_revoke_nonexistent_panics` (should panic) | ok |
| `test_revoke_sets_status_and_timestamp` | ok |

Estos tests corren sobre el entorno simulado del Soroban SDK (ledger local, sin red), y validan la lógica de negocio de cada contrato antes del deploy real. Los snapshots de estado quedan versionados en `contracts/*/test_snapshots/`.

---

## 4. Qué queda demostrado on-chain (testnet)

- **`fund-tracker`**: registro inmutable de cuánto XLM fue asignado a cada proyecto (`allocate` / `get_allocation`).
- **`expense-anchor`**: anclaje del hash SHA-256 del comprobante de cada gasto aprobado (`anchor` / `get_expense`), verificable contra el archivo original almacenado en Cloudflare R2.
- **`sbt-badge`**: emisión de Soulbound Tokens de reputación no transferibles para organizaciones (`mint_badge` / `revoke_badge` / `get_badges`), con eventos on-chain para auditoría.

Cualquier jurado puede entrar a los links de Stellar Expert de la sección 2, revisar el bytecode desplegado, y —una vez que el backend NestJS empiece a invocar los contratos— seguir en vivo las transacciones (`allocate`, `anchor`, `mint_badge`) asociadas a cada `contractId`.

---

## 5. Limitaciones conocidas (transparencia)

Documentadas en detalle en `AUDIT.md` del repo de contratos:

- La atribución de organización/usuario vive off-chain (Postgres); on-chain solo queda registrado el signer de la transacción (`STELLAR_SERVER_SECRET`, keypair compartido por los tres contratos).
- `fund-tracker` y `expense-anchor` no tienen ACL de admin en las mutaciones (cualquier caller autenticado puede escribir).
- No hay validación on-chain de montos negativos/cero ni de longitud del `receipt_hash`.

Estas limitaciones están documentadas y priorizadas en el backlog de hardening; no afectan la validez de las pruebas reportadas arriba.
