# Business Rules — Cross-Document Match: old inventory vs. new extraction

Compares `planning/research/business-rules-inventory.md` (KILL-001/002, 166 rules, `Kill-BR-NNNN`) against `analysis/killbill/BUSINESS_RULES.md` (modernize-extract-rules workflow, 184 rules, `RULE-NNN`).

**Matching method:** a rule in one doc is considered matched to a rule in the other if they cite the same source file with an overlapping line range. This is a citation-based match, not a semantic one — two rules can cite overlapping lines while describing slightly different aspects of that code (and vice versa: the same business behavior can be cited at different line numbers if one doc points at a helper and the other at its caller, in which case they show up as unmatched here). Matches are many-to-many: one old rule can map to several new rules (the new extraction split it) or several old rules to one new rule.

## Summary

- Old inventory: 166 rules total — 99 matched, **67 unique to old doc**
- New extraction: 184 rules total — 110 matched, **74 unique to new doc**
- Matched pairs (old↔new citation overlaps): 137

The new extraction's higher rule count (184 vs 166) plus a large unique-to-new set is consistent with its process: loop-until-dry rounds explicitly hunting for gaps after each pass, versus the old inventory's single-pass module survey. The unique-to-old set is smaller and worth checking individually — see table below — since it may point to rules the new extraction's referees rejected, or genuine coverage gaps in the new pass.

## Rules unique to the old inventory (business-rules-inventory.md)

67 rules with no overlapping citation in the new extraction:

| ID | Name | Confidence | Citations |
|---|---|---|---|
| [Kill-BR-0003] | Immutable billing-cycle-day (BCD) once set, feature-flag gated | High | `account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:198-203`; `account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:260-274` |
| [Kill-BR-0004] | BCD can only be set once via updateBCD API (separate from general update) | High | `account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java:86-99` |
| [Kill-BR-0005] | Immutable account timezone once set | High | `account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:205-210` |
| [Kill-BR-0006] | Immutable account reference time (date-of-day only) | High | `account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:212-215` |
| [Kill-BR-0010] | External key cannot resolve to null | High | `account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:240-247` |
| [Kill-BR-0012] | Payment-method update is a no-op when unchanged | High | `account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:296-327` |
| [Kill-BR-0014] | BCD sentinel value and default-timezone-UTC on creation | High | `account/src/main/java/org/killbill/billing/account/api/DefaultMutableAccountData.java:29-30`; `account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:74-85` |
| [Kill-BR-0015] | isPaymentDelegatedToParent defaults to false | High | `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:145-145` |
| [Kill-BR-0016] | Cached BCD treats "unset" (0) as absent, forcing recompute | High | `account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java:156-167`; `account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java:101-108` |
| [Kill-BR-0017] | Fixed-offset timezone derived from account timezone + reference time (DST snapshot) | Medium | `util/src/main/java/org/killbill/billing/util/account/AccountDateTimeUtils.java:36-44`; `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:354-357`; `account/src/main/java/org/killbill/billing/account/api/DefaultImmutableAccountData.java:55-71` |
| [Kill-BR-0030] | EVERGREEN-duration-must-be-UNLIMITED constraint | High | `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java:290-310` |
| [Kill-BR-0038] | Reserved "DEFAULT" price-list name protection | High | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java:107-118`; `catalog/src/main/java/org/killbill/billing/catalog/PriceListDefault.java:36-45` |
| [Kill-BR-0040] | Unit-limit compliance cascades from usage to product level | High | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlanPhase.java:129-138`; `catalog/src/main/java/org/killbill/billing/catalog/DefaultProduct.java:156-172` |
| [Kill-BR-0042] | Add-on availability/inclusion governs listing | High | `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java:340-382`; `catalog/src/main/java/org/killbill/billing/catalog/DefaultProduct.java:99-103` |
| [Kill-BR-0045] | Deterministic overridden-plan naming/identity rule | High | `catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java:121-134` |
| [Kill-BR-0046] | Simple-plan-descriptor trial/recurring compatibility rule | High | `catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java:227-283` |
| [Kill-BR-0048] | Hardcoded default rule set for programmatically-built ("simple") catalogs | High | `catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java:332-365` |
| [Kill-BR-0049] | Cancellation effective-date policy resolution (IMMEDIATE / START_OF_TERM / END_OF_TERM) | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:811-895` |
| [Kill-BR-0052] | Uncancel only valid on a future-cancelled subscription | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:318-356` |
| [Kill-BR-0053] | Undo-change-plan only valid with a pending plan change | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:663-701`; `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:796-809` |
| [Kill-BR-0057] | One base/standalone plan per bundle; add-on requires an active base | High | `subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:223-258` |
| [Kill-BR-0060] | FIXEDTERM phase auto-expiry scheduling | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:554-563`; `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:615-622` |
| [Kill-BR-0062] | Catalog-version upgrade alignment to next BCD boundary | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:682-738`; `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:740-767` |
| [Kill-BR-0063] | Add-on cannot be created/changed past its own start-date guard relative to base | Medium | `subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:237-245`; `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:757-760` |
| [Kill-BR-0066] | Per-service "latest wins", not global latest | High | `entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:448-471` |
| [Kill-BR-0076] | Whole-billing-period counting for bulk period advancement | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:34-51` |
| [Kill-BR-0079] | Invoice items are append-only; retroactive changes produce REPAIR_ADJ items instead of editing history | High | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:166-199`; `invoice/src/main/java/org/killbill/billing/invoice/tree/ItemsNodeInterval.java:161-262` |
| [Kill-BR-0082] | Fully-repaired item closure exclusion | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java:89-99` |
| [Kill-BR-0083] | No double-billing / no double-repair chronological invariant | High | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:220-247` |
| [Kill-BR-0084] | Account-level AUTO_INVOICING_OFF suppresses scheduled invoicing but not explicit API calls | High | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:387-389` |
| [Kill-BR-0085] | Per-subscription auto-invoice-off exclusion | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:102-116` |
| [Kill-BR-0086] | Removing AUTO_INVOICING_OFF triggers catch-up invoicing | High | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceTagHandler.java:72-84` |
| [Kill-BR-0088] | AUTO_INVOICING_REUSE_DRAFT reuses an existing DRAFT invoice | Medium | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:606-611` |
| [Kill-BR-0090] | Dry-run invoice generation never commits to disk, three distinct modes | High | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:403-439` |
| [Kill-BR-0093] | Adjustable item-type restriction | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1362-1370` |
| [Kill-BR-0096] | Credit items are stored as negated amounts | High | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:633-647` |
| [Kill-BR-0105] | Invoice numbering is plugin/custom-field driven, not a native sequence | High | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:785-803`; `invoice/src/main/java/org/killbill/billing/invoice/api/InvoiceApiHelper.java:123-129`; `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:337-371` |
| [Kill-BR-0107] | Charged-through-date (CTD) updated only on invoice commit | High | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:688-708` |
| [Kill-BR-0110] | Control-plugin chain with abort-on-first-abort | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:94-155` |
| [Kill-BR-0111] | Earliest-retry-date-wins across failure-handling control plugins | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:263-294` |
| [Kill-BR-0112] | Invoice-driven purchase gated on COMMITTED invoice status | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:341-351` |
| [Kill-BR-0113] | Child-account payments delegated to parent are blocked | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:354-360` |
| [Kill-BR-0115] | Zero/negative computed amount aborts unless "allow empty invoice" is configured | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:362-374` |
| [Kill-BR-0117] | Removing AUTO_PAY_OFF replays all deferred payment attempts | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:312-319` |
| [Kill-BR-0118] | Two-phase-commit attempt row to prevent double payment | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:410-424`; `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:665-708` |
| [Kill-BR-0119] | Existing UNKNOWN transaction blocks a new purchase attempt on the same invoice | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:399-408` |
| [Kill-BR-0124] | Currency-mismatch handling for successful invoice payments | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:179-198` |
| [Kill-BR-0125] | Payment-level transaction currency must be consistent across a payment's transactions | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java:105-110` |
| [Kill-BR-0126] | Automatic invoice payment always uses the account's currency | High | `payment/src/main/java/org/killbill/billing/payment/bus/PaymentBusEventHandler.java:101-107` |
| [Kill-BR-0127] | API-originated payment failures are not retried by default | High | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:266-272`; `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:567-572` |
| [Kill-BR-0130] | UNKNOWN-status transactions are never rescheduled by the retry state machine | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/control/DefaultControlCompleted.java:56-106` |
| [Kill-BR-0134] | Janitor attempt-completion: missing transaction => ABORTED | High | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:229-242` |
| [Kill-BR-0135] | Janitor attempt-completion defers to the transaction Janitor for UNKNOWN | High | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:244-248` |
| [Kill-BR-0136] | isApiPayment must be true to guarantee retries happen | Medium | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:205-210` |
| [Kill-BR-0138] | Control plugins may only overwrite paymentMethodId if explicitly allowed | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:128-136`; `util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java:133-136` |
| [Kill-BR-0142] | Chargeback (and reversal) states are always treated as "success" states for last-success tracking | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentStateMachineHelper.java:236-239`; `payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java:150-150` |
| [Kill-BR-0143] | Payment-plugin exceptions/timeouts degrade to UNKNOWN transaction + ERRORED payment state | High | `payment/src/main/java/org/killbill/billing/payment/core/sm/payments/PaymentOperation.java:84-110` |
| [Kill-BR-0152] | Balance-clearing / invoice/payment events trigger re-evaluation | High | `overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java:96-157` |
| [Kill-BR-0156] | Plugin-sourced usage supersedes internally recorded usage | High | `usage/src/main/java/org/killbill/billing/usage/api/BaseUserApi.java:63-94`; `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:94-108`; `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:110-134`; `usage/src/main/java/org/killbill/billing/usage/api/svcs/DefaultInternalUserApi.java:63-84` |
| [Kill-BR-0157] | Per-unit-type usage aggregation (roll-up) | High | `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:136-156`; `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:158-168` |
| [Kill-BR-0158] | Usage periods segmented by subscription transition times | High | `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:110-134` |
| [Kill-BR-0159] | Cancellation-day inclusive boundary for invoiced usage | High | `usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg:37-48`; `usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg:50-60`; `usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg:62-73`; `usage/src/main/java/org/killbill/billing/usage/api/svcs/DefaultInternalUserApi.java:63-84` |
| [Kill-BR-0160] | Per-tenant config single-value / "latest wins" invariant | High | `tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenantInternalApi.java:132-141`; `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:126-137`; `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:140-157`; `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:202-204` |
| [Kill-BR-0161] | System-key-only cross-node broadcast/cache-invalidation | High | `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:190-197`; `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:202-204` |
| [Kill-BR-0162] | Selective server-side caching of tenant config keys | High | `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:63-69`; `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:133-140`; `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:206-208` |
| [Kill-BR-0165] | API secret is salted, iteratively hashed, and never stored/exposed in clear text | High | `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:87-103`; `tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java:105-115`; `tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java:43-43`; `tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java:113-123`; `tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java:174-185` |
| [Kill-BR-0166] | Single configured provider, no cross-provider fallback | High | `currency/src/main/java/org/killbill/billing/currency/api/DefaultCurrencyConversionApi.java:43-49`; `util/src/main/java/org/killbill/billing/util/config/definition/CurrencyConfig.java:26-29`; `currency/src/main/java/org/killbill/billing/currency/api/DefaultCurrencyConversionApi.java:58-67` |

## Rules unique to the new extraction (BUSINESS_RULES.md)

74 rules with no overlapping citation in the old inventory:

| ID | Name | Category | Priority | Confidence | Source |
|---|---|---|---|---|---|
| [RULE-001] | Recurring proration percentage between two dates | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:80-89` |
| [RULE-003] | Recurring invoice item amount formula | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:260-278` |
| [RULE-004] | Fixed price invoice item amount | Calculation | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:434-462` |
| [RULE-005] | Bill-cycle-day alignment caps at month length | Calculation | P1 | High | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:74-88` |
| [RULE-006] | Invoice balance formula | Calculation | P0 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:96-110,157-242` |
| [RULE-008] | Tiered usage billing — ALL_TIERS policy | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:219-266` |
| [RULE-009] | Tiered usage billing — TOP_TIER policy | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:268-299` |
| [RULE-010] | Capacity-based usage billing — flat tier price | Calculation | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalCapacityUsageInArrear.java:114-153` |
| [RULE-016] | Month-based billing periods roll the aligned date to the next month once the BCD has passed | Calculation | P1 | High | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:97-120` |
| [RULE-017] | Overlapping/adjacent disabled billing durations across account, bundle, and subscription level blocks are merged | Calculation | P1 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:320-355;junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DisabledDuration.java:90-121` |
| [RULE-018] | Blocked-duration billing suppression with merge of overlapping durations | Calculation | P0 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-259,320-354` |
| [RULE-021] | Currency rounding for stored monetary amounts (half-up) | Calculation | P0 | High | `util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:27-35` |
| [RULE-022] | Display-amount rounding for overdue/billing-state formatting | Calculation | P2 | High | `util/src/main/java/org/killbill/billing/util/DefaultAmountFormatter.java:23-35` |
| [RULE-023] | Usage in-arrear reconciliation bills only the newly-owed delta | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:88-99` |
| [RULE-025] | Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total | Calculation | P1 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:39-70` |
| [RULE-028] | Plan-phase duration date arithmetic | Calculation | P0 | High | `catalog/src/main/java/org/killbill/billing/catalog/DefaultDuration.java:64-104` |
| [RULE-031] | Repair amount is capped at the item's remaining net amount | Calculation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:182-199` |
| [RULE-032] | Item split proration for partial-period repairs | Calculation | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:152-166` |
| [RULE-033] | Migrated invoices always report zero balance | Calculation | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceModelDaoHelper.java:33-46` |
| [RULE-045] | TOP_UP usage block requires a minimum top-up credit | Validation | P2 | High | `catalog/src/main/java/org/killbill/billing/catalog/DefaultBlock.java:91-111` |
| [RULE-048] | Usage periods already covered by an existing invoice item are skipped to prevent double billing | Validation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:294-307` |
| [RULE-049] | Same-day usage items are excluded when re-billing a larger period | Validation | P1 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:573-598` |
| [RULE-059] | Payment transaction external key uniqueness and account isolation | Validation | P0 | Medium | `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:327-350` |
| [RULE-060] | At most one PENDING initial payment transaction per payment | Validation | P0 | Medium | `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:352-385` |
| [RULE-066] | Plugins may not set reserved entitlement blocking states or service name | Validation | P1 | High | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultSubscriptionApi.java:369-379` |
| [RULE-068] | Bundle transfer eligibility: skip cancelled subscriptions and optionally skip add-ons | Validation | P1 | High | `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:242-245,279-282` |
| [RULE-082] | Default payment method must belong to the target account | Validation | P0 | High | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:552-556` |
| [RULE-083] | AUTO_PAY_OFF tag cannot be removed without a default payment method on file | Validation | P0 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:1441-1455` |
| [RULE-084] | Invoice-list query modes are mutually exclusive | Validation | P2 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:711-713` |
| [RULE-085] | Plan resolution requires product + billing period, with price-list fallback to default | Validation | P1 | High | `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java:206-227` |
| [RULE-087] | Control tags may only be applied to their designated object types | Validation | P1 | High | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDao.java:204-213` |
| [RULE-088] | User-defined tag definition name cannot collide with a reserved control tag name | Validation | P1 | High | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:141-144` |
| [RULE-089] | Tag definition names must be globally unique | Validation | P1 | High | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:150-154` |
| [RULE-093] | Role-permission definitions are sanitized and collapsed to group-level wildcards | Validation | P1 | High | `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:236-246,265-304` |
| [RULE-094] | Credit line-item currency must match account currency (or defaults to it) | Validation | P0 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/JaxRsResourceBase.java:617-661` |
| [RULE-095] | Usage records cannot be recorded past a subscription's entitlement end date | Validation | P1 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/UsageResource.java:119-141` |
| [RULE-096] | Plan-phase price override must specify at least one price component | Validation | P1 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/SubscriptionResourceHelpers.java:89-97` |
| [RULE-097] | Dry-run subscription-event spec requires mutually-consistent fields per action type | Validation | P1 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:469-488` |
| [RULE-098] | External invoice payments cannot specify an explicit payment method | Validation | P1 | High | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:716-739` |
| [RULE-102] | Base-entitlement bundle creation requires at least one base entitlement specifier | Validation | P1 | High | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:453-466` |
| [RULE-103] | Chargeback amount bounded by remaining paid amount | Validation | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:933-954` |
| [RULE-107] | START_OF_TERM is not a legal default cancellation policy in the base catalog | Validation | P2 | High | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseCancelPolicy.java:51-55` |
| [RULE-108] | Security user, role, and role-permission uniqueness on create | Validation | P1 | High | `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:57-64,72-89,105-113` |
| [RULE-111] | Overdue transition is a no-op on unchanged state name | Lifecycle | P1 | High | `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:127-132` |
| [RULE-115] | Committing fires invoice-creation event; voiding fires adjustment event and deactivates usage tracking | Lifecycle | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1397-1414` |
| [RULE-117] | Reused draft invoices auto-promote to COMMITTED but never demote | Lifecycle | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:513-536` |
| [RULE-118] | Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED | Lifecycle | P1 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:1481-1499` |
| [RULE-119] | Payment transaction state machine per transaction type | Lifecycle | P0 | High | `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:38-409` |
| [RULE-120] | Cross-transaction-type payment workflow linkage | Lifecycle | P0 | High | `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:412-497` |
| [RULE-121] | Payment retry loop is unbounded at the state-machine level | Lifecycle | P1 | Medium | `payment/src/main/resources/org/killbill/billing/payment/retry/RetryStates.xml:22-84` |
| [RULE-122] | Billing-blocked periods insert a zero-charge 'disable' event and restore pricing on re-enable | Lifecycle | P0 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-317` |
| [RULE-125] | Subscription entitlement-state derivation from event stream | Lifecycle | P1 | Medium | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:192-205,1007-1050` |
| [RULE-128] | Blocked periods are aggregated into disabled billing durations and injected as billing on/off events | Lifecycle | P0 | Medium | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:82-161,196-282,319-357` |
| [RULE-129] | Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation | Lifecycle | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:313-334` |
| [RULE-133] | Refund idempotency by transaction cookie id | Lifecycle | P0 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:842-880` |
| [RULE-134] | Chargeback reversal resets invoice payment status to INIT | Lifecycle | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:970-1003` |
| [RULE-137] | Idempotent get-or-create for the external/manual-pay payment method | Lifecycle | P1 | High | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:475-491` |
| [RULE-138] | Subscription events occurring at or after a CANCEL/EXPIRED event are discarded | Lifecycle | P0 | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:1093-1123` |
| [RULE-140] | Account parking/unparking implemented via idempotent system tag | Lifecycle | P1 | Medium | `invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java:46-61` |
| [RULE-141] | Role permission update computes and applies a diff (add new, deactivate removed) | Lifecycle | P2 | High | `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:117-140` |
| [RULE-147] | Default overdue escalation tiers | Policy | P1 | Medium | `profiles/killbill/src/main/resources/overdue.xml:19-61` |
| [RULE-151] | Blocked billing periods shorter than one day are not disabled | Policy | P2 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68` |
| [RULE-154] | Cancellation of an EXPIRED subscription is blocked | Policy | P1 | High | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:191-194,213-217,230-234` |
| [RULE-156] | Blocking-state cascade blocks change/entitlement/billing actions (OR across levels) | Policy | P0 | High | `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:44-91,137-188,218-248; entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java:25-38` |
| [RULE-160] | Sub-one-day blocking periods are not disabled for billing | Policy | P1 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68` |
| [RULE-161] | AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing | Policy | P0 | High | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DefaultInternalBillingApi.java:99-115,208-214` |
| [RULE-163] | Bundle transfer: old subscription cancellation date honors charged-through date | Policy | P1 | High | `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:250-268` |
| [RULE-166] | Chargeable invoice item type classification | Policy | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:87-94` |
| [RULE-168] | Existing-subscription catalog grandfathering | Policy | P0 | High | `subscription/src/main/java/org/killbill/billing/subscription/catalog/SubscriptionCatalog.java:168-222` |
| [RULE-175] | Item adjustments linked to an ignored item are themselves dropped | Policy | P1 | High | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:121-133` |
| [RULE-177] | RBAC permission check supports AND/OR logic across required permissions | Policy | P0 | High | `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:177-204` |
| [RULE-178] | Bundle transfer requires the bundle to actually belong to the declared source account | Policy | P0 | High | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:313-323` |
| [RULE-179] | Bundle transfer billing policy determines whether the source subscription is cancelled immediately or at end of term | Policy | P1 | High | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:298-311` |
| [RULE-184] | Parked accounts are skipped for automatic invoicing | Policy | P0 | High | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:303-320` |

## Matched rules

99 old rules matched to 110 new rules via 137 citation-overlap pairs:

| Old ID | Old Name | New ID | New Name | New Priority | New Confidence |
|---|---|---|---|---|---|
| [Kill-BR-0001] | Immutable account external key | [RULE-057] | Account external key, currency, BCD, timezone, and reference time are immutable once set | P0 | Medium |
| [Kill-BR-0002] | Immutable account currency once set | [RULE-057] | Account external key, currency, BCD, timezone, and reference time are immutable once set | P0 | Medium |
| [Kill-BR-0007] | External key uniqueness at account creation | [RULE-055] | Account external key must be unique and ≤ 255 characters | P1 | High |
| [Kill-BR-0008] | External key length limit (255 chars) | [RULE-055] | Account external key must be unique and ≤ 255 characters | P1 | High |
| [Kill-BR-0009] | Parent account must exist before assignment | [RULE-055] | Account external key must be unique and ≤ 255 characters | P1 | High |
| [Kill-BR-0009] | Parent account must exist before assignment | [RULE-056] | Parent account must exist when linking a new account | P1 | High |
| [Kill-BR-0011] | Account email must be unique by ID (no duplicate insert) | [RULE-080] | Account email record add is idempotent by record id, not by email address | P2 | Medium |
| [Kill-BR-0013] | Update semantics — merge vs. reset (treatNullValueAsReset) | [RULE-123] | Account BCD is set exactly once via merge, then locked | P1 | High |
| [Kill-BR-0018] | Plan-change policy resolution via ordered rule matching | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0019] | ILLEGAL plan-change rejection | [RULE-155] | Illegal plan-change policy blocks the change | P0 | High |
| [Kill-BR-0019] | ILLEGAL plan-change rejection | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0020] | Plan-change realignment resolution | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0021] | Cancellation policy resolution | [RULE-172] | Default plan-creation alignment and cancellation policy | P1 | High |
| [Kill-BR-0021] | Cancellation policy resolution | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0022] | Billing alignment resolution (ACCOUNT vs BUNDLE vs SUBSCRIPTION) | [RULE-169] | Default billing alignment falls back to ACCOUNT | P1 | High |
| [Kill-BR-0022] | Billing alignment resolution (ACCOUNT vs BUNDLE vs SUBSCRIPTION) | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0023] | Create-alignment resolution (START_OF_BUNDLE default) | [RULE-172] | Default plan-creation alignment and cancellation policy | P1 | High |
| [Kill-BR-0023] | Create-alignment resolution (START_OF_BUNDLE default) | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0024] | Price-list-transition rule on plan change | [RULE-155] | Illegal plan-change policy blocks the change | P0 | High |
| [Kill-BR-0024] | Price-list-transition rule on plan change | [RULE-183] | Default fallbacks when no catalog rule case matches | P1 | High |
| [Kill-BR-0025] | First-match-wins rule evaluation order | [RULE-171] | Catalog rule-case matching (first match, wildcard nulls) | P1 | Medium |
| [Kill-BR-0025] | First-match-wins rule evaluation order | [RULE-182] | Catalog case-rule matching: first matching rule wins, null fields act as wildcards | P1 | High |
| [Kill-BR-0026] | Mandatory catch-all default rule + duplicate-rule rejection | [RULE-100] | Plan-change and plan-cancellation catalog rules require a catch-all default case | P1 | Medium |
| [Kill-BR-0026] | Mandatory catch-all default rule + duplicate-rule rejection | [RULE-106] | Catalog validation requires an explicit wildcard/default rule for change and cancel policies | P1 | High |
| [Kill-BR-0027] | Effective-dated catalog version selection | [RULE-167] | Catalog version selection by effective date | P0 | High |
| [Kill-BR-0028] | Duplicate catalog-version effective-date rejection | [RULE-069] | Catalog version consistency validation | P1 | High |
| [Kill-BR-0029] | Uniform plan-shape-across-versions constraint | [RULE-069] | Catalog version consistency validation | P1 | High |
| [Kill-BR-0031] | Phase-type placement constraints on a plan | [RULE-091] | Plan phase-type composition constraints | P1 | High |
| [Kill-BR-0031] | Phase-type placement constraints on a plan | [RULE-101] | Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase | P1 | High |
| [Kill-BR-0032] | Price-effective-date-can't-precede-catalog-version rule | [RULE-070] | Grandfather date cannot precede its own catalog version's effective date | P2 | High |
| [Kill-BR-0032] | Price-effective-date-can't-precede-catalog-version rule | [RULE-101] | Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase | P1 | High |
| [Kill-BR-0033] | Recurring-billing-mode requirement for recurring (non-usage-only) plans | [RULE-101] | Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase | P1 | High |
| [Kill-BR-0034] | Currency-support enforcement on prices | [RULE-044] | Negative catalog prices rejected | P1 | High |
| [Kill-BR-0035] | Zero-price-in-all-currencies default rule | [RULE-011] | Zero-price plan when no prices defined | P1 | High |
| [Kill-BR-0035] | Zero-price-in-all-currencies default rule | [RULE-043] | Missing currency price is a hard error | P1 | High |
| [Kill-BR-0036] | Price-list plan lookup with DEFAULT-price-list fallback | [RULE-086] | Price-list plan lookup falls back to default list; ambiguous match is rejected | P1 | High |
| [Kill-BR-0037] | Ambiguous-plan-for-pricelist rejection | [RULE-086] | Price-list plan lookup falls back to default list; ambiguous match is rejected | P1 | High |
| [Kill-BR-0039] | Usage/tier structural requirements by billing mode and usage type | [RULE-067] | Usage tier section must define blocks or limits appropriate to its type | P1 | High |
| [Kill-BR-0039] | Usage/tier structural requirements by billing mode and usage type | [RULE-092] | Usage catalog section must declare pricing structures matching its billing mode/type | P1 | High |
| [Kill-BR-0039] | Usage/tier structural requirements by billing mode and usage type | [RULE-099] | Catalog usage sections must define pricing structures appropriate to their billing mode | P1 | Medium |
| [Kill-BR-0041] | Limit min/max bound validation | [RULE-054] | Usage/product limit compliance check (min/max) | P1 | Low |
| [Kill-BR-0043] | First-non-zero-recurring-charge computation (trial/discount skip-through) | [RULE-029] | First non-zero recurring charge date | P1 | High |
| [Kill-BR-0044] | Per-tenant price-override validation (can't add a price dimension that doesn't exist) | [RULE-046] | Price override cannot introduce a price dimension that didn't exist | P1 | High |
| [Kill-BR-0047] | Add-on simple-plan creation prerequisite | [RULE-090] | Simple plan descriptor validation for API-created plans | P1 | High |
| [Kill-BR-0050] | Cancel is blocked on terminal/conflicting states | [RULE-062] | Re-cancellation of an already future-cancelled/expiring subscription is blocked unless the new date is earlier | P1 | High |
| [Kill-BR-0051] | Base-plan cancellation/expiry cascades to add-ons | [RULE-126] | Add-on cascade cancellation on base plan cancel/change (subscription-level) | P0 | High |
| [Kill-BR-0054] | Change-plan state/date validation | [RULE-053] | Change/cancel effective date cannot precede last processed transition | P1 | High |
| [Kill-BR-0054] | Change-plan state/date validation | [RULE-063] | Effective date for subscription mutation must not precede last recorded transition | P1 | High |
| [Kill-BR-0054] | Change-plan state/date validation | [RULE-064] | Change-plan blocked for non-active, future-cancelled, or already-future-expired subscriptions | P1 | High |
| [Kill-BR-0054] | Change-plan state/date validation | [RULE-153] | Subscription state guard for plan changes | P0 | High |
| [Kill-BR-0055] | Add-on plan-category and product eligibility on change | [RULE-051] | Max add-ons of the same plan per bundle (plan change) | P1 | High |
| [Kill-BR-0055] | Add-on plan-category and product eligibility on change | [RULE-052] | Plan change forbidden across product categories | P0 | High |
| [Kill-BR-0055] | Add-on plan-category and product eligibility on change | [RULE-065] | Max add-on instances per bundle enforced on plan change | P1 | High |
| [Kill-BR-0056] | Add-on creation eligibility (checkAddonCreationRights) | [RULE-061] | Add-on creation eligibility against base plan | P1 | Medium |
| [Kill-BR-0056] | Add-on creation eligibility (checkAddonCreationRights) | [RULE-152] | Add-on creation eligibility against base subscription | P0 | Medium |
| [Kill-BR-0058] | Per-bundle add-on quantity cap at creation time | [RULE-050] | Max add-ons of the same plan per bundle (creation) | P1 | High |
| [Kill-BR-0059] | Plan-phase alignment on create/change (trial -> discount -> evergreen sequencing) | [RULE-026] | Plan phase sequencing via cumulative duration | P1 | High |
| [Kill-BR-0059] | Plan-phase alignment on create/change (trial -> discount -> evergreen sequencing) | [RULE-148] | Plan-change alignment determines the effective phase start date | P0 | High |
| [Kill-BR-0061] | Bill-cycle-day (BCD) resolution by billing alignment | [RULE-015] | Bill-cycle-day source depends on alignment type | P1 | High |
| [Kill-BR-0061] | Bill-cycle-day (BCD) resolution by billing alignment | [RULE-150] | Account-level BCD alignment falls back to subscription alignment until the account BCD is set | P1 | High |
| [Kill-BR-0064] | Cross-source blocking-state OR aggregation | [RULE-159] | Blocking-state flags OR-aggregate across account, bundle, and subscription levels | P0 | High |
| [Kill-BR-0065] | Account/bundle/subscription blocking-state hierarchy (no downward override) | [RULE-159] | Blocking-state flags OR-aggregate across account, bundle, and subscription levels | P0 | High |
| [Kill-BR-0067] | Entitlement state derivation precedence (CANCELLED > EXPIRED > PENDING > BLOCKED/ACTIVE) | [RULE-110] | Entitlement state derivation precedence | P1 | Medium |
| [Kill-BR-0068] | Cancellation date validation and idempotency guard | [RULE-039] | Cannot cancel an already-cancelled entitlement | P1 | High |
| [Kill-BR-0068] | Cancellation date validation and idempotency guard | [RULE-040] | Cancellation effective date cannot precede entitlement or subscription start | P1 | High |
| [Kill-BR-0069] | Cancellation/change cascades to compatible add-ons | [RULE-113] | Add-on cascade cancellation on base plan change or cancel | P1 | High |
| [Kill-BR-0070] | Un-cancellation (uncancel) rules | [RULE-041] | Uncancel only allowed with a pending or existing cancellation to reverse | P1 | High |
| [Kill-BR-0071] | Bundle-level pause/resume blocks/unblocks all three axes uniformly | [RULE-124] | Pause bundle fully blocks; resume bundle fully clears | P1 | High |
| [Kill-BR-0071] | Bundle-level pause/resume blocks/unblocks all three axes uniformly | [RULE-127] | Bundle pause blocks everything; resume clears everything | P0 | High |
| [Kill-BR-0072] | Change-plan and add-entitlement are gated by isBlockChange/isBlockEntitlement, not just isBlockBilling | [RULE-157] | Add-on creation blocked when base subscription is cancelled, not-yet-pending, or itself blocked | P0 | High |
| [Kill-BR-0073] | Blocking-state deduplication/history compaction on write | [RULE-135] | Consecutive duplicate blocking states are pruned when inserting a new one | P1 | High |
| [Kill-BR-0073] | Blocking-state deduplication/history compaction on write | [RULE-136] | Blocking-state notification/bus-event batching by aggregation mode | P2 | Medium |
| [Kill-BR-0074] | Overdue-driven (and any third-party service) blocking integrates via the same generic per-service OR mechanism, with no entitlement-side special-casing | [RULE-110] | Entitlement state derivation precedence | P1 | Medium |
| [Kill-BR-0074] | Overdue-driven (and any third-party service) blocking integrates via the same generic per-service OR mechanism, with no entitlement-side special-casing | [RULE-159] | Blocking-state flags OR-aggregate across account, bundle, and subscription levels | P0 | High |
| [Kill-BR-0075] | Leading/trailing period proration with optional fixed-days-in-month mode | [RULE-002] | Fixed-days-in-month proration override | P1 | High |
| [Kill-BR-0077] | IN_ARREAR vs IN_ADVANCE billing-mode effective end date, with "greedy" early in-arrear billing | [RULE-142] | In-arrear greedy billing mode | P1 | High |
| [Kill-BR-0078] | Usage billing is IN_ARREAR only | [RULE-173] | Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling | P1 | Medium |
| [Kill-BR-0080] | $0 RECURRING items and all FIXED items are excluded from repair | [RULE-174] | Zero-amount recurring and fixed items are excluded from the repair tree | P1 | High |
| [Kill-BR-0081] | Over-repair / double-repair guard (no more can be repaired+adjusted than the original amount) | [RULE-047] | A recurring invoice item is only pruned once fully repaired, never over-repaired | P1 | High |
| [Kill-BR-0087] | AUTO_INVOICING_DRAFT tag governs generated-invoice status | [RULE-116] | New invoice DRAFT vs COMMITTED driven by account auto-invoice-draft setting | P0 | High |
| [Kill-BR-0089] | Target-date validation and clamping | [RULE-030] | Invoice target date is pinned to the latest existing invoice with usage/recurring items | P1 | High |
| [Kill-BR-0089] | Target-date validation and clamping | [RULE-038] | Target date cannot be too far in the future | P1 | High |
| [Kill-BR-0091] | Invoice voiding immutability rules | [RULE-042] | Cannot void a repaired, used-credit-generating, or paid invoice | P0 | High |
| [Kill-BR-0092] | Invoice status transitions are one-directional and idempotency-guarded | [RULE-114] | Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID | P0 | Medium |
| [Kill-BR-0094] | Item-adjustment/refund amount cannot exceed remaining adjustable amount | [RULE-073] | Invoice item adjustment amount must be positive | P0 | High |
| [Kill-BR-0094] | Item-adjustment/refund amount cannot exceed remaining adjustable amount | [RULE-074] | Cannot adjust a VOID invoice | P0 | High |
| [Kill-BR-0094] | Item-adjustment/refund amount cannot exceed remaining adjustable amount | [RULE-075] | Adjustment currency must match invoice currency | P0 | High |
| [Kill-BR-0094] | Item-adjustment/refund amount cannot exceed remaining adjustable amount | [RULE-104] | Refund amount must not exceed original payment and must match specified item adjustments | P0 | High |
| [Kill-BR-0095] | Adjustment/credit/external-charge items blocked on non-DRAFT invoices | [RULE-078] | Cannot add external charge/credit to an already-committed invoice | P0 | High |
| [Kill-BR-0097] | External charge / credit amount and currency validation | [RULE-076] | External charge / credit amount must be non-negative | P0 | High |
| [Kill-BR-0097] | External charge / credit amount and currency validation | [RULE-077] | External charge / credit currency must match account currency | P1 | High |
| [Kill-BR-0098] | CBA credit generation on negative invoice balance | [RULE-012] | Negative invoice balance auto-generates account credit | P0 | Medium |
| [Kill-BR-0098] | CBA credit generation on negative invoice balance | [RULE-013] | Available account credit is auto-applied to a committed positive-balance invoice | P0 | High |
| [Kill-BR-0098] | CBA credit generation on negative invoice balance | [RULE-019] | Automatic account credit (CBA) generation and application | P0 | High |
| [Kill-BR-0099] | CBA credit consumption on positive balance of a committed, non-written-off invoice with no pending payment | [RULE-012] | Negative invoice balance auto-generates account credit | P0 | Medium |
| [Kill-BR-0099] | CBA credit consumption on positive balance of a committed, non-written-off invoice with no pending payment | [RULE-013] | Available account credit is auto-applied to a committed positive-balance invoice | P0 | High |
| [Kill-BR-0099] | CBA credit consumption on positive balance of a committed, non-written-off invoice with no pending payment | [RULE-019] | Automatic account credit (CBA) generation and application | P0 | High |
| [Kill-BR-0100] | CBA distributed oldest-invoice-first across unpaid invoices | [RULE-019] | Automatic account credit (CBA) generation and application | P0 | High |
| [Kill-BR-0100] | CBA distributed oldest-invoice-first across unpaid invoices | [RULE-149] | Account credit is distributed across unpaid invoices oldest-first | P1 | High |
| [Kill-BR-0101] | CBA deletion/reclaim rules distinguish credit consumption vs. credit generation, and block deleting system-generated credit | [RULE-105] | Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice | P1 | High |
| [Kill-BR-0101] | CBA deletion/reclaim rules distinguish credit consumption vs. credit generation, and block deleting system-generated credit | [RULE-180] | Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first | P1 | Medium |
| [Kill-BR-0102] | Written-off and zero-parent-balance child invoices excluded from account balance | [RULE-020] | Account balance calculation excludes non-committed and written-off/child-consolidated invoices | P0 | High |
| [Kill-BR-0103] | Child invoice balance derived from parent PARENT_SUMMARY allocation when parent has zero raw balance | [RULE-014] | Child-account invoice balance nets against the parent invoice's charge for that child | P1 | Medium |
| [Kill-BR-0104] | Child-to-parent credit transfer requires positive child CBA and creates a linked charge/credit pair | [RULE-079] | Child-to-parent credit transfer eligibility | P0 | High |
| [Kill-BR-0106] | Migration invoices bypass the normal generation/repair pipeline | [RULE-105] | Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice | P1 | High |
| [Kill-BR-0108] | Account parking on unexpected/inconsistent invoicing state | [RULE-139] | Accounts are auto-parked on unrecoverable invoice-generation errors | P0 | High |
| [Kill-BR-0109] | Sanity safety-bound duplicate/overlap detection during item generation | [RULE-034] | Invoice-generation safety bound: max daily items per subscription | P0 | High |
| [Kill-BR-0109] | Sanity safety-bound duplicate/overlap detection during item generation | [RULE-035] | Invoice-generation safety bound: duplicate FIXED/RECURRING items | P0 | High |
| [Kill-BR-0114] | Payment amount is capped at, and defaults to, the invoice balance | [RULE-036] | API payment cannot exceed invoice balance | P0 | High |
| [Kill-BR-0116] | AUTO_PAY_OFF suppresses only system-triggered (non-API) auto-charges | [RULE-143] | AUTO_PAY_OFF blocks only system-triggered payments | P1 | High |
| [Kill-BR-0120] | Refunds are never retried | [RULE-165] | REFUND and CREDIT transactions are never retried on failure | P1 | High |
| [Kill-BR-0121] | Refund amount capped by invoice item sums, aborted if fully consumed | [RULE-037] | Refund amount validation against invoice items | P0 | High |
| [Kill-BR-0122] | No partial chargebacks | [RULE-164] | Chargeback is idempotent — no partial/duplicate chargebacks | P0 | High |
| [Kill-BR-0123] | Chargeback amount/currency fallback chain | [RULE-024] | Chargeback amount/currency resolution fallback | P1 | Medium |
| [Kill-BR-0123] | Chargeback amount/currency fallback chain | [RULE-164] | Chargeback is idempotent — no partial/duplicate chargebacks | P0 | High |
| [Kill-BR-0128] | Fixed-day-list retry backoff for PAYMENT_FAILURE | [RULE-144] | Payment failure retry schedule | P1 | Medium |
| [Kill-BR-0129] | Exponential-backoff retry for PLUGIN_FAILURE | [RULE-145] | Plugin-failure retry backoff (suspected off-by-one) | P1 | Medium |
| [Kill-BR-0131] | Janitor only reconciles PENDING/UNKNOWN transactions | [RULE-131] | Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones | P0 | High |
| [Kill-BR-0132] | Janitor state-repair mapping from re-queried plugin status | [RULE-131] | Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones | P0 | High |
| [Kill-BR-0133] | Gateway status -> internal TransactionStatus mapping (and its retry implications) | [RULE-130] | Payment plugin status mapped to internal transaction status | P0 | High |
| [Kill-BR-0137] | Janitor re-check backoff differs by transaction status (UNKNOWN vs PENDING) | [RULE-162] | Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list | P1 | Medium |
| [Kill-BR-0139] | Deleting the account's default payment method can auto-enable AUTO_PAY_OFF | [RULE-158] | Default payment method cannot be deleted without explicit override | P0 | High |
| [Kill-BR-0140] | Kill Bill is authoritative for "default payment method," except when it isn't | [RULE-181] | Refreshing payment methods from a plugin never un-sets the KB default | P1 | High |
| [Kill-BR-0141] | Single external-payment-method-per-account | [RULE-081] | Only one payment method per account may use the external/manual-pay plugin | P1 | High |
| [Kill-BR-0144] | AUTO_PAY_OFF gate duplicated between ProcessorBase and InvoicePaymentControlPluginApi | [RULE-143] | AUTO_PAY_OFF blocks only system-triggered payments | P1 | High |
| [Kill-BR-0145] | First-matching-state overdue evaluation (state-order priority) | [RULE-132] | Overdue state evaluation order: first matching configured state wins | P1 | Medium |
| [Kill-BR-0146] | Composite unpaid-invoice condition evaluation | [RULE-109] | Overdue state trigger conditions (all must hold) | P1 | Medium |
| [Kill-BR-0147] | Unified entitlement/billing block flag | [RULE-146] | Billing-blocking overdue state auto-disables invoicing | P0 | High |
| [Kill-BR-0148] | AUTO_INVOICING_OFF toggled on billing block/unblock transitions | [RULE-146] | Billing-blocking overdue state auto-disables invoicing | P0 | High |
| [Kill-BR-0149] | Subscription cancellation policy on entering an overdue state | [RULE-112] | Overdue-driven subscription cancellation policy | P0 | High |
| [Kill-BR-0150] | OVERDUE_ENFORCEMENT_OFF short-circuits all overdue processing | [RULE-170] | OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation | P0 | High |
| [Kill-BR-0151] | Auto-reevaluation notification scheduling (state-specific vs. initial interval) | [RULE-176] | Overdue re-evaluation notification scheduling | P1 | High |
| [Kill-BR-0153] | Parent-account payment delegation for billing state | [RULE-027] | Payment-delegated child account borrows the parent's billing state for overdue calculation | P1 | High |
| [Kill-BR-0154] | Unpaid-invoice set defines "overdue-relevant" invoices as of evaluation time | [RULE-007] | Unpaid invoice balance aggregation for overdue evaluation | P1 | High |
| [Kill-BR-0155] | Usage-record idempotency via tracking ID | [RULE-058] | Usage record idempotency by tracking id | P1 | High |
| [Kill-BR-0163] | Tenant external-key length limit | [RULE-071] | Tenant external key length limit | P1 | High |
| [Kill-BR-0164] | API-key uniqueness enforced at tenant creation | [RULE-072] | Tenant API key must be unique | P1 | Medium |
