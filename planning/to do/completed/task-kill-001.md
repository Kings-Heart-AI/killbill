# KILL-001 — Business rules & unique-IP inventory

## Source
Requested by the user as a follow-up to the `/graphify` knowledge-graph build for this repo.

## Summary
Build an inventory of Kill Bill's business rules and unique IP — domain/billing policy logic that
differentiates it from a generic CRUD billing system — using the graphify output as an orientation
map, then verifying each rule against actual source code.

## Assessment
The graphify knowledge graph already exists at `graphify-out/` (`GRAPH_REPORT.md`, `graph.json`,
`graph.html`) — 25,588 nodes, 712 communities, already labeled by module (e.g. "Payment State
Machine", "Catalog Rules & Policies", "Payment Admin & Janitor"). No business-rules inventory exists
yet.

**Location:** `graphify-out/GRAPH_REPORT.md` — community list, God Nodes, and Hyperedges sections;
underlying source across `account/`, `catalog/`, `subscription/`, `entitlement/`, `invoice/`,
`payment/`, `overdue/`, `usage/`, `tenant/`, `currency/`.

## Plan

1. Use GRAPH_REPORT.md's community list and God Nodes/Hyperedges sections to identify which
   modules/communities are likely to contain business logic (favor Catalog, Subscription, Invoice,
   Payment, Overdue, Entitlement, Usage over infra/test/vendored-JS communities).
2. For each candidate area, open and read the actual source files graphify points to (not just the
   report text) to confirm the rule exists, understand its exact logic, and identify edge cases.
   Do not assign a confidence score based on the graph labels alone — ground it in source.
3. Work through modules in this order, documenting rules as you go: account, catalog, subscription,
   entitlement, invoice, payment, overdue, usage, tenant, currency/tax. Skip a module if it has no
   qualifying business logic.
4. Number rules sequentially in the order documented (zero-padded 4 digits, starting at 0001) —
   this doubles as the module grouping order.
5. Write the full inventory to `planning/research/business-rules-inventory.md` using the required
   convention (ID, Description, Applies to, Confidence score).

## Acceptance Criteria
All criteria verified 2026-08-27 before commit.
- [x] `planning/research/business-rules-inventory.md` exists and contains one entry per identified
      business rule using the exact convention: `- [Kill-BR-<nnnn>] - Business Rule Name` followed
      by `Description:`, `Applies to:`, and `Confidence score:` lines.
- [x] Rule IDs are sequential, zero-padded 4-digit numbers starting at `0001`, grouped in module
      order (account, catalog, subscription, entitlement, invoice, payment, overdue, usage, tenant,
      currency/tax).
- [x] Every rule's description and confidence score is grounded in an actual source file read
      during this task, not solely inferred from graph labels.
- [x] Purely technical/infrastructure logic (DI wiring, REST plumbing, generic caching) is excluded
      unless it encodes a business policy.
- [x] A summary is reported back: total rule count, counts by module, and which entries were scored
      Low confidence.
