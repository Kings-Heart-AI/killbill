# Seam 1 — Event-Driven Workflow (Subscription → Invoice → Payment → Overdue)

See [evidence-map.md](./evidence-map.md#seam-1--event-driven-workflow-subscription--invoice--payment--overdue)
for the full list of classes, tables and the verifying test backing every claim in this chapter.

## What happens

When a subscription transitions (created, cancelled, changed, phase change, etc.), the subscription
module does not call the invoice module directly. It publishes an internal event onto a durable,
database-backed event bus. The invoice module's listener consumes that event asynchronously and
generates the invoice; invoice creation itself then publishes another event, which the payment module's
bus event handler consumes to attempt payment; and blocking/overdue state changes are likewise
delivered as events to the overdue module's listener. Every hop in this chain is a publish onto a
durable queue table and a later, independent consume — not a synchronous method call chain across
module boundaries.

This means the whole workflow is **durable and asynchronous**: it rides a DB-backed bus and
notification queue (the `bus_events`/`bus_events_history` and `notifications`/`notifications_history`
tables — see the evidence map), so an event survives a process restart between being published and
being consumed, and consumption happens on the consumer's own schedule rather than inline with the
event that produced it.

## Consequences

- **Polling/consumption latency.** Because consumption is decoupled from publication, there is an
  inherent (usually small, but nonzero) delay between a triggering event and its downstream effect —
  e.g. between a subscription change and the resulting invoice, or between an invoice and the payment
  attempt it triggers.
- **Table growth.** Every event and notification is a row in `bus_events`/`notifications` (and their
  `_history` counterparts once processed), so these tables grow with system activity and are an
  operational surface (indexing, retention) that a purely synchronous design would not have.
- **Janitor reconciliation of PENDING/UNKNOWN payments.** Because payment attempts are driven through
  this asynchronous chain and talk to external gateways, a transaction can be left in the `PENDING` or
  `UNKNOWN` `TransactionStatus` when the workflow doesn't get a definitive answer synchronously. The
  payment module's `Janitor` background task (`IncompletePaymentTransactionTask`,
  `IncompletePaymentAttemptTask`) exists specifically to re-check and reconcile exactly those two
  statuses later, rather than assuming the initial event-driven attempt is final.

## Why this design / what it costs

Driving the lifecycle through a durable bus/notification queue instead of direct calls means a
subscription-state change and its billing consequences don't have to happen inside one transaction or
one thread; the state change commits immediately, and invoicing/payment/overdue reactions follow as
independent, retriable steps. The `Janitor`'s dedicated reconciliation of `PENDING`/`UNKNOWN` payment
statuses (see the evidence map's Seam 1 section, `IncompletePaymentTransactionTask`) is itself evidence
of this trade-off: an asynchronous, at-least-once delivery model needs an explicit sweep to close out
transactions that never got a definitive synchronous answer from a gateway, a problem a purely
synchronous call chain would not create in the same form. The cost side — polling/consumption latency
and unbounded growth of `bus_events`/`notifications` — is the direct consequence of choosing durability
and decoupling over immediacy.
