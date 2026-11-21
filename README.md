# Karkain Inventory Management

A local-first, CLI/TUI inventory ledger for SKUs, warehouses, stock counts, valuation and reorder.

## Status

| Item | Status |
|---|---|
| Research | COMPLETE |
| Product design | COMPLETE |
| Repository creation | COMPLETE |
| Implementation | NOT STARTED |
| Compile/execute verification of the application | NOT STARTED (toolchain itself VERIFIED on Karkain v1.2.0) |

## What it does

Karkain Inventory Management tracks stock as an append-only ledger of movements against SKUs in warehouses and bins. It supports goods receipts, issues, transfers, returns and cycle/stock counts, with FEFO/FIFO allocation, lot/serial tracking, valuation summaries and low-stock reorder alerts. Reports are precomputed materialized summaries rather than JOIN-heavy queries, because the embedded `db2` SQL subset has no JOINs. A versioned, partial-JSON event export hands valuation data to the standalone Karkain Accounting application, and ERP consumes stock summaries through a defined contract. Interaction is through a command-line interface and text UI, including barcode-style scanner input routed through the TUI; there is no graphical desktop client.

## Who it serves

Small warehouses, retail and e-commerce operations, and light manufacturers that need deterministic local stock control without a full WMS. Typical users are storekeepers and operators who want offline barcode/TUI entry, scriptable stock adjustments, predictable flat licensing and a clean handoff to a separate accounting product.

## Planned features

- **SKUs & catalog** — products, variants, units of measure, barcodes (planned); lot/serial definitions (planned).
- **Warehouses & bins** — locations, bin codes, inter-warehouse transfers (planned).
- **Stock movements** — receipts, issues, returns, transfers, append-only movement log (planned); FEFO/FIFO allocation (planned).
- **Stock counts** — cycle counts, variance approval, adjustments (planned).
- **Valuation & reorder** — materialized valuation summaries (planned); low-stock thresholds and reorder alerts (planned).
- **Integration** — partial-JSON valuation/event export to Karkain Accounting (planned); stock summaries to Karkain ERP (planned).
- **Administration** — roles for counting vs. approving (planned); audit log (planned); backup/restore (planned).

## Explicit non-goals

- Manufacturing/MRP, BOMs and production scheduling; those are out of scope and may be integrated with separately.
- Posting journal entries, tax rules or financial reporting — that is Karkain Accounting's role; this app only supplies valuation data.
- ERP coordination workflows or CRM front-office records.
- Native GUI in v1.2.0, external SQL engines with JOIN reporting, carrier/marketplace cloud APIs requiring TLS, or any alternate implementation language.

## Karkain foundation

Exclusive foundation: Karkain Language v1.2.0. All application source will be Karkain. No other implementation language. No compiler modifications.

## Platform support status

Windows x86-64 toolchain VERIFIED (compile to native executable and execute). Linux cross-target output PRESENT but execution NOT VERIFIED. macOS NOT VERIFIED. These describe the toolchain, not this application, whose compilation is NOT STARTED.

## Build and run status

No application binary exists yet. The intended generic commands are `karkain build src/main.kark -o karkain-inventory` and `karkain run src/main.kark`. These become active only when implementation starts; until then they are informational and have not been executed for this application.

## Known blockers

- Native GUI toolkit BLOCKED — CLI/TUI-first design; GUI is a planned, blocked layer.
- OS atomic-write/file locking PARTIAL — `db_begin`/`db_commit`/`db_rollback` exist, but OS-level atomic write is unverified; movement-log durability needs validation before release.
- Full JSON DOM/schema PARTIAL — valuation/event export limited to simple field extraction.
- `system`/`env`/`process`/`time` PARTIAL shell wrappers — must be probed before scanner/date claims.
- Embedded `db2` SQL subset with no JOINs; valuation uses precomputed summaries.
- TLS/WebSocket BLOCKED — no carrier, marketplace or cloud barcode-service integration.

## Repository

GitHub: https://github.com/ajit-ai/Karkain-Inventory-Management — issues: https://github.com/ajit-ai/Karkain-Inventory-Management/issues — License: Apache-2.0.
