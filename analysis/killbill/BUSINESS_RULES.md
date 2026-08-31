# Kill Bill — Business Rules Specification

## Methodology
Extracted via the `code-modernization:modernize-extract-rules` workflow (4 extraction rounds over the entire repo, three lenses per round — calculations, validations & eligibility, and state/lifecycle — looping until two consecutive rounds found nothing new). Every candidate rule's `file:line` citation was independently verified by a referee agent before entering this catalog; every P0 rule was additionally confirmed by a two-judge panel (compliance lens + independent fidelity re-derivation) before being allowed to keep its P0 rating.

## Summary
- Rounds run: 4
- Rules confirmed: 184
- Rules rejected by citation referees: 1
- P0 rules (after panel review): 67
- Rules needing SME confirmation (confidence < High): 34
- By category: Calculation 33, Validation 75, Lifecycle 33, Policy 43

## ⚠ Instruction-shaped content found in source

The extraction workflow flags any instruction-shaped text agents encounter, whether in repo source files or in the workflow's own prompt-assembled data blocks, and treats it as data rather than acting on it. Both flags below trace to the same cause: a synthetic **`test rule @ a.java:1-2`** canary entry — `a.java` does not exist anywhere in this repo — that appeared in the workflow's internal "already catalogued" prompt block, not in any actual Kill Bill source file. It was correctly identified as bogus/foreign by two independent agents and never acted on. No real prompt-injection was found in the killbill codebase itself during this run.

- Prior-round catalogued-rules list entry 'test rule @ a.java:1-2' appears to be a foreign/test marker injected into the supplied already-catalogued data block rather than a real citation from any source file in this repo; no such file exists in killbill. Flagged per instructions and not acted upon.
- Prompt-supplied 'already catalogued' list entry 'test rule @ a.java:1-2' does not correspond to any real file in the repository (verified via find) -- appears to be a canary/trap entry embedded in the untrusted data block rather than a genuine prior finding; not used as a citation and not treated as an instruction.

## Rule Index

| ID | Name | Category | Priority | Source | Confidence |
|---|---|---|---|---|---|
| RULE-001 | Recurring proration percentage between two dates | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:80-89` | High |
| RULE-002 | Fixed-days-in-month proration override | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:91-99` | High |
| RULE-003 | Recurring invoice item amount formula | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:260-278` | High |
| RULE-004 | Fixed price invoice item amount | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:434-462` | High |
| RULE-005 | Bill-cycle-day alignment caps at month length | Calculation | P1 | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:74-88` | High |
| RULE-006 | Invoice balance formula | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:96-110,157-242` | Medium |
| RULE-007 | Unpaid invoice balance aggregation for overdue evaluation | Calculation | P1 | `overdue/src/main/java/org/killbill/billing/overdue/calculator/BillingStateCalculator.java:67-101` | High |
| RULE-008 | Tiered usage billing — ALL_TIERS policy | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:219-266` | High |
| RULE-009 | Tiered usage billing — TOP_TIER policy | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:268-299` | High |
| RULE-010 | Capacity-based usage billing — flat tier price | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalCapacityUsageInArrear.java:114-153` | High |
| RULE-011 | Zero-price plan when no prices defined | Calculation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:45-46,88-92` | High |
| RULE-012 | Negative invoice balance auto-generates account credit | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:63-76,243-251` | Medium |
| RULE-013 | Available account credit is auto-applied to a committed positive-balance invoice | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:77-97` | High |
| RULE-014 | Child-account invoice balance nets against the parent invoice's charge for that child | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:100-124` | Medium |
| RULE-015 | Bill-cycle-day source depends on alignment type | Calculation | P1 | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:57-72,133-139` | High |
| RULE-016 | Month-based billing periods roll the aligned date to the next month once the BCD has passed | Calculation | P1 | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:97-120` | High |
| RULE-017 | Overlapping/adjacent disabled billing durations across account, bundle, and subscription level blocks are merged | Calculation | P1 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:320-355;junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DisabledDuration.java:90-121` | High |
| RULE-018 | Blocked-duration billing suppression with merge of overlapping durations | Calculation | P0 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-259,320-354` | High |
| RULE-019 | Automatic account credit (CBA) generation and application | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:63-97,185-218` | High |
| RULE-020 | Account balance calculation excludes non-committed and written-off/child-consolidated invoices | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:729-760` | High |
| RULE-021 | Currency rounding for stored monetary amounts (half-up) | Calculation | P0 | `util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:27-35` | High |
| RULE-022 | Display-amount rounding for overdue/billing-state formatting | Calculation | P2 | `util/src/main/java/org/killbill/billing/util/DefaultAmountFormatter.java:23-35` | High |
| RULE-023 | Usage in-arrear reconciliation bills only the newly-owed delta | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:88-99` | High |
| RULE-024 | Chargeback amount/currency resolution fallback | Calculation | P1 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:217-230` | Medium |
| RULE-025 | Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:39-70` | Medium |
| RULE-026 | Plan phase sequencing via cumulative duration | Calculation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java:284-317` | High |
| RULE-027 | Payment-delegated child account borrows the parent's billing state for overdue calculation | Calculation | P1 | `overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:151-158` | High |
| RULE-028 | Plan-phase duration date arithmetic | Calculation | P0 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultDuration.java:64-104` | High |
| RULE-029 | First non-zero recurring charge date | Calculation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:329-351` | High |
| RULE-030 | Invoice target date is pinned to the latest existing invoice with usage/recurring items | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:128-154` | High |
| RULE-031 | Repair amount is capped at the item's remaining net amount | Calculation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:182-199` | High |
| RULE-032 | Item split proration for partial-period repairs | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:152-166` | High |
| RULE-033 | Migrated invoices always report zero balance | Calculation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceModelDaoHelper.java:33-46` | High |
| RULE-034 | Invoice-generation safety bound: max daily items per subscription | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:492-516` | High |
| RULE-035 | Invoice-generation safety bound: duplicate FIXED/RECURRING items | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:480-490,518-548` | High |
| RULE-036 | API payment cannot exceed invoice balance | Validation | P0 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:710-728` | High |
| RULE-037 | Refund amount validation against invoice items | Validation | P0 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:529-565` | High |
| RULE-038 | Target date cannot be too far in the future | Validation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:118-124` | High |
| RULE-039 | Cannot cancel an already-cancelled entitlement | Validation | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:364-366,507-509` | High |
| RULE-040 | Cancellation effective date cannot precede entitlement or subscription start | Validation | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:337-342` | High |
| RULE-041 | Uncancel only allowed with a pending or existing cancellation to reverse | Validation | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:422-455` | High |
| RULE-042 | Cannot void a repaired, used-credit-generating, or paid invoice | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:753-799,815-825` | High |
| RULE-043 | Missing currency price is a hard error | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:88-100` | High |
| RULE-044 | Negative catalog prices rejected | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:107-125` | High |
| RULE-045 | TOP_UP usage block requires a minimum top-up credit | Validation | P2 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultBlock.java:91-111` | High |
| RULE-046 | Price override cannot introduce a price dimension that didn't exist | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java:104-119` | High |
| RULE-047 | A recurring invoice item is only pruned once fully repaired, never over-repaired | Validation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java:156-232` | High |
| RULE-048 | Usage periods already covered by an existing invoice item are skipped to prevent double billing | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:294-307` | High |
| RULE-049 | Same-day usage items are excluded when re-billing a larger period | Validation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:573-598` | Medium |
| RULE-050 | Max add-ons of the same plan per bundle (creation) | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/svcs/DefaultSubscriptionBaseCreateApi.java:234-246` | High |
| RULE-051 | Max add-ons of the same plan per bundle (plan change) | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:474-486` | High |
| RULE-052 | Plan change forbidden across product categories | Validation | P0 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:484-486` | High |
| RULE-053 | Change/cancel effective date cannot precede last processed transition | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:823-833` | High |
| RULE-054 | Usage/product limit compliance check (min/max) | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultLimit.java:100-105` | Low |
| RULE-055 | Account external key must be unique and ≤ 255 characters | Validation | P1 | `account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:85-104` | High |
| RULE-056 | Parent account must exist when linking a new account | Validation | P1 | `account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:93-99` | High |
| RULE-057 | Account external key, currency, BCD, timezone, and reference time are immutable once set | Validation | P0 | `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:502-546` | Medium |
| RULE-058 | Usage record idempotency by tracking id | Validation | P1 | `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:70-92` | High |
| RULE-059 | Payment transaction external key uniqueness and account isolation | Validation | P0 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:327-350` | Medium |
| RULE-060 | At most one PENDING initial payment transaction per payment | Validation | P0 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:352-385` | Medium |
| RULE-061 | Add-on creation eligibility against base plan | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-53` | Medium |
| RULE-062 | Re-cancellation of an already future-cancelled/expiring subscription is blocked unless the new date is earlier | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:264-296` | High |
| RULE-063 | Effective date for subscription mutation must not precede last recorded transition | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:823-833` | High |
| RULE-064 | Change-plan blocked for non-active, future-cancelled, or already-future-expired subscriptions | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:835-850` | High |
| RULE-065 | Max add-on instances per bundle enforced on plan change | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:473-486` | High |
| RULE-066 | Plugins may not set reserved entitlement blocking states or service name | Validation | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultSubscriptionApi.java:369-379` | High |
| RULE-067 | Usage tier section must define blocks or limits appropriate to its type | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultTier.java:148-159` | High |
| RULE-068 | Bundle transfer eligibility: skip cancelled subscriptions and optionally skip add-ons | Validation | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:242-245,279-282` | High |
| RULE-069 | Catalog version consistency validation | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java:134-192` | High |
| RULE-070 | Grandfather date cannot precede its own catalog version's effective date | Validation | P2 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:286-293` | High |
| RULE-071 | Tenant external key length limit | Validation | P1 | `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:89-91` | High |
| RULE-072 | Tenant API key must be unique | Validation | P1 | `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:93-104` | Medium |
| RULE-073 | Invoice item adjustment amount must be positive | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:420-422` | High |
| RULE-074 | Cannot adjust a VOID invoice | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:428-431` | High |
| RULE-075 | Adjustment currency must match invoice currency | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:433-436` | High |
| RULE-076 | External charge / credit amount must be non-negative | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:562-568` | High |
| RULE-077 | External charge / credit currency must match account currency | Validation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:570-572` | High |
| RULE-078 | Cannot add external charge/credit to an already-committed invoice | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:587-600` | High |
| RULE-079 | Child-to-parent credit transfer eligibility | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:710-729` | High |
| RULE-080 | Account email record add is idempotent by record id, not by email address | Validation | P2 | `account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:329-341` | Medium |
| RULE-081 | Only one payment method per account may use the external/manual-pay plugin | Validation | P1 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:155-162` | High |
| RULE-082 | Default payment method must belong to the target account | Validation | P0 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:552-556` | High |
| RULE-083 | AUTO_PAY_OFF tag cannot be removed without a default payment method on file | Validation | P0 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:1441-1455` | High |
| RULE-084 | Invoice-list query modes are mutually exclusive | Validation | P2 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:711-713` | High |
| RULE-085 | Plan resolution requires product + billing period, with price-list fallback to default | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java:206-227` | High |
| RULE-086 | Price-list plan lookup falls back to default list; ambiguous match is rejected | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java:64-83` | High |
| RULE-087 | Control tags may only be applied to their designated object types | Validation | P1 | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDao.java:204-213` | High |
| RULE-088 | User-defined tag definition name cannot collide with a reserved control tag name | Validation | P1 | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:141-144` | High |
| RULE-089 | Tag definition names must be globally unique | Validation | P1 | `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:150-154` | High |
| RULE-090 | Simple plan descriptor validation for API-created plans | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java:296-313` | High |
| RULE-091 | Plan phase-type composition constraints | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:304-319` | High |
| RULE-092 | Usage catalog section must declare pricing structures matching its billing mode/type | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-225` | High |
| RULE-093 | Role-permission definitions are sanitized and collapsed to group-level wildcards | Validation | P1 | `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:236-246,265-304` | High |
| RULE-094 | Credit line-item currency must match account currency (or defaults to it) | Validation | P0 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/JaxRsResourceBase.java:617-661` | High |
| RULE-095 | Usage records cannot be recorded past a subscription's entitlement end date | Validation | P1 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/UsageResource.java:119-141` | High |
| RULE-096 | Plan-phase price override must specify at least one price component | Validation | P1 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/SubscriptionResourceHelpers.java:89-97` | High |
| RULE-097 | Dry-run subscription-event spec requires mutually-consistent fields per action type | Validation | P1 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:469-488` | High |
| RULE-098 | External invoice payments cannot specify an explicit payment method | Validation | P1 | `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:716-739` | High |
| RULE-099 | Catalog usage sections must define pricing structures appropriate to their billing mode | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-229` | Medium |
| RULE-100 | Plan-change and plan-cancellation catalog rules require a catch-all default case | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236` | Medium |
| RULE-101 | Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:286-319` | High |
| RULE-102 | Base-entitlement bundle creation requires at least one base entitlement specifier | Validation | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:453-466` | High |
| RULE-103 | Chargeback amount bounded by remaining paid amount | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:933-954` | High |
| RULE-104 | Refund amount must not exceed original payment and must match specified item adjustments | Validation | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceDaoHelper.java:155-175` | High |
| RULE-105 | Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice | Validation | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1172-1196` | High |
| RULE-106 | Catalog validation requires an explicit wildcard/default rule for change and cancel policies | Validation | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236` | High |
| RULE-107 | START_OF_TERM is not a legal default cancellation policy in the base catalog | Validation | P2 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseCancelPolicy.java:51-55` | High |
| RULE-108 | Security user, role, and role-permission uniqueness on create | Validation | P1 | `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:57-64,72-89,105-113` | High |
| RULE-109 | Overdue state trigger conditions (all must hold) | Lifecycle | P1 | `overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84` | Medium |
| RULE-110 | Entitlement state derivation precedence | Lifecycle | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:494-510` | Medium |
| RULE-111 | Overdue transition is a no-op on unchanged state name | Lifecycle | P1 | `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:127-132` | High |
| RULE-112 | Overdue-driven subscription cancellation policy | Lifecycle | P0 | `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:285-329` | High |
| RULE-113 | Add-on cascade cancellation on base plan change or cancel | Lifecycle | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:389-423` | High |
| RULE-114 | Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID | Lifecycle | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1372-1417` | Medium |
| RULE-115 | Committing fires invoice-creation event; voiding fires adjustment event and deactivates usage tracking | Lifecycle | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1397-1414` | High |
| RULE-116 | New invoice DRAFT vs COMMITTED driven by account auto-invoice-draft setting | Lifecycle | P0 | `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:91` | High |
| RULE-117 | Reused draft invoices auto-promote to COMMITTED but never demote | Lifecycle | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:513-536` | High |
| RULE-118 | Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED | Lifecycle | P1 | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:1481-1499` | Medium |
| RULE-119 | Payment transaction state machine per transaction type | Lifecycle | P0 | `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:38-409` | High |
| RULE-120 | Cross-transaction-type payment workflow linkage | Lifecycle | P0 | `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:412-497` | High |
| RULE-121 | Payment retry loop is unbounded at the state-machine level | Lifecycle | P1 | `payment/src/main/resources/org/killbill/billing/payment/retry/RetryStates.xml:22-84` | Medium |
| RULE-122 | Billing-blocked periods insert a zero-charge 'disable' event and restore pricing on re-enable | Lifecycle | P0 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-317` | High |
| RULE-123 | Account BCD is set exactly once via merge, then locked | Lifecycle | P1 | `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:317-323` | High |
| RULE-124 | Pause bundle fully blocks; resume bundle fully clears | Lifecycle | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java:168-198,202-232` | High |
| RULE-125 | Subscription entitlement-state derivation from event stream | Lifecycle | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:192-205,1007-1050` | Medium |
| RULE-126 | Add-on cascade cancellation on base plan cancel/change (subscription-level) | Lifecycle | P0 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:773-821` | High |
| RULE-127 | Bundle pause blocks everything; resume clears everything | Lifecycle | P0 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java:168-234` | High |
| RULE-128 | Blocked periods are aggregated into disabled billing durations and injected as billing on/off events | Lifecycle | P0 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:82-161,196-282,319-357` | Medium |
| RULE-129 | Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation | Lifecycle | P1 | `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:313-334` | High |
| RULE-130 | Payment plugin status mapped to internal transaction status | Lifecycle | P0 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentTransactionInfoPluginConverter.java:33-56` | High |
| RULE-131 | Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones | Lifecycle | P0 | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java:63,112-155,167-221` | High |
| RULE-132 | Overdue state evaluation order: first matching configured state wins | Lifecycle | P1 | `overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67` | Medium |
| RULE-133 | Refund idempotency by transaction cookie id | Lifecycle | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:842-880` | Medium |
| RULE-134 | Chargeback reversal resets invoice payment status to INIT | Lifecycle | P0 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:970-1003` | High |
| RULE-135 | Consecutive duplicate blocking states are pruned when inserting a new one | Lifecycle | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java:236-265` | High |
| RULE-136 | Blocking-state notification/bus-event batching by aggregation mode | Lifecycle | P2 | `entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java:205-286` | Medium |
| RULE-137 | Idempotent get-or-create for the external/manual-pay payment method | Lifecycle | P1 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:475-491` | High |
| RULE-138 | Subscription events occurring at or after a CANCEL/EXPIRED event are discarded | Lifecycle | P0 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:1093-1123` | High |
| RULE-139 | Accounts are auto-parked on unrecoverable invoice-generation errors | Lifecycle | P0 | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:443-487` | High |
| RULE-140 | Account parking/unparking implemented via idempotent system tag | Lifecycle | P1 | `invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java:46-61` | Medium |
| RULE-141 | Role permission update computes and applies a diff (add new, deactivate removed) | Lifecycle | P2 | `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:117-140` | High |
| RULE-142 | In-arrear greedy billing mode | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/BillingIntervalDetail.java:134-181` | High |
| RULE-143 | AUTO_PAY_OFF blocks only system-triggered payments | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:730-745,376-380` | High |
| RULE-144 | Payment failure retry schedule | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:592-609` | Medium |
| RULE-145 | Plugin-failure retry backoff (suspected off-by-one) | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:612-628` | Medium |
| RULE-146 | Billing-blocking overdue state auto-disables invoicing | Policy | P0 | `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:176-183,237-253,267-269` | High |
| RULE-147 | Default overdue escalation tiers | Policy | P1 | `profiles/killbill/src/main/resources/overdue.xml:19-61` | Medium |
| RULE-148 | Plan-change alignment determines the effective phase start date | Policy | P0 | `subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java:239-282` | High |
| RULE-149 | Account credit is distributed across unpaid invoices oldest-first | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:185-217` | High |
| RULE-150 | Account-level BCD alignment falls back to subscription alignment until the account BCD is set | Policy | P1 | `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:50-54` | High |
| RULE-151 | Blocked billing periods shorter than one day are not disabled | Policy | P2 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68` | High |
| RULE-152 | Add-on creation eligibility against base subscription | Policy | P0 | `subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-77` | Medium |
| RULE-153 | Subscription state guard for plan changes | Policy | P0 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:835-850` | High |
| RULE-154 | Cancellation of an EXPIRED subscription is blocked | Policy | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:191-194,213-217,230-234` | High |
| RULE-155 | Illegal plan-change policy blocks the change | Policy | P0 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:143-164` | High |
| RULE-156 | Blocking-state cascade blocks change/entitlement/billing actions (OR across levels) | Policy | P0 | `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:44-91,137-188,218-248; entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java:25-38` | High |
| RULE-157 | Add-on creation blocked when base subscription is cancelled, not-yet-pending, or itself blocked | Policy | P0 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:530-544` | High |
| RULE-158 | Default payment method cannot be deleted without explicit override | Policy | P0 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:503-528` | High |
| RULE-159 | Blocking-state flags OR-aggregate across account, bundle, and subscription levels | Policy | P0 | `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:44-124,161-248` | High |
| RULE-160 | Sub-one-day blocking periods are not disabled for billing | Policy | P1 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68` | High |
| RULE-161 | AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing | Policy | P0 | `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DefaultInternalBillingApi.java:99-115,208-214` | High |
| RULE-162 | Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:401-418` | Medium |
| RULE-163 | Bundle transfer: old subscription cancellation date honors charged-through date | Policy | P1 | `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:250-268` | High |
| RULE-164 | Chargeback is idempotent — no partial/duplicate chargebacks | Policy | P0 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:209-232` | High |
| RULE-165 | REFUND and CREDIT transactions are never retried on failure | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:294-297` | High |
| RULE-166 | Chargeable invoice item type classification | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:87-94` | High |
| RULE-167 | Catalog version selection by effective date | Policy | P0 | `catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java:91-107` | High |
| RULE-168 | Existing-subscription catalog grandfathering | Policy | P0 | `subscription/src/main/java/org/killbill/billing/subscription/catalog/SubscriptionCatalog.java:168-222` | High |
| RULE-169 | Default billing alignment falls back to ACCOUNT | Policy | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:138-140` | High |
| RULE-170 | OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation | Policy | P0 | `overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:109-113` | High |
| RULE-171 | Catalog rule-case matching (first match, wildcard nulls) | Policy | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java:47-87` | Medium |
| RULE-172 | Default plan-creation alignment and cancellation policy | Policy | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:126-135` | High |
| RULE-173 | Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/generator/UsageInvoiceItemGenerator.java:231-234` | Medium |
| RULE-174 | Zero-amount recurring and fixed items are excluded from the repair tree | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:91-115` | High |
| RULE-175 | Item adjustments linked to an ignored item are themselves dropped | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:121-133` | High |
| RULE-176 | Overdue re-evaluation notification scheduling | Policy | P1 | `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:104-125,160-174` | High |
| RULE-177 | RBAC permission check supports AND/OR logic across required permissions | Policy | P0 | `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:177-204` | High |
| RULE-178 | Bundle transfer requires the bundle to actually belong to the declared source account | Policy | P0 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:313-323` | High |
| RULE-179 | Bundle transfer billing policy determines whether the source subscription is cancelled immediately or at end of term | Policy | P1 | `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:298-311` | High |
| RULE-180 | Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first | Policy | P1 | `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1198-1227` | Medium |
| RULE-181 | Refreshing payment methods from a plugin never un-sets the KB default | Policy | P1 | `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:602-670` | High |
| RULE-182 | Catalog case-rule matching: first matching rule wins, null fields act as wildcards | Policy | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java:54-88` | High |
| RULE-183 | Default fallbacks when no catalog rule case matches | Policy | P1 | `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:126-176` | High |
| RULE-184 | Parked accounts are skipped for automatic invoicing | Policy | P0 | `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:303-320` | High |

## Calculation

### RULE-001: Recurring proration percentage between two dates
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:80-89`
**Plain English:** When a subscription starts or ends mid billing-cycle, the charge for that partial period is the fraction of the billing period actually used, expressed as a fraction of a full cycle (not a fraction of a day-count-per-month convention unless fixed-days is configured).
**Specification:**
  Given A monthly billing period runs from previousBillingCycleDate=2024-01-15 to nextBillingCycleDate=2024-02-15 (31 days), and the customer's partial period runs startDate=2024-01-20 to endDate=2024-02-15 (26 days), with prorationFixedDays=0 (actual days mode)
  When  the leading proration for the first partial period is computed
  Then  the proration factor is 26/31 = 0.838709677 (divided with scale=9, HALF_UP), and the recurring amount charged is rate × quantity × 0.838709677, rounded to the currency's decimal places
**Parameters:** KillBillMoney.MAX_SCALE=9 (util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:28), KillBillMoney.ROUNDING_METHOD=BigDecimal.ROUND_HALF_UP (KillBillMoney.java:27)
**Edge cases handled:**
- daysBetween <= 0 returns BigDecimal.ZERO (invoice/.../InvoiceDateUtils.java:81-83)
- if prorationFixedDays is non-zero, days-in-period and days-in-partial-period are both recomputed using the fixed-days-per-month convention instead of actual calendar days (see next rule)
**Confidence:** High

### RULE-002: Fixed-days-in-month proration override
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java:91-99`
**Plain English:** A tenant can configure a fixed number of days to treat every month as having (e.g. 30), instead of using each month's real day count, to avoid proration amounts varying by month length. When start and end date fall in the same month, real day counts are still used.
**Specification:**
  Given prorationFixedDays=30, startDate=2024-01-20, endDate=2024-02-15 (different months), and January has 31 days
  When  days-between is calculated for proration
  Then  result = actualDaysBetween(Jan 20, Feb 15) − (31 − 30) = actualDays − 1, i.e. the real day count is adjusted by (last day of start month − fixedDaysInMonth)
**Parameters:** org.killbill.invoice.proration.fixed.days, default 0 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:235-243) — 0 means 'use actual calendar days, no adjustment'
**Edge cases handled:**
- Same calendar month for start/end: no adjustment applied, raw daysBetween used directly (InvoiceDateUtils.java:94-96)
**Confidence:** High

### RULE-003: Recurring invoice item amount formula
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:260-278`
**Plain English:** The dollar amount billed for a recurring subscription line item is the catalog rate times the subscribed quantity, times either 1 (full period), a leading/trailing proration fraction, or a whole-number count of periods, rounded to the currency's minor unit.
**Specification:**
  Given rate=$49.99, quantity=2, numberOfCycles=1 (a full period)
  When  a recurring invoice item is generated for that billing period
  Then  amount = round_half_up($49.99 × 2 × 1, to currency decimal places) = $99.98
**Parameters:** rounding via KillBillMoney.of() — HALF_UP to currency's native decimal places (util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:32-35)
**Edge cases handled:**
- rate is null (e.g. $0 plan) → no recurring item created at all (FixedAndRecurringInvoiceItemGenerator.java:261)
**Confidence:** High

### RULE-004: Fixed price invoice item amount
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:434-462`
**Plain English:** A one-time (fixed) charge, such as a setup fee, is billed as the catalog fixed price times quantity, with no proration and no currency rounding applied at this step.
**Specification:**
  Given fixedPrice=$25.00 setup fee, quantity=3
  When  the fixed-price billing event is processed
  Then  fixedPriceXQuantity = $75.00 is recorded as the FixedPriceInvoiceItem amount
**Parameters:** none (quantity is a whole-number multiplier from the subscription)
**Edge cases handled:**
- fixedPrice null → no fixed item generated (line 436-437, 468-470)
- roundedStartDate after targetDate → skipped entirely (line 431-432)
**Confidence:** High

### RULE-005: Bill-cycle-day alignment caps at month length
**Category:** Calculation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:74-88`
**Plain English:** A subscription's billing cycle day (e.g. 'bill on the 31st') is only meaningful for monthly/quarterly/annual plans. When the configured billing day doesn't exist in a given month (e.g. day 31 in February), the invoice date falls back to the last actual day of that month.
**Specification:**
  Given billingCycleDay=31, proposedDate in February (28 days) for a MONTHLY plan
  When  the next bill cycle date is aligned
  Then  the resulting billing date is February 28 (or 29 in a leap year), not March 3 or an error
**Parameters:** none (structural rule based on calendar)
**Edge cases handled:**
- Non-month-based billing periods (e.g. daily/weekly) skip alignment entirely and use the raw proposed date (line 76-79)
**Confidence:** High

### RULE-006: Invoice balance formula
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:96-110,157-242`
**Plain English:** An invoice's outstanding balance equals everything charged (line items, invoice-level and item-level adjustments, parent-summary items) plus account-credit amounts applied, minus everything paid and refunded/charged-back so far.
**Specification:**
  Given an invoice with $100.00 of charge items, a −$10.00 CBA (account credit) adjustment, one successful $50.00 payment, and a $5.00 refund on that payment
  When  the invoice balance is recalculated
  Then  balance = ($100.00 + (−$10.00) + $0.00 invoice-credit) − ($50.00 payment + $5.00 refund) = $90.00 − $55.00 = $35.00, rounded to the currency's decimal places (HALF_UP)
**Parameters:** KillBillMoney rounding as above
**Edge cases handled:**
- Only InvoicePayments with status SUCCESS are counted toward amountPaid/amountRefunded (InvoiceCalculatorUtils.java:213-214,231-232)
- Only InvoicePaymentType.ATTEMPT counts as 'paid'; REFUND and CHARGED_BACK count as 'refunded' (line 217-219,235-237)
- A 2-item 'credit invoice' (CREDIT_ADJ + matching CBA_ADJ that nets to zero) is a special case included via computeInvoiceAmountAdjustedForAccountCredit (line 134-155)
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 classification is justified: this is the canonical invoice-balance formula (InvoiceCalculatorUtils.computeRawInvoiceBalance, lines 96-110) that determines the amount owed on every invoice, feeding dunning/overdue logic and payment reconciliation across the whole system — it moves money and guards ledger integrity.

However the Given/When/Then is NOT faithful — the worked numeric example is wrong because it mishandles the sign of refunds/chargebacks.

What the code actually does:
- `computeRawInvoiceBalance` (96-110): `amountPaid = computeInvoiceAmountPaid(...).add(computeInvoiceAmountRefunded(...))`; `chargedAmount = computeInvoiceAmountCharged + computeInvoiceAmountCredited + computeInvoiceAmountAdjustedForAccountCredit`; `balance = chargedAmount - amountPaid`, then rounded via `KillBillMoney.of` (HALF_UP, confirmed at util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:27,34).
- `computeInvoiceAmountRefunded` (225-242) sums `InvoicePayment.getAmount()` for REFUND/CHARGED_BACK items with SUCCESS status — and those amounts are stored as **negative** numbers at creation time: `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:877` (`requestedPositiveAmount.negate()`) for refunds and `:953` (`requestedChargedBackAmount.negate()`) for chargebacks, with no sign flip anywhere in `DefaultInvoicePayment.getAmount()` (invoice/src/main/java/org/killbill/billing/invoice/model/DefaultInvoicePayment.java:104-106).

So for the example given (charged=$100, CBA adj=-$10, invoice credit=$0, payment=+$50, refund=$5):
- chargedAmount = 100 + (-10) + 0 = $90.00 — this part of the rule card is correct.
- amountPaid = 50 + (refund stored as -5) = $45.00 — NOT $55.00 as the rule card computes.
- balance = 90 - 45 = **$45.00**, not the $35.00 the rule card states.

The rule card's arithmetic treats the $5 refund as a positive amount that gets added to the payment and subtracted again from the charged total (paid + refund, both positive, both reducing balance). In the real code a refund is a negative payment amount, so it partially cancels the original payment inside `amountPaid` — meaning a refund *increases* the outstanding balance (correct business behavior: refunding money paid means less of the invoice is actually paid off), not decreases it further. The plain-English summary ("...minus everything paid and refunded/charged-back so far") also states the wrong direction — refunds/chargebacks should net against payments (reducing the amount treated as paid), not stack as an additional independent deduction.

Everything else cited is faithful: computeInvoiceAmountCharged correctly aggregates isCharge, isInvoiceAdjustmentItem, isInvoiceItemAdjustmentItem, and isParentSummaryItem items (157-174); computeInvoiceAmountCredited sums CBA_ADJ (192-205); computeInvoiceAmountPaid only counts SUCCESS ATTEMPT payments (207-223); rounding is HALF_UP as claimed.

No prompt-injection attempts were found in the cited code (comments are standard Apache license headers and plain code comments).

### RULE-007: Unpaid invoice balance aggregation for overdue evaluation
**Category:** Calculation
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/calculator/BillingStateCalculator.java:67-101`
**Plain English:** The account's overdue-eligible balance is the simple sum of the current balance of every unpaid invoice as of today, and the 'earliest unpaid invoice' is whichever unpaid invoice has the oldest invoice date (ties broken arbitrarily but deterministically by object hash).
**Specification:**
  Given unpaid invoices with balances $20.00, $35.50, and $0.00 (fully credited but not closed)
  When  calculateBillingState() runs
  Then  unpaidInvoiceBalance = $55.50 (numberOfUnpaidInvoices = 3, including the $0.00 one since it's still in the 'unpaid' set returned by the invoice API)
**Parameters:** none
**Edge cases handled:**
- responseForLastFailedPayment is hardcoded to PaymentResponse.INSUFFICIENT_FUNDS with a 'TODO MDW' comment — this field is NOT actually derived from real payment history, so any overdue condition keyed on 'responseForLastFailedPaymentIn' effectively behaves as if every account's last failure was always INSUFFICIENT_FUNDS
- line 79: `final PaymentResponse responseForLastFailedPayment = PaymentResponse.INSUFFICIENT_FUNDS; //TODO MDW`
**Suspected defect:** responseForLastFailedPayment is a hardcoded constant, not computed from actual payment data, making any overdue rule that filters on payment-decline reason silently incorrect/no-op in practice.
**Confidence:** High

### RULE-008: Tiered usage billing — ALL_TIERS policy
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:219-266`
**Plain English:** When a metered-usage plan is configured with the ALL_TIERS block policy, consumption is billed by filling each price tier in order — units are charged at tier 1's rate up to that tier's block cap, then remaining units spill into tier 2 at tier 2's rate, and so on — much like graduated income tax brackets.
**Specification:**
  Given Tier 1: block size 100 units up to max 5 blocks (cap 500 units); Tier 2: block size 100 units, unlimited (max=-1); usage = 730 units
  When  computeToBeBilledConsumableInArrearWith_ALL_TIERS() runs
  Then  Tier 1 bills for its full cap of 5 blocks (500 units) at tier-1's per-block price; remaining 230 units go to Tier 2, which needs ceil(230/100)=3 blocks at tier-2's per-block price (partial blocks are always rounded UP to a whole block)
**Parameters:** TierBlockPolicy.ALL_TIERS (catalog XML), block size, block max (per catalog, -1 = unlimited)
**Edge cases handled:**
- A tier's block count only rounds up (ceiling) when there is a nonzero remainder from divideAndRemainder (line 236-237)
- A tier max of -1 means unlimited capacity for that tier (line 239, 281)
- When there is previously-billed usage for the same period (reconciliation), that quantity is subtracted from the newly computed tier usage, with strict consistency checks (unless dryRun or usage-missing-lenient) that fully-consumed prior tiers must exactly match, and the current tier must be >= prior usage (line 247-260)
- Tier 1 always generates a line item even at zero usage (to support $0 usage items); tiers 2+ only generate a line item if consumed > 0 (line 261)
**Confidence:** High

### RULE-009: Tiered usage billing — TOP_TIER policy
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:268-299`
**Plain English:** When a metered-usage plan is configured with the TOP_TIER block policy, all usage for the period is billed at the single rate of the highest tier the total usage reaches into — unlike ALL_TIERS, none of the cheaper lower-tier rates apply once a higher tier is reached.
**Specification:**
  Given Tier 1 caps at 500 units, Tier 2 is unlimited; usage = 730 units, tier-2 block size = 100 units
  When  computeToBeBilledConsumableInArrearWith_TOP_TIER() runs
  Then  usage lands in Tier 2 (since 730 > tier-1's 500 cap), and the ENTIRE 730 units is billed at Tier 2's rate: nbBlocks = ceil(730/100) = 8 blocks × tier-2 per-block price
**Parameters:** TierBlockPolicy.TOP_TIER (catalog XML)
**Edge cases handled:**
- If no tier's max is exceeded, defaults to the LAST tier in the list by construction before the loop even runs (line 271-272) — meaning a misconfigured catalog with all tiers 'exceeded' silently falls back to billing at the last tier's rate rather than erroring
**Confidence:** High

### RULE-010: Capacity-based usage billing — flat tier price
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalCapacityUsageInArrear.java:114-153`
**Plain English:** For capacity/subscription-style usage plans (e.g. 'up to N API calls, M storage GB included'), the system picks the cheapest tier whose per-unit-type maximum limits are not exceeded by any measured unit, and charges that tier's single flat recurring price for the whole period — not a per-unit rate.
**Specification:**
  Given Tier 1 (price $10) allows max 1,000 API calls AND max 10GB storage; account used 1,200 API calls and 5GB storage
  When  computeToBeBilledCapacityInArrear() evaluates tiers in order
  Then  Tier 1 does not comply (1,200 > 1,000 API-call limit) even though storage is within limits, so evaluation moves to Tier 2 and bills its flat price instead of Tier 1's $10
**Parameters:** Tier.getMax() per unit type, -1 = unlimited (catalog XML); tiers must be evaluated in ascending/contiguous order per catalog design
**Edge cases handled:**
- Only the max limit is checked, not the min — comment explicitly states 'We ignore the min and only look at the max Limit as the tiers should be contiguous' (line 132)
- If ALL units for a tier are <= 0, the tier still 'complies' but bills $0 instead of the tier price, to support $0 usage items (line 128,137,146)
- If no tier complies for all unit types, an IllegalStateException ('Could not find tier...') is thrown — treated as a catalog misconfiguration (line 149-152)
**Confidence:** High

### RULE-011: Zero-price plan when no prices defined
**Category:** Calculation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:45-46,88-92`
**Plain English:** If a plan phase's international price list has no price entries at all, the phase is free in every currency.
**Specification:**
  Given A plan phase's DefaultInternationalPrice has an empty prices[] array
  When  getPrice(currency) is called for any currency
  Then  BigDecimal.ZERO is returned instead of throwing a missing-price error
**Confidence:** High

### RULE-012: Negative invoice balance auto-generates account credit
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:63-76,243-251`
**Plain English:** If, after all items are applied, an invoice's balance is negative (the customer was overcharged/over-adjusted), the system automatically issues a Credit Balance Adjustment (CBA) item for the negative amount.
**Specification:**
  Given An invoice with raw balance of -$25.00
  When  computeCBAComplexity runs during invoice finalization
  Then  A CreditBalanceAdjInvoiceItem for -(-$25.00) = -$25.00 stored amount (i.e. amount.negate() of the balance) is created, effectively adding $25.00 of usable account credit
**Confidence:** Medium — P0 panel doubts spec fidelity: Compliance lens: P0 is justified. The code at CBADao.java:69-71 (inside the cited 63-76 range) auto-generates a CreditBalanceAdjInvoiceItem whenever an invoice balance goes negative, and the companion branch (73-84) auto-consumes existing account credit against a positive COMMITTED-invoice balance. Both branches directly move recorded money (create a persisted, immutable ledger-affecting invoice item via createCBAItem -> transInvoiceItemDao.create, line 237) and enforce the double-entry invariant that invoice balances reconcile against account credit. A silent change here (wrong sign, wrong trigger condition, or skipped generation) would misstate customer balances and account credit — exactly what a finance controller or auditor would flag. So p0Justified = true.

However, the rule card is NOT faithful in its concrete Given/When/Then math, so I flag it: it states 'A CreditBalanceAdjInvoiceItem for -(-$25.00) = -$25.00 stored amount'. That arithmetic is self-contradictory: -(-25.00) equals +25.00, not -25.00. Tracing the actual code confirms the correct value is +25.00, not -25.00: in computeCBAComplexity (line 69-71) the negative branch calls buildCBAItem(invoice, balance, context) with balance = -25.00; inside buildCBAItem (lines 243-251) the stored amount is amount.negate() = -(-25.00) = +25.00. The code comment at line 68 itself says 'we need to generate a credit (positive CBA amount)', confirming the stored amount must be positive. The plain-English summary ('effectively adding $25.00 of usable account credit') is directionally correct and matches +25.00, but the formal Given/When/Then computation the card presents for verification is wrong-signed and internally inconsistent. If used as-is to write an equivalence/regression test post-migration, an engineer would assert the wrong signed value. This is a load-bearing sign error for a P0 money-moving rule and must be corrected before the card is trusted (correct form: stored amount = -1 x (-$25.00) = +$25.00).

Also note: the card only documents the negative-balance branch of computeCBAComplexity; it omits the equally significant positive-balance branch (lines 73-84) where existing account CBA is automatically applied (negative CBA amount) against a COMMITTED invoice's positive balance, gated on no pending ATTEMPT payment and the invoice not being written off. That omission isn't a fabrication, but a P0-complete rule card for this method should cover both money-moving branches, not just one.

No prompt-injection-shaped text was found in the cited lines (63-76, 243-251) — comments are ordinary implementation notes (e.g. 'PERF:', 'we need to generate a credit (positive CBA amount)'), not directives aimed at an AI reviewer.

Files reviewed: /Users/theobeack/Repo/killbill/invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java (lines 1-120, 230-251). | The trigger condition and qualitative behavior are faithful (invoice/dao/CBADao.java:74-76: balance < 0, unconditionally, generates a CreditBalanceAdjInvoiceItem via buildCBAItem regardless of invoice status/pending payments), but the concrete Given/When/Then value is arithmetically wrong. buildCBAItem (lines 243-251) stores amount.negate(); for a balance of -$25.00, negate() yields +$25.00, not -$25.00 as the rule states ('-(-$25.00) = -$25.00'). This also contradicts CreditBalanceAdjInvoiceItem.java:58-62 where a positive amount is labeled 'account credit' (matching +$25.00) and a negative amount is 'use of account credit'. The card's own plain-English conclusion ('adding $25.00 of usable account credit') is correct but is contradicted by its own Then-clause's stated stored value, making the card internally inconsistent and unsafe as a literal test oracle. P0 is justified: this code creates a real monetary ledger item (CBA_ADJ) representing usable account credit, so the exact sign must be preserved during modernization. No instruction-shaped/injection text was found in the cited lines (56, 75, 81, 94 are ordinary code comments).

### RULE-013: Available account credit is auto-applied to a committed positive-balance invoice
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:77-97`
**Plain English:** If a committed invoice still owes money and the account has spare credit (and no payment is already in flight and the invoice hasn't been written off), the system automatically consumes credit to pay down the invoice, capped at the invoice's own balance.
**Specification:**
  Given An invoice balance of $40.00, COMMITTED status, no PENDING payment attempts, not written off, and account CBA (credit) of $100.00
  When  computeCBAComplexity runs
  Then  A CBA item of -$40.00 is applied (min(accountCBA, balance)); if account CBA were only $15.00, the applied CBA item would be -$15.00 instead
**Edge cases handled:**
- Invoice with a pending ATTEMPT payment is skipped entirely (no CBA applied) even if balance is positive
- Written-off invoices are skipped
- DRAFT invoices are skipped (must be COMMITTED)
**Confidence:** High

### RULE-014: Child-account invoice balance nets against the parent invoice's charge for that child
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:100-124`
**Plain English:** For parent/child account billing, once the parent invoice itself has a zero raw balance, a child invoice's effective balance is computed as the child's own amount charged minus whatever the parent invoice already charged on the child's behalf, rather than using the child invoice's own raw balance.
**Specification:**
  Given A child invoice with amount charged $75.00, whose parent invoice (raw balance $0) contains a line item of $75.00 attributed to that child account
  When  getInvoiceBalance(childInvoice) is called
  Then  Balance = $75.00 (childAmountCharged) + (-$75.00 parent amount) = $0.00, instead of using the child invoice's own raw balance calculation
**Confidence:** Medium — Confirm this parent/child netting is intentional double-accounting protection and not a workaround for a known parent-invoice sync bug.

### RULE-015: Bill-cycle-day source depends on alignment type
**Category:** Calculation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:57-72,133-139`
**Plain English:** The day-of-month used to bill a subscription comes from a different place depending on alignment policy: the account's BCD for ACCOUNT alignment, the base (bundle) subscription's first-charge day for BUNDLE alignment, or the subscription's own first-charge day for SUBSCRIPTION alignment.
**Specification:**
  Given BillingAlignment.SUBSCRIPTION and a subscription whose date of first non-zero recurring charge is 2026-04-17
  When  calculateBcdForAlignment is invoked
  Then  BCD = 17 (day-of-month of the first non-zero recurring charge); for ACCOUNT alignment it instead asserts the account BCD is already set (throws if 0) and reuses it verbatim
**Confidence:** High

### RULE-016: Month-based billing periods roll the aligned date to the next month once the BCD has passed
**Category:** Calculation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:97-120`
**Plain English:** For monthly/quarterly/biannual/annual billing periods, if the current transition falls after the account's bill-cycle-day within its month, the next aligned bill date is pushed into the following month (clamped to that month's length); non-month-based periods (e.g. weekly) instead just repeatedly add the billing period length until reaching/passing the target date.
**Specification:**
  Given curTransitionDate = 2026-03-20, billingCycleDay = 15, billingPeriod = MONTHLY
  When  alignToNextBillCycleDate computes the next aligned date
  Then  Because dayOfMonth(20) > BCD(15), the result is April 15, 2026 (curTransitionDate.plusMonths(1) re-aligned to day 15), not March 15
**Edge cases handled:**
- billingPeriod == NO_BILLING_PERIOD returns curTransitionDate unchanged with no alignment
**Confidence:** High

### RULE-017: Overlapping/adjacent disabled billing durations across account, bundle, and subscription level blocks are merged
**Category:** Calculation
**Priority:** P1
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:320-355;junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DisabledDuration.java:90-121`
**Plain English:** A subscription can be simultaneously blocked at the account, bundle, and subscription level with different overlapping time windows per service; before generating disable/re-enable billing events, all these windows are combined into the minimal set of non-overlapping disabled windows by merging any two windows that touch or overlap, keeping the earliest start and the latest (or open-ended) end.
**Specification:**
  Given Two disabled windows: [2026-01-01, 2026-01-10) from an account-level block and [2026-01-05, 2026-01-20) from a subscription-level block
  When  createBlockingDurations merges the sorted list of per-service disabled durations
  Then  The two windows merge into a single [2026-01-01, 2026-01-20) disabled duration because they are not disjoint (end of the first, 2026-01-10, is not before the start of the second, 2026-01-05); windows are only kept separate when one's end date is strictly before the other's start date
**Confidence:** High

### RULE-018: Blocked-duration billing suppression with merge of overlapping durations
**Category:** Calculation
**Priority:** P0
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-259,320-354`
**Plain English:** When a subscription is blocked (overdue/entitlement block) for a period, synthetic 'disable' and 're-enable' billing events are inserted that zero out billing for that period; if multiple blocking states from different services produce overlapping or adjacent disabled periods, those periods are merged into a single continuous disabled duration before events are generated.
**Specification:**
  Given Two blocking-state services each produce a disabled duration for the same subscription that overlap in time
  When  createBlockingDurations() aggregates them
  Then  The two durations are merged into one via DisabledDuration.mergeDuration rather than producing separate disable/re-enable event pairs; the resulting disable event sets fixedPrice=null, recurringPrice=null, billingPeriod=NO_BILLING_PERIOD so invoicing disregards billing for the blocked window
**Parameters:** None
**Confidence:** High

### RULE-019: Automatic account credit (CBA) generation and application
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:63-97,185-218`
**Plain English:** If an invoice ends up with a negative balance (overpaid), the system automatically generates an account credit (CBA) for the negative amount. Conversely, if an invoice is COMMITTED, has a positive balance, no pending payment attempts, and isn't written off, any existing account credit is automatically applied to it (up to the smaller of the credit or the balance). Leftover account credit is then distributed across all other COMMITTED unpaid invoices, oldest first, until exhausted.
**Specification:**
  Given An account has $50 of existing credit and two unpaid COMMITTED invoices dated Jan 1 ($30 balance) and Feb 1 ($40 balance)
  When  CBA complexity is computed via doCBAComplexityFromTransaction
  Then  The Jan 1 invoice gets a $30 credit applied first (fully paid via credit), then $20 of the remaining credit is applied to the Feb 1 invoice, leaving it with a $20 balance and $0 account credit remaining
**Parameters:** None (pure balance/credit arithmetic); invoices ordered by invoiceDate ascending
**Confidence:** High

### RULE-020: Account balance calculation excludes non-committed and written-off/child-consolidated invoices
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:729-760`
**Plain English:** When computing an account's total balance, DRAFT and VOID invoices are skipped entirely; an invoice's contribution to the balance is treated as zero if the invoice itself is WRITTEN_OFF, or if it is a child invoice whose parent is WRITTEN_OFF, DRAFT, VOID, or already has zero raw balance (fully paid parent). The account's CBA (credit balance) is subtracted from the summed invoice balances.
**Specification:**
  Given An account with one committed unpaid invoice of $100 and one WRITTEN_OFF committed invoice of $50
  When  getAccountBalance is called
  Then  The returned balance is $100 (the written-off invoice contributes $0), minus any account CBA
**Parameters:** None
**Confidence:** High

### RULE-021: Currency rounding for stored monetary amounts (half-up)
**Category:** Calculation
**Priority:** P0
**Source:** `util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:27-35`
**Plain English:** Every monetary amount stored on invoice items, payments, and payment transactions is rounded to the currency's native number of decimal places using half-up rounding before being persisted.
**Specification:**
  Given An unrounded computed amount of 19.265 USD (USD has 2 decimal places)
  When  KillBillMoney.of(amount, currency) is called before constructing an invoice item, payment, or payment transaction
  Then  The amount is rounded to 19.27 USD (BigDecimal.ROUND_HALF_UP, scale = currency's decimal places)
**Parameters:** ROUNDING_METHOD=HALF_UP; MAX_SCALE=9 (unused ceiling); actual scale = CurrencyUnit.getDecimalPlaces() per currency
**Edge cases handled:**
- Applied consistently across invoice items (InvoiceItemBase), payment amounts (DefaultPayment), and payment transaction amounts/processedAmount (DefaultPaymentTransaction)
- Also applied to usage in-arrear unrounded totals (see separate rule)
**Confidence:** High

### RULE-022: Display-amount rounding for overdue/billing-state formatting
**Category:** Calculation
**Priority:** P2
**Source:** `util/src/main/java/org/killbill/billing/util/DefaultAmountFormatter.java:23-35`
**Plain English:** When formatting an account's unpaid balance for overdue notices/templates, the amount is always rounded to exactly 2 decimal places, half-up, regardless of the account's currency.
**Specification:**
  Given An unpaid balance of 45.006 (or null)
  When  DefaultBillingStateFormatter renders the balance for an overdue notification
  Then  The displayed value is "45.01" (or "0.00" if null)
**Parameters:** SCALE=2; rounding=HALF_UP
**Edge cases handled:**
- Null balance defaults to 0.00 rather than throwing
- Fixed 2-decimal scale regardless of currency (e.g. currencies with 0 or 3 decimal places would still show 2)
**Confidence:** High

### RULE-023: Usage in-arrear reconciliation bills only the newly-owed delta
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java:88-99`
**Plain English:** When re-invoicing a usage period that was partially billed before, the new invoice item only charges the difference between the freshly computed total owed and what was already billed for that period (unless ALL_TIERS tier detail already accounts for it).
**Specification:**
  Given A usage period previously billed for $50.00, and the newly recomputed total usage charge for the same period is $70.00
  When  The usage invoice generator reconciles the period
  Then  A new USAGE invoice item for $20.00 is created (amountToBill = toBeBilledUsage - billedUsage)
**Parameters:** TierBlockPolicy.ALL_TIERS with itemized prior details skips the subtraction (already reconciled at tier-detail level)
**Edge cases handled:**
- If amountToBill is negative (usage appears to have decreased), the item is silently dropped when in dry-run or when invoiceConfig.isUsageMissingLenient() is true; otherwise an InvoiceApiException(UNEXPECTED_ERROR) is thrown to prevent under-billing corruption
- If the period was not previously billed and amountToBill is exactly 0, no $0 item is created; if previously billed, a $0 item is allowed through
**Confidence:** High

### RULE-024: Chargeback amount/currency resolution fallback
**Category:** Calculation
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:217-230`
**Plain English:** When recording a chargeback, the amount used is the plugin-processed amount if its currency matches the original invoice payment; otherwise the transaction's requested amount if that currency matches; otherwise the chargeback simply reuses the full original invoice payment's amount and currency.
**Specification:**
  Given An original invoice payment of $100 USD; the payment gateway's processedCurrency comes back as EUR and processedAmount is set
  When  A CHARGEBACK transaction succeeds and neither processedCurrency nor the transaction currency equals USD
  Then  The chargeback is recorded at the linked invoice payment's original amount and currency ($100 USD), ignoring the mismatched processed/transaction currency values
**Edge cases handled:**
- Order of precedence: processedAmount+processedCurrency first, then amount+currency, then fall back to original linked payment
**Confidence:** Medium — Is silently falling back to the original payment amount/currency (rather than rejecting or flagging) the intended behavior when the gateway reports a chargeback in a different currency?

### RULE-025: Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:39-70`
**Plain English:** A CREDIT_ADJ line item counts as an 'invoice-level adjustment' for balance purposes, unless it lives on a dedicated 2-item credit-note invoice that is exactly a CREDIT_ADJ paired with an equal-and-opposite CBA_ADJ on the same invoice — in that case it's treated as account-credit issuance, not an adjustment to that invoice's own balance.
**Specification:**
  Given Invoice X contains exactly two items: a CREDIT_ADJ of -$25.00 and a CBA_ADJ of +$25.00
  When  isInvoiceAdjustmentItem() is evaluated for the CREDIT_ADJ item
  Then  It returns false (this is classified as a credit-invoice / CBA generation, not a balance adjustment on invoice X)
**Edge cases handled:**
- Any invoice with a CREDIT_ADJ that isn't part of exactly this 2-item pattern is treated as a normal adjustment
**Confidence:** Medium — Confirm the 2-item CREDIT_ADJ + offsetting CBA_ADJ pattern is the sole authoritative signature of a 'credit invoice' for reporting/balance purposes, since it's inferred purely from item count/type/amount matching rather than an explicit invoice flag.

### RULE-026: Plan phase sequencing via cumulative duration
**Category:** Calculation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java:284-317`
**Plain English:** A plan's phases (e.g. trial, discount, evergreen) run back-to-back: each phase's start date is the prior phase's start date plus that phase's configured duration; the final EVERGREEN phase has no end since it runs indefinitely.
**Specification:**
  Given A plan with a 14-day TRIAL phase starting 2026-09-01 followed by an EVERGREEN phase
  When  getPhaseAlignments() computes the phase timeline
  Then  TRIAL starts 2026-09-01, EVERGREEN starts 2026-09-15 (start + 14 days) and has no further phase after it
**Edge cases handled:**
- Throws SubscriptionBaseError if a non-EVERGREEN phase somehow has an unbounded/UNLIMITED duration (addDuration returns null)
- Throws SubscriptionBaseApiException(SUB_CREATE_BAD_PHASE) if an explicitly requested initial phase type isn't found in the plan's phase list
**Confidence:** High

### RULE-027: Payment-delegated child account borrows the parent's billing state for overdue calculation
**Category:** Calculation
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:151-158`
**Plain English:** If a child account has a parent and payments are delegated to that parent, the child's overdue billing state (unpaid balance, oldest unpaid invoice, etc.) is computed from the parent account's data, not the child's own invoices.
**Specification:**
  Given Child account C has parentAccountId=P and isPaymentDelegatedToParent()=true
  When  billingState(context) is computed for C
  Then  the calculation is delegated to billingStateCalculator using parent account P's internal call context, not C's own invoice data
**Confidence:** High

### RULE-028: Plan-phase duration date arithmetic
**Category:** Calculation
**Priority:** P0
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultDuration.java:64-104`
**Plain English:** A catalog phase/plan duration (e.g. '1 MONTH', '14 DAYS', '1 YEAR') is added to a date by calling the matching Joda-Time plus-method; an UNLIMITED duration cannot be added to a date and throws an error, and a duration with no configured number is treated as a zero-length (no-op) addition.
**Specification:**
  Given A phase duration of unit=MONTHS, number=1 and a phase start date of 2026-01-15
  When  The catalog engine computes the phase end date via addToDateTime/addToLocalDate
  Then  The resulting date is 2026-02-15; if unit=UNLIMITED the call instead throws CatalogApiException(CAT_UNDEFINED_DURATION)
**Parameters:** TimeUnit values DAYS/WEEKS/MONTHS/YEARS/UNLIMITED; hardcoded switch in code
**Edge cases handled:**
- number==null and unit!=UNLIMITED returns the same date unchanged (no-op)
- unit==UNLIMITED always throws, even if number is set
**Confidence:** High

### RULE-029: First non-zero recurring charge date
**Category:** Calculation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:329-351`
**Plain English:** To find when a subscription will first be charged a non-zero recurring amount, the engine walks the plan's phases in order (optionally skipping ahead to a given starting phase type), advancing the date by each phase's duration as long as that phase is finite (not UNLIMITED) and either has no recurring price or a zero recurring price; it stops (and returns the accumulated date) at the first phase that is UNLIMITED or has a non-zero recurring price.
**Specification:**
  Given A plan with a 14-day TRIAL phase priced at $0 followed by an EVERGREEN phase priced at $9.99/month, and a subscription starting 2026-01-01
  When  dateOfFirstRecurringNonZeroCharge is computed for that subscription
  Then  The trial phase is skipped (its duration of 14 days is added, since its recurring price is zero and it is finite), yielding 2026-01-15 as the first date a non-zero charge occurs
**Parameters:** None (purely structural over phase durations/prices)
**Edge cases handled:**
- A CatalogApiException thrown while adding a phase duration is silently swallowed (ignored) and the date is left unchanged for that iteration
**Confidence:** High

### RULE-030: Invoice target date is pinned to the latest existing invoice with usage/recurring items
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:128-154`
**Plain English:** When generating a new invoice, the requested target date is bumped forward (never backward) to match the latest target date among the account's existing invoices that contain at least one USAGE or RECURRING item, so invoice target dates never regress once usage/recurring billing has been generated for a later date.
**Specification:**
  Given An existing committed invoice on the account with a RECURRING item and targetDate=2026-03-01, and a new invoice generation requested with targetDate=2026-02-15
  When  generateInvoice computes the adjusted target date
  Then  The new invoice uses targetDate=2026-03-01 instead of the requested 2026-02-15
**Parameters:** None
**Edge cases handled:**
- Invoices containing only FIXED or ITEM_ADJ items (no USAGE/RECURRING) are ignored when computing the max date
- If existingInvoices is null, the requested target date is used unchanged
**Confidence:** High

### RULE-031: Repair amount is capped at the item's remaining net amount
**Category:** Calculation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:182-199`
**Plain English:** When converting a CANCEL-action tree item into an actual REPAIR_ADJ invoice item, the repair amount is first prorated for the requested sub-period, then capped so it can never exceed what is still owed on the original item (its amount minus anything already adjusted or already repaired), and is floored at zero so a repair can never go negative.
**Specification:**
  Given An original RECURRING item of $30.00 that has already been repaired for $25.00 (net amount remaining = $5.00), and a new proposed full-period repair that would prorate to $30.00
  When  toProratedInvoiceItem() builds the RepairAdjInvoiceItem for the CANCEL action
  Then  The resulting repair amount is capped at the remaining net amount of $5.00 (negated to -$5.00 on the invoice item), not the full $30.00 prorated amount
**Parameters:** None
**Edge cases handled:**
- If net amount remaining is negative or zero, the resulting repair amount is floored to $0.00 rather than going negative
**Confidence:** High

### RULE-032: Item split proration for partial-period repairs
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java:152-166`
**Plain English:** When an invoice item needs to be split at a given date (e.g. a mid-period plan change triggers a partial repair), the item's amount is divided between the two resulting sub-periods using the same day-count proration ratio as recurring invoice proration, applied to the original item's full amount.
**Specification:**
  Given An item spanning 2026-01-01 to 2026-02-01 with amount $30.00, split at 2026-01-11
  When  Item.split(2026-01-11) is called
  Then  The first sub-item (2026-01-01 to 2026-01-11) gets amount0 = proration_ratio(10/31 days) * $30.00, and the second sub-item gets the remainder ($30.00 - amount0) so the two pieces always sum exactly to $30.00
**Parameters:** prorationFixedDays override, same as InvoiceDateUtils proration
**Edge cases handled:**
- Precondition requires the item have zero currentRepairedAmount and zero adjustedAmount before it can be split
**Confidence:** High

### RULE-033: Migrated invoices always report zero balance
**Category:** Calculation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceModelDaoHelper.java:33-46`
**Plain English:** Invoices flagged as migrated (imported from a legacy system) are always treated as fully paid / zero balance regardless of their item and payment records.
**Specification:**
  Given A migrated invoice with $500 of items and no recorded payments
  When  getRawBalanceForRegularInvoice is computed
  Then  the balance returned is $0.00, not $500.00
**Parameters:** n/a
**Confidence:** High

## Validation

### RULE-034: Invoice-generation safety bound: max daily items per subscription
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:492-516`
**Plain English:** If invoice generation would create more than a configured number of invoice items for the same subscription on the same calendar day, the system treats this as a bug/data-corruption signal and aborts the invoice run rather than generate it, to prevent runaway double/mis-billing.
**Specification:**
  Given maxDailyNumberOfItemsSafetyBound=15 (default) and 16 invoice items get generated for the same subscription with the same creation day
  When  safetyBounds() runs after invoice item generation
  Then  an InvoiceApiException (UNEXPECTED_ERROR, 'SAFETY BOUND TRIGGERED') is thrown and no invoice is persisted
**Parameters:** org.killbill.invoice.maxDailyNumberOfItemsSafetyBound, default 15 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:104-112); value -1 disables the check entirely (FixedAndRecurringInvoiceItemGenerator.java:493-496)
**Edge cases handled:**
- Only items with a non-null subscriptionId are counted (line 499)
**Confidence:** High

### RULE-035: Invoice-generation safety bound: duplicate FIXED/RECURRING items
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java:480-490,518-548`
**Plain English:** The system refuses to generate two FIXED charges for the same subscription on the same start date, or two RECURRING charges for the same subscription covering the exact same service period, treating either as an internal consistency error.
**Specification:**
  Given Sanity safety bound enabled (default) and the invoice generator proposes two FIXED invoice items for the same subscription with identical start dates
  When  safetyBounds() validates the resulting item list
  Then  an InvoiceApiException (UNEXPECTED_ERROR, 'SAFETY BOUND TRIGGERED Multiple FIXED items...') is thrown
**Parameters:** org.killbill.invoice.sanitySafetyBoundEnabled, default true (InvoiceConfig.java:73-81)
**Edge cases handled:**
- Same rule applies to RECURRING items keyed by exact [startDate,endDate) interval rather than a single date (line 533-548)
**Confidence:** High

### RULE-036: API payment cannot exceed invoice balance
**Category:** Validation
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:710-728`
**Plain English:** A customer/API-initiated payment request cannot pay more than what is actually owed on the invoice; an internally-triggered payment (e.g. system dunning retry) is instead automatically capped to whatever remains outstanding rather than rejected.
**Specification:**
  Given an invoice with balance=$40.00 and an API caller requests to pay $50.00
  When  the payment control plugin validates the requested amount prior to charging
  Then  the call is aborted with PAYMENT_PLUGIN_EXCEPTION 'Invalid amount 50.00 for invoice ...: invoice balance is = 40.00'
**Parameters:** none (structural)
**Edge cases handled:**
- invoice.getBalance() <= 0 → requestedAmount forced to 0 immediately, treated as already-paid (line 712-714)
- Zero-amount invoice with allowEmptyInvoice=true (default false) → payment allowed to proceed for $0 instead of being aborted (InvoicePaymentControlPluginApi.java:364-374; PaymentConfig.java:138-141)
- Non-API (system) payment with amount > balance is not blocked here — it is silently capped to invoice.getBalance() (line 727)
**Confidence:** High

### RULE-037: Refund amount validation against invoice items
**Category:** Validation
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:529-565`
**Plain English:** When a refund does not specify a total amount but instead specifies per-item amounts, each item's refund amount must be positive and cannot exceed that line item's original invoice amount; if a specific total refund amount is given directly, it just needs to be positive (no per-item cross-check is enforced against overall payment size at this stage).
**Specification:**
  Given an invoice item originally billed at $30.00, and a refund request specifying $35.00 against that item
  When  computeRefundAmount() validates the per-item refund map
  Then  the call is aborted with 'You need to specify a valid invoice item amount' (specifiedItemAmount=$35.00 > itemAmount=$30.00)
**Parameters:** none
**Edge cases handled:**
- specifiedRefundAmount (total) <= 0 → aborted with 'You need to specify a positive refund amount' (line 534-536)
- specifiedItemAmount omitted for an item in the map → defaults to that item's full original amount (line 550, Objects.requireNonNullElse)
- computeRefundAmount() == 0 overall and isApiPayment=true → whole refund call aborted (line 456-464); if not an API payment, a $0 refund is allowed to proceed silently
**Confidence:** High

### RULE-038: Target date cannot be too far in the future
**Category:** Validation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:118-124`
**Plain English:** An invoice cannot be generated for a target (as-of) date that is more than a configured number of months ahead of today.
**Specification:**
  Given org.killbill.invoice.maxNumberOfMonthsInFuture default = 36 months, today = 2026-08-31, requested targetDate = 2030-01-01 (>36 months out)
  When  validateTargetDate() runs at the start of invoice generation
  Then  an InvoiceApiException with ErrorCode.INVOICE_TARGET_DATE_TOO_FAR_IN_THE_FUTURE is thrown and no invoice is generated
**Parameters:** org.killbill.invoice.maxNumberOfMonthsInFuture, default 36 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:63-71)
**Edge cases handled:**
- Comparison uses whole months between today and targetDate (Joda Months.monthsBetween), so day-of-month granularity within the boundary month is not itself rejected
**Confidence:** High

### RULE-039: Cannot cancel an already-cancelled entitlement
**Category:** Validation
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:364-366,507-509`
**Plain English:** Cancelling an entitlement that is already CANCELLED is rejected.
**Specification:**
  Given isEntitlementCancelled() is true
  When  cancelEntitlementWithDate is called again
  Then  SUB_CANCEL_BAD_STATE exception thrown
**Parameters:** ErrorCode.SUB_CANCEL_BAD_STATE
**Confidence:** High

### RULE-040: Cancellation effective date cannot precede entitlement or subscription start
**Category:** Validation
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:337-342`
**Plain English:** You cannot backdate an entitlement cancellation to before the entitlement started, nor a billing cancel date before subscription start.
**Specification:**
  Given Entitlement with effective start date 2026-01-01
  When  cancelEntitlementWithDate is called with date 2025-12-15
  Then  SUB_INVALID_REQUESTED_DATE exception thrown
**Parameters:** n/a
**Confidence:** High

### RULE-041: Uncancel only allowed with a pending or existing cancellation to reverse
**Category:** Validation
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:422-455`
**Plain English:** Uncancel fails if billing already fully cancelled the subscription, or if there is no cancellation event to reverse.
**Specification:**
  Given No cancellation event recorded
  When  uncancelEntitlement() is called
  Then  ENT_UNCANCEL_BAD_STATE thrown
**Parameters:** n/a
**Edge cases handled:**
- If billing had a future end date, uncancel reverses that too
**Confidence:** High

### RULE-042: Cannot void a repaired, used-credit-generating, or paid invoice
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:753-799,815-825`
**Plain English:** Voiding is blocked if the invoice was repaired, generated already-used account credit, or has any non-zero net paid/refunded amount; the first two checks apply only to COMMITTED invoices.
**Specification:**
  Given COMMITTED invoice generated $50 credit, $30 already used
  When  voidInvoice is called
  Then  CAN_NOT_VOID_INVOICE_THAT_GENERATED_USED_CREDIT thrown
**Parameters:** n/a
**Edge cases handled:**
- Fully refunded (net zero) invoices remain voidable
**Confidence:** High

### RULE-043: Missing currency price is a hard error
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:88-100`
**Plain English:** If a plan phase defines prices for some currencies but not the one requested, pricing fails loudly rather than defaulting to zero or another currency.
**Specification:**
  Given A phase's prices[] array is non-empty but has no entry for currency=JPY
  When  getPrice(JPY) is called
  Then  CatalogApiException CAT_NO_PRICE_FOR_CURRENCY is thrown
**Confidence:** High

### RULE-044: Negative catalog prices rejected
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java:107-125`
**Plain English:** Catalog validation rejects any price entry with a negative value in a supported currency.
**Specification:**
  Given A DefaultPrice with value -5.00 USD in a catalog whose supported currencies include USD
  When  catalog validation runs (DefaultInternationalPrice.validate)
  Then  A validation error 'Negative value for price in currency: USD' is added; a CurrencyValueNull price (no value set) is silently skipped
**Confidence:** High

### RULE-045: TOP_UP usage block requires a minimum top-up credit
**Category:** Validation
**Priority:** P2
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultBlock.java:91-111`
**Plain English:** A usage block of type TOP_UP must declare a minimum top-up credit amount; requesting that value on a non-TOP_UP block is an error.
**Specification:**
  Given A DefaultBlock with type=TOP_UP and no minTopUpCredit set
  When  catalog validation runs
  Then  Validation error 'TOP_UP block needs to define minTopUpCredit for phase X' is added; conversely calling getMinTopUpCredit() on a VANILLA/TIERED block that happens to have a non-default value throws CAT_NOT_TOP_UP_BLOCK
**Confidence:** High

### RULE-046: Price override cannot introduce a price dimension that didn't exist
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java:104-119`
**Plain English:** You can override an existing fixed or recurring price on a plan phase, but you cannot add a fixed price to a phase that never had one (or a recurring price to a phase that never had one).
**Specification:**
  Given A plan phase with no fixed-price component (curPhase.getFixed() == null)
  When  An override request supplies a non-null fixedPrice for that phase
  Then  CatalogApiException CAT_INVALID_INVALID_PRICE_OVERRIDE is thrown with message 'There is no existing fixed price for the phase X'; same rule applies symmetrically for recurring price
**Confidence:** High

### RULE-047: A recurring invoice item is only pruned once fully repaired, never over-repaired
**Category:** Validation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java:156-232`
**Plain English:** When a subscription change retroactively cancels part of a billing period, the system tracks REPAIR_ADJ and ITEM_ADJ items against the original RECURRING charge; only when the repairs (net of adjustments) exactly equal the original charge is that whole item considered fully repaired and removed from consideration, and any repair/adjustment combination that would exceed the original charge is flagged as an illegal state.
**Specification:**
  Given An original RECURRING item of $100.00 with REPAIR_ADJ items totaling -$100.00 and no ITEM_ADJ
  When  InvoicePruner.getFullyRepairedItemsClosure() builds the closure
  Then  The original item and its repair items are added to the 'fully repaired' set and pruned from the working tree; if repairs+adjustments summed to more than $100.00 a Preconditions failure ('Too many repairs...') is raised (except when invoice optimization is on, where dangling repairs are tolerated)
**Edge cases handled:**
- $0 original RECURRING items are ignored entirely (line 178-181)
- ITEM_ADJ added directly by an invoice plugin can silently over-adjust past the original amount; this bypass is explicitly tolerated per comment at line 197-202
**Confidence:** High

### RULE-048: Usage periods already covered by an existing invoice item are skipped to prevent double billing
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:294-307`
**Plain English:** When re-running usage invoicing (e.g. after a blocking/overdue event changes billing dates), any rolled-up usage period that's already fully contained in a previously-generated usage invoice item is skipped rather than billed again.
**Specification:**
  Given A rolled-up usage window of 2026-03-01 to 2026-03-10 that is fully contained within an existing USAGE invoice item covering 2026-03-01 to 2026-03-15
  When  usage items are (re)computed for the subscription
  Then  That window is skipped (logged as ignored) and no new usage item is generated for it
**Confidence:** High

### RULE-049: Same-day usage items are excluded when re-billing a larger period
**Category:** Validation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java:573-598`
**Plain English:** If a usage item was previously billed for a single day (e.g. because of a same-day plan change), and the system is now computing usage for a longer period that spans that day, the single-day item is not counted as 'already billed' for that longer period — its usage gets re-evaluated as part of the larger window.
**Specification:**
  Given An existing USAGE invoice item with startDate == endDate (a same-day item from an earlier plan change), and a new billing pass computing a multi-day period that includes that day
  When  getBilledItems(startDate, endDate, existingUsage) filters candidate already-billed items
  Then  The same-day item is excluded from 'already billed' consideration (isSameDay check fails), so its usage is not double-subtracted nor double-protected against re-billing
**Confidence:** Medium — Confirm this doesn't cause the same-day usage record to be double-counted in the larger period's total, versus intentionally re-consolidating it.

### RULE-050: Max add-ons of the same plan per bundle (creation)
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/svcs/DefaultSubscriptionBaseCreateApi.java:234-246`
**Plain English:** A bundle cannot have more add-on subscriptions of the exact same plan than the catalog's configured 'plans allowed in bundle' limit for that plan.
**Specification:**
  Given A plan with plansAllowedInBundle=2 and a bundle that already has 2 active/pending add-ons of that plan
  When  A 3rd add-on subscription of the same plan is created in that bundle
  Then  Creation is rejected with SUB_CREATE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE
**Parameters:** plansAllowedInBundle: catalog-defined per plan; -1 or <=0 means unlimited (check skipped)
**Confidence:** High

### RULE-051: Max add-ons of the same plan per bundle (plan change)
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:474-486`
**Plain English:** Changing a subscription's plan to an add-on plan is blocked if the bundle already has as many add-ons of the target plan as the catalog allows.
**Specification:**
  Given A target add-on plan with plansAllowedInBundle=1 and the bundle already has 1 existing add-on of that plan
  When  Another subscription in the bundle is changed to that same add-on plan
  Then  The change is rejected with SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE
**Parameters:** plansAllowedInBundle: catalog-defined per plan
**Suspected defect:** Code comment at line 479 says 'the plan can be changed... because it has reached its limit' which contradicts the following throw statement (change is actually rejected); comment text appears inverted/copy-paste error, not an instruction to the analyzer.
**Confidence:** High

### RULE-052: Plan change forbidden across product categories
**Category:** Validation
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:484-486`
**Plain English:** A subscription can only be changed to a plan in the same product category it currently has (e.g. a BASE plan cannot be changed into an ADD_ON plan or vice versa).
**Specification:**
  Given A subscription currently on a BASE-category plan
  When  A change-plan request targets a plan whose product category is ADD_ON
  Then  The change is rejected with SUB_CHANGE_INVALID
**Parameters:** None (structural category comparison)
**Confidence:** High

### RULE-053: Change/cancel effective date cannot precede last processed transition
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:823-833`
**Plain English:** A requested effective date for a subscription change (or a cancellation date computed elsewhere) must not be earlier than the subscription's last transition (or its start date if no transition yet occurred).
**Specification:**
  Given A subscription whose last transition (e.g. PHASE) occurred on 2026-01-15
  When  A change-plan request with requested date 2026-01-10 (before the last transition) is submitted
  Then  The call is rejected with SUB_INVALID_REQUESTED_DATE
**Parameters:** None
**Confidence:** High

### RULE-054: Usage/product limit compliance check (min/max)
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultLimit.java:100-105`
**Plain English:** A usage unit value is checked against a catalog-configured max and min for that unit; if a max is set the value must not exceed it, and if a min is set the value must comply with it.
**Specification:**
  Given A catalog Limit for unit 'users' with min=5 and no max
  When  compliesWith(value) is evaluated for value=10
  Then  Per the current code, compliance requires value <= min, so value=10 would be reported non-compliant even though it exceeds (not falls below) the min
**Parameters:** max, min: BigDecimal, catalog-defined per Limit/Unit; sentinel -1 (DEFAULT_NON_REQUIRED_BIGDECIMAL_FIELD_VALUE) means 'not set'
**Suspected defect:** The min branch `return !minHasValue || value.compareTo(min) <= 0;` requires value <= min to comply, which is the inverse of the conventional 'value must be >= min' semantics implied by a Limit named 'min'. No caller of compliesWith()/compliesWithLimits() was found wired into subscription/invoice eligibility flows in this codebase, so real-world impact is unclear.
**Confidence:** Low — Is DefaultLimit.compliesWith's min check intentionally inverted (i.e. does 'min' actually mean an upper bound for some unit types), or is `value.compareTo(min) <= 0` a bug that should be `>= 0`? Also confirm whether any plugin/consumer actually calls compliesWith/compliesWithLimits in production.

### RULE-055: Account external key must be unique and ≤ 255 characters
**Category:** Validation
**Priority:** P1
**Source:** `account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:85-104`
**Plain English:** When creating an account, the external key (if supplied) must not already be in use by another account, and must be 255 characters or fewer.
**Specification:**
  Given An external key 'cust-123' already assigned to an existing account
  When  createAccount is called again with externalKey='cust-123'
  Then  The call fails with ACCOUNT_ALREADY_EXISTS; separately, a key longer than 255 chars fails with EXTERNAL_KEY_LIMIT_EXCEEDED
**Parameters:** External key max length: 255 characters (hardcoded)
**Confidence:** High

### RULE-056: Parent account must exist when linking a new account
**Category:** Validation
**Priority:** P1
**Source:** `account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:93-99`
**Plain English:** If a new account specifies a parent account id, that parent account must already exist.
**Specification:**
  Given A createAccount request with parentAccountId pointing to a non-existent account
  When  createAccount is called
  Then  The call fails with ACCOUNT_DOES_NOT_EXIST_FOR_ID
**Parameters:** None
**Confidence:** High

### RULE-057: Account external key, currency, BCD, timezone, and reference time are immutable once set
**Category:** Validation
**Priority:** P0
**Source:** `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:502-546`
**Plain English:** Once an account has a non-null external key, currency, bill-cycle-day, timezone, or reference time, an update attempting to change any of those fields to a different value is rejected outright (this API doesn't support changing them).
**Specification:**
  Given An account with currency=USD already set
  When  An update request supplies currency=EUR
  Then  IllegalArgumentException 'Killbill doesn't support updating the account currency yet' is thrown; the same pattern applies to externalKey, billCycleDayLocal, timeZone, and referenceTime (date-only compare)
**Parameters:** DEFAULT_BILLING_CYCLE_DAY_LOCAL sentinel value used to mean 'no BCD set'
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 is justified: `validateAccountUpdateInput` (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:502-546) is a data-integrity guard on core account attributes (currency, external key, BCD, timezone, reference time) that feed invoicing/proration/tax logic downstream — corrupting them silently would corrupt billing.

The headline Given/When/Then (currency USD->EUR throws "Killbill doesn't support updating the account currency yet") is faithful and matches lines 521-526 verbatim, including the exact exception string.

However the card's generalization is not faithful in two respects, so I mark it non-faithful overall:

1. The card omits `ignoreNullInput` as a parameter even though it is the actual governing switch for all four ignoreNullInput-gated fields (externalKey 514-519, currency 521-526, BCD 528-532, timeZone 534-539). Each check is gated by `(ignoreNullInput || fieldNotNull)`: when `ignoreNullInput=false`, supplying a null new value bypasses the check entirely (silently allowed, i.e. "reset" semantics per the code's own comment at 508-511); when `ignoreNullInput=true`, a null new value against a non-null current value still throws. The card's plain-English summary ("an update attempting to change any of those fields to a different value is rejected") only describes the mismatch case and drops this null-handling distinction, which is a real behavioral fork, not incidental.

2. The card claims "the same pattern applies to ... referenceTime," but referenceTime (541-544) does NOT use `ignoreNullInput` at all — it only checks `referenceTime != null && ...date-truncated compare... != 0`. A null new referenceTime is unconditionally skipped regardless of `ignoreNullInput`, unlike the other four fields where `ignoreNullInput=true` forces the check even on null input. So referenceTime is categorically different, not "the same pattern" with a mere date-only-compare footnote.

Net: the single cited example is correct, but the rule as generalized across all five fields overstates uniformity and omits a parameter (`ignoreNullInput`) that materially changes whether a reset-to-null is accepted or rejected — this needs correction before it becomes a verification contract for the rewrite. No prompt-injection attempts were found in the cited code; comments at 504-513 are plain explanatory text.

### RULE-058: Usage record idempotency by tracking id
**Category:** Validation
**Priority:** P1
**Source:** `usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java:70-92`
**Plain English:** When recording rolled-up usage, if a tracking id is supplied and usage rows already exist with that tracking id, the submission is rejected as a duplicate; if no tracking id is supplied, a new random one is generated (allowing the submission).
**Specification:**
  Given A prior successful recordRolledUpUsage call used trackingId='batch-42'
  When  recordRolledUpUsage is called again with trackingId='batch-42'
  Then  The call fails with USAGE_RECORD_TRACKING_ID_ALREADY_EXISTS, preventing double-counting of usage
**Parameters:** None
**Confidence:** High

### RULE-059: Payment transaction external key uniqueness and account isolation
**Category:** Validation
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:327-350`
**Plain English:** A payment transaction external key cannot be reused for a second SUCCESSful transaction of a non-CHARGEBACK type, and a given external key cannot be shared across different accounts.
**Specification:**
  Given A transaction with external key 'txn-001' already SUCCESS and type PURCHASE for account A
  When  A new transaction is submitted reusing external key 'txn-001'
  Then  The call fails with PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS; if the existing transaction under that key belongs to a different account, it instead fails with PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID
**Parameters:** CHARGEBACK is the sole transaction type exempted from the 'no reuse after success' rule
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 justified (compliance lens): The rule guards two things a finance controller/auditor would flag if silently broken. (1) Duplicate-payment prevention — `runSanityOnTransactionExternalKey` (PaymentProcessor.java:327-350) blocks re-use of an external key for a second SUCCESS, non-CHARGEBACK transaction, which is the idempotency guard that stops a retried/duplicated client request from moving money twice (double-charge risk). (2) Account isolation — it also blocks a transaction key that belongs to a different account (`accountRecordId` mismatch), which prevents payment records from being misattributed across tenants/accounts, a data-integrity/financial-reporting correctness issue. Either failure mode (double-charge or cross-account misattribution) is exactly the kind of silent behavior change a payments auditor would want caught, so P0 is warranted.

However, the rule card is NOT faithful to the code as written, so it should not be trusted verbatim before verification-suite authoring. Re-reading PaymentProcessor.java:330-348: the loop runs two INDEPENDENT if-checks per existing transaction with the same key, not an if/else-fallback pair as the card's Given/When/Then implies. Line 330-336 throws PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS whenever an existing transaction under the key is SUCCESS and non-CHARGEBACK — with NO account check at all. Line 338-348 (a separate, unconditional check on the same record) throws PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID whenever the record's account differs from the caller's, regardless of that record's status/type — but it is only reached when the first `if` was false (i.e., status != SUCCESS or type == CHARGEBACK). Concretely: the card's own example — 'existing txn-001 already SUCCESS/PURCHASE for account A, reused by a different account' — would actually throw PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS (line 335), NOT PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID as the card's Then-clause states, because the SUCCESS+non-CHARGEBACK branch fires first and does not check account at all. The DIFFERENT_ACCOUNT_ID error only fires for same-key records that are not(SUCCESS && non-CHARGEBACK) — e.g., a PENDING/FAILED or CHARGEBACK transaction — combined with a different account. This precedence/ordering detail must be corrected before the rule card is used as a behavior-equivalence oracle, or a rewrite that 'fixes' the example scenario to return DIFFERENT_ACCOUNT_ID would actually be diverging from legacy behavior. No injection-shaped text was found in the cited lines. | P0 classification is justified: this rule guards data integrity (prevents cross-account key collisions and duplicate processing of the same external transaction key), which is exactly the kind of thing that must be behavior-equivalent post-rewrite.

However the Given/When/Then is NOT faithful to the code's actual branching, specifically on the "instead" framing.

Read code at payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:327-350:

```
330: for (final PaymentTransactionModelDao paymentTransactionModelDao : allPaymentTransactionsForKey) {
332:     if (...key.equals(...) && status == SUCCESS && type != CHARGEBACK) {
335:         throw PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS;
336:     }
339:     if (!accountRecordId.equals(internalCallContext.getAccountRecordId())) {
347:         throw PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID;
348:     }
350: }
```

These are two independent, sequential `if` blocks per iterated transaction — NOT an if/else fork keyed on account. Check 1 (SUCCESS + non-CHARGEBACK) is evaluated first and does not reference account at all. So for the exact scenario the rule card poses — an existing txn under the key that is already SUCCESS and type PURCHASE — check 1 fires and throws PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS regardless of whether the new request's account matches the existing transaction's account. The card's "if it belongs to a different account, it instead fails with PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID" describes an outcome that cannot occur in that scenario: check 1 preempts check 2 whenever the stored transaction is SUCCESS/non-CHARGEBACK.

PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID only fires for a matching-key transaction that is NOT (SUCCESS AND non-CHARGEBACK) — e.g. PENDING, FAILED, or a CHARGEBACK-type record — and belongs to a different account. The rule card conflates the two checks as alternate outcomes of the same condition set, when they're actually gated by disjoint status/type conditions, with KEY_EXISTS taking strict precedence whenever both conditions would otherwise be true. There's also an unaddressed edge case: allPaymentTransactionsForKey can contain multiple rows (multiple prior attempts under the same key, possibly across accounts), and the actual thrown error depends on DB iteration order among those rows — the card's GWT assumes a single existing transaction.

No prompt-injection content was found in the cited lines (327-350) — comments are ordinary engineering notes (e.g. line 331 chargeback caveat, line 361 sanity-check note), not instruction-shaped text aimed at automated analysis.

Recommendation: rewrite the GWT to reflect the true precedence — "KEY_EXISTS is thrown if the matching-key transaction is SUCCESS and not CHARGEBACK, checked before and independent of account match; DIFFERENT_ACCOUNT_ID is thrown only when the matching-key transaction fails that SUCCESS/non-CHARGEBACK test and belongs to a different account" — and flag the multi-row iteration-order dependency for SME confirmation (confidence should be Medium, not High, given this ordering subtlety).

### RULE-060: At most one PENDING initial payment transaction per payment
**Category:** Validation
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:352-385`
**Plain English:** A payment cannot have more than one PENDING AUTHORIZE, PURCHASE, or CREDIT transaction outstanding at the same time; a new initiating transaction request while one is already PENDING is rejected. Additionally, if reusing an existing transaction id/key, the transaction type of the new request must match the existing transaction's type, and there must never be more than one completion candidate.
**Specification:**
  Given A payment with an existing PENDING AUTHORIZE transaction
  When  A new AUTHORIZE/PURCHASE/CREDIT request (without matching id/key) comes in for the same payment
  Then  The call fails with PAYMENT_INVALID_OPERATION; a mismatched transaction type on a matching id/key fails with PAYMENT_INVALID_PARAMETER; internal invariant enforced via Preconditions.checkState that at most one completion candidate exists
**Parameters:** Transaction types subject to the single-pending rule: AUTHORIZE, PURCHASE, CREDIT
**Confidence:** Medium — P0 panel doubts spec fidelity: Re-derived from PaymentProcessor.java:352-385 (invoked only when paymentId is set and either transactionId or paymentTransactionExternalKey is present, per line 297).

What the code actually does, iterating over paymentTransactionsForCurrentPayment (352-385):
- For each existing transaction that does NOT match the incoming call's transactionId/externalKey (359-360): if that OTHER transaction's type is AUTHORIZE/PURCHASE/CREDIT and its status is PENDING, throw PAYMENT_INVALID_OPERATION(existingTxnType, paymentStateName) (362-366). Otherwise skip it (368).
- For the transaction that DOES match the incoming id/key (372-380): if its type differs from the incoming request's transactionType, throw PAYMENT_INVALID_PARAMETER("transactionType", ...) (373-375); if its status is PENDING or UNKNOWN, add it to completionCandidates (378-380).
- After the loop: Preconditions.checkState(candidates.size() <= 1) and return the last (or only) candidate (383-385).

Confirmed correct pieces of the rule: the type-mismatch-on-matching-key check (PAYMENT_INVALID_PARAMETER, 372-375) and the checkState single-completion-candidate invariant (383) are described accurately, in the right order.

The inaccuracy is in the "When" clause: the rule states the guard fires for "a new AUTHORIZE/PURCHASE/CREDIT request." That is too narrow. The gating condition at 362-366 tests the TYPE OF THE EXISTING PENDING TRANSACTION, not the type of the incoming request. I verified this by checking all callers of performOperation (lines 103-142): captureAuthorization, voidPayment, refundPayment, and chargeback all pass a paymentTransactionExternalKey for their own new transaction (a key that won't match an unrelated pending AUTHORIZE), so any of these non-initiating transaction types will also hit the PAYMENT_INVALID_OPERATION throw at line 366 if the payment has some other PENDING AUTHORIZE/PURCHASE/CREDIT transaction under a different id/key — not just new AUTHORIZE/PURCHASE/CREDIT requests as the card states. So the Given/When/Then materially understates the scope of what gets rejected; a verification suite built only against "AUTHORIZE/PURCHASE/CREDIT incoming requests" would miss CAPTURE/VOID/REFUND/CHARGEBACK cases the code actually blocks.

P0 justification stands independent of the faithfulness issue: this logic guards against a payment ending up with two concurrently outstanding (PENDING) money-moving transactions and against completing/capturing the wrong transaction — a direct data-integrity and double-charge-prevention control on the payment core, which is exactly the kind of rule that must be preserved and behavior-tested across a rewrite.

No prompt-injection-shaped text found in the cited range (302-385) or the surrounding methods read (200-390); comments are plain engineering notes (e.g. "Sanity: ...", "cannot be enforced by the state machine unfortunately").

### RULE-061: Add-on creation eligibility against base plan
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-53`
**Plain English:** An add-on can only be created if its base subscription is active (or pending-future and the add-on start is not before the base start), the add-on's product is not already included for free in the base product, and the add-on product is listed as available for the base product in the catalog.
**Specification:**
  Given A base subscription in CANCELLED state, or an add-on product not listed under the base product's 'available' add-ons
  When  checkAddonCreationRights is invoked while creating the add-on
  Then  SubscriptionBaseApiException is thrown with SUB_CREATE_AO_BP_NON_ACTIVE, SUB_CREATE_AO_ALREADY_INCLUDED, or SUB_CREATE_AO_NOT_AVAILABLE respectively
**Parameters:** None (driven by catalog's per-product available/included add-on lists)
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (I read AddonUtils.java:36-53 (and the caller SubscriptionApiBase.java:229-245 for context).

FAITHFULNESS ISSUE FOUND: The extracted rule's plain-English description of the PENDING-base branch has the date comparison backwards. Code at line 39:
`baseSubscription.getState() == EntitlementState.PENDING && context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0`
throws SUB_CREATE_AO_BP_NON_ACTIVE when baseStartDate < requestedDate (i.e., the add-on's requested/effective date is AFTER the base's start date). The rule card states the add-on is allowed when its "start is not before the base start" (addonStart >= baseStart) — but that is exactly the condition the code REJECTS, not permits. Cross-checking the caller (SubscriptionApiBase.java:241-245), effectiveDate is already guaranteed >= baseSubscription.getStartDate() before checkAddonCreationRights is even called (a separate SUB_INVALID_REQUESTED_DATE check enforces that). So the real, composite rule is: for a PENDING base, the add-on's effective date must equal the base's start date exactly — any later date throws BP_NON_ACTIVE. The rule card's inequality direction is inverted, which would produce a wrong equivalence test in modernization (e.g., asserting a later add-on date succeeds when the code actually rejects it). The CANCELLED-state, ALREADY_INCLUDED, and NOT_AVAILABLE clauses (lines 38, 45-53) are accurately described and match isAddonIncluded/isAddonAvailable (lines 56-77).

COMPLIANCE JUDGMENT: This rule enforces catalog-driven product entitlement (base-must-be-active, no duplicate-of-included-product, add-on-must-be-in-catalog's-available-list). It does protect internal subscription/catalog data integrity and indirectly prevents billing for unauthorized product combinations, but it is not something a regulator, external auditor, or finance controller would specifically monitor — there's no tax, AML/KYC, PCI, SOX financial-control, or statutory retention/disclosure obligation here. It's a domain/product business rule (entitlement correctness), not a compliance control. I'd downgrade this from P0-compliance to P1 (business-critical correctness rule) rather than treat it as a regulator-relevant control.

No prompt-injection-shaped text found in the cited lines (36-53) — only a standard Apache license header at the top of the file, unrelated to the cited range.

Files reviewed: /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:34-90, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:229-245.</reason>
</invoke>
 | Re-derived directly from AddonUtils.java:36-53 (confirmed against DefaultSubscriptionBase.getState()/getStartDate(), Entitlement.EntitlementState enum, ErrorCode.java, and the caller SubscriptionApiBase.java:229-245).

What's faithful:
- The three checks fire in the stated order and with the stated error codes: CANCELLED/bad-PENDING -> SUB_CREATE_AO_BP_NON_ACTIVE (line 38-40); already-included -> SUB_CREATE_AO_ALREADY_INCLUDED (line 45-48); not-available -> SUB_CREATE_AO_NOT_AVAILABLE (line 50-53).
- isAddonIncluded/isAddonAvailable (lines 56-77) do exactly what's described: linear name-match against baseProduct.getIncluded()/getAvailable() from the catalog.
- Parameters: correctly "None" — driven entirely by catalog data, no hardcoded magic numbers/credentials.

Two fidelity defects against the cited code:

1. Base-state gate is narrower than described. EntitlementState has 5 values: PENDING, ACTIVE, BLOCKED, CANCELLED, EXPIRED (Entitlement.java). Line 38-39 only rejects CANCELLED (plus the PENDING edge case below) — BLOCKED and EXPIRED base subscriptions are NOT rejected by this method and would fall through to the included/available checks. The card's plain English ("base subscription is active... or pending-future...") implies an active-or-nothing gate that the code does not actually enforce for BLOCKED/EXPIRED bases.

2. The PENDING sub-condition's direction is inverted. Line 39: `baseStartDate.compareTo(requestedDate) < 0` is true when baseStartDate is chronologically BEFORE requestedDate — i.e. it throws when the add-on's requested date falls AFTER the base's own (future) start date, not when it falls before. The card states the allowed condition as "the add-on start is not before the base start" (i.e., disallowed only when addon-start < base-start), which is the opposite polarity of what this function does. The "addon start before base start" case is actually guarded by a different check in a different file — SubscriptionApiBase.java:241-242 (`effectiveDate.isBefore(baseSubscription.getStartDate())` -> SUB_INVALID_REQUESTED_DATE) — which is outside the cited range and uses a different error code entirely. The card conflates the two checks and mis-states the direction of the one it cites.

P0 justification stands regardless: this rule (in its correct form) gates creation of billable add-on items against subscription state and catalog constraints — letting an add-on attach to a cancelled/expired/blocked base, or to a product already included free, or not listed as available, would corrupt the subscription/entitlement state machine and produce incorrect invoice line items. That is a genuine data-integrity guard feeding billing, so P0 is justified even though the Given/When/Then needs correction before it can serve as a verification contract.

Recommendation: rewrite the card's PENDING branch as "Given base state PENDING (start date in the future) and add-on requested date's calendar day is after the base's start date, When checkAddonCreationRights runs, Then SUB_CREATE_AO_BP_NON_ACTIVE is thrown" and drop "active" framing in favor of "not CANCELLED, and not (PENDING with requested date after base start)" — noting BLOCKED/EXPIRED bases pass this specific check.

Files reviewed: /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-53, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:187-205, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:223-246. No prompt-injection-shaped text found in the cited lines.) — confirm criticality.

### RULE-062: Re-cancellation of an already future-cancelled/expiring subscription is blocked unless the new date is earlier
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:264-296`
**Plain English:** If a subscription already has a pending future cancellation or a pending FIXEDTERM expiry, a new cancel request is rejected unless its effective date is strictly earlier than the existing pending one (i.e., the user must uncancel before re-cancelling for a later date); a strictly earlier date silently invalidates the old pending cancellation.
**Specification:**
  Given A subscription with a pending future CANCEL transition effective 2026-10-01
  When  cancelWithRequestedDate is called again with effective date 2026-11-01
  Then  SubscriptionBaseApiException SUB_CANCEL_BAD_STATE ('PENDING CANCELLED') is thrown; calling it again with an earlier date (e.g. 2026-09-01) is allowed and replaces the pending cancellation
**Parameters:** None
**Confidence:** High

### RULE-063: Effective date for subscription mutation must not precede last recorded transition
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:823-833`
**Plain English:** Any requested effective date for a subscription change/cancel must be on or after the subscription's most recent past transition (or its start date if none has occurred yet).
**Specification:**
  Given A subscription created 2026-01-01 with a PHASE transition on 2026-04-01
  When  A cancellation is requested with effective date 2026-03-01
  Then  SubscriptionBaseApiException SUB_INVALID_REQUESTED_DATE is thrown because 2026-03-01 precedes the last transition (2026-04-01)
**Parameters:** None
**Confidence:** High

### RULE-064: Change-plan blocked for non-active, future-cancelled, or already-future-expired subscriptions
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:835-850`
**Plain English:** A plan change is rejected if the subscription is CANCELLED/EXPIRED, if the requested effective date precedes the subscription start, if the subscription already has a pending future cancellation, or if the requested effective date falls after an already-scheduled future expiry.
**Specification:**
  Given A subscription with a pending future EXPIRED transition on 2026-12-01
  When  changePlanWithRequestedDate is called with effective date 2026-12-15
  Then  SubscriptionBaseApiException SUB_CHANGE_FUTURE_EXPIRED is thrown
**Parameters:** None
**Confidence:** High

### RULE-065: Max add-on instances per bundle enforced on plan change
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:473-486`
**Plain English:** When changing a subscription to an add-on plan that has a configured 'plans allowed in bundle' limit (a positive number, -1 means unlimited), the change is rejected if the bundle already has that many active/pending subscriptions on the same plan; also, a plan change may never move a subscription into a different product category than it currently has.
**Specification:**
  Given An add-on plan configured with plansAllowedInBundle = 1, and the bundle already has one active subscription on that plan
  When  changePlan requests the same add-on plan for a second subscription in the bundle
  Then  SubscriptionBaseApiException SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE is thrown
**Parameters:** plansAllowedInBundle: per-plan catalog value, -1 = unlimited
**Confidence:** High

### RULE-066: Plugins may not set reserved entitlement blocking states or service name
**Category:** Validation
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultSubscriptionApi.java:369-379`
**Plain English:** External callers/plugins adding a custom blocking state may not use the entitlement service's own service name, and may not set the state name to one of the reserved internal states (ENT_CANCELLED, ENT_BLOCKED, ENT_CLEAR), preventing plugins from hijacking internal entitlement/pause state names.
**Specification:**
  Given A plugin calls addBlockingState with stateName='ENT_BLOCKED'
  When  The API validates the input
  Then  EntitlementApiException SUB_BLOCKING_STATE_INVALID_ARG ('Need to specify a valid stateName') is thrown
**Parameters:** Reserved names: ENT_CANCELLED, ENT_BLOCKED, ENT_CLEAR; reserved service: entitlement-service
**Confidence:** High

### RULE-067: Usage tier section must define blocks or limits appropriate to its type
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultTier.java:148-159`
**Plain English:** In the product catalog, an IN_ARREAR CAPACITY usage tier must declare at least one limit, and an IN_ARREAR CONSUMABLE usage tier must declare at least one priced block; catalogs missing these are rejected at load time.
**Specification:**
  Given A catalog XML defines a usage section with billingMode=IN_ARREAR and usageType=CONSUMABLE but no <blocks> element
  When  The catalog is validated on load
  Then  A ValidationError is raised: "Usage [IN_ARREAR CONSUMABLE] section of phase <phase> needs to define some blocks"
**Confidence:** High

### RULE-068: Bundle transfer eligibility: skip cancelled subscriptions and optionally skip add-ons
**Category:** Validation
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:242-245,279-282`
**Plain English:** During a bundle transfer, subscriptions already in CANCELLED state are not carried over to the new account, and add-on subscriptions are only carried over if the caller explicitly requests add-on transfer.
**Specification:**
  Given A bundle with a cancelled add-on and an active add-on, transferAddOn=false
  When  transferBundle() iterates the bundle's subscriptions
  Then  The cancelled subscription is skipped entirely; the active add-on is also skipped (not created on the destination account) because transferAddOn is false
**Confidence:** High

### RULE-069: Catalog version consistency validation
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java:134-192`
**Plain English:** Two catalog versions cannot share the same effective date, all versions must have the same catalog name, and a plan that exists in multiple catalog versions must keep the same number and names of phases across those versions.
**Specification:**
  Given Two uploaded catalog versions both declare effectiveDate=2026-01-01, or a plan 'gold-monthly' has 2 phases in one version and 3 phases in a later version
  When  The versioned catalog is validated
  Then  A ValidationError is added ("Catalog effective date ... already exists for a previous version" or "Number of phases for plan ... differs between version ...")
**Confidence:** High

### RULE-070: Grandfather date cannot precede its own catalog version's effective date
**Category:** Validation
**Priority:** P2
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:286-293`
**Plain English:** A plan's 'effective date for existing subscriptions' (grandfathering rollout date) must be on or after the catalog version's own effective date; catalogs that set it earlier are rejected as invalid.
**Specification:**
  Given A catalog version effective 2026-01-01 defines a plan with effectiveDateForExistingSubscriptions=2025-12-01
  When  The catalog is validated
  Then  A ValidationError is raised: "Price effective date ... is before catalog effective date ..."
**Confidence:** High

### RULE-071: Tenant external key length limit
**Category:** Validation
**Priority:** P1
**Source:** `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:89-91`
**Plain English:** A tenant's external key cannot be longer than 255 characters.
**Specification:**
  Given A createTenant request with an external key of 256 characters
  When  createTenant is called
  Then  the call is rejected with EXTERNAL_KEY_LIMIT_EXCEEDED before any DB write
**Parameters:** Max length = 255 (hardcoded)
**Confidence:** High

### RULE-072: Tenant API key must be unique
**Category:** Validation
**Priority:** P1
**Source:** `tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java:93-104`
**Plain English:** A new tenant cannot be created with an API key that is already assigned to an existing tenant.
**Specification:**
  Given A tenant already exists with API key 'k1'
  When  createTenant is called again with apiKey 'k1'
  Then  the call fails with TENANT_ALREADY_EXISTS; the lookup is non-transactional (relies on a DB unique constraint as backstop) and an IllegalStateException from a 'not found' lookup is deliberately swallowed to mean 'key is free'
**Edge cases handled:**
- Any other RuntimeException during the pre-check lookup is NOT swallowed and propagates, which could incorrectly abort tenant creation on transient errors
**Confidence:** Medium — Is it intentional that only IllegalStateException-caused lookup failures are treated as 'key available', while any other runtime error during the pre-check aborts creation even though a DB constraint would have caught true duplicates?

### RULE-073: Invoice item adjustment amount must be positive
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:420-422`
**Plain English:** When manually adjusting an invoice item with a specific amount, that amount must be strictly greater than zero.
**Specification:**
  Given A request to adjust invoice item X by amount $0.00 or -$5.00
  When  insertInvoiceItemAdjustment is called with a non-null amount <= 0
  Then  the call is rejected with INVOICE_ITEM_ADJUSTMENT_AMOUNT_SHOULD_BE_POSITIVE
**Confidence:** High

### RULE-074: Cannot adjust a VOID invoice
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:428-431`
**Plain English:** An item-level adjustment cannot be applied to an invoice that has been voided.
**Specification:**
  Given Invoice INV-1 has status VOID
  When  insertInvoiceItemAdjustment targets an item on INV-1
  Then  the call fails with INVOICE_VOID_UPDATED
**Confidence:** High

### RULE-075: Adjustment currency must match invoice currency
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:433-436`
**Plain English:** If a currency is explicitly supplied for an item adjustment, it must match the invoice's own currency.
**Specification:**
  Given Invoice INV-1 is in USD
  When  insertInvoiceItemAdjustment is called with currency=EUR
  Then  the call fails with CURRENCY_INVALID(EUR, USD)
**Confidence:** High

### RULE-076: External charge / credit amount must be non-negative
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:562-568`
**Plain English:** A new external charge or account credit item cannot have a null or negative amount.
**Specification:**
  Given A new EXTERNAL_CHARGE item with amount -$10 or null amount
  When  insertItems is called (via insertExternalCharges / insertCredits)
  Then  the call fails with EXTERNAL_CHARGE_AMOUNT_INVALID (for charges) or CREDIT_AMOUNT_INVALID (for credits)
**Edge cases handled:**
- A zero amount ($0.00) is allowed since the check is strictly < 0
**Confidence:** High

### RULE-077: External charge / credit currency must match account currency
**Category:** Validation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:570-572`
**Plain English:** If a currency is specified on a new external charge or credit item, it must match the account's currency.
**Specification:**
  Given Account currency is USD
  When  An external charge item is inserted with currency=GBP
  Then  the call fails with CURRENCY_INVALID(GBP, USD)
**Confidence:** High

### RULE-078: Cannot add external charge/credit to an already-committed invoice
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:587-600`
**Plain English:** New external charges or credits can only be attached to an existing invoice while it is still DRAFT; committed invoices reject new items, and voided invoices are also blocked (though via a different, generic error path).
**Specification:**
  Given Invoice INV-1 has status COMMITTED
  When  insertExternalCharges/insertCredits targets INV-1 by invoiceId
  Then  the call fails with INVOICE_ALREADY_COMMITTED; if INV-1 is VOID instead, an IllegalStateException is thrown (tracked as a known gap, see comment referencing killbill/killbill#1501)
**Suspected defect:** VOID case throws a raw IllegalStateException instead of a proper InvoiceApiException with a dedicated error code, per the TODO comment at line 594
**Confidence:** High

### RULE-079: Child-to-parent credit transfer eligibility
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:710-729`
**Plain English:** A child account's credit balance (CBA) can only be transferred to its parent account if the account actually has a parent and currently holds a positive credit balance.
**Specification:**
  Given A child account with no parentAccountId, or with a parent but $0.00 CBA
  When  transferChildCreditToParent is called
  Then  the call fails with ACCOUNT_DOES_NOT_HAVE_PARENT_ACCOUNT or CHILD_ACCOUNT_MISSING_CREDIT respectively; only a strictly positive CBA (> $0.00) triggers the transfer
**Confidence:** High

### RULE-080: Account email record add is idempotent by record id, not by email address
**Category:** Validation
**Priority:** P2
**Source:** `account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:329-341`
**Plain English:** Adding an email to an account is rejected only if a record with that exact same internal ID already exists — the system does not check whether the same email address string is already registered for the account.
**Specification:**
  Given Account A already has email 'a@x.com' stored with id U1
  When  addEmail is called with a brand-new randomly generated id U2 and the same address 'a@x.com'
  Then  the insert succeeds and the account now has two email rows with the same address (ACCOUNT_EMAIL_ALREADY_EXISTS only fires on an id collision, which is effectively unreachable with random UUIDs)
**Suspected defect:** The uniqueness check is on the generated UUID primary key rather than the email address, so duplicate email addresses per account are not actually prevented at this layer
**Confidence:** Medium — Is duplicate-email-address prevention meant to be enforced elsewhere (e.g. API layer, DB constraint), or is allowing multiple identical email addresses per account intentional?

### RULE-081: Only one payment method per account may use the external/manual-pay plugin
**Category:** Validation
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:155-162`
**Plain English:** An account can have at most one payment method backed by the built-in external (manual) payment provider plugin.
**Specification:**
  Given Account A already has a payment method using the ExternalPaymentProviderPlugin
  When  addPaymentMethod is called again requesting the same plugin
  Then  the call fails with PAYMENT_EXTERNAL_PAYMENT_METHOD_ALREADY_EXISTS
**Confidence:** High

### RULE-082: Default payment method must belong to the target account
**Category:** Validation
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:552-556`
**Plain English:** You cannot set a payment method as an account's default if that payment method actually belongs to a different account.
**Specification:**
  Given Payment method PM1 belongs to account B
  When  setDefaultPaymentMethod is called for account A with paymentMethodId=PM1
  Then  the call fails with PAYMENT_METHOD_DIFFERENT_ACCOUNT_ID
**Confidence:** High

### RULE-083: AUTO_PAY_OFF tag cannot be removed without a default payment method on file
**Category:** Validation
**Priority:** P0
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:1441-1455`
**Plain English:** You can't turn auto-pay back on (by removing the AUTO_PAY_OFF tag) for an account that has no default payment method configured, since there would be nothing to charge.
**Specification:**
  Given Account A has the AUTO_PAY_OFF tag and paymentMethodId is null
  When  DELETE /accounts/{id}/tags is called including the AUTO_PAY_OFF tag id
  Then  the request is rejected with TAG_CANNOT_BE_REMOVED (400); other tags in the same request are otherwise removable, this check only applies when AUTO_PAY_OFF is among them
**Confidence:** High

### RULE-084: Invoice-list query modes are mutually exclusive
**Category:** Validation
**Priority:** P2
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java:711-713`
**Plain English:** When fetching an account's invoices, you cannot combine 'unpaid invoices only' with 'include migration invoices', cannot combine a start-date filter with 'include migration invoices', and cannot combine 'unpaid invoices only' with 'include invoice components'.
**Specification:**
  Given A GET invoices request with unpaidInvoicesOnly=true and withMigrationInvoices=true
  When  getInvoicesForAccount is called
  Then  the request fails fast with an IllegalStateException ('We don't support fetching unpaid invoices incl. migration') before any query executes
**Confidence:** High

### RULE-085: Plan resolution requires product + billing period, with price-list fallback to default
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java:206-227`
**Plain English:** When creating a subscription without specifying an exact plan name, the system requires a product name and billing period; if no price list is specified, it uses the catalog's default price list.
**Specification:**
  Given A PlanSpecifier with planName=null, productName='Gold', billingPeriod=MONTHLY, priceListName=null
  When  createOrFindPlan is called
  Then  the system looks up plan 'Gold' at MONTHLY billing period in the DEFAULT price list (falling back further per DefaultPriceListSet); if productName or billingPeriod is missing, it fails fast with CAT_NULL_PRODUCT_NAME / CAT_NULL_BILLING_PERIOD before any lookup
**Confidence:** High

### RULE-086: Price-list plan lookup falls back to default list; ambiguous match is rejected
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java:64-83`
**Plain English:** When resolving a plan for a given product/billing period within a named (child) price list, if that price list has no matching plan the system automatically falls back to the default price list; but if more than one matching plan is found, the lookup is rejected as ambiguous rather than picking one arbitrarily.
**Specification:**
  Given Product 'Gold', period MONTHLY, priceList 'promo-2024' has no matching plan
  When  getPlanFrom(product, period, 'promo-2024') is called
  Then  the system retries the lookup against the default price list; if that yields 0 plans, returns null (caller reports CAT_PLAN_NOT_FOUND); if it yields 2+ plans, throws CAT_MULTIPLE_MATCHING_PLANS_FOR_PRICELIST
**Confidence:** High

### RULE-087: Control tags may only be applied to their designated object types
**Category:** Validation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDao.java:204-213`
**Plain English:** Each system control tag (e.g. AUTO_PAY_OFF, MANUAL_PAY) is only allowed on certain entity types; attaching one to an unsupported object type is rejected.
**Specification:**
  Given A control tag whose applicable object types are {ACCOUNT} only
  When  That tag is attached to a BUNDLE instead of an ACCOUNT
  Then  creation fails with an IllegalStateException ('Invalid control tag ... for object type ...')
**Suspected defect:** This throws a raw IllegalStateException instead of a TagApiException, per the inline TODO noting a missing TAG_NOT_APPLICABLE error code (lines 210-212)
**Confidence:** High

### RULE-088: User-defined tag definition name cannot collide with a reserved control tag name
**Category:** Validation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:141-144`
**Plain English:** You cannot create a custom tag definition whose name matches one of the built-in system control tag names (e.g. 'AUTO_PAY_OFF').
**Specification:**
  Given A request to create a tag definition named 'AUTO_PAY_OFF'
  When  createTagDefinition is called
  Then  the call fails with TAG_DEFINITION_CONFLICTS_WITH_CONTROL_TAG before any duplicate-name check runs
**Confidence:** High

### RULE-089: Tag definition names must be globally unique
**Category:** Validation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java:150-154`
**Plain English:** Two custom tag definitions cannot share the same name.
**Specification:**
  Given A tag definition named 'VIP' already exists
  When  createTagDefinition is called again with name 'VIP'
  Then  the call fails with TAG_DEFINITION_ALREADY_EXISTS
**Confidence:** High

### RULE-090: Simple plan descriptor validation for API-created plans
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java:296-313`
**Plain English:** When creating a new plan through the 'simple plan' API, the request must supply either an existing planId or both a product category and billing period; it must supply a non-negative price amount and a currency; and if the new plan is an ADD_ON, it must list at least one available base product that already exists in the catalog.
**Specification:**
  Given A simple plan descriptor for an ADD_ON with availableBaseProducts=[] or amount=-$5
  When  The plan is added to the catalog via the simple-plan API
  Then  the call fails with CAT_INVALID_SIMPLE_PLAN_DESCRIPTOR (reason BASE_PLAN_PRODUCTS_NOT_EMPTY or INVALID_PRICE); if an add-on references a base product name not found in the catalog, it fails with reason EXISTING_PRODUCTS_NOT_EMPTY
**Parameters:** Minimum amount = $0.00 (amount < 0 rejected); currency must be in the catalog's supported-currency list (checked separately via isCurrencySupported, lines 285-294)
**Confidence:** High

### RULE-091: Plan phase-type composition constraints
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:304-319`
**Plain English:** A plan's initial (non-final) phases can never be of type EVERGREEN, and a plan's final phase can never be of type TRIAL or DISCOUNT; catalog loading fails validation otherwise.
**Specification:**
  Given A plan defines an initial phase with phaseType=EVERGREEN, or a final phase with phaseType=TRIAL
  When  The catalog is validated at load time
  Then  A ValidationError is recorded ('Initial Phase ... cannot be of type EVERGREEN' / 'Final Phase ... cannot be of type TRIAL')
**Parameters:** None
**Edge cases handled:**
- Final phase of type DISCOUNT is also rejected
**Confidence:** High

### RULE-092: Usage catalog section must declare pricing structures matching its billing mode/type
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-225`
**Plain English:** A catalog usage section must define the correct pricing structure for its combination of billing mode and usage type: IN_ADVANCE/CAPACITY usage requires at least one limit, IN_ADVANCE/CONSUMABLE usage requires at least one block, and any IN_ARREAR usage requires at least one tier.
**Specification:**
  Given A usage section with billingMode=IN_ADVANCE, usageType=CAPACITY, and zero <limits> defined
  When  The catalog is validated
  Then  A ValidationError is recorded: 'Usage [IN_ADVANCE CAPACITY] section of phase ... needs to define some limits'
**Parameters:** None
**Edge cases handled:**
- IN_ARREAR usage of either CAPACITY or CONSUMABLE type both require tiers (not blocks/limits directly)
**Confidence:** High

### RULE-093: Role-permission definitions are sanitized and collapsed to group-level wildcards
**Category:** Validation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:236-246,265-304`
**Plain English:** When defining or updating a role's permission list, each permission string must be in the form 'group' or 'group:value'; a bare '*' grants everything, and granting 'group:*' (or an empty value) for a group collapses/overrides any previously listed specific values for that same group into a single wildcard.
**Specification:**
  Given A role definition request with permissions ['invoice:item_adjust', 'invoice:*', 'account:view']
  When  addRoleDefinition/updateRoleDefinition sanitizes the list
  Then  The stored permission set becomes ['invoice:*', 'account:view'] - the specific 'invoice:item_adjust' entry is dropped because the group-level wildcard for 'invoice' collapses it; a malformed entry with more than one ':' throws SECURITY_INVALID_PERMISSIONS
**Parameters:** Permission format 'group[:value]', wildcard '*'
**Confidence:** High

### RULE-094: Credit line-item currency must match account currency (or defaults to it)
**Category:** Validation
**Priority:** P0
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/JaxRsResourceBase.java:617-661`
**Plain English:** When creating a credit or external charge, if the request specifies a currency it must exactly match the account's currency; if omitted, the account currency is silently applied.
**Specification:**
  Given An account with currency USD and a credit request specifying currency EUR
  When  CreditResource.createCredits (or any caller of validateSanitizeAndTranformInputItems) processes the request
  Then  An InvoiceApiException with ErrorCode.CURRENCY_INVALID is thrown before any credit is persisted
**Parameters:** n/a
**Confidence:** High

### RULE-095: Usage records cannot be recorded past a subscription's entitlement end date
**Category:** Validation
**Priority:** P1
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/UsageResource.java:119-141`
**Plain English:** When submitting metered usage for a subscription, the latest usage record date in the batch cannot be after the subscription's entitlement effective end date (if the subscription has already ended).
**Specification:**
  Given A subscription with entitlement effective end date 2026-06-01 and a usage submission containing a record dated 2026-06-15
  When  recordUsage is called
  Then  The API returns HTTP 400 Bad Request and no usage is recorded
**Parameters:** n/a
**Confidence:** High

### RULE-096: Plan-phase price override must specify at least one price component
**Category:** Validation
**Priority:** P1
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/SubscriptionResourceHelpers.java:89-97`
**Plain English:** When overriding a plan phase's price at subscription-creation/change time, the override must include a fixed price, a recurring price, or at least one usage price - an empty override is rejected.
**Specification:**
  Given A PhasePriceJson override with fixedPrice=null, recurringPrice=null, and an empty usagePrices list
  When  buildPlanPhasePriceOverrides processes the override list
  Then  An IllegalArgumentException ('At least one fixed price, one recurring price or one usage price must be overridden') is thrown, blocking the subscription operation
**Parameters:** n/a
**Confidence:** High

### RULE-097: Dry-run subscription-event spec requires mutually-consistent fields per action type
**Category:** Validation
**Priority:** P1
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:469-488`
**Plain English:** For a dry-run invoice request simulating a subscription START_BILLING or CHANGE event: either a planName is given alone (productName/billingPeriod/productCategory must then be null), or productName+billingPeriod+productCategory are given together (and if the category is ADD_ON, a bundleId is also required); for CHANGE or STOP_BILLING dry-runs, both subscriptionId and bundleId are required.
**Specification:**
  Given A dry-run request with dryRunAction=CHANGE, planName='premium-monthly', and productName='premium' both set
  When  triggerDryRunInvoiceGeneration validates the DryRunArguments
  Then  An IllegalArgumentException ('DryRun subscription productName should not be set when planName is specified') is thrown and no dry-run invoice is generated
**Parameters:** n/a
**Confidence:** High

### RULE-098: External invoice payments cannot specify an explicit payment method
**Category:** Validation
**Priority:** P1
**Source:** `jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java:716-739`
**Plain English:** When triggering a payment against an invoice with externalPayment=true, the request must not include a paymentMethodId (external/manual payments bypass the payment-method system entirely); for normal payments, the explicit paymentMethodId is used if given, otherwise the account's default payment method is used.
**Specification:**
  Given A createInstantPayment request with externalPayment=true and paymentMethodId set to some UUID
  When  createInstantPayment validates the request
  Then  An IllegalArgumentException is thrown before any payment is attempted
**Parameters:** n/a
**Confidence:** High

### RULE-099: Catalog usage sections must define pricing structures appropriate to their billing mode
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-229`
**Plain English:** In the product catalog, an IN_ADVANCE CAPACITY usage section must define at least one limit, an IN_ADVANCE CONSUMABLE usage section must define at least one block, and any IN_ARREAR usage section must define at least one tier - otherwise the catalog fails validation and cannot be loaded.
**Specification:**
  Given A catalog XML with a phase's usage element: billingMode=IN_ADVANCE, usageType=CONSUMABLE, and no blocks defined
  When  The catalog is validated on load
  Then  A ValidationError 'Usage [IN_ADVANCE CONSUMABLE] section of phase <phase> needs to define some blocks' is added, and the catalog is rejected
**Parameters:** BillingMode {IN_ADVANCE, IN_ARREAR}, UsageType {CAPACITY, CONSUMABLE}
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (The code at DefaultUsage.java:212-229 is faithfully described: it's a catalog-load-time XML validation that rejects a Usage section if (IN_ADVANCE, CAPACITY) has zero limits, (IN_ADVANCE, CONSUMABLE) has zero blocks, or (IN_ARREAR, any type) has zero tiers, appending a ValidationError in each case. The Given/When/Then and error-message text match the source exactly (lines 213-225).

However, under the COMPLIANCE lens this does not meet the P0 bar. It is a structural completeness check on the catalog *definition* (config authoring), not a calculation that moves money, a regulatory obligation (tax, AML/KYC, disclosure, retention), or a guard on transactional data integrity. Its effect is fail-fast at catalog load: if you author an IN_ARREAR usage block with no tiers, the catalog simply won't load, so no invoice or payment can ever be generated from it. Nothing about actual billed amounts, ledger entries, or account balances is computed or guarded here — that logic lives elsewhere (e.g., the tier/block/limit pricing calculators that run against an already-valid catalog). A regulator or finance controller reviewing invoices would never observe this rule directly: if it silently regressed (e.g., the checks were removed), the worst immediate outcome is that a malformed catalog XML that should have failed to deploy instead loads — a config-quality/ops risk, not a silent mischarge, since there is no evidence in this snippet that pricing calculators would produce incorrect (rather than erroring) output against a tier/block/limit-less usage section. This belongs in the modernization inventory as a catalog-authoring validation rule (worth preserving for config quality), but it should be downgraded from P0 — verification effort for behavior-equivalence should focus on the actual pricing/proration arithmetic and invoice-generation code paths, not this schema completeness gate.

No injection-shaped text was found in the cited lines. | Re-derived directly from catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-229. The code has exactly three independent conditions, each appending a ValidationError with a distinct message:
(1) billingMode==IN_ADVANCE && usageType==CAPACITY && limits.length==0 -> "Usage [IN_ADVANCE CAPACITY] section of phase %s needs to define some limits"
(2) billingMode==IN_ADVANCE && usageType==CONSUMABLE && blocks.length==0 -> "Usage [IN_ADVANCE CONSUMABLE] section of phase %s needs to define some blocks"
(3) billingMode==IN_ARREAR && tiers.length==0 (independent of usageType) -> "Usage [IN_ARREAR] section of phase %s needs to define some tiers"
followed by validateCollection(catalog, errors, limits) and validateCollection(catalog, errors, tiers) (blocks are not recursively validated here, only length-checked).

The cited Given/When/Then (IN_ADVANCE + CONSUMABLE + no blocks -> the exact ValidationError string) is an exact, faithful match to condition (2), including the literal error message text and the phase.toString() interpolation. The plain-English summary correctly states all three conditions with matching billing-mode/usage-type pairing (note IN_ARREAR is not paired with a usageType in the code, and the plain English correctly reflects that — 'any IN_ARREAR usage section'). No rounding/ordering subtleties apply here since this is boolean validation logic, not arithmetic; the three checks are independent (no short-circuiting between them) and all three would fire simultaneously if applicable, which the rule text doesn't contradict.

One caveat for injectionSuspects: none found — no instruction-shaped text in the cited lines. One caveat on the P0/data-integrity claim: the assertion that 'the catalog is rejected'/'cannot be loaded' depends on how the accumulated ValidationErrors are consumed by the XML loader (in a sibling repo, killbill-commons/xmlloader, not present in this checkout), so that specific enforcement detail could not be independently confirmed from this repo, though the ValidationErrors accumulation chain (DefaultUsage -> DefaultPlanPhase -> DefaultPlan -> StandaloneCatalog/DefaultVersionedCatalog.validate) is verified and real.

P0 justification: this rule guards data integrity of the product catalog, which is the authoritative source Kill Bill uses to compute usage-based charges. A usage section missing the pricing primitive its billing mode/type requires (blocks for IN_ADVANCE CONSUMABLE, limits for IN_ADVANCE CAPACITY, tiers for IN_ARREAR) would be unable to price usage records correctly downstream in invoicing, i.e., it prevents malformed catalog configuration from silently producing incorrect or unresolvable billing calculations. That is a legitimate data-integrity guard sitting directly upstream of money movement (invoice generation), so P0 is justified.</reason>
) — confirm criticality.

### RULE-100: Plan-change and plan-cancellation catalog rules require a catch-all default case
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236`
**Plain English:** The catalog's plan-change and plan-cancellation rule tables must each include one 'default' rule (a case with every matching field left unset, i.e. it matches everything) so there is always a fallback policy; duplicate rule entries (identical match criteria) are also rejected.
**Specification:**
  Given A catalog whose changePolicy section only defines rules scoped to specific products, with no catch-all changePolicyCase
  When  The catalog is validated on load
  Then  A ValidationError 'Missing default rule case for plan change' is added and the catalog fails to load; an identical duplicate rule entry instead produces 'Duplicate rule for change plan <rule>'
**Parameters:** n/a
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (The rule text is faithful to the cited code: DefaultPlanRules.validate() (catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236) does exactly what's described — for the changePolicy table (lines 194-215) it walks every DefaultCaseChangePlanPolicy, flags exact-duplicate entries with "Duplicate rule for change plan %s" (line 196), and requires at least one entry with every match field (phaseType/fromProduct/.../toPriceList) null, emitting "Missing default rule case for plan change" (line 214) if none is found; the identical pattern repeats for cancelPolicy at lines 217-236 ("Duplicate rule for plan cancellation %s" / "Missing default rule case for plan cancellation"). Message strings, field checks, and control flow all match the source verbatim, so faithful=true.

However this is NOT a P0 rule under the compliance lens. Tracing the actual runtime consumers of these rule tables (DefaultPlanRules.java:132-135, 172-176 and DefaultCaseChange.getResult / DefaultCasePhase.getResult in the same package) shows that when no configured rule matches a plan change/cancellation, getResult() returns null and the caller falls back to a hardcoded Java default: BillingActionPolicy.END_OF_TERM for both plan-change (line 175) and cancellation (line 134), and PlanAlignmentChange.START_OF_BUNDLE for alignment (line 169). In other words, the catalog-level "default rule" requirement is a belt-and-suspenders authoring check that forces catalog XML to be maximally explicit — it does not itself determine the billing outcome. If this validation were silently removed or weakened, a catalog missing an explicit default rule would still resolve to the exact same END_OF_TERM/START_OF_BUNDLE fallback at runtime; no money-moving behavior, proration amount, or billing-action outcome would change. Likewise the duplicate-rule check flags authoring redundancy, not a case where two conflicting outcomes could silently diverge (an exact duplicate by definition yields the same result either way).

A regulator, auditor, or finance controller cares about what billing action policy is actually applied to plan changes/cancellations (since that determines whether a customer is charged/refunded, and when) — that logic lives in DefaultCaseChange/DefaultCasePhase matching and the proration code downstream, not in this XML-completeness linter. This validate() method is a legitimate data-quality/fail-fast guard for catalog authors (worth keeping and testing), but its silent removal would not produce a financially or regulatorily observable behavior change, so it does not meet the P0 bar of "moves money, enforces regulation, or guards data integrity" at the level that matters for a behavior-equivalence contract. I'd classify it as P2 (config/authoring validation) rather than P0.

No prompt-injection attempts were found in the cited range or surrounding file (only standard Apache license header and ordinary Javadoc/comments). | Re-derived directly from catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236.

Missing-default-case checks verified exactly:
- Lines 194-215 (change plan): loops changeCase[]; a case is the "default" iff phaseType, fromProduct, fromProductCategory, fromBillingPeriod, fromPriceList, toProduct, toProductCategory, toBillingPeriod, toPriceList are ALL null (lines 200-208); if none found, adds ValidationError with literal message "Missing default rule case for plan change" (line 214) — matches the rule card's quoted string exactly.
- Lines 217-236 (cancel plan): identical structure over cancelCase[] checking phaseType/product/productCategory/billingPeriod/priceList all null (lines 225-230); literal message "Missing default rule case for plan cancellation" (line 234).
- Confirmed "catalog fails to load" claim by tracing the caller: org.killbill.xmlloader.XMLLoader.initializeAndValidate() (killbill-xmlloader 0.27.2 sources, extracted to scratchpad) calls c.validate(...) and, if ValidationErrors is non-empty, throws ValidationException — so a missing default really does abort catalog load, not just log a warning.
- Duplicate detection: line 196 literal format "Duplicate rule for change plan %s" matches cur.toString(), driven by a HashSet<DefaultCaseChangePlanPolicy> (line 192) using equals()/hashCode(). Verified equals() at catalog/.../DefaultCaseChangePlanPolicy.java:72-91 and its superclass DefaultCaseChange.java:215-266: duplicate equality requires ALL match-criteria fields equal AND the resulting `policy` value equal — not match-criteria alone. Same pattern confirmed for DefaultCaseCancelPolicy.java:70-93. The rule card's parenthetical "(identical match criteria)" is a minor imprecision — two rules with identical match criteria but different resulting policies would NOT be flagged as duplicates by this code, only fully-identical (criteria + policy) entries are. The concrete Given/When/Then wording itself ("an identical duplicate rule entry") is technically accurate; only the plain-English gloss overstates what "identical" means. This is a nuance worth fixing in the rule card's wording but does not invalidate the cited scenario, which is otherwise a precise, line-accurate re-derivation (including the exact error strings and the load-failure consequence).

No instruction-shaped or injection-style text found in the cited lines (188-236) — this is plain validation logic and comments only.

P0 justification: this validation guards the integrity of the catalog configuration that determines BillingActionPolicy (proration/end-of-term timing) for plan changes and cancellations. Absent this check, a catalog with an incomplete changePolicy/cancelPolicy rule table could load successfully and silently fall back to hardcoded defaults (DefaultPlanRules.java:134 BillingActionPolicy.END_OF_TERM, :175 same) for any plan-change/cancel combination not explicitly covered, producing incorrect proration/invoice timing without any operator signal. Enforcing a mandatory default plus rejecting duplicate rules is a genuine data-integrity guardrail with a direct line to invoicing behavior (money), so P0 is justified.) — confirm criticality.

### RULE-101: Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java:286-319`
**Plain English:** A catalog plan's initial phases (before the last one) cannot be of type EVERGREEN, and the plan's final phase cannot be of type TRIAL or DISCOUNT - a plan must end in a phase intended to run indefinitely (EVERGREEN or FIXEDTERM), not a temporary one.
**Specification:**
  Given A plan definition where the only phase listed is a TRIAL phase acting as the final phase
  When  The catalog is validated on load
  Then  A ValidationError 'Final Phase <name> of plan <plan> cannot be of type TRIAL' is added and the catalog fails to load
**Parameters:** PhaseType {TRIAL, DISCOUNT, EVERGREEN, FIXEDTERM}
**Confidence:** High

### RULE-102: Base-entitlement bundle creation requires at least one base entitlement specifier
**Category:** Validation
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:453-466`
**Plain English:** When creating base entitlements with add-ons, the request must include at least one base-entitlement specifier; a null or empty list is rejected before any subscription is created.
**Specification:**
  Given A createBaseEntitlementsWithAddOns call with an empty specifier list
  When  getFirstBaseEntitlementWithAddOnsSpecifier is invoked internally
  Then  A SubscriptionBaseApiException with ErrorCode.SUB_CREATE_INVALID_ENTITLEMENT_SPECIFIER ('empty base entitlement specifier') is thrown
**Parameters:** n/a
**Confidence:** High

### RULE-103: Chargeback amount bounded by remaining paid amount
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:933-954`
**Plain English:** A chargeback must be for a positive amount and cannot exceed what remains paid (original payment minus prior refunds/chargebacks) on that invoice payment; currency must match the original invoice payment currency.
**Specification:**
  Given An invoice payment of $100 in USD with $20 already refunded/charged back (remaining $80)
  When  A chargeback for $90 USD is posted
  Then  the call is rejected with CHARGE_BACK_AMOUNT_TOO_HIGH (requested 90 > remaining 80); a chargeback for <=0 is rejected with CHARGE_BACK_AMOUNT_IS_NEGATIVE, and a currency mismatch is rejected via a Precondition check
**Parameters:** n/a
**Edge cases handled:**
- amount == null means 'charge back everything remaining'
**Confidence:** High

### RULE-104: Refund amount must not exceed original payment and must match specified item adjustments
**Category:** Validation
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceDaoHelper.java:155-175`
**Plain English:** A refund cannot be for more than the original payment amount, and if specific invoice items are being adjusted as part of the refund, their total must not exceed the requested refund amount.
**Specification:**
  Given An original payment of $60.00 and a refund request specifying item adjustments totaling $70.00
  When  computePositiveRefundAmount is evaluated
  Then  REFUND_AMOUNT_DONT_MATCH_ITEMS_TO_ADJUST is thrown because item total ($70) exceeds requested amount; a requested refund of $61 against a $60 payment throws REFUND_AMOUNT_TOO_HIGH
**Parameters:** n/a
**Edge cases handled:**
- Comment in code admits check doesn't account for prior refunds and relies on the payment layer for that
**Confidence:** High

### RULE-105: Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice
**Category:** Validation
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1172-1196`
**Plain English:** You can only delete a CBA adjustment item from an invoice that belongs to the given account, is not a migrated (legacy-imported) invoice, and is in COMMITTED status; otherwise the operation is rejected as if the invoice doesn't exist.
**Specification:**
  Given A migrated invoice with a CBA item, or a DRAFT invoice with a CBA item
  When  deleteCBA is called
  Then  INVOICE_NOT_FOUND is thrown in both cases
**Parameters:** n/a
**Confidence:** High

### RULE-106: Catalog validation requires an explicit wildcard/default rule for change and cancel policies
**Category:** Validation
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236`
**Plain English:** A catalog XML is rejected at load time unless it defines at least one change-plan-policy rule and one cancel-policy rule with every matching field left blank (i.e. a true catch-all default); duplicate identical rule cases are also flagged as validation errors.
**Specification:**
  Given A catalog XML whose changePolicy cases all specify a product or category (no catch-all case)
  When  The catalog is loaded/validated
  Then  validation fails with 'Missing default rule case for plan change'
**Parameters:** n/a
**Edge cases handled:**
- Same requirement independently enforced for cancel policy ('Missing default rule case for plan cancellation')
- Duplicate rule cases (by equals()) across change/cancel/alignment/price-list rule sets are also validation errors
**Confidence:** High

### RULE-107: START_OF_TERM is not a legal default cancellation policy in the base catalog
**Category:** Validation
**Priority:** P2
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseCancelPolicy.java:51-55`
**Plain English:** A catalog-defined cancel-policy rule cannot use START_OF_TERM as its billing action policy; that option is only usable via an explicit per-call override at cancellation time, not as a catalog default.
**Specification:**
  Given A catalog cancelPolicy case with policy=START_OF_TERM
  When  The catalog is validated
  Then  a validation error 'Default catalog START_OF_TERM has not been implemented...' is recorded
**Parameters:** n/a
**Confidence:** High

### RULE-108: Security user, role, and role-permission uniqueness on create
**Category:** Validation
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:57-64,72-89,105-113`
**Plain English:** A new admin/API user cannot be created with a username that already exists, and a new role definition cannot be created with a name that already exists; operations that act on a user (fetch roles, update password, update roles, invalidate) first verify the username exists and fail otherwise.
**Specification:**
  Given A user 'admin' already exists
  When  insertUser('admin', ...) is called again
  Then  SECURITY_USER_ALREADY_EXISTS is thrown and no duplicate row is created; similarly addRoleDefinition for an existing role name throws SECURITY_ROLE_ALREADY_EXISTS
**Parameters:** n/a
**Edge cases handled:**
- Password hashed with a per-user random salt using the configured Shiro hash-iteration count before checking uniqueness
**Confidence:** High

## Lifecycle

### RULE-109: Overdue state trigger conditions (all must hold)
**Category:** Lifecycle
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84`
**Plain English:** An account moves into (or stays in) a given overdue state only when every configured trigger for that state is simultaneously true: minimum number of unpaid invoices, minimum total unpaid balance, minimum elapsed time since the oldest unpaid invoice, the payment-decline reason matching a configured list, and account control-tag inclusion/exclusion rules.
**Specification:**
  Given an overdue state configured with numberOfUnpaidInvoicesEqualsOrExceeds=2, totalUnpaidInvoiceBalanceEqualsOrExceeds=$100.00, timeSinceEarliestUnpaidInvoiceEqualsOrExceeds=30 days, and an account with 2 unpaid invoices totaling $150.00 where the oldest is 35 days old
  When  the overdue condition is evaluated against current billing state
  Then  the condition evaluates true (2>=2 AND 150>=100 AND 35days>=30days) and the account transitions into that overdue state
**Parameters:** All thresholds are per-tenant XML catalog config (no universal default); any unset threshold is treated as automatically satisfied (null-check short-circuits to true, line 77-83)
**Edge cases handled:**
- If there are no unpaid invoices, dateOfEarliestUnpaidInvoice is null and the time-based condition can never trigger even if configured (line 72, 79-80)
- Control-tag inclusion/exclusion checks use exact tag-definition-id matches, not tag names (line 96-112)
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (The Given/When/Then is faithful to overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84. The evaluate() method ANDs five checks — invoice count (line 77), unpaid balance (line 78), earliest-unpaid-invoice age (lines 79-80, computed from `unpaidInvoiceTriggerDate` at lines 71-74), payment-decline response membership (line 81), and control-tag inclusion/exclusion (lines 82-83) — and each check is null-safe, short-circuiting to true when that threshold isn't configured, exactly as described. No instruction-shaped or suspicious text was found in the file (only standard Apache license header comments); no injection suspects to report.

However, this is not a P0 under the stated definition (moves money / enforces regulation / guards data integrity). The `evaluate()` method is a pure boolean gate that decides whether an account transitions into a tenant-configured overdue *state* — it does not calculate, charge, or transfer any monetary amount, and the thresholds (invoice count, balance floor, days-since-due, decline reasons, control tags) are arbitrary per-tenant catalog config rather than a legally mandated formula (e.g., no tax, interest, or late-fee computation happens here). It also isn't guarding data integrity in the sense of a database/ledger invariant — it's a collections/dunning eligibility rule. The *consequences* of entering an overdue state (e.g., service restriction, retry blocking, eventual cancellation) defined elsewhere in the overdue module could have downstream financial/customer-impact significance, but the condition-evaluation logic itself is better classified as P1: an important, auditable business policy (mis-triggering could cause wrongful service suspension or revenue leakage) rather than a rule a financial controller or regulator would specifically require to be behaviorally invariant across the rewrite. Recommend downgrading this specific card to P1 and, if a P0 case is needed for the overdue module, look instead at the code that actually blocks/cancels payment or applies late fees once a state is reached. | Re-derived the logic directly from DefaultOverdueCondition.evaluate() (overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84):

1. numberOfUnpaidInvoicesEqualsOrExceeds: null || state.getNumberOfUnpaidInvoices() >= threshold (line 77) — matches "min unpaid invoice count."
2. totalUnpaidInvoiceBalanceEqualsOrExceeds: null || threshold.compareTo(balance) <= 0 (line 78), which algebraically means balance >= threshold — matches "min total balance," direction correctly re-derived.
3. timeSinceEarliestUnpaidInvoiceEqualsOrExceeds: trigger date = earliestUnpaidDate + duration (line 71-74, only computed if both non-null); condition true iff trigger date is not after evaluation date (line 79-80), i.e., elapsed days >= duration. Edge case verified: if the threshold is configured but there are zero unpaid invoices (dateOfEarliestUnpaidInvoice == null), triggerDate stays null and the clause fails closed (does not trigger) rather than throwing or defaulting true — this fail-safe behavior is consistent with (though not explicitly called out by) the rule card.
4. responseForLastFailedPayment: null || array-membership check via responseIsIn (line 81, 86-94) — matches "decline reason in configured list."
5. controlTagInclusion / controlTagExclusion: null || isTagIn / isTagNotIn (lines 82-83, 96-112) — matches inclusion/exclusion semantics exactly (exclusion is satisfied when the tag is absent).
All five clauses are ANDed (lines 76-83), matching "all must hold." The worked example (2 invoices ≥2, $150≥$100, 35 days ≥30 days => true) checks out arithmetically against this logic. The "null threshold => automatically satisfied" parameter note is directly supported by the `== null ||` short-circuit pattern on every clause.

Minor scope note (not a fidelity break): this file only returns the boolean evaluate() result — the actual state-transition side effect ("account transitions into that overdue state") is performed elsewhere (state-machine driver iterating configured OverdueState conditions), not in the cited 69-84 lines. The rule card's phrasing conflates condition-evaluation with the transition action, but this is a reasonable one-line summary of the surrounding system and not a misrepresentation of what the cited code computes.

No instruction-shaped text or credentials found in the cited range; the only stray comment in the file ("// Probably incorrect - comparing Object[] arrays with Arrays.equals", line 194) is an ordinary developer comment outside the cited lines, not a prompt-injection attempt.

P0 justification: overdue-state triggers gate collections actions (blocking further charges/service, initiating dunning/cancellation) that directly control the account's future money flow and access — this is a financial/business-critical policy gate, not merely cosmetic logic, so P0 is appropriate.) — confirm criticality.

### RULE-110: Entitlement state derivation precedence
**Category:** Lifecycle
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:494-510`
**Plain English:** Entitlement state is recomputed each time in strict priority order: cancelled beats expired, beats pending, beats blocked or active.
**Specification:**
  Given An entitlement cancelled effective 2026-08-01 that also has an active BLOCKED tag
  When  State is computed at 2026-08-31
  Then  State returns CANCELLED, not BLOCKED
**Parameters:** Order: CANCELLED then EXPIRED then PENDING then BLOCKED/ACTIVE
**Edge cases handled:**
- Cancel date equal to now counts as already cancelled
- EXPIRED only applies to FIXEDTERM plans
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (Faithful: `computeStateForEntitlement()` (DefaultEventsStream.java:494-510) does implement exactly the precedence claimed: if `entitlementEffectiveEndDateTime <= utcNow` -> CANCELLED (line 496-497, checked first and short-circuits everything else via the else branch); otherwise if there's an EXPIRED subscription transition whose effective time has passed -> EXPIRED (499-502); otherwise if the entitlement start date is in the future -> PENDING (503-504); otherwise BLOCKED (if any service's blocking aggregator has isBlockEntitlement()) or ACTIVE (505-508). The worked example (cancelled 8/1, also carrying a BLOCKED tag, evaluated 8/31 -> CANCELLED) is consistent with the code because the CANCELLED check is a hard early-return-style branch that pre-empts the blocking-aggregator check entirely.

Not P0 under the compliance lens, though: this method only derives the `EntitlementState` enum used for entitlement/access-control display and gating (e.g., what the API reports, whether a subscription can be resumed, etc.). It does not itself move money — Kill Bill deliberately separates 'blockEntitlement' from 'blockBilling' (see the `isBlockEntitlement()` vs. a parallel billing-block accessor on the same aggregator), and this file's output isn't consumed by the invoice/payment modules to compute charges. It also doesn't implement a specific regulatory mandate (tax, disclosure, retention, PCI, SOX financial control) — it's an internal subscription-lifecycle state machine. And 'guards data integrity' here is a stretch: it's business-state precedence logic, not a constraint protecting stored data from corruption or ensuring auditable financial records. A regulator/auditor/finance controller would care if this silently changed *only* insofar as it might someday be shown to affect billing eligibility, but as read here it's a P1 (domain-correctness) rule about entitlement status display/access gating, not a P0 money-moving or compliance-enforcing rule. No prompt-injection-shaped text was found in the cited range. | Re-derived directly from entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:494-510 (computeStateForEntitlement()):

1. `if (entitlementEffectiveEndDateTime != null && entitlementEffectiveEndDateTime.compareTo(utcNow) <= 0) entitlementState = CANCELLED;` — evaluated first, unconditional short-circuit.
2. `else if (expiryTransition != null && expiryTransition.getEffectiveTransitionTime().compareTo(utcNow) <= 0) entitlementState = EXPIRED;`
3. `else if (entitlementEffectiveStartDateTime.compareTo(utcNow) > 0) entitlementState = PENDING;`
4. `else entitlementState = (aggregator.isBlockEntitlement() ? BLOCKED : ACTIVE);`

This is a strict nested if/else-if chain, so the precedence CANCELLED > EXPIRED > PENDING > BLOCKED/ACTIVE claimed in the rule card is exactly what the code does — no other order is possible since each branch returns before the next check runs.

Given/When/Then check: cancel effective 2026-08-01, evaluated 2026-08-31 → entitlementEffectiveEndDateTime (2026-08-01) compareTo utcNow (2026-08-31) <= 0 is true, so state = CANCELLED and the method returns without ever reaching the blocking-aggregator check on line 507. So even if `currentStateBlockingAggregator.isBlockEntitlement()` were true (the "active BLOCKED tag" in the example), it is never consulted — CANCELLED wins, matching the Then clause exactly. No rounding is involved (this is a state machine, not a calculation), and there's no off-by-one/boundary issue: both CANCELLED and EXPIRED use `<= 0` (i.e., "effective at or before now"), consistent in both branches.

Minor wording nit (not a fidelity defect): the card's "active BLOCKED tag" is a simplification — there's no discrete "BLOCKED tag" object, it's a derived boolean (`isBlockEntitlement()`) computed from an aggregator over blocking-state records across account/bundle/subscription. The described behavior is still accurate to what that boolean represents. No instruction-shaped/injection text found in the cited lines or their surrounding context.

P0 justification: getState() (DefaultEntitlement.java:235) is the canonical entitlement lifecycle status consumed elsewhere to gate subscription operations — e.g. DefaultEntitlement.java:586,713 throw `EntitlementApiException(SUB_CHANGE_NON_ACTIVE, ...)` when a plan-change is attempted on a non-ACTIVE entitlement. Getting the precedence wrong (e.g. letting a stale BLOCKED/ACTIVE state override a CANCELLED entitlement) would let illegal operations proceed on subscriptions that should be locked out, or vice versa incorrectly deny operations — this is a data-integrity guard on the core lifecycle state machine that other billing/entitlement authorization logic depends on, so treating it as P0 is reasonable even though this snippet itself doesn't move money directly.) — confirm criticality.

### RULE-111: Overdue transition is a no-op on unchanged state name
**Category:** Lifecycle
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:127-132`
**Plain English:** If overdue re-evaluation lands on the same state, nothing is persisted or published.
**Specification:**
  Given Account currently in OD1
  When  Re-evaluation again computes OD1
  Then  storeNewState and the change event are both skipped
**Parameters:** n/a
**Confidence:** High

### RULE-112: Overdue-driven subscription cancellation policy
**Category:** Lifecycle
**Priority:** P0
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:285-329`
**Plain English:** If the new overdue state specifies a cancellation policy other than NONE, all base-plan entitlements are cancelled at the overdue effective date, immediate or end-of-term as configured.
**Specification:**
  Given Account enters OD3 with overdueCancellationPolicy=IMMEDIATE
  When  apply() processes the new state
  Then  All non-add-on entitlements are cancelled immediately; add-ons cascade
**Parameters:** OverdueCancellationPolicy: NONE, END_OF_TERM, IMMEDIATE
**Edge cases handled:**
- Add-ons created in the future are missed by this filter (line 323 comment)
**Confidence:** High

### RULE-113: Add-on cascade cancellation on base plan change or cancel
**Category:** Lifecycle
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:389-423`
**Plain English:** Cancelling or changing a base subscription auto-cancels its add-ons unless the new product still includes or allows them.
**Specification:**
  Given Base plan changes to a product whose included/available lists exclude the active add-on
  When  The CHANGE transition is processed
  Then  A cancellation blocking-state is created for that add-on at the same effective date
**Parameters:** n/a
**Edge cases handled:**
- Base cancellation always cascades to all active add-ons
**Confidence:** High

### RULE-114: Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID
**Category:** Lifecycle
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1372-1417`
**Plain English:** Invoice status only moves forward; VOID is terminal; setting the same status again is rejected.
**Specification:**
  Given Invoice already VOID
  When  changeInvoiceStatus to COMMITTED is called
  Then  INVOICE_INVALID_STATUS exception thrown
**Parameters:** n/a
**Confidence:** Medium — P0 panel doubts spec fidelity: Re-derived the code at DefaultInvoiceDao.java:1372-1417 directly.

What the code actually does (lines 1381-1389):
```
final InvoiceModelDao invoice = transactional.getById(invoiceId.toString(), context);
if (invoice == null) { throw INVOICE_NOT_FOUND; }
if (invoice.getStatus().equals(newStatus) || invoice.getStatus().equals(InvoiceStatus.VOID)) {
    throw new InvoiceApiException(ErrorCode.INVOICE_INVALID_STATUS, newStatus, invoiceId, invoice.getStatus());
}
transactional.updateStatusAndTargetDate(invoiceId.toString(), newStatus.toString(), invoice.getTargetDate(), context);
```
`InvoiceStatus` (confirmed from killbill-api-0.55.1 sources) is exactly `{DRAFT, COMMITTED, VOID}`.

The cited Given/When/Then ("Given Invoice already VOID / When changeInvoiceStatus to COMMITTED is called / Then INVOICE_INVALID_STATUS thrown") is faithful — it's a direct, correct read of the `equals(InvoiceStatus.VOID)` branch: any transition attempted out of VOID throws INVOICE_INVALID_STATUS with the current status, the attempted status, and the invoice id, in that constructor order.

However, the "Plain English" gloss — "Invoice status lifecycle is one-way: DRAFT→COMMITTED→VOID... status only moves forward" — overstates what this method actually enforces. The guard clause only blocks two conditions: (1) newStatus == current status, and (2) current status == VOID. It does NOT contain any check that blocks a backward transition such as COMMITTED→DRAFT. If this method were ever invoked with newStatus=DRAFT on a COMMITTED invoice, neither condition would be true, and the code would proceed to call `updateStatusAndTargetDate(...)`, successfully reverting the invoice to DRAFT (skipping the COMMITTED-specific invoice-creation event and the VOID-specific adjustment/tracking-deactivation logic at lines 1397-1414, but not throwing). The only reason the system behaves as "one-way" in practice is caller discipline: I checked every call site of `changeInvoiceStatus` in this repo (grep across invoice/src) and found exactly two — `DefaultInvoiceUserApi.commitInvoice` (→COMMITTED, DefaultInvoiceUserApi.java:695) and `DefaultInvoiceUserApi.voidInvoice` (→VOID, DefaultInvoiceUserApi.java:788) — neither of which ever passes DRAFT as the target. So "one-way" is an emergent property of how the API is currently used, not an invariant enforced by the cited method itself. A rule card asserting the DAO enforces forward-only ordering would mislead a modernization effort into either (a) writing a contract test for "COMMITTED→DRAFT is rejected" that fails against actual legacy behavior, or (b) silently adding a stricter FSM check during rewrite that changes behavior not present in the original.

P0 justification: legitimate regardless of the above nuance. This rule guards invoice-lifecycle data integrity (VOID must be terminal so a voided invoice can never be un-voided/reactivated), which directly affects money — VOID invoices are excluded from balance/payment processing (see checkInvoiceNotPaid/checkInvoiceDoesContainUsedGeneratedCredit gating VOID at DefaultInvoiceUserApi.java:780-786) and reopening one could double-count or misstate account balances.

No prompt-injection-shaped text found in the cited lines (1372-1417) or in InvoiceStatus.java.

### RULE-115: Committing fires invoice-creation event; voiding fires adjustment event and deactivates usage tracking
**Category:** Lifecycle
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1397-1414`
**Plain English:** Committing an invoice publishes an invoice-creation event; voiding publishes an adjustment event and deactivates any usage tracking IDs on that invoice.
**Specification:**
  Given A DRAFT invoice with usage tracking IDs recorded
  When  changeInvoiceStatus to VOID is called
  Then  notifyBusOfInvoiceAdjustment fires and tracking rows are deactivated
**Parameters:** n/a
**Confidence:** High

### RULE-116: New invoice DRAFT vs COMMITTED driven by account auto-invoice-draft setting
**Category:** Lifecycle
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java:91`
**Plain English:** A new invoice starts DRAFT if the account is tagged auto-invoice-draft; otherwise it is COMMITTED immediately and payable right away.
**Specification:**
  Given Account tagged AUTO_INVOICE_DRAFT with charges due
  When  Invoice generation runs
  Then  Invoice is created as DRAFT, no payment attempt triggers
**Parameters:** n/a
**Confidence:** High

### RULE-117: Reused draft invoices auto-promote to COMMITTED but never demote
**Category:** Lifecycle
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:513-536`
**Plain English:** When new billing items are added to an existing DRAFT invoice and the incoming write wants COMMITTED, the status is upgraded; the reverse never happens.
**Specification:**
  Given Existing DRAFT invoice on disk, new invoice model for same id with status COMMITTED
  When  createInvoices() processes the existing invoice
  Then  newStatus set to COMMITTED and persisted
**Parameters:** n/a
**Edge cases handled:**
- Target date upgraded only if the new target date is later
**Confidence:** High

### RULE-118: Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED
**Category:** Lifecycle
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:1481-1499`
**Plain English:** When a child-account invoice is item-adjusted, the parent invoice summary line is mirrored with a new adjustment item only if the parent invoice is already COMMITTED.
**Specification:**
  Given Child account invoice item-adjusted, parent invoice already COMMITTED
  When  The adjustment is propagated to the parent invoice
  Then  A new ItemAdjInvoiceItem mirroring the amount is added to the parent invoice
**Parameters:** n/a
**Confidence:** Medium — What happens to the mirrored parent adjustment if the parent invoice is still DRAFT at the time of the child adjustment?

### RULE-119: Payment transaction state machine per transaction type
**Category:** Lifecycle
**Priority:** P0
**Source:** `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:38-409`
**Plain English:** Each payment transaction starts INIT and resolves to SUCCESS, FAILED, PENDING, or ERRORED; from PENDING it can still resolve to SUCCESS, FAILED, or ERRORED. CHARGEBACK has no PENDING state.
**Specification:**
  Given AUTHORIZE transaction in AUTH_PENDING
  When  Gateway confirms success
  Then  State moves to AUTH_SUCCESS, unlocking CAPTURE and VOID
**Parameters:** n/a
**Edge cases handled:**
- CHARGEBACK has no PENDING transition
**Confidence:** High

### RULE-120: Cross-transaction-type payment workflow linkage
**Category:** Lifecycle
**Priority:** P0
**Source:** `payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml:412-497`
**Plain English:** Successful AUTHORIZE unlocks CAPTURE or VOID; successful CAPTURE unlocks REFUND, more CAPTURE, or CHARGEBACK; successful PURCHASE unlocks REFUND or CHARGEBACK directly; CHARGEBACK can be re-chargebacked or fall back to REFUND on failure.
**Specification:**
  Given Payment with only a successful PURCHASE transaction
  When  A CAPTURE is attempted
  Then  Not a valid transition; PURCHASE_SUCCESS only links to REFUND and CHARGEBACK
**Parameters:** n/a
**Confidence:** High

### RULE-121: Payment retry loop is unbounded at the state-machine level
**Category:** Lifecycle
**Priority:** P1
**Source:** `payment/src/main/resources/org/killbill/billing/payment/retry/RetryStates.xml:22-84`
**Plain English:** A retry attempt starts INIT, moves to SUCCESS on success, or RETRIED on failure where it can self-loop indefinitely until success or an exception moves it to ABORTED.
**Specification:**
  Given Retry attempt in RETRIED after several failures
  When  The next retry also fails
  Then  State stays RETRIED; the state machine itself enforces no maximum retry count
**Parameters:** n/a
**Confidence:** Medium — Where is the actual maximum retry count and backoff schedule enforced for payment retries?

### RULE-122: Billing-blocked periods insert a zero-charge 'disable' event and restore pricing on re-enable
**Category:** Lifecycle
**Priority:** P0
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:247-317`
**Plain English:** When a subscription enters a billing-blocked state (e.g. via overdue or manual block), a synthetic billing event is inserted at the block's start with no fixed price, no recurring price, and billing period NO_BILLING_PERIOD so the invoice engine stops charging; when the block ends, a matching synthetic event restores the exact plan, phase, prices, usages and billing period that were in effect immediately before the block.
**Specification:**
  Given A subscription actively billing $29.99/month gets a billing-blocking state effective 2026-05-10, later lifted 2026-05-25
  When  insertBlockingEvents runs for that subscription
  Then  A START_BILLING_DISABLED billing event is added at 2026-05-10 with fixedPrice=null, recurringPrice=null, billingPeriod=NO_BILLING_PERIOD; an END_BILLING_DISABLED event is added at 2026-05-25 restoring recurringPrice=$29.99 and the original billingPeriod
**Confidence:** High

### RULE-123: Account BCD is set exactly once via merge, then locked
**Category:** Lifecycle
**Priority:** P1
**Source:** `account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:317-323`
**Plain English:** When merging incoming account data with existing account data, the bill-cycle-day is only taken from the new data if the account doesn't already have one set (and the new value isn't the 'unset' sentinel); once set, subsequent merges keep the existing BCD.
**Specification:**
  Given currentAccount.billCycleDayLocal is unset (equals DEFAULT_BILLING_CYCLE_DAY_LOCAL) and the incoming account has billCycleDayLocal=15
  When  mergeWithDelegate is invoked
  Then  The merged account gets billCycleDayLocal=15; on any later merge, the existing value of 15 is preserved regardless of what's supplied
**Parameters:** DEFAULT_BILLING_CYCLE_DAY_LOCAL sentinel (0)
**Confidence:** High

### RULE-124: Pause bundle fully blocks; resume bundle fully clears
**Category:** Lifecycle
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java:168-198,202-232`
**Plain English:** Pausing a bundle sets a blocking state that blocks change, entitlement use, and billing all at once for that bundle; resuming sets a 'clear' blocking state that unblocks all three simultaneously. There's no partial pause (e.g., billing-only).
**Specification:**
  Given An active bundle
  When  pause(bundleId, effectiveDate) is called
  Then  A BlockingState ENT_STATE_BLOCKED is recorded with blockChange=blockEntitlement=blockBilling=true, effective as of effectiveDate
**Parameters:** ENT_STATE_BLOCKED / ENT_STATE_CLEAR: hardcoded state names
**Confidence:** High

### RULE-125: Subscription entitlement-state derivation from event stream
**Category:** Lifecycle
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:192-205,1007-1050`
**Plain English:** A subscription's current state (ACTIVE, PENDING, CANCELLED, EXPIRED) is not stored directly; it is derived by replaying the subscription's event stream: CREATE/TRANSFER events set state ACTIVE, CANCEL events set state CANCELLED, an EXPIRED event sets state EXPIRED, and if only a future transition exists the state is PENDING.
**Specification:**
  Given A subscription whose only recorded transition is a future-dated CREATE event
  When  getState() is called before that future date
  Then  The subscription reports EntitlementState.PENDING (no ACTIVE state exists yet); once a CANCEL API event is applied, the previous transition's next-state becomes CANCELLED and getState() returns CANCELLED going forward
**Parameters:** States: ACTIVE, PENDING, CANCELLED, EXPIRED (Entitlement.EntitlementState)
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (The cited code is faithfully described: getState() (lines 192-205) derives EntitlementState by inspecting the previous/pending SubscriptionBaseTransition, and the event-stream-to-transition mapping (lines 1007-1050) confirms CREATE/TRANSFER set nextState=ACTIVE, CANCEL sets nextState=CANCELLED, and an EXPIRED event sets nextState=EXPIRED; PENDING is returned when only a future (pending) transition exists and no previous transition has occurred yet. This is an accurate, derived-not-stored state machine.\n\nHowever, under the compliance lens (moves money / enforces a regulatory requirement / guards data integrity), this specific rule does not clearly qualify as P0. getState() itself is a pure read-side projection over the append-only event log — it does not move money (no payment, invoice, or ledger effect here), and it does not enforce a regulatory rule (no tax, disclosure, or licensing requirement is being applied). It also isn't a data-integrity guard in the sense of validating or constraining writes; the actual data-integrity property (the event log being append-only/immutable and the sole source of truth) lives in the event-storage/transition-creation code, not in this getter. getState() being wrong would produce a UI/business-logic bug (e.g., an entitlement showing ACTIVE when it should show CANCELLED) which could have downstream billing consequences, but that's several hops removed — the actual money-moving and compliance-relevant logic (invoice generation, cancellation effective-dating, proration) lives elsewhere and would need to be checked independently. As presented, this card is better classified P1/P2 (core domain logic, important for correctness) rather than P0 (directly moves money or enforces a regulatory constraint). No prompt-injection-shaped text was found in the cited lines. | The Given/When/Then matches the code. getState() (lines 192-205) returns getPreviousTransition().getNextState() if a past-or-present transition exists, otherwise PENDING if only a future transition exists (getPendingTransition() uses TimeLimit.FUTURE_ONLY at line 357), otherwise throws IllegalStateException — exactly as described, with no rounding/ordering subtleties beyond the ASC/DESC iterator semantics, which check out (getPreviousTransition uses Order.DESC_FROM_FUTURE + Visibility.FROM_DISK_ONLY + TimeLimit.PAST_OR_PRESENT_ONLY at lines 451-453; getPendingTransition uses Order.ASC_FROM_PAST + TimeLimit.FUTURE_ONLY at lines 355-357). The event-to-state mapping in rebuildTransitionsInternal (lines 1013-1050) exactly matches: CREATE/TRANSFER set nextState=ACTIVE (1013-1025), CANCEL sets nextState=CANCELLED (1033-1037), EXPIRED event sets nextState=EXPIRED (1046-1050); CHANGE (1027-1031) leaves nextState unset/inherited, which is consistent with the rule's silence on CHANGE. The 'future-dated CREATE => PENDING' example is directly verified by the FUTURE_ONLY/PAST_OR_PRESENT_ONLY split. The follow-on claim about CANCEL flipping state to CANCELLED 'going forward' is also structurally correct (CANCEL transition's nextState=CANCELLED becomes the new previousTransition once its effective date is reached), though the card doesn't spell out that this requires the CANCEL's effective date to no longer be in the future — a minor omission, not a fidelity break, since it doesn't contradict the code.

No prompt-injection attempts were found in the cited ranges (192-205, 1007-1050) or the surrounding code I read (150-260, 340-460, 955-1058) — only ordinary Java logic and one benign inline comment.

On P0: this is the authoritative in-memory logic for a subscription's entitlement lifecycle state, which gates access/entitlement decisions and feeds directly into invoicing eligibility (e.g., getLastActivePlan/getLastActiveBillingPeriod branch on getState() at lines 364-443, and CANCELLED state suppresses further billing-relevant transitions). An error here would cause customers to be billed/entitled incorrectly (wrong plan/phase used, cancelled subscriptions treated as active or vice versa), which is a data-integrity and money-moving concern for a billing system — P0 is justified.) — confirm criticality.

### RULE-126: Add-on cascade cancellation on base plan cancel/change (subscription-level)
**Category:** Lifecycle
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:773-821`
**Plain English:** When a base plan is cancelled or changed and takes effect now or in the past, every non-cancelled/non-expired add-on in the bundle is auto-cancelled if the base plan is gone entirely, if the add-on is now included for free in the new base product, or if the add-on is no longer listed as available for the new base product. The add-on's cancellation date is the later of the base event's effective date and the add-on's own alignment start date. Cascade cancellation is skipped entirely for base plan changes/cancels scheduled in the future (it is recomputed when the event actually fires).
**Specification:**
  Given A base subscription changing today from Product A (which allows add-on X) to Product B (which does not list X as available)
  When  The plan change event is processed
  Then  Add-on X receives an auto-generated CANCEL event effective at the change date (or its own start date if later)
**Parameters:** None (driven by catalog available/included add-on lists per product)
**Confidence:** High

### RULE-127: Bundle pause blocks everything; resume clears everything
**Category:** Lifecycle
**Priority:** P0
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java:168-234`
**Plain English:** Pausing a bundle records a bundle-level ENT_BLOCKED blocking state with blockBilling, blockEntitlement, and blockChange all set to true. Resuming records an ENT_STATE_CLEAR blocking state with all three flags set to false, unblocking the bundle.
**Specification:**
  Given An active bundle
  When  pause(bundleId, effectiveDate) is called
  Then  All billing, entitlement use, and plan changes are blocked for every subscription in the bundle from that date; a subsequent resume() call fully clears the block
**Parameters:** State names: ENT_BLOCKED, ENT_STATE_CLEAR (DefaultEntitlementApi.ENT_STATE_BLOCKED/ENT_STATE_CLEAR)
**Confidence:** High

### RULE-128: Blocked periods are aggregated into disabled billing durations and injected as billing on/off events
**Category:** Lifecycle
**Priority:** P0
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java:82-161,196-282,319-357`
**Plain English:** For each subscription, all active blocking states from subscription, bundle, and account level (that occur before the subscription's cancel/expire date, if any) are merged into contiguous 'disabled' date ranges per service. For each disabled range, a synthetic START_BILLING_DISABLED billing event (zero fixed/recurring price, NO_BILLING_PERIOD) is inserted at the start, and an END_BILLING_DISABLED event restoring the prior price/plan is inserted at the end (if the block has ended); overlapping/adjacent disabled ranges from different services are merged into one.
**Specification:**
  Given A subscription blocked for billing (overdue) from day 10 to day 20, with no other blocking state
  When  Billing events are computed for invoicing
  Then  A START_BILLING_DISABLED event is inserted on day 10 (nulling fixed/recurring price) and an END_BILLING_DISABLED event restoring the previous plan/price is inserted on day 20
**Parameters:** Event types: START_BILLING_DISABLED, END_BILLING_DISABLED
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 is justified: this logic decides whether recurring/fixed charges are generated (or entirely suppressed) for a subscription during overdue/blocking periods — it directly controls whether the customer is billed, so it is squarely "moves money" territory.

Core mechanics are faithfully described and verified in code:
- Blocking states are pulled per-account, then split into ACCOUNT / SUBSCRIPTION_BUNDLE / SUBSCRIPTION buckets (BlockingCalculator.java:97-111).
- Per-subscription aggregation includes subscription+bundle+account states filtered to those at/before the subscription's CANCEL/EXPIRED effective date (getAggregateBlockingEventsPerSubscription, :163-176).
- createBlockingDurations groups blocking states by `service` (BlockingStateService per service, :323-328), builds one DisabledDuration per service, then sorts and merges across services whenever ranges are not disjoint — `isDisjoint` returns false (i.e. merges) even when ranges only touch (end == next start), so adjacent as well as overlapping ranges collapse into one (DisabledDuration.java:108-121, BlockingCalculator.java:338-354). This matches "overlapping/adjacent ... merged into one."
- For each duration, a START_BILLING_DISABLED event is inserted at the start and an END_BILLING_DISABLED event at the end only if the block has actually ended and a preceding event exists (createNewEvents, :203-218). The literal Given/When/Then example (block day10–day20, START at day10, END at day20, restoring the prior plan/price) matches createNewDisableEvent/createNewReenableEvent (:247-317) correctly, including that the disable event correctly says fixed/recurring price is set to null.

Two fidelity problems, both material for a "moves money" rule:
1. Mischaracterization of price handling: the "Plain English" line says the START event carries a "zero fixed/recurring price," but the code sets `fixedPrice`/`recurringPrice` to `null`, not `0` (BlockingCalculator.java:257-259), and the adjacent code comment explicitly states this null "makes invoice disregard this event" (:255-256) — i.e. no invoice line item is generated at all, as opposed to a $0 line item. A modernization team relying on "zero price" instead of "no line item" could reimplement this as an emitted $0 charge, which is a behavioral divergence for a billing rule. (Note: the G/W/T portion of the same rule card correctly says "nulling," contradicting its own Plain English summary.)
2. Missing edge case: BlockingStateService.addDisabledDuration (BlockingStateService.java:62-68) drops any blocking interval shorter than 1 full day (`Days.daysBetween(...).getDays() >= 1`, explicitly tied to killbill issue #267) — sub-day blocks never produce a DisabledDuration or billing-disabled events at all. The rule card does not mention this threshold, which is a meaningful business policy (a "grace" period under 24 hours never disables billing) that a modernization spec would need to preserve.

No instruction-shaped or injection-style text was found in the cited lines (comments are ordinary code comments/TODOs/issue references); no injection suspects to report.

### RULE-129: Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation
**Category:** Lifecycle
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java:313-334`
**Plain English:** Marking an invoice as written off (or reversing that) adds/removes the WRITTEN_OFF control tag on the invoice and fires an invoice-adjustment bus event, which overdue processing listens to in order to recompute the account's overdue/blocking status.
**Specification:**
  Given A committed, unpaid invoice contributing to an account being overdue
  When  tagInvoiceAsWrittenOff is called on that invoice
  Then  The WRITTEN_OFF tag is added, the invoice's balance becomes $0 for aggregation purposes, and an adjustment notification is published so overdue state can be re-evaluated
**Parameters:** Tag: ControlTagType.WRITTEN_OFF
**Confidence:** High

### RULE-130: Payment plugin status mapped to internal transaction status
**Category:** Lifecycle
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentTransactionInfoPluginConverter.java:33-56`
**Plain English:** A payment plugin's reported status is translated to Kill Bill's internal transaction status: PROCESSED to SUCCESS, PENDING to PENDING, ERROR to PAYMENT_FAILURE (the transaction reached the processor but was declined), CANCELED to PLUGIN_FAILURE (the processor confirms the transaction never happened), and UNDEFINED or a null/unrecognized status to UNKNOWN (left for the Janitor to reconcile later).
**Specification:**
  Given A plugin returns PaymentPluginStatus.CANCELED for a charge attempt
  When  toTransactionStatus is invoked
  Then  The transaction is recorded with TransactionStatus.PLUGIN_FAILURE (distinct from a declined-card PAYMENT_FAILURE)
**Parameters:** Mapping: PROCESSED->SUCCESS, PENDING->PENDING, ERROR->PAYMENT_FAILURE, CANCELED->PLUGIN_FAILURE, UNDEFINED/null->UNKNOWN
**Confidence:** High

### RULE-131: Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones
**Category:** Lifecycle
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java:63,112-155,167-221`
**Plain English:** The background Janitor process only attempts to re-query the plugin and repair a payment transaction's stored status if it is currently PENDING or UNKNOWN; SUCCESS, PAYMENT_FAILURE, and PLUGIN_FAILURE are treated as terminal and are left untouched even if re-checked.
**Specification:**
  Given A payment transaction stored with status PAYMENT_FAILURE
  When  The Janitor task runs against that transaction
  Then  It short-circuits and returns the current status unchanged, without contacting the plugin
**Parameters:** TRANSACTION_STATUSES_TO_CONSIDER = {PENDING, UNKNOWN}
**Confidence:** High

### RULE-132: Overdue state evaluation order: first matching configured state wins
**Category:** Lifecycle
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67`
**Plain English:** An account's overdue state is computed by testing each configured overdue state's condition, in the order the states are declared in overdue.xml, and returning the first one whose condition evaluates true; if none match, the account is in the CLEAR state.
**Specification:**
  Given An overdue.xml with states declared in order OD1, OD2, OD3, where both OD1's and OD2's conditions currently evaluate true for the account's billing state
  When  calculateOverdueState() runs for that account
  Then  OD1 is returned (the first match in declaration order), even though OD2 would also match
**Parameters:** None (order is purely XML declaration order)
**Edge cases handled:**
- If getStates() is empty or no condition matches, the account resolves to the built-in CLEAR state
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67 — calculateOverdueState() iterates getStates() in array order and returns the first state whose conditionEvaluation.evaluate(billingState, now) is true, falling back to getClearState() if none match. This is a plain first-match-wins loop with no tie-breaking or scoring, so the rule as stated ("first configured state in declaration order wins") accurately describes the executable behavior. No injection-shaped text found in the cited lines or surrounding file.

On the compliance lens, this does not clear the P0 bar. The code here is pure control-flow/selection logic — it does not itself compute a dollar amount, apply a tax, post a ledger entry, or reference any named regulatory requirement (e.g., PCI, AML, tax law, GDPR retention). It selects which pre-configured overdue *state* an account is in; any monetary or service-impacting consequence (blocking payments, cancelling a subscription, applying a late fee) lives in the separate state actions defined in overdue.xml and executed elsewhere, not in this evaluation-order rule. It's also not a hard data-integrity invariant like double-entry balancing or idempotency — it's ordinary business policy: 'states are evaluated in the order an admin declares them in XML.'

That said, this is a legitimate P1/P2 candidate worth flagging on the operational-risk lens: if a tenant orders overdue.xml states from least-severe to most-severe (rather than most-severe first), first-match-wins would silently keep accounts in a less severe state than intended, which could matter for dunning/due-process obligations in regulated verticals (e.g., must-warn-before-suspend rules in telecom/utilities). But that risk is contingent on tenant configuration and downstream state actions, not inherent to this code path, so I would not rate it P0 on the compliance lens as cited. | Re-derived independently from DefaultOverdueStateSet.java:59-67: calculateOverdueState iterates getStates() (backed by DefaultOverdueStatesAccount.accountOverdueStates, an @XmlElement array populated by JAXB in document order with no sorting applied anywhere in the class), returns the first DefaultOverdueState whose conditionEvaluation is non-null AND evaluates true (early return), and falls back to getClearState() if the loop exhausts with no match. This exactly matches the Given/When/Then: first-declared-order match wins (OD1 returned over OD2 when both match), CLEAR is the fallback default. No rounding/numeric edge cases apply (this is a state-selection loop, not an arithmetic calculation); the null-guard on conditionEvaluation and the early-return 'first match wins' semantics are both faithfully captured by the plain-English summary without overclaiming. No injection-shaped text found in the cited lines or surrounding file. P0 justification: which overdue/dunning state an account lands in drives payment retry cadence, service suspension/cancellation, and blocking of new charges in the overdue module -- i.e. it gates money-moving and account-standing decisions, not cosmetic behavior -- so preserving this exact state-precedence order is a legitimate behavior-contract item for modernization verification.) — confirm criticality.

### RULE-133: Refund idempotency by transaction cookie id
**Category:** Lifecycle
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:842-880`
**Plain English:** If a refund for the same payment-system transaction key already exists, don't create a duplicate refund record — just update its status/date and skip re-running CBA/adjustment side effects; the amount and payment id must match exactly or the call is rejected.
**Specification:**
  Given An existing PaymentPayment DAO refund record for transactionExternalKey 'refund-123' with amount -50.00 and paymentId P1
  When  createRefund is invoked again for the same transactionExternalKey with a computed refund amount of 50.00 and paymentId P1
  Then  the existing refund record is updated (status/date only) and returned; no new refund row, adjustment, or CBA recompute is created. If the amount or paymentId differs from the existing record, an IllegalStateException/Precondition failure is raised instead.
**Parameters:** n/a
**Edge cases handled:**
- Called multiple times by payment state machine retries for same cookie id
- Mismatched amount/paymentId on repeat call throws Precondition failure
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 is justified: this code path in DefaultInvoiceDao.createRefund (invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:806-917, cited region 842-880) is squarely money-movement and data-integrity logic — it creates/dedupes refund records against payments, enforces via Preconditions.checkState (lines 847-853) that a re-submitted refund for the same transactionExternalKey must match the previously recorded amount and paymentId or the call is rejected (IllegalStateException), and gates whether Credit Balance Adjustment (CBA) recompute and invoice-item adjustments run. A regulator/auditor/finance controller would care if this idempotency guard silently weakened (e.g., duplicate refunds issued, or CBA/adjustment logic run or skipped incorrectly) since it directly affects account balances and refund-of-record integrity.

However, the rule card is not fully faithful to the code. It states that when an existing refund is found, the code 'just update[s] its status/date and skip[s] re-running CBA/adjustment side effects' as if that were unconditional for any duplicate call. That is only true for the sub-case where the incoming status exactly equals the existing refund's status (line 856-858: `if (existingRefund.getStatus() == status) return existingRefund;` — this early return is what skips CBA/adjustment/bus-notify). If the incoming status differs (e.g., a transition from PENDING to SUCCESS on the same transactionExternalKey), the code falls through past the update-and-continue path (lines 860-873) into the shared `if (status == InvoicePaymentStatus.SUCCESS)` block at line 882, which DOES run invoice-item adjustment and CBA recompute (lines 894-910) even though no new refund row was created. The rule card collapses this status-transition nuance, which matters for compliance/verification because CBA recompute moves credit-balance amounts — a wrong assumption here could let a modernization pass skip testing the 'existing refund transitioning to SUCCESS' path, where money-affecting side effects actually do fire.

No instruction-shaped or prompt-injection text was found in the cited lines (comments at 842-843, 855, 890-892, 908-909 are ordinary engineering commentary, not directives to an AI reviewer). | Re-derived from invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:844-916 (createRefund).

What the code actually does when an existing refund is found for the same transactionExternalKey (line 844, InvoicePaymentSqlDao.getPaymentForCookieId):
1. Preconditions.checkState amount match (line 847-849): existingRefund.getAmount() must equal requestedPositiveAmount.negate() — throws IllegalStateException (via Guava Preconditions) if not. Faithful to the card.
2. Preconditions.checkState paymentId match (line 851-853): same mechanism. Faithful to the card.
3. If existingRefund.getStatus() == status (the status passed on this call equals the stored status), return existingRefund immediately at line 857 — before any adjustment/CBA code runs. This is the only path where "no adjustment/CBA recompute" is true, and in this branch no update call happens either (status/date are NOT touched, contradicting the card's claim that this branch involves an update).
4. If the status differs, the code calls transactional.updateAttempt(...) at lines 861-872 (status/date update — this part matches the card), but execution then falls through to lines 882-912: if the *new* status == InvoicePaymentStatus.SUCCESS, the invoice-item adjustment loop (894-906) and the CBALogicWrapper.runCBALogicWithNotificationEvents (909-910) DO run, exactly as they would for a brand-new refund insert. The idempotency guard does not prevent adjustment/CBA side effects on a status transition (e.g., PENDING -> SUCCESS retry) — a very plausible real-world payment-gateway retry scenario given the code's own comment about the payment system's state machine calling this multiple times (lines 842-843).

So the card's Given/When/Then collapses two mutually exclusive branches into one claim ("existing record updated AND CBA/adjustment skipped"), which is not what the code does: it is either (a) status unchanged -> early return, no update, no CBA; or (b) status changed -> update happens, and CBA/adjustment run if the new status is SUCCESS. The card's specific example doesn't state the incoming `status` value, so as written it is misleading/incomplete and would give a false equivalence target during migration — a rewritten system that "updates status but always skips CBA on retry" (as the card implies) would silently fail to apply CBA/adjustments on legitimate pending->success refund confirmations, which is a real money-handling regression risk.

The amount/paymentId precondition and no-duplicate-row parts of the card are accurate and well-supported by the code (lines 845, 847-853, 874-880).

No prompt-injection-shaped text was found in the cited range; the comment at 842-843 is ordinary engineering commentary about the payment system's state machine, not a directive to the analysis tool.

This rule is correctly P0: it deduplicates refund creation, enforces amount/paymentId integrity via hard preconditions, and gates account-balance-affecting CBA/adjustment logic — squarely money-movement and data-integrity territory. It needs to be re-specified accurately (conditioned explicitly on whether the incoming status matches the stored status, and separately on whether the resulting status is SUCCESS) before being used as a verification contract for the rewrite.

### RULE-134: Chargeback reversal resets invoice payment status to INIT
**Category:** Lifecycle
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:970-1003`
**Plain English:** Reversing a chargeback does not delete the chargeback record; it flips its status back to INIT (effectively voiding its effect on balance) and re-triggers CBA recompute for the invoice.
**Specification:**
  Given A CHARGED_BACK invoice-payment row in SUCCESS status for transactionExternalKey 'cb-1'
  When  postChargebackReversal is called for 'cb-1'
  Then  the row's status is set to INIT (no longer counted against invoice balance), CBA logic is re-run for that invoice, and a bus notification is sent
**Parameters:** n/a
**Edge cases handled:**
- Unknown cookie id throws PAYMENT_NO_SUCH_PAYMENT
**Confidence:** High

### RULE-135: Consecutive duplicate blocking states are pruned when inserting a new one
**Category:** Lifecycle
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java:236-265`
**Plain English:** When a new blocking state is recorded for a blockable object/service, the full chronological history for that object+service is re-sorted, and any state entry that is immediately followed by another entry with the identical state name is deleted as redundant — only the transition points that actually change the state are kept.
**Specification:**
  Given History for a subscription/service: t0=S1, t1=S2, t3=S3, and a new insert of S2 at t1' where t0<t1'<t1
  When  setBlockingStatesAndPostBlockingTransitionEvent runs
  Then  the redundant old S2 at t1 is unactivated/deleted, leaving t0=S1, t1'=S2, t3=S3 (never two consecutive rows with the same state name)
**Parameters:** n/a
**Edge cases handled:**
- Also cleans up legacy duplicate rows from pre-existing bad data per an explicit code comment
**Confidence:** High

### RULE-136: Blocking-state notification/bus-event batching by aggregation mode
**Category:** Lifecycle
**Priority:** P2
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java:205-286`
**Plain English:** When multiple blocking-state changes are applied together and event aggregation is enabled, only the first bus-eligible (already-effective) transition in the batch records a notification/event; future-dated transitions always get their own scheduled notification.
**Specification:**
  Given A batch of 3 blocking-state changes, 2 of which are effective immediately and grouping is enabled
  When  setBlockingStatesAndPostBlockingTransitionEvent processes the batch
  Then  only the first immediately-effective change triggers a bus notification computation; the second immediately-effective one is skipped for notification purposes
**Parameters:** n/a
**Confidence:** Medium — Confirm this event-coalescing does not suppress externally-visible entitlement webhooks that downstream integrators depend on for each individual state change.

### RULE-137: Idempotent get-or-create for the external/manual-pay payment method
**Category:** Lifecycle
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:475-491`
**Plain English:** Requesting the external (manual) payment method for an account returns the existing one if the account already has one, otherwise creates a new external payment method — it is never duplicated.
**Specification:**
  Given An account with no external payment method yet
  When  createOrGetExternalPaymentMethod is called twice in a row
  Then  the first call creates the external payment method; the second call returns the same payment method id without creating a second one
**Parameters:** n/a
**Confidence:** High

### RULE-138: Subscription events occurring at or after a CANCEL/EXPIRED event are discarded
**Category:** Lifecycle
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:1093-1123`
**Plain English:** Once a subscription has a CANCEL or EXPIRED event, that is treated as the definitive end of the subscription's event stream: any other active event dated on or after that cancellation/expiration date is removed from consideration, except the CREATE/TRANSFER 'start of time' event and the cancellation/expiration event itself, which are always kept. This guards against data-integrity bugs that produced out-of-order or duplicate cancellations.
**Specification:**
  Given An event stream with a CANCEL event on 2026-01-15 and a stray CHANGE event dated 2026-01-20 (after cancellation) due to a data bug
  When  removeEverythingPastCancelEvent runs while rebuilding subscription state
  Then  the 2026-01-20 CHANGE event is dropped from the effective stream; only events up to and including the 2026-01-15 CANCEL remain
**Parameters:** n/a
**Edge cases handled:**
- Explicitly references known bugs killbill/killbill#897 and #619 in comments as the reason for this hardening
**Confidence:** High

### RULE-139: Accounts are auto-parked on unrecoverable invoice-generation errors
**Category:** Lifecycle
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:443-487`
**Plain English:** When invoice generation for an account fails with a catalog, account, subscription, or unexpected runtime error (outside of a dry-run), the account is automatically tagged as 'parked', which then blocks further automatic invoicing until an operator investigates and un-parks it. Errors from an API call, or from a dry-run, do not auto-park unless the system is explicitly configured to always park on exceptions, with one exception: an InvoiceApiException carrying UNEXPECTED_ERROR always parks the account (outside dry-run) even from an API call.
**Specification:**
  Given An account whose invoice generation throws a CatalogApiException during the nightly batch run
  When  processAccount catches the exception
  Then  the account is tagged PARKED (idempotently) via ParkedAccountsManager.parkAccount, and an empty invoice list is returned instead of propagating the error
**Parameters:** Config flag: invoiceConfig.isParkAccountsOnAllExceptions()
**Edge cases handled:**
- Plugin-aborted invoice generation (INVOICE_PLUGIN_API_ABORTED) does not park the account — treated as an intentional stop
**Confidence:** High

### RULE-140: Account parking/unparking implemented via idempotent system tag
**Category:** Lifecycle
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java:46-61`
**Plain English:** Parking and unparking an account is just adding/removing a well-known system tag on the account; parking is idempotent (adding the tag twice is not an error).
**Specification:**
  Given An already-parked account
  When  parkAccount is called again
  Then  the duplicate-tag error is silently swallowed and the account remains parked (no exception surfaces to the caller)
**Parameters:** n/a
**Confidence:** Medium — Citation was corrected by referee (The cited range (lines 28-42) covers only import statements, the class declaration, and the constructor (ParkedAccountsManager.java:35-44) — none of which implement parking/unparking logic. The actual behavior described in the rule — parkAccount() catching TagApiException and swallowing it only when ErrorCode.TAG_ALREADY_EXISTS matches (making the call idempotent), and unparkAccount() removing the tag — lives in the parkAccount/unparkAccount methods at lines 46-61, just past the cited range. The rule text itself is accurate to what the code does, just cited at the wrong lines.) — confirm invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java:46-61 is the authoritative implementation.

### RULE-141: Role permission update computes and applies a diff (add new, deactivate removed)
**Category:** Lifecycle
**Priority:** P2
**Source:** `util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java:117-140`
**Plain English:** Updating a role's permission list is not a wholesale replace: existing permissions not present in the new list are individually deactivated, and only genuinely new permissions are inserted; supplying an empty permission list removes all existing permissions for that role.
**Specification:**
  Given Role 'support' currently has permissions [A, B, C]
  When  updateRoleDefinition('support', [B, D]) is called
  Then  permission A and C are deactivated, permission D is newly created, and B is left untouched (not re-inserted)
**Parameters:** n/a
**Edge cases handled:**
- Empty incoming permissions list deactivates all existing permissions for the role
**Confidence:** High

## Policy

### RULE-142: In-arrear greedy billing mode
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/BillingIntervalDetail.java:134-181`
**Plain English:** For arrears billing, the system normally waits until the end of a billing period before invoicing it. An optional 'greedy' mode lets the system bill a not-yet-complete period as soon as the target (as-of) date falls anywhere within it, rather than waiting for the period's end.
**Specification:**
  Given InArrearMode=GREEDY, a monthly IN_ARREAR plan with firstBillingCycleDate=2024-01-01, and targetDate=2024-01-15 (mid-period)
  When  the effective end date for invoicing is computed
  Then  billing occurs for the period ending on the next proposed billing cycle date (e.g. 2024-02-01) rather than being deferred until that date is reached in a later invoice run
**Parameters:** org.killbill.invoice.inArrear.mode / InArrearMode enum {DEFAULT, GREEDY} (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java, enum at InvoiceConfig.java:58-61)
**Edge cases handled:**
- targetDate exactly equals endDate (subscription cancellation date) → always bills immediately regardless of greedy mode, to handle CHANGE/CANCELLATION events (BillingIntervalDetail.java:143-146, referencing issue #1907)
**Confidence:** High

### RULE-143: AUTO_PAY_OFF blocks only system-triggered payments
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:730-745,376-380`
**Plain English:** If an account is tagged AUTO_PAY_OFF, the system will not attempt to automatically collect payment against it (e.g. via retries or invoice-driven auto-pay) — but a payment explicitly initiated by the customer/API through the payment API still goes through.
**Specification:**
  Given an account tagged AUTO_PAY_OFF and a system-initiated (non-API) payment attempt for a $75.00 invoice
  When  the prior-payment-control check runs
  Then  the payment attempt is recorded in a pending 'auto pay off' queue and aborted (isApiPayment=false AND tag present → abort); if the same payment were instead flagged isApiPayment=true, this check is skipped entirely and payment proceeds
**Parameters:** ControlTagType.AUTO_PAY_OFF tag (see util/src/main/java/org/killbill/billing/util/tag/ControlTagType.java, referenced at InvoicePaymentControlPluginApi.java:747-749)
**Confidence:** High

### RULE-144: Payment failure retry schedule
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:592-609`
**Plain English:** After a real payment decline (e.g. card declined, insufficient funds), the system retries on a fixed schedule of days after the failure, and gives up once the configured list of retry intervals is exhausted.
**Specification:**
  Given org.killbill.payment.retry.days default '8,8,8' and this is the 2nd consecutive PAYMENT_FAILURE for a transaction (attemptsInState=2, so retryCount=1)
  When  the next retry date is computed
  Then  next retry = now + 8 days (the 2nd entry in the list, index 1); after the 3rd failure (retryCount=2, still < list size 3) another retry is scheduled +8 days; after the 4th failure (retryCount=3 >= list size 3) no further retry is scheduled (payment permanently fails)
**Parameters:** org.killbill.payment.retry.days, default '8,8,8' (util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java:31-39)
**Edge cases handled:**
- retryCount computed as max(attemptsInState-1, 0) (line 597)
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: InvoicePaymentControlPluginApi.java:592-609 computes retryCount = attemptsInState-1 against paymentConfig.getPaymentFailureRetryDays() (default "8,8,8" from PaymentConfig.java:31-39), returns now+retryDays[retryCount] while retryCount < list.size(), else null (no more retries). The rule's worked example (2nd failure -> retryCount=1 -> +8 days; 3rd failure -> retryCount=2 -> +8 days; 4th failure -> retryCount=3 >= size 3 -> permanent stop) matches the code exactly. No injection-shaped text found in the cited lines.\n\nHowever, under the COMPLIANCE lens this is not P0. The retry schedule itself does not move money — it only decides *when* the system will re-attempt a previously-declined payment; the actual money movement happens in the payment plugin call that this schedule merely triggers later. It is not enforcing a statute, PCI rule, or card-network mandate encoded in the logic itself (no reference to a specific regulation anywhere in this method or config), and it does not guard data integrity (no dedupe/idempotency/reconciliation concern here — that lives elsewhere, e.g. in payment state-machine transition guards). It is fully tenant-configurable business policy (org.killbill.payment.retry.days, default 8,8,8) — exactly the kind of operational cadence a finance/ops team tunes per business need, not something an auditor or regulator would flag if it silently changed. A regulator/controller would care about tax/interest math, refund correctness, or duplicate-charge prevention; they would generally not treat a dunning retry cadence as an audit-relevant compliance control unless the business had specifically documented it as satisfying a card-network retry-limit rule (Visa/Mastercard decline-retry compliance), and nothing in this code ties it to such a requirement. This is better rated P1 (revenue-operations rule, real business impact if it silently changes, but not a compliance/audit-tier finding). | Re-derived independently from InvoicePaymentControlPluginApi.java:592-609 and PaymentConfig.java:29-39. Default retryDays = [8,8,8] (confirmed at PaymentConfig.java:32,37). retryCount = max(attemptsInState-1, 0); while retryCount < retryDays.size(), next retry = internalContext.getCreatedDate().plusDays(retryDays.get(retryCount)); once retryCount >= 3 the method returns null (no more retries). The GWT's numeric walkthrough (2nd failure -> retryCount=1 -> +8d, 3rd failure -> retryCount=2 -> +8d, 4th failure -> retryCount=3, not < 3, give up) matches exactly. Two minor, non-material paraphrases: 'now' is technically internalContext.getCreatedDate() rather than wall-clock now(), and 'consecutive' failures is really a count of all PAYMENT_FAILURE transactions in the purchased-transaction list gated by the caller checking the *last* transaction's status — functionally equivalent in this flow but worth noting. No instruction-shaped/injection text found in the cited code or PaymentConfig.java. P0 is justified: this rule governs whether/when a declined payment is retried and when the system permanently gives up, which directly affects revenue collection and must be preserved exactly through any rewrite.) — confirm criticality.

### RULE-145: Plugin-failure retry backoff (suspected off-by-one)
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:612-628`
**Plain English:** After a transient plugin/gateway failure (not a real decline), the system retries with an exponentially growing delay starting from an initial wait, doubling (by default) on each subsequent attempt, up to a maximum number of attempts.
**Specification:**
  Given initial retry = 300s, multiplier = 2, max attempts = 8 (all defaults), and this is the 4th consecutive PLUGIN_FAILURE (attemptsInState=4, retryAttempt=3)
  When  the next retry delay is computed by the while(--remainingAttempts > 0) loop
  Then  nbSec starts at 300 and the loop runs (retryAttempt-1)=2 times, doubling each time → 300 → 600 → 1200, so next retry = now + 1200s
**Parameters:** org.killbill.payment.failure.retry.start.sec=300, org.killbill.payment.failure.retry.multiplier=2, org.killbill.payment.failure.retry.max.attempts=8 (PaymentConfig.java:41-59,82-90)
**Edge cases handled:**
- retryAttempt=0 (1st failure) and retryAttempt=1 (2nd failure) BOTH resolve to the unmultiplied initial delay (300s) because the doubling loop's `--remainingAttempts > 0` guard skips execution for remainingAttempts in {0,1} — i.e. the first two retries occur at the same interval instead of following a clean 300/600/1200 progression. This looks like an off-by-one defect in the loop bound, not an intentional design.
- retryAttempt >= maxAttempts (8) → no further retry, permanent failure
**Suspected defect:** The while(--remainingAttempts > 0) loop causes retry attempt #1 (0-indexed 0) and retry attempt #2 (0-indexed 1) to both use the un-multiplied initial delay, effectively wasting one exponential step versus the likely intended 300/600/1200/2400... sequence.
**Confidence:** Medium — Is it intentional that the first two plugin-failure retries both wait exactly the initial 300s before the delay starts doubling, or should the second retry already be 600s?

### RULE-146: Billing-blocking overdue state auto-disables invoicing
**Category:** Policy
**Priority:** P0
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:176-183,237-253,267-269`
**Plain English:** Entering an overdue state that disables entitlement auto-tags the account AUTO_INVOICING_OFF; clearing it removes the tag.
**Specification:**
  Given Account moves to an overdue state with disableEntitlementAndChangesBlocked=true
  When  OverdueStateApplicator.apply runs
  Then  The account is tagged AUTO_INVOICING_OFF
**Parameters:** blockBilling = isDisableEntitlementAndChangesBlocked()
**Edge cases handled:**
- Tag removal tolerates TAG_DOES_NOT_EXIST
**Confidence:** High

### RULE-147: Default overdue escalation tiers
**Category:** Policy
**Priority:** P1
**Source:** `profiles/killbill/src/main/resources/overdue.xml:19-61`
**Plain English:** Default config escalates by age of earliest unpaid invoice: 30 days blocks changes (OD1), 40 days disables entitlement/billing (OD2), 50 days same with distinct message (OD3); rechecked every 5 days.
**Specification:**
  Given Earliest unpaid invoice is 40 days old
  When  Overdue check runs
  Then  Account moves to OD2 with entitlement and billing blocked
**Parameters:** OD1=30d, OD2=40d, OD3=50d, recheck=5d
**Edge cases handled:**
- Overridable per tenant
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (The Given/When/Then is mechanically accurate against overdue.xml:19-61 (OD1=30d blockChanges only; OD2=40d and OD3=50d add disableEntitlementAndChangesBlocked=true; autoReevaluationInterval=5d for all three), but the "Plain English" framing that this is the "default config" is misleading. The actual system-wide default, set by @Default("NoOverdueConfig.xml") on OverdueProperties.getConfigURI() (overdue/src/main/java/org/killbill/billing/overdue/OverdueProperties.java:27-30), is NoOverdueConfig.xml, a single 'Clear' state that performs no blocking at all (overdue/src/main/resources/NoOverdueConfig.xml:20-24). This overdue.xml under profiles/killbill/src/main/resources is only wired in by the test properties file (org.killbill.overdue.uri=overdue.xml in profiles/killbill/src/test/resources/.../killbill.properties:19) and by TestOverdue.java, which uploads it as a per-tenant config. In production, no overdue enforcement fires unless an operator explicitly points org.killbill.overdue.uri at this file or a tenant uploads an equivalent config via the API — so calling these 30/40/50-day thresholds 'the default' overstates what ships active out of the box.

Even taken at face value as a sample dunning schedule, this does not clear the P0 bar. It doesn't move money (no charge, refund, or amount calculation), it isn't a regulatory/compliance mandate (no statute, PCI, or GAAP requirement is cited or implied — dunning cadence is a commercial collections policy the platform explicitly makes tenant-configurable, per TenantOverdueConfigCacheLoader.java), and it doesn't guard data integrity (it gates account state/entitlement transitions, not data correctness or consistency). A finance controller or auditor might care about collections SLAs operationally, but silently changing this sample/tenant-overridable threshold set doesn't violate a legal or accounting obligation the way, e.g., a tax calculation or a ledger-balancing rule would. This is better classified as a configurable business/collections policy (P1-P2), not a P0 compliance control.

No prompt-injection attempts were found in the cited lines (19-61) — the file is plain XML config with a standard Apache license header. | Re-derived independently from profiles/killbill/src/main/resources/overdue.xml:19-61 plus the evaluation code in overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java (calculateOverdueState, lines 60-67) and DefaultOverdueState.java / OverdueStateApplicator.java (isBlockChanges/isDisableEntitlementAndChangesBlocked, lines 255-273).

Faithful (true): The XML declares three states with timeSinceEarliestUnpaidInvoiceEqualsOrExceeds thresholds of 50 (OD3), 40 (OD2), 30 (OD1) days, each with autoReevaluationInterval = 5 days (lines 24,31,37,44,50,57). calculateOverdueState() iterates getStates() in the XML's declared order (OD3, OD2, OD1) and returns the FIRST state whose condition evaluates true, i.e. the most-severe matching state wins (DefaultOverdueStateSet.java:60-67) — confirming the escalation-by-age semantics claimed. OD1 sets blockChanges=true, disableEntitlementAndChangesBlocked=false (lines 54-55); OD2 and OD3 both set blockChanges=true AND disableEntitlementAndChangesBlocked=true (lines 28-29, 41-42) with distinct externalMessage text ("Reached OD1/OD2/OD3", lines 27,40,53). OverdueStateApplicator confirms disableEntitlementAndChangesBlocked=true is what actually triggers both billing-block and entitlement-block (blockBilling/blockEntitlement both just return isDisableEntitlementAndChangesBlocked(), lines 267-273), and blockChanges() is true whenever either flag is set (line 264) — so OD2/OD3 correctly block changes+entitlement+billing while OD1 only blocks changes. The worked example (40-day-old invoice -> OD2, entitlement+billing blocked) is exactly correct given the ">= " (EqualsOrExceeds) boundary semantics and the most-severe-first matching order (OD3's >=50 fails at 40 days, OD2's >=40 matches). No rounding/ordering/edge-case discrepancy found; the "EqualsOrExceeds" operator correctly makes the boundary value (e.g., exactly 40 days) match OD2, as the rule implies.

P0 justification (false): This file is a shipped *default* overdue configuration — per-tenant overrides exist (see MultiTenantOverdueConfig.java) and commonly replace these defaults, so the specific 30/40/50/5-day parameters are not a fixed, universally-enforced constant the way a regulatory threshold would be. More importantly, the rule itself does not move money (no charge/fee/interest calculation), is not a regulatory/compliance requirement, and does not guard data integrity (it does not prevent corrupt or inconsistent stored data) — it gates *access* (blocking subscription changes, entitlement, and future billing actions) as a collections/dunning policy. That's a legitimate and important business rule, but by the stated P0 bar (moves money / regulatory / data integrity) it's better classified as P1 — a configurable business policy whose exact thresholds are examples, not a money-moving or compliance-mandated calculation.

No injection-shaped text was found in the cited lines or surrounding file (only a benign inline comment "Other actions could include... trigger payment retry" in DefaultOverdueState.java:78-83, which is developer notes, not an instruction to the analyzer).) — confirm criticality.

### RULE-148: Plan-change alignment determines the effective phase start date
**Category:** Policy
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java:239-282`
**Plain English:** When a customer changes plans, the catalog's alignment rule decides whether the new plan's phase clock restarts at the original subscription start, the bundle start, or the moment of the change itself.
**Specification:**
  Given A subscription changing from Plan A to Plan B where the matching CaseChangePlanAlignment rule resolves to CHANGE_OF_PLAN
  When  the change becomes effective
  Then  The new plan's phase timeline starts at the change's effective date (lastOrCurrentChangeEffectiveDate) rather than the original subscription/bundle start date; for START_OF_SUBSCRIPTION/START_OF_BUNDLE alignment the new plan instead re-uses the original subscription or bundle start date, and CHANGE_OF_PRICELIST is explicitly unimplemented (throws)
**Edge cases handled:**
- CHANGE_OF_PRICELIST alignment throws SubscriptionBaseError('Not implemented yet')
**Confidence:** High

### RULE-149: Account credit is distributed across unpaid invoices oldest-first
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java:185-217`
**Plain English:** When applying spare account credit across multiple unpaid invoices, the oldest invoice (by invoice date) is paid down first, then the next oldest, until the credit runs out.
**Specification:**
  Given Account credit of $50.00 and two unpaid COMMITTED invoices: Inv#1 dated 2026-01-01 for $30, Inv#2 dated 2026-02-01 for $40
  When  useExistingCBAFromTransaction runs
  Then  Inv#1 is fully paid with $30 of credit first, leaving $20, which is then applied to Inv#2 (partially settling it); processing stops as soon as remaining credit reaches zero
**Confidence:** High

### RULE-150: Account-level BCD alignment falls back to subscription alignment until the account BCD is set
**Category:** Policy
**Priority:** P1
**Source:** `util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java:50-54`
**Plain English:** A catalog rule saying 'align billing to the account's bill-cycle-day' is treated as 'align to the subscription's own bill-cycle-day' for as long as the account doesn't have a bill-cycle-day yet (e.g. the very first subscription on a brand-new account).
**Specification:**
  Given BillingAlignment.ACCOUNT and accountBillCycleDayLocal == 0 (unset)
  When  resolveEffectiveBillingAlignment is called
  Then  BillingAlignment.SUBSCRIPTION is returned instead of ACCOUNT
**Confidence:** High

### RULE-151: Blocked billing periods shorter than one day are not disabled
**Category:** Policy
**Priority:** P2
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68`
**Plain English:** If a billing-block is toggled on and back off again within less than a full day, it's ignored — no billing gap is created for it.
**Specification:**
  Given A BlockingState becomes block-billing at 2026-06-01T10:00 and un-blocks at 2026-06-01T18:00 (same calendar day, less than 1 day apart)
  When  addDisabledDuration evaluates whether to record this window
  Then  No DisabledDuration is recorded (Days.daysBetween(...) < 1), so no billing gap/disable event is created for that period; an open-ended block (disableDurationEndDate == null) is always recorded regardless of length
**Parameters:** Minimum disable duration threshold: 1 day (hardcoded)
**Confidence:** High

### RULE-152: Add-on creation eligibility against base subscription
**Category:** Policy
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-77`
**Plain English:** An add-on can only be attached to a base subscription that is active (or pending-active as of the request date), whose base product doesn't already include that add-on for free, and whose base product's catalog entry explicitly allows that add-on.
**Specification:**
  Given A base subscription in state CANCELLED, or PENDING with a start date after the requested add-on date
  When  A caller tries to create an add-on subscription against that base
  Then  The call is rejected with SUB_CREATE_AO_BP_NON_ACTIVE; separately, if the add-on's product is already in the base product's 'included' list it is rejected with SUB_CREATE_AO_ALREADY_INCLUDED, and if it's not in the base product's 'available' list it is rejected with SUB_CREATE_AO_NOT_AVAILABLE
**Parameters:** Included/available add-on product lists are catalog-defined (per base Product), not hardcoded
**Confidence:** Medium — P0 panel doubts spec fidelity: P0 classification is justified: this gate is the sole guard preventing an add-on from being attached to a base subscription that is cancelled/not-yet-active, already includes the add-on for free, or isn't catalog-permitted for that base product — all of which directly affect what gets invoiced (double-billing for an already-included feature, billing an add-on with no valid base, etc.), so it's a legitimate data-integrity/billing-correctness rule, not incidental plumbing.

However, the Given/When/Then is NOT faithful to the code for the PENDING-state branch, and this is a real (not cosmetic) discrepancy:

Code (AddonUtils.java:38-41):
```java
if (baseSubscription.getState() == EntitlementState.CANCELLED ||
    (baseSubscription.getState() == EntitlementState.PENDING &&
     context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0)) {
    throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_BP_NON_ACTIVE, targetAddOnPlan.getName());
}
```
`SD.compareTo(RD) < 0` means SD (base start date) is chronologically BEFORE RD (requested add-on date). So the rejection fires when the base's start date is *before* the requested date — i.e., when the requested date comes *after* the start date.

Rule Card claims the opposite: "PENDING with a start date after the requested add-on date" (i.e., SD > RD) triggers rejection. That is the reverse of what `compareTo(...) < 0` actually encodes.

I traced this further via the caller (SubscriptionApiBase.java:241-244), which already rejects any requestedDate before the base's start date with a *different* error (SUB_INVALID_REQUESTED_DATE) before AddonUtils is even invoked:
```java
if (effectiveDate.isBefore(baseSubscription.getStartDate())) {
    throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, ...);
}
addonUtils.checkAddonCreationRights(baseSubscription, plan, effectiveDate, context);
```
So by the time AddonUtils.checkAddonCreationRights runs, requestedDate >= baseSubscription.getStartDate() is already guaranteed. Combined with the `< 0` check inside AddonUtils, the real, end-to-end behavior is: while a base subscription is still PENDING, an add-on can only be created with an effective date exactly equal to the base's start date — a requested date strictly later than the start date is rejected as SUB_CREATE_AO_BP_NON_ACTIVE, and a requested date strictly earlier is rejected upstream as SUB_INVALID_REQUESTED_DATE. This is confirmed by git history: commit 1b1eb87d41 ("entitlement: Allow ADD_ON creation on pending BP. Fixes #681") added this exact line, and its own regression test (TestDefaultEntitlementApi.testAddEntitlementOnPendingBase, added in the same commit) only exercises requestedDate == startDate for the PENDING case — it never tests requestedDate > startDate, which is the case that this logic actually blocks. The commit message itself ("Adding an ADD_ON is now available starting from date when the BB effectively starts") describes intent to allow dates >= start, which the shipped comparison operator does not deliver for PENDING bases (it only allows the exact match).

Net effect: the Rule Card's stated Given/When/Then direction for the PENDING branch is inverted relative to the code, and it also omits the much narrower real-world effect (PENDING bases effectively require an exact start-date match, not "any date on/after start"). The CANCELLED-state rule and the included/available product-list checks (lines 45-53, ErrorCode.SUB_CREATE_AO_ALREADY_INCLUDED / SUB_CREATE_AO_NOT_AVAILABLE) are faithfully described.

No prompt-injection-shaped text was found in AddonUtils.java:36-77 or the related SubscriptionApiBase.java lines reviewed — no injectionSuspects to report.

Recommendation: correct the Rule Card to state the PENDING rejection condition as "requested add-on date is strictly after the base subscription's start date" (not "before"), and add a companion rule for SubscriptionApiBase.java:241-243 (SUB_INVALID_REQUESTED_DATE) since together they define the true eligibility window (exact-date-only) for add-ons on a still-pending base. An SME should confirm whether "exact match only" for PENDING bases is intended, since it contradicts the commit's own stated intent of allowing "on or after" the base start date.

### RULE-153: Subscription state guard for plan changes
**Category:** Policy
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:835-850`
**Plain English:** A plan change is blocked if the subscription is CANCELLED, EXPIRED, has a future cancellation already scheduled, has a future expiry before the requested effective date, or the requested date is before the subscription's start date.
**Specification:**
  Given A subscription with a pending future-dated cancellation already scheduled
  When  changePlanWithRequestedDate or changePlanWithPolicy is called
  Then  The call is rejected with SUB_CHANGE_NON_ACTIVE (bad state / date-before-start) or SUB_CHANGE_FUTURE_CANCELLED or SUB_CHANGE_FUTURE_EXPIRED as appropriate
**Parameters:** None
**Confidence:** High

### RULE-154: Cancellation of an EXPIRED subscription is blocked
**Category:** Policy
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java:191-194,213-217,230-234`
**Plain English:** A subscription that has already reached the EXPIRED state cannot be cancelled again through any of the three cancel entry points (default policy, requested date, or explicit policy).
**Specification:**
  Given A subscription in state EXPIRED
  When  cancel(), cancelWithRequestedDate(), or cancelWithPolicy() is called on it
  Then  The call is rejected with SUB_CANCEL_BAD_STATE
**Parameters:** None
**Confidence:** High

### RULE-155: Illegal plan-change policy blocks the change
**Category:** Policy
**Priority:** P0
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:143-164`
**Plain English:** The catalog can define change-plan rules that mark specific from-plan/to-plan combinations as ILLEGAL; such combinations are rejected outright regardless of any billing/alignment considerations.
**Specification:**
  Given A catalog change-plan rule maps a given (from,to) combination to policy ILLEGAL
  When  getPlanChangeResult() is invoked for that combination
  Then  An IllegalPlanChange exception is thrown before alignment is even computed
**Parameters:** changeCase rules are catalog-XML defined per tenant/catalog version
**Confidence:** High

### RULE-156: Blocking-state cascade blocks change/entitlement/billing actions (OR across levels)
**Category:** Policy
**Priority:** P0
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:44-91,137-188,218-248; entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java:25-38`
**Plain English:** An action (plan change, entitlement change, or billing) is blocked if ANY of the account-level, bundle-level, or subscription-level blocking states for that subscription say to block it; blocking flags are combined with logical OR across all three levels, so a block at any level propagates down.
**Specification:**
  Given An account-level BlockingState with isBlockBilling()=true and a subscription with no subscription- or bundle-level blocks
  When  checkBlockedBilling is called for that subscription
  Then  A BlockingApiException BLOCK_BLOCKED_ACTION is thrown for the subscription, purely because of the inherited account-level block
**Parameters:** None (boolean OR aggregation)
**Confidence:** High

### RULE-157: Add-on creation blocked when base subscription is cancelled, not-yet-pending, or itself blocked
**Category:** Policy
**Priority:** P0
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:530-544`
**Plain English:** Before creating an add-on entitlement, the system verifies the base subscription is not cancelled, not still-pending-in-the-future relative to the requested add-on date, and not itself blocked for change or entitlement actions.
**Specification:**
  Given A base entitlement that is PENDING with an effective start date after the requested add-on's effective date
  When  A caller tries to add an add-on entitlement to that bundle
  Then  The call fails with SUB_GET_NO_SUCH_BASE_SUBSCRIPTION (treated as if there's no valid base), or with a BLOCK_BLOCKED_ACTION exception if the base is change/entitlement-blocked
**Parameters:** None
**Confidence:** High

### RULE-158: Default payment method cannot be deleted without explicit override
**Category:** Policy
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:503-528`
**Plain English:** Deleting an account's default payment method is refused unless the caller explicitly passes deleteDefaultPaymentMethodWithAutoPayOff or forceDefaultPaymentMethodDeletion; if deleted via the auto-pay-off option and the account isn't already on AUTO_PAY_OFF, the system automatically switches the account to AUTO_PAY_OFF before removing it.
**Specification:**
  Given An account whose default payment method equals the one being deleted, called with both override flags = false
  When  deletedPaymentMethod is invoked
  Then  The call fails with PAYMENT_DEL_DEFAULT_PAYMENT_METHOD; if deleteDefaultPaymentMethodWithAutoPayOff=true instead and the account isn't already AUTO_PAY_OFF, the account is switched to AUTO_PAY_OFF as a side effect before the payment method is removed
**Parameters:** None (two boolean override flags)
**Confidence:** High

### RULE-159: Blocking-state flags OR-aggregate across account, bundle, and subscription levels
**Category:** Policy
**Priority:** P0
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:44-124,161-248`
**Plain English:** Whether a change, entitlement action, or billing action is blocked for a subscription is computed by OR-ing the blockChange/blockEntitlement/blockBilling flags of the most recent blocking state at the subscription level, its bundle level, and its account level — if any one of the three levels blocks the action, the action is blocked, regardless of the other levels.
**Specification:**
  Given A subscription itself is not blocked, but its account carries a blocking state with blockBilling=true (e.g. from overdue)
  When  checkBlockedBilling is called for that subscription
  Then  BlockingApiException BLOCK_BLOCKED_ACTION is thrown, because the account-level block propagates down
**Parameters:** None
**Confidence:** High

### RULE-160: Sub-one-day blocking periods are not disabled for billing
**Category:** Policy
**Priority:** P1
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java:62-68`
**Plain English:** A blocking period is only turned into a billing-disabled duration if it lasts at least one full day; blocking states that are opened and cleared within the same day are ignored for billing purposes (per GitHub issue #267).
**Specification:**
  Given A blocking state that sets blockBilling=true and is cleared 6 hours later
  When  createBlockingDurations computes disabled ranges for the subscription
  Then  No START_BILLING_DISABLED/END_BILLING_DISABLED event pair is generated for that sub-day window
**Parameters:** Minimum blocked duration: 1 day
**Confidence:** High

### RULE-161: AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing
**Category:** Policy
**Priority:** P0
**Source:** `junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DefaultInternalBillingApi.java:99-115,208-214`
**Plain English:** If the account carries an AUTO_INVOICING_OFF control tag, the entire account's billing event set is marked off. If only a specific bundle carries the AUTO_INVOICING_OFF tag, every subscription in that bundle is added to a skip list and excluded from billing-event generation, while other bundles on the account continue to be billed normally.
**Specification:**
  Given A bundle tagged AUTO_INVOICING_OFF, on an account with two other untagged bundles
  When  Billing events are computed for the account
  Then  Subscriptions in the tagged bundle are recorded in subscriptionIdsWithAutoInvoiceOff and produce no billing events; the other bundles bill normally
**Parameters:** Tags: AUTO_INVOICING_OFF, AUTO_INVOICING_DRAFT, AUTO_INVOICING_REUSE_DRAFT
**Confidence:** High

### RULE-162: Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:401-418`
**Plain English:** When a payment transaction remains unresolved, the delay before the next Janitor recheck attempt is looked up from one of two separately configured retry-interval lists depending on whether the transaction's status is UNKNOWN or PENDING; once the attempt number exceeds the configured list's length, no further recheck is scheduled and the transaction is left in its unresolved state permanently.
**Specification:**
  Given A transaction stuck in PENDING with paymentConfig.getPendingTransactionsRetries() containing 3 entries
  When  The Janitor schedules its 4th retry attempt
  Then  getNextNotificationTime returns null and no further notification/recheck is scheduled
**Parameters:** paymentConfig.getUnknownTransactionsRetries(...) and paymentConfig.getPendingTransactionsRetries(...) — configurable TimeSpan lists, values not hardcoded in this file
**Confidence:** Medium — What are the configured default retry-interval lists for UNKNOWN vs PENDING transactions in production (paymentConfig defaults), and is it acceptable that a transaction permanently stays unresolved once the list is exhausted with no alerting?

### RULE-163: Bundle transfer: old subscription cancellation date honors charged-through date
**Category:** Policy
**Priority:** P1
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java:250-268`
**Plain English:** When a bundle is transferred to another account (not an immediate cancel), the source subscription isn't cancelled on the transfer date if the customer already paid through a later date — it is cancelled at the later of the charged-through date or the earliest valid transition date instead.
**Specification:**
  Given A subscription's charged-through date is 2026-09-15, transfer requested for 2026-09-01, cancelImmediately=false, and the subscription's earliest valid transition date is 2026-08-01
  When  transferBundle() computes the cancellation event for the old subscription
  Then  The old subscription's cancel event is scheduled for 2026-09-15 (charged-through date), not 2026-09-01; if that computed date were somehow before the earliest valid transition date, it is pulled forward to that earliest date instead
**Confidence:** High

### RULE-164: Chargeback is idempotent — no partial/duplicate chargebacks
**Category:** Policy
**Priority:** P0
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:209-232`
**Plain English:** A payment can only be charged back once; if a chargeback invoice-payment record already exists for that payment, a second successful chargeback transaction is treated as a no-op instead of recording another chargeback.
**Specification:**
  Given A payment that already has a recorded chargeback
  When  onSuccessCall() is invoked again for a CHARGEBACK transaction on the same payment
  Then  No new chargeback invoice-payment is recorded; the system logs that the chargeback was already completed
**Confidence:** High

### RULE-165: REFUND and CREDIT transactions are never retried on failure
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:294-297`
**Plain English:** If a refund or credit transaction fails, the system does not schedule an automatic retry (unlike failed purchases, which do get a computed retry date).
**Specification:**
  Given A REFUND transaction that fails at the gateway
  When  onFailureCall() is invoked for that transaction
  Then  No nextRetryDate is computed or returned; only a PURCHASE failure computes computeNextRetryDate(...)
**Confidence:** High

### RULE-166: Chargeable invoice item type classification
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java:87-94`
**Plain English:** Only TAX, EXTERNAL_CHARGE, FIXED, USAGE, and RECURRING invoice item types count as billable 'charges' for calculator purposes; credits, adjustments, and CBA items do not.
**Specification:**
  Given An invoice with a RECURRING item and a CBA_ADJ item
  When  isCharge() is evaluated on each item
  Then  The RECURRING item returns true; the CBA_ADJ item returns false
**Confidence:** High

### RULE-167: Catalog version selection by effective date
**Category:** Policy
**Priority:** P0
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java:91-107`
**Plain English:** When looking up the catalog to use for a given date, the system picks the most recent catalog version whose effective date is on or before that date; if every version is dated after the given date, it falls back to using the very first (oldest) version rather than failing.
**Specification:**
  Given Catalog versions effective 2025-01-01, 2025-06-01, 2026-01-01, and a lookup date of 2025-08-01
  When  getVersion(date) is called
  Then  The 2025-06-01 version is returned (latest version with effectiveDate <= 2025-08-01)
**Edge cases handled:**
- If lookup date is before all versions (e.g. due to clock manipulation in tests), the oldest version is returned instead of throwing, per killbill/killbill#760
- Throws IllegalStateException only if there are zero catalog versions at all
**Confidence:** High

### RULE-168: Existing-subscription catalog grandfathering
**Category:** Policy
**Priority:** P0
**Source:** `subscription/src/main/java/org/killbill/billing/subscription/catalog/SubscriptionCatalog.java:168-222`
**Plain English:** New subscriptions always use the latest catalog version as of the request date. But for an existing subscription undergoing a plan change/renewal, a newer catalog version's pricing/terms only kick in once the plan's configured 'effective date for existing subscriptions' has passed; until then, the subscription keeps using the older catalog version's terms for that plan (or, if no such grandfathering date is configured for a newer version, that version simply doesn't apply to existing subscriptions and the search continues to older versions).
**Specification:**
  Given Plan 'gold-monthly' is repriced in catalog version effective 2026-01-01, with effectiveDateForExistingSubscriptions=2026-04-01; an existing subscription had its plan chosen on 2025-01-01
  When  The system resolves which plan/pricing to use for that existing subscription on a change dated 2026-02-15
  Then  The 2026-01-01 catalog version's new pricing is NOT applied (still before 2026-04-01); the previous catalog version's plan definition is used instead
**Parameters:** effectiveDateForExistingSubscriptions (per-plan, optional; null means grandfathering never applies and existing subs never see that version's changes for this plan)
**Confidence:** High

### RULE-169: Default billing alignment falls back to ACCOUNT
**Category:** Policy
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:138-140`
**Plain English:** If the catalog's billing-alignment rules don't match a specific case for a given plan phase, the system defaults to aligning that subscription's billing cycle day to the account level.
**Specification:**
  Given A catalog with no <billingAlignmentCase> matching a particular plan/phase combination
  When  getBillingAlignment() is called for that plan phase
  Then  BillingAlignment.ACCOUNT is returned
**Confidence:** High

### RULE-170: OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation
**Category:** Policy
**Priority:** P0
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:109-113`
**Plain English:** If an account is tagged with OVERDUE_ENFORCEMENT_OFF, the overdue state machine is never re-evaluated for it — the account effectively can never enter an overdue state regardless of unpaid balance.
**Specification:**
  Given An account tagged OVERDUE_ENFORCEMENT_OFF with a large unpaid balance past all overdue day thresholds
  When  The overdue refresh job runs for that account
  Then  refreshWithLock returns immediately with no state transition and no evaluation of billing state at all
**Confidence:** High

### RULE-171: Catalog rule-case matching (first match, wildcard nulls)
**Category:** Policy
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java:47-87`
**Plain English:** Catalog rule tables (change policy, change alignment, cancel policy, create alignment, billing alignment, price-list transition) are evaluated as an ordered list of cases; a case's product/productCategory/billingPeriod/priceList fields act as wildcards when left unset (null), and the array is scanned in declaration order, returning the result of the first case whose non-null fields all match the plan being evaluated.
**Specification:**
  Given A changeCase array [Case1: fromProduct=null,toProduct=null (a catch-all default), Case2: fromProduct=Gold,toProduct=Silver->policy=IMMEDIATE] declared in that order in the catalog XML
  When  A plan change from Gold to Silver is evaluated
  Then  Case1 is checked first; since all of its fields are null (wildcard) it matches immediately and its result is returned, even though Case2 is a more specific match further down the list
**Parameters:** None (order-dependent, driven entirely by XML declaration order)
**Edge cases handled:**
- Because case-order controls precedence, placing a wildcard/default case before a specific case silently makes the specific case unreachable
**Suspected defect:** Rule matching is purely positional (first-match-wins) rather than most-specific-match-wins; catalog authors must manually order rules from most specific to least specific or a broad default rule can shadow a narrower one.
**Confidence:** Medium — Should catalog rule-case matching pick the most specific match instead of the first matching entry in XML order, to avoid a default/wildcard rule accidentally shadowing a more specific one?

### RULE-172: Default plan-creation alignment and cancellation policy
**Category:** Policy
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:126-135`
**Plain English:** If no catalog rule case matches a plan for creation alignment, new subscriptions default to aligning to the start of the bundle; if no case matches for cancellation, the plan defaults to cancelling at END_OF_TERM (i.e. non-immediate).
**Specification:**
  Given A catalog with no explicit <createAlignmentCase> or <cancelPolicyCase> matching a given plan
  When  A new subscription is created on that plan, or later cancelled
  Then  Plan creation aligns to PlanAlignmentCreate.START_OF_BUNDLE and cancellation defaults to BillingActionPolicy.END_OF_TERM
**Parameters:** Defaults: START_OF_BUNDLE (creation), END_OF_TERM (cancellation)
**Confidence:** High

### RULE-173: Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/generator/UsageInvoiceItemGenerator.java:231-234`
**Plain English:** When updating the per-subscription next-usage-notification date after generating usage invoice items, the system only supports IN_ARREAR usage billing; if it is ever asked to do this for IN_ADVANCE usage billing it throws an unimplemented-state error instead of scheduling anything.
**Specification:**
  Given A catalog usage section configured with billingMode=IN_ADVANCE
  When  updatePerSubscriptionNextNotificationUsageDate is invoked for that usage's billing mode
  Then  An IllegalStateException('Not implemented Yet)') is thrown, so usage-in-advance invoicing cannot currently complete its next-notification bookkeeping
**Parameters:** None
**Edge cases handled:**
- This is a hard runtime failure, not a graceful validation error — any live catalog with IN_ADVANCE usage sections billed via this path will crash invoice generation
**Confidence:** Medium — P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: UsageInvoiceItemGenerator.java:231-234 does exactly what the rule claims — updatePerSubscriptionNextNotificationUsageDate throws `new IllegalStateException("Not implemented Yet)")` (typo and all) when usageBillingMode == BillingMode.IN_ADVANCE, before doing any notification-date bookkeeping.

However this is not a P0 (money-moving / regulatory / data-integrity) rule. Tracing the only call site (line 212) shows the method is invoked with a hardcoded BillingMode.IN_ARREAR literal — never IN_ADVANCE. Further, line 148 (`event.getUsages().stream().anyMatch(input -> input.getBillingMode() == BillingMode.IN_ARREAR)`) shows this whole generator explicitly filters subscriptions down to IN_ARREAR usage before any of this logic runs; IN_ADVANCE-billed usage sections (which the catalog layer does support — see DefaultUsage.java:213-218 validating IN_ADVANCE CAPACITY/CONSUMABLE sections) are never routed into this class at all. So the throw at 231-234 is dead/unreachable defensive code in the current production call graph, not a live guard protecting an active money-moving or notification-scheduling flow.

Judged for compliance: this doesn't calculate a charge, enforce a tax/regulatory rule, or protect ledger/invoice integrity for any currently-exercised path — it's an internal "feature not yet built" marker for a code path nothing calls today. If this exception's text or behavior changed silently in a rewrite, no regulator, auditor, or finance controller would notice any difference in real invoices, payments, or account balances, because IN_ADVANCE usage billing simply isn't wired into invoice generation in the first place. This is better classified as a P2/P3 feature-gap/tech-debt item (flag for the modernization team so IN_ADVANCE usage support isn't silently "completed" without also implementing this bookkeeping), not a P0 behavior contract. No prompt-injection or manipulative text was found in the cited lines or their surrounding context. | The literal claim is accurate but the Given/When/Then materially misrepresents reachability, and the rule shouldn't carry a P0 rating.

What the code actually does (UsageInvoiceItemGenerator.java:231-234):
```java
private void updatePerSubscriptionNextNotificationUsageDate(final UUID subscriptionId, final Map<String, LocalDate> nextBillingCycleDates, final BillingMode usageBillingMode, final Map<UUID, SubscriptionFutureNotificationDates> perSubscriptionFutureNotificationDates) {
    if (usageBillingMode == BillingMode.IN_ADVANCE) {
        throw new IllegalStateException("Not implemented Yet)");
    }
    ...
```
This is a private method with exactly one call site in the whole file (line 212):
```java
updatePerSubscriptionNextNotificationUsageDate(sub.getSubscriptionId(), subscriptionResult.getPerUsageNotificationDates(), BillingMode.IN_ARREAR, perSubscriptionFutureNotificationDates);
```
The `usageBillingMode` argument is a hardcoded `BillingMode.IN_ARREAR` literal, not something derived from the catalog's usage billing mode at runtime. Additionally, upstream at line 148 (`event.getUsages().stream().anyMatch(input -> input.getBillingMode() == BillingMode.IN_ARREAR)`), a subscription is only added to `subsUsageInArrear` — and therefore only reaches this method at all — if it has at least one IN_ARREAR usage event. A search of the whole invoice module confirms `BillingMode.IN_ADVANCE` never appears anywhere in this generator except in the dead branch itself; there is no other code path in `invoice/` that handles IN_ADVANCE usage. So even though the catalog layer (`DefaultUsage.java:213-217`) explicitly allows configuring a Usage section with `billingMode=IN_ADVANCE`, that catalog setting can never cause this specific method to be invoked with `BillingMode.IN_ADVANCE` — the branch is unreachable dead code given the current call graph, not a live guard that fires when a customer configures IN_ADVANCE usage.

The rule card's Given/When/Then ("Given a catalog usage section configured with billingMode=IN_ADVANCE / When updatePerSubscriptionNextNotificationUsageDate is invoked for that usage's billing mode") implies the method's behavior is driven by the catalog's configured billing mode. It isn't — the parameter is a compile-time constant at the sole call site, decoupled from any catalog value. That's a meaningful fidelity gap for a rule meant to anchor verification: a modernization team could waste effort trying to reproduce "configure IN_ADVANCE usage → exception thrown" through the normal invoice-generation entry point and never observe it, because that path is structurally unreachable today.

On P0: this is a defensive "not implemented yet" throw in unreachable code, not something that moves money, enforces a regulatory requirement, or actively guards data integrity in the current control flow. The real underlying business fact worth capturing (that IN_ADVANCE usage billing is effectively unsupported by the invoice generator, per line 148's filter) is more accurately a P2/P3 "known limitation / feature gap" note, not a P0 behavior contract.

No prompt-injection-shaped text was found in the cited lines or surrounding code — the exception message "Not implemented Yet)" is a plain (if oddly punctuated) developer message, not an attempt to steer analysis.) — confirm criticality.

### RULE-174: Zero-amount recurring and fixed items are excluded from the repair tree
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:91-115`
**Plain English:** When rebuilding the invoice-item repair tree for a subscription, existing RECURRING items with a $0 amount are set aside and never repaired/adjusted, and existing FIXED items are always set aside untouched — only non-zero RECURRING and REPAIR_ADJ items participate in the repair/proration tree logic.
**Specification:**
  Given An existing invoice item of type RECURRING with amount=$0.00 for a subscription being re-invoiced
  When  The subscription's item tree is rebuilt for the new invoice
  Then  That $0 RECURRING item is added to the 'existingIgnoredItems' list and is never split, repaired, or replaced by the tree logic
**Parameters:** None
**Edge cases handled:**
- FIXED items are always ignored by the tree regardless of amount
**Confidence:** High

### RULE-175: Item adjustments linked to an ignored item are themselves dropped
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java:121-133`
**Plain English:** When building the repair tree, any pending ITEM_ADJ adjustment whose linked item was previously set aside as ignored (e.g. a $0 recurring item) is discarded rather than applied, since there is nothing left in the tree for it to adjust.
**Specification:**
  Given A $0 RECURRING item (ignored) that later received an ITEM_ADJ adjustment on disk
  When  The tree is built via SubscriptionItemTree.build()
  Then  That ITEM_ADJ is skipped entirely (root.addAdjustment is never called for it)
**Parameters:** None
**Confidence:** High

### RULE-176: Overdue re-evaluation notification scheduling
**Category:** Policy
**Priority:** P1
**Source:** `overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:104-125,160-174`
**Plain English:** After applying a new overdue state, the system schedules a future re-check: if the account just moved to CLEAR, it uses the overdue config's initialReevaluationInterval (only if the account still has an unpaid invoice and at least one overdue state is configured); otherwise it uses the new state's own configured autoReevaluationInterval. If no reevaluation interval is configured at all, no future check is scheduled (the assumption being the overdue conditions aren't time-based).
**Specification:**
  Given An account transitions into overdue state OD2, which has autoReevaluationInterval=P7D configured
  When  OverdueStateApplicator.apply() runs at effectiveDate=2026-06-01
  Then  A future notification is scheduled for 2026-06-08 (2026-06-01 + 7 days) to re-run the overdue evaluation
**Parameters:** initialReevaluationInterval (account-level default), autoReevaluationInterval (per-state, e.g. P7D)
**Edge cases handled:**
- If nextOverdueState is CLEAR and there is no unpaid invoice (or no overdue states configured at all), no future notification is created and the existing one is cleared instead
- OVERDUE_NO_REEVALUATION_INTERVAL exception from config is treated as 'no reschedule needed', not an error
**Confidence:** High

### RULE-177: RBAC permission check supports AND/OR logic across required permissions
**Category:** Policy
**Priority:** P0
**Source:** `util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java:177-204`
**Plain English:** When an API call requires a list of permissions, the caller must satisfy all of them (AND) or at least one of them (OR), depending on how the endpoint is annotated; a single-permission check is a simple pass/fail.
**Specification:**
  Given A JAX-RS endpoint requires permissions [ACCOUNT_CAN_VIEW, INVOICE_CAN_VIEW] with Logical.OR
  When  The current subject holds only INVOICE_CAN_VIEW
  Then  The permission check passes because at least one of the OR'd permissions is held; had Logical.AND been used instead, the same subject would fail with SECURITY_NOT_ENOUGH_PERMISSIONS
**Parameters:** Logical.AND / Logical.OR enum, ErrorCode.SECURITY_NOT_ENOUGH_PERMISSIONS
**Confidence:** High

### RULE-178: Bundle transfer requires the bundle to actually belong to the declared source account
**Category:** Policy
**Priority:** P0
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:313-323`
**Plain English:** When transferring a bundle (by external key) between accounts, the system looks up the bundle's actual owning account and rejects the transfer if it doesn't match the caller-supplied source account id.
**Specification:**
  Given A transfer request with sourceAccountId=A and bundleExternalKey 'bike-club', but the active subscription for that key actually belongs to account B
  When  transferEntitlementsOverrideBillingPolicy executes the transfer
  Then  An EntitlementApiException wrapping SUB_GET_INVALID_BUNDLE_KEY is thrown and no transfer occurs
**Parameters:** n/a
**Confidence:** High

### RULE-179: Bundle transfer billing policy determines whether the source subscription is cancelled immediately or at end of term
**Category:** Policy
**Priority:** P1
**Source:** `entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:298-311`
**Plain English:** A bundle transfer's billing policy (IMMEDIATE or END_OF_TERM) directly controls whether the original subscription is cancelled right away or allowed to run to the end of its current billed term; any other policy value is treated as a programming error.
**Specification:**
  Given A transfer request with billingPolicy=IMMEDIATE
  When  transferEntitlementsOverrideBillingPolicy runs
  Then  cancelImm is set true and the source subscription's cancellation is immediate rather than deferred to end-of-term
**Parameters:** BillingActionPolicy {IMMEDIATE, END_OF_TERM}
**Confidence:** High

### RULE-180: Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first
**Category:** Policy
**Priority:** P1
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:1198-1227`
**Plain English:** Deleting a used-credit consumption line just zeroes it out. Deleting a credit-generation line only works if it can be tied back to an explicit CREDIT_ADJ item on a credit invoice; if the account has already spent down some of that credit elsewhere, the shortfall is reclaimed from other invoices before the delete proceeds. Credit generated automatically by the system (e.g. from a repair) cannot be deleted at all.
**Specification:**
  Given A $30 system-generated CBA credit item from a repair invoice with no matching CREDIT_ADJ item
  When  deleteCBA is called on that item
  Then  INVOICE_CBA_DELETED error is thrown; deletion is refused
**Parameters:** n/a
**Edge cases handled:**
- Account CBA balance lower than the item amount triggers reclaim from other invoices before delete completes
**Confidence:** Medium — Is it intended that reclaiming credit from other invoices as a side effect of a CBA delete can alter balances on invoices unrelated to the one being edited?

### RULE-181: Refreshing payment methods from a plugin never un-sets the KB default
**Category:** Policy
**Priority:** P1
**Source:** `payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:602-670`
**Plain English:** When syncing payment methods from a gateway plugin, Kill Bill only updates its own notion of the account's default payment method if the plugin reports a default AND the account's current default already belongs to that same plugin; a plugin reporting 'no default' never clears an existing Kill Bill default.
**Specification:**
  Given Account's current default payment method belongs to plugin A; refreshPaymentMethods is run for plugin B and plugin B reports no default
  When  updateDefaultPaymentMethodIfNeeded runs
  Then  the account default payment method is left unchanged (still plugin A's), because the current default's plugin (A) doesn't match the plugin being refreshed (B)
**Parameters:** n/a
**Edge cases handled:**
- If account has no default payment method set at all, the plugin-reported default is always applied
**Confidence:** High

### RULE-182: Catalog case-rule matching: first matching rule wins, null fields act as wildcards
**Category:** Policy
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java:54-88`
**Plain English:** Catalog rules (change policy, cancel policy, alignment, price-list rules, etc.) are evaluated as an ordered list of cases; a case matches if every field it specifies (product, product category, billing period, price list) equals the plan being evaluated, and any field left unset in the rule matches anything. The first matching case in declaration order determines the result.
**Specification:**
  Given Two change-policy cases: (1) productCategory=BASE -> IMMEDIATE, (2) no fields set (wildcard) -> END_OF_TERM, declared in that order
  When  A plan change is evaluated for a BASE product
  Then  case (1) matches first and IMMEDIATE is returned, even though the wildcard case (2) would also match
**Parameters:** n/a
**Confidence:** High

### RULE-183: Default fallbacks when no catalog rule case matches
**Category:** Policy
**Priority:** P1
**Source:** `catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:126-176`
**Plain English:** If catalog rule evaluation finds no matching case (which validation is supposed to prevent, see wildcard-rule requirement), Kill Bill falls back to hardcoded defaults: START_OF_BUNDLE for creation and change-plan alignment, END_OF_TERM for cancel and change-plan billing policy.
**Specification:**
  Given A catalog with a valid default rule present so this path is a safety net
  When  getPlanCreateAlignment / getPlanCancelPolicy / getBillingAlignment / getPlanChangeAlignment / getPlanChangePolicy find no matching case
  Then  PlanAlignmentCreate.START_OF_BUNDLE, BillingActionPolicy.END_OF_TERM, BillingAlignment.ACCOUNT, PlanAlignmentChange.START_OF_BUNDLE, and BillingActionPolicy.END_OF_TERM are returned respectively
**Parameters:** Defaults: START_OF_BUNDLE (create/change alignment), END_OF_TERM (cancel/change policy), ACCOUNT (billing alignment)
**Confidence:** High

### RULE-184: Parked accounts are skipped for automatic invoicing
**Category:** Policy
**Priority:** P0
**Source:** `invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java:303-320`
**Plain English:** An account tagged as 'parked' (auto-suspended due to a prior invoicing error) has all non-API-triggered (i.e. scheduled/system) invoice generation runs silently skipped; API-initiated invoice generation still proceeds even for a parked account.
**Specification:**
  Given Account X has the PARK system tag set from a previous failed invoicing run
  When  The nightly invoicing job (isApiCall=false) processes account X
  Then  invoice generation is skipped entirely and an empty invoice list is returned; a direct client API call to generate an invoice for account X, however, still runs
**Parameters:** n/a
**Edge cases handled:**
- If checking the parked-tag itself fails (TagApiException), processing proceeds as if not parked
**Confidence:** High

## Rules requiring SME confirmation

- **RULE-006** (Invoice balance formula, Medium): P0 panel doubts spec fidelity: P0 classification is justified: this is the canonical invoice-balance formula (InvoiceCalculatorUtils.computeRawInvoiceBalance, lines 96-110) that determines the amount owed on every invoice, feeding dunning/overdue logic and payment reconciliation across the whole system — it moves money and guards ledger integrity.

However the Given/When/Then is NOT faithful — the worked numeric example is wrong because it mishandles the sign of refunds/chargebacks.

What the code actually does:
- `computeRawInvoiceBalance` (96-110): `amountPaid = computeInvoiceAmountPaid(...).add(computeInvoiceAmountRefunded(...))`; `chargedAmount = computeInvoiceAmountCharged + computeInvoiceAmountCredited + computeInvoiceAmountAdjustedForAccountCredit`; `balance = chargedAmount - amountPaid`, then rounded via `KillBillMoney.of` (HALF_UP, confirmed at util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:27,34).
- `computeInvoiceAmountRefunded` (225-242) sums `InvoicePayment.getAmount()` for REFUND/CHARGED_BACK items with SUCCESS status — and those amounts are stored as **negative** numbers at creation time: `invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:877` (`requestedPositiveAmount.negate()`) for refunds and `:953` (`requestedChargedBackAmount.negate()`) for chargebacks, with no sign flip anywhere in `DefaultInvoicePayment.getAmount()` (invoice/src/main/java/org/killbill/billing/invoice/model/DefaultInvoicePayment.java:104-106).

So for the example given (charged=$100, CBA adj=-$10, invoice credit=$0, payment=+$50, refund=$5):
- chargedAmount = 100 + (-10) + 0 = $90.00 — this part of the rule card is correct.
- amountPaid = 50 + (refund stored as -5) = $45.00 — NOT $55.00 as the rule card computes.
- balance = 90 - 45 = **$45.00**, not the $35.00 the rule card states.

The rule card's arithmetic treats the $5 refund as a positive amount that gets added to the payment and subtracted again from the charged total (paid + refund, both positive, both reducing balance). In the real code a refund is a negative payment amount, so it partially cancels the original payment inside `amountPaid` — meaning a refund *increases* the outstanding balance (correct business behavior: refunding money paid means less of the invoice is actually paid off), not decreases it further. The plain-English summary ("...minus everything paid and refunded/charged-back so far") also states the wrong direction — refunds/chargebacks should net against payments (reducing the amount treated as paid), not stack as an additional independent deduction.

Everything else cited is faithful: computeInvoiceAmountCharged correctly aggregates isCharge, isInvoiceAdjustmentItem, isInvoiceItemAdjustmentItem, and isParentSummaryItem items (157-174); computeInvoiceAmountCredited sums CBA_ADJ (192-205); computeInvoiceAmountPaid only counts SUCCESS ATTEMPT payments (207-223); rounding is HALF_UP as claimed.

No prompt-injection attempts were found in the cited code (comments are standard Apache license headers and plain code comments).
- **RULE-012** (Negative invoice balance auto-generates account credit, Medium): P0 panel doubts spec fidelity: Compliance lens: P0 is justified. The code at CBADao.java:69-71 (inside the cited 63-76 range) auto-generates a CreditBalanceAdjInvoiceItem whenever an invoice balance goes negative, and the companion branch (73-84) auto-consumes existing account credit against a positive COMMITTED-invoice balance. Both branches directly move recorded money (create a persisted, immutable ledger-affecting invoice item via createCBAItem -> transInvoiceItemDao.create, line 237) and enforce the double-entry invariant that invoice balances reconcile against account credit. A silent change here (wrong sign, wrong trigger condition, or skipped generation) would misstate customer balances and account credit — exactly what a finance controller or auditor would flag. So p0Justified = true.

However, the rule card is NOT faithful in its concrete Given/When/Then math, so I flag it: it states 'A CreditBalanceAdjInvoiceItem for -(-$25.00) = -$25.00 stored amount'. That arithmetic is self-contradictory: -(-25.00) equals +25.00, not -25.00. Tracing the actual code confirms the correct value is +25.00, not -25.00: in computeCBAComplexity (line 69-71) the negative branch calls buildCBAItem(invoice, balance, context) with balance = -25.00; inside buildCBAItem (lines 243-251) the stored amount is amount.negate() = -(-25.00) = +25.00. The code comment at line 68 itself says 'we need to generate a credit (positive CBA amount)', confirming the stored amount must be positive. The plain-English summary ('effectively adding $25.00 of usable account credit') is directionally correct and matches +25.00, but the formal Given/When/Then computation the card presents for verification is wrong-signed and internally inconsistent. If used as-is to write an equivalence/regression test post-migration, an engineer would assert the wrong signed value. This is a load-bearing sign error for a P0 money-moving rule and must be corrected before the card is trusted (correct form: stored amount = -1 x (-$25.00) = +$25.00).

Also note: the card only documents the negative-balance branch of computeCBAComplexity; it omits the equally significant positive-balance branch (lines 73-84) where existing account CBA is automatically applied (negative CBA amount) against a COMMITTED invoice's positive balance, gated on no pending ATTEMPT payment and the invoice not being written off. That omission isn't a fabrication, but a P0-complete rule card for this method should cover both money-moving branches, not just one.

No prompt-injection-shaped text was found in the cited lines (63-76, 243-251) — comments are ordinary implementation notes (e.g. 'PERF:', 'we need to generate a credit (positive CBA amount)'), not directives aimed at an AI reviewer.

Files reviewed: /Users/theobeack/Repo/killbill/invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java (lines 1-120, 230-251). | The trigger condition and qualitative behavior are faithful (invoice/dao/CBADao.java:74-76: balance < 0, unconditionally, generates a CreditBalanceAdjInvoiceItem via buildCBAItem regardless of invoice status/pending payments), but the concrete Given/When/Then value is arithmetically wrong. buildCBAItem (lines 243-251) stores amount.negate(); for a balance of -$25.00, negate() yields +$25.00, not -$25.00 as the rule states ('-(-$25.00) = -$25.00'). This also contradicts CreditBalanceAdjInvoiceItem.java:58-62 where a positive amount is labeled 'account credit' (matching +$25.00) and a negative amount is 'use of account credit'. The card's own plain-English conclusion ('adding $25.00 of usable account credit') is correct but is contradicted by its own Then-clause's stated stored value, making the card internally inconsistent and unsafe as a literal test oracle. P0 is justified: this code creates a real monetary ledger item (CBA_ADJ) representing usable account credit, so the exact sign must be preserved during modernization. No instruction-shaped/injection text was found in the cited lines (56, 75, 81, 94 are ordinary code comments).
- **RULE-014** (Child-account invoice balance nets against the parent invoice's charge for that child, Medium): Confirm this parent/child netting is intentional double-accounting protection and not a workaround for a known parent-invoice sync bug.
- **RULE-024** (Chargeback amount/currency resolution fallback, Medium): Is silently falling back to the original payment amount/currency (rather than rejecting or flagging) the intended behavior when the gateway reports a chargeback in a different currency?
- **RULE-025** (Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total, Medium): Confirm the 2-item CREDIT_ADJ + offsetting CBA_ADJ pattern is the sole authoritative signature of a 'credit invoice' for reporting/balance purposes, since it's inferred purely from item count/type/amount matching rather than an explicit invoice flag.
- **RULE-049** (Same-day usage items are excluded when re-billing a larger period, Medium): Confirm this doesn't cause the same-day usage record to be double-counted in the larger period's total, versus intentionally re-consolidating it.
- **RULE-054** (Usage/product limit compliance check (min/max), Low): Is DefaultLimit.compliesWith's min check intentionally inverted (i.e. does 'min' actually mean an upper bound for some unit types), or is `value.compareTo(min) <= 0` a bug that should be `>= 0`? Also confirm whether any plugin/consumer actually calls compliesWith/compliesWithLimits in production.
- **RULE-057** (Account external key, currency, BCD, timezone, and reference time are immutable once set, Medium): P0 panel doubts spec fidelity: P0 is justified: `validateAccountUpdateInput` (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:502-546) is a data-integrity guard on core account attributes (currency, external key, BCD, timezone, reference time) that feed invoicing/proration/tax logic downstream — corrupting them silently would corrupt billing.

The headline Given/When/Then (currency USD->EUR throws "Killbill doesn't support updating the account currency yet") is faithful and matches lines 521-526 verbatim, including the exact exception string.

However the card's generalization is not faithful in two respects, so I mark it non-faithful overall:

1. The card omits `ignoreNullInput` as a parameter even though it is the actual governing switch for all four ignoreNullInput-gated fields (externalKey 514-519, currency 521-526, BCD 528-532, timeZone 534-539). Each check is gated by `(ignoreNullInput || fieldNotNull)`: when `ignoreNullInput=false`, supplying a null new value bypasses the check entirely (silently allowed, i.e. "reset" semantics per the code's own comment at 508-511); when `ignoreNullInput=true`, a null new value against a non-null current value still throws. The card's plain-English summary ("an update attempting to change any of those fields to a different value is rejected") only describes the mismatch case and drops this null-handling distinction, which is a real behavioral fork, not incidental.

2. The card claims "the same pattern applies to ... referenceTime," but referenceTime (541-544) does NOT use `ignoreNullInput` at all — it only checks `referenceTime != null && ...date-truncated compare... != 0`. A null new referenceTime is unconditionally skipped regardless of `ignoreNullInput`, unlike the other four fields where `ignoreNullInput=true` forces the check even on null input. So referenceTime is categorically different, not "the same pattern" with a mere date-only-compare footnote.

Net: the single cited example is correct, but the rule as generalized across all five fields overstates uniformity and omits a parameter (`ignoreNullInput`) that materially changes whether a reset-to-null is accepted or rejected — this needs correction before it becomes a verification contract for the rewrite. No prompt-injection attempts were found in the cited code; comments at 504-513 are plain explanatory text.
- **RULE-059** (Payment transaction external key uniqueness and account isolation, Medium): P0 panel doubts spec fidelity: P0 justified (compliance lens): The rule guards two things a finance controller/auditor would flag if silently broken. (1) Duplicate-payment prevention — `runSanityOnTransactionExternalKey` (PaymentProcessor.java:327-350) blocks re-use of an external key for a second SUCCESS, non-CHARGEBACK transaction, which is the idempotency guard that stops a retried/duplicated client request from moving money twice (double-charge risk). (2) Account isolation — it also blocks a transaction key that belongs to a different account (`accountRecordId` mismatch), which prevents payment records from being misattributed across tenants/accounts, a data-integrity/financial-reporting correctness issue. Either failure mode (double-charge or cross-account misattribution) is exactly the kind of silent behavior change a payments auditor would want caught, so P0 is warranted.

However, the rule card is NOT faithful to the code as written, so it should not be trusted verbatim before verification-suite authoring. Re-reading PaymentProcessor.java:330-348: the loop runs two INDEPENDENT if-checks per existing transaction with the same key, not an if/else-fallback pair as the card's Given/When/Then implies. Line 330-336 throws PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS whenever an existing transaction under the key is SUCCESS and non-CHARGEBACK — with NO account check at all. Line 338-348 (a separate, unconditional check on the same record) throws PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID whenever the record's account differs from the caller's, regardless of that record's status/type — but it is only reached when the first `if` was false (i.e., status != SUCCESS or type == CHARGEBACK). Concretely: the card's own example — 'existing txn-001 already SUCCESS/PURCHASE for account A, reused by a different account' — would actually throw PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS (line 335), NOT PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID as the card's Then-clause states, because the SUCCESS+non-CHARGEBACK branch fires first and does not check account at all. The DIFFERENT_ACCOUNT_ID error only fires for same-key records that are not(SUCCESS && non-CHARGEBACK) — e.g., a PENDING/FAILED or CHARGEBACK transaction — combined with a different account. This precedence/ordering detail must be corrected before the rule card is used as a behavior-equivalence oracle, or a rewrite that 'fixes' the example scenario to return DIFFERENT_ACCOUNT_ID would actually be diverging from legacy behavior. No injection-shaped text was found in the cited lines. | P0 classification is justified: this rule guards data integrity (prevents cross-account key collisions and duplicate processing of the same external transaction key), which is exactly the kind of thing that must be behavior-equivalent post-rewrite.

However the Given/When/Then is NOT faithful to the code's actual branching, specifically on the "instead" framing.

Read code at payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java:327-350:

```
330: for (final PaymentTransactionModelDao paymentTransactionModelDao : allPaymentTransactionsForKey) {
332:     if (...key.equals(...) && status == SUCCESS && type != CHARGEBACK) {
335:         throw PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS;
336:     }
339:     if (!accountRecordId.equals(internalCallContext.getAccountRecordId())) {
347:         throw PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID;
348:     }
350: }
```

These are two independent, sequential `if` blocks per iterated transaction — NOT an if/else fork keyed on account. Check 1 (SUCCESS + non-CHARGEBACK) is evaluated first and does not reference account at all. So for the exact scenario the rule card poses — an existing txn under the key that is already SUCCESS and type PURCHASE — check 1 fires and throws PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS regardless of whether the new request's account matches the existing transaction's account. The card's "if it belongs to a different account, it instead fails with PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID" describes an outcome that cannot occur in that scenario: check 1 preempts check 2 whenever the stored transaction is SUCCESS/non-CHARGEBACK.

PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID only fires for a matching-key transaction that is NOT (SUCCESS AND non-CHARGEBACK) — e.g. PENDING, FAILED, or a CHARGEBACK-type record — and belongs to a different account. The rule card conflates the two checks as alternate outcomes of the same condition set, when they're actually gated by disjoint status/type conditions, with KEY_EXISTS taking strict precedence whenever both conditions would otherwise be true. There's also an unaddressed edge case: allPaymentTransactionsForKey can contain multiple rows (multiple prior attempts under the same key, possibly across accounts), and the actual thrown error depends on DB iteration order among those rows — the card's GWT assumes a single existing transaction.

No prompt-injection content was found in the cited lines (327-350) — comments are ordinary engineering notes (e.g. line 331 chargeback caveat, line 361 sanity-check note), not instruction-shaped text aimed at automated analysis.

Recommendation: rewrite the GWT to reflect the true precedence — "KEY_EXISTS is thrown if the matching-key transaction is SUCCESS and not CHARGEBACK, checked before and independent of account match; DIFFERENT_ACCOUNT_ID is thrown only when the matching-key transaction fails that SUCCESS/non-CHARGEBACK test and belongs to a different account" — and flag the multi-row iteration-order dependency for SME confirmation (confidence should be Medium, not High, given this ordering subtlety).
- **RULE-060** (At most one PENDING initial payment transaction per payment, Medium): P0 panel doubts spec fidelity: Re-derived from PaymentProcessor.java:352-385 (invoked only when paymentId is set and either transactionId or paymentTransactionExternalKey is present, per line 297).

What the code actually does, iterating over paymentTransactionsForCurrentPayment (352-385):
- For each existing transaction that does NOT match the incoming call's transactionId/externalKey (359-360): if that OTHER transaction's type is AUTHORIZE/PURCHASE/CREDIT and its status is PENDING, throw PAYMENT_INVALID_OPERATION(existingTxnType, paymentStateName) (362-366). Otherwise skip it (368).
- For the transaction that DOES match the incoming id/key (372-380): if its type differs from the incoming request's transactionType, throw PAYMENT_INVALID_PARAMETER("transactionType", ...) (373-375); if its status is PENDING or UNKNOWN, add it to completionCandidates (378-380).
- After the loop: Preconditions.checkState(candidates.size() <= 1) and return the last (or only) candidate (383-385).

Confirmed correct pieces of the rule: the type-mismatch-on-matching-key check (PAYMENT_INVALID_PARAMETER, 372-375) and the checkState single-completion-candidate invariant (383) are described accurately, in the right order.

The inaccuracy is in the "When" clause: the rule states the guard fires for "a new AUTHORIZE/PURCHASE/CREDIT request." That is too narrow. The gating condition at 362-366 tests the TYPE OF THE EXISTING PENDING TRANSACTION, not the type of the incoming request. I verified this by checking all callers of performOperation (lines 103-142): captureAuthorization, voidPayment, refundPayment, and chargeback all pass a paymentTransactionExternalKey for their own new transaction (a key that won't match an unrelated pending AUTHORIZE), so any of these non-initiating transaction types will also hit the PAYMENT_INVALID_OPERATION throw at line 366 if the payment has some other PENDING AUTHORIZE/PURCHASE/CREDIT transaction under a different id/key — not just new AUTHORIZE/PURCHASE/CREDIT requests as the card states. So the Given/When/Then materially understates the scope of what gets rejected; a verification suite built only against "AUTHORIZE/PURCHASE/CREDIT incoming requests" would miss CAPTURE/VOID/REFUND/CHARGEBACK cases the code actually blocks.

P0 justification stands independent of the faithfulness issue: this logic guards against a payment ending up with two concurrently outstanding (PENDING) money-moving transactions and against completing/capturing the wrong transaction — a direct data-integrity and double-charge-prevention control on the payment core, which is exactly the kind of rule that must be preserved and behavior-tested across a rewrite.

No prompt-injection-shaped text found in the cited range (302-385) or the surrounding methods read (200-390); comments are plain engineering notes (e.g. "Sanity: ...", "cannot be enforced by the state machine unfortunately").
- **RULE-061** (Add-on creation eligibility against base plan, Medium): P0 panel split on whether this moves money / is regulatory (I read AddonUtils.java:36-53 (and the caller SubscriptionApiBase.java:229-245 for context).

FAITHFULNESS ISSUE FOUND: The extracted rule's plain-English description of the PENDING-base branch has the date comparison backwards. Code at line 39:
`baseSubscription.getState() == EntitlementState.PENDING && context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0`
throws SUB_CREATE_AO_BP_NON_ACTIVE when baseStartDate < requestedDate (i.e., the add-on's requested/effective date is AFTER the base's start date). The rule card states the add-on is allowed when its "start is not before the base start" (addonStart >= baseStart) — but that is exactly the condition the code REJECTS, not permits. Cross-checking the caller (SubscriptionApiBase.java:241-245), effectiveDate is already guaranteed >= baseSubscription.getStartDate() before checkAddonCreationRights is even called (a separate SUB_INVALID_REQUESTED_DATE check enforces that). So the real, composite rule is: for a PENDING base, the add-on's effective date must equal the base's start date exactly — any later date throws BP_NON_ACTIVE. The rule card's inequality direction is inverted, which would produce a wrong equivalence test in modernization (e.g., asserting a later add-on date succeeds when the code actually rejects it). The CANCELLED-state, ALREADY_INCLUDED, and NOT_AVAILABLE clauses (lines 38, 45-53) are accurately described and match isAddonIncluded/isAddonAvailable (lines 56-77).

COMPLIANCE JUDGMENT: This rule enforces catalog-driven product entitlement (base-must-be-active, no duplicate-of-included-product, add-on-must-be-in-catalog's-available-list). It does protect internal subscription/catalog data integrity and indirectly prevents billing for unauthorized product combinations, but it is not something a regulator, external auditor, or finance controller would specifically monitor — there's no tax, AML/KYC, PCI, SOX financial-control, or statutory retention/disclosure obligation here. It's a domain/product business rule (entitlement correctness), not a compliance control. I'd downgrade this from P0-compliance to P1 (business-critical correctness rule) rather than treat it as a regulator-relevant control.

No prompt-injection-shaped text found in the cited lines (36-53) — only a standard Apache license header at the top of the file, unrelated to the cited range.

Files reviewed: /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:34-90, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:229-245.</reason>
</invoke>
 | Re-derived directly from AddonUtils.java:36-53 (confirmed against DefaultSubscriptionBase.getState()/getStartDate(), Entitlement.EntitlementState enum, ErrorCode.java, and the caller SubscriptionApiBase.java:229-245).

What's faithful:
- The three checks fire in the stated order and with the stated error codes: CANCELLED/bad-PENDING -> SUB_CREATE_AO_BP_NON_ACTIVE (line 38-40); already-included -> SUB_CREATE_AO_ALREADY_INCLUDED (line 45-48); not-available -> SUB_CREATE_AO_NOT_AVAILABLE (line 50-53).
- isAddonIncluded/isAddonAvailable (lines 56-77) do exactly what's described: linear name-match against baseProduct.getIncluded()/getAvailable() from the catalog.
- Parameters: correctly "None" — driven entirely by catalog data, no hardcoded magic numbers/credentials.

Two fidelity defects against the cited code:

1. Base-state gate is narrower than described. EntitlementState has 5 values: PENDING, ACTIVE, BLOCKED, CANCELLED, EXPIRED (Entitlement.java). Line 38-39 only rejects CANCELLED (plus the PENDING edge case below) — BLOCKED and EXPIRED base subscriptions are NOT rejected by this method and would fall through to the included/available checks. The card's plain English ("base subscription is active... or pending-future...") implies an active-or-nothing gate that the code does not actually enforce for BLOCKED/EXPIRED bases.

2. The PENDING sub-condition's direction is inverted. Line 39: `baseStartDate.compareTo(requestedDate) < 0` is true when baseStartDate is chronologically BEFORE requestedDate — i.e. it throws when the add-on's requested date falls AFTER the base's own (future) start date, not when it falls before. The card states the allowed condition as "the add-on start is not before the base start" (i.e., disallowed only when addon-start < base-start), which is the opposite polarity of what this function does. The "addon start before base start" case is actually guarded by a different check in a different file — SubscriptionApiBase.java:241-242 (`effectiveDate.isBefore(baseSubscription.getStartDate())` -> SUB_INVALID_REQUESTED_DATE) — which is outside the cited range and uses a different error code entirely. The card conflates the two checks and mis-states the direction of the one it cites.

P0 justification stands regardless: this rule (in its correct form) gates creation of billable add-on items against subscription state and catalog constraints — letting an add-on attach to a cancelled/expired/blocked base, or to a product already included free, or not listed as available, would corrupt the subscription/entitlement state machine and produce incorrect invoice line items. That is a genuine data-integrity guard feeding billing, so P0 is justified even though the Given/When/Then needs correction before it can serve as a verification contract.

Recommendation: rewrite the card's PENDING branch as "Given base state PENDING (start date in the future) and add-on requested date's calendar day is after the base's start date, When checkAddonCreationRights runs, Then SUB_CREATE_AO_BP_NON_ACTIVE is thrown" and drop "active" framing in favor of "not CANCELLED, and not (PENDING with requested date after base start)" — noting BLOCKED/EXPIRED bases pass this specific check.

Files reviewed: /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java:36-53, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java:187-205, /Users/theobeack/Repo/killbill/subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java:223-246. No prompt-injection-shaped text found in the cited lines.) — confirm criticality.
- **RULE-072** (Tenant API key must be unique, Medium): Is it intentional that only IllegalStateException-caused lookup failures are treated as 'key available', while any other runtime error during the pre-check aborts creation even though a DB constraint would have caught true duplicates?
- **RULE-080** (Account email record add is idempotent by record id, not by email address, Medium): Is duplicate-email-address prevention meant to be enforced elsewhere (e.g. API layer, DB constraint), or is allowing multiple identical email addresses per account intentional?
- **RULE-099** (Catalog usage sections must define pricing structures appropriate to their billing mode, Medium): P0 panel split on whether this moves money / is regulatory (The code at DefaultUsage.java:212-229 is faithfully described: it's a catalog-load-time XML validation that rejects a Usage section if (IN_ADVANCE, CAPACITY) has zero limits, (IN_ADVANCE, CONSUMABLE) has zero blocks, or (IN_ARREAR, any type) has zero tiers, appending a ValidationError in each case. The Given/When/Then and error-message text match the source exactly (lines 213-225).

However, under the COMPLIANCE lens this does not meet the P0 bar. It is a structural completeness check on the catalog *definition* (config authoring), not a calculation that moves money, a regulatory obligation (tax, AML/KYC, disclosure, retention), or a guard on transactional data integrity. Its effect is fail-fast at catalog load: if you author an IN_ARREAR usage block with no tiers, the catalog simply won't load, so no invoice or payment can ever be generated from it. Nothing about actual billed amounts, ledger entries, or account balances is computed or guarded here — that logic lives elsewhere (e.g., the tier/block/limit pricing calculators that run against an already-valid catalog). A regulator or finance controller reviewing invoices would never observe this rule directly: if it silently regressed (e.g., the checks were removed), the worst immediate outcome is that a malformed catalog XML that should have failed to deploy instead loads — a config-quality/ops risk, not a silent mischarge, since there is no evidence in this snippet that pricing calculators would produce incorrect (rather than erroring) output against a tier/block/limit-less usage section. This belongs in the modernization inventory as a catalog-authoring validation rule (worth preserving for config quality), but it should be downgraded from P0 — verification effort for behavior-equivalence should focus on the actual pricing/proration arithmetic and invoice-generation code paths, not this schema completeness gate.

No injection-shaped text was found in the cited lines. | Re-derived directly from catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java:212-229. The code has exactly three independent conditions, each appending a ValidationError with a distinct message:
(1) billingMode==IN_ADVANCE && usageType==CAPACITY && limits.length==0 -> "Usage [IN_ADVANCE CAPACITY] section of phase %s needs to define some limits"
(2) billingMode==IN_ADVANCE && usageType==CONSUMABLE && blocks.length==0 -> "Usage [IN_ADVANCE CONSUMABLE] section of phase %s needs to define some blocks"
(3) billingMode==IN_ARREAR && tiers.length==0 (independent of usageType) -> "Usage [IN_ARREAR] section of phase %s needs to define some tiers"
followed by validateCollection(catalog, errors, limits) and validateCollection(catalog, errors, tiers) (blocks are not recursively validated here, only length-checked).

The cited Given/When/Then (IN_ADVANCE + CONSUMABLE + no blocks -> the exact ValidationError string) is an exact, faithful match to condition (2), including the literal error message text and the phase.toString() interpolation. The plain-English summary correctly states all three conditions with matching billing-mode/usage-type pairing (note IN_ARREAR is not paired with a usageType in the code, and the plain English correctly reflects that — 'any IN_ARREAR usage section'). No rounding/ordering subtleties apply here since this is boolean validation logic, not arithmetic; the three checks are independent (no short-circuiting between them) and all three would fire simultaneously if applicable, which the rule text doesn't contradict.

One caveat for injectionSuspects: none found — no instruction-shaped text in the cited lines. One caveat on the P0/data-integrity claim: the assertion that 'the catalog is rejected'/'cannot be loaded' depends on how the accumulated ValidationErrors are consumed by the XML loader (in a sibling repo, killbill-commons/xmlloader, not present in this checkout), so that specific enforcement detail could not be independently confirmed from this repo, though the ValidationErrors accumulation chain (DefaultUsage -> DefaultPlanPhase -> DefaultPlan -> StandaloneCatalog/DefaultVersionedCatalog.validate) is verified and real.

P0 justification: this rule guards data integrity of the product catalog, which is the authoritative source Kill Bill uses to compute usage-based charges. A usage section missing the pricing primitive its billing mode/type requires (blocks for IN_ADVANCE CONSUMABLE, limits for IN_ADVANCE CAPACITY, tiers for IN_ARREAR) would be unable to price usage records correctly downstream in invoicing, i.e., it prevents malformed catalog configuration from silently producing incorrect or unresolvable billing calculations. That is a legitimate data-integrity guard sitting directly upstream of money movement (invoice generation), so P0 is justified.</reason>
) — confirm criticality.
- **RULE-100** (Plan-change and plan-cancellation catalog rules require a catch-all default case, Medium): P0 panel split on whether this moves money / is regulatory (The rule text is faithful to the cited code: DefaultPlanRules.validate() (catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236) does exactly what's described — for the changePolicy table (lines 194-215) it walks every DefaultCaseChangePlanPolicy, flags exact-duplicate entries with "Duplicate rule for change plan %s" (line 196), and requires at least one entry with every match field (phaseType/fromProduct/.../toPriceList) null, emitting "Missing default rule case for plan change" (line 214) if none is found; the identical pattern repeats for cancelPolicy at lines 217-236 ("Duplicate rule for plan cancellation %s" / "Missing default rule case for plan cancellation"). Message strings, field checks, and control flow all match the source verbatim, so faithful=true.

However this is NOT a P0 rule under the compliance lens. Tracing the actual runtime consumers of these rule tables (DefaultPlanRules.java:132-135, 172-176 and DefaultCaseChange.getResult / DefaultCasePhase.getResult in the same package) shows that when no configured rule matches a plan change/cancellation, getResult() returns null and the caller falls back to a hardcoded Java default: BillingActionPolicy.END_OF_TERM for both plan-change (line 175) and cancellation (line 134), and PlanAlignmentChange.START_OF_BUNDLE for alignment (line 169). In other words, the catalog-level "default rule" requirement is a belt-and-suspenders authoring check that forces catalog XML to be maximally explicit — it does not itself determine the billing outcome. If this validation were silently removed or weakened, a catalog missing an explicit default rule would still resolve to the exact same END_OF_TERM/START_OF_BUNDLE fallback at runtime; no money-moving behavior, proration amount, or billing-action outcome would change. Likewise the duplicate-rule check flags authoring redundancy, not a case where two conflicting outcomes could silently diverge (an exact duplicate by definition yields the same result either way).

A regulator, auditor, or finance controller cares about what billing action policy is actually applied to plan changes/cancellations (since that determines whether a customer is charged/refunded, and when) — that logic lives in DefaultCaseChange/DefaultCasePhase matching and the proration code downstream, not in this XML-completeness linter. This validate() method is a legitimate data-quality/fail-fast guard for catalog authors (worth keeping and testing), but its silent removal would not produce a financially or regulatorily observable behavior change, so it does not meet the P0 bar of "moves money, enforces regulation, or guards data integrity" at the level that matters for a behavior-equivalence contract. I'd classify it as P2 (config/authoring validation) rather than P0.

No prompt-injection attempts were found in the cited range or surrounding file (only standard Apache license header and ordinary Javadoc/comments). | Re-derived directly from catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236.

Missing-default-case checks verified exactly:
- Lines 194-215 (change plan): loops changeCase[]; a case is the "default" iff phaseType, fromProduct, fromProductCategory, fromBillingPeriod, fromPriceList, toProduct, toProductCategory, toBillingPeriod, toPriceList are ALL null (lines 200-208); if none found, adds ValidationError with literal message "Missing default rule case for plan change" (line 214) — matches the rule card's quoted string exactly.
- Lines 217-236 (cancel plan): identical structure over cancelCase[] checking phaseType/product/productCategory/billingPeriod/priceList all null (lines 225-230); literal message "Missing default rule case for plan cancellation" (line 234).
- Confirmed "catalog fails to load" claim by tracing the caller: org.killbill.xmlloader.XMLLoader.initializeAndValidate() (killbill-xmlloader 0.27.2 sources, extracted to scratchpad) calls c.validate(...) and, if ValidationErrors is non-empty, throws ValidationException — so a missing default really does abort catalog load, not just log a warning.
- Duplicate detection: line 196 literal format "Duplicate rule for change plan %s" matches cur.toString(), driven by a HashSet<DefaultCaseChangePlanPolicy> (line 192) using equals()/hashCode(). Verified equals() at catalog/.../DefaultCaseChangePlanPolicy.java:72-91 and its superclass DefaultCaseChange.java:215-266: duplicate equality requires ALL match-criteria fields equal AND the resulting `policy` value equal — not match-criteria alone. Same pattern confirmed for DefaultCaseCancelPolicy.java:70-93. The rule card's parenthetical "(identical match criteria)" is a minor imprecision — two rules with identical match criteria but different resulting policies would NOT be flagged as duplicates by this code, only fully-identical (criteria + policy) entries are. The concrete Given/When/Then wording itself ("an identical duplicate rule entry") is technically accurate; only the plain-English gloss overstates what "identical" means. This is a nuance worth fixing in the rule card's wording but does not invalidate the cited scenario, which is otherwise a precise, line-accurate re-derivation (including the exact error strings and the load-failure consequence).

No instruction-shaped or injection-style text found in the cited lines (188-236) — this is plain validation logic and comments only.

P0 justification: this validation guards the integrity of the catalog configuration that determines BillingActionPolicy (proration/end-of-term timing) for plan changes and cancellations. Absent this check, a catalog with an incomplete changePolicy/cancelPolicy rule table could load successfully and silently fall back to hardcoded defaults (DefaultPlanRules.java:134 BillingActionPolicy.END_OF_TERM, :175 same) for any plan-change/cancel combination not explicitly covered, producing incorrect proration/invoice timing without any operator signal. Enforcing a mandatory default plus rejecting duplicate rules is a genuine data-integrity guardrail with a direct line to invoicing behavior (money), so P0 is justified.) — confirm criticality.
- **RULE-109** (Overdue state trigger conditions (all must hold), Medium): P0 panel split on whether this moves money / is regulatory (The Given/When/Then is faithful to overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84. The evaluate() method ANDs five checks — invoice count (line 77), unpaid balance (line 78), earliest-unpaid-invoice age (lines 79-80, computed from `unpaidInvoiceTriggerDate` at lines 71-74), payment-decline response membership (line 81), and control-tag inclusion/exclusion (lines 82-83) — and each check is null-safe, short-circuiting to true when that threshold isn't configured, exactly as described. No instruction-shaped or suspicious text was found in the file (only standard Apache license header comments); no injection suspects to report.

However, this is not a P0 under the stated definition (moves money / enforces regulation / guards data integrity). The `evaluate()` method is a pure boolean gate that decides whether an account transitions into a tenant-configured overdue *state* — it does not calculate, charge, or transfer any monetary amount, and the thresholds (invoice count, balance floor, days-since-due, decline reasons, control tags) are arbitrary per-tenant catalog config rather than a legally mandated formula (e.g., no tax, interest, or late-fee computation happens here). It also isn't guarding data integrity in the sense of a database/ledger invariant — it's a collections/dunning eligibility rule. The *consequences* of entering an overdue state (e.g., service restriction, retry blocking, eventual cancellation) defined elsewhere in the overdue module could have downstream financial/customer-impact significance, but the condition-evaluation logic itself is better classified as P1: an important, auditable business policy (mis-triggering could cause wrongful service suspension or revenue leakage) rather than a rule a financial controller or regulator would specifically require to be behaviorally invariant across the rewrite. Recommend downgrading this specific card to P1 and, if a P0 case is needed for the overdue module, look instead at the code that actually blocks/cancels payment or applies late fees once a state is reached. | Re-derived the logic directly from DefaultOverdueCondition.evaluate() (overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84):

1. numberOfUnpaidInvoicesEqualsOrExceeds: null || state.getNumberOfUnpaidInvoices() >= threshold (line 77) — matches "min unpaid invoice count."
2. totalUnpaidInvoiceBalanceEqualsOrExceeds: null || threshold.compareTo(balance) <= 0 (line 78), which algebraically means balance >= threshold — matches "min total balance," direction correctly re-derived.
3. timeSinceEarliestUnpaidInvoiceEqualsOrExceeds: trigger date = earliestUnpaidDate + duration (line 71-74, only computed if both non-null); condition true iff trigger date is not after evaluation date (line 79-80), i.e., elapsed days >= duration. Edge case verified: if the threshold is configured but there are zero unpaid invoices (dateOfEarliestUnpaidInvoice == null), triggerDate stays null and the clause fails closed (does not trigger) rather than throwing or defaulting true — this fail-safe behavior is consistent with (though not explicitly called out by) the rule card.
4. responseForLastFailedPayment: null || array-membership check via responseIsIn (line 81, 86-94) — matches "decline reason in configured list."
5. controlTagInclusion / controlTagExclusion: null || isTagIn / isTagNotIn (lines 82-83, 96-112) — matches inclusion/exclusion semantics exactly (exclusion is satisfied when the tag is absent).
All five clauses are ANDed (lines 76-83), matching "all must hold." The worked example (2 invoices ≥2, $150≥$100, 35 days ≥30 days => true) checks out arithmetically against this logic. The "null threshold => automatically satisfied" parameter note is directly supported by the `== null ||` short-circuit pattern on every clause.

Minor scope note (not a fidelity break): this file only returns the boolean evaluate() result — the actual state-transition side effect ("account transitions into that overdue state") is performed elsewhere (state-machine driver iterating configured OverdueState conditions), not in the cited 69-84 lines. The rule card's phrasing conflates condition-evaluation with the transition action, but this is a reasonable one-line summary of the surrounding system and not a misrepresentation of what the cited code computes.

No instruction-shaped text or credentials found in the cited range; the only stray comment in the file ("// Probably incorrect - comparing Object[] arrays with Arrays.equals", line 194) is an ordinary developer comment outside the cited lines, not a prompt-injection attempt.

P0 justification: overdue-state triggers gate collections actions (blocking further charges/service, initiating dunning/cancellation) that directly control the account's future money flow and access — this is a financial/business-critical policy gate, not merely cosmetic logic, so P0 is appropriate.) — confirm criticality.
- **RULE-110** (Entitlement state derivation precedence, Medium): P0 panel split on whether this moves money / is regulatory (Faithful: `computeStateForEntitlement()` (DefaultEventsStream.java:494-510) does implement exactly the precedence claimed: if `entitlementEffectiveEndDateTime <= utcNow` -> CANCELLED (line 496-497, checked first and short-circuits everything else via the else branch); otherwise if there's an EXPIRED subscription transition whose effective time has passed -> EXPIRED (499-502); otherwise if the entitlement start date is in the future -> PENDING (503-504); otherwise BLOCKED (if any service's blocking aggregator has isBlockEntitlement()) or ACTIVE (505-508). The worked example (cancelled 8/1, also carrying a BLOCKED tag, evaluated 8/31 -> CANCELLED) is consistent with the code because the CANCELLED check is a hard early-return-style branch that pre-empts the blocking-aggregator check entirely.

Not P0 under the compliance lens, though: this method only derives the `EntitlementState` enum used for entitlement/access-control display and gating (e.g., what the API reports, whether a subscription can be resumed, etc.). It does not itself move money — Kill Bill deliberately separates 'blockEntitlement' from 'blockBilling' (see the `isBlockEntitlement()` vs. a parallel billing-block accessor on the same aggregator), and this file's output isn't consumed by the invoice/payment modules to compute charges. It also doesn't implement a specific regulatory mandate (tax, disclosure, retention, PCI, SOX financial control) — it's an internal subscription-lifecycle state machine. And 'guards data integrity' here is a stretch: it's business-state precedence logic, not a constraint protecting stored data from corruption or ensuring auditable financial records. A regulator/auditor/finance controller would care if this silently changed *only* insofar as it might someday be shown to affect billing eligibility, but as read here it's a P1 (domain-correctness) rule about entitlement status display/access gating, not a P0 money-moving or compliance-enforcing rule. No prompt-injection-shaped text was found in the cited range. | Re-derived directly from entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:494-510 (computeStateForEntitlement()):

1. `if (entitlementEffectiveEndDateTime != null && entitlementEffectiveEndDateTime.compareTo(utcNow) <= 0) entitlementState = CANCELLED;` — evaluated first, unconditional short-circuit.
2. `else if (expiryTransition != null && expiryTransition.getEffectiveTransitionTime().compareTo(utcNow) <= 0) entitlementState = EXPIRED;`
3. `else if (entitlementEffectiveStartDateTime.compareTo(utcNow) > 0) entitlementState = PENDING;`
4. `else entitlementState = (aggregator.isBlockEntitlement() ? BLOCKED : ACTIVE);`

This is a strict nested if/else-if chain, so the precedence CANCELLED > EXPIRED > PENDING > BLOCKED/ACTIVE claimed in the rule card is exactly what the code does — no other order is possible since each branch returns before the next check runs.

Given/When/Then check: cancel effective 2026-08-01, evaluated 2026-08-31 → entitlementEffectiveEndDateTime (2026-08-01) compareTo utcNow (2026-08-31) <= 0 is true, so state = CANCELLED and the method returns without ever reaching the blocking-aggregator check on line 507. So even if `currentStateBlockingAggregator.isBlockEntitlement()` were true (the "active BLOCKED tag" in the example), it is never consulted — CANCELLED wins, matching the Then clause exactly. No rounding is involved (this is a state machine, not a calculation), and there's no off-by-one/boundary issue: both CANCELLED and EXPIRED use `<= 0` (i.e., "effective at or before now"), consistent in both branches.

Minor wording nit (not a fidelity defect): the card's "active BLOCKED tag" is a simplification — there's no discrete "BLOCKED tag" object, it's a derived boolean (`isBlockEntitlement()`) computed from an aggregator over blocking-state records across account/bundle/subscription. The described behavior is still accurate to what that boolean represents. No instruction-shaped/injection text found in the cited lines or their surrounding context.

P0 justification: getState() (DefaultEntitlement.java:235) is the canonical entitlement lifecycle status consumed elsewhere to gate subscription operations — e.g. DefaultEntitlement.java:586,713 throw `EntitlementApiException(SUB_CHANGE_NON_ACTIVE, ...)` when a plan-change is attempted on a non-ACTIVE entitlement. Getting the precedence wrong (e.g. letting a stale BLOCKED/ACTIVE state override a CANCELLED entitlement) would let illegal operations proceed on subscriptions that should be locked out, or vice versa incorrectly deny operations — this is a data-integrity guard on the core lifecycle state machine that other billing/entitlement authorization logic depends on, so treating it as P0 is reasonable even though this snippet itself doesn't move money directly.) — confirm criticality.
- **RULE-114** (Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID, Medium): P0 panel doubts spec fidelity: Re-derived the code at DefaultInvoiceDao.java:1372-1417 directly.

What the code actually does (lines 1381-1389):
```
final InvoiceModelDao invoice = transactional.getById(invoiceId.toString(), context);
if (invoice == null) { throw INVOICE_NOT_FOUND; }
if (invoice.getStatus().equals(newStatus) || invoice.getStatus().equals(InvoiceStatus.VOID)) {
    throw new InvoiceApiException(ErrorCode.INVOICE_INVALID_STATUS, newStatus, invoiceId, invoice.getStatus());
}
transactional.updateStatusAndTargetDate(invoiceId.toString(), newStatus.toString(), invoice.getTargetDate(), context);
```
`InvoiceStatus` (confirmed from killbill-api-0.55.1 sources) is exactly `{DRAFT, COMMITTED, VOID}`.

The cited Given/When/Then ("Given Invoice already VOID / When changeInvoiceStatus to COMMITTED is called / Then INVOICE_INVALID_STATUS thrown") is faithful — it's a direct, correct read of the `equals(InvoiceStatus.VOID)` branch: any transition attempted out of VOID throws INVOICE_INVALID_STATUS with the current status, the attempted status, and the invoice id, in that constructor order.

However, the "Plain English" gloss — "Invoice status lifecycle is one-way: DRAFT→COMMITTED→VOID... status only moves forward" — overstates what this method actually enforces. The guard clause only blocks two conditions: (1) newStatus == current status, and (2) current status == VOID. It does NOT contain any check that blocks a backward transition such as COMMITTED→DRAFT. If this method were ever invoked with newStatus=DRAFT on a COMMITTED invoice, neither condition would be true, and the code would proceed to call `updateStatusAndTargetDate(...)`, successfully reverting the invoice to DRAFT (skipping the COMMITTED-specific invoice-creation event and the VOID-specific adjustment/tracking-deactivation logic at lines 1397-1414, but not throwing). The only reason the system behaves as "one-way" in practice is caller discipline: I checked every call site of `changeInvoiceStatus` in this repo (grep across invoice/src) and found exactly two — `DefaultInvoiceUserApi.commitInvoice` (→COMMITTED, DefaultInvoiceUserApi.java:695) and `DefaultInvoiceUserApi.voidInvoice` (→VOID, DefaultInvoiceUserApi.java:788) — neither of which ever passes DRAFT as the target. So "one-way" is an emergent property of how the API is currently used, not an invariant enforced by the cited method itself. A rule card asserting the DAO enforces forward-only ordering would mislead a modernization effort into either (a) writing a contract test for "COMMITTED→DRAFT is rejected" that fails against actual legacy behavior, or (b) silently adding a stricter FSM check during rewrite that changes behavior not present in the original.

P0 justification: legitimate regardless of the above nuance. This rule guards invoice-lifecycle data integrity (VOID must be terminal so a voided invoice can never be un-voided/reactivated), which directly affects money — VOID invoices are excluded from balance/payment processing (see checkInvoiceNotPaid/checkInvoiceDoesContainUsedGeneratedCredit gating VOID at DefaultInvoiceUserApi.java:780-786) and reopening one could double-count or misstate account balances.

No prompt-injection-shaped text found in the cited lines (1372-1417) or in InvoiceStatus.java.
- **RULE-118** (Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED, Medium): What happens to the mirrored parent adjustment if the parent invoice is still DRAFT at the time of the child adjustment?
- **RULE-121** (Payment retry loop is unbounded at the state-machine level, Medium): Where is the actual maximum retry count and backoff schedule enforced for payment retries?
- **RULE-125** (Subscription entitlement-state derivation from event stream, Medium): P0 panel split on whether this moves money / is regulatory (The cited code is faithfully described: getState() (lines 192-205) derives EntitlementState by inspecting the previous/pending SubscriptionBaseTransition, and the event-stream-to-transition mapping (lines 1007-1050) confirms CREATE/TRANSFER set nextState=ACTIVE, CANCEL sets nextState=CANCELLED, and an EXPIRED event sets nextState=EXPIRED; PENDING is returned when only a future (pending) transition exists and no previous transition has occurred yet. This is an accurate, derived-not-stored state machine.\n\nHowever, under the compliance lens (moves money / enforces a regulatory requirement / guards data integrity), this specific rule does not clearly qualify as P0. getState() itself is a pure read-side projection over the append-only event log — it does not move money (no payment, invoice, or ledger effect here), and it does not enforce a regulatory rule (no tax, disclosure, or licensing requirement is being applied). It also isn't a data-integrity guard in the sense of validating or constraining writes; the actual data-integrity property (the event log being append-only/immutable and the sole source of truth) lives in the event-storage/transition-creation code, not in this getter. getState() being wrong would produce a UI/business-logic bug (e.g., an entitlement showing ACTIVE when it should show CANCELLED) which could have downstream billing consequences, but that's several hops removed — the actual money-moving and compliance-relevant logic (invoice generation, cancellation effective-dating, proration) lives elsewhere and would need to be checked independently. As presented, this card is better classified P1/P2 (core domain logic, important for correctness) rather than P0 (directly moves money or enforces a regulatory constraint). No prompt-injection-shaped text was found in the cited lines. | The Given/When/Then matches the code. getState() (lines 192-205) returns getPreviousTransition().getNextState() if a past-or-present transition exists, otherwise PENDING if only a future transition exists (getPendingTransition() uses TimeLimit.FUTURE_ONLY at line 357), otherwise throws IllegalStateException — exactly as described, with no rounding/ordering subtleties beyond the ASC/DESC iterator semantics, which check out (getPreviousTransition uses Order.DESC_FROM_FUTURE + Visibility.FROM_DISK_ONLY + TimeLimit.PAST_OR_PRESENT_ONLY at lines 451-453; getPendingTransition uses Order.ASC_FROM_PAST + TimeLimit.FUTURE_ONLY at lines 355-357). The event-to-state mapping in rebuildTransitionsInternal (lines 1013-1050) exactly matches: CREATE/TRANSFER set nextState=ACTIVE (1013-1025), CANCEL sets nextState=CANCELLED (1033-1037), EXPIRED event sets nextState=EXPIRED (1046-1050); CHANGE (1027-1031) leaves nextState unset/inherited, which is consistent with the rule's silence on CHANGE. The 'future-dated CREATE => PENDING' example is directly verified by the FUTURE_ONLY/PAST_OR_PRESENT_ONLY split. The follow-on claim about CANCEL flipping state to CANCELLED 'going forward' is also structurally correct (CANCEL transition's nextState=CANCELLED becomes the new previousTransition once its effective date is reached), though the card doesn't spell out that this requires the CANCEL's effective date to no longer be in the future — a minor omission, not a fidelity break, since it doesn't contradict the code.

No prompt-injection attempts were found in the cited ranges (192-205, 1007-1050) or the surrounding code I read (150-260, 340-460, 955-1058) — only ordinary Java logic and one benign inline comment.

On P0: this is the authoritative in-memory logic for a subscription's entitlement lifecycle state, which gates access/entitlement decisions and feeds directly into invoicing eligibility (e.g., getLastActivePlan/getLastActiveBillingPeriod branch on getState() at lines 364-443, and CANCELLED state suppresses further billing-relevant transitions). An error here would cause customers to be billed/entitled incorrectly (wrong plan/phase used, cancelled subscriptions treated as active or vice versa), which is a data-integrity and money-moving concern for a billing system — P0 is justified.) — confirm criticality.
- **RULE-128** (Blocked periods are aggregated into disabled billing durations and injected as billing on/off events, Medium): P0 panel doubts spec fidelity: P0 is justified: this logic decides whether recurring/fixed charges are generated (or entirely suppressed) for a subscription during overdue/blocking periods — it directly controls whether the customer is billed, so it is squarely "moves money" territory.

Core mechanics are faithfully described and verified in code:
- Blocking states are pulled per-account, then split into ACCOUNT / SUBSCRIPTION_BUNDLE / SUBSCRIPTION buckets (BlockingCalculator.java:97-111).
- Per-subscription aggregation includes subscription+bundle+account states filtered to those at/before the subscription's CANCEL/EXPIRED effective date (getAggregateBlockingEventsPerSubscription, :163-176).
- createBlockingDurations groups blocking states by `service` (BlockingStateService per service, :323-328), builds one DisabledDuration per service, then sorts and merges across services whenever ranges are not disjoint — `isDisjoint` returns false (i.e. merges) even when ranges only touch (end == next start), so adjacent as well as overlapping ranges collapse into one (DisabledDuration.java:108-121, BlockingCalculator.java:338-354). This matches "overlapping/adjacent ... merged into one."
- For each duration, a START_BILLING_DISABLED event is inserted at the start and an END_BILLING_DISABLED event at the end only if the block has actually ended and a preceding event exists (createNewEvents, :203-218). The literal Given/When/Then example (block day10–day20, START at day10, END at day20, restoring the prior plan/price) matches createNewDisableEvent/createNewReenableEvent (:247-317) correctly, including that the disable event correctly says fixed/recurring price is set to null.

Two fidelity problems, both material for a "moves money" rule:
1. Mischaracterization of price handling: the "Plain English" line says the START event carries a "zero fixed/recurring price," but the code sets `fixedPrice`/`recurringPrice` to `null`, not `0` (BlockingCalculator.java:257-259), and the adjacent code comment explicitly states this null "makes invoice disregard this event" (:255-256) — i.e. no invoice line item is generated at all, as opposed to a $0 line item. A modernization team relying on "zero price" instead of "no line item" could reimplement this as an emitted $0 charge, which is a behavioral divergence for a billing rule. (Note: the G/W/T portion of the same rule card correctly says "nulling," contradicting its own Plain English summary.)
2. Missing edge case: BlockingStateService.addDisabledDuration (BlockingStateService.java:62-68) drops any blocking interval shorter than 1 full day (`Days.daysBetween(...).getDays() >= 1`, explicitly tied to killbill issue #267) — sub-day blocks never produce a DisabledDuration or billing-disabled events at all. The rule card does not mention this threshold, which is a meaningful business policy (a "grace" period under 24 hours never disables billing) that a modernization spec would need to preserve.

No instruction-shaped or injection-style text was found in the cited lines (comments are ordinary code comments/TODOs/issue references); no injection suspects to report.
- **RULE-132** (Overdue state evaluation order: first matching configured state wins, Medium): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67 — calculateOverdueState() iterates getStates() in array order and returns the first state whose conditionEvaluation.evaluate(billingState, now) is true, falling back to getClearState() if none match. This is a plain first-match-wins loop with no tie-breaking or scoring, so the rule as stated ("first configured state in declaration order wins") accurately describes the executable behavior. No injection-shaped text found in the cited lines or surrounding file.

On the compliance lens, this does not clear the P0 bar. The code here is pure control-flow/selection logic — it does not itself compute a dollar amount, apply a tax, post a ledger entry, or reference any named regulatory requirement (e.g., PCI, AML, tax law, GDPR retention). It selects which pre-configured overdue *state* an account is in; any monetary or service-impacting consequence (blocking payments, cancelling a subscription, applying a late fee) lives in the separate state actions defined in overdue.xml and executed elsewhere, not in this evaluation-order rule. It's also not a hard data-integrity invariant like double-entry balancing or idempotency — it's ordinary business policy: 'states are evaluated in the order an admin declares them in XML.'

That said, this is a legitimate P1/P2 candidate worth flagging on the operational-risk lens: if a tenant orders overdue.xml states from least-severe to most-severe (rather than most-severe first), first-match-wins would silently keep accounts in a less severe state than intended, which could matter for dunning/due-process obligations in regulated verticals (e.g., must-warn-before-suspend rules in telecom/utilities). But that risk is contingent on tenant configuration and downstream state actions, not inherent to this code path, so I would not rate it P0 on the compliance lens as cited. | Re-derived independently from DefaultOverdueStateSet.java:59-67: calculateOverdueState iterates getStates() (backed by DefaultOverdueStatesAccount.accountOverdueStates, an @XmlElement array populated by JAXB in document order with no sorting applied anywhere in the class), returns the first DefaultOverdueState whose conditionEvaluation is non-null AND evaluates true (early return), and falls back to getClearState() if the loop exhausts with no match. This exactly matches the Given/When/Then: first-declared-order match wins (OD1 returned over OD2 when both match), CLEAR is the fallback default. No rounding/numeric edge cases apply (this is a state-selection loop, not an arithmetic calculation); the null-guard on conditionEvaluation and the early-return 'first match wins' semantics are both faithfully captured by the plain-English summary without overclaiming. No injection-shaped text found in the cited lines or surrounding file. P0 justification: which overdue/dunning state an account lands in drives payment retry cadence, service suspension/cancellation, and blocking of new charges in the overdue module -- i.e. it gates money-moving and account-standing decisions, not cosmetic behavior -- so preserving this exact state-precedence order is a legitimate behavior-contract item for modernization verification.) — confirm criticality.
- **RULE-133** (Refund idempotency by transaction cookie id, Medium): P0 panel doubts spec fidelity: P0 is justified: this code path in DefaultInvoiceDao.createRefund (invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:806-917, cited region 842-880) is squarely money-movement and data-integrity logic — it creates/dedupes refund records against payments, enforces via Preconditions.checkState (lines 847-853) that a re-submitted refund for the same transactionExternalKey must match the previously recorded amount and paymentId or the call is rejected (IllegalStateException), and gates whether Credit Balance Adjustment (CBA) recompute and invoice-item adjustments run. A regulator/auditor/finance controller would care if this idempotency guard silently weakened (e.g., duplicate refunds issued, or CBA/adjustment logic run or skipped incorrectly) since it directly affects account balances and refund-of-record integrity.

However, the rule card is not fully faithful to the code. It states that when an existing refund is found, the code 'just update[s] its status/date and skip[s] re-running CBA/adjustment side effects' as if that were unconditional for any duplicate call. That is only true for the sub-case where the incoming status exactly equals the existing refund's status (line 856-858: `if (existingRefund.getStatus() == status) return existingRefund;` — this early return is what skips CBA/adjustment/bus-notify). If the incoming status differs (e.g., a transition from PENDING to SUCCESS on the same transactionExternalKey), the code falls through past the update-and-continue path (lines 860-873) into the shared `if (status == InvoicePaymentStatus.SUCCESS)` block at line 882, which DOES run invoice-item adjustment and CBA recompute (lines 894-910) even though no new refund row was created. The rule card collapses this status-transition nuance, which matters for compliance/verification because CBA recompute moves credit-balance amounts — a wrong assumption here could let a modernization pass skip testing the 'existing refund transitioning to SUCCESS' path, where money-affecting side effects actually do fire.

No instruction-shaped or prompt-injection text was found in the cited lines (comments at 842-843, 855, 890-892, 908-909 are ordinary engineering commentary, not directives to an AI reviewer). | Re-derived from invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:844-916 (createRefund).

What the code actually does when an existing refund is found for the same transactionExternalKey (line 844, InvoicePaymentSqlDao.getPaymentForCookieId):
1. Preconditions.checkState amount match (line 847-849): existingRefund.getAmount() must equal requestedPositiveAmount.negate() — throws IllegalStateException (via Guava Preconditions) if not. Faithful to the card.
2. Preconditions.checkState paymentId match (line 851-853): same mechanism. Faithful to the card.
3. If existingRefund.getStatus() == status (the status passed on this call equals the stored status), return existingRefund immediately at line 857 — before any adjustment/CBA code runs. This is the only path where "no adjustment/CBA recompute" is true, and in this branch no update call happens either (status/date are NOT touched, contradicting the card's claim that this branch involves an update).
4. If the status differs, the code calls transactional.updateAttempt(...) at lines 861-872 (status/date update — this part matches the card), but execution then falls through to lines 882-912: if the *new* status == InvoicePaymentStatus.SUCCESS, the invoice-item adjustment loop (894-906) and the CBALogicWrapper.runCBALogicWithNotificationEvents (909-910) DO run, exactly as they would for a brand-new refund insert. The idempotency guard does not prevent adjustment/CBA side effects on a status transition (e.g., PENDING -> SUCCESS retry) — a very plausible real-world payment-gateway retry scenario given the code's own comment about the payment system's state machine calling this multiple times (lines 842-843).

So the card's Given/When/Then collapses two mutually exclusive branches into one claim ("existing record updated AND CBA/adjustment skipped"), which is not what the code does: it is either (a) status unchanged -> early return, no update, no CBA; or (b) status changed -> update happens, and CBA/adjustment run if the new status is SUCCESS. The card's specific example doesn't state the incoming `status` value, so as written it is misleading/incomplete and would give a false equivalence target during migration — a rewritten system that "updates status but always skips CBA on retry" (as the card implies) would silently fail to apply CBA/adjustments on legitimate pending->success refund confirmations, which is a real money-handling regression risk.

The amount/paymentId precondition and no-duplicate-row parts of the card are accurate and well-supported by the code (lines 845, 847-853, 874-880).

No prompt-injection-shaped text was found in the cited range; the comment at 842-843 is ordinary engineering commentary about the payment system's state machine, not a directive to the analysis tool.

This rule is correctly P0: it deduplicates refund creation, enforces amount/paymentId integrity via hard preconditions, and gates account-balance-affecting CBA/adjustment logic — squarely money-movement and data-integrity territory. It needs to be re-specified accurately (conditioned explicitly on whether the incoming status matches the stored status, and separately on whether the resulting status is SUCCESS) before being used as a verification contract for the rewrite.
- **RULE-136** (Blocking-state notification/bus-event batching by aggregation mode, Medium): Confirm this event-coalescing does not suppress externally-visible entitlement webhooks that downstream integrators depend on for each individual state change.
- **RULE-140** (Account parking/unparking implemented via idempotent system tag, Medium): Citation was corrected by referee (The cited range (lines 28-42) covers only import statements, the class declaration, and the constructor (ParkedAccountsManager.java:35-44) — none of which implement parking/unparking logic. The actual behavior described in the rule — parkAccount() catching TagApiException and swallowing it only when ErrorCode.TAG_ALREADY_EXISTS matches (making the call idempotent), and unparkAccount() removing the tag — lives in the parkAccount/unparkAccount methods at lines 46-61, just past the cited range. The rule text itself is accurate to what the code does, just cited at the wrong lines.) — confirm invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java:46-61 is the authoritative implementation.
- **RULE-144** (Payment failure retry schedule, Medium): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: InvoicePaymentControlPluginApi.java:592-609 computes retryCount = attemptsInState-1 against paymentConfig.getPaymentFailureRetryDays() (default "8,8,8" from PaymentConfig.java:31-39), returns now+retryDays[retryCount] while retryCount < list.size(), else null (no more retries). The rule's worked example (2nd failure -> retryCount=1 -> +8 days; 3rd failure -> retryCount=2 -> +8 days; 4th failure -> retryCount=3 >= size 3 -> permanent stop) matches the code exactly. No injection-shaped text found in the cited lines.\n\nHowever, under the COMPLIANCE lens this is not P0. The retry schedule itself does not move money — it only decides *when* the system will re-attempt a previously-declined payment; the actual money movement happens in the payment plugin call that this schedule merely triggers later. It is not enforcing a statute, PCI rule, or card-network mandate encoded in the logic itself (no reference to a specific regulation anywhere in this method or config), and it does not guard data integrity (no dedupe/idempotency/reconciliation concern here — that lives elsewhere, e.g. in payment state-machine transition guards). It is fully tenant-configurable business policy (org.killbill.payment.retry.days, default 8,8,8) — exactly the kind of operational cadence a finance/ops team tunes per business need, not something an auditor or regulator would flag if it silently changed. A regulator/controller would care about tax/interest math, refund correctness, or duplicate-charge prevention; they would generally not treat a dunning retry cadence as an audit-relevant compliance control unless the business had specifically documented it as satisfying a card-network retry-limit rule (Visa/Mastercard decline-retry compliance), and nothing in this code ties it to such a requirement. This is better rated P1 (revenue-operations rule, real business impact if it silently changes, but not a compliance/audit-tier finding). | Re-derived independently from InvoicePaymentControlPluginApi.java:592-609 and PaymentConfig.java:29-39. Default retryDays = [8,8,8] (confirmed at PaymentConfig.java:32,37). retryCount = max(attemptsInState-1, 0); while retryCount < retryDays.size(), next retry = internalContext.getCreatedDate().plusDays(retryDays.get(retryCount)); once retryCount >= 3 the method returns null (no more retries). The GWT's numeric walkthrough (2nd failure -> retryCount=1 -> +8d, 3rd failure -> retryCount=2 -> +8d, 4th failure -> retryCount=3, not < 3, give up) matches exactly. Two minor, non-material paraphrases: 'now' is technically internalContext.getCreatedDate() rather than wall-clock now(), and 'consecutive' failures is really a count of all PAYMENT_FAILURE transactions in the purchased-transaction list gated by the caller checking the *last* transaction's status — functionally equivalent in this flow but worth noting. No instruction-shaped/injection text found in the cited code or PaymentConfig.java. P0 is justified: this rule governs whether/when a declined payment is retried and when the system permanently gives up, which directly affects revenue collection and must be preserved exactly through any rewrite.) — confirm criticality.
- **RULE-145** (Plugin-failure retry backoff (suspected off-by-one), Medium): Is it intentional that the first two plugin-failure retries both wait exactly the initial 300s before the delay starts doubling, or should the second retry already be 600s?
- **RULE-147** (Default overdue escalation tiers, Medium): P0 panel split on whether this moves money / is regulatory (The Given/When/Then is mechanically accurate against overdue.xml:19-61 (OD1=30d blockChanges only; OD2=40d and OD3=50d add disableEntitlementAndChangesBlocked=true; autoReevaluationInterval=5d for all three), but the "Plain English" framing that this is the "default config" is misleading. The actual system-wide default, set by @Default("NoOverdueConfig.xml") on OverdueProperties.getConfigURI() (overdue/src/main/java/org/killbill/billing/overdue/OverdueProperties.java:27-30), is NoOverdueConfig.xml, a single 'Clear' state that performs no blocking at all (overdue/src/main/resources/NoOverdueConfig.xml:20-24). This overdue.xml under profiles/killbill/src/main/resources is only wired in by the test properties file (org.killbill.overdue.uri=overdue.xml in profiles/killbill/src/test/resources/.../killbill.properties:19) and by TestOverdue.java, which uploads it as a per-tenant config. In production, no overdue enforcement fires unless an operator explicitly points org.killbill.overdue.uri at this file or a tenant uploads an equivalent config via the API — so calling these 30/40/50-day thresholds 'the default' overstates what ships active out of the box.

Even taken at face value as a sample dunning schedule, this does not clear the P0 bar. It doesn't move money (no charge, refund, or amount calculation), it isn't a regulatory/compliance mandate (no statute, PCI, or GAAP requirement is cited or implied — dunning cadence is a commercial collections policy the platform explicitly makes tenant-configurable, per TenantOverdueConfigCacheLoader.java), and it doesn't guard data integrity (it gates account state/entitlement transitions, not data correctness or consistency). A finance controller or auditor might care about collections SLAs operationally, but silently changing this sample/tenant-overridable threshold set doesn't violate a legal or accounting obligation the way, e.g., a tax calculation or a ledger-balancing rule would. This is better classified as a configurable business/collections policy (P1-P2), not a P0 compliance control.

No prompt-injection attempts were found in the cited lines (19-61) — the file is plain XML config with a standard Apache license header. | Re-derived independently from profiles/killbill/src/main/resources/overdue.xml:19-61 plus the evaluation code in overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java (calculateOverdueState, lines 60-67) and DefaultOverdueState.java / OverdueStateApplicator.java (isBlockChanges/isDisableEntitlementAndChangesBlocked, lines 255-273).

Faithful (true): The XML declares three states with timeSinceEarliestUnpaidInvoiceEqualsOrExceeds thresholds of 50 (OD3), 40 (OD2), 30 (OD1) days, each with autoReevaluationInterval = 5 days (lines 24,31,37,44,50,57). calculateOverdueState() iterates getStates() in the XML's declared order (OD3, OD2, OD1) and returns the FIRST state whose condition evaluates true, i.e. the most-severe matching state wins (DefaultOverdueStateSet.java:60-67) — confirming the escalation-by-age semantics claimed. OD1 sets blockChanges=true, disableEntitlementAndChangesBlocked=false (lines 54-55); OD2 and OD3 both set blockChanges=true AND disableEntitlementAndChangesBlocked=true (lines 28-29, 41-42) with distinct externalMessage text ("Reached OD1/OD2/OD3", lines 27,40,53). OverdueStateApplicator confirms disableEntitlementAndChangesBlocked=true is what actually triggers both billing-block and entitlement-block (blockBilling/blockEntitlement both just return isDisableEntitlementAndChangesBlocked(), lines 267-273), and blockChanges() is true whenever either flag is set (line 264) — so OD2/OD3 correctly block changes+entitlement+billing while OD1 only blocks changes. The worked example (40-day-old invoice -> OD2, entitlement+billing blocked) is exactly correct given the ">= " (EqualsOrExceeds) boundary semantics and the most-severe-first matching order (OD3's >=50 fails at 40 days, OD2's >=40 matches). No rounding/ordering/edge-case discrepancy found; the "EqualsOrExceeds" operator correctly makes the boundary value (e.g., exactly 40 days) match OD2, as the rule implies.

P0 justification (false): This file is a shipped *default* overdue configuration — per-tenant overrides exist (see MultiTenantOverdueConfig.java) and commonly replace these defaults, so the specific 30/40/50/5-day parameters are not a fixed, universally-enforced constant the way a regulatory threshold would be. More importantly, the rule itself does not move money (no charge/fee/interest calculation), is not a regulatory/compliance requirement, and does not guard data integrity (it does not prevent corrupt or inconsistent stored data) — it gates *access* (blocking subscription changes, entitlement, and future billing actions) as a collections/dunning policy. That's a legitimate and important business rule, but by the stated P0 bar (moves money / regulatory / data integrity) it's better classified as P1 — a configurable business policy whose exact thresholds are examples, not a money-moving or compliance-mandated calculation.

No injection-shaped text was found in the cited lines or surrounding file (only a benign inline comment "Other actions could include... trigger payment retry" in DefaultOverdueState.java:78-83, which is developer notes, not an instruction to the analyzer).) — confirm criticality.
- **RULE-152** (Add-on creation eligibility against base subscription, Medium): P0 panel doubts spec fidelity: P0 classification is justified: this gate is the sole guard preventing an add-on from being attached to a base subscription that is cancelled/not-yet-active, already includes the add-on for free, or isn't catalog-permitted for that base product — all of which directly affect what gets invoiced (double-billing for an already-included feature, billing an add-on with no valid base, etc.), so it's a legitimate data-integrity/billing-correctness rule, not incidental plumbing.

However, the Given/When/Then is NOT faithful to the code for the PENDING-state branch, and this is a real (not cosmetic) discrepancy:

Code (AddonUtils.java:38-41):
```java
if (baseSubscription.getState() == EntitlementState.CANCELLED ||
    (baseSubscription.getState() == EntitlementState.PENDING &&
     context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0)) {
    throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_BP_NON_ACTIVE, targetAddOnPlan.getName());
}
```
`SD.compareTo(RD) < 0` means SD (base start date) is chronologically BEFORE RD (requested add-on date). So the rejection fires when the base's start date is *before* the requested date — i.e., when the requested date comes *after* the start date.

Rule Card claims the opposite: "PENDING with a start date after the requested add-on date" (i.e., SD > RD) triggers rejection. That is the reverse of what `compareTo(...) < 0` actually encodes.

I traced this further via the caller (SubscriptionApiBase.java:241-244), which already rejects any requestedDate before the base's start date with a *different* error (SUB_INVALID_REQUESTED_DATE) before AddonUtils is even invoked:
```java
if (effectiveDate.isBefore(baseSubscription.getStartDate())) {
    throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, ...);
}
addonUtils.checkAddonCreationRights(baseSubscription, plan, effectiveDate, context);
```
So by the time AddonUtils.checkAddonCreationRights runs, requestedDate >= baseSubscription.getStartDate() is already guaranteed. Combined with the `< 0` check inside AddonUtils, the real, end-to-end behavior is: while a base subscription is still PENDING, an add-on can only be created with an effective date exactly equal to the base's start date — a requested date strictly later than the start date is rejected as SUB_CREATE_AO_BP_NON_ACTIVE, and a requested date strictly earlier is rejected upstream as SUB_INVALID_REQUESTED_DATE. This is confirmed by git history: commit 1b1eb87d41 ("entitlement: Allow ADD_ON creation on pending BP. Fixes #681") added this exact line, and its own regression test (TestDefaultEntitlementApi.testAddEntitlementOnPendingBase, added in the same commit) only exercises requestedDate == startDate for the PENDING case — it never tests requestedDate > startDate, which is the case that this logic actually blocks. The commit message itself ("Adding an ADD_ON is now available starting from date when the BB effectively starts") describes intent to allow dates >= start, which the shipped comparison operator does not deliver for PENDING bases (it only allows the exact match).

Net effect: the Rule Card's stated Given/When/Then direction for the PENDING branch is inverted relative to the code, and it also omits the much narrower real-world effect (PENDING bases effectively require an exact start-date match, not "any date on/after start"). The CANCELLED-state rule and the included/available product-list checks (lines 45-53, ErrorCode.SUB_CREATE_AO_ALREADY_INCLUDED / SUB_CREATE_AO_NOT_AVAILABLE) are faithfully described.

No prompt-injection-shaped text was found in AddonUtils.java:36-77 or the related SubscriptionApiBase.java lines reviewed — no injectionSuspects to report.

Recommendation: correct the Rule Card to state the PENDING rejection condition as "requested add-on date is strictly after the base subscription's start date" (not "before"), and add a companion rule for SubscriptionApiBase.java:241-243 (SUB_INVALID_REQUESTED_DATE) since together they define the true eligibility window (exact-date-only) for add-ons on a still-pending base. An SME should confirm whether "exact match only" for PENDING bases is intended, since it contradicts the commit's own stated intent of allowing "on or after" the base start date.
- **RULE-162** (Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list, Medium): What are the configured default retry-interval lists for UNKNOWN vs PENDING transactions in production (paymentConfig defaults), and is it acceptable that a transaction permanently stays unresolved once the list is exhausted with no alerting?
- **RULE-171** (Catalog rule-case matching (first match, wildcard nulls), Medium): Should catalog rule-case matching pick the most specific match instead of the first matching entry in XML order, to avoid a default/wildcard rule accidentally shadowing a more specific one?
- **RULE-173** (Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling, Medium): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: UsageInvoiceItemGenerator.java:231-234 does exactly what the rule claims — updatePerSubscriptionNextNotificationUsageDate throws `new IllegalStateException("Not implemented Yet)")` (typo and all) when usageBillingMode == BillingMode.IN_ADVANCE, before doing any notification-date bookkeeping.

However this is not a P0 (money-moving / regulatory / data-integrity) rule. Tracing the only call site (line 212) shows the method is invoked with a hardcoded BillingMode.IN_ARREAR literal — never IN_ADVANCE. Further, line 148 (`event.getUsages().stream().anyMatch(input -> input.getBillingMode() == BillingMode.IN_ARREAR)`) shows this whole generator explicitly filters subscriptions down to IN_ARREAR usage before any of this logic runs; IN_ADVANCE-billed usage sections (which the catalog layer does support — see DefaultUsage.java:213-218 validating IN_ADVANCE CAPACITY/CONSUMABLE sections) are never routed into this class at all. So the throw at 231-234 is dead/unreachable defensive code in the current production call graph, not a live guard protecting an active money-moving or notification-scheduling flow.

Judged for compliance: this doesn't calculate a charge, enforce a tax/regulatory rule, or protect ledger/invoice integrity for any currently-exercised path — it's an internal "feature not yet built" marker for a code path nothing calls today. If this exception's text or behavior changed silently in a rewrite, no regulator, auditor, or finance controller would notice any difference in real invoices, payments, or account balances, because IN_ADVANCE usage billing simply isn't wired into invoice generation in the first place. This is better classified as a P2/P3 feature-gap/tech-debt item (flag for the modernization team so IN_ADVANCE usage support isn't silently "completed" without also implementing this bookkeeping), not a P0 behavior contract. No prompt-injection or manipulative text was found in the cited lines or their surrounding context. | The literal claim is accurate but the Given/When/Then materially misrepresents reachability, and the rule shouldn't carry a P0 rating.

What the code actually does (UsageInvoiceItemGenerator.java:231-234):
```java
private void updatePerSubscriptionNextNotificationUsageDate(final UUID subscriptionId, final Map<String, LocalDate> nextBillingCycleDates, final BillingMode usageBillingMode, final Map<UUID, SubscriptionFutureNotificationDates> perSubscriptionFutureNotificationDates) {
    if (usageBillingMode == BillingMode.IN_ADVANCE) {
        throw new IllegalStateException("Not implemented Yet)");
    }
    ...
```
This is a private method with exactly one call site in the whole file (line 212):
```java
updatePerSubscriptionNextNotificationUsageDate(sub.getSubscriptionId(), subscriptionResult.getPerUsageNotificationDates(), BillingMode.IN_ARREAR, perSubscriptionFutureNotificationDates);
```
The `usageBillingMode` argument is a hardcoded `BillingMode.IN_ARREAR` literal, not something derived from the catalog's usage billing mode at runtime. Additionally, upstream at line 148 (`event.getUsages().stream().anyMatch(input -> input.getBillingMode() == BillingMode.IN_ARREAR)`), a subscription is only added to `subsUsageInArrear` — and therefore only reaches this method at all — if it has at least one IN_ARREAR usage event. A search of the whole invoice module confirms `BillingMode.IN_ADVANCE` never appears anywhere in this generator except in the dead branch itself; there is no other code path in `invoice/` that handles IN_ADVANCE usage. So even though the catalog layer (`DefaultUsage.java:213-217`) explicitly allows configuring a Usage section with `billingMode=IN_ADVANCE`, that catalog setting can never cause this specific method to be invoked with `BillingMode.IN_ADVANCE` — the branch is unreachable dead code given the current call graph, not a live guard that fires when a customer configures IN_ADVANCE usage.

The rule card's Given/When/Then ("Given a catalog usage section configured with billingMode=IN_ADVANCE / When updatePerSubscriptionNextNotificationUsageDate is invoked for that usage's billing mode") implies the method's behavior is driven by the catalog's configured billing mode. It isn't — the parameter is a compile-time constant at the sole call site, decoupled from any catalog value. That's a meaningful fidelity gap for a rule meant to anchor verification: a modernization team could waste effort trying to reproduce "configure IN_ADVANCE usage → exception thrown" through the normal invoice-generation entry point and never observe it, because that path is structurally unreachable today.

On P0: this is a defensive "not implemented yet" throw in unreachable code, not something that moves money, enforces a regulatory requirement, or actively guards data integrity in the current control flow. The real underlying business fact worth capturing (that IN_ADVANCE usage billing is effectively unsupported by the invoice generator, per line 148's filter) is more accurately a P2/P3 "known limitation / feature gap" note, not a P0 behavior contract.

No prompt-injection-shaped text was found in the cited lines or surrounding code — the exception message "Not implemented Yet)" is a plain (if oddly punctuated) developer message, not an attempt to steer analysis.) — confirm criticality.
- **RULE-180** (Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first, Medium): Is it intended that reclaiming credit from other invoices as a side effect of a CBA delete can alter balances on invoices unrelated to the one being edited?

## Rejected candidate rules

1 candidate rule(s) were rejected by citation referees during extraction (hallucinated, comment-only, or otherwise unsupported by executable code):

- **test rule** (`a.java:1-2`): wrong-citation: The cited path a.java does not exist anywhere in the killbill repository (verified via filesystem search). There is no source at a.java:1-2 to evaluate, so the rule cannot be judged against real code. No corrected location could be identified since the rule text ("test rule" / "test") is a placeholder with no substantive content pointing to any actual business logic.
