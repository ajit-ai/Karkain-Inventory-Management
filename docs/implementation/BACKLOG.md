# Karkain Inventory Management — First Implementation Backlog

## First vertical slice

**Slice goal:** Prove a complete stock workflow: define a SKU and a warehouse/bin, receive two lots with expiry dates, issue stock using FEFO allocation, print on-hand and a valuation summary, and export a flat valuation event — real persisted movements, not empty forms.

**Karkain modules/files expected:** `src/main.kark` (dispatcher), `src/catalog.kark`, `src/warehouses.kark`, `src/movements.kark`, `src/allocation.kark`, `src/valuation.kark`, `src/store.kark` (using `db` with `db_begin`/`db_commit`/`db_rollback`, `io`, `json` PARTIAL, `crypto`, `collections`).

**Exact input:** `karkain-inventory sku create --sku SKU-1 --name Widget --uom each`, `karkain-inventory warehouse create --code WH1`, `karkain-inventory movement receive --id mv-1 --sku SKU-1 --wh WH1 --qty 10 --lot L-A --expiry 2026-12-31 --cost 5.00`, `... --id mv-2 --lot L-B --expiry 2027-06-30 --qty 10 --cost 6.00`, `karkain-inventory movement issue --id mv-3 --sku SKU-1 --wh WH1 --qty 12 --strategy FEFO`, `karkain-inventory report onhand --sku SKU-1`, then `karkain-inventory export valuation --contract valuation.v1`.

**Exact expected observable output:** `Created SKU-1`, `Created WH1`; `Received mv-1`, `Received mv-2`; `Issued mv-3 allocated=L-A:10,L-B:2` (earlier expiry first); on-hand prints `SKU-1|WH1|onhand=8|value=48.00` (2 units of L-B at 6.00); export prints `Exported evt-1 contract=valuation.v1 sku=SKU-1 value=48.00`. Issuing more than available prints `Rejected: insufficient stock` and writes no movement.

**Verification command pattern (Karkain v1.2.0):** With the Go front-end engine selected, build/run with the verified C-free native target —
```
karkain build src/main.kark --target native-x86_64-windows -o karkain-inventory
.\karkain-inventory sku create --sku SKU-1 --name Widget --uom each
.\karkain-inventory warehouse create --code WH1
.\karkain-inventory movement receive --id mv-1 --sku SKU-1 --wh WH1 --qty 10 --lot L-A --expiry 2026-12-31 --cost 5.00
.\karkain-inventory movement receive --id mv-2 --sku SKU-1 --wh WH1 --qty 10 --lot L-B --expiry 2027-06-30 --cost 6.00
.\karkain-inventory movement issue --id mv-3 --sku SKU-1 --wh WH1 --qty 12 --strategy FEFO
.\karkain-inventory report onhand --sku SKU-1
.\karkain-inventory export valuation --contract valuation.v1
```
Build must exit 0 and the executable must print the exact lines above. No test has been run yet; this is the planned verification pattern.

## Backlog items

| ID | Feature | Files/modules | Dependencies | Acceptance criteria | Tests | Complexity | Blocking Karkain capability | Definition of done |
|---|---|---|---|---|---|---|---|---|
| INV-001 | Command dispatcher & config profile | `main.kark`, `config_profile.kark` | None | All subcommands dispatch; no hard-coded paths | Unit, CLI smoke | S | `system`/`env` PARTIAL | Builds, help lists commands |
| INV-002 | SKU catalog & lot/serial definitions | `catalog.kark`, `store.kark` | INV-001 | Unique SKUs, units, barcode field, persisted | Unit, integration | M | `db` PRESENT/PARTIAL | CRUD survives restart |
| INV-003 | Warehouses, bins & transfers | `warehouses.kark` | INV-001 | Locations created, transfers move stock | Unit, integration | M | `db` PRESENT/PARTIAL | Transfer conserves quantity |
| INV-004 | Append-only movement log & on-hand | `movements.kark`, `store.kark` | INV-002, INV-003 | Receipts/issues logged; on-hand derived; negatives blocked | Integration, failure | L | `db` transactions; atomic write PARTIAL | Replay reproduces on-hand |
| INV-005 | FEFO/FIFO allocation & reservations | `allocation.kark` | INV-004 | Earlier expiry allocated first; deterministic | Unit, regression | L | None | Allocation matches fixtures |
| INV-006 | Cycle counts & approval-gated variance | `counts.kark`, `audit.kark` | INV-004 | Variance computed; override logged | Integration, security | M | `time` PARTIAL wrapper | Approved adjustment auditable |
| INV-007 | Materialized valuation summaries (no JOINs) | `valuation.kark` | INV-004, INV-005 | FIFO/AVCO summary matches movement math | Unit, regression | L | `db2` no JOINs | Valuation reproducible |
| INV-008 | Reorder thresholds & low-stock alerts | `reorder.kark` | INV-004 | Alert fires at/below threshold | Unit | M | None | Alerts deterministic |
| INV-009 | Partial-JSON valuation export & ERP contract | `integration.kark` | INV-007 | Flat export round-trips; versioned contract | Integration, compatibility | L | `json` PARTIAL; contracts undefined | Stub Accounting consumes export |
| INV-010 | Roles, audit trail & backup/restore | `auth.kark`, `audit.kark`, `backup.kark` | INV-001, INV-004 | Counter/approver separated; consistent pair backup | Security, failure | L | atomic write PARTIAL | Restore recomputes identical stock |

## Dependency order and gates

1. INV-001 → 2. INV-002 + INV-003 (parallel) → 3. INV-004 (movements) → 4. INV-005 + INV-006 + INV-007 + INV-008 (allocation/counts/valuation/reorder) → 5. INV-009 (export/integration) → 6. INV-010 (admin/durability). Gate between phases: the actual Karkain v1.2.0 compiled executable must be run and expected output verified before advancing; movement-log durability must be explicitly validated before any production-readiness claim. Blockers (GUI, JSON, atomic write, TLS/WebSocket) are recorded, never bypassed by changing Karkain.

## Status

All items: **NOT STARTED**. Implementation must not begin until all designs and repository initialization are complete. No tests have been run and no binary exists for this application.
