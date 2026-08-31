# Kill Bill — Data Objects Catalog

Core DTOs / records / entities identified during business-rule extraction, with the rules (from `BUSINESS_RULES.md`) that consume or produce each.

Total objects: 20

## DefaultInvoice / InvoiceModelDao
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/model/DefaultInvoice.java:44-127`

| Field | Type | Note |
|---|---|---|
| id | UUID | invoice id, from EntityBase |
| accountId | UUID |  |
| invoiceNumber | Integer | null until persisted; DefaultInvoice.java:220 |
| invoiceDate | LocalDate |  |
| targetDate | LocalDate |  |
| currency | Currency |  |
| processedCurrency | Currency |  |
| migrationInvoice | boolean |  |
| isWrittenOff | boolean |  |
| status | InvoiceStatus | DRAFT/COMMITTED/VOID, DefaultInvoice.java:59 |
| isParentInvoice | boolean |  |
| parentInvoice | Invoice | nullable, self-referential for child-account consolidation |
| grpId | UUID |  |
| invoiceItems | List<InvoiceItem> |  |
| payments | List<InvoicePayment> |  |
| trackingIds | List<String> |  |

**Consumed/produced by:**
- Invoice balance formula
- Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID
- Committing fires invoice-creation event; voiding fires adjustment event and deactivates usage tracking
- Cannot void a repaired, used-credit-generating, or paid invoice
- New invoice DRAFT vs COMMITTED driven by account auto-invoice-draft setting
- Reused draft invoices auto-promote to COMMITTED but never demote
- Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED
- Child-account invoice balance nets against the parent invoice's charge for that child
- Migrated invoices always report zero balance
- Account balance calculation excludes non-committed and written-off/child-consolidated invoices
- Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation
- Invoice target date is pinned to the latest existing invoice with usage/recurring items
- Target date cannot be too far in the future
- Invoice-generation safety bound: max daily items per subscription
- Invoice-generation safety bound: duplicate FIXED/RECURRING items
- Usage record idempotency by tracking id
- Usage records cannot be recorded past a subscription's entitlement end date
- Cannot adjust a VOID invoice
- Cannot add external charge/credit to an already-committed invoice
- API payment cannot exceed invoice balance
- Available account credit is auto-applied to a committed positive-balance invoice

## InvoiceItemBase (abstract) / InvoiceItemCatalogBase / DefaultInvoiceItem family (RecurringInvoiceItem, FixedPriceInvoiceItem, UsageInvoiceItem, CreditBalanceAdjInvoiceItem, ItemAdjInvoiceItem, RepairAdjInvoiceItem, AdjInvoiceItem, ExternalChargeInvoiceItem, CreditAdjInvoiceItem, TaxInvoiceItem, ParentInvoiceItem)
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/model/InvoiceItemBase.java:34-100; InvoiceItemCatalogBase.java:33-42`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| invoiceId | UUID |  |
| accountId | UUID |  |
| childAccountId | UUID | only for parent-invoice child items |
| startDate | LocalDate |  |
| endDate | LocalDate |  |
| amount | BigDecimal | rounded via KillBillMoney.of, line 93 |
| currency | Currency |  |
| description | String |  |
| invoiceItemType | InvoiceItemType | FIXED/RECURRING/USAGE/CBA_ADJ/ITEM_ADJ/REPAIR_ADJ/EXTERNAL_CHARGE/CREDIT_ADJ/TAX/PARENT_SUMMARY etc |
| subscriptionId | UUID |  |
| bundleId | UUID |  |
| rate | BigDecimal | recurring-specific |
| linkedItemId | UUID | points to the item being repaired/adjusted/reversed |
| quantity | BigDecimal | usage-specific |
| itemDetails | String | usage-specific JSON blob |
| catalogEffectiveDate | DateTime | InvoiceItemCatalogBase.java:33 |
| productName / planName / phaseName / usageName | String | InvoiceItemCatalogBase.java:34-37 |

**Consumed/produced by:**
- Recurring invoice item amount formula
- Fixed price invoice item amount
- Recurring proration percentage between two dates
- Fixed-days-in-month proration override
- Item split proration for partial-period repairs
- A recurring invoice item is only pruned once fully repaired, never over-repaired
- Repair amount is capped at the item's remaining net amount
- Zero-amount recurring and fixed items are excluded from the repair tree
- Item adjustments linked to an ignored item are themselves dropped
- Usage periods already covered by an existing invoice item are skipped to prevent double billing
- Same-day usage items are excluded when re-billing a larger period
- Invoice-generation safety bound: duplicate FIXED/RECURRING items
- Invoice balance formula
- Refund amount validation against invoice items
- Refund amount must not exceed original payment and must match specified item adjustments
- Invoice item adjustment amount must be positive
- Adjustment currency must match invoice currency
- External charge / credit amount must be non-negative
- External charge / credit currency must match account currency
- Credit line-item currency must match account currency (or defaults to it)
- Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total
- Chargeable invoice item type classification
- Negative invoice balance auto-generates account credit
- Automatic account credit (CBA) generation and application
- Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice
- Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first
- Usage in-arrear reconciliation bills only the newly-owed delta
- Child-account invoice balance nets against the parent invoice's charge for that child
- Child-to-parent credit transfer eligibility

## PaymentModelDao
**Source:** `payment/src/main/java/org/killbill/billing/payment/dao/PaymentModelDao.java:32-52`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| accountId | UUID |  |
| paymentNumber | Integer | defaults to INVALID_PAYMENT_NUMBER=-17, line 34 |
| paymentMethodId | UUID |  |
| externalKey | String | defaults to id.toString() if null, line 51 |
| stateName | String | current payment state-machine state |
| lastSuccessStateName | String |  |

**Consumed/produced by:**
- Payment transaction state machine per transaction type
- Cross-transaction-type payment workflow linkage
- Payment retry loop is unbounded at the state-machine level
- API payment cannot exceed invoice balance
- AUTO_PAY_OFF blocks only system-triggered payments
- Payment transaction external key uniqueness and account isolation
- At most one PENDING initial payment transaction per payment
- Payment failure retry schedule
- Plugin-failure retry backoff (suspected off-by-one)

## PaymentTransactionModelDao
**Source:** `payment/src/main/java/org/killbill/billing/payment/dao/PaymentTransactionModelDao.java:36-49`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| attemptId | UUID |  |
| paymentId | UUID |  |
| transactionExternalKey | String | defaults to id.toString(), line 59 |
| transactionType | TransactionType | AUTHORIZE/CAPTURE/PURCHASE/REFUND/CREDIT/CHARGEBACK/VOID |
| effectiveDate | DateTime |  |
| transactionStatus | TransactionStatus | PENDING/SUCCESS/PAYMENT_FAILURE/PLUGIN_FAILURE/UNKNOWN |
| amount | BigDecimal |  |
| currency | Currency |  |
| processedAmount | BigDecimal |  |
| processedCurrency | Currency |  |
| gatewayErrorCode | String |  |
| gatewayErrorMsg | String |  |

**Consumed/produced by:**
- Payment transaction state machine per transaction type
- Cross-transaction-type payment workflow linkage
- Payment retry loop is unbounded at the state-machine level
- Payment failure retry schedule
- Plugin-failure retry backoff (suspected off-by-one)
- REFUND and CREDIT transactions are never retried on failure
- Refund idempotency by transaction cookie id
- Chargeback is idempotent — no partial/duplicate chargebacks
- Chargeback amount/currency resolution fallback
- Chargeback amount bounded by remaining paid amount
- Chargeback reversal resets invoice payment status to INIT
- Payment plugin status mapped to internal transaction status
- Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones
- Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list
- Refund amount validation against invoice items
- Refund amount must not exceed original payment and must match specified item adjustments

## PaymentMethodModelDao
**Source:** `payment/src/main/java/org/killbill/billing/payment/dao/PaymentMethodModelDao.java:31-46`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| externalKey | String |  |
| accountId | UUID |  |
| pluginName | String | e.g. __EXTERNAL_PAYMENT__/manual-pay plugin |
| isActive | Boolean |  |

**Consumed/produced by:**
- Default payment method cannot be deleted without explicit override
- Only one payment method per account may use the external/manual-pay plugin
- Default payment method must belong to the target account
- AUTO_PAY_OFF tag cannot be removed without a default payment method on file
- Idempotent get-or-create for the external/manual-pay payment method
- Refreshing payment methods from a plugin never un-sets the KB default
- External invoice payments cannot specify an explicit payment method

## PaymentAttemptModelDao
**Source:** `payment/src/main/java/org/killbill/billing/payment/dao/PaymentAttemptModelDao.java:36-50`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| accountId | UUID |  |
| paymentMethodId | UUID |  |
| paymentExternalKey | String |  |
| transactionId | UUID |  |
| transactionExternalKey | String |  |
| transactionType | TransactionType |  |
| stateName | String |  |
| amount | BigDecimal |  |
| currency | Currency |  |
| pluginName | String | comma-joined payment-control plugin names |
| pluginProperties | byte[] |  |

**Consumed/produced by:**
- Payment failure retry schedule
- Plugin-failure retry backoff (suspected off-by-one)
- Payment transaction external key uniqueness and account isolation
- At most one PENDING initial payment transaction per payment

## AccountModelDao
**Source:** `account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:40-97`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| externalKey | String | immutable once set, defaults to id.toString(); AccountModelDao.java:184-189 |
| email | String |  |
| name | String |  |
| firstNameLength | Integer |  |
| currency | Currency | immutable once set, AccountModelDao.java:191-196 |
| parentAccountId | UUID |  |
| isPaymentDelegatedToParent | Boolean |  |
| billingCycleDayLocal | int | immutable once set unless allowAccountBCDUpdate, lines 198-203 |
| paymentMethodId | UUID | default payment method |
| referenceTime | DateTime | immutable once set, lines 212-215 |
| timeZone | DateTimeZone | immutable once set, lines 205-210 |
| locale | String |  |
| address1/address2/companyName/city/stateOrProvince/country/postalCode/phone/notes | String |  |
| migrated | Boolean |  |

**Consumed/produced by:**
- Account external key must be unique and ≤ 255 characters
- Parent account must exist when linking a new account
- Account external key, currency, BCD, timezone, and reference time are immutable once set
- Account BCD is set exactly once via merge, then locked
- Account-level BCD alignment falls back to subscription alignment until the account BCD is set
- Bill-cycle-day source depends on alignment type
- Bill-cycle-day alignment caps at month length
- Payment-delegated child account borrows the parent's billing state for overdue calculation
- Account balance calculation excludes non-committed and written-off/child-consolidated invoices
- Default billing alignment falls back to ACCOUNT

## SubscriptionModelDao
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/engine/dao/model/SubscriptionModelDao.java:32-58`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| bundleId | UUID |  |
| externalKey | String |  |
| category | ProductCategory | BASE/ADD_ON/STANDALONE |
| startDate | DateTime |  |
| bundleStartDate | DateTime |  |
| chargedThroughDate | DateTime |  |
| migrated | boolean |  |

**Consumed/produced by:**
- Add-on creation eligibility against base subscription
- Add-on creation eligibility against base plan
- Max add-ons of the same plan per bundle (creation)
- Max add-ons of the same plan per bundle (plan change)
- Max add-on instances per bundle enforced on plan change
- Add-on creation blocked when base subscription is cancelled, not-yet-pending, or itself blocked
- Add-on cascade cancellation on base plan change or cancel
- Add-on cascade cancellation on base plan cancel/change (subscription-level)
- Plan change forbidden across product categories
- Change/cancel effective date cannot precede last processed transition
- Effective date for subscription mutation must not precede last recorded transition
- Subscription state guard for plan changes
- Change-plan blocked for non-active, future-cancelled, or already-future-expired subscriptions
- Cancellation of an EXPIRED subscription is blocked
- Cannot cancel an already-cancelled entitlement
- Re-cancellation of an already future-cancelled/expiring subscription is blocked unless the new date is earlier
- Cancellation effective date cannot precede entitlement or subscription start
- Uncancel only allowed with a pending or existing cancellation to reverse
- Bundle transfer: old subscription cancellation date honors charged-through date
- Bundle transfer eligibility: skip cancelled subscriptions and optionally skip add-ons
- Subscription events occurring at or after a CANCEL/EXPIRED event are discarded
- Subscription entitlement-state derivation from event stream
- Entitlement state derivation precedence

## SubscriptionBundleModelDao
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/engine/dao/model/SubscriptionBundleModelDao.java:31-53`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| externalKey | String | may carry an internal tsf: rename prefix, lines 33-36,97-98 |
| accountId | UUID |  |
| lastSysUpdateDate | DateTime | deprecated |
| originalCreatedDate | DateTime |  |

**Consumed/produced by:**
- Base-entitlement bundle creation requires at least one base entitlement specifier
- Bundle transfer requires the bundle to actually belong to the declared source account
- Bundle transfer billing policy determines whether the source subscription is cancelled immediately or at end of term
- Pause bundle fully blocks; resume bundle fully clears
- Bundle pause blocks everything; resume clears everything
- AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing

## BlockingStateModelDao / DefaultBlockingState
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/dao/BlockingStateModelDao.java:32-58`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| blockableId | UUID | account/bundle/subscription id being blocked |
| type | BlockingStateType |  |
| state | String |  |
| service | String |  |
| blockChange | Boolean |  |
| blockEntitlement | Boolean |  |
| blockBilling | Boolean |  |
| effectiveDate | DateTime |  |
| isActive | boolean |  |

**Consumed/produced by:**
- Blocking-state cascade blocks change/entitlement/billing actions (OR across levels)
- Blocking-state flags OR-aggregate across account, bundle, and subscription levels
- Pause bundle fully blocks; resume bundle fully clears
- Bundle pause blocks everything; resume clears everything
- Billing-blocking overdue state auto-disables invoicing
- Billing-blocked periods insert a zero-charge 'disable' event and restore pricing on re-enable
- Blocked periods are aggregated into disabled billing durations and injected as billing on/off events
- Blocked-duration billing suppression with merge of overlapping durations
- Overlapping/adjacent disabled billing durations across account, bundle, and subscription level blocks are merged
- Blocked billing periods shorter than one day are not disabled
- Sub-one-day blocking periods are not disabled for billing
- Consecutive duplicate blocking states are pruned when inserting a new one
- Blocking-state notification/bus-event batching by aggregation mode
- Plugins may not set reserved entitlement blocking states or service name
- Overdue transition is a no-op on unchanged state name
- Overdue-driven subscription cancellation policy

## DefaultOverdueState / OverdueState config (overdue.xml-derived)
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueState.java:45-77`

| Field | Type | Note |
|---|---|---|
| condition | DefaultOverdueCondition | the trigger predicate for this state |
| name | String | max 50 chars, MAX_NAME_LENGTH line 47 |
| externalMessage | String |  |
| blockChanges | Boolean |  |
| disableEntitlement | Boolean | disableEntitlementAndChangesBlocked |
| subscriptionCancellationPolicy | OverdueCancellationPolicy | default NONE |
| isClearState | Boolean |  |
| autoReevaluationInterval | DefaultDuration |  |

**Consumed/produced by:**
- Overdue state trigger conditions (all must hold)
- Overdue state evaluation order: first matching configured state wins
- Overdue transition is a no-op on unchanged state name
- Billing-blocking overdue state auto-disables invoicing
- Overdue-driven subscription cancellation policy
- Default overdue escalation tiers
- Overdue re-evaluation notification scheduling
- Unpaid invoice balance aggregation for overdue evaluation
- OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation
- Display-amount rounding for overdue/billing-state formatting
- Payment-delegated child account borrows the parent's billing state for overdue calculation

## DefaultPlanPhase (catalog)
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlanPhase.java:53-101`

| Field | Type | Note |
|---|---|---|
| prettyName | String |  |
| type | PhaseType | TRIAL/DISCOUNT/EVERGREEN/FIXEDTERM |
| duration | DefaultDuration |  |
| fixed | DefaultFixed |  |
| recurring | DefaultRecurring |  |
| usages | DefaultUsage[] |  |
| planName | String | not in XML |
| product | Product | not in XML |

**Consumed/produced by:**
- Plan-phase sequencing via cumulative duration
- Plan-phase duration date arithmetic
- Plan-change alignment determines the effective phase start date
- Plan phase-type composition constraints
- Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase
- First non-zero recurring charge date
- Zero-price plan when no prices defined
- Missing currency price is a hard error
- Negative catalog prices rejected
- Plan-phase price override must specify at least one price component
- Price override cannot introduce a price dimension that didn't exist
- Usage catalog section must declare pricing structures matching its billing mode/type
- Usage tier section must define blocks or limits appropriate to its type
- Catalog usage sections must define pricing structures appropriate to their billing mode
- Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling

## DefaultPrice (catalog)
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPrice.java:41-73`

| Field | Type | Note |
|---|---|---|
| currency | Currency |  |
| value | BigDecimal | throws CurrencyValueNull if requested for a currency with no defined value, line 68-72 |

**Consumed/produced by:**
- Zero-price plan when no prices defined
- Missing currency price is a hard error
- Negative catalog prices rejected
- Currency rounding for stored monetary amounts (half-up)

## DefaultTier (catalog usage tier)
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultTier.java:45-66`

| Field | Type | Note |
|---|---|---|
| limits | DefaultLimit[] |  |
| blocks | DefaultTieredBlock[] |  |
| fixedPrice | DefaultInternationalPrice | whole-tier fixed price |
| recurringPrice | DefaultInternationalPrice | whole-tier recurring price |
| billingMode | BillingMode | not in XML, set at init |
| usageType | UsageType | CAPACITY/CONSUMABLE |
| phase | PlanPhase |  |

**Consumed/produced by:**
- Tiered usage billing — ALL_TIERS policy
- Tiered usage billing — TOP_TIER policy
- Capacity-based usage billing — flat tier price
- TOP_UP usage block requires a minimum top-up credit
- Usage tier section must define blocks or limits appropriate to its type
- Usage catalog section must declare pricing structures matching its billing mode/type
- Usage/product limit compliance check (min/max)

## TagModelDao
**Source:** `util/src/main/java/org/killbill/billing/util/tag/dao/TagModelDao.java:30-51`

| Field | Type | Note |
|---|---|---|
| id | UUID | the tag record id |
| tagDefinitionId | UUID |  |
| objectId | UUID | tagged object (account/bundle/invoice/...) |
| objectType | ObjectType |  |
| isActive | Boolean |  |

**Consumed/produced by:**
- Control tags may only be applied to their designated object types
- AUTO_PAY_OFF blocks only system-triggered payments
- AUTO_PAY_OFF tag cannot be removed without a default payment method on file
- AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing
- OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation
- Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation
- Account parking/unparking implemented via idempotent system tag
- Account email record add is idempotent by record id, not by email address

## TagDefinitionModelDao
**Source:** `util/src/main/java/org/killbill/billing/util/tag/dao/TagDefinitionModelDao.java:31-61`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| name | String |  |
| applicableObjectTypes | String | comma-joined ObjectType list |
| description | String |  |
| isActive | Boolean |  |

**Consumed/produced by:**
- User-defined tag definition name cannot collide with a reserved control tag name
- Tag definition names must be globally unique
- Control tags may only be applied to their designated object types

## InvoicePaymentModelDao
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoicePaymentModelDao.java:36-66`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| type | InvoicePaymentType | ATTEMPT/REFUND/CHARGED_BACK |
| invoiceId | UUID |  |
| paymentId | UUID |  |
| paymentDate | DateTime |  |
| amount | BigDecimal |  |
| currency | Currency |  |
| processedCurrency | Currency |  |
| paymentCookieId | String |  |
| linkedInvoicePaymentId | UUID | links a REFUND/CHARGEBACK to its original ATTEMPT |
| status | InvoicePaymentStatus | INIT/COMMITTED |

**Consumed/produced by:**
- Refund idempotency by transaction cookie id
- Refund amount validation against invoice items
- Refund amount must not exceed original payment and must match specified item adjustments
- Chargeback is idempotent — no partial/duplicate chargebacks
- Chargeback amount/currency resolution fallback
- Chargeback amount bounded by remaining paid amount
- Chargeback reversal resets invoice payment status to INIT
- Invoice balance formula
- API payment cannot exceed invoice balance
- Account balance calculation excludes non-committed and written-off/child-consolidated invoices

## RolledUpUsageModelDao
**Source:** `usage/src/main/java/org/killbill/billing/usage/dao/RolledUpUsageModelDao.java:30-47`

| Field | Type | Note |
|---|---|---|
| id | UUID |  |
| subscriptionId | UUID |  |
| unitType | String |  |
| recordDate | DateTime |  |
| amount | BigDecimal |  |
| trackingId | String | idempotency key for usage recording |

**Consumed/produced by:**
- Usage record idempotency by tracking id
- Usage records cannot be recorded past a subscription's entitlement end date
- Tiered usage billing — ALL_TIERS policy
- Tiered usage billing — TOP_TIER policy
- Capacity-based usage billing — flat tier price
- Usage in-arrear reconciliation bills only the newly-owed delta
- Usage periods already covered by an existing invoice item are skipped to prevent double billing
- Same-day usage items are excluded when re-billing a larger period
- TOP_UP usage block requires a minimum top-up credit

## RolesPermissionsModelDao (RBAC role-permission)
**Source:** `util/src/main/java/org/killbill/billing/util/security/shiro/dao/RolesPermissionsModelDao.java:22-45`

| Field | Type | Note |
|---|---|---|
| recordId | Long |  |
| roleName | String |  |
| permission | String | e.g. wildcard permission string |
| isActive | Boolean |  |
| createdDate | DateTime |  |
| createdBy | String |  |
| updatedDate | DateTime |  |
| updatedBy | String |  |

**Consumed/produced by:**
- RBAC permission check supports AND/OR logic across required permissions
- Role-permission definitions are sanitized and collapsed to group-level wildcards
- Role permission update computes and applies a diff (add new, deactivate removed)
- Security user, role, and role-permission uniqueness on create

## UserRolesModelDao (RBAC user-role assignment)
**Source:** `util/src/main/java/org/killbill/billing/util/security/shiro/dao/UserRolesModelDao.java:22-45`

| Field | Type | Note |
|---|---|---|
| recordId | Long |  |
| username | String |  |
| roleName | String |  |
| isActive | Boolean |  |
| createdDate | DateTime |  |
| createdBy | String |  |
| updatedDate | DateTime |  |
| updatedBy | String |  |

**Consumed/produced by:**
- Security user, role, and role-permission uniqueness on create
- RBAC permission check supports AND/OR logic across required permissions
