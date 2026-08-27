# Seam 2 — Invoice Generation & Immutable Retroactive Repair

See [evidence-map.md](./evidence-map.md#seam-2--invoice-generation--immutable-retroactive-repair)
for the full list of classes, item types, tables and the verifying test backing every claim in this
chapter.

## What happens

Kill Bill generates invoices by computing, for a subscription or a whole account, the set of billing
"items" that should exist over a time range (recurring charges, usage, fixed charges, credits,
adjustments) and comparing that against the items already committed to prior invoices.
`SubscriptionItemTree` performs this diff per subscription; `AccountItemTree` aggregates the result
across all of an account's subscriptions; `DefaultInvoiceGenerator` drives the overall generation using
these trees.

**A committed invoice is never mutated.** When something retroactive happens — a backdated
cancellation, a catalog price change applied retroactively, a correction to a past billing period — the
system does not go back and edit the invoice items that were already committed. Instead, the item-tree
diff (`SubscriptionItemTree`/`AccountItemTree`) recomputes what the timeline of items *should* look like
now, compares it against what was already committed, and for anything that no longer matches, emits
new, additional invoice items of type `REPAIR_ADJ` (repair adjustments) — along with the related `CBA`
(credit balance adjustment) and `ITEM_ADJ` (item adjustment) types where applicable — on a new invoice
that references the change. `RepairAdjInvoiceItem` and `InvoiceItemFactory` are the model-layer pieces
that represent and construct these repair items (see the evidence map).

## Why this design / what it costs

Reconciling retroactive change by diffing and emitting new `REPAIR_ADJ` items — rather than editing
`invoice_items` rows in place — is what **preserves an immutable billing history**: every invoice a
customer was ever shown, and every item ever committed to it, remains exactly as it was generated and
exactly as it was reported (e.g. to a customer, an accountant, or a downstream ledger integration).
Correction is expressed as new, additional, auditable entries rather than as a silent edit to the past.
The evidence map's Seam 2 section shows this is enforced at the model level: `RepairAdjInvoiceItem` is a
distinct item type from a plain invoice item, so a repair is always visibly a repair. The cost is
complexity concentrated in the diffing logic itself (`AccountItemTree`/`SubscriptionItemTree`), which
has to correctly reconstruct "what should the timeline look like now" and produce a minimal, correct
set of `REPAIR_ADJ`/`CBA`/`ITEM_ADJ` items rather than simply overwriting a value — the verifying test
`TestIntegrationInvoiceWithRepairLogic` (see the evidence map) exists specifically to exercise that
reconciliation path end to end.
