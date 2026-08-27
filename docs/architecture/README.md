# Kill Bill Architecture — Overview

This directory documents Kill Bill's architecture seam-by-seam: the event-driven billing workflow,
invoice generation & repair, the versioned catalog, and the BlockingState control plane, plus a
factual note on RBAC enforcement and a known code-duplication item.

Read in this order:

1. [evidence-map.md](./evidence-map.md) — the seam-to-code evidence map. Every class, table and test
   named anywhere in this directory is listed there with its confirmed repository path (or flagged
   `unconfirmed`). This map was built first, and every narrative claim below points at it rather than
   restating it.
2. [seam-1-workflow.md](./seam-1-workflow.md) — the subscription → invoice → payment → overdue
   event-driven workflow.
3. [seam-2-invoicing.md](./seam-2-invoicing.md) — proration and immutable retroactive invoice repair.
4. [seam-3-catalog.md](./seam-3-catalog.md) — versioned catalog XML, per-tenant overrides and plan
   alignment.
5. [seam-4-control-plane.md](./seam-4-control-plane.md) — the BlockingState / control-tag control
   plane.
6. [posture-and-known-gaps.md](./posture-and-known-gaps.md) — the RBAC enforcement posture and the
   `AccountResource` payment-endpoint duplication, as factual, evidence-bound notes.

## Deployment profiles

Kill Bill ships two Maven-assembled deployment profiles, under `profiles/`:

- **`profiles/killbill`** — the full killbill WAR: it wires every business module (account, catalog,
  entitlement, invoice, jaxrs, junction, overdue, payment, subscription, tenant, usage, util) plus the
  platform server into one deployable web application exposing the full JAX-RS API surface.
- **`profiles/killpay`** — a payment-only profile. It does **not** simply omit the other business
  modules: its POM declares direct dependencies on `killbill-account`, `killbill-catalog`,
  `killbill-entitlement`, `killbill-invoice`, `killbill-jaxrs`, `killbill-junction`, `killbill-overdue`,
  `killbill-payment`, `killbill-subscription`, `killbill-tenant`, `killbill-usage` and `killbill-util` —
  the same core business modules `profiles/killbill` wires — because the payment and control-plane code
  paths (Seam 4) reach into invoicing, entitlement and subscription state to decide whether to charge.
  `killpay` wires these as internal dependencies of a narrower deployment rather than treating them as
  optional; see the cross-cutting evidence entries in
  [evidence-map.md](./evidence-map.md#cross-cutting-tenancy-audit-rbac-osgiplugin-loading) for the
  provider-plugin registry pattern both profiles rely on to load pluggable implementations.

## Three standing constraints

Every seam chapter that follows must respect these three constraints; none of the four seams'
mechanisms make sense without them:

1. **Tenant isolation.** Every request is resolved to a tenant before any business logic runs
   (`TenantFilter`, `KillbillJdbcTenantRealm` — see the evidence map's cross-cutting section), and
   catalog, invoice, payment and entitlement state are all partitioned by tenant. Seam 3's per-tenant
   catalog overrides and cache-invalidation broadcasts exist specifically to keep that isolation correct
   under caching.
2. **Audit / history immutability.** Mutating rows never simply overwrite prior values: an append-only
   `audit_log` table plus per-entity `*_history` tables record every change (see the evidence map's
   cross-cutting section). Seam 2's invoice repair mechanism is this same principle applied to ordinary
   billing items on invoices specifically — such an item on a committed invoice is corrected by adding
   a new `REPAIR_ADJ`/`ITEM_ADJ` item rather than by editing it in place (see
   [seam-2-invoicing.md](./seam-2-invoicing.md) for the narrower `CBA`-bookkeeping exception).
3. **Plugin extensibility.** Payment gateways, invoice formatters, currency conversion and usage
   metering are all pluggable behind provider-plugin registries (the evidence map's cross-cutting
   section lists them), so seam-level mechanisms like Seam 4's control plane must express state through
   data (BlockingState rows, control tags) that plugins can also observe, rather than through
   in-process flags a plugin couldn't see.
