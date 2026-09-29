# FozPay Clone — PRD

## Original problem statement
Clone/expand https://github.com/migomariia-netizen/mmmm (FastAPI + React + MongoDB crypto payment gateway). Add:
1. Merchant-API withdrawal (payout) endpoint that REALLY auto-converts from USDT (or other configured source) when the merchant lacks the requested currency (e.g. BNB) but holds enough of a source currency — gated by a cabinet toggle.
2. New network **opBNB** (network_id=9, chainId 204, native BNB) with USDT + BNB. Since 1inch does not support opBNB, auto-convert swaps for opBNB route on BSC.
3. Admin-configurable auto-convert source currencies ("directions").

Provider keys: ALCHEMY_KEY, TRONGRID_KEY, ONEINCH_KEY (in backend/.env).

## Architecture
- Backend: FastAPI (`server.py`, `core.py`, `catalog.py`, `admin_router.py`, `recovery.py`, `oneinch.py`, `hd_wallet.py`, `wallet_crypto.py`, `aml.py`, `binance_prices.py`).
- HD wallet (BIP-44), per-user deposit addresses; hot wallet at EVM index 0. Real on-chain deposit detection (Alchemy/TronGrid/mempool) + real 1inch swaps + EVM payouts.
- Frontend: React (pages: Dashboard, Wallet, Requests, Settings, ApiDocs, Checkout, Recovery, Contacts, Login).
- Auth: JWT (email/password) + Emergent Google session. Admin seeded from ADMIN_EMAIL/ADMIN_PASSWORD.

## Implemented (2026-09-29)
- **opBNB network** added end-to-end: `catalog.NETWORKS[9]`, USDT+BNB currency networks, explorer (opbnb.bscscan.com), `recovery.EVM['opbnb']` + USDT token (0x9e5A...96f3, 18 dec), `hd_wallet` EVM set, admin treasury label, EVM deposit/withdraw/detection.
- **Merchant API payout**: `POST /api/v1/private/create-output` (X-Auth-Token + X-Auth-Sign). Documented in ApiDocs.
- **Auto-convert on withdrawal** (shared `_prepare_withdraw`): if insufficient target currency, debits a configured source currency (priority list) and records `convert_from/convert_amount/convert_chain`; payout worker runs the REAL 1inch swap then sends. `_swap_chain_for` routes opBNB→BSC.
- **Gating**: merchant field `auto_convert` (cabinet toggle, default on) AND platform `auto_convert_enabled`.
- **Admin config**: `GET/PUT /api/admin/convert-config` (auto_convert_enabled + auto_convert_sources); Settings → Платформа UI card.
- backend/.env: added ALCHEMY_KEY, TRONGRID_KEY, ONEINCH_KEY, MNEMONIC_ENC_KEY, ADMIN_EMAIL, ADMIN_PASSWORD, JWT_SECRET.

## Testing
- iteration_2.json: 16/16 backend pass; frontend data-testids render. No regressions.
- NOTE: hot wallet has no real funds, so worker's on-chain swap/payout stays Pending (expected). Ledger/API logic validated.

## Backlog / next
- P1: fetch live 1inch quote at prepare-time (vs PRICES_USD snapshot + 1.02 buffer) to reduce over/under-debit.
- P2: validate convert-config sources have an EVM/token representation on a supported swap chain.
- P2: opBNB direct DEX (PancakeSwap) swap instead of BSC routing, if true opBNB on-chain swap is desired.
- P2: webhook delivery journal / retry counter in cabinet.
