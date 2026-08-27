# Seam 4 — BlockingState / Control-Tag Control Plane

See [evidence-map.md](./evidence-map.md#seam-4--blockingstate--control-tag-control-plane) for the full
list of classes and tables and the verifying test backing every claim in this chapter.

## What happens

Suspending or resuming service — for an account, a bundle, or a subscription — is not represented as a
boolean flag somewhere on the entity. It is expressed as **timestamped `BlockingState` rows**, persisted
in the `blocking_states` table, plus **control tags** (like `AUTO_PAY_OFF`) attached to the account.
`DefaultBlockingChecker` is the central authority that answers "is this entity currently blocked" by
reading that history, and `BlockingCalculator` turns a timeline of `BlockingState` rows into concrete
billing-exclusion periods.

This single mechanism does three distinct jobs through one control plane rather than three separate
ones:

1. **Gates entitlement operations** — `DefaultBlockingChecker` is consulted before entitlement-affecting
   operations to decide whether they're currently permitted for a blocked entity.
2. **Excludes periods from billing** — `BlockingCalculator` converts blocked intervals into billing
   exclusions so invoicing does not charge for time a service was suspended.
3. **Switches off auto-pay** — the `AUTO_PAY_OFF` control tag is checked by the payment side
   (`ProcessorBase`, and independently by `InvoicePaymentControlPluginApi` — see the duplication note in
   the evidence map) so that an account under this control tag is not auto-charged, and
   `PluginControlPaymentAutomatonRunner` is what invokes that control-plugin check before auto-charging,
   able to drive the retry state machine (`RetryStates.xml`) on failure.

## Why this design / what it costs

Expressing suspension/resumption as timestamped `BlockingState` rows and control tags — rather than a
mutable boolean flag on the account/bundle/subscription — is what lets `BlockingCalculator` reconstruct
exactly which time intervals were blocked after the fact, which a flag (which only tells you the
*current* state) could not do; that reconstruction is what feeds billing exclusion. Driving entitlement
gating, billing exclusion, and auto-pay suppression off the same `BlockingState`/control-tag data,
instead of three separate mechanisms, means a single write (a new `BlockingState` row, or adding the
`AUTO_PAY_OFF` tag) is consistently visible to all three consumers. The cost of that consistency is
duplication at the *consumption* edge: the evidence map records that the `AUTO_PAY_OFF` check itself is
implemented independently in `ProcessorBase` and in `InvoicePaymentControlPluginApi` rather than through
one shared gate, so the same control tag is currently interpreted by two separate code paths that must
be kept in agreement by hand.
