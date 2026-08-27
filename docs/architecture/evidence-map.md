# Evidence Map

This document is the factual, diffable record of which classes, database tables and tests implement
each of the four architectural seams described in the narrative chapters, plus the cross-cutting
concerns (tenancy, audit, OSGi/plugin loading) and the two posture items covered in
[posture-and-known-gaps.md](./posture-and-known-gaps.md).

Every entry below was checked against the pinned repository commit `cb60779c171391be558cd7aebb1eafea60ad2b82`.
An entry marked **unconfirmed** means the cited class, table or test could not be located at that commit;
no entry in this document asserts existence without having been checked.

The narrative chapters (`seam-1-workflow.md`, `seam-2-invoicing.md`, `seam-3-catalog.md`,
`seam-4-control-plane.md`) and the posture note link back to the specific section below instead of
restating these paths inline.

## Seam 1 — Event-driven workflow (subscription → invoice → payment → overdue)

Kill Bill's core lifecycle is wired through a persistent, DB-backed bus and notification queue rather
than direct in-process calls between the subscription, invoice, payment and overdue modules. Each module
publishes internal events and consumes them via listener classes; the payment module additionally runs a
background Janitor to reconcile transactions left in an indeterminate state.

| Role | Class / Table | Path |
|---|---|---|
| Invoice event listener | `InvoiceListener` | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceListener.java` |
| Payment bus event handler | `PaymentBusEventHandler` | `payment/src/main/java/org/killbill/billing/payment/bus/PaymentBusEventHandler.java` |
| Overdue event listener | `OverdueListener` | `overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java` |
| Payment reconciliation background task | `Janitor` | `payment/src/main/java/org/killbill/billing/payment/core/janitor/Janitor.java` |
| Janitor: reconciles PENDING/UNKNOWN payment transactions | `IncompletePaymentTransactionTask` | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java` |
| Janitor: reconciles PENDING/UNKNOWN payment attempts | `IncompletePaymentAttemptTask` | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java` |
| Notification key used by the Janitor's own re-check notifications | `JanitorNotificationKey` | `payment/src/main/java/org/killbill/billing/payment/core/janitor/JanitorNotificationKey.java` |
| Durable, at-least-once event bus table | `bus_events` / `bus_events_history` | `util/src/main/resources/org/killbill/billing/util/ddl.sql` (lines 210, 230) |
| Durable notification queue table (used for retries, Janitor re-checks, overdue re-evaluation, etc.) | `notifications` / `notifications_history` | `util/src/main/resources/org/killbill/billing/util/ddl.sql` (lines 164, 188) |

`IncompletePaymentTransactionTask` explicitly enumerates the statuses it reconciles:
`TRANSACTION_STATUSES_TO_CONSIDER = List.of(TransactionStatus.PENDING, TransactionStatus.UNKNOWN)`
(`payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java:63`).

Verifying test: `TestIntegration` (beatrix), which exercises the end-to-end
subscription/invoice/payment event chain —
`beatrix/src/test/java/org/killbill/billing/beatrix/integration/TestIntegration.java`.

The underlying bus/notification-queue engine (`PersistentBus` / `NotificationQueue` implementations) lives
in the separate `killbill-platform` dependency, not in this repository; it is consumed here only through
its published Java API and the `bus_events`/`notifications` tables above, which are unconfirmed as being
"in this repository" beyond the schema — the schema itself is confirmed at the path cited.

## Seam 2 — Invoice generation & immutable retroactive repair

| Role | Class / Item type / Table | Path |
|---|---|---|
| Top-level invoice generation entry point | `DefaultInvoiceGenerator` | `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java` |
| Whole-account item-tree diffing structure | `AccountItemTree` | `invoice/src/main/java/org/killbill/billing/invoice/tree/AccountItemTree.java` |
| Per-subscription item-tree diffing structure | `SubscriptionItemTree` | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java` |
| Repair-adjustment invoice item emitted by the diff | `RepairAdjInvoiceItem` | `invoice/src/main/java/org/killbill/billing/invoice/model/RepairAdjInvoiceItem.java` |
| Factory that instantiates invoice item model objects (including repair/adjustment types) from persisted rows | `InvoiceItemFactory` | `invoice/src/main/java/org/killbill/billing/invoice/model/InvoiceItemFactory.java` |
| Item types produced/consumed by the repair mechanism | `REPAIR_ADJ`, `CBA`, `ITEM_ADJ` | enumerated in `InvoiceItemType`, referenced throughout `invoice/src/main/java/org/killbill/billing/invoice/model/` (e.g. `RepairAdjInvoiceItem.java`, `CreditBalanceAdjInvoiceItem.java`, `ItemAdjInvoiceItem.java`) |
| Invoice tables | `invoices`, `invoice_items`, `invoice_payments` | `invoice/src/main/resources/org/killbill/billing/invoice/ddl.sql` (lines 124, 50, 176 respectively) |

Verifying test: `TestIntegrationInvoiceWithRepairLogic` —
`beatrix/src/test/java/org/killbill/billing/beatrix/integration/TestIntegrationInvoiceWithRepairLogic.java`.

## Seam 3 — Versioned catalog, per-tenant overrides & plan alignment

| Role | Class / Table | Path |
|---|---|---|
| In-memory representation of one catalog XML version | `StandaloneCatalog` | `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java` |
| Ordered collection of catalog versions exposed to the rest of the system | `DefaultVersionedCatalog` | `catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java` |
| Applies per-tenant DB overrides on top of the versioned XML catalog | `CatalogUpdater` | `catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java` |
| Parses/loads catalog XML from disk or the catalog bundle | `LoadCatalog` | `catalog/src/main/java/org/killbill/billing/catalog/LoadCatalog.java` |
| Computes phase timeline / billing-cycle-day realignment on plan/phase change | `PlanAligner` | `subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java` |
| Shared alignment computation used by `PlanAligner` | `BaseAligner` | `subscription/src/main/java/org/killbill/billing/subscription/alignment/BaseAligner.java` |
| Add-on compatibility/inclusion rules | `AddonUtils` | `subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java` |
| Per-tenant catalog override tables | `catalog_override_plan_definition`, `catalog_override_phase_definition`, `catalog_override_plan_phase`, `catalog_override_usage_definition`, `catalog_override_tier_definition`, `catalog_override_block_definition`, `catalog_override_phase_usage`, `catalog_override_usage_tier`, `catalog_override_tier_block` | `catalog/src/main/resources/org/killbill/billing/catalog/ddl.sql` |
| Generic per-tenant key/value store (used for catalog and other per-tenant config) | `tenant_kvs` | `tenant/src/main/resources/org/killbill/billing/tenant/ddl.sql` (line 22) |
| Cluster-wide cache-invalidation broadcast table | `tenant_broadcasts` | `tenant/src/main/resources/org/killbill/billing/tenant/ddl.sql` (line 39) |

Verifying test: `TestIntegrationWithCatalogUpdate` —
`beatrix/src/test/java/org/killbill/billing/beatrix/integration/TestIntegrationWithCatalogUpdate.java`.

## Seam 4 — BlockingState / control-tag control plane

| Role | Class / Table | Path |
|---|---|---|
| Central authority answering "is this account/bundle/subscription blocked" | `DefaultBlockingChecker` | `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java` |
| Computes billing-exclusion periods from BlockingState history | `BlockingCalculator` | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java` |
| Shared base for the payment automaton processors that check control tags before charging | `ProcessorBase` | `payment/src/main/java/org/killbill/billing/payment/core/ProcessorBase.java` |
| Control-plugin implementation that gates auto-invoicing/auto-pay via the AUTO_PAY_OFF control tag | `InvoicePaymentControlPluginApi` | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java` |
| Automaton runner that invokes configured control plugins (e.g. `InvoicePaymentControlPluginApi`) before auto-charging, and can trigger the retry state machine on failure | `PluginControlPaymentAutomatonRunner` | `payment/src/main/java/org/killbill/billing/payment/core/sm/PluginControlPaymentAutomatonRunner.java` |
| Overdue state persistence table | `blocking_states` | `entitlement/src/main/resources/org/killbill/billing/entitlement/ddl.sql` (line 4) |
| Auto-pay retry state machine definition | `PaymentStates.xml` | `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml` |
| Payment retry state machine definition | `RetryStates.xml` | `payment/src/main/resources/org/killbill/billing/payment/retry/RetryStates.xml` |

**Duplicated AUTO_PAY_OFF check:** the control tag `AUTO_PAY_OFF` is checked independently in two places
rather than through a single shared gate:
- `ProcessorBase.isAccountAutoPayOff(...)` / `ProcessorBase.setAccountAutoPayOff(...)` —
  `payment/src/main/java/org/killbill/billing/payment/core/ProcessorBase.java:87-99`.
- `InvoicePaymentControlPluginApi.insert_AUTO_PAY_OFF_ifRequired(...)` (line 730) and
  `InvoicePaymentControlPluginApi.process_AUTO_PAY_OFF_removal(...)` (line 312) —
  `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java`.

Verifying test: `TestIntegrationWithAutoPayOff` —
`beatrix/src/test/java/org/killbill/billing/beatrix/integration/TestIntegrationWithAutoPayOff.java`.

## Cross-cutting: tenancy, audit, RBAC, OSGi/plugin loading

| Role | Class / Table | Path |
|---|---|---|
| Servlet filter that resolves the tenant for every incoming request | `TenantFilter` | `profiles/killbill/src/main/java/org/killbill/billing/server/security/TenantFilter.java` |
| Shiro realm backing tenant API-key/secret authentication against the DB | `KillbillJdbcTenantRealm` | `profiles/killbill/src/main/java/org/killbill/billing/server/security/KillbillJdbcTenantRealm.java` |
| Central append-only audit trail table | `audit_log` | `util/src/main/resources/org/killbill/billing/util/ddl.sql` (line 140) |
| Per-entity immutable history tables (pattern: `<entity>_history`) | e.g. `custom_field_history`, `tag_definition_history`, `tag_history`, `notifications_history`, `bus_events_history` | `util/src/main/resources/org/killbill/billing/util/ddl.sql` (lines 28, 71, 117, 188, 230) |
| Provider-plugin registries (the OSGi-facing extension point pattern used to register pluggable implementations behind `OSGIServiceRegistration`) | `DefaultCatalogProviderPluginRegistry`, `DefaultInvoiceProviderPluginRegistry`, `DefaultInvoiceFormatterFactoryProviderPluginRegistry`, `DefaultCurrencyProviderPluginRegistry`, `DefaultUsageProviderPluginRegistry`, `DefaultPaymentProviderPluginRegistry`, `DefaultPaymentControlProviderPluginRegistry`, `DefaultEntitlementProviderPluginRegistry` | `catalog/src/main/java/org/killbill/billing/catalog/provider/DefaultCatalogProviderPluginRegistry.java`, `invoice/src/main/java/org/killbill/billing/invoice/provider/DefaultInvoiceProviderPluginRegistry.java`, `invoice/src/main/java/org/killbill/billing/invoice/provider/DefaultInvoiceFormatterFactoryProviderPluginRegistry.java`, `currency/src/main/java/org/killbill/billing/currency/DefaultCurrencyProviderPluginRegistry.java`, `usage/src/main/java/org/killbill/billing/usage/glue/DefaultUsageProviderPluginRegistry.java`, `payment/src/main/java/org/killbill/billing/payment/provider/DefaultPaymentProviderPluginRegistry.java`, `payment/src/main/java/org/killbill/billing/payment/provider/DefaultPaymentControlProviderPluginRegistry.java`, `entitlement/src/main/java/org/killbill/billing/entitlement/provider/DefaultEntitlementProviderPluginRegistry.java` |

### RBAC mechanism (built but not applied — see [posture-and-known-gaps.md](./posture-and-known-gaps.md))

| Role | Class | Path |
|---|---|---|
| AOP interceptor handler that enforces `@RequiresPermissions` when RBAC is enabled | `PermissionAnnotationHandler` | `util/src/main/java/org/killbill/billing/util/security/PermissionAnnotationHandler.java` |
| Guice module wiring the Shiro AOP interceptor that dispatches to `PermissionAnnotationHandler` | `KillBillShiroAopModule` | `util/src/main/java/org/killbill/billing/util/glue/KillBillShiroAopModule.java` |

The `@RequiresPermissions` annotation itself is referenced in exactly two files in this repository: its
handler (`PermissionAnnotationHandler.java`, part of the mechanism) and its sole usage on any method,
which is a test fixture — `TestPermissionAnnotationMethodInterceptor.java` —
`util/src/test/java/org/killbill/billing/util/security/TestPermissionAnnotationMethodInterceptor.java`.
No JAX-RS resource class in `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/` carries
`@RequiresPermissions`.

### AccountResource payment-endpoint duplication (see [posture-and-known-gaps.md](./posture-and-known-gaps.md))

`AccountResource.getInvoicePayments` and `AccountResource.getPaymentsForAccount` are flagged in-code as
near-duplicates by the author:

```
// STEPH should refactor code since very similar to @Path("/{accountId:" + UUID_PATTERN + "}/" + PAYMENTS)
```

`jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:784`, immediately preceding
the `getInvoicePayments` method (line 793); the sibling `getPaymentsForAccount` method is at line 1049.

## Completeness check

Every class, table and test cited across the five sections above (Seam 1, Seam 2, Seam 3, Seam 4, and
Cross-cutting) was located at the pinned commit `cb60779c171391be558cd7aebb1eafea60ad2b82` by path, and no
entry in this document is left unlabeled: none required the **unconfirmed** label because every cited
path resolved. If a future update to this map cites something that cannot be located in the pinned repo,
it must be marked **unconfirmed** rather than asserted, per instruction I-0002.
