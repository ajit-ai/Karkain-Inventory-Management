# Karkain Inventory Management — Product Specification

**Document status:** Design COMPLETE. Research = COMPLETE. Repository creation = COMPLETE. Application implementation = NOT STARTED. Application compile/execute verification = NOT STARTED (toolchain path itself VERIFIED on Karkain v1.2.0).


## A. Product definition

**Product name:** Karkain Inventory Management.

**Purpose/problem:** Small operations need reliable stock control without full WMS cost and complexity. Cloud inventory tools are browser-bound and metered by orders/users/locations and couple stock data to their own accounting suites. A local, scriptable stock ledger with a clean handoff to a separate accounting product fills the gap.

**Vision:** "The inventory ledger you can script" — a deterministic, offline-first store of SKUs, warehouses and stock movements with explicit valuation and clean contracts to standalone Karkain Accounting and Karkain ERP.

**Users/segments:** Storekeepers and operators in small warehouses, retail and e-commerce sellers, light manufacturers (excluding MRP); maintainers needing offline barcode/TUI entry, reproducible adjustments and data sovereignty.

**Usage scenarios:** Receive goods against a purchase with a lot/serial; pick stock with FEFO allocation; run a cycle count and approve a variance; review low-stock alerts; export a valuation summary to Accounting.

**Positioning/value:** A CLI/TUI inventory service on embedded `db2` with append-only movements, cycle counts, FEFO/FIFO allocation and low-stock alerts. Value: deterministic offline tracking, scriptable commands, materialized valuation without JOINs, integration with — not duplication of — Accounting/ERP.

**Non-goals/boundaries:** Manufacturing/MRP, BOMs and scheduling are out of scope (integrate separately). No journal entries, tax rules or financial reporting — Karkain Accounting consumes this app's valuation. No ERP coordination or CRM records. No GUI, external SQL/JOIN reporting, TLS carrier/marketplace APIs or alternate language.

## B. Functional architecture

| Module | Responsibility |
|---|---|
| SKU catalog | Products, variants, units, barcodes, lot/serial definitions |
| Warehouses & bins | Locations, bin codes, inter-warehouse transfers |
| Stock movements | Append-only receipts, issues, returns, transfers with lot/serial refs |
| Allocation | FEFO/FIFO removal strategies, reservations |
| Stock counts | Cycle counts, variance calculation, approval-gated adjustments |
| Valuation | Materialized FIFO/AVCO/standard summaries from movements |
| Reorder | Low-stock thresholds, alerts, suggested quantities |
| Reporting & queries | Precomputed on-hand/valuation summaries (no JOINs) |
| Integration & admin | Partial-JSON valuation export, ERP contract, roles, audit, backup/restore |

**Core workflows:** Create SKU → receive goods → pick/issue with allocation → transfer bins/warehouses → run cycle count → approve variance → review alerts → export valuation.

**Admin workflows:** Define warehouses/bins, valuation method and reorder thresholds; define counter vs. approver roles; review audit; back up/restore store.

**Error/recovery:** Reject movements causing negative stock (unless configured); quarantine duplicate movement ids; reconcile after abrupt termination by replaying the movement log; require logged approver for variances beyond tolerance.

**Roles/permissions:** Administrator (config), Storekeeper (movements/counts), Approver (variances), Auditor (read-only), enforced at CLI/TUI layer; not a security certification.

**Configuration/customization:** Local profile selects valuation method, FEFO/FIFO, count tolerance and contract version; customization via config and Karkain scripts, no compiler changes.

**Import/export:** Deterministic flat format plus simple `json` field extraction; valuation/movement events exported to Accounting via a versioned contract; no full JSON DOM assumed.

**Integration:** Provide valuation summaries to Accounting and stock summaries to ERP via defined contracts over local CLI/file drop; no TLS, WebSocket, carrier or marketplace APIs.

## C. Competitor analysis

Competitors: **Odoo Inventory**, **Zoho Inventory**, **Fishbowl Inventory**. Evidence and sources are in `docs/competitor-analysis/COMPETITORS.md`; labels match that document.

**Positioning:** [VF] Odoo Inventory is "both an inventory application and a warehouse management system" with an advanced barcode app. [VF] Zoho serves retail, ecommerce, manufacturing and dropshipping with a free micro-business tier. [VF] Fishbowl targets barcode-driven manufacturers/warehouses syncing to QuickBooks/Xero ("over 6,000 businesses" — vendor claim). [RI] Zoho wins on price/channel breadth, Odoo on depth, Fishbowl on the accounting bridge.

| Capability | Karkain Inventory [P] | Odoo | Zoho | Fishbowl |
|---|---|---|---|---|
| CLI/TUI-first | Yes | No | No | No |
| Offline local store (db2) | Yes | On-prem option | No | Desktop option |
| Barcode entry | [P] TUI/scanner input | Advanced barcode app | Generate/scan | Barcode-driven |
| Lot/serial + FEFO | [P] subset | Full | Serial/batch | Lot traceability |
| Manufacturing/MRP | Out of scope (integrate) | Built-in | Limited | Core strength |
| Order caps published | [P] to publish | Suite-based | 50–15,000/mo | Quote-based |
| Accounting handoff | Standalone Accounting app | Native app | Zoho Books | QuickBooks/Xero |

**Platform/deployment:** [VF] Odoo offers web + mobile barcode app, SaaS/on-premise; Zoho web, iOS/Android and a Windows app (cloud); Fishbowl cloud (Drive) and desktop. [RI] No competitor is CLI/TUI-first; Karkain [P] is a local CLI/TUI service on `db2`.

**Licensing (verified only):** [VF] Zoho Inventory (India, INR, billed annually): Free ₹0, Standard ₹999, Premium ₹2,299, Plus ₹4,999, Enterprise ₹7,499 per org/month. [VF] Odoo Inventory is included in Odoo Standard/Custom plans (region-localized, 12-month introductory discount; see ERP research §1). [VF] Fishbowl: Essentials $229 USD/month billed annually (2 users, QuickBooks Online & Xero sync); other tiers quote-based. Karkain licensing: [P] flat Apache-2.0. No prices invented.

**Strengths/limitations:** [VF/R] Odoo: deepest WMS/manufacturing but part of a large suite. Zoho: aggressive price/order caps and channel integrations, advanced warehousing top-tier. Fishbowl: manufacturing and accounting bridge, quote-based upper tiers and GUI-bound.

**Market gaps:** [RI] Lightweight, scriptable stock control for small operations needing deterministic local tracking, offline barcode/TUI entry and a clean handoff to a separate accounting product — without a full WMS.

**Differentiation:** [P] CLI/TUI inventory service with append-only events, cycle counts, FEFO/FIFO allocation, materialized valuation and partial-JSON export to standalone Accounting; avoids JOIN-heavy reporting via precomputed summaries.

**Deliberately not copied:** Full WMS/MRP, cloud carrier/marketplace integrations requiring TLS, GUI barcode apps, per-order/user/location metering, built-in accounting ledgers.

## D. Technical architecture

**Application/boundaries:** A dispatcher fronts catalog, warehouses, movements, allocation, counts, valuation, reorder, reporting and integration/admin. The movement log is the single source of truth; on-hand and valuation are derived materialized summaries, never authoritative mutable counts. Single executable as one-shot CLI, TUI loop or minimal local service; Accounting/ERP exchange contract files. No non-Karkain in-process extensions.

**UI architecture:** The UI is **CLI/TUI/service-first because Karkain v1.2.0 has no native GUI toolkit (BLOCKED)**. A GUI is a planned layer contingent on ecosystem capability, not part of v1.2.0. Scanner input is plain text routed through the same commands; every TUI action has a CLI equivalent.

**Domain model:** SKU, Variant, Warehouse, Bin, LotSerial, Movement, Allocation, StockCount, CountLine, ValuationSummary, ReorderRule, ContractVersion, Role, AuditEntry. Corrections are new reversing movements, never edits.

**Storage strategy:** Embedded `db2` SQL subset for master data plus append-only movement log; no external SQL engine; valuation/counts computed sequentially and stored as materialized summaries (no JOINs).

**API/events:** Stable CLI subcommands plus minimal `http` building-block request/response; no routing/concurrent-server framework. Each movement emits a movement-sequence event for replay; handlers idempotent by movement id; duplicates quarantined; export events carry a contract version.

**Logging/config/extensions:** Structured text logs with run id, movement counters and an audit trail with hash chaining planned via `crypto` sha256 (designed, not implemented). Local profile selects valuation method, strategy, tolerance and contract via `config`/`io` once probed, no machine-specific paths. Valuation strategies are Karkain-native modules; any future FFI must be a justified Karkain-compatible native interface, never a replacement language; none planned.

## E. Data design

**Entities/ownership:** SKU, Variant, Warehouse, Bin, LotSerial, Movement, Allocation, StockCount, CountLine, ValuationSummary, ReorderRule, ContractVersion, AuditEntry, Role. Movements reference SKU, location and optional lot; summaries derive per SKU. Inventory owns stock data only.

**Validation/lifecycle:** Quantities, non-negative stock (unless configured), FEFO expiry ordering and count tolerance enforced in code. Movement: recorded→applied→reversed; StockCount: drafting→counting→approved/rejected; ReorderRule: active→retired.

**Persistence/transactions:** `db2` tables plus append-only movement log. `db_begin`/`db_commit`/`db_rollback` exist in v1.2.0, but **OS-level atomic write is unverified**. Because stock correctness depends on durable movements, atomicity must be validated before production readiness; checkpoints conservative, replay reconciliation mandatory. No unsupported durability claim.

**Backup/migration/IO/audit:** Stop service, copy store and log as pair, restore by replacing both and verify by recomputing on-hand; no online snapshot. Explicit version integers with adjacent-version Karkain migration routines; no pre-v1.2.0 schema. Deterministic flat format and simple `json` extraction. Every movement, count approval and config change appends an AuditEntry with actor, command and timestamp (once `time` is verified); hash chaining planned.

## F. Karkain implementation architecture

**Capabilities (status per register):** Compiler v1.2.0 VERIFIED; lint VERIFIED; C-free native Windows backend VERIFIED with Go engine (default `kcc` BLOCKED — environment); `db` PRESENT/PARTIAL (`db2` subset, transactions present); `json` PARTIAL; `io`/`crypto`/`encoding`/`collections`/`generics` PRESENT; `async`/`net`/`http` PRESENT (building blocks/cooperative); `filesystem` PARTIAL; `system`/`env`/`process`/`time` PARTIAL shell wrappers needing probes. No GUI module.

**Runtime/packaging:** Karkain v1.2.0 only, no compiler modification; Go front-end with C-free native target. Required ecosystem components: Accounting with a valuation contract and ERP for stock summaries, which must exist before integration work. One native executable in a directory package with default config and empty data directory; no machine-specific paths.

**Blockers:** (1) GUI toolkit BLOCKER — CLI/TUI only, GUI a blocked planned layer; (2) OS atomic write PARTIAL — movement durability must be validated, never claimed; (3) full JSON DOM PARTIAL — flat export formats; (4) `system`/`env`/`time` PARTIAL — probe before scanner/date claims; (5) TLS/WebSocket BLOCKER — no carrier/marketplace/cloud barcode services; (6) concurrent HTTP framework BLOCKED — minimal sequential service. Each recorded, never resolved by changing Karkain or using another language.

**Native/FFI:** None for MVP; any future FFI must be a justified Karkain-compatible native interface, never a replacement language.

## G. Non-functional requirements

Targets give workload and measurement method; no arbitrary numbers asserted. Practical machine ~4 GB RAM.

| Area | Target / method |
|---|---|
| Performance | Record 1,000 movements, time with `bench`, expect linear scaling |
| Memory/CPU | Profile batch replay with `prof`; steady state within ~4 GB budget |
| Scalability | 10× movements, record time ratio; document `db2` ceiling |
| Reliability | Inject duplicate ids and abrupt stop; replay yields identical on-hand/valuation |
| Security/privacy | Counter/approver separation, no secret logging, no telemetry; not a certification |
| Portability | Cross-build Linux; execution UNVERIFIED on this host |
| Maintainability/accessibility | Interfaces documented and unit-tested; text-only keyboard TUI, plain-text scanner input |
| Observability/recovery | Run id, movement counters, audit entries; kill mid-movement, no partial/duplicate movements |

## H. Compatibility and portability

Windows x86-64 toolchain VERIFIED; Linux cross-target output PRESENT but execution NOT VERIFIED; macOS NOT VERIFIED. All source targets v1.2.0 with Go front-end/C-free backend; default `kcc` unusable here (environment blocker). Native executable plus directory package. Install/uninstall by directory operations after export. Adjacent-version config/data/contract migration; export before upgrade. Backward compatibility not required before v1.2.0. Upgrade/rollback via stop, backup pair, replace, migrate, verify by recomputed valuation; restore on failure. Unsupported: GUI, TLS/WebSocket cloud services, external SQL, concurrent HTTP server, MRP/manufacturing, non-Windows execution verification.

## I. Testing strategy

All tests **PLANNED**; none have been run for this application.

- **Unit:** FEFO/FIFO allocation, valuation math, variance tolerance, duplicate detection — deterministic.
- **Integration:** `db2` movements + replay + valuation export — state identity after restart.
- **E2E:** Receive→pick→count→export with stub Accounting — expected valuation summary.
- **Platform-specific:** Windows native compile-and-run exit 0; other platforms cross-build only.
- **Security:** Counter/approver separation and no secret logging — unauthorized approval rejected.
- **Performance:** Movement batch with `bench` — measured, documented scaling.
- **Regression/compatibility:** Replay fixtures and migrate versions — identical on-hand/valuation, lossless.
- **Actual-compiled-executable:** Karkain v1.2.0 build, run native binary — expected stdout.
- **Failure/recovery:** Kill mid-movement — no partial/duplicate movements; audit intact.

## J. Implementation roadmap

**MVP:** SKU catalog, warehouses/bins, append-only movements, on-hand queries, CLI/TUI entry. **Core:** FEFO/FIFO allocation, lot/serial, cycle counts with approval, materialized valuation, reorder alerts, partial-JSON export. **Advanced:** Replay tooling, multiple valuation methods, hash-chained audit, Accounting/ERP contracts, minimal local service. **Production-readiness:** Explicit durability validation, observability, published limits, cross-target builds, full suite.

**Dependencies/gates:** Catalog before movements; movements before allocation/counts; counts before valuation; valuation contract before Accounting integration; durability validation before production readiness. Each phase exits only on passing tests and a verified actual compiled executable. No dates.

**Risks/mitigations:** Atomic write unverified — checkpoints and replay reconciliation; PARTIAL JSON — flat export; `time`/`system` PARTIAL — probe before scanner/date claims; Accounting contract undefined — define first; GUI BLOCKED — CLI/TUI only. No completion claims.
