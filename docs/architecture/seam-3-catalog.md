# Seam 3 — Versioned Catalog, Per-Tenant Overrides & Plan Alignment

See [evidence-map.md](./evidence-map.md#seam-3--versioned-catalog-per-tenant-overrides--plan-alignment)
for the full list of classes and tables and the verifying test backing every claim in this chapter.

## What happens

Pricing and plan structure are declared as **effective-dated XML catalogs**: each catalog version
(`StandaloneCatalog`) has an effective date, and a tenant's full pricing history is the ordered sequence
of these versions (`DefaultVersionedCatalog`, loaded via `LoadCatalog`). On top of the XML-declared
prices, a tenant can apply its own **per-tenant DB price overrides** — `CatalogUpdater` applies these
overrides onto the versioned XML catalog rather than requiring a new XML upload for every tenant-specific
price change; the override data itself lives in the `catalog_override_*` tables (see the evidence map).

Because catalog state is cached for performance, any catalog change — a new XML version, or a per-tenant
override — must invalidate that cache everywhere the tenant's catalog might be held in memory, not just
on the node that made the change. Kill Bill does this by writing to the `tenant_broadcasts` table:
**catalog changes require cluster-wide cache invalidation via `tenant_broadcasts`**, rather than relying
on in-process cache expiry alone.

When a subscription changes plan or phase, adds an add-on, or is transferred, the billing-cycle-day and
phase timeline have to be recomputed. `PlanAligner` (using the shared computation in `BaseAligner`)
performs this recomputation according to **alignment rules declared in the catalog itself**: a plan can
declare its alignment as `START_OF_SUBSCRIPTION` (the new plan's phases align to when the subscription
itself started) or `START_OF_BUNDLE` (phases align to when the containing bundle started). Add-on
compatibility and inclusion — which add-ons are permitted with which base plans — is governed separately
by `AddonUtils`.

## Why this design / what it costs

Declaring pricing as effective-dated XML plus per-tenant DB overrides, instead of pricing living only in
one place, lets a vendor version pricing globally while still letting individual tenants negotiate
custom prices without forking the catalog XML — `CatalogUpdater` is the evidence that overrides are
layered onto the versioned catalog rather than replacing it. That layering is also exactly what makes
cluster-wide invalidation via `tenant_broadcasts` necessary: because the effective catalog for a tenant
is a computed combination of "which XML version is active now" plus "which DB overrides apply," any
node's in-memory catalog cache can go stale the moment either input changes anywhere in the cluster, so
Kill Bill has to broadcast the invalidation rather than rely on each node noticing on its own. Declaring
`START_OF_SUBSCRIPTION` vs `START_OF_BUNDLE` alignment in the catalog itself (rather than hardcoding
alignment behavior in `PlanAligner`) is what lets a plan's realignment behavior on change/add-on/transfer
be a catalog-authoring decision instead of a code change — the cost is that `PlanAligner`/`BaseAligner`
must correctly interpret whichever rule a given catalog version declares, for every kind of subscription
transition.
