# Kill Bill — Business Rules Inventory (v2)

## Methodology
This file restructures `analysis/killbill/BUSINESS_RULES.md` (produced by the `code-modernization:modernize-extract-rules` workflow — 4 extraction rounds, referee-verified citations, two-judge panel review for P0 rules) into the per-rule schema used in `planning/research/business-rules-inventory.md`: each rule is documented with a Description, Line Numbers, Source Code File Name, Source Code, Applies to, and Confidence score. All source code snippets below were re-fetched directly from the repository files at the cited line ranges (not copied from the source document) to guarantee they match the current codebase exactly. The original document's Category and Priority classifications are preserved inside the "Applies to" field rather than as separate columns. One candidate rule ("test rule" @ `a.java:1-2`) was rejected by the original extraction's citation referees as a non-existent/injected citation and is excluded here. Rules are numbered sequentially in the order they appeared in the source document, which doubles as the category grouping order: Calculation, Validation, Lifecycle, Policy.

## Summary
- Total rules: 184
- By category (from source document): Calculation 33, Validation 75, Lifecycle 33, Policy 43
- Confidence distribution: 150 High, 33 Medium, 1 Low
- 34 rules carry a "Note (extraction review flagged this rule for SME confirmation)" clause in their Description, condensed from the source document's SME-review panel findings (full unabridged critiques remain in `analysis/killbill/BUSINESS_RULES.md` under "Rules requiring SME confirmation").

## Business Rules

- [Kill-BR-0001] - Recurring proration percentage between two dates
  [Description]: When a subscription starts or ends mid billing-cycle, the charge for that partial period is the fraction of the billing period actually used, expressed as a fraction of a full cycle (not a fraction of a day-count-per-month convention unless fixed-days is configured). For example: given A monthly billing period runs from previousBillingCycleDate=2024-01-15 to nextBillingCycleDate=2024-02-15 (31 days), and the customer's partial period runs startDate=2024-01-20 to endDate=2024-02-15 (26 days), with prorationFixedDays=0 (actual days mode), when the leading proration for the first partial period is computed, then the proration factor is 26/31 = 0.838709677 (divided with scale=9, HALF_UP), and the recurring amount charged is rate × quantity × 0.838709677, rounded to the currency's decimal places. Edge cases: daysBetween <= 0 returns BigDecimal.ZERO (invoice/.../InvoiceDateUtils.java:81-83); if prorationFixedDays is non-zero, days-in-period and days-in-partial-period are both recomputed using the fixed-days-per-month convention instead of actual calendar days (see next rule). Parameters: KillBillMoney.MAX_SCALE=9 (util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:28), KillBillMoney.ROUNDING_METHOD=BigDecimal.ROUND_HALF_UP (KillBillMoney.java:27).
  [Line Numbers]: 80 to 89
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java
  [Source Code]:
  ```java
      public static BigDecimal calculateProrationBetweenDates(final LocalDate startDate, final LocalDate endDate, final int daysBetween, final int fixedDaysInMonth) {
          if (daysBetween <= 0) {
              return BigDecimal.ZERO;
          }

          final BigDecimal daysInPeriod = new BigDecimal(daysBetween);
          final BigDecimal days = fixedDaysInMonth == 0 ? new BigDecimal(Days.daysBetween(startDate, endDate).getDays()) : new BigDecimal(daysBetweenWithFixedDaysInMonth(startDate, endDate, fixedDaysInMonth));

          return days.divide(daysInPeriod, KillBillMoney.MAX_SCALE, KillBillMoney.ROUNDING_METHOD);
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (InvoiceDateUtils)
  [Confidence score]: High

- [Kill-BR-0002] - Fixed-days-in-month proration override
  [Description]: A tenant can configure a fixed number of days to treat every month as having (e.g. 30), instead of using each month's real day count, to avoid proration amounts varying by month length. When start and end date fall in the same month, real day counts are still used. For example: given prorationFixedDays=30, startDate=2024-01-20, endDate=2024-02-15 (different months), and January has 31 days, when days-between is calculated for proration, then result = actualDaysBetween(Jan 20, Feb 15) − (31 − 30) = actualDays − 1, i.e. the real day count is adjusted by (last day of start month − fixedDaysInMonth). Edge cases: Same calendar month for start/end: no adjustment applied, raw daysBetween used directly (InvoiceDateUtils.java:94-96). Parameters: org.killbill.invoice.proration.fixed.days, default 0 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:235-243) — 0 means 'use actual calendar days, no adjustment'.
  [Line Numbers]: 91 to 99
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java
  [Source Code]:
  ```java
      @VisibleForTesting
      public static int daysBetweenWithFixedDaysInMonth(final LocalDate startDate, final LocalDate endDate, final int fixedDaysInMonth) {
          final int daysBetween = Days.daysBetween(startDate, endDate).getDays();
          if(startDate.getMonthOfYear() == endDate.getMonthOfYear()) { //same month, no need for extra logic
              return daysBetween;
          }
          final int lastDayOfMonth = startDate.dayOfMonth().withMaximumValue().getDayOfMonth();
          return daysBetween - (lastDayOfMonth - fixedDaysInMonth);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (InvoiceDateUtils)
  [Confidence score]: High

- [Kill-BR-0003] - Recurring invoice item amount formula
  [Description]: The dollar amount billed for a recurring subscription line item is the catalog rate times the subscribed quantity, times either 1 (full period), a leading/trailing proration fraction, or a whole-number count of periods, rounded to the currency's minor unit. For example: given rate=$49.99, quantity=2, numberOfCycles=1 (a full period), when a recurring invoice item is generated for that billing period, then amount = round_half_up($49.99 × 2 × 1, to currency decimal places) = $99.98. Edge cases: rate is null (e.g. $0 plan) → no recurring item created at all (FixedAndRecurringInvoiceItemGenerator.java:261). Parameters: rounding via KillBillMoney.of() — HALF_UP to currency's native decimal places (util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java:32-35).
  [Line Numbers]: 260 to 278
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  [Source Code]:
  ```java
                      final BigDecimal rate = thisEvent.getRecurringPrice();
                      if (rate != null) {
                          final BigDecimal quantity = new BigDecimal(thisEvent.getQuantity());
                          final BigDecimal rateXQuantity = rate.multiply(quantity);

                          final BigDecimal amount = KillBillMoney.of(itemDatum.getNumberOfCycles().multiply(rateXQuantity), currency);
                          final DateTime catalogEffectiveDate = thisEvent.getCatalogEffectiveDate() != null ? thisEvent.getCatalogEffectiveDate() : null;
                          final RecurringInvoiceItem recurringItem = new RecurringInvoiceItem(invoiceId,
                                                                                              accountId,
                                                                                              thisEvent.getBundleId(),
                                                                                              thisEvent.getSubscriptionId(),
                                                                                              currentPlan.getProduct().getName(),
                                                                                              currentPlan.getName(),
                                                                                              thisEvent.getPlanPhase().getName(),
                                                                                              catalogEffectiveDate,
                                                                                              itemDatum.getStartDate(),
                                                                                              itemDatum.getEndDate(),
                                                                                              amount, rateXQuantity, currency);
                          items.add(recurringItem);
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (FixedAndRecurringInvoiceItemGenerator)
  [Confidence score]: High

- [Kill-BR-0004] - Fixed price invoice item amount
  [Description]: A one-time (fixed) charge, such as a setup fee, is billed as the catalog fixed price times quantity, with no proration and no currency rounding applied at this step. For example: given fixedPrice=$25.00 setup fee, quantity=3, when the fixed-price billing event is processed, then fixedPriceXQuantity = $75.00 is recorded as the FixedPriceInvoiceItem amount. Edge cases: fixedPrice null → no fixed item generated (line 436-437, 468-470); roundedStartDate after targetDate → skipped entirely (line 431-432). Parameters: none (quantity is a whole-number multiplier from the subscription).
  [Line Numbers]: 434 to 462
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  [Source Code]:
  ```java
              final BigDecimal fixedPrice = thisEvent.getFixedPrice();

              if (fixedPrice != null) {
                  final BigDecimal quantity = new BigDecimal(thisEvent.getQuantity());
                  final BigDecimal fixedPriceXQuantity = fixedPrice.multiply(quantity);

                  final Plan currentPlan = thisEvent.getPlan();
                  Preconditions.checkNotNull(currentPlan, String.format("Unexpected null Plan name event = %s", thisEvent));

                  final DateTime catalogEffectiveDate = thisEvent.getCatalogEffectiveDate() != null ? thisEvent.getCatalogEffectiveDate() : null;
                  LocalDate invoiceItemEndDate = null;
                  if (thisEvent.getRecurringPrice() == null && thisEvent.getPlanPhase().getPhaseType() != PhaseType.EVERGREEN) {
                      try {
                          if (nextEvent != null && nextEvent.getTransitionType() == SubscriptionBaseTransitionType.PHASE) {
                              invoiceItemEndDate = internalCallContext.toLocalDate(nextEvent.getEffectiveDate());
                          }
                          else {
                              invoiceItemEndDate = thisEvent.getPlanPhase().getDuration().addToLocalDate(roundedStartDate);
                          }

                      } catch (final CatalogApiException e) {
                          log.warn("Error while computing invoice item end date, ", e.getMessage());
                      }
                  }
                  final FixedPriceInvoiceItem fixedPriceInvoiceItem = new FixedPriceInvoiceItem(invoiceId, accountId, thisEvent.getBundleId(),
                                                                                                thisEvent.getSubscriptionId(),
                                                                                                currentPlan.getProduct().getName(), currentPlan.getName(), thisEvent.getPlanPhase().getName(),
                                                                                                catalogEffectiveDate,
                                                                                                roundedStartDate, invoiceItemEndDate, fixedPriceXQuantity, currency);
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (FixedAndRecurringInvoiceItemGenerator)
  [Confidence score]: High

- [Kill-BR-0005] - Bill-cycle-day alignment caps at month length
  [Description]: A subscription's billing cycle day (e.g. 'bill on the 31st') is only meaningful for monthly/quarterly/annual plans. When the configured billing day doesn't exist in a given month (e.g. day 31 in February), the invoice date falls back to the last actual day of that month. For example: given billingCycleDay=31, proposedDate in February (28 days) for a MONTHLY plan, when the next bill cycle date is aligned, then the resulting billing date is February 28 (or 29 in a leap year), not March 3 or an error. Edge cases: Non-month-based billing periods (e.g. daily/weekly) skip alignment entirely and use the raw proposed date (line 76-79). Parameters: none (structural rule based on calendar).
  [Line Numbers]: 74 to 88
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  [Source Code]:
  ```java
      public static LocalDate alignProposedBillCycleDate(final LocalDate proposedDate, final int billingCycleDay, final BillingPeriod billingPeriod) {
          // billingCycleDay alignment only makes sense for month based BillingPeriod (MONTHLY, QUARTERLY, BIANNUAL, ANNUAL)
          final boolean isMonthBased = (billingPeriod.getPeriod().getMonths() | billingPeriod.getPeriod().getYears()) > 0;
          if (!isMonthBased) {
              return proposedDate;
          }
          final int lastDayOfMonth = proposedDate.dayOfMonth().getMaximumValue();
          int proposedBillCycleDate = proposedDate.getDayOfMonth();
          if (billingCycleDay <= lastDayOfMonth) {
              proposedBillCycleDate = billingCycleDay;
          } else {
              proposedBillCycleDate = lastDayOfMonth;
          }
          return new LocalDate(proposedDate.getYear(), proposedDate.getMonthOfYear(), proposedBillCycleDate, proposedDate.getChronology());
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — util module (BillCycleDayCalculator)
  [Confidence score]: High

- [Kill-BR-0006] - Invoice balance formula
  [Description]: An invoice's outstanding balance equals everything charged (line items, invoice-level and item-level adjustments, parent-summary items) plus account-credit amounts applied, minus everything paid and refunded/charged-back so far. For example: given an invoice with $100.00 of charge items, a −$10.00 CBA (account credit) adjustment, one successful $50.00 payment, and a $5.00 refund on that payment, when the invoice balance is recalculated, then balance = ($100.00 + (−$10.00) + $0.00 invoice-credit) − ($50.00 payment + $5.00 refund) = $90.00 − $55.00 = $35.00, rounded to the currency's decimal places (HALF_UP). Edge cases: Only InvoicePayments with status SUCCESS are counted toward amountPaid/amountRefunded (InvoiceCalculatorUtils.java:213-214,231-232); Only InvoicePaymentType.ATTEMPT counts as 'paid'; REFUND and CHARGED_BACK count as 'refunded' (line 217-219,235-237); A 2-item 'credit invoice' (CREDIT_ADJ + matching CBA_ADJ that nets to zero) is a special case included via computeInvoiceAmountAdjustedForAccountCredit (line 134-155). Parameters: KillBillMoney rounding as above. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 classification is justified: this is the canonical invoice-balance formula (InvoiceCalculatorUtils.computeRawInvoiceBalance, lines 96-110) that determines the amount owed on every invoice, feeding dunning/overdue logic and payment reconciliation across the whole system — it moves money.
  [Line Numbers]: 96 to 110
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java
  [Source Code]:
  ```java
      public static BigDecimal computeRawInvoiceBalance(final Currency currency,
                                                        @Nullable final Iterable<InvoiceItem> invoiceItems,
                                                        @Nullable final Iterable<InvoicePayment> invoicePayments) {

          final BigDecimal amountPaid = computeInvoiceAmountPaid(currency, invoicePayments)
                  .add(computeInvoiceAmountRefunded(currency, invoicePayments));

          final BigDecimal chargedAmount = computeInvoiceAmountCharged(currency, invoiceItems)
                  .add(computeInvoiceAmountCredited(currency, invoiceItems))
                  .add(computeInvoiceAmountAdjustedForAccountCredit(currency, invoiceItems));

          final BigDecimal invoiceBalance = chargedAmount.add(amountPaid.negate());

          return KillBillMoney.of(invoiceBalance, currency);
      }
  ```
  [Line Numbers]: 157 to 242
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java
  [Source Code]:
  ```java
      public static BigDecimal computeInvoiceAmountCharged(final Currency currency, @Nullable final Iterable<InvoiceItem> invoiceItems) {
          BigDecimal amountCharged = BigDecimal.ZERO;
          if (invoiceItems == null || !invoiceItems.iterator().hasNext()) {
              return KillBillMoney.of(amountCharged, currency);
          }

          for (final InvoiceItem invoiceItem : invoiceItems) {

              if (isCharge(invoiceItem) ||
                  isInvoiceAdjustmentItem(invoiceItem, invoiceItems) ||
                  isInvoiceItemAdjustmentItem(invoiceItem) ||
                  isParentSummaryItem(invoiceItem)) {
                  amountCharged = amountCharged.add(invoiceItem.getAmount());
              }
          }

          return KillBillMoney.of(amountCharged, currency);
      }

      public static BigDecimal computeInvoiceOriginalAmountCharged(final DateTime invoiceCreatedDate, final Currency currency, @Nullable final Iterable<InvoiceItem> invoiceItems) {
          BigDecimal amountCharged = BigDecimal.ZERO;
          if (invoiceItems == null || !invoiceItems.iterator().hasNext()) {
              return KillBillMoney.of(amountCharged, currency);
          }

          for (final InvoiceItem invoiceItem : invoiceItems) {
              if (isCharge(invoiceItem) &&
                  (invoiceItem.getCreatedDate() != null && invoiceItem.getCreatedDate().equals(invoiceCreatedDate))) {
                  amountCharged = amountCharged.add(invoiceItem.getAmount());
              }
          }

          return KillBillMoney.of(amountCharged, currency);
      }

      public static BigDecimal computeInvoiceAmountCredited(final Currency currency, @Nullable final Iterable<InvoiceItem> invoiceItems) {
          BigDecimal amountCredited = BigDecimal.ZERO;
          if (invoiceItems == null || !invoiceItems.iterator().hasNext()) {
              return KillBillMoney.of(amountCredited, currency);
          }

          for (final InvoiceItem invoiceItem : invoiceItems) {
              if (isAccountCreditItem(invoiceItem)) {
                  amountCredited = amountCredited.add(invoiceItem.getAmount());
              }
          }

          return KillBillMoney.of(amountCredited, currency);
      }

      public static BigDecimal computeInvoiceAmountPaid(final Currency currency, @Nullable final Iterable<InvoicePayment> invoicePayments) {
          BigDecimal amountPaid = BigDecimal.ZERO;
          if (invoicePayments == null || !invoicePayments.iterator().hasNext()) {
              return KillBillMoney.of(amountPaid, currency);
          }

          for (final InvoicePayment invoicePayment : invoicePayments) {
              if (invoicePayment.getStatus() != InvoicePaymentStatus.SUCCESS) {
                  continue;
              }
              if (InvoicePaymentType.ATTEMPT.equals(invoicePayment.getType())) {
                  amountPaid = amountPaid.add(invoicePayment.getAmount());
              }
          }

          return KillBillMoney.of(amountPaid, currency);
      }

      public static BigDecimal computeInvoiceAmountRefunded(final Currency currency, @Nullable final Iterable<InvoicePayment> invoicePayments) {
          BigDecimal amountRefunded = BigDecimal.ZERO;
          if (invoicePayments == null || !invoicePayments.iterator().hasNext()) {
              return KillBillMoney.of(amountRefunded, currency);
          }

          for (final InvoicePayment invoicePayment : invoicePayments) {
              if (invoicePayment.getStatus() != InvoicePaymentStatus.SUCCESS) {
                  continue;
              }
              if (InvoicePaymentType.REFUND.equals(invoicePayment.getType()) ||
                  InvoicePaymentType.CHARGED_BACK.equals(invoicePayment.getType())) {
                  amountRefunded = amountRefunded.add(invoicePayment.getAmount());
              }
          }

          return KillBillMoney.of(amountRefunded, currency);
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (InvoiceCalculatorUtils)
  [Confidence score]: Medium

- [Kill-BR-0007] - Unpaid invoice balance aggregation for overdue evaluation
  [Description]: The account's overdue-eligible balance is the simple sum of the current balance of every unpaid invoice as of today, and the 'earliest unpaid invoice' is whichever unpaid invoice has the oldest invoice date (ties broken arbitrarily but deterministically by object hash). For example: given unpaid invoices with balances $20.00, $35.50, and $0.00 (fully credited but not closed), when calculateBillingState() runs, then unpaidInvoiceBalance = $55.50 (numberOfUnpaidInvoices = 3, including the $0.00 one since it's still in the 'unpaid' set returned by the invoice API). Edge cases: responseForLastFailedPayment is hardcoded to PaymentResponse.INSUFFICIENT_FUNDS with a 'TODO MDW' comment — this field is NOT actually derived from real payment history, so any overdue condition keyed on 'responseForLastFailedPaymentIn' effectively behaves as if every account's last failure was always INSUFFICIENT_FUNDS; line 79: `final PaymentResponse responseForLastFailedPayment = PaymentResponse.INSUFFICIENT_FUNDS; //TODO MDW`; Suspected defect: responseForLastFailedPayment is a hardcoded constant, not computed from actual payment data, making any overdue rule that filters on payment-decline reason silently incorrect/no-op in practice.
  [Line Numbers]: 67 to 101
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/calculator/BillingStateCalculator.java
  [Source Code]:
  ```java
      public BillingState calculateBillingState(final ImmutableAccountData account, final InternalCallContext context) throws OverdueException {
          final SortedSet<Invoice> unpaidInvoices = unpaidInvoicesForAccount(account.getId(), context);

          final int numberOfUnpaidInvoices = unpaidInvoices.size();
          final BigDecimal unpaidInvoiceBalance = sumBalance(unpaidInvoices);
          LocalDate dateOfEarliestUnpaidInvoice = null;
          UUID idOfEarliestUnpaidInvoice = null;
          final Invoice invoice = earliest(unpaidInvoices);
          if (invoice != null) {
              dateOfEarliestUnpaidInvoice = invoice.getInvoiceDate();
              idOfEarliestUnpaidInvoice = invoice.getId();
          }
          final PaymentResponse responseForLastFailedPayment = PaymentResponse.INSUFFICIENT_FUNDS; //TODO MDW
          final List<Tag> accountTags = tagApi.getTags(account.getId(), ObjectType.ACCOUNT, context);
          final Tag[] tags = accountTags.toArray(new Tag[accountTags.size()]);

          return new BillingState(account.getId(), numberOfUnpaidInvoices, unpaidInvoiceBalance, dateOfEarliestUnpaidInvoice, idOfEarliestUnpaidInvoice, responseForLastFailedPayment, tags);
      }

      // Package scope for testing
      Invoice earliest(final SortedSet<Invoice> unpaidInvoices) {
          try {
              return unpaidInvoices.first();
          } catch (final NoSuchElementException e) {
              return null;
          }
      }

      BigDecimal sumBalance(final SortedSet<Invoice> unpaidInvoices) {
          BigDecimal sum = BigDecimal.ZERO;
          for (final Invoice unpaidInvoice : unpaidInvoices) {
              sum = sum.add(unpaidInvoice.getBalance());
          }
          return sum;
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — overdue module (BillingStateCalculator)
  [Confidence score]: High

- [Kill-BR-0008] - Tiered usage billing — ALL_TIERS policy
  [Description]: When a metered-usage plan is configured with the ALL_TIERS block policy, consumption is billed by filling each price tier in order — units are charged at tier 1's rate up to that tier's block cap, then remaining units spill into tier 2 at tier 2's rate, and so on — much like graduated income tax brackets. For example: given Tier 1: block size 100 units up to max 5 blocks (cap 500 units); Tier 2: block size 100 units, unlimited (max=-1); usage = 730 units, when computeToBeBilledConsumableInArrearWith_ALL_TIERS() runs, then Tier 1 bills for its full cap of 5 blocks (500 units) at tier-1's per-block price; remaining 230 units go to Tier 2, which needs ceil(230/100)=3 blocks at tier-2's per-block price (partial blocks are always rounded UP to a whole block). Edge cases: A tier's block count only rounds up (ceiling) when there is a nonzero remainder from divideAndRemainder (line 236-237); A tier max of -1 means unlimited capacity for that tier (line 239, 281); When there is previously-billed usage for the same period (reconciliation), that quantity is subtracted from the newly computed tier usage, with strict consistency checks (unless dryRun or usage-missing-lenient) that fully-consumed prior tiers must exactly match, and the current tier must be >= prior usage (line 247-260); Tier 1 always generates a line item even at zero usage (to support $0 usage items); tiers 2+ only generate a line item if consumed > 0 (line 261). Parameters: TierBlockPolicy.ALL_TIERS (catalog XML), block size, block max (per catalog, -1 = unlimited).
  [Line Numbers]: 219 to 266
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java
  [Source Code]:
  ```java
      List<UsageConsumableInArrearTierUnitAggregate> computeToBeBilledConsumableInArrearWith_ALL_TIERS(final LocalDate startDate,
                                                                                                       final LocalDate endDate,
                                                                                                       final List<TieredBlock> tieredBlocks,
                                                                                                       final List<UsageConsumableInArrearTierUnitAggregate> previousUsage,
                                                                                                       final BigDecimal units,
                                                                                                       final boolean isDryRun) throws CatalogApiException {
          final List<UsageConsumableInArrearTierUnitAggregate> toBeBilledDetails = new LinkedList<>();
          BigDecimal remainingUnits = units;
          int tierNum = 0;

          final int lastPreviousUsageTier = previousUsage.size(); // we count tier from 1, 2, ...
          final boolean hasPreviousUsage = lastPreviousUsageTier > 0;

          for (final TieredBlock tieredBlock : tieredBlocks) {
              tierNum++;
              final BigDecimal blockTierSize = tieredBlock.getSize();
              final BigDecimal blockTierMax = tieredBlock.getMax();
              final BigDecimal[] divRemaining =  remainingUnits.divideAndRemainder(blockTierSize);
              final BigDecimal tmp =  (divRemaining[1].compareTo(BigDecimal.ZERO) == 0) ? divRemaining[0] : divRemaining[0].add(BigDecimal.ONE);
              BigDecimal nbUsedTierBlocks;
              if (blockTierMax.compareTo(new BigDecimal("-1")) != 0 && tmp.compareTo(blockTierMax) > 0) {
                  nbUsedTierBlocks = tieredBlock.getMax();
                  remainingUnits = remainingUnits.subtract(blockTierMax.multiply(blockTierSize));
              } else {
                  nbUsedTierBlocks = tmp;
                  remainingUnits = BigDecimal.ZERO;
              }
              // We generate an entry if we consumed anything on this tier or if this is the first tier to also support $0 Usage item
              if (hasPreviousUsage) {
                  final BigDecimal previousUsageQuantity = tierNum <= lastPreviousUsageTier ? previousUsage.get(tierNum - 1).getQuantity() : BigDecimal.ZERO;
                  // Be lenient for dryRun use cases as we could have plugin optimizations not returning full usage data
                  if (!isDryRun && !invoiceConfig.isUsageMissingLenient(internalTenantContext)) {
                      if (tierNum < lastPreviousUsageTier) {
                          Preconditions.checkState(nbUsedTierBlocks.compareTo(previousUsageQuantity) == 0, String.format("Expected usage for subscription='%s', targetDate='%s', startDt='%s', endDt='%s', tier='%s', unit='%s' to be full, instead found units='[%s/%s]'",
                                                                                                            getSubscriptionId(), targetDate, startDate, endDate, tierNum, tieredBlock.getUnit().getName(), nbUsedTierBlocks, previousUsageQuantity));
                      } else {
                          Preconditions.checkState(nbUsedTierBlocks.subtract(previousUsageQuantity).compareTo(BigDecimal.ZERO) >= 0, String.format("Expected usage for subscription='%s', targetDate='%s', startDt='%s', endDt='%s', tier='%s', unit='%s' to contain at least as much as current usage, instead found units='[%s/%s]'",
                                                                                                                getSubscriptionId(), targetDate, startDate, endDate, tierNum, tieredBlock.getUnit().getName(), nbUsedTierBlocks, previousUsageQuantity));
                      }
                  }
                  nbUsedTierBlocks = nbUsedTierBlocks.subtract(previousUsageQuantity);
              }
              if (tierNum == 1 || nbUsedTierBlocks.compareTo(BigDecimal.ZERO) > 0) {
                  toBeBilledDetails.add(new UsageConsumableInArrearTierUnitAggregate(tierNum, tieredBlock.getUnit().getName(), tieredBlock.getPrice().getPrice(getCurrency()), blockTierSize, nbUsedTierBlocks));
              }
          }
          return toBeBilledDetails;
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (ContiguousIntervalConsumableUsageInArrear)
  [Confidence score]: High

- [Kill-BR-0009] - Tiered usage billing — TOP_TIER policy
  [Description]: When a metered-usage plan is configured with the TOP_TIER block policy, all usage for the period is billed at the single rate of the highest tier the total usage reaches into — unlike ALL_TIERS, none of the cheaper lower-tier rates apply once a higher tier is reached. For example: given Tier 1 caps at 500 units, Tier 2 is unlimited; usage = 730 units, tier-2 block size = 100 units, when computeToBeBilledConsumableInArrearWith_TOP_TIER() runs, then usage lands in Tier 2 (since 730 > tier-1's 500 cap), and the ENTIRE 730 units is billed at Tier 2's rate: nbBlocks = ceil(730/100) = 8 blocks × tier-2 per-block price. Edge cases: If no tier's max is exceeded, defaults to the LAST tier in the list by construction before the loop even runs (line 271-272) — meaning a misconfigured catalog with all tiers 'exceeded' silently falls back to billing at the last tier's rate rather than erroring. Parameters: TierBlockPolicy.TOP_TIER (catalog XML).
  [Line Numbers]: 268 to 299
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java
  [Source Code]:
  ```java
      UsageConsumableInArrearTierUnitAggregate computeToBeBilledConsumableInArrearWith_TOP_TIER(final List<TieredBlock> tieredBlocks, final BigDecimal units) throws CatalogApiException {
          BigDecimal remainingUnits = units;
          // By default last tierBlock
          TieredBlock targetBlock = tieredBlocks.get(tieredBlocks.size() - 1);
          int targetTierNum = tieredBlocks.size();
          int tierNum = 0;
          // Loop through all tier block
          for (final TieredBlock tieredBlock : tieredBlocks) {
              tierNum++;
              final BigDecimal blockTierMax = tieredBlock.getMax();
              final BigDecimal blockTierSize = tieredBlock.getSize();
              final BigDecimal[] divRemaining =  remainingUnits.divideAndRemainder(blockTierSize);
              final BigDecimal tmp =  (divRemaining[1].compareTo(BigDecimal.ZERO) == 0) ? divRemaining[0] : divRemaining[0].add(BigDecimal.ONE);
              if (tmp.compareTo(blockTierMax) > 0) { /* Includes the case where max is unlimited (-1) */
                  remainingUnits = remainingUnits.subtract(blockTierMax.multiply(blockTierSize));
              } else {
                  targetBlock = tieredBlock;
                  targetTierNum = tierNum;
                  break;
              }
          }

          final BigDecimal lastBlockTierSize = targetBlock.getSize();
          final BigDecimal[] divRemaining =  units.divideAndRemainder(lastBlockTierSize);
          final BigDecimal nbBlocks =  (divRemaining[1].compareTo(BigDecimal.ZERO) == 0) ? divRemaining[0] : divRemaining[0].add(BigDecimal.ONE);
          return new UsageConsumableInArrearTierUnitAggregate(
                  targetTierNum,
                  targetBlock.getUnit().getName(),
                  targetBlock.getPrice().getPrice(getCurrency()),
                  lastBlockTierSize,
                  nbBlocks);
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (ContiguousIntervalConsumableUsageInArrear)
  [Confidence score]: High

- [Kill-BR-0010] - Capacity-based usage billing — flat tier price
  [Description]: For capacity/subscription-style usage plans (e.g. 'up to N API calls, M storage GB included'), the system picks the cheapest tier whose per-unit-type maximum limits are not exceeded by any measured unit, and charges that tier's single flat recurring price for the whole period — not a per-unit rate. For example: given Tier 1 (price $10) allows max 1,000 API calls AND max 10GB storage; account used 1,200 API calls and 5GB storage, when computeToBeBilledCapacityInArrear() evaluates tiers in order, then Tier 1 does not comply (1,200 > 1,000 API-call limit) even though storage is within limits, so evaluation moves to Tier 2 and bills its flat price instead of Tier 1's $10. Edge cases: Only the max limit is checked, not the min — comment explicitly states 'We ignore the min and only look at the max Limit as the tiers should be contiguous' (line 132); If ALL units for a tier are <= 0, the tier still 'complies' but bills $0 instead of the tier price, to support $0 usage items (line 128,137,146); If no tier complies for all unit types, an IllegalStateException ('Could not find tier...') is thrown — treated as a catalog misconfiguration (line 149-152). Parameters: Tier.getMax() per unit type, -1 = unlimited (catalog XML); tiers must be evaluated in ascending/contiguous order per catalog design.
  [Line Numbers]: 114 to 153
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalCapacityUsageInArrear.java
  [Source Code]:
  ```java
      @VisibleForTesting
      UsageCapacityInArrearAggregate computeToBeBilledCapacityInArrear(final List<RolledUpUnit> roUnits) throws CatalogApiException {
          Preconditions.checkState(isBuilt.get(), "#computeToBeBilledCapacityInArrear() isBuilt.get() return false");

          final List<Tier> tiers = getCapacityInArrearTier(usage);

          final Set<String> perUnitTypeDetailTierLevel = new HashSet<>();
          int tierNum = 0;
          final List<UsageInArrearTierUnitDetail> toBeBilledDetails = new LinkedList<>();
          for (final Tier cur : tiers) {
              tierNum++;
              final BigDecimal curTierPrice = cur.getRecurringPrice().getPrice(getCurrency());

              boolean complies = true;
              boolean allUnitAmountToZero = true;  // Support for $0 Usage item
              for (final RolledUpUnit ro : roUnits) {

                  final Limit tierLimit = getTierLimit(cur, ro.getUnitType());
                  // We ignore the min and only look at the max Limit as the tiers should be contiguous.
                  // Specifying a -1 value for last max tier will make the validation works
                  if (tierLimit.getMax().compareTo(new BigDecimal("-1")) != 0 && ro.getAmount().compareTo(tierLimit.getMax())> 0) {
                      complies = false;
                  } else {
                      allUnitAmountToZero = ro.getAmount().compareTo(BigDecimal.ZERO) <= 0 && allUnitAmountToZero;

                      if (!perUnitTypeDetailTierLevel.contains(ro.getUnitType())) {
                          toBeBilledDetails.add(new UsageInArrearTierUnitDetail(tierNum, ro.getUnitType(), curTierPrice, ro.getAmount()));
                          perUnitTypeDetailTierLevel.add(ro.getUnitType());
                      }
                  }
              }
              if (complies) {
                  return new UsageCapacityInArrearAggregate(toBeBilledDetails, allUnitAmountToZero ? BigDecimal.ZERO : curTierPrice);
              }
          }
          // Probably invalid catalog config
          joiner.join(roUnits);
          Preconditions.checkState(false, "Could not find tier for usage " + usage.getName() + "matching with data = " + joiner.join(roUnits));
          return null;
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (ContiguousIntervalCapacityUsageInArrear)
  [Confidence score]: High

- [Kill-BR-0011] - Zero-price plan when no prices defined
  [Description]: If a plan phase's international price list has no price entries at all, the phase is free in every currency. For example: given A plan phase's DefaultInternationalPrice has an empty prices[] array, when getPrice(currency) is called for any currency, then BigDecimal.ZERO is returned instead of throwing a missing-price error.
  [Line Numbers]: 45 to 46
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  [Source Code]:
  ```java
      // No prices is a zero cost plan in all currencies
      @XmlElement(name = "price", required = false)
  ```
  [Line Numbers]: 88 to 92
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  [Source Code]:
  ```java
      @Override
      public BigDecimal getPrice(final Currency currency) throws CatalogApiException {
          if (prices.length == 0) {
              return BigDecimal.ZERO;
          }
  ```
  [Applies to]: Calculation rule (Priority P1) — catalog module (DefaultInternationalPrice)
  [Confidence score]: High

- [Kill-BR-0012] - Negative invoice balance auto-generates account credit
  [Description]: If, after all items are applied, an invoice's balance is negative (the customer was overcharged/over-adjusted), the system automatically issues a Credit Balance Adjustment (CBA) item for the negative amount. For example: given An invoice with raw balance of -$25.00, when computeCBAComplexity runs during invoice finalization, then A CreditBalanceAdjInvoiceItem for -(-$25.00) = -$25.00 stored amount (i.e. amount.negate() of the balance) is created, effectively adding $25.00 of usable account credit. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: Compliance lens: P0 is justified.
  [Line Numbers]: 63 to 76
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      public InvoiceItemModelDao computeCBAComplexity(final InvoiceModelDao invoice,
                                                      @Nullable final BigDecimal accountCBAOrNull,
                                                      @Nullable final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                      final InternalCallContext context) throws EntityPersistenceException, InvoiceApiException {
          final BigDecimal balance = getInvoiceBalance(invoice);

          final boolean hasPendingPayments = invoice.getInvoicePayments().stream()
                                                    .filter(p -> p.getType() == InvoicePaymentType.ATTEMPT && p.getStatus() == InvoicePaymentStatus.PENDING)
                                                    .findFirst()
                                                    .isPresent();

          if (balance.compareTo(BigDecimal.ZERO) < 0) {
              // Current balance is negative, we need to generate a credit (positive CBA amount)
              return buildCBAItem(invoice, balance, context);
  ```
  [Line Numbers]: 243 to 251
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      private InvoiceItemModelDao buildCBAItem(final InvoiceModelDao invoice,
                                               final BigDecimal amount,
                                               final InternalCallContext context) {
          return new InvoiceItemModelDao(new CreditBalanceAdjInvoiceItem(invoice.getId(),
                                                                         invoice.getAccountId(),
                                                                         context.getCreatedDate().toLocalDate(),
                                                                         amount.negate(),
                                                                         invoice.getCurrency()));
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (CBADao)
  [Confidence score]: Medium

- [Kill-BR-0013] - Available account credit is auto-applied to a committed positive-balance invoice
  [Description]: If a committed invoice still owes money and the account has spare credit (and no payment is already in flight and the invoice hasn't been written off), the system automatically consumes credit to pay down the invoice, capped at the invoice's own balance. For example: given An invoice balance of $40.00, COMMITTED status, no PENDING payment attempts, not written off, and account CBA (credit) of $100.00, when computeCBAComplexity runs, then A CBA item of -$40.00 is applied (min(accountCBA, balance)); if account CBA were only $15.00, the applied CBA item would be -$15.00 instead. Edge cases: Invoice with a pending ATTEMPT payment is skipped entirely (no CBA applied) even if balance is positive; Written-off invoices are skipped; DRAFT invoices are skipped (must be COMMITTED).
  [Line Numbers]: 77 to 97
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
          } else if (balance.compareTo(BigDecimal.ZERO) > 0
                     && invoice.getStatus() == InvoiceStatus.COMMITTED
                     && !hasPendingPayments
                     && !invoice.isWrittenOff()) {
              // Current balance is positive and the invoice is COMMITTED, we need to use some of the existing if available (negative CBA amount)
              // PERF: in some codepaths, the CBA maybe have already been computed
              BigDecimal accountCBA = accountCBAOrNull;
              if (accountCBAOrNull == null) {
                  accountCBA = getAccountCBAFromTransaction(entitySqlDaoWrapperFactory, context);
              }

              if (accountCBA.compareTo(BigDecimal.ZERO) <= 0) {
                  return null;
              }
              final BigDecimal positiveCreditAmount = accountCBA.compareTo(balance) > 0 ? balance : accountCBA;
              return buildCBAItem(invoice, positiveCreditAmount, context);
          } else {
              // 0 balance, nothing to do.
              return null;
          }
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (CBADao)
  [Confidence score]: High

- [Kill-BR-0014] - Child-account invoice balance nets against the parent invoice's charge for that child
  [Description]: For parent/child account billing, once the parent invoice itself has a zero raw balance, a child invoice's effective balance is computed as the child's own amount charged minus whatever the parent invoice already charged on the child's behalf, rather than using the child invoice's own raw balance. For example: given A child invoice with amount charged $75.00, whose parent invoice (raw balance $0) contains a line item of $75.00 attributed to that child account, when getInvoiceBalance(childInvoice) is called, then Balance = $75.00 (childAmountCharged) + (-$75.00 parent amount) = $0.00, instead of using the child invoice's own raw balance calculation. Note (extraction review flagged this rule for SME confirmation): Confirm this parent/child netting is intentional double-accounting protection and not a workaround for a known parent-invoice sync bug.
  [Line Numbers]: 100 to 124
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      boolean isParentExistAndRawBalanceIsZero(final InvoiceModelDao invoice) {
          return invoice.getParentInvoice() != null &&
                 InvoiceModelDaoHelper.getRawBalanceForRegularInvoice(invoice.getParentInvoice()).compareTo(BigDecimal.ZERO) == 0;
      }

      @VisibleForTesting
      BigDecimal getChildInvoiceAmountCharged(final InvoiceModelDao invoice) {
          return InvoiceModelDaoHelper.getAmountCharged(invoice);
      }

      @VisibleForTesting
      BigDecimal getInvoiceBalance(final InvoiceModelDao invoice) {
          if (isParentExistAndRawBalanceIsZero(invoice)) {
              final BigDecimal parentInvoiceAmountChargedForChild = invoice.getParentInvoice().getInvoiceItems().stream()
                      .filter(input -> input.getChildAccountId().equals(invoice.getAccountId()))
                      .map(InvoiceItemModelDao::getAmount)
                      .reduce(BigDecimal.ZERO, BigDecimal::add);

              final BigDecimal childInvoiceAmountCharged = getChildInvoiceAmountCharged(invoice);

              return childInvoiceAmountCharged.add(parentInvoiceAmountChargedForChild.negate());
          }

          return InvoiceModelDaoHelper.getRawBalanceForRegularInvoice(invoice);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (CBADao)
  [Confidence score]: Medium

- [Kill-BR-0015] - Bill-cycle-day source depends on alignment type
  [Description]: The day-of-month used to bill a subscription comes from a different place depending on alignment policy: the account's BCD for ACCOUNT alignment, the base (bundle) subscription's first-charge day for BUNDLE alignment, or the subscription's own first-charge day for SUBSCRIPTION alignment. For example: given BillingAlignment.SUBSCRIPTION and a subscription whose date of first non-zero recurring charge is 2026-04-17, when calculateBcdForAlignment is invoked, then BCD = 17 (day-of-month of the first non-zero recurring charge); for ACCOUNT alignment it instead asserts the account BCD is already set (throws if 0) and reuses it verbatim.
  [Line Numbers]: 57 to 72
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  [Source Code]:
  ```java
      public static int calculateBcdForAlignment(@Nullable final Map<UUID, Integer> bcdCache, final SubscriptionBase subscription, final SubscriptionBase baseSubscription, final BillingAlignment alignment, final InternalTenantContext internalTenantContext, final int accountBillCycleDayLocal) {
          int result = 0;
          switch (alignment) {
              case ACCOUNT:
                  Preconditions.checkState(accountBillCycleDayLocal != 0, "Account BCD should be set at this point");
                  result = accountBillCycleDayLocal;
                  break;
              case BUNDLE:
                  result = calculateOrRetrieveBcdFromSubscription(bcdCache, baseSubscription, internalTenantContext);
                  break;
              case SUBSCRIPTION:
                  result = calculateOrRetrieveBcdFromSubscription(bcdCache, subscription, internalTenantContext);
                  break;
          }
          return result;
      }
  ```
  [Line Numbers]: 133 to 139
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  [Source Code]:
  ```java
      private static int calculateBcdFromSubscription(final SubscriptionBase subscription, final InternalTenantContext internalTenantContext) {
          final DateTime date = subscription.getDateOfFirstRecurringNonZeroCharge();
          final int bcdLocal = internalTenantContext.toLocalDate(date).getDayOfMonth();
          log.debug("Calculated BCD: subscriptionId='{}', subscriptionStartDate='{}', bcd='{}'",
                    subscription.getId(), date.toDateTimeISO(), bcdLocal);
          return bcdLocal;
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — util module (BillCycleDayCalculator)
  [Confidence score]: High

- [Kill-BR-0016] - Month-based billing periods roll the aligned date to the next month once the BCD has passed
  [Description]: For monthly/quarterly/biannual/annual billing periods, if the current transition falls after the account's bill-cycle-day within its month, the next aligned bill date is pushed into the following month (clamped to that month's length); non-month-based periods (e.g. weekly) instead just repeatedly add the billing period length until reaching/passing the target date. For example: given curTransitionDate = 2026-03-20, billingCycleDay = 15, billingPeriod = MONTHLY, when alignToNextBillCycleDate computes the next aligned date, then Because dayOfMonth(20) > BCD(15), the result is April 15, 2026 (curTransitionDate.plusMonths(1) re-aligned to day 15), not March 15. Edge cases: billingPeriod == NO_BILLING_PERIOD returns curTransitionDate unchanged with no alignment.
  [Line Numbers]: 97 to 120
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  [Source Code]:
  ```java
      private static LocalDate alignToNextBillCycleDate(final LocalDate prevTransitionDate, final LocalDate curTransitionDate, /* date to be aligned */ final int billingCycleDay, final BillingPeriod billingPeriod) {
          // Trivial case, nothing to align with
          if (billingPeriod == BillingPeriod.NO_BILLING_PERIOD) {
              return curTransitionDate;
          }
          // billingCycleDay alignment only makes sense for month based BillingPeriod (MONTHLY, QUARTERLY, BIANNUAL, ANNUAL)
          final boolean isMonthBased = (billingPeriod.getPeriod().getMonths() | billingPeriod.getPeriod().getYears()) > 0;
          if (!isMonthBased) {
              LocalDate newDate = prevTransitionDate.plus(billingPeriod.getPeriod());
              while (newDate.isBefore(curTransitionDate)) {
                  newDate = newDate.plus(billingPeriod.getPeriod());
              }
              return newDate;
          }
          if (curTransitionDate.getDayOfMonth() > billingCycleDay) {
              return alignProposedBillCycleDate(curTransitionDate.plusMonths(1), billingCycleDay, billingPeriod);
          } else {
              return alignProposedBillCycleDate(curTransitionDate, billingCycleDay, billingPeriod);
          }
      }

      public static LocalDate alignToNextBillCycleDate(final DateTime prevTransitionDate, final DateTime curTransitionDate,/* date to be aligned */ final int billingCycleDay, final BillingPeriod billingPeriod, final InternalTenantContext internalTenantContext) {
          return alignToNextBillCycleDate(internalTenantContext.toLocalDate(prevTransitionDate), internalTenantContext.toLocalDate(curTransitionDate), billingCycleDay, billingPeriod);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — util module (BillCycleDayCalculator)
  [Confidence score]: High

- [Kill-BR-0017] - Overlapping/adjacent disabled billing durations across account, bundle, and subscription level blocks are merged
  [Description]: A subscription can be simultaneously blocked at the account, bundle, and subscription level with different overlapping time windows per service; before generating disable/re-enable billing events, all these windows are combined into the minimal set of non-overlapping disabled windows by merging any two windows that touch or overlap, keeping the earliest start and the latest (or open-ended) end. For example: given Two disabled windows: [2026-01-01, 2026-01-10) from an account-level block and [2026-01-05, 2026-01-20) from a subscription-level block, when createBlockingDurations merges the sorted list of per-service disabled durations, then The two windows merge into a single [2026-01-01, 2026-01-20) disabled duration because they are not disjoint (end of the first, 2026-01-10, is not before the start of the second, 2026-01-05); windows are only kept separate when one's end date is strictly before the other's start date.
  [Line Numbers]: 320 to 355
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      protected List<DisabledDuration> createBlockingDurations(final Iterable<BlockingState> inputBundleEvents) {
          final List<DisabledDuration> result = new LinkedList<DisabledDuration>();

          final Map<String, BlockingStateService> svcBlockedMap = new HashMap<String, BlockingStateService>();
          for (final BlockingState bs : inputBundleEvents) {
              final String service = bs.getService();
              svcBlockedMap.putIfAbsent(service, new BlockingStateService());
              svcBlockedMap.get(service).addBlockingState(bs);
          }

          final Collection<DisabledDuration> unorderedDisabledDuration = new LinkedList<DisabledDuration>();
          for (final Entry<String, BlockingStateService> entry : svcBlockedMap.entrySet()) {
              unorderedDisabledDuration.addAll(entry.getValue().build());
          }
          final List<DisabledDuration> sortedDisabledDuration = unorderedDisabledDuration.stream()
                  .sorted()
                  .collect(Collectors.toUnmodifiableList());

          DisabledDuration prevDuration = null;
          for (final DisabledDuration d : sortedDisabledDuration) {
              // isDisjoint
              if (prevDuration == null) {
                  prevDuration = d;
              } else {
                  if (prevDuration.isDisjoint(d)) {
                      result.add(prevDuration);
                      prevDuration = d;
                  } else {
                      prevDuration = DisabledDuration.mergeDuration(prevDuration, d);
                  }
              }
          }
          if (prevDuration != null) {
              result.add(prevDuration);
          }
  ```
  [Line Numbers]: 90 to 121
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DisabledDuration.java
  [Source Code]:
  ```java
      // Case 1: this contained into o => false
      // |---------|       this
      // |--------------|  o
      //
      // Case 2: this overlaps with o => false
      // |---------|            this
      //      |--------------|  o
      //
      // Case 3: o contains into this => false
      // |---------| this
      //      |---|  o
      //
      // Case 4: this and o are adjacent => false
      // |---------| this
      //           |---|  o
      // Case 5: this and o are disjoint => true
      // |---------| this
      //             |---|  o
      public boolean isDisjoint(final DisabledDuration o) {
          return end!= null && end.compareTo(o.getStart()) < 0;
      }

      public static DisabledDuration mergeDuration(final DisabledDuration d1, final DisabledDuration d2) {
          Preconditions.checkState(d1.getStart().compareTo(d2.getStart()) <= 0);
          Preconditions.checkState(!d1.isDisjoint(d2));

          final DateTime endDate = (d1.getEnd() != null && d2.getEnd() != null) ?
                                   d1.getEnd().compareTo(d2.getEnd()) < 0 ? d2.getEnd() : d1.getEnd() :
                                   null;

          return new DisabledDuration(d1.getStart(), endDate);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — junction module (BlockingCalculator, DisabledDuration)
  [Confidence score]: High

- [Kill-BR-0018] - Blocked-duration billing suppression with merge of overlapping durations
  [Description]: When a subscription is blocked (overdue/entitlement block) for a period, synthetic 'disable' and 're-enable' billing events are inserted that zero out billing for that period; if multiple blocking states from different services produce overlapping or adjacent disabled periods, those periods are merged into a single continuous disabled duration before events are generated. For example: given Two blocking-state services each produce a disabled duration for the same subscription that overlap in time, when createBlockingDurations() aggregates them, then The two durations are merged into one via DisabledDuration.mergeDuration rather than producing separate disable/re-enable event pairs; the resulting disable event sets fixedPrice=null, recurringPrice=null, billingPeriod=NO_BILLING_PERIOD so invoicing disregards billing for the blocked window.
  [Line Numbers]: 247 to 259
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      protected BillingEvent createNewDisableEvent(final DateTime disabledDurationStart,
                                                   final BillingEvent previousEvent) {
          final int billCycleDay = previousEvent.getBillCycleDayLocal();
          final int quantity = previousEvent.getQuantity();
          final DateTime effectiveDate = disabledDurationStart;
          final PlanPhase planPhase = previousEvent.getPlanPhase();
          final Plan plan = previousEvent.getPlan();

          // Make sure to set the fixed price to null and the billing period to NO_BILLING_PERIOD,
          // which makes invoice disregard this event
          final BigDecimal fixedPrice = null;
          final BigDecimal recurringPrice = null;
          final BillingPeriod billingPeriod = BillingPeriod.NO_BILLING_PERIOD;
  ```
  [Line Numbers]: 320 to 354
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      protected List<DisabledDuration> createBlockingDurations(final Iterable<BlockingState> inputBundleEvents) {
          final List<DisabledDuration> result = new LinkedList<DisabledDuration>();

          final Map<String, BlockingStateService> svcBlockedMap = new HashMap<String, BlockingStateService>();
          for (final BlockingState bs : inputBundleEvents) {
              final String service = bs.getService();
              svcBlockedMap.putIfAbsent(service, new BlockingStateService());
              svcBlockedMap.get(service).addBlockingState(bs);
          }

          final Collection<DisabledDuration> unorderedDisabledDuration = new LinkedList<DisabledDuration>();
          for (final Entry<String, BlockingStateService> entry : svcBlockedMap.entrySet()) {
              unorderedDisabledDuration.addAll(entry.getValue().build());
          }
          final List<DisabledDuration> sortedDisabledDuration = unorderedDisabledDuration.stream()
                  .sorted()
                  .collect(Collectors.toUnmodifiableList());

          DisabledDuration prevDuration = null;
          for (final DisabledDuration d : sortedDisabledDuration) {
              // isDisjoint
              if (prevDuration == null) {
                  prevDuration = d;
              } else {
                  if (prevDuration.isDisjoint(d)) {
                      result.add(prevDuration);
                      prevDuration = d;
                  } else {
                      prevDuration = DisabledDuration.mergeDuration(prevDuration, d);
                  }
              }
          }
          if (prevDuration != null) {
              result.add(prevDuration);
          }
  ```
  [Applies to]: Calculation rule (Priority P0) — junction module (BlockingCalculator)
  [Confidence score]: High

- [Kill-BR-0019] - Automatic account credit (CBA) generation and application
  [Description]: If an invoice ends up with a negative balance (overpaid), the system automatically generates an account credit (CBA) for the negative amount. Conversely, if an invoice is COMMITTED, has a positive balance, no pending payment attempts, and isn't written off, any existing account credit is automatically applied to it (up to the smaller of the credit or the balance). Leftover account credit is then distributed across all other COMMITTED unpaid invoices, oldest first, until exhausted. For example: given An account has $50 of existing credit and two unpaid COMMITTED invoices dated Jan 1 ($30 balance) and Feb 1 ($40 balance), when CBA complexity is computed via doCBAComplexityFromTransaction, then The Jan 1 invoice gets a $30 credit applied first (fully paid via credit), then $20 of the remaining credit is applied to the Feb 1 invoice, leaving it with a $20 balance and $0 account credit remaining. Parameters: None (pure balance/credit arithmetic); invoices ordered by invoiceDate ascending.
  [Line Numbers]: 63 to 97
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      public InvoiceItemModelDao computeCBAComplexity(final InvoiceModelDao invoice,
                                                      @Nullable final BigDecimal accountCBAOrNull,
                                                      @Nullable final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                      final InternalCallContext context) throws EntityPersistenceException, InvoiceApiException {
          final BigDecimal balance = getInvoiceBalance(invoice);

          final boolean hasPendingPayments = invoice.getInvoicePayments().stream()
                                                    .filter(p -> p.getType() == InvoicePaymentType.ATTEMPT && p.getStatus() == InvoicePaymentStatus.PENDING)
                                                    .findFirst()
                                                    .isPresent();

          if (balance.compareTo(BigDecimal.ZERO) < 0) {
              // Current balance is negative, we need to generate a credit (positive CBA amount)
              return buildCBAItem(invoice, balance, context);
          } else if (balance.compareTo(BigDecimal.ZERO) > 0
                     && invoice.getStatus() == InvoiceStatus.COMMITTED
                     && !hasPendingPayments
                     && !invoice.isWrittenOff()) {
              // Current balance is positive and the invoice is COMMITTED, we need to use some of the existing if available (negative CBA amount)
              // PERF: in some codepaths, the CBA maybe have already been computed
              BigDecimal accountCBA = accountCBAOrNull;
              if (accountCBAOrNull == null) {
                  accountCBA = getAccountCBAFromTransaction(entitySqlDaoWrapperFactory, context);
              }

              if (accountCBA.compareTo(BigDecimal.ZERO) <= 0) {
                  return null;
              }
              final BigDecimal positiveCreditAmount = accountCBA.compareTo(balance) > 0 ? balance : accountCBA;
              return buildCBAItem(invoice, positiveCreditAmount, context);
          } else {
              // 0 balance, nothing to do.
              return null;
          }
      }
  ```
  [Line Numbers]: 185 to 218
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      // Distribute account CBA across all COMMITTED unpaid invoices
      private List<InvoiceItemModelDao> useExistingCBAFromTransaction(final BigDecimal accountCBA,
                                                                      final List<CustomField> invoiceCustomFields,
                                                                      final List<Tag> invoicesTags,
                                                                      final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                                      final InternalCallContext context) throws InvoiceApiException, EntityPersistenceException {
          if (accountCBA.compareTo(BigDecimal.ZERO) <= 0) {
              return Collections.emptyList();
          }

          final List<InvoiceItemModelDao> result = new ArrayList<>();

          // PERF: Computing the invoice balance is difficult to do in the DB, so we effectively need to retrieve all invoices on the account and filter the unpaid ones in memory.
          // This should be infrequent though because of the account CBA check above.
          final List<InvoiceModelDao> allInvoices = invoiceDaoHelper.getAllInvoicesByAccountFromTransaction(false, true, invoiceCustomFields, invoicesTags, entitySqlDaoWrapperFactory, context);
          final List<InvoiceModelDao> unpaidInvoices = invoiceDaoHelper.getUnpaidInvoicesByAccountFromTransaction(allInvoices, null, null);
          // We order the same os BillingStateCalculator-- should really share the comparator
          final List<InvoiceModelDao> orderedUnpaidInvoices = unpaidInvoices.stream()
                  .sorted(Comparator.comparing(InvoiceModelDao::getInvoiceDate))
                  .collect(Collectors.toUnmodifiableList());

          BigDecimal remainingAccountCBA = accountCBA;
          for (final InvoiceModelDao unpaidInvoice : orderedUnpaidInvoices) {
              final InvoiceItemModelDao cbaItem = computeCBAComplexityAndCreateCBAItem(remainingAccountCBA, unpaidInvoice, entitySqlDaoWrapperFactory, context);
              if (cbaItem != null) {
                  result.add(cbaItem);
                  remainingAccountCBA = remainingAccountCBA.add(cbaItem.getAmount());
              }
              if (remainingAccountCBA.compareTo(BigDecimal.ZERO) <= 0) {
                  break;
              }
          }
          return result;
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (CBADao)
  [Confidence score]: High

- [Kill-BR-0020] - Account balance calculation excludes non-committed and written-off/child-consolidated invoices
  [Description]: When computing an account's total balance, DRAFT and VOID invoices are skipped entirely; an invoice's contribution to the balance is treated as zero if the invoice itself is WRITTEN_OFF, or if it is a child invoice whose parent is WRITTEN_OFF, DRAFT, VOID, or already has zero raw balance (fully paid parent). The account's CBA (credit balance) is subtracted from the summed invoice balances. For example: given An account with one committed unpaid invoice of $100 and one WRITTEN_OFF committed invoice of $50, when getAccountBalance is called, then The returned balance is $100 (the written-off invoice contributes $0), minus any account CBA.
  [Line Numbers]: 729 to 760
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
      @Override
      public BigDecimal getAccountBalance(final UUID accountId, final InternalTenantContext context) {
          final List<CustomField> invoiceCustomFields = getInvoiceCustomFields(context);
          final List<Tag> invoicesTags = getInvoicesTags(context);

          return transactionalSqlDao.execute(true, entitySqlDaoWrapperFactory -> {
              BigDecimal cba = BigDecimal.ZERO;

              BigDecimal accountBalance = BigDecimal.ZERO;
              final List<InvoiceModelDao> invoices = invoiceDaoHelper.getAllInvoicesByAccountFromTransaction(false, true, invoiceCustomFields, invoicesTags, entitySqlDaoWrapperFactory, context);
              for (final InvoiceModelDao cur : invoices) {

                  // Skip DRAFT OR VOID invoices
                  if (cur.getStatus().equals(InvoiceStatus.DRAFT) || cur.getStatus().equals(InvoiceStatus.VOID)) {
                      continue;
                  }

                  final boolean hasZeroParentBalance =
                          cur.getParentInvoice() != null &&
                          (cur.getParentInvoice().isWrittenOff() ||
                           cur.getParentInvoice().getStatus() == InvoiceStatus.DRAFT ||
                           cur.getParentInvoice().getStatus() == InvoiceStatus.VOID ||
                           InvoiceModelDaoHelper.getRawBalanceForRegularInvoice(cur.getParentInvoice()).compareTo(BigDecimal.ZERO) == 0);

                      // invoices that are WRITTEN_OFF or paid children invoices are excluded from balance computation but the cba summation needs to be included
                      final BigDecimal invoiceBalance = cur.isWrittenOff() || hasZeroParentBalance ? BigDecimal.ZERO : InvoiceModelDaoHelper.getRawBalanceForRegularInvoice(cur);
                      accountBalance = accountBalance.add(invoiceBalance);
                      cba = cba.add(InvoiceModelDaoHelper.getCBAAmount(cur));
                  }
                  return accountBalance.subtract(cba);
          });
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0021] - Currency rounding for stored monetary amounts (half-up)
  [Description]: Every monetary amount stored on invoice items, payments, and payment transactions is rounded to the currency's native number of decimal places using half-up rounding before being persisted. For example: given An unrounded computed amount of 19.265 USD (USD has 2 decimal places), when KillBillMoney.of(amount, currency) is called before constructing an invoice item, payment, or payment transaction, then The amount is rounded to 19.27 USD (BigDecimal.ROUND_HALF_UP, scale = currency's decimal places). Edge cases: Applied consistently across invoice items (InvoiceItemBase), payment amounts (DefaultPayment), and payment transaction amounts/processedAmount (DefaultPaymentTransaction); Also applied to usage in-arrear unrounded totals (see separate rule). Parameters: ROUNDING_METHOD=HALF_UP; MAX_SCALE=9 (unused ceiling); actual scale = CurrencyUnit.getDecimalPlaces() per currency.
  [Line Numbers]: 27 to 35
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/currency/KillBillMoney.java
  [Source Code]:
  ```java
      public static final int ROUNDING_METHOD = BigDecimal.ROUND_HALF_UP;
      public static final int MAX_SCALE = 9;

      private KillBillMoney() {}

      public static BigDecimal of(final BigDecimal amount, final Currency currency) {
          final CurrencyUnit currencyUnit = CurrencyUnit.of(currency.toString());
          return amount.setScale(currencyUnit.getDecimalPlaces(), ROUNDING_METHOD);
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — util module (KillBillMoney)
  [Confidence score]: High

- [Kill-BR-0022] - Display-amount rounding for overdue/billing-state formatting
  [Description]: When formatting an account's unpaid balance for overdue notices/templates, the amount is always rounded to exactly 2 decimal places, half-up, regardless of the account's currency. For example: given An unpaid balance of 45.006 (or null), when DefaultBillingStateFormatter renders the balance for an overdue notification, then The displayed value is "45.01" (or "0.00" if null). Edge cases: Null balance defaults to 0.00 rather than throwing; Fixed 2-decimal scale regardless of currency (e.g. currencies with 0 or 3 decimal places would still show 2). Parameters: SCALE=2; rounding=HALF_UP.
  [Line Numbers]: 23 to 35
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/DefaultAmountFormatter.java
  [Source Code]:
  ```java
      public static final int SCALE = 2;

      // Static only
      private DefaultAmountFormatter() {
      }

      public static BigDecimal round(final BigDecimal decimal) {
          if (decimal == null) {
              return BigDecimal.ZERO.setScale(SCALE, BigDecimal.ROUND_HALF_UP);
          } else {
              return decimal.setScale(SCALE, BigDecimal.ROUND_HALF_UP);
          }
      }
  ```
  [Applies to]: Calculation rule (Priority P2) — util module (DefaultAmountFormatter)
  [Confidence score]: High

- [Kill-BR-0023] - Usage in-arrear reconciliation bills only the newly-owed delta
  [Description]: When re-invoicing a usage period that was partially billed before, the new invoice item only charges the difference between the freshly computed total owed and what was already billed for that period (unless ALL_TIERS tier detail already accounts for it). For example: given A usage period previously billed for $50.00, and the newly recomputed total usage charge for the same period is $70.00, when The usage invoice generator reconciles the period, then A new USAGE invoice item for $20.00 is created (amountToBill = toBeBilledUsage - billedUsage). Edge cases: If amountToBill is negative (usage appears to have decreased), the item is silently dropped when in dry-run or when invoiceConfig.isUsageMissingLenient() is true; otherwise an InvoiceApiException(UNEXPECTED_ERROR) is thrown to prevent under-billing corruption; If the period was not previously billed and amountToBill is exactly 0, no $0 item is created; if previously billed, a $0 item is allowed through. Parameters: TierBlockPolicy.ALL_TIERS with itemized prior details skips the subtraction (already reconciled at tier-detail level).
  [Line Numbers]: 88 to 99
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalConsumableUsageInArrear.java
  [Source Code]:
  ```java
          // In the case past invoice items showed the details (areAllBilledItemsWithDetails=true), billed usage has already been taken into account
          // as it part of the reconciliation logic, so no need to subtract it here
          final BigDecimal amountToBill = (usage.getTierBlockPolicy() == TierBlockPolicy.ALL_TIERS && areAllBilledItemsWithDetails) ? toBeBilledUsage : toBeBilledUsage.subtract(billedUsage);

          if (amountToBill.compareTo(BigDecimal.ZERO) < 0) {
              if (isDryRun || invoiceConfig.isUsageMissingLenient(internalTenantContext)) {
                  return;
              } else {
                  throw new InvoiceApiException(ErrorCode.UNEXPECTED_ERROR,
                                                String.format("ILLEGAL INVOICING STATE: Usage period start='%s', end='%s', amountToBill='%s', (previously billed amount='%s', new proposed amount='%s')",
                                                              startDate, endDate, amountToBill, billedUsage, toBeBilledUsage));
              }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (ContiguousIntervalConsumableUsageInArrear)
  [Confidence score]: High

- [Kill-BR-0024] - Chargeback amount/currency resolution fallback
  [Description]: When recording a chargeback, the amount used is the plugin-processed amount if its currency matches the original invoice payment; otherwise the transaction's requested amount if that currency matches; otherwise the chargeback simply reuses the full original invoice payment's amount and currency. For example: given An original invoice payment of $100 USD; the payment gateway's processedCurrency comes back as EUR and processedAmount is set, when A CHARGEBACK transaction succeeds and neither processedCurrency nor the transaction currency equals USD, then The chargeback is recorded at the linked invoice payment's original amount and currency ($100 USD), ignoring the mismatched processed/transaction currency values. Edge cases: Order of precedence: processedAmount+processedCurrency first, then amount+currency, then fall back to original linked payment. Note (extraction review flagged this rule for SME confirmation): Is silently falling back to the original payment amount/currency (rather than rejecting or flagging) the intended behavior when the gateway reports a chargeback in a different currency?.
  [Line Numbers]: 217 to 230
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
                          final BigDecimal amount;
                          final Currency currency;
                          if (linkedInvoicePayment.getCurrency().equals(paymentControlContext.getProcessedCurrency()) && paymentControlContext.getProcessedAmount() != null) {
                              amount = paymentControlContext.getProcessedAmount();
                              currency = paymentControlContext.getProcessedCurrency();
                          } else if (linkedInvoicePayment.getCurrency().equals(paymentControlContext.getCurrency()) && paymentControlContext.getAmount() != null) {
                              amount = paymentControlContext.getAmount();
                              currency = paymentControlContext.getCurrency();
                          } else {
                              amount = linkedInvoicePayment.getAmount();
                              currency = linkedInvoicePayment.getCurrency();
                          }

                          invoiceApi.recordChargeback(paymentControlContext.getPaymentId(), paymentControlContext.getAttemptPaymentId(), paymentControlContext.getTransactionExternalKey(), amount, currency, internalContext);
  ```
  [Applies to]: Calculation rule (Priority P1) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: Medium

- [Kill-BR-0025] - Credit-invoice detection excludes offsetting CBA_ADJ pairs from invoice-level adjustment total
  [Description]: A CREDIT_ADJ line item counts as an 'invoice-level adjustment' for balance purposes, unless it lives on a dedicated 2-item credit-note invoice that is exactly a CREDIT_ADJ paired with an equal-and-opposite CBA_ADJ on the same invoice — in that case it's treated as account-credit issuance, not an adjustment to that invoice's own balance. For example: given Invoice X contains exactly two items: a CREDIT_ADJ of -$25.00 and a CBA_ADJ of +$25.00, when isInvoiceAdjustmentItem() is evaluated for the CREDIT_ADJ item, then It returns false (this is classified as a credit-invoice / CBA generation, not a balance adjustment on invoice X). Edge cases: Any invoice with a CREDIT_ADJ that isn't part of exactly this 2-item pattern is treated as a normal adjustment. Note (extraction review flagged this rule for SME confirmation): Confirm the 2-item CREDIT_ADJ + offsetting CBA_ADJ pattern is the sole authoritative signature of a 'credit invoice' for reporting/balance purposes, since it's inferred purely from item count/type/amount matching rather than an explicit invoice flag.
  [Line Numbers]: 39 to 70
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java
  [Source Code]:
  ```java
      public static boolean isInvoiceAdjustmentItem(final InvoiceItem invoiceItemToCheck, final Iterable<InvoiceItem> invoiceItems) {
          // Invoice level credit, i.e. credit adj, but NOT on its on own invoice
      	
      	return InvoiceItemType.CREDIT_ADJ.equals(invoiceItemToCheck.getInvoiceItemType())  &&
                  !isCreditInvoice(invoiceItems);
      }

      private static boolean isCreditInvoice(final Iterable<InvoiceItem> invoiceItems) { 

          if (Iterables.size(invoiceItems) != 2) {
              return false;
          }

          final Iterator<InvoiceItem> itr = invoiceItems.iterator();

          final InvoiceItem item1 = itr.next();
          final InvoiceItem item2 = itr.next();

          if (!InvoiceItemType.CREDIT_ADJ.equals(item1.getInvoiceItemType()) && !InvoiceItemType.CREDIT_ADJ.equals(item2.getInvoiceItemType())) {
              return false;
          }
          
          return ((InvoiceItemType.CREDIT_ADJ.equals(item1.getInvoiceItemType()) && isCbaAdjItemInCreditInvoice(item1, item2)) || 
          		(InvoiceItemType.CREDIT_ADJ.equals(item2.getInvoiceItemType()) && isCbaAdjItemInCreditInvoice(item2, item1)));

      }

      private static boolean isCbaAdjItemInCreditInvoice(final InvoiceItem creditAdjItem, final InvoiceItem itemToCheck) { 
      	return (InvoiceItemType.CBA_ADJ.equals(itemToCheck.getInvoiceItemType()) && 
          		itemToCheck.getInvoiceId().equals(creditAdjItem.getInvoiceId()) && 
          		itemToCheck.getAmount().compareTo(creditAdjItem.getAmount().negate()) == 0);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (InvoiceCalculatorUtils)
  [Confidence score]: Medium

- [Kill-BR-0026] - Plan phase sequencing via cumulative duration
  [Description]: A plan's phases (e.g. trial, discount, evergreen) run back-to-back: each phase's start date is the prior phase's start date plus that phase's configured duration; the final EVERGREEN phase has no end since it runs indefinitely. For example: given A plan with a 14-day TRIAL phase starting 2026-09-01 followed by an EVERGREEN phase, when getPhaseAlignments() computes the phase timeline, then TRIAL starts 2026-09-01, EVERGREEN starts 2026-09-15 (start + 14 days) and has no further phase after it. Edge cases: Throws SubscriptionBaseError if a non-EVERGREEN phase somehow has an unbounded/UNLIMITED duration (addDuration returns null); Throws SubscriptionBaseApiException(SUB_CREATE_BAD_PHASE) if an explicitly requested initial phase type isn't found in the plan's phase list.
  [Line Numbers]: 284 to 317
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java
  [Source Code]:
  ```java
      private List<TimedPhase> getPhaseAlignments(final Plan plan, @Nullable final PhaseType initialPhase, final DateTime initialPhaseStartDate, final InternalTenantContext context) throws SubscriptionBaseApiException {
          if (plan == null) {
              return Collections.emptyList();
          }

          final List<TimedPhase> result = new LinkedList<TimedPhase>();
          DateTime curPhaseStart = (initialPhase == null) ? initialPhaseStartDate : null;
          DateTime nextPhaseStart;
          for (final PlanPhase cur : plan.getAllPhases()) {
              // For create we can specify the phase so skip any phase until we reach initialPhase
              if (curPhaseStart == null) {
                  if (initialPhase != cur.getPhaseType()) {
                      continue;
                  }
                  curPhaseStart = initialPhaseStartDate;
              }

              result.add(new TimedPhase(cur, curPhaseStart));

              // STEPH check for duration null instead TimeUnit UNLIMITED
              if (cur.getPhaseType() != PhaseType.EVERGREEN) {
                  final Duration curPhaseDuration = cur.getDuration();
                  nextPhaseStart = addDuration(curPhaseStart, curPhaseDuration, context);
                  if (nextPhaseStart == null) {
                      throw new SubscriptionBaseError(String.format("Unexpected non ending UNLIMITED phase for plan %s",
                                                                    plan.getName()));
                  }
                  curPhaseStart = nextPhaseStart;
              }
          }

          if (initialPhase != null && curPhaseStart == null) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_BAD_PHASE, initialPhase);
          }
  ```
  [Applies to]: Calculation rule (Priority P1) — subscription module (PlanAligner)
  [Confidence score]: High

- [Kill-BR-0027] - Payment-delegated child account borrows the parent's billing state for overdue calculation
  [Description]: If a child account has a parent and payments are delegated to that parent, the child's overdue billing state (unpaid balance, oldest unpaid invoice, etc.) is computed from the parent account's data, not the child's own invoices. For example: given Child account C has parentAccountId=P and isPaymentDelegatedToParent()=true, when billingState(context) is computed for C, then the calculation is delegated to billingStateCalculator using parent account P's internal call context, not C's own invoice data.
  [Line Numbers]: 151 to 158
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java
  [Source Code]:
  ```java
      public BillingState billingState(final InternalCallContext context) throws OverdueException {
          if ((overdueable.getParentAccountId() != null) && (overdueable.isPaymentDelegatedToParent())) {
              // calculate billing state from parent account
              final InternalTenantContext internalTenantContext = internalCallContextFactory.createInternalTenantContext(overdueable.getParentAccountId(), context);
              final InternalCallContext parentAccountContext = internalCallContextFactory.createInternalCallContext(internalTenantContext.getAccountRecordId(), context);
              return billingStateCalcuator.calculateBillingState(overdueable, parentAccountContext);
          }
          return billingStateCalcuator.calculateBillingState(overdueable, context);
  ```
  [Applies to]: Calculation rule (Priority P1) — overdue module (OverdueWrapper)
  [Confidence score]: High

- [Kill-BR-0028] - Plan-phase duration date arithmetic
  [Description]: A catalog phase/plan duration (e.g. '1 MONTH', '14 DAYS', '1 YEAR') is added to a date by calling the matching Joda-Time plus-method; an UNLIMITED duration cannot be added to a date and throws an error, and a duration with no configured number is treated as a zero-length (no-op) addition. For example: given A phase duration of unit=MONTHS, number=1 and a phase start date of 2026-01-15, when The catalog engine computes the phase end date via addToDateTime/addToLocalDate, then The resulting date is 2026-02-15; if unit=UNLIMITED the call instead throws CatalogApiException(CAT_UNDEFINED_DURATION). Edge cases: number==null and unit!=UNLIMITED returns the same date unchanged (no-op); unit==UNLIMITED always throws, even if number is set. Parameters: TimeUnit values DAYS/WEEKS/MONTHS/YEARS/UNLIMITED; hardcoded switch in code.
  [Line Numbers]: 64 to 104
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultDuration.java
  [Source Code]:
  ```java
      @Override
      public DateTime addToDateTime(final DateTime dateTime) throws CatalogApiException {
          if ((number == null) && (unit != TimeUnit.UNLIMITED)) {
              return dateTime;
          }

          switch (unit) {
              case DAYS:
                  return dateTime.plusDays(number);
              case WEEKS:
                  return dateTime.plusWeeks(number);
              case MONTHS:
                  return dateTime.plusMonths(number);
              case YEARS:
                  return dateTime.plusYears(number);
              case UNLIMITED:
              default:
                  throw new CatalogApiException(ErrorCode.CAT_UNDEFINED_DURATION, unit);
          }
      }

      @Override
      public LocalDate addToLocalDate(final LocalDate localDate) throws CatalogApiException {
          if ((number == null) && (unit != TimeUnit.UNLIMITED)) {
              return localDate;
          }

          switch (unit) {
              case DAYS:
                  return localDate.plusDays(number);
              case WEEKS:
                  return localDate.plusWeeks(number);
              case MONTHS:
                  return localDate.plusMonths(number);
              case YEARS:
                  return localDate.plusYears(number);
              case UNLIMITED:
              default:
                  throw new CatalogApiException(ErrorCode.CAT_UNDEFINED_DURATION, unit);
          }
      }
  ```
  [Applies to]: Calculation rule (Priority P0) — catalog module (DefaultDuration)
  [Confidence score]: High

- [Kill-BR-0029] - First non-zero recurring charge date
  [Description]: To find when a subscription will first be charged a non-zero recurring amount, the engine walks the plan's phases in order (optionally skipping ahead to a given starting phase type), advancing the date by each phase's duration as long as that phase is finite (not UNLIMITED) and either has no recurring price or a zero recurring price; it stops (and returns the accumulated date) at the first phase that is UNLIMITED or has a non-zero recurring price. For example: given A plan with a 14-day TRIAL phase priced at $0 followed by an EVERGREEN phase priced at $9.99/month, and a subscription starting 2026-01-01, when dateOfFirstRecurringNonZeroCharge is computed for that subscription, then The trial phase is skipped (its duration of 14 days is added, since its recurring price is zero and it is finite), yielding 2026-01-15 as the first date a non-zero charge occurs. Edge cases: A CatalogApiException thrown while adding a phase duration is silently swallowed (ignored) and the date is left unchanged for that iteration. Parameters: None (purely structural over phase durations/prices).
  [Line Numbers]: 329 to 351
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  [Source Code]:
  ```java
      public DateTime dateOfFirstRecurringNonZeroCharge(final DateTime subscriptionStartDate, final PhaseType initialPhaseType) {
          DateTime result = subscriptionStartDate;
          boolean skipPhase = initialPhaseType != null;
          for (final PlanPhase phase : getAllPhases()) {
              if (skipPhase) {
                  if (phase.getPhaseType() != initialPhaseType) {
                      continue;
                  } else {
                      skipPhase = false;
                  }
              }
              final Recurring recurring = phase.getRecurring();
              if (phase.getDuration().getUnit() != TimeUnit.UNLIMITED &&
                  (recurring == null || recurring.getRecurringPrice() == null || recurring.getRecurringPrice().isZero())) {
                  try {
                      result = phase.getDuration().addToDateTime(result);
                  } catch (final CatalogApiException ignored) {
                  }
              } else {
                  break;
              }
          }
          return result;
  ```
  [Applies to]: Calculation rule (Priority P1) — catalog module (DefaultPlan)
  [Confidence score]: High

- [Kill-BR-0030] - Invoice target date is pinned to the latest existing invoice with usage/recurring items
  [Description]: When generating a new invoice, the requested target date is bumped forward (never backward) to match the latest target date among the account's existing invoices that contain at least one USAGE or RECURRING item, so invoice target dates never regress once usage/recurring billing has been generated for a later date. For example: given An existing committed invoice on the account with a RECURRING item and targetDate=2026-03-01, and a new invoice generation requested with targetDate=2026-02-15, when generateInvoice computes the adjusted target date, then The new invoice uses targetDate=2026-03-01 instead of the requested 2026-02-15. Edge cases: Invoices containing only FIXED or ITEM_ADJ items (no USAGE/RECURRING) are ignored when computing the max date; If existingInvoices is null, the requested target date is used unchanged.
  [Line Numbers]: 128 to 154
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java
  [Source Code]:
  ```java
      private LocalDate adjustTargetDate(final Iterable<Invoice> existingInvoices, final LocalDate targetDate) {
          if (existingInvoices == null) {
              return targetDate;
          }

          LocalDate maxDate = targetDate;

          for (final Invoice invoice : existingInvoices) {

              // See https://github.com/killbill/killbill/issues/1241
              final boolean containsUsageOrRecurringItems = invoice.getInvoiceItems().stream()
                      .anyMatch(input -> input.getInvoiceItemType() == InvoiceItemType.RECURRING || input.getInvoiceItemType() == InvoiceItemType.USAGE);
              if (!containsUsageOrRecurringItems) {
                  continue;
              }

              if ((invoice.getTargetDate() != null) && invoice.getTargetDate().isAfter(maxDate)) {
                  maxDate = invoice.getTargetDate();
              }
          }

          if (targetDate.compareTo(maxDate) != 0) {
              logger.info("Adjusting target date from {} to {}", targetDate, maxDate);
          }

          return maxDate;
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (DefaultInvoiceGenerator)
  [Confidence score]: High

- [Kill-BR-0031] - Repair amount is capped at the item's remaining net amount
  [Description]: When converting a CANCEL-action tree item into an actual REPAIR_ADJ invoice item, the repair amount is first prorated for the requested sub-period, then capped so it can never exceed what is still owed on the original item (its amount minus anything already adjusted or already repaired), and is floored at zero so a repair can never go negative. For example: given An original RECURRING item of $30.00 that has already been repaired for $25.00 (net amount remaining = $5.00), and a new proposed full-period repair that would prorate to $30.00, when toProratedInvoiceItem() builds the RepairAdjInvoiceItem for the CANCEL action, then The resulting repair amount is capped at the remaining net amount of $5.00 (negated to -$5.00 on the invoice item), not the full $30.00 prorated amount. Edge cases: If net amount remaining is negative or zero, the resulting repair amount is floored to $0.00 rather than going negative.
  [Line Numbers]: 182 to 199
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java
  [Source Code]:
  ```java
      public InvoiceItem toProratedInvoiceItem(final LocalDate newStartDate, final LocalDate newEndDate) {

          final int nbTotalDays = prorationFixedDays == 0 ? Days.daysBetween(startDate, endDate).getDays() : prorationFixedDays;
          final boolean prorated = !(newStartDate.compareTo(startDate) == 0 && newEndDate.compareTo(endDate) == 0);

          // Pro-ration is built by using the startDate, endDate and amount of this item instead of using the rate and a potential full period.
          final BigDecimal positiveAmount = prorated ? InvoiceDateUtils.calculateProrationBetweenDates(newStartDate, newEndDate, nbTotalDays, prorationFixedDays)
                                                                       .multiply(amount) : amount;

          if (action == ItemAction.ADD) {
              return new RecurringInvoiceItem(id, createdDate, invoiceId, accountId, bundleId, subscriptionId, productName, planName, phaseName, catalogEffectiveDate, newStartDate, newEndDate, positiveAmount, rate, currency);
          } else {
              // We first compute the maximum amount after adjustment and that sets the amount limit of how much can be repaired.
              final BigDecimal netAmount = getNetAmount();
              final BigDecimal maxAmountForRepair = positiveAmount.compareTo(netAmount) <= 0 ? positiveAmount : netAmount;
              final BigDecimal resultingAmountForRepair = maxAmountForRepair.compareTo(BigDecimal.ZERO) > 0 ? maxAmountForRepair : BigDecimal.ZERO;
              return new RepairAdjInvoiceItem(targetInvoiceId, accountId, newStartDate, newEndDate, resultingAmountForRepair.negate(), currency, linkedId);
          }
  ```
  [Applies to]: Calculation rule (Priority P0) — invoice module (Item)
  [Confidence score]: High

- [Kill-BR-0032] - Item split proration for partial-period repairs
  [Description]: When an invoice item needs to be split at a given date (e.g. a mid-period plan change triggers a partial repair), the item's amount is divided between the two resulting sub-periods using the same day-count proration ratio as recurring invoice proration, applied to the original item's full amount. For example: given An item spanning 2026-01-01 to 2026-02-01 with amount $30.00, split at 2026-01-11, when Item.split(2026-01-11) is called, then The first sub-item (2026-01-01 to 2026-01-11) gets amount0 = proration_ratio(10/31 days) * $30.00, and the second sub-item gets the remainder ($30.00 - amount0) so the two pieces always sum exactly to $30.00. Edge cases: Precondition requires the item have zero currentRepairedAmount and zero adjustedAmount before it can be split. Parameters: prorationFixedDays override, same as InvoiceDateUtils proration.
  [Line Numbers]: 152 to 166
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/tree/Item.java
  [Source Code]:
  ```java
      public Item[] split(final LocalDate splitDate) {

          // Relax this pre-condition to allow splitting from 'merge' phase as well.
          //Preconditions.checkState(action == ItemAction.ADD);
          Preconditions.checkState(currentRepairedAmount.compareTo(BigDecimal.ZERO) == 0);
          Preconditions.checkState(adjustedAmount.compareTo(BigDecimal.ZERO) == 0);


          final Item[] result =  new Item[2];

          final BigDecimal amount0 = InvoiceDateUtils.calculateProrationBetweenDates(startDate, splitDate, Days.daysBetween(startDate, endDate).getDays(), prorationFixedDays).multiply(amount);
          final BigDecimal amount1 = amount.subtract(amount0);

          result[0] = new Item(this, this.startDate, splitDate, amount0);
          result[1] = new Item(this, splitDate, this.endDate, amount1);
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (Item)
  [Confidence score]: High

- [Kill-BR-0033] - Migrated invoices always report zero balance
  [Description]: Invoices flagged as migrated (imported from a legacy system) are always treated as fully paid / zero balance regardless of their item and payment records. For example: given A migrated invoice with $500 of items and no recorded payments, when getRawBalanceForRegularInvoice is computed, then the balance returned is $0.00, not $500.00.
  [Line Numbers]: 33 to 46
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceModelDaoHelper.java
  [Source Code]:
  ```java
      public static BigDecimal getRawBalanceForRegularInvoice(final InvoiceModelDao invoiceModelDao) {

          if (invoiceModelDao.isMigrated()) {
              return BigDecimal.ZERO;
          }

          final Iterable<InvoiceItem> invoiceItems = mapInvoiceItemModelDaoToInvoiceItem(invoiceModelDao.getInvoiceItems());

          final Iterable<InvoicePayment> invoicePayments = invoiceModelDao.getInvoicePayments().stream()
                  .map(DefaultInvoicePayment::new)
                  .collect(Collectors.toUnmodifiableList());

          return InvoiceCalculatorUtils.computeRawInvoiceBalance(invoiceModelDao.getCurrency(), invoiceItems, invoicePayments);
      }
  ```
  [Applies to]: Calculation rule (Priority P1) — invoice module (InvoiceModelDaoHelper)
  [Confidence score]: High

- [Kill-BR-0034] - Invoice-generation safety bound: max daily items per subscription
  [Description]: If invoice generation would create more than a configured number of invoice items for the same subscription on the same calendar day, the system treats this as a bug/data-corruption signal and aborts the invoice run rather than generate it, to prevent runaway double/mis-billing. For example: given maxDailyNumberOfItemsSafetyBound=15 (default) and 16 invoice items get generated for the same subscription with the same creation day, when safetyBounds() runs after invoice item generation, then an InvoiceApiException (UNEXPECTED_ERROR, 'SAFETY BOUND TRIGGERED') is thrown and no invoice is persisted. Edge cases: Only items with a non-null subscriptionId are counted (line 499). Parameters: org.killbill.invoice.maxDailyNumberOfItemsSafetyBound, default 15 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:104-112); value -1 disables the check entirely (FixedAndRecurringInvoiceItemGenerator.java:493-496).
  [Line Numbers]: 492 to 516
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  [Source Code]:
  ```java
          // Trigger an exception if we create too many invoice items for a subscription on a given day
          if (config.getMaxDailyNumberOfItemsSafetyBound(internalCallContext) == -1) {
              // Safety bound disabled
              return;
          }

          for (final InvoiceItem invoiceItem : resultingItems) {
              if (invoiceItem.getSubscriptionId() != null) {
                  final LocalDate resultingItemCreationDay = trackInvoiceItemCreatedDay(invoiceItem, createdItemsPerDayPerSubscription, internalCallContext);

                  final Collection<LocalDate> creationDaysForSubscription = createdItemsPerDayPerSubscription.get(invoiceItem.getSubscriptionId());
                  int i = 0;
                  for (final LocalDate creationDayForSubscription : creationDaysForSubscription) {
                      if (creationDayForSubscription.compareTo(resultingItemCreationDay) == 0) {
                          i++;
                          if (i > config.getMaxDailyNumberOfItemsSafetyBound(internalCallContext)) {
                              // Proposed items have already been logged
                              throw new InvoiceApiException(ErrorCode.UNEXPECTED_ERROR, String.format("SAFETY BOUND TRIGGERED subscriptionId='%s', resultingItem=%s", invoiceItem.getSubscriptionId(), invoiceItem));
                          }

                      }
                  }
              }
          }
      }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (FixedAndRecurringInvoiceItemGenerator)
  [Confidence score]: High

- [Kill-BR-0035] - Invoice-generation safety bound: duplicate FIXED/RECURRING items
  [Description]: The system refuses to generate two FIXED charges for the same subscription on the same start date, or two RECURRING charges for the same subscription covering the exact same service period, treating either as an internal consistency error. For example: given Sanity safety bound enabled (default) and the invoice generator proposes two FIXED invoice items for the same subscription with identical start dates, when safetyBounds() validates the resulting item list, then an InvoiceApiException (UNEXPECTED_ERROR, 'SAFETY BOUND TRIGGERED Multiple FIXED items...') is thrown. Edge cases: Same rule applies to RECURRING items keyed by exact [startDate,endDate) interval rather than a single date (line 533-548). Parameters: org.killbill.invoice.sanitySafetyBoundEnabled, default true (InvoiceConfig.java:73-81).
  [Line Numbers]: 480 to 490
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  [Source Code]:
  ```java
          if (config.isSanitySafetyBoundEnabled(internalCallContext)) {
              final Map<UUID, MultiValueMap<LocalDate, InvoiceItem>> fixedItemsPerDateAndSubscription = new HashMap<>();
              final Map<UUID, MultiValueMap<Interval, InvoiceItem>> recurringItemsPerServicePeriodAndSubscription = new HashMap<>();
              for (final InvoiceItem resultingItem : resultingItems) {
                  if (resultingItem.getInvoiceItemType() == InvoiceItemType.FIXED) {
                      validateSafetyBoundsWithFixedInvoiceItem(fixedItemsPerDateAndSubscription, resultingItem);
                  } else if (resultingItem.getInvoiceItemType() == InvoiceItemType.RECURRING) {
                      validateSafetyBoundsWithRecurringInvoiceItem(recurringItemsPerServicePeriodAndSubscription, resultingItem);
                  }
              }
          }
  ```
  [Line Numbers]: 518 to 548
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  [Source Code]:
  ```java
      @VisibleForTesting
      void validateSafetyBoundsWithFixedInvoiceItem(final Map<UUID, MultiValueMap<LocalDate, InvoiceItem>> fixedItemsPerDateAndSubscription,
                                                    final InvoiceItem resultingItem) throws InvoiceApiException {
          if (fixedItemsPerDateAndSubscription.get(resultingItem.getSubscriptionId()) == null) {
              fixedItemsPerDateAndSubscription.put(resultingItem.getSubscriptionId(), new MultiValueHashMap<>());
          }
          fixedItemsPerDateAndSubscription.get(resultingItem.getSubscriptionId()).putElement(resultingItem.getStartDate(), resultingItem);

          final Collection<InvoiceItem> resultingInvoiceItems = fixedItemsPerDateAndSubscription.get(resultingItem.getSubscriptionId()).get(resultingItem.getStartDate());
          if (resultingInvoiceItems.size() > 1) {
              throw new InvoiceApiException(ErrorCode.UNEXPECTED_ERROR, String.format("SAFETY BOUND TRIGGERED Multiple FIXED items for subscriptionId='%s', startDate='%s', resultingItems=%s",
                                                                                      resultingItem.getSubscriptionId(), resultingItem.getStartDate(), resultingInvoiceItems));
          }
      }

      @VisibleForTesting
      void validateSafetyBoundsWithRecurringInvoiceItem(final Map<UUID, MultiValueMap<Interval, InvoiceItem>> recurringItemsPerServicePeriodAndSubscription,
                                                        final InvoiceItem resultingItem) throws InvoiceApiException {
          if (recurringItemsPerServicePeriodAndSubscription.get(resultingItem.getSubscriptionId()) == null) {
              recurringItemsPerServicePeriodAndSubscription.put(resultingItem.getSubscriptionId(), new MultiValueHashMap<>());
          }
          final Interval interval = new Interval(resultingItem.getStartDate().toDateTimeAtStartOfDay(DateTimeZone.UTC),
                                                 resultingItem.getEndDate().toDateTimeAtStartOfDay(DateTimeZone.UTC));
          recurringItemsPerServicePeriodAndSubscription.get(resultingItem.getSubscriptionId()).putElement(interval, resultingItem);

          final Collection<InvoiceItem> resultingInvoiceItems = recurringItemsPerServicePeriodAndSubscription.get(resultingItem.getSubscriptionId()).get(interval);
          if (resultingInvoiceItems.size() > 1) {
              throw new InvoiceApiException(ErrorCode.UNEXPECTED_ERROR, String.format("SAFETY BOUND TRIGGERED Multiple RECURRING items for subscriptionId='%s', startDate='%s', endDate='%s', resultingItems=%s",
                                                                                      resultingItem.getSubscriptionId(), resultingItem.getStartDate(), resultingItem.getEndDate(), resultingInvoiceItems));
          }
      }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (FixedAndRecurringInvoiceItemGenerator)
  [Confidence score]: High

- [Kill-BR-0036] - API payment cannot exceed invoice balance
  [Description]: A customer/API-initiated payment request cannot pay more than what is actually owed on the invoice; an internally-triggered payment (e.g. system dunning retry) is instead automatically capped to whatever remains outstanding rather than rejected. For example: given an invoice with balance=$40.00 and an API caller requests to pay $50.00, when the payment control plugin validates the requested amount prior to charging, then the call is aborted with PAYMENT_PLUGIN_EXCEPTION 'Invalid amount 50.00 for invoice ...: invoice balance is = 40.00'. Edge cases: invoice.getBalance() <= 0 → requestedAmount forced to 0 immediately, treated as already-paid (line 712-714); Zero-amount invoice with allowEmptyInvoice=true (default false) → payment allowed to proceed for $0 instead of being aborted (InvoicePaymentControlPluginApi.java:364-374; PaymentConfig.java:138-141); Non-API (system) payment with amount > balance is not blocked here — it is silently capped to invoice.getBalance() (line 727). Parameters: none (structural).
  [Line Numbers]: 710 to 728
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
      private BigDecimal validateAndComputePaymentAmount(final Invoice invoice, @Nullable final BigDecimal inputAmount, final boolean isApiPayment) throws PaymentControlApiException {

          if (invoice.getBalance().compareTo(BigDecimal.ZERO) <= 0) {
              return BigDecimal.ZERO;
          }

          if (isApiPayment &&
              inputAmount != null &&
              invoice.getBalance().compareTo(inputAmount) < 0) {
              log.info("invoiceId='{}' has a balance='{}' < paymentAmount='{}'", invoice.getId(), invoice.getBalance().floatValue(), inputAmount.floatValue());
              throw new PaymentControlApiException("Abort purchase call: ", new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION,
                                                                                                    String.format("Invalid amount '%s' for invoice '%s': invoice balance is = '%s'",
                                                                                                                  inputAmount,
                                                                                                                  invoice.getId(),
                                                                                                                  invoice.getBalance())));
          }

          return (inputAmount == null || invoice.getBalance().compareTo(inputAmount) < 0) ? invoice.getBalance() : inputAmount;
      }
  ```
  [Applies to]: Validation rule (Priority P0) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: High

- [Kill-BR-0037] - Refund amount validation against invoice items
  [Description]: When a refund does not specify a total amount but instead specifies per-item amounts, each item's refund amount must be positive and cannot exceed that line item's original invoice amount; if a specific total refund amount is given directly, it just needs to be positive (no per-item cross-check is enforced against overall payment size at this stage). For example: given an invoice item originally billed at $30.00, and a refund request specifying $35.00 against that item, when computeRefundAmount() validates the per-item refund map, then the call is aborted with 'You need to specify a valid invoice item amount' (specifiedItemAmount=$35.00 > itemAmount=$30.00). Edge cases: specifiedRefundAmount (total) <= 0 → aborted with 'You need to specify a positive refund amount' (line 534-536); specifiedItemAmount omitted for an item in the map → defaults to that item's full original amount (line 550, Objects.requireNonNullElse); computeRefundAmount() == 0 overall and isApiPayment=true → whole refund call aborted (line 456-464); if not an API payment, a $0 refund is allowed to proceed silently.
  [Line Numbers]: 529 to 565
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
      private BigDecimal computeRefundAmount(final UUID paymentId, @Nullable final BigDecimal specifiedRefundAmount,
                                             final Map<UUID, BigDecimal> invoiceItemIdsWithAmounts, final InternalTenantContext context)
              throws PaymentControlApiException {

          if (specifiedRefundAmount != null) {
              if (specifiedRefundAmount.compareTo(BigDecimal.ZERO) <= 0) {
                  throw new PaymentControlApiException("Failed to compute refund: ", new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION, "You need to specify a positive refund amount"));
              }
              return specifiedRefundAmount;
          }

          try {
              final List<InvoiceItem> items = invoiceApi.getInvoiceForPaymentId(paymentId, context).getInvoiceItems();
              BigDecimal amountFromItems = BigDecimal.ZERO;
              for (final Entry<UUID, BigDecimal> entry : invoiceItemIdsWithAmounts.entrySet()) {
                  final BigDecimal specifiedItemAmount = entry.getValue();
                  final BigDecimal itemAmount = getAmountFromItem(items, entry.getKey());
                  if (specifiedItemAmount != null &&
                      (specifiedItemAmount.compareTo(BigDecimal.ZERO) <= 0 || specifiedItemAmount.compareTo(itemAmount) > 0)) {
                      throw new PaymentControlApiException("Failed to compute refund: ", new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION, "You need to specify a valid invoice item amount"));
                  }
                  amountFromItems = amountFromItems.add(Objects.requireNonNullElse(specifiedItemAmount, itemAmount));
              }
              return amountFromItems;
          } catch (final InvoiceApiException e) {
              throw new PaymentControlApiException(e);
          }
      }

      private BigDecimal getAmountFromItem(final List<InvoiceItem> items, final UUID itemId) throws PaymentControlApiException {
          for (final InvoiceItem item : items) {
              if (item.getId().equals(itemId)) {
                  return item.getAmount();
              }
          }
          throw new PaymentControlApiException(String.format("Unable to find invoice item for id %s", itemId), new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION, "Invalid plugin properties"));
      }
  ```
  [Applies to]: Validation rule (Priority P0) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: High

- [Kill-BR-0038] - Target date cannot be too far in the future
  [Description]: An invoice cannot be generated for a target (as-of) date that is more than a configured number of months ahead of today. For example: given org.killbill.invoice.maxNumberOfMonthsInFuture default = 36 months, today = 2026-08-31, requested targetDate = 2030-01-01 (>36 months out), when validateTargetDate() runs at the start of invoice generation, then an InvoiceApiException with ErrorCode.INVOICE_TARGET_DATE_TOO_FAR_IN_THE_FUTURE is thrown and no invoice is generated. Edge cases: Comparison uses whole months between today and targetDate (Joda Months.monthsBetween), so day-of-month granularity within the boundary month is not itself rejected. Parameters: org.killbill.invoice.maxNumberOfMonthsInFuture, default 36 (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java:63-71).
  [Line Numbers]: 118 to 124
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java
  [Source Code]:
  ```java


      private void validateTargetDate(final LocalDate targetDate, final InternalTenantContext context) throws InvoiceApiException {
          final int maximumNumberOfMonths = config.getNumberOfMonthsInFuture(context);

          if (Months.monthsBetween(clock.getUTCToday(), targetDate).getMonths() > maximumNumberOfMonths) {
              throw new InvoiceApiException(ErrorCode.INVOICE_TARGET_DATE_TOO_FAR_IN_THE_FUTURE, targetDate.toString());
  ```
  [Applies to]: Validation rule (Priority P1) — invoice module (DefaultInvoiceGenerator)
  [Confidence score]: High

- [Kill-BR-0039] - Cannot cancel an already-cancelled entitlement
  [Description]: Cancelling an entitlement that is already CANCELLED is rejected. For example: given isEntitlementCancelled() is true, when cancelEntitlementWithDate is called again, then SUB_CANCEL_BAD_STATE exception thrown. Parameters: ErrorCode.SUB_CANCEL_BAD_STATE.
  [Line Numbers]: 364 to 366
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  [Source Code]:
  ```java
                  if (eventsStream.isEntitlementCancelled()) {
                      throw new EntitlementApiException(ErrorCode.SUB_CANCEL_BAD_STATE, getId(), EntitlementState.CANCELLED);
                  }
  ```
  [Line Numbers]: 507 to 509
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  [Source Code]:
  ```java
                  if (eventsStream.isEntitlementCancelled()) {
                      throw new EntitlementApiException(ErrorCode.SUB_CANCEL_BAD_STATE, getId(), EntitlementState.CANCELLED);
                  }
  ```
  [Applies to]: Validation rule (Priority P1) — entitlement module (DefaultEntitlement)
  [Confidence score]: High

- [Kill-BR-0040] - Cancellation effective date cannot precede entitlement or subscription start
  [Description]: You cannot backdate an entitlement cancellation to before the entitlement started, nor a billing cancel date before subscription start. For example: given Entitlement with effective start date 2026-01-01, when cancelEntitlementWithDate is called with date 2025-12-15, then SUB_INVALID_REQUESTED_DATE exception thrown.
  [Line Numbers]: 337 to 342
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  [Source Code]:
  ```java
          if (entitlementEffectiveDate == null || entitlementEffectiveDate.compareTo(getEffectiveStartDate()) < 0) {
              throw new EntitlementApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, entitlementEffectiveDate, getEffectiveStartDate());
          }
          if (billingEffectiveDate != null && billingEffectiveDate.compareTo(getSubscriptionBase().getStartDate()) < 0) {
              throw new EntitlementApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, entitlementEffectiveDate, getEffectiveStartDate());
          }
  ```
  [Applies to]: Validation rule (Priority P1) — entitlement module (DefaultEntitlement)
  [Confidence score]: High

- [Kill-BR-0041] - Uncancel only allowed with a pending or existing cancellation to reverse
  [Description]: Uncancel fails if billing already fully cancelled the subscription, or if there is no cancellation event to reverse. For example: given No cancellation event recorded, when uncancelEntitlement() is called, then ENT_UNCANCEL_BAD_STATE thrown. Edge cases: If billing had a future end date, uncancel reverses that too.
  [Line Numbers]: 422 to 455
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  [Source Code]:
  ```java
              public Void doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  if (eventsStream.isSubscriptionCancelled()) {
                      throw new EntitlementApiException(ErrorCode.SUB_UNCANCEL_BAD_STATE, getId());
                  }

                  final InternalCallContext contextWithValidAccountRecordId = internalCallContextFactory.createInternalCallContext(getAccountId(), callContext);
                  final Collection<BlockingState> pendingEntitlementCancellationEvents = eventsStream.getPendingEntitlementCancellationEvents();
                  if (eventsStream.isEntitlementCancelled()) {
                      final BlockingState cancellationEvent = eventsStream.getEntitlementCancellationEvent();
                      blockingStateDao.unactiveBlockingState(cancellationEvent.getId(), contextWithValidAccountRecordId);
                  } else if (pendingEntitlementCancellationEvents.size() > 0) {
                      // Reactivate entitlements
                      // See also https://github.com/killbill/killbill/issues/111
                      //
                      // Today we only support cancellation at SUBSCRIPTION level (Not ACCOUNT or BUNDLE), so we should really have only
                      // one future event in the list
                      //
                      for (final BlockingState futureCancellation : pendingEntitlementCancellationEvents) {
                          blockingStateDao.unactiveBlockingState(futureCancellation.getId(), contextWithValidAccountRecordId);
                      }
                  } else {
                      // Entitlement is NOT cancelled (or future cancelled), there is nothing to do
                      throw new EntitlementApiException(ErrorCode.ENT_UNCANCEL_BAD_STATE, getId());
                  }

                  // If billing was previously cancelled, reactivate
                  if (getSubscriptionBase().getFutureEndDate() != null) {
                      try {
                          getSubscriptionBase().uncancel(callContext);
                      } catch (final SubscriptionBaseApiException e) {
                          throw new EntitlementApiException(e);
                      }
                  }
                  return null;
  ```
  [Applies to]: Validation rule (Priority P1) — entitlement module (DefaultEntitlement)
  [Confidence score]: High

- [Kill-BR-0042] - Cannot void a repaired, used-credit-generating, or paid invoice
  [Description]: Voiding is blocked if the invoice was repaired, generated already-used account credit, or has any non-zero net paid/refunded amount; the first two checks apply only to COMMITTED invoices. For example: given COMMITTED invoice generated $50 credit, $30 already used, when voidInvoice is called, then CAN_NOT_VOID_INVOICE_THAT_GENERATED_USED_CREDIT thrown. Edge cases: Fully refunded (net zero) invoices remain voidable.
  [Line Numbers]: 753 to 799
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
      private void checkInvoiceNotRepaired(final InvoiceModelDao invoice) throws InvoiceApiException {
          if (invoice.getIsRepaired()) {
              throw new InvoiceApiException(ErrorCode.CAN_NOT_VOID_INVOICE_THAT_IS_REPAIRED, invoice.getId());
          }
      }

      private void checkInvoiceDoesContainUsedGeneratedCredit(final UUID accountId, final InvoiceModelDao invoice, final CallContext context) throws InvoiceApiException {
          final BigDecimal accountCBA = dao.getAccountCBA(accountId, internalCallContextFactory.createInternalTenantContext(accountId, context));
          final boolean largeCreditGenExist = invoice.getInvoiceItems().stream()
                  .anyMatch(invoiceItemModelDao -> /* Positive CBA */
                                    InvoiceItemType.CBA_ADJ == invoiceItemModelDao.getType() && /* CBA item */
                                    invoiceItemModelDao.getAmount().compareTo(BigDecimal.ZERO) > 0 && /* Credit generation */
                                    invoiceItemModelDao.getAmount().compareTo(accountCBA) > 0 /* Some of it was used already */);
          if (largeCreditGenExist) {
              throw new InvoiceApiException(ErrorCode.CAN_NOT_VOID_INVOICE_THAT_GENERATED_USED_CREDIT, invoice.getId());
          }
      }

      @Override
      public void voidInvoice(final UUID invoiceId, final CallContext context) throws InvoiceApiException {

          final UUID accountId = getInvoiceInternal(invoiceId, context).getAccountId();
          final WithAccountLock withAccountLock = new WithAccountLock() {
              @Override
              public Iterable<DefaultInvoice> prepareInvoices() throws InvoiceApiException {
                  final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(invoiceId, ObjectType.INVOICE, context);
                  final InvoiceModelDao rawInvoice = dao.getById(invoiceId, true, internalCallContext);
                  if (rawInvoice.getStatus() == InvoiceStatus.COMMITTED) {
                      checkInvoiceNotRepaired(rawInvoice);
                      checkInvoiceDoesContainUsedGeneratedCredit(accountId, rawInvoice, context);
                  }

                  final Invoice currentInvoice = new DefaultInvoice(rawInvoice, getCatalogSafelyForPrettyNames(internalCallContext));
                  checkInvoiceNotPaid(currentInvoice);

                  dao.changeInvoiceStatus(invoiceId, InvoiceStatus.VOID, internalCallContext);

                  final DefaultInvoice invoice = getInvoiceInternal(invoiceId, context);
                  return List.of(invoice);
              }
          };

          final LinkedList<PluginProperty> properties = new LinkedList<PluginProperty>();
          properties.add(new PluginProperty(INVOICE_OPERATION, "void", false));

          invoiceApiHelper.dispatchToInvoicePluginsAndInsertItems(accountId, false, withAccountLock, properties, false, context);
      }
  ```
  [Line Numbers]: 815 to 825
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
      private void checkInvoiceNotPaid(final Invoice invoice) throws InvoiceApiException {
          if (invoice.getNumberOfPayments() > 0) {
              final List<InvoicePayment> invoicePayments = invoice.getPayments();
              final BigDecimal amountPaid = InvoiceCalculatorUtils.computeInvoiceAmountPaid(invoice.getCurrency(), invoicePayments)
                                                                  .add(InvoiceCalculatorUtils.computeInvoiceAmountRefunded(invoice.getCurrency(), invoicePayments));

              if (amountPaid.compareTo(BigDecimal.ZERO) != 0) {
                  throw new InvoiceApiException(ErrorCode.CAN_NOT_VOID_INVOICE_THAT_IS_PAID, invoice.getId().toString());
              }
          }
      }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0043] - Missing currency price is a hard error
  [Description]: If a plan phase defines prices for some currencies but not the one requested, pricing fails loudly rather than defaulting to zero or another currency. For example: given A phase's prices[] array is non-empty but has no entry for currency=JPY, when getPrice(JPY) is called, then CatalogApiException CAT_NO_PRICE_FOR_CURRENCY is thrown.
  [Line Numbers]: 88 to 100
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  [Source Code]:
  ```java
      @Override
      public BigDecimal getPrice(final Currency currency) throws CatalogApiException {
          if (prices.length == 0) {
              return BigDecimal.ZERO;
          }

          for (final Price p : prices) {
              if (p.getCurrency() == currency) {
                  return p.getValue();
              }
          }
          throw new CatalogApiException(ErrorCode.CAT_NO_PRICE_FOR_CURRENCY, currency);
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultInternationalPrice)
  [Confidence score]: High

- [Kill-BR-0044] - Negative catalog prices rejected
  [Description]: Catalog validation rejects any price entry with a negative value in a supported currency. For example: given A DefaultPrice with value -5.00 USD in a catalog whose supported currencies include USD, when catalog validation runs (DefaultInternationalPrice.validate), then A validation error 'Negative value for price in currency: USD' is added; a CurrencyValueNull price (no value set) is silently skipped.
  [Line Numbers]: 107 to 125
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  [Source Code]:
  ```java
      @Override
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          final Currency[] supportedCurrencies = catalog.getSupportedCurrencies();
          if (supportedCurrencies == null) {
              return errors;
          }
          for (final Price p : prices) {
              final Currency currency = p.getCurrency();
              if (!currencyIsSupported(currency, supportedCurrencies)) {
                  errors.add("Unsupported currency: " + currency, this.getClass(), "");
              }
              try {
                  if (p.getValue().doubleValue() < 0.0) {
                      errors.add("Negative value for price in currency: " + currency, this.getClass(), "");
                  }
              } catch (CurrencyValueNull e) {
                  // No currency => nothing to check, ignore exception
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultInternationalPrice)
  [Confidence score]: High

- [Kill-BR-0045] - TOP_UP usage block requires a minimum top-up credit
  [Description]: A usage block of type TOP_UP must declare a minimum top-up credit amount; requesting that value on a non-TOP_UP block is an error. For example: given A DefaultBlock with type=TOP_UP and no minTopUpCredit set, when catalog validation runs, then Validation error 'TOP_UP block needs to define minTopUpCredit for phase X' is added; conversely calling getMinTopUpCredit() on a VANILLA/TIERED block that happens to have a non-default value throws CAT_NOT_TOP_UP_BLOCK.
  [Line Numbers]: 91 to 111
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultBlock.java
  [Source Code]:
  ```java
      @Override
      public BigDecimal getMinTopUpCredit() throws CatalogApiException {
          if (minTopUpCreditHasValue && type != BlockType.TOP_UP) {
              throw new CatalogApiException(ErrorCode.CAT_NOT_TOP_UP_BLOCK, phase.getName());
          }
          return minTopUpCredit;
      }

      @Override
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          // Safety check
          if (type == null) {
              throw new IllegalStateException("type should have been automatically been initialized with VANILLA ");
          }

          if (type == BlockType.TOP_UP && !minTopUpCreditHasValue) {
              errors.add(new ValidationError(String.format("TOP_UP block needs to define minTopUpCredit for phase %s",
                                                           phase.getName()), DefaultUsage.class, ""));
          }
          return errors;
      }
  ```
  [Applies to]: Validation rule (Priority P2) — catalog module (DefaultBlock)
  [Confidence score]: High

- [Kill-BR-0046] - Price override cannot introduce a price dimension that didn't exist
  [Description]: You can override an existing fixed or recurring price on a plan phase, but you cannot add a fixed price to a phase that never had one (or a recurring price to a phase that never had one). For example: given A plan phase with no fixed-price component (curPhase.getFixed() == null), when An override request supplies a non-null fixedPrice for that phase, then CatalogApiException CAT_INVALID_INVALID_PRICE_OVERRIDE is thrown with message 'There is no existing fixed price for the phase X'; same rule applies symmetrically for recurring price.
  [Line Numbers]: 104 to 119
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java
  [Source Code]:
  ```java
          for (int i = 0; i < resolvedOverride.length; i++) {
              final PlanPhasePriceOverride curOverride = resolvedOverride[i];
              if (curOverride != null) {
                  final DefaultPlanPhase curPhase = (DefaultPlanPhase) parentPlan.getAllPhases()[i];

                  if (curPhase.getFixed() == null && curOverride.getFixedPrice() != null) {
                      final String error = String.format("There is no existing fixed price for the phase %s", curPhase.getName());
                      throw new CatalogApiException(ErrorCode.CAT_INVALID_INVALID_PRICE_OVERRIDE, parentPlan.getName(), error);
                  }

                  if (curPhase.getRecurring() == null && curOverride.getRecurringPrice() != null) {
                      final String error = String.format("There is no existing recurring price for the phase %s", curPhase.getName());
                      throw new CatalogApiException(ErrorCode.CAT_INVALID_INVALID_PRICE_OVERRIDE, parentPlan.getName(), error);
                  }
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPriceOverrideSvc)
  [Confidence score]: High

- [Kill-BR-0047] - A recurring invoice item is only pruned once fully repaired, never over-repaired
  [Description]: When a subscription change retroactively cancels part of a billing period, the system tracks REPAIR_ADJ and ITEM_ADJ items against the original RECURRING charge; only when the repairs (net of adjustments) exactly equal the original charge is that whole item considered fully repaired and removed from consideration, and any repair/adjustment combination that would exceed the original charge is flagged as an illegal state. For example: given An original RECURRING item of $100.00 with REPAIR_ADJ items totaling -$100.00 and no ITEM_ADJ, when InvoicePruner.getFullyRepairedItemsClosure() builds the closure, then The original item and its repair items are added to the 'fully repaired' set and pruned from the working tree; if repairs+adjustments summed to more than $100.00 a Preconditions failure ('Too many repairs...') is raised (except when invoice optimization is on, where dangling repairs are tolerated). Edge cases: $0 original RECURRING items are ignored entirely (line 178-181); ITEM_ADJ added directly by an invoice plugin can silently over-adjust past the original amount; this bypass is explicitly tolerated per comment at line 197-202.
  [Line Numbers]: 156 to 232
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java
  [Source Code]:
  ```java
          private void build() {

              if (isBuilt) {
                  return;
              }

              if (target == null || target.getAmount() == null) {
                  if (repaired != null) {

                      for (final InvoiceItem i : repaired) {
                          // When invoice optimization is ON, we may miss state from previous invoices leading to having 'dangling' repair item
                          if (isInvoiceOptimizationOn) {
                              result.add(i.getId());
                          } else {
                              // Keep previous precondition behavior
                              Preconditions.checkState(false, "Missing cancelledItem for cancelItem=%s", i);
                          }
                      }
                  }
                  return;
              }

              if (target.getAmount().compareTo(BigDecimal.ZERO) == 0) {
                  // Ignore $0 items
                  return;
              }

              final BigDecimal repairedAmount = sumAmounts(repaired).negate();

              if (repaired != null) {
                  for (final InvoiceItem i : repaired) {
                      // Keep previous precondition behavior
                      Preconditions.checkState(i.getStartDate().compareTo(target.getStartDate()) >= 0 && i.getEndDate().compareTo(target.getEndDate()) <= 0,
                                               "Invalid cancelledItem=%s for cancelItem=%s", i.getId(), target.getId());
                  }
              }


              final BigDecimal totalAdjusted = sumAmounts(adjusted).negate();
              final BigDecimal remainingFromAdjusted = target.getAmount().subtract(totalAdjusted);
              final int compAdjOnly = remainingFromAdjusted.compareTo(BigDecimal.ZERO);
              if (compAdjOnly == -1) {
                  // We adjusted too much :-(
                  // In normal cases, the code should prevent this : https://github.com/killbill/killbill/blob/killbill-0.21.6/invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceDaoHelper.java#L115
                  // However, this is possible to bypass this logic when ITEM_ADJ are added from within a invoice plugin, so we ignore it -- this does not seem to create too much side effects.
                  // @see TestIntegrationInvoiceWithRepairLogic#testAdjustmentsToolarge
                  //
              } else {
                  final BigDecimal remainingFromRepair = target.getAmount().subtract(repairedAmount);
                  final int compRepair = remainingFromRepair.compareTo(BigDecimal.ZERO);


                  // We repaired the whole thing
                  if (compRepair == 0) {
                      // Keep previous precondition behavior and check there is no extra adjustment on top of it.
                      // @see TestIntegrationInvoiceWithRepairLogic#testWithFullRepairAndExistingPartialAdjustment
                      Preconditions.checkState(adjusted == null || adjusted.isEmpty(), "Too many repairs for invoiceItemId='%s', fully repaired and adjusted='%s'",
                                               target.getId(),
                                               totalAdjusted);
                      result.add(target.getId());
                      result.addAll(getIds(repaired));
                  } else {
                      // Keep previous precondition behavior and check the sum of total repair + total adjustments is not more than original amount
                      final int compWithAdjustments = remainingFromRepair.subtract(totalAdjusted).compareTo(BigDecimal.ZERO);
                      if (compWithAdjustments == -1) {
                          // @see TestIntegrationInvoiceWithRepairLogic#testWithPartialRepairAndExistingPartialTooLargeAdjustment
                          Preconditions.checkState(false, "Too many repairs for invoiceItemId='%s', partially repaired='%s', partially adjusted='%s', total='%s'",
                                                   target.getId(),
                                                   repairedAmount,
                                                   totalAdjusted,
                                                   target.getAmount());
                      }
                  }
              }

              this.isBuilt = true;
          }
  ```
  [Applies to]: Validation rule (Priority P1) — invoice module (InvoicePruner)
  [Confidence score]: High

- [Kill-BR-0048] - Usage periods already covered by an existing invoice item are skipped to prevent double billing
  [Description]: When re-running usage invoicing (e.g. after a blocking/overdue event changes billing dates), any rolled-up usage period that's already fully contained in a previously-generated usage invoice item is skipped rather than billed again. For example: given A rolled-up usage window of 2026-03-01 to 2026-03-10 that is fully contained within an existing USAGE invoice item covering 2026-03-01 to 2026-03-15, when usage items are (re)computed for the subscription, then That window is skipped (logged as ignored) and no new usage item is generated for it.
  [Line Numbers]: 294 to 307
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java
  [Source Code]:
  ```java
          // Each RolledUpUsage 'ru' is for a specific time period and across all units
          for (final RolledUpUsageWithMetadata ru : allUsage) {

              final LocalDate ruStartLocal = usageClockUtil.toLocalDate(ru.getStart(), internalTenantContext);
              final LocalDate ruEndLocal = usageClockUtil.toLocalDate(ru.getEnd(), internalTenantContext);

              final InvoiceItem existingOverlappingItem = isContainedIntoExistingUsage(ruStartLocal, ruEndLocal, existingUsage);
              if (existingOverlappingItem != null) {
                  // In case of blocking situations, when re-running the invoicing code, already billed usage maybe have another start and end date
                  // because of blocking events. We need to skip these to avoid double billing (see gotchas in testWithPartialBlockBilling).
                  log.warn("Ignoring usage {} between start={} and end={} as it has already been invoiced by invoiceItemId={}",
                            usage.getName(), ru.getStart(), ru.getEnd(), existingOverlappingItem.getId());
                  continue;
              }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (ContiguousIntervalUsageInArrear)
  [Confidence score]: High

- [Kill-BR-0049] - Same-day usage items are excluded when re-billing a larger period
  [Description]: If a usage item was previously billed for a single day (e.g. because of a same-day plan change), and the system is now computing usage for a longer period that spans that day, the single-day item is not counted as 'already billed' for that longer period — its usage gets re-evaluated as part of the larger window. For example: given An existing USAGE invoice item with startDate == endDate (a same-day item from an earlier plan change), and a new billing pass computing a multi-day period that includes that day, when getBilledItems(startDate, endDate, existingUsage) filters candidate already-billed items, then The same-day item is excluded from 'already billed' consideration (isSameDay check fails), so its usage is not double-subtracted nor double-protected against re-billing. Note (extraction review flagged this rule for SME confirmation): Confirm this doesn't cause the same-day usage record to be double-counted in the larger period's total, versus intentionally re-consolidating it.
  [Line Numbers]: 573 to 598
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/usage/ContiguousIntervalUsageInArrear.java
  [Source Code]:
  ```java
      List<InvoiceItem> getBilledItems(final LocalDate startDate, final LocalDate endDate, final List<InvoiceItem> existingUsage) {
          Preconditions.checkState(isBuilt.get(), "#getBilledItems(): isBuilt");


          return existingUsage.stream().filter(input -> {
              if (input.getInvoiceItemType() != InvoiceItemType.USAGE) {
                  return false;
              }

              // STEPH what happens if we discover usage period that overlap (one side or both side) the [startDate, endDate] interval
              final UsageInvoiceItem usageInput = (UsageInvoiceItem) input;

              // If we encounter items that were already built on the same day (e.g previous change of Plan on the same day)
              // but we are now billing for a larger period, we want to exclude such existing invoice item as they were already
              // billed for their own usage records - see TestChangeUsagePlanWithDateTime#testChangePlanOnSameDayAndRecordUsage
              final boolean isSameDay = startDate.compareTo(endDate) == 0;
              if (!isSameDay &&
                  (usageInput.getStartDate().compareTo(usageInput.getEndDate()) == 0)) {
                  return false;
              }

              return usageInput.getUsageName().equals(usage.getName()) &&
                     usageInput.getStartDate().compareTo(startDate) >= 0 &&
                     usageInput.getEndDate().compareTo(endDate) <= 0;
          }).collect(Collectors.toUnmodifiableList());
      }
  ```
  [Applies to]: Validation rule (Priority P1) — invoice module (ContiguousIntervalUsageInArrear)
  [Confidence score]: Medium

- [Kill-BR-0050] - Max add-ons of the same plan per bundle (creation)
  [Description]: A bundle cannot have more add-on subscriptions of the exact same plan than the catalog's configured 'plans allowed in bundle' limit for that plan. For example: given A plan with plansAllowedInBundle=2 and a bundle that already has 2 active/pending add-ons of that plan, when A 3rd add-on subscription of the same plan is created in that bundle, then Creation is rejected with SUB_CREATE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE. Parameters: plansAllowedInBundle: catalog-defined per plan; -1 or <=0 means unlimited (check skipped).
  [Line Numbers]: 234 to 246
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/svcs/DefaultSubscriptionBaseCreateApi.java
  [Source Code]:
  ```java
              // verify the number of subscriptions (of the same kind) allowed per bundle and the existing ones
              if (ProductCategory.ADD_ON.toString().equalsIgnoreCase(plan.getProduct().getCategory().toString())) {
                  if (plan.getPlansAllowedInBundle() != -1 && plan.getPlansAllowedInBundle() > 0) {
                      // TODO We should also look to the specifiers being created for validation
                      final List<DefaultSubscriptionBase> subscriptionsForBundle = getSubscriptionsForBundle(bundle.getId(), null, catalog, addonUtils, callContext, context);
                      final int existingAddOnsWithSamePlanName = addonUtils.countExistingAddOnsWithSamePlanName(subscriptionsForBundle, plan.getName());
                      final int currentAddOnsWithSamePlanName = countCurrentAddOnsWithSamePlanName(entitlementsPlans, plan);
                      if ((existingAddOnsWithSamePlanName + currentAddOnsWithSamePlanName) > plan.getPlansAllowedInBundle()) {
                          // a new ADD_ON subscription of the same plan can't be added because it has reached its limit by bundle
                          throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE, plan.getName());
                      }
                  }
              }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseCreateApi)
  [Confidence score]: High

- [Kill-BR-0051] - Max add-ons of the same plan per bundle (plan change)
  [Description]: Changing a subscription's plan to an add-on plan is blocked if the bundle already has as many add-ons of the target plan as the catalog allows. For example: given A target add-on plan with plansAllowedInBundle=1 and the bundle already has 1 existing add-on of that plan, when Another subscription in the bundle is changed to that same add-on plan, then The change is rejected with SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE. Parameters: plansAllowedInBundle: catalog-defined per plan
**Suspected defect:** Code comment at line 479 says 'the plan can be changed... because it has reached its limit' which contradicts the following throw statement (change is actually rejected); comment text appears inverted/copy-paste error, not an instruction to the analyzer.
  [Line Numbers]: 474 to 486
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
          if (ProductCategory.ADD_ON.toString().equalsIgnoreCase(newPlan.getProduct().getCategory().toString())) {
              if (newPlan.getPlansAllowedInBundle() != -1
                  && newPlan.getPlansAllowedInBundle() > 0
                  && addonUtils.countExistingAddOnsWithSamePlanName(dao.getSubscriptions(subscription.getBundleId(), null, catalog, internalCallContext), newPlan.getName())
                     >= newPlan.getPlansAllowedInBundle()) {
                  // the plan can be changed to the new value, because it has reached its limit by bundle
                  throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE, newPlan.getName());
              }
          }

          if (newPlan.getProduct().getCategory() != subscription.getCategory()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_INVALID, subscription.getId());
          }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0052] - Plan change forbidden across product categories
  [Description]: A subscription can only be changed to a plan in the same product category it currently has (e.g. a BASE plan cannot be changed into an ADD_ON plan or vice versa). For example: given A subscription currently on a BASE-category plan, when A change-plan request targets a plan whose product category is ADD_ON, then The change is rejected with SUB_CHANGE_INVALID. Parameters: None (structural category comparison).
  [Line Numbers]: 484 to 486
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
          if (newPlan.getProduct().getCategory() != subscription.getCategory()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_INVALID, subscription.getId());
          }
  ```
  [Applies to]: Validation rule (Priority P0) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0053] - Change/cancel effective date cannot precede last processed transition
  [Description]: A requested effective date for a subscription change (or a cancellation date computed elsewhere) must not be earlier than the subscription's last transition (or its start date if no transition yet occurred). For example: given A subscription whose last transition (e.g. PHASE) occurred on 2026-01-15, when A change-plan request with requested date 2026-01-10 (before the last transition) is submitted, then The call is rejected with SUB_INVALID_REQUESTED_DATE.
  [Line Numbers]: 823 to 833
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private void validateEffectiveDate(final SubscriptionBase subscription, final ReadableInstant effectiveDate) throws SubscriptionBaseApiException {

          final SubscriptionBaseTransition previousTransition = subscription.getPreviousTransition();

          // Our effectiveDate must be after or equal the last transition that already occured (START, PHASE1, PHASE2,...) or the startDate for future started subscription
          final DateTime earliestValidDate = previousTransition != null ? previousTransition.getEffectiveTransitionTime() : subscription.getStartDate();
          if (effectiveDate.isBefore(earliestValidDate)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE,
                                                     effectiveDate.toString(), previousTransition != null ? previousTransition.getEffectiveTransitionTime() : "null");
          }
      }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0054] - Usage/product limit compliance check (min/max)
  [Description]: A usage unit value is checked against a catalog-configured max and min for that unit; if a max is set the value must not exceed it, and if a min is set the value must comply with it. For example: given A catalog Limit for unit 'users' with min=5 and no max, when compliesWith(value) is evaluated for value=10, then Per the current code, compliance requires value <= min, so value=10 would be reported non-compliant even though it exceeds (not falls below) the min. Parameters: max, min: BigDecimal, catalog-defined per Limit/Unit; sentinel -1 (DEFAULT_NON_REQUIRED_BIGDECIMAL_FIELD_VALUE) means 'not set'
**Suspected defect:** The min branch `return !minHasValue || value.compareTo(min) <= 0;` requires value <= min to comply, which is the inverse of the conventional 'value must be >= min' semantics implied by a Limit named 'min'. No caller of compliesWith()/compliesWithLimits() was found wired into subscription/invoice eligibility flows in this codebase, so real-world impact is unclear.. Note (extraction review flagged this rule for SME confirmation): Is DefaultLimit.compliesWith's min check intentionally inverted (i.e. does 'min' actually mean an upper bound for some unit types), or is `value.compareTo(min) <= 0` a bug that should be `>= 0`? Also confirm whether any plugin/consumer actually calls compliesWith/compliesWithLimits in production.
  [Line Numbers]: 100 to 105
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultLimit.java
  [Source Code]:
  ```java
      public boolean compliesWith(final BigDecimal value) {
          if (maxHasValue && value.compareTo(max) > 0) {
              return false;
          }
          return !minHasValue || value.compareTo(min) <= 0;
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultLimit)
  [Confidence score]: Low

- [Kill-BR-0055] - Account external key must be unique and ≤ 255 characters
  [Description]: When creating an account, the external key (if supplied) must not already be in use by another account, and must be 255 characters or fewer. For example: given An external key 'cust-123' already assigned to an existing account, when createAccount is called again with externalKey='cust-123', then The call fails with ACCOUNT_ALREADY_EXISTS; separately, a key longer than 255 chars fails with EXTERNAL_KEY_LIMIT_EXCEEDED. Parameters: External key max length: 255 characters (hardcoded).
  [Line Numbers]: 85 to 104
  [Source Code File Name]: account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java
  [Source Code]:
  ```java
      public Account createAccount(final AccountData data, final CallContext context) throws AccountApiException {
          // Not transactional, but there is a db constraint on that column
          if (data.getExternalKey() != null && getIdFromKey(data.getExternalKey(), context) != null) {
              throw new AccountApiException(ErrorCode.ACCOUNT_ALREADY_EXISTS, data.getExternalKey());
          }

          final InternalCallContext internalContext = internalCallContextFactory.createInternalCallContextWithoutAccountRecordId(context);

          if (data.getParentAccountId() != null) {
              // verify that parent account exists if parentAccountId is not null
              final ImmutableAccountData immutableAccountData = immutableAccountInternalApi.getImmutableAccountDataById(data.getParentAccountId(), internalContext);
              if (immutableAccountData == null) {
                  throw new AccountApiException(ErrorCode.ACCOUNT_DOES_NOT_EXIST_FOR_ID, data.getParentAccountId());
              }
          }

          final AccountModelDao account = new AccountModelDao(data);
          if (null != account.getExternalKey() && account.getExternalKey().length() > 255) {
              throw new AccountApiException(ErrorCode.EXTERNAL_KEY_LIMIT_EXCEEDED);
          }
  ```
  [Applies to]: Validation rule (Priority P1) — account module (DefaultAccountUserApi)
  [Confidence score]: High

- [Kill-BR-0056] - Parent account must exist when linking a new account
  [Description]: If a new account specifies a parent account id, that parent account must already exist. For example: given A createAccount request with parentAccountId pointing to a non-existent account, when createAccount is called, then The call fails with ACCOUNT_DOES_NOT_EXIST_FOR_ID.
  [Line Numbers]: 93 to 99
  [Source Code File Name]: account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java
  [Source Code]:
  ```java
          if (data.getParentAccountId() != null) {
              // verify that parent account exists if parentAccountId is not null
              final ImmutableAccountData immutableAccountData = immutableAccountInternalApi.getImmutableAccountDataById(data.getParentAccountId(), internalContext);
              if (immutableAccountData == null) {
                  throw new AccountApiException(ErrorCode.ACCOUNT_DOES_NOT_EXIST_FOR_ID, data.getParentAccountId());
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — account module (DefaultAccountUserApi)
  [Confidence score]: High

- [Kill-BR-0057] - Account external key, currency, BCD, timezone, and reference time are immutable once set
  [Description]: Once an account has a non-null external key, currency, bill-cycle-day, timezone, or reference time, an update attempting to change any of those fields to a different value is rejected outright (this API doesn't support changing them). For example: given An account with currency=USD already set, when An update request supplies currency=EUR, then IllegalArgumentException 'Killbill doesn't support updating the account currency yet' is thrown; the same pattern applies to externalKey, billCycleDayLocal, timeZone, and referenceTime (date-only compare). Parameters: DEFAULT_BILLING_CYCLE_DAY_LOCAL sentinel value used to mean 'no BCD set'. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 is justified: `validateAccountUpdateInput` (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:502-546) is a data-integrity guard on core account attributes (currency, external key, BCD, timezone, reference time) that feed invoicing/proration/tax logic downstrea.
  [Line Numbers]: 502 to 546
  [Source Code File Name]: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  [Source Code]:
  ```java
      public void validateAccountUpdateInput(final Account currentAccount, boolean ignoreNullInput) {

          //
          // We don't allow update on the following fields:
          //
          // All these conditions are written in the exact same way:
          //
          // There is already a defined value BUT those don't match (either input is null or different) => Not Allowed
          // * ignoreNullInput = false (case where we allow to reset values)
          // * ignoreNullInput = true (case where we DON'T allow to reset values and so if such value is null we ignore the check)
          //
          //
          if ((ignoreNullInput || externalKey != null) &&
              currentAccount.getExternalKey() != null &&
              !currentAccount.getExternalKey().equals(externalKey)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account external key yet: new=%s, current=%s",
                                                               externalKey, currentAccount.getExternalKey()));
          }

          if ((ignoreNullInput || currency != null) &&
              currentAccount.getCurrency() != null &&
              !currentAccount.getCurrency().equals(currency)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account currency yet: new=%s, current=%s",
                                                               currency, currentAccount.getCurrency()));
          }

          if ((ignoreNullInput || (billCycleDayLocal != null && billCycleDayLocal != DEFAULT_BILLING_CYCLE_DAY_LOCAL)) &&
              currentAccount.getBillCycleDayLocal() != DEFAULT_BILLING_CYCLE_DAY_LOCAL && // There is already a BCD set
              !currentAccount.getBillCycleDayLocal().equals(billCycleDayLocal)) { // and it does not match we we have
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account BCD yet: new=%s, current=%s", billCycleDayLocal, currentAccount.getBillCycleDayLocal()));
          }

          if ((ignoreNullInput || timeZone != null) &&
              currentAccount.getTimeZone() != null &&
              !currentAccount.getTimeZone().equals(timeZone)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account timeZone yet: new=%s, current=%s",
                                                               timeZone, currentAccount.getTimeZone()));
          }

          if (referenceTime != null && currentAccount.getReferenceTime().withMillisOfDay(0).compareTo(referenceTime.withMillisOfDay(0)) != 0) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account referenceTime yet: new=%s, current=%s",
                                                               referenceTime, currentAccount.getReferenceTime()));
          }

      }
  ```
  [Applies to]: Validation rule (Priority P0) — account module (DefaultAccount)
  [Confidence score]: Medium

- [Kill-BR-0058] - Usage record idempotency by tracking id
  [Description]: When recording rolled-up usage, if a tracking id is supplied and usage rows already exist with that tracking id, the submission is rejected as a duplicate; if no tracking id is supplied, a new random one is generated (allowing the submission). For example: given A prior successful recordRolledUpUsage call used trackingId='batch-42', when recordRolledUpUsage is called again with trackingId='batch-42', then The call fails with USAGE_RECORD_TRACKING_ID_ALREADY_EXISTS, preventing double-counting of usage.
  [Line Numbers]: 70 to 92
  [Source Code File Name]: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  [Source Code]:
  ```java
      @Override
      public void recordRolledUpUsage(final SubscriptionUsageRecord record, final CallContext callContext) throws UsageApiException {
          final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(record.getSubscriptionId(), ObjectType.SUBSCRIPTION, callContext);

          final String trackingIds;
          if (record.getTrackingId() == null || record.getTrackingId().isEmpty()) {
              trackingIds = UUIDs.randomUUID().toString();
          // check if we have (at least) one row with the supplied tracking id
          } else if (recordsWithTrackingIdExist(record, internalCallContext)) {
              throw new UsageApiException(ErrorCode.USAGE_RECORD_TRACKING_ID_ALREADY_EXISTS, record.getTrackingId());
          } else {
              trackingIds = record.getTrackingId();
          }


          final List<RolledUpUsageModelDao> usages = new ArrayList<>();
          for (final UnitUsageRecord unitUsageRecord : record.getUnitUsageRecord()) {
              for (final UsageRecord usageRecord : unitUsageRecord.getDailyAmount()) {
                  usages.add(new RolledUpUsageModelDao(record.getSubscriptionId(), unitUsageRecord.getUnitType(), usageRecord.getDate(), usageRecord.getAmount(), trackingIds));
              }
          }
          rolledUpUsageDao.record(usages, internalCallContext);
      }
  ```
  [Applies to]: Validation rule (Priority P1) — usage module (DefaultUsageUserApi)
  [Confidence score]: High

- [Kill-BR-0059] - Payment transaction external key uniqueness and account isolation
  [Description]: A payment transaction external key cannot be reused for a second SUCCESSful transaction of a non-CHARGEBACK type, and a given external key cannot be shared across different accounts. For example: given A transaction with external key 'txn-001' already SUCCESS and type PURCHASE for account A, when A new transaction is submitted reusing external key 'txn-001', then The call fails with PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS; if the existing transaction under that key belongs to a different account, it instead fails with PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID. Parameters: CHARGEBACK is the sole transaction type exempted from the 'no reuse after success' rule. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 justified (compliance lens): The rule guards two things a finance controller/auditor would flag if silently broken.
  [Line Numbers]: 327 to 350
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java
  [Source Code]:
  ```java
      private void runSanityOnTransactionExternalKey(final Iterable<PaymentTransactionModelDao> allPaymentTransactionsForKey,
                                                     final PaymentStateContext paymentStateContext,
                                                     final InternalCallContext internalCallContext) throws PaymentApiException {
          for (final PaymentTransactionModelDao paymentTransactionModelDao : allPaymentTransactionsForKey) {
              // Sanity: verify we don't already have a successful transaction for that key (chargeback reversals are a bit special, it's the only transaction type we can revert)
              if (paymentTransactionModelDao.getTransactionExternalKey().equals(paymentStateContext.getPaymentTransactionExternalKey()) &&
                  paymentTransactionModelDao.getTransactionStatus() == TransactionStatus.SUCCESS &&
                  paymentTransactionModelDao.getTransactionType() != TransactionType.CHARGEBACK) {
                  throw new PaymentApiException(ErrorCode.PAYMENT_ACTIVE_TRANSACTION_KEY_EXISTS, paymentStateContext.getPaymentTransactionExternalKey());
              }

              // Sanity: don't share keys across accounts
              if (!paymentTransactionModelDao.getAccountRecordId().equals(internalCallContext.getAccountRecordId())) {
                  UUID accountId;
                  try {
                      accountId = accountInternalApi.getAccountByRecordId(paymentTransactionModelDao.getAccountRecordId(), internalCallContext).getId();
                  } catch (final AccountApiException e) {
                      log.warn("Unable to retrieve account", e);
                      accountId = null;
                  }
                  throw new PaymentApiException(ErrorCode.PAYMENT_TRANSACTION_DIFFERENT_ACCOUNT_ID, accountId);
              }
          }
      }
  ```
  [Applies to]: Validation rule (Priority P0) — payment module (PaymentProcessor)
  [Confidence score]: Medium

- [Kill-BR-0060] - At most one PENDING initial payment transaction per payment
  [Description]: A payment cannot have more than one PENDING AUTHORIZE, PURCHASE, or CREDIT transaction outstanding at the same time; a new initiating transaction request while one is already PENDING is rejected. Additionally, if reusing an existing transaction id/key, the transaction type of the new request must match the existing transaction's type, and there must never be more than one completion candidate. For example: given A payment with an existing PENDING AUTHORIZE transaction, when A new AUTHORIZE/PURCHASE/CREDIT request (without matching id/key) comes in for the same payment, then The call fails with PAYMENT_INVALID_OPERATION; a mismatched transaction type on a matching id/key fails with PAYMENT_INVALID_PARAMETER; internal invariant enforced via Preconditions.checkState that at most one completion candidate exists. Parameters: Transaction types subject to the single-pending rule: AUTHORIZE, PURCHASE, CREDIT. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: Re-derived from PaymentProcessor.java:352-385 (invoked only when paymentId is set and either transactionId or paymentTransactionExternalKey is present, per line 297).
  [Line Numbers]: 352 to 385
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentProcessor.java
  [Source Code]:
  ```java
      private PaymentTransactionModelDao findTransactionToCompleteAndRunSanityChecks(final PaymentModelDao paymentModelDao,
                                                                                     final Iterable<PaymentTransactionModelDao> paymentTransactionsForCurrentPayment,
                                                                                     final PaymentStateContext paymentStateContext,
                                                                                     final InternalCallContext internalCallContext) throws PaymentApiException {
          final Collection<PaymentTransactionModelDao> completionCandidates = new LinkedList<>();
          for (final PaymentTransactionModelDao paymentTransactionModelDao : paymentTransactionsForCurrentPayment) {
              // Check if we already have a transaction for that id or key
              if (!(paymentStateContext.getTransactionId() != null && paymentTransactionModelDao.getId().equals(paymentStateContext.getTransactionId())) &&
                  !(paymentStateContext.getPaymentTransactionExternalKey() != null && paymentTransactionModelDao.getTransactionExternalKey().equals(paymentStateContext.getPaymentTransactionExternalKey()))) {
                  // Sanity: if not, prevent multiple PENDING transactions for initial calls (cannot be enforced by the state machine unfortunately)
                  if ((paymentTransactionModelDao.getTransactionType() == TransactionType.AUTHORIZE ||
                       paymentTransactionModelDao.getTransactionType() == TransactionType.PURCHASE ||
                       paymentTransactionModelDao.getTransactionType() == TransactionType.CREDIT) &&
                      paymentTransactionModelDao.getTransactionStatus() == TransactionStatus.PENDING) {
                      throw new PaymentApiException(ErrorCode.PAYMENT_INVALID_OPERATION, paymentTransactionModelDao.getTransactionType(), paymentModelDao.getStateName());
                  } else {
                      continue;
                  }
              }

              // Sanity: if we already have a transaction for that id or key, the transaction type must match
              if (paymentTransactionModelDao.getTransactionType() != paymentStateContext.getTransactionType()) {
                  throw new PaymentApiException(ErrorCode.PAYMENT_INVALID_PARAMETER, "transactionType", String.format("%s doesn't match existing transaction type %s", paymentStateContext.getTransactionType(), paymentTransactionModelDao.getTransactionType()));
              }

              // UNKNOWN transactions are potential candidates, we'll invoke the Janitor first though
              if (paymentTransactionModelDao.getTransactionStatus() == TransactionStatus.PENDING || paymentTransactionModelDao.getTransactionStatus() == TransactionStatus.UNKNOWN) {
                  completionCandidates.add(paymentTransactionModelDao);
              }
          }

          Preconditions.checkState(Iterables.size(completionCandidates) <= 1, "There should be at most one completion candidate");
          return Iterables.getLast(completionCandidates, null);
      }
  ```
  [Applies to]: Validation rule (Priority P0) — payment module (PaymentProcessor)
  [Confidence score]: Medium

- [Kill-BR-0061] - Add-on creation eligibility against base plan
  [Description]: An add-on can only be created if its base subscription is active (or pending-future and the add-on start is not before the base start), the add-on's product is not already included for free in the base product, and the add-on product is listed as available for the base product in the catalog. For example: given A base subscription in CANCELLED state, or an add-on product not listed under the base product's 'available' add-ons, when checkAddonCreationRights is invoked while creating the add-on, then SubscriptionBaseApiException is thrown with SUB_CREATE_AO_BP_NON_ACTIVE, SUB_CREATE_AO_ALREADY_INCLUDED, or SUB_CREATE_AO_NOT_AVAILABLE respectively. Parameters: None (driven by catalog's per-product available/included add-on lists). Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (I read AddonUtils.java:36-53 (and the caller SubscriptionApiBase.java:229-245 for context). FAITHFULNESS ISSUE FOUND: The extracted rule's plain-English description of the PENDING-base branch has the date comparison backwards.
  [Line Numbers]: 36 to 53
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java
  [Source Code]:
  ```java
      public void checkAddonCreationRights(final SubscriptionBase baseSubscription, final Plan targetAddOnPlan, final DateTime requestedDate, final InternalTenantContext context)
              throws SubscriptionBaseApiException {
          if (baseSubscription.getState() == EntitlementState.CANCELLED ||
              (baseSubscription.getState() == EntitlementState.PENDING && context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_BP_NON_ACTIVE, targetAddOnPlan.getName());
          }

          final Plan currentOrPendingPlan = baseSubscription.getCurrentOrPendingPlan();
          final Product baseProduct = currentOrPendingPlan.getProduct();
          if (isAddonIncluded(baseProduct, targetAddOnPlan)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_ALREADY_INCLUDED,
                                                     targetAddOnPlan.getName(), currentOrPendingPlan.getProduct().getName());
          }

          if (!isAddonAvailable(baseProduct, targetAddOnPlan)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_NOT_AVAILABLE,
                                                     targetAddOnPlan.getName(), currentOrPendingPlan.getProduct().getName());
          }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (AddonUtils)
  [Confidence score]: Medium

- [Kill-BR-0062] - Re-cancellation of an already future-cancelled/expiring subscription is blocked unless the new date is earlier
  [Description]: If a subscription already has a pending future cancellation or a pending FIXEDTERM expiry, a new cancel request is rejected unless its effective date is strictly earlier than the existing pending one (i.e., the user must uncancel before re-cancelling for a later date); a strictly earlier date silently invalidates the old pending cancellation. For example: given A subscription with a pending future CANCEL transition effective 2026-10-01, when cancelWithRequestedDate is called again with effective date 2026-11-01, then SubscriptionBaseApiException SUB_CANCEL_BAD_STATE ('PENDING CANCELLED') is thrown; calling it again with an earlier date (e.g. 2026-09-01) is allowed and replaces the pending cancellation.
  [Line Numbers]: 264 to 296
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private boolean doCancelPlan(final Map<DefaultSubscriptionBase, DateTime> subscriptions, final SubscriptionCatalog catalog, final InternalCallContext internalCallContext) throws SubscriptionBaseApiException {
          final List<DefaultSubscriptionBase> subscriptionsToBeCancelled = new LinkedList<>();
          final List<SubscriptionBaseEvent> cancelEvents = new LinkedList<>();

          try {
              for (final Entry<DefaultSubscriptionBase, DateTime> entry : subscriptions.entrySet()) {
                  final DefaultSubscriptionBase subscription = entry.getKey();
                  final EntitlementState currentState = subscription.getState();
                  if (currentState == EntitlementState.CANCELLED || currentState == EntitlementState.EXPIRED) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), currentState);
                  }
                  final DateTime effectiveDate = entry.getValue();

                  // If subscription was future cancelled at an earlier date we disallow the operation -- i.e,
                  // user should first uncancel prior trying to cancel again.
                  // However, note that in case a future cancellation already exists with a greater or equal effectiveDate,
                  // the operation is allowed, but such existing cancellation would become invalidated (is_active=0)
                  final SubscriptionBaseTransition pendingTransition = subscription.getPendingTransition();
                  if (pendingTransition != null &&
                      pendingTransition.getTransitionType() == SubscriptionBaseTransitionType.CANCEL &&
                      pendingTransition.getEffectiveTransitionTime().compareTo(effectiveDate) < 0) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), "PENDING CANCELLED");
                  }
                  // Similarly, if subscription is cancelled with date past the expiry date (in case of a FIXEDTERM phase), we disallow the operation
                  if (pendingTransition != null &&
                      pendingTransition.getTransitionType() == SubscriptionBaseTransitionType.EXPIRED &&
                      pendingTransition.getEffectiveTransitionTime().compareTo(effectiveDate) < 0) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), "PENDING EXPIRY");
                  }

                  validateEffectiveDate(subscription, effectiveDate);

                  subscriptionsToBeCancelled.add(subscription);
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0063] - Effective date for subscription mutation must not precede last recorded transition
  [Description]: Any requested effective date for a subscription change/cancel must be on or after the subscription's most recent past transition (or its start date if none has occurred yet). For example: given A subscription created 2026-01-01 with a PHASE transition on 2026-04-01, when A cancellation is requested with effective date 2026-03-01, then SubscriptionBaseApiException SUB_INVALID_REQUESTED_DATE is thrown because 2026-03-01 precedes the last transition (2026-04-01).
  [Line Numbers]: 823 to 833
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private void validateEffectiveDate(final SubscriptionBase subscription, final ReadableInstant effectiveDate) throws SubscriptionBaseApiException {

          final SubscriptionBaseTransition previousTransition = subscription.getPreviousTransition();

          // Our effectiveDate must be after or equal the last transition that already occured (START, PHASE1, PHASE2,...) or the startDate for future started subscription
          final DateTime earliestValidDate = previousTransition != null ? previousTransition.getEffectiveTransitionTime() : subscription.getStartDate();
          if (effectiveDate.isBefore(earliestValidDate)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE,
                                                     effectiveDate.toString(), previousTransition != null ? previousTransition.getEffectiveTransitionTime() : "null");
          }
      }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0064] - Change-plan blocked for non-active, future-cancelled, or already-future-expired subscriptions
  [Description]: A plan change is rejected if the subscription is CANCELLED/EXPIRED, if the requested effective date precedes the subscription start, if the subscription already has a pending future cancellation, or if the requested effective date falls after an already-scheduled future expiry. For example: given A subscription with a pending future EXPIRED transition on 2026-12-01, when changePlanWithRequestedDate is called with effective date 2026-12-15, then SubscriptionBaseApiException SUB_CHANGE_FUTURE_EXPIRED is thrown.
  [Line Numbers]: 835 to 850
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private void validateSubscriptionStateForChangePlan(final DefaultSubscriptionBase subscription, @Nullable final DateTime effectiveDate) throws SubscriptionBaseApiException {

          final EntitlementState currentState = subscription.getState();
          if (currentState == EntitlementState.CANCELLED ||
              currentState == EntitlementState.EXPIRED ||
              // We don't look for PENDING because as long as change is after startDate, we want to allow it.
              effectiveDate != null && effectiveDate.compareTo(subscription.getStartDate()) < 0) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_NON_ACTIVE, subscription.getId(), currentState);
          }
          if (subscription.isFutureCancelled()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_FUTURE_CANCELLED, subscription.getId());
          }
          if (effectiveDate != null && subscription.getFutureExpiryDate() != null && subscription.getFutureExpiryDate().isBefore(effectiveDate)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_FUTURE_EXPIRED, subscription.getId());
          }
      }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0065] - Max add-on instances per bundle enforced on plan change
  [Description]: When changing a subscription to an add-on plan that has a configured 'plans allowed in bundle' limit (a positive number, -1 means unlimited), the change is rejected if the bundle already has that many active/pending subscriptions on the same plan; also, a plan change may never move a subscription into a different product category than it currently has. For example: given An add-on plan configured with plansAllowedInBundle = 1, and the bundle already has one active subscription on that plan, when changePlan requests the same add-on plan for a second subscription in the bundle, then SubscriptionBaseApiException SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE is thrown. Parameters: plansAllowedInBundle: per-plan catalog value, -1 = unlimited.
  [Line Numbers]: 473 to 486
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
          final PhaseType initialPhaseType = planPhaseSpecifier.getPhaseType();
          if (ProductCategory.ADD_ON.toString().equalsIgnoreCase(newPlan.getProduct().getCategory().toString())) {
              if (newPlan.getPlansAllowedInBundle() != -1
                  && newPlan.getPlansAllowedInBundle() > 0
                  && addonUtils.countExistingAddOnsWithSamePlanName(dao.getSubscriptions(subscription.getBundleId(), null, catalog, internalCallContext), newPlan.getName())
                     >= newPlan.getPlansAllowedInBundle()) {
                  // the plan can be changed to the new value, because it has reached its limit by bundle
                  throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE, newPlan.getName());
              }
          }

          if (newPlan.getProduct().getCategory() != subscription.getCategory()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_INVALID, subscription.getId());
          }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0066] - Plugins may not set reserved entitlement blocking states or service name
  [Description]: External callers/plugins adding a custom blocking state may not use the entitlement service's own service name, and may not set the state name to one of the reserved internal states (ENT_CANCELLED, ENT_BLOCKED, ENT_CLEAR), preventing plugins from hijacking internal entitlement/pause state names. For example: given A plugin calls addBlockingState with stateName='ENT_BLOCKED', when The API validates the input, then EntitlementApiException SUB_BLOCKING_STATE_INVALID_ARG ('Need to specify a valid stateName') is thrown. Parameters: Reserved names: ENT_CANCELLED, ENT_BLOCKED, ENT_CLEAR; reserved service: entitlement-service.
  [Line Numbers]: 369 to 379
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultSubscriptionApi.java
  [Source Code]:
  ```java
          // This is in no way an exhaustive arg validation, but to to ensure plugin would not hijack private entitlement state or service name
          if (inputBlockingState.getService() == null || inputBlockingState.getService().equals(KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName())) {
              throw new EntitlementApiException(ErrorCode.SUB_BLOCKING_STATE_INVALID_ARG, "Need to specify a valid serviceName");
          }

          if (inputBlockingState.getStateName() == null ||
              inputBlockingState.getStateName().equals(DefaultEntitlementApi.ENT_STATE_CANCELLED) ||
              inputBlockingState.getStateName().equals(DefaultEntitlementApi.ENT_STATE_BLOCKED) ||
              inputBlockingState.getStateName().equals(DefaultEntitlementApi.ENT_STATE_CLEAR)) {
              throw new EntitlementApiException(ErrorCode.SUB_BLOCKING_STATE_INVALID_ARG, "Need to specify a valid stateName");
          }
  ```
  [Applies to]: Validation rule (Priority P1) — entitlement module (DefaultSubscriptionApi)
  [Confidence score]: High

- [Kill-BR-0067] - Usage tier section must define blocks or limits appropriate to its type
  [Description]: In the product catalog, an IN_ARREAR CAPACITY usage tier must declare at least one limit, and an IN_ARREAR CONSUMABLE usage tier must declare at least one priced block; catalogs missing these are rejected at load time. For example: given A catalog XML defines a usage section with billingMode=IN_ARREAR and usageType=CONSUMABLE but no <blocks> element, when The catalog is validated on load, then A ValidationError is raised: "Usage [IN_ARREAR CONSUMABLE] section of phase <phase> needs to define some blocks".
  [Line Numbers]: 148 to 159
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultTier.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (billingMode == BillingMode.IN_ARREAR && usageType == UsageType.CAPACITY && limits.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ARREAR CAPACITY] section of phase %s needs to define some limits",
                                                           phase.getName()), DefaultUsage.class, ""));
          }
          if (billingMode == BillingMode.IN_ARREAR && usageType == UsageType.CONSUMABLE && blocks.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ARREAR CONSUMABLE] section of phase %s needs to define some blocks",
                                                           phase.getName()), DefaultUsage.class, ""));
          }
          validateCollection(catalog, errors, limits);
          return errors;
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultTier)
  [Confidence score]: High

- [Kill-BR-0068] - Bundle transfer eligibility: skip cancelled subscriptions and optionally skip add-ons
  [Description]: During a bundle transfer, subscriptions already in CANCELLED state are not carried over to the new account, and add-on subscriptions are only carried over if the caller explicitly requests add-on transfer. For example: given A bundle with a cancelled add-on and an active add-on, transferAddOn=false, when transferBundle() iterates the bundle's subscriptions, then The cancelled subscription is skipped entirely; the active add-on is also skipped (not created on the destination account) because transferAddOn is false.
  [Line Numbers]: 242 to 245
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java
  [Source Code]:
  ```java
                  // Skip already cancelled subscriptions
                  if (oldSubscription.getState() == EntitlementState.CANCELLED) {
                      continue;
                  }
  ```
  [Line Numbers]: 279 to 282
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java
  [Source Code]:
  ```java
                  if (productCategory == ProductCategory.ADD_ON && !transferAddOn) {
                      continue;
                  }
  ```
  [Applies to]: Validation rule (Priority P1) — subscription module (DefaultSubscriptionBaseTransferApi)
  [Confidence score]: High

- [Kill-BR-0069] - Catalog version consistency validation
  [Description]: Two catalog versions cannot share the same effective date, all versions must have the same catalog name, and a plan that exists in multiple catalog versions must keep the same number and names of phases across those versions. For example: given Two uploaded catalog versions both declare effectiveDate=2026-01-01, or a plan 'gold-monthly' has 2 phases in one version and 3 phases in a later version, when The versioned catalog is validated, then A ValidationError is added ("Catalog effective date ... already exists for a previous version" or "Number of phases for plan ... differs between version ...").
  [Line Numbers]: 134 to 192
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final DefaultVersionedCatalog catalog, final ValidationErrors errors) {
          final Set<Date> effectiveDates = new TreeSet<Date>();

          for (final StaticCatalog c : versions) {
              if (effectiveDates.contains(c.getEffectiveDate())) {
                  errors.add(new ValidationError(String.format("Catalog effective date '%s' already exists for a previous version", c.getEffectiveDate()),
                                                 DefaultVersionedCatalog.class, ""));
              } else {
                  effectiveDates.add(c.getEffectiveDate());
              }
              if (!c.getCatalogName().equals(catalogName)) {
                  errors.add(new ValidationError(String.format("Catalog name '%s' is not consistent across versions ", c.getCatalogName()),
                                                 DefaultVersionedCatalog.class, ""));
              }
              ((StandaloneCatalog) c).validate((StandaloneCatalog) c, errors);
          }

          validateUniformPlanShapeAcrossVersions(errors);

          return errors;
      }

      private void validateUniformPlanShapeAcrossVersions(final ValidationErrors errors) {
          for (int i = 0; i < versions.size(); i++) {
              final StaticCatalog c = versions.get(i);
              for (final Plan plan : ((StandaloneCatalog) c).getPlans()) {

                  for (int j = i + 1; j < versions.size(); j++) {
                      final StaticCatalog next = versions.get(j);
                      final Plan targetPlan = ((StandaloneCatalog) next).getPlansMap().findByName(plan.getName());
                      if (targetPlan != null) {
                          validatePlanShape(plan, targetPlan, errors);
                      }
                      // We don't break if null , targetPlan could be re-defined on a subsequent version
                      // TODO enforce that we can't skip versions?
                  }
              }
          }
      }

      private void validatePlanShape(final Plan plan, final Plan targetPlan, final ValidationErrors errors) {
          if (plan.getAllPhases().length != targetPlan.getAllPhases().length) {
              errors.add(new ValidationError(String.format("Number of phases for plan '%s' differs between version '%s' and '%s'",
                                                           plan.getName(), plan.getCatalog().getEffectiveDate(), targetPlan.getCatalog().getEffectiveDate()),
                                             DefaultVersionedCatalog.class, ""));
              // In this case we don't bother checking each phase -- the code below assumes the # are equal
              return;
          }

          for (int i = 0; i < plan.getAllPhases().length; i++) {
              final PlanPhase cur = plan.getAllPhases()[i];
              final PlanPhase target = targetPlan.getAllPhases()[i];
              if (!cur.getName().equals(target.getName())) {
                 errors.add(new ValidationError(String.format("Phase '%s'for plan '%s' in version '%s' does not exist in version '%s'",
                                                               cur.getName(), plan.getName(), plan.getCatalog().getEffectiveDate(), targetPlan.getCatalog().getEffectiveDate()),
                                                 DefaultVersionedCatalog.class, ""));
              }
          }
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultVersionedCatalog)
  [Confidence score]: High

- [Kill-BR-0070] - Grandfather date cannot precede its own catalog version's effective date
  [Description]: A plan's 'effective date for existing subscriptions' (grandfathering rollout date) must be on or after the catalog version's own effective date; catalogs that set it earlier are rejected as invalid. For example: given A catalog version effective 2026-01-01 defines a plan with effectiveDateForExistingSubscriptions=2025-12-01, when The catalog is validated, then A ValidationError is raised: "Price effective date ... is before catalog effective date ...".
  [Line Numbers]: 286 to 293
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (effectiveDateForExistingSubscriptions != null &&
              catalog.getEffectiveDate().getTime() > effectiveDateForExistingSubscriptions.getTime()) {
              errors.add(new ValidationError(String.format("Price effective date %s is before catalog effective date '%s'",
                                                           effectiveDateForExistingSubscriptions,
                                                           catalog.getEffectiveDate()),
                                             DefaultPlan.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P2) — catalog module (DefaultPlan)
  [Confidence score]: High

- [Kill-BR-0071] - Tenant external key length limit
  [Description]: A tenant's external key cannot be longer than 255 characters. For example: given A createTenant request with an external key of 256 characters, when createTenant is called, then the call is rejected with EXTERNAL_KEY_LIMIT_EXCEEDED before any DB write. Parameters: Max length = 255 (hardcoded).
  [Line Numbers]: 89 to 91
  [Source Code File Name]: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  [Source Code]:
  ```java
          if (null != tenant.getExternalKey() && tenant.getExternalKey().length() > 255) {
              throw new TenantApiException(ErrorCode.EXTERNAL_KEY_LIMIT_EXCEEDED);
          }
  ```
  [Applies to]: Validation rule (Priority P1) — tenant module (DefaultTenantUserApi)
  [Confidence score]: High

- [Kill-BR-0072] - Tenant API key must be unique
  [Description]: A new tenant cannot be created with an API key that is already assigned to an existing tenant. For example: given A tenant already exists with API key 'k1', when createTenant is called again with apiKey 'k1', then the call fails with TENANT_ALREADY_EXISTS; the lookup is non-transactional (relies on a DB unique constraint as backstop) and an IllegalStateException from a 'not found' lookup is deliberately swallowed to mean 'key is free'. Edge cases: Any other RuntimeException during the pre-check lookup is NOT swallowed and propagates, which could incorrectly abort tenant creation on transient errors. Note (extraction review flagged this rule for SME confirmation): Is it intentional that only IllegalStateException-caused lookup failures are treated as 'key available', while any other runtime error during the pre-check aborts creation even though a DB constraint would have caught true duplicates?.
  [Line Numbers]: 93 to 104
  [Source Code File Name]: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  [Source Code]:
  ```java
          try {
              // Not transactional, but there is a db constraint on that column
              if (data.getApiKey() != null && getTenantByApiKey(data.getApiKey()) != null) {
                  throw new TenantApiException(ErrorCode.TENANT_ALREADY_EXISTS, data.getExternalKey());
              }
          } catch (final RuntimeException e) {
              if (e.getCause() instanceof IllegalStateException) {
                  // could happen exemption, stating that the key is not found
              } else {
                  throw e;
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — tenant module (DefaultTenantUserApi)
  [Confidence score]: Medium

- [Kill-BR-0073] - Invoice item adjustment amount must be positive
  [Description]: When manually adjusting an invoice item with a specific amount, that amount must be strictly greater than zero. For example: given A request to adjust invoice item X by amount $0.00 or -$5.00, when insertInvoiceItemAdjustment is called with a non-null amount <= 0, then the call is rejected with INVOICE_ITEM_ADJUSTMENT_AMOUNT_SHOULD_BE_POSITIVE.
  [Line Numbers]: 420 to 422
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
          if (amount != null && amount.compareTo(BigDecimal.ZERO) <= 0) {
              throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_ADJUSTMENT_AMOUNT_SHOULD_BE_POSITIVE, amount);
          }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0074] - Cannot adjust a VOID invoice
  [Description]: An item-level adjustment cannot be applied to an invoice that has been voided. For example: given Invoice INV-1 has status VOID, when insertInvoiceItemAdjustment targets an item on INV-1, then the call fails with INVOICE_VOID_UPDATED.
  [Line Numbers]: 428 to 431
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
                  final DefaultInvoice invoice = getInvoiceInternal(invoiceId, context);
                  if (InvoiceStatus.VOID == invoice.getStatus()) {
                      throw new InvoiceApiException(ErrorCode.INVOICE_VOID_UPDATED, invoice.getId());
                  }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0075] - Adjustment currency must match invoice currency
  [Description]: If a currency is explicitly supplied for an item adjustment, it must match the invoice's own currency. For example: given Invoice INV-1 is in USD, when insertInvoiceItemAdjustment is called with currency=EUR, then the call fails with CURRENCY_INVALID(EUR, USD).
  [Line Numbers]: 433 to 436
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
                  // Check the specified currency matches the one of the existing invoice
                  if (currency != null && invoice.getCurrency() != currency) {
                      throw new InvoiceApiException(ErrorCode.CURRENCY_INVALID, currency, invoice.getCurrency());
                  }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0076] - External charge / credit amount must be non-negative
  [Description]: A new external charge or account credit item cannot have a null or negative amount. For example: given A new EXTERNAL_CHARGE item with amount -$10 or null amount, when insertItems is called (via insertExternalCharges / insertCredits), then the call fails with EXTERNAL_CHARGE_AMOUNT_INVALID (for charges) or CREDIT_AMOUNT_INVALID (for credits). Edge cases: A zero amount ($0.00) is allowed since the check is strictly < 0.
  [Line Numbers]: 562 to 568
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
                      if (inputItem.getAmount() == null || inputItem.getAmount().compareTo(BigDecimal.ZERO) < 0) {
                          if (itemType == InvoiceItemType.EXTERNAL_CHARGE) {
                              throw new InvoiceApiException(ErrorCode.EXTERNAL_CHARGE_AMOUNT_INVALID, inputItem.getAmount());
                          } else if (itemType == InvoiceItemType.CREDIT_ADJ) {
                              throw new InvoiceApiException(ErrorCode.CREDIT_AMOUNT_INVALID, inputItem.getAmount());
                          }
                      }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0077] - External charge / credit currency must match account currency
  [Description]: If a currency is specified on a new external charge or credit item, it must match the account's currency. For example: given Account currency is USD, when An external charge item is inserted with currency=GBP, then the call fails with CURRENCY_INVALID(GBP, USD).
  [Line Numbers]: 570 to 572
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
                      if (inputItem.getCurrency() != null && !inputItem.getCurrency().equals(accountCurrency)) {
                          throw new InvoiceApiException(ErrorCode.CURRENCY_INVALID, inputItem.getCurrency(), accountCurrency);
                      }
  ```
  [Applies to]: Validation rule (Priority P1) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0078] - Cannot add external charge/credit to an already-committed invoice
  [Description]: New external charges or credits can only be attached to an existing invoice while it is still DRAFT; committed invoices reject new items, and voided invoices are also blocked (though via a different, generic error path). For example: given Invoice INV-1 has status COMMITTED, when insertExternalCharges/insertCredits targets INV-1 by invoiceId, then the call fails with INVOICE_ALREADY_COMMITTED; if INV-1 is VOID instead, an IllegalStateException is thrown (tracked as a known gap, see comment referencing killbill/killbill#1501) **Suspected defect:** VOID case throws a raw IllegalStateException instead of a proper InvoiceApiException with a dedicated error code, per the TODO comment at line 594.
  [Line Numbers]: 587 to 600
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
                      } else {
                          if (newAndExistingInvoices.get(invoiceIdForItem) == null) {
                              final DefaultInvoice existingInvoiceForExternalCharge = getInvoiceInternal(invoiceIdForItem, context);
                              switch (existingInvoiceForExternalCharge.getStatus()) {
                                  case COMMITTED:
                                      throw new InvoiceApiException(ErrorCode.INVOICE_ALREADY_COMMITTED, existingInvoiceForExternalCharge.getId());
                                  case VOID:
                                      // TODO Add missing error https://github.com/killbill/killbill/issues/1501
                                      throw new IllegalStateException(String.format("Cannot add credit or external charge for invoice id %s because it is in \" + InvoiceStatus.VOID + \" status\"",
                                                                                    existingInvoiceForExternalCharge.getId()));
                                  case DRAFT:
                                  default:
                                      break;
                              }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0079] - Child-to-parent credit transfer eligibility
  [Description]: A child account's credit balance (CBA) can only be transferred to its parent account if the account actually has a parent and currently holds a positive credit balance. For example: given A child account with no parentAccountId, or with a parent but $0.00 CBA, when transferChildCreditToParent is called, then the call fails with ACCOUNT_DOES_NOT_HAVE_PARENT_ACCOUNT or CHILD_ACCOUNT_MISSING_CREDIT respectively; only a strictly positive CBA (> $0.00) triggers the transfer.
  [Line Numbers]: 710 to 729
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
      public void transferChildCreditToParent(final UUID childAccountId, final CallContext context) throws InvoiceApiException {

          final Account childAccount;
          final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(childAccountId, ObjectType.ACCOUNT, context);
          try {
              childAccount = accountUserApi.getAccountById(childAccountId, internalCallContext);
          } catch (final AccountApiException e) {
              throw new InvoiceApiException(e);
          }

          if (childAccount.getParentAccountId() == null) {
              throw new InvoiceApiException(ErrorCode.ACCOUNT_DOES_NOT_HAVE_PARENT_ACCOUNT, childAccountId);
          }

          final BigDecimal accountCBA = getAccountCBA(childAccountId, context);
          if (accountCBA.compareTo(BigDecimal.ZERO) <= 0) {
              throw new InvoiceApiException(ErrorCode.CHILD_ACCOUNT_MISSING_CREDIT, childAccountId);
          }

          dao.transferChildCreditToParent(childAccount, internalCallContext);
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0080] - Account email record add is idempotent by record id, not by email address
  [Description]: Adding an email to an account is rejected only if a record with that exact same internal ID already exists — the system does not check whether the same email address string is already registered for the account. For example: given Account A already has email 'a@x.com' stored with id U1, when addEmail is called with a brand-new randomly generated id U2 and the same address 'a@x.com', then the insert succeeds and the account now has two email rows with the same address (ACCOUNT_EMAIL_ALREADY_EXISTS only fires on an id collision, which is effectively unreachable with random UUIDs) **Suspected defect:** The uniqueness check is on the generated UUID primary key rather than the email address, so duplicate email addresses per account are not actually prevented at this layer. Note (extraction review flagged this rule for SME confirmation): Is duplicate-email-address prevention meant to be enforced elsewhere (e.g. API layer, DB constraint), or is allowing multiple identical email addresses per account intentional?.
  [Line Numbers]: 329 to 341
  [Source Code File Name]: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  [Source Code]:
  ```java
      @Override
      public void addEmail(final AccountEmailModelDao email, final InternalCallContext context) throws AccountApiException {
          transactionalSqlDao.execute(false, AccountApiException.class, entitySqlDaoWrapperFactory -> {
              final AccountEmailSqlDao transactional = entitySqlDaoWrapperFactory.become(AccountEmailSqlDao.class);

              if (transactional.getById(email.getId().toString(), context) != null) {
                  throw new AccountApiException(ErrorCode.ACCOUNT_EMAIL_ALREADY_EXISTS, email.getId());
              }

              createAndRefresh(transactional, email, context);
              return null;
          });
      }
  ```
  [Applies to]: Validation rule (Priority P2) — account module (DefaultAccountDao)
  [Confidence score]: Medium

- [Kill-BR-0081] - Only one payment method per account may use the external/manual-pay plugin
  [Description]: An account can have at most one payment method backed by the built-in external (manual) payment provider plugin. For example: given Account A already has a payment method using the ExternalPaymentProviderPlugin, when addPaymentMethod is called again requesting the same plugin, then the call fails with PAYMENT_EXTERNAL_PAYMENT_METHOD_ALREADY_EXISTS.
  [Line Numbers]: 155 to 162
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  [Source Code]:
  ```java
                                                                                                          private void validateUniqueExternalPaymentMethod(final UUID accountId, final String pluginName) throws PaymentApiException {
                                                                                                              if (ExternalPaymentProviderPlugin.PLUGIN_NAME.equals(pluginName)) {
                                                                                                                  final List<PaymentMethodModelDao> accountPaymentMethods = paymentDao.getPaymentMethods(context);
                                                                                                                  if (accountPaymentMethods.stream().anyMatch(input -> ExternalPaymentProviderPlugin.PLUGIN_NAME.equals(input.getPluginName()))) {
                                                                                                                      throw new PaymentApiException(ErrorCode.PAYMENT_EXTERNAL_PAYMENT_METHOD_ALREADY_EXISTS, accountId);
                                                                                                                  }
                                                                                                              }
                                                                                                          }
  ```
  [Applies to]: Validation rule (Priority P1) — payment module (PaymentMethodProcessor)
  [Confidence score]: High

- [Kill-BR-0082] - Default payment method must belong to the target account
  [Description]: You cannot set a payment method as an account's default if that payment method actually belongs to a different account. For example: given Payment method PM1 belongs to account B, when setDefaultPaymentMethod is called for account A with paymentMethodId=PM1, then the call fails with PAYMENT_METHOD_DIFFERENT_ACCOUNT_ID.
  [Line Numbers]: 552 to 556
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  [Source Code]:
  ```java
                      final PaymentMethodModelDao paymentMethodModel = getPaymentMethodById(paymentMethodId, false, context);

                      if (!paymentMethodModel.getAccountId().equals(account.getId())) {
                          throw new PaymentApiException(ErrorCode.PAYMENT_METHOD_DIFFERENT_ACCOUNT_ID, paymentMethodId);
                      }
  ```
  [Applies to]: Validation rule (Priority P0) — payment module (PaymentMethodProcessor)
  [Confidence score]: High

- [Kill-BR-0083] - AUTO_PAY_OFF tag cannot be removed without a default payment method on file
  [Description]: You can't turn auto-pay back on (by removing the AUTO_PAY_OFF tag) for an account that has no default payment method configured, since there would be nothing to charge. For example: given Account A has the AUTO_PAY_OFF tag and paymentMethodId is null, when DELETE /accounts/{id}/tags is called including the AUTO_PAY_OFF tag id, then the request is rejected with TAG_CANNOT_BE_REMOVED (400); other tags in the same request are otherwise removable, this check only applies when AUTO_PAY_OFF is among them.
  [Line Numbers]: 1441 to 1455
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java
  [Source Code]:
  ```java
          // Look if there is an AUTO_PAY_OFF for that account and check if the account has a default paymentMethod
          // If not we can't remove the AUTO_PAY_OFF tag
          boolean isTagAutoPayOff = false;
          for (final UUID cur : tagList) {
              if (cur.equals(ControlTagType.AUTO_PAY_OFF.getId())) {
                  isTagAutoPayOff = true;
                  break;
              }
          }
          if (isTagAutoPayOff) {
              final Account account = accountUserApi.getAccountById(accountId, callContext);
              if (account.getPaymentMethodId() == null) {
                  throw new TagApiException(ErrorCode.TAG_CANNOT_BE_REMOVED, ControlTagType.AUTO_PAY_OFF, " the account does not have a default payment method");
              }
          }
  ```
  [Applies to]: Validation rule (Priority P0) — jaxrs module (AccountResource)
  [Confidence score]: High

- [Kill-BR-0084] - Invoice-list query modes are mutually exclusive
  [Description]: When fetching an account's invoices, you cannot combine 'unpaid invoices only' with 'include migration invoices', cannot combine a start-date filter with 'include migration invoices', and cannot combine 'unpaid invoices only' with 'include invoice components'. For example: given A GET invoices request with unpaidInvoicesOnly=true and withMigrationInvoices=true, when getInvoicesForAccount is called, then the request fails fast with an IllegalStateException ('We don't support fetching unpaid invoices incl. migration') before any query executes.
  [Line Numbers]: 711 to 713
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/AccountResource.java
  [Source Code]:
  ```java
          Preconditions.checkState(!unpaidInvoicesOnly || !withMigrationInvoices, "We don't support fetching unpaid invoices incl. migration");
          Preconditions.checkState(startDateStr == null || !withMigrationInvoices, "We don't support fetching migration invoices and specifying a start date");
          Preconditions.checkState(!unpaidInvoicesOnly || !includeInvoiceComponents, "We don't support fetching unpaid invoices with invoice components");
  ```
  [Applies to]: Validation rule (Priority P2) — jaxrs module (AccountResource)
  [Confidence score]: High

- [Kill-BR-0085] - Plan resolution requires product + billing period, with price-list fallback to default
  [Description]: When creating a subscription without specifying an exact plan name, the system requires a product name and billing period; if no price list is specified, it uses the catalog's default price list. For example: given A PlanSpecifier with planName=null, productName='Gold', billingPeriod=MONTHLY, priceListName=null, when createOrFindPlan is called, then the system looks up plan 'Gold' at MONTHLY billing period in the DEFAULT price list (falling back further per DefaultPriceListSet); if productName or billingPeriod is missing, it fails fast with CAT_NULL_PRODUCT_NAME / CAT_NULL_BILLING_PERIOD before any lookup.
  [Line Numbers]: 206 to 227
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java
  [Source Code]:
  ```java
      public Plan createOrFindPlan(final PlanSpecifier spec, final PlanPhasePriceOverridesWithCallContext unused) throws CatalogApiException {
          final Plan result;
          if (spec.getPlanName() != null) {
              result = findPlan(spec.getPlanName());
          } else {
              if (spec.getProductName() == null) {
                  throw new CatalogApiException(ErrorCode.CAT_NULL_PRODUCT_NAME);
              }
              if (spec.getBillingPeriod() == null) {
                  throw new CatalogApiException(ErrorCode.CAT_NULL_BILLING_PERIOD);
              }
              final String inputOrDefaultPricelist = (spec.getPriceListName() == null) ? PriceListSet.DEFAULT_PRICELIST_NAME : spec.getPriceListName();
              final Product product = findProduct(spec.getProductName());
              result = ((DefaultPriceListSet) priceLists).getPlanFrom(product, spec.getBillingPeriod(), inputOrDefaultPricelist);
          }
          if (result == null) {
              throw new CatalogApiException(ErrorCode.CAT_PLAN_NOT_FOUND,
                                            spec.getPlanName() != null ? spec.getPlanName() : "undefined",
                                            spec.getProductName() != null ? spec.getProductName() : "undefined",
                                            spec.getBillingPeriod() != null ? spec.getBillingPeriod() : "undefined",
                                            spec.getPriceListName() != null ? spec.getPriceListName() : "undefined");
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (StandaloneCatalog)
  [Confidence score]: High

- [Kill-BR-0086] - Price-list plan lookup falls back to default list; ambiguous match is rejected
  [Description]: When resolving a plan for a given product/billing period within a named (child) price list, if that price list has no matching plan the system automatically falls back to the default price list; but if more than one matching plan is found, the lookup is rejected as ambiguous rather than picking one arbitrarily. For example: given Product 'Gold', period MONTHLY, priceList 'promo-2024' has no matching plan, when getPlanFrom(product, period, 'promo-2024') is called, then the system retries the lookup against the default price list; if that yields 0 plans, returns null (caller reports CAT_PLAN_NOT_FOUND); if it yields 2+ plans, throws CAT_MULTIPLE_MATCHING_PLANS_FOR_PRICELIST.
  [Line Numbers]: 64 to 83
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java
  [Source Code]:
  ```java
      public Plan getPlanFrom(final Product product, final BillingPeriod period, final String priceListName) throws CatalogApiException {

          Collection<Plan> plans = null;
          final DefaultPriceList pl = findPriceListFrom(priceListName);
          if (pl != null) {
              plans = pl.findPlans(product, period);
          }
          if (plans.size() == 0) {
              plans = defaultPricelist.findPlans(product, period);
          }
          switch (plans.size()) {
              case 0:
                  return null;
              case 1:
                  return plans.iterator().next();
              default:
                  throw new CatalogApiException(ErrorCode.CAT_MULTIPLE_MATCHING_PLANS_FOR_PRICELIST,
                                                priceListName, product.getName(), period);
          }
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPriceListSet)
  [Confidence score]: High

- [Kill-BR-0087] - Control tags may only be applied to their designated object types
  [Description]: Each system control tag (e.g. AUTO_PAY_OFF, MANUAL_PAY) is only allowed on certain entity types; attaching one to an unsupported object type is rejected. For example: given A control tag whose applicable object types are {ACCOUNT} only, when That tag is attached to a BUNDLE instead of an ACCOUNT, then creation fails with an IllegalStateException ('Invalid control tag ... for object type ...') **Suspected defect:** This throws a raw IllegalStateException instead of a TagApiException, per the inline TODO noting a missing TAG_NOT_APPLICABLE error code (lines 210-212).
  [Line Numbers]: 204 to 213
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDao.java
  [Source Code]:
  ```java
      private void validateApplicableObjectTypes(final UUID tagDefinitionId, final ObjectType objectType) {
          final ControlTagType controlTagType = Stream.of(ControlTagType.values())
                  .filter(input -> input.getId().equals(tagDefinitionId))
                  .findFirst().orElse(null);

          if (controlTagType != null && !controlTagType.getApplicableObjectTypes().contains(objectType)) {
              // TODO Add missing ErrorCode.TAG_NOT_APPLICABLE
              // throw new TagApiException(ErrorCode.TAG_NOT_APPLICABLE);
              throw new IllegalStateException(String.format("Invalid control tag '%s' for object type '%s'", controlTagType.name(), objectType));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — util module (DefaultTagDao)
  [Confidence score]: High

- [Kill-BR-0088] - User-defined tag definition name cannot collide with a reserved control tag name
  [Description]: You cannot create a custom tag definition whose name matches one of the built-in system control tag names (e.g. 'AUTO_PAY_OFF'). For example: given A request to create a tag definition named 'AUTO_PAY_OFF', when createTagDefinition is called, then the call fails with TAG_DEFINITION_CONFLICTS_WITH_CONTROL_TAG before any duplicate-name check runs.
  [Line Numbers]: 141 to 144
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java
  [Source Code]:
  ```java
          // Make sure an invoice tag with this name don't already exist
          if (TagModelDaoHelper.isControlTag(definitionName)) {
              throw new TagDefinitionApiException(ErrorCode.TAG_DEFINITION_CONFLICTS_WITH_CONTROL_TAG, definitionName);
          }
  ```
  [Applies to]: Validation rule (Priority P1) — util module (DefaultTagDefinitionDao)
  [Confidence score]: High

- [Kill-BR-0089] - Tag definition names must be globally unique
  [Description]: Two custom tag definitions cannot share the same name. For example: given A tag definition named 'VIP' already exists, when createTagDefinition is called again with name 'VIP', then the call fails with TAG_DEFINITION_ALREADY_EXISTS.
  [Line Numbers]: 150 to 154
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/tag/dao/DefaultTagDefinitionDao.java
  [Source Code]:
  ```java
                  // Make sure the tag definition doesn't exist already
                  final TagDefinitionModelDao existingDefinition = tagDefinitionSqlDao.getByName(definitionName, context);
                  if (existingDefinition != null) {
                      throw new TagDefinitionApiException(ErrorCode.TAG_DEFINITION_ALREADY_EXISTS, definitionName);
                  }
  ```
  [Applies to]: Validation rule (Priority P1) — util module (DefaultTagDefinitionDao)
  [Confidence score]: High

- [Kill-BR-0090] - Simple plan descriptor validation for API-created plans
  [Description]: When creating a new plan through the 'simple plan' API, the request must supply either an existing planId or both a product category and billing period; it must supply a non-negative price amount and a currency; and if the new plan is an ADD_ON, it must list at least one available base product that already exists in the catalog. For example: given A simple plan descriptor for an ADD_ON with availableBaseProducts=[] or amount=-$5, when The plan is added to the catalog via the simple-plan API, then the call fails with CAT_INVALID_SIMPLE_PLAN_DESCRIPTOR (reason BASE_PLAN_PRODUCTS_NOT_EMPTY or INVALID_PRICE); if an add-on references a base product name not found in the catalog, it fails with reason EXISTING_PRODUCTS_NOT_EMPTY. Parameters: Minimum amount = $0.00 (amount < 0 rejected); currency must be in the catalog's supported-currency list (checked separately via isCurrencySupported, lines 285-294).
  [Line Numbers]: 296 to 313
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java
  [Source Code]:
  ```java
      private void validateNewPlanDescriptor(final SimplePlanDescriptor desc) throws CatalogApiException {
          final boolean invalidPlan = desc.getPlanId() == null && (desc.getProductCategory() == null || desc.getBillingPeriod() == null);
          final boolean invalidPrice = (desc.getAmount() == null || desc.getAmount().compareTo(BigDecimal.ZERO) < 0) ||
                                       desc.getCurrency() == null;
          if (invalidPlan || invalidPrice) {
              throw new CatalogApiException(ErrorCode.CAT_INVALID_SIMPLE_PLAN_DESCRIPTOR,  INVALID_PRICE);
          }

          if (desc.getProductCategory() == ProductCategory.ADD_ON) {
              if (desc.getAvailableBaseProducts() == null || desc.getAvailableBaseProducts().isEmpty()) {
                  throw new CatalogApiException(ErrorCode.CAT_INVALID_SIMPLE_PLAN_DESCRIPTOR,  BASE_PLAN_PRODUCTS_NOT_EMPTY);
              }
              for (final String cur : desc.getAvailableBaseProducts()) {
                  if (getExistingProduct(cur) == null) {
                      throw new CatalogApiException(ErrorCode.CAT_INVALID_SIMPLE_PLAN_DESCRIPTOR, EXISTING_PRODUCTS_NOT_EMPTY);
                  }
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (CatalogUpdater)
  [Confidence score]: High

- [Kill-BR-0091] - Plan phase-type composition constraints
  [Description]: A plan's initial (non-final) phases can never be of type EVERGREEN, and a plan's final phase can never be of type TRIAL or DISCOUNT; catalog loading fails validation otherwise. For example: given A plan defines an initial phase with phaseType=EVERGREEN, or a final phase with phaseType=TRIAL, when The catalog is validated at load time, then A ValidationError is recorded ('Initial Phase ... cannot be of type EVERGREEN' / 'Final Phase ... cannot be of type TRIAL'). Edge cases: Final phase of type DISCOUNT is also rejected.
  [Line Numbers]: 304 to 319
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  [Source Code]:
  ```java
          for (final DefaultPlanPhase cur : initialPhases) {
              cur.validate(catalog, errors);
              if (cur.getPhaseType() == PhaseType.EVERGREEN) {
                  errors.add(new ValidationError(String.format("Initial Phase %s of plan %s cannot be of type %s",
                                                               cur.getName(), name, cur.getPhaseType()),
                                                 DefaultPlan.class, ""));
              }
          }

          finalPhase.validate(catalog, errors);

          if (finalPhase.getPhaseType() == PhaseType.TRIAL || finalPhase.getPhaseType() == PhaseType.DISCOUNT) {
              errors.add(new ValidationError(String.format("Final Phase %s of plan %s cannot be of type %s",
                                                           finalPhase.getName(), name, finalPhase.getPhaseType()),
                                             DefaultPlan.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPlan)
  [Confidence score]: High

- [Kill-BR-0092] - Usage catalog section must declare pricing structures matching its billing mode/type
  [Description]: A catalog usage section must define the correct pricing structure for its combination of billing mode and usage type: IN_ADVANCE/CAPACITY usage requires at least one limit, IN_ADVANCE/CONSUMABLE usage requires at least one block, and any IN_ARREAR usage requires at least one tier. For example: given A usage section with billingMode=IN_ADVANCE, usageType=CAPACITY, and zero <limits> defined, when The catalog is validated, then A ValidationError is recorded: 'Usage [IN_ADVANCE CAPACITY] section of phase ... needs to define some limits'. Edge cases: IN_ARREAR usage of either CAPACITY or CONSUMABLE type both require tiers (not blocks/limits directly).
  [Line Numbers]: 212 to 225
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (billingMode == BillingMode.IN_ADVANCE && usageType == UsageType.CAPACITY && limits.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ADVANCE CAPACITY] section of phase %s needs to define some limits",
                                                           phase.toString()), DefaultUsage.class, ""));
          }
          if (billingMode == BillingMode.IN_ADVANCE && usageType == UsageType.CONSUMABLE && blocks.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ADVANCE CONSUMABLE] section of phase %s needs to define some blocks",
                                                           phase.toString()), DefaultUsage.class, ""));
          }

          if (billingMode == BillingMode.IN_ARREAR && tiers.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ARREAR] section of phase %s needs to define some tiers",
                                                           phase.toString()), DefaultUsage.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultUsage)
  [Confidence score]: High

- [Kill-BR-0093] - Role-permission definitions are sanitized and collapsed to group-level wildcards
  [Description]: When defining or updating a role's permission list, each permission string must be in the form 'group' or 'group:value'; a bare '*' grants everything, and granting 'group:*' (or an empty value) for a group collapses/overrides any previously listed specific values for that same group into a single wildcard. For example: given A role definition request with permissions ['invoice:item_adjust', 'invoice:*', 'account:view'], when addRoleDefinition/updateRoleDefinition sanitizes the list, then The stored permission set becomes ['invoice:*', 'account:view'] - the specific 'invoice:item_adjust' entry is dropped because the group-level wildcard for 'invoice' collapses it; a malformed entry with more than one ':' throws SECURITY_INVALID_PERMISSIONS. Parameters: Permission format 'group[:value]', wildcard '*'.
  [Line Numbers]: 236 to 246
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java
  [Source Code]:
  ```java
      @Override
      public void addRoleDefinition(final String role, final List<String> permissions, final CallContext callContext) throws SecurityApiException {
          final List<String> sanitizedPermissions = sanitizePermissions(permissions);
          userDao.addRoleDefinition(role, sanitizedPermissions, callContext.getUserName());
      }

      @Override
      public void updateRoleDefinition(final String role, final List<String> permissions, final CallContext callContext) throws SecurityApiException {
          final List<String> sanitizedPermissions = sanitizePermissions(permissions);
          userDao.updateRoleDefinition(role, sanitizedPermissions, callContext.getUserName());
      }
  ```
  [Line Numbers]: 265 to 304
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java
  [Source Code]:
  ```java
      private List<String> sanitizePermissions(final List<String> permissionsRaw) throws SecurityApiException {
          if (permissionsRaw == null) {
              return Collections.emptyList();
          }
          final Collection<String> permissions = permissionsRaw.stream()
                  .filter(permission -> permission != null && !permission.isEmpty())
                  .collect(Collectors.toUnmodifiableList());

          final Map<String, Set<String>> groupToValues = new HashMap<>();
          for (final String curPerm : permissions) {
              if ("*".equals(curPerm)) {
                  return List.of("*");
              }

              final String[] permissionParts = curPerm.split(":");
              if (permissionParts.length != 1 && permissionParts.length != 2) {
                  throw new SecurityApiException(ErrorCode.SECURITY_INVALID_PERMISSIONS, curPerm);
              }

              Set<String> groupPermissions = groupToValues.get(permissionParts[0]);
              if (groupPermissions == null) {
                  groupPermissions = new HashSet<String>();
                  groupToValues.put(permissionParts[0], groupPermissions);
              }
              if (permissionParts.length == 1 || "*".equals(permissionParts[1]) || Strings.emptyToNull(permissionParts[1]) == null) {
                  groupPermissions.clear();
                  groupPermissions.add("*");
              } else {
                  groupPermissions.add(permissionParts[1]);
              }
          }

          final List<String> expandedPermissions = new ArrayList<>();
          for (final Entry<String, Set<String>> entry : groupToValues.entrySet()) {
              for (final String value : entry.getValue()) {
                  expandedPermissions.add(String.format("%s:%s", entry.getKey(), value));
              }
          }
          return expandedPermissions;
      }
  ```
  [Applies to]: Validation rule (Priority P1) — util module (DefaultSecurityApi)
  [Confidence score]: High

- [Kill-BR-0094] - Credit line-item currency must match account currency (or defaults to it)
  [Description]: When creating a credit or external charge, if the request specifies a currency it must exactly match the account's currency; if omitted, the account currency is silently applied. For example: given An account with currency USD and a credit request specifying currency EUR, when CreditResource.createCredits (or any caller of validateSanitizeAndTranformInputItems) processes the request, then An InvoiceApiException with ErrorCode.CURRENCY_INVALID is thrown before any credit is persisted.
  [Line Numbers]: 617 to 661
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/JaxRsResourceBase.java
  [Source Code]:
  ```java
      protected Iterable<InvoiceItem> validateSanitizeAndTranformInputItems(final Currency accountCurrency, final Iterable<InvoiceItemJson> inputItems) throws InvoiceApiException {
          try {
              return Iterables.toStream(inputItems)
                              .map(input -> {
                                  if (input.getCurrency() != null) {
                                      if (!input.getCurrency().equals(accountCurrency)) {
                                          throw new IllegalArgumentException(input.getCurrency().toString());
                                      }
                                      return input;
                                  } else {
                                      return new InvoiceItemJson(null,
                                                                 input.getInvoiceId(),
                                                                 input.getLinkedInvoiceItemId(),
                                                                 input.getAccountId(),
                                                                 input.getChildAccountId(),
                                                                 input.getBundleId(),
                                                                 input.getSubscriptionId(),
                                                                 input.getProductName(),
                                                                 input.getPlanName(),
                                                                 input.getPhaseName(),
                                                                 input.getUsageName(),
                                                                 input.getPrettyProductName(),
                                                                 input.getPrettyPlanName(),
                                                                 input.getPrettyPhaseName(),
                                                                 input.getPrettyUsageName(),
                                                                 input.getItemType(),
                                                                 input.getDescription(),
                                                                 input.getStartDate(),
                                                                 input.getEndDate(),
                                                                 input.getAmount(),
                                                                 input.getRate(),
                                                                 accountCurrency,
                                                                 input.getQuantity(),
                                                                 input.getItemDetails(),
                                                                 input.getCatalogEffectiveDate(),
                                                                 null,
                                                                 null);
                                  }
                              })
                              .map(InvoiceItemJson::toInvoiceItem)
                              .collect(Collectors.toUnmodifiableList());
          } catch (IllegalArgumentException e) {
              throw new InvoiceApiException(ErrorCode.CURRENCY_INVALID, accountCurrency, e.getMessage());
          }
      }
  ```
  [Applies to]: Validation rule (Priority P0) — jaxrs module (JaxRsResourceBase)
  [Confidence score]: High

- [Kill-BR-0095] - Usage records cannot be recorded past a subscription's entitlement end date
  [Description]: When submitting metered usage for a subscription, the latest usage record date in the batch cannot be after the subscription's entitlement effective end date (if the subscription has already ended). For example: given A subscription with entitlement effective end date 2026-06-01 and a usage submission containing a record dated 2026-06-15, when recordUsage is called, then The API returns HTTP 400 Bad Request and no usage is recorded.
  [Line Numbers]: 119 to 141
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/UsageResource.java
  [Source Code]:
  ```java
          verifyNonNullOrEmpty(json, "SubscriptionUsageRecordJson body should be specified");
          verifyNonNullOrEmpty(json.getSubscriptionId(), "SubscriptionUsageRecordJson subscriptionId needs to be set",
                               json.getUnitUsageRecords(), "SubscriptionUsageRecordJson unitUsageRecords needs to be set");
          Preconditions.checkArgument(!json.getUnitUsageRecords().isEmpty(), "json.getUnitUsageRecords() is empty");

          for (final UnitUsageRecordJson unitUsageRecordJson : json.getUnitUsageRecords()) {
              verifyNonNullOrEmpty(unitUsageRecordJson.getUnitType(), "UnitUsageRecordJson unitType need to be set");
              Preconditions.checkArgument(Iterables.size(unitUsageRecordJson.getUsageRecords()) > 0,
                                          "UnitUsageRecordJson usageRecords must have at least one element.");
              for (final UsageRecordJson usageRecordJson : unitUsageRecordJson.getUsageRecords()) {
                  verifyNonNull(usageRecordJson.getAmount(), "UsageRecordJson amount needs to be set");
                  verifyNonNull(usageRecordJson.getRecordDate(), "UsageRecordJson recordDate needs to be set");
              }
          }
          final CallContext callContextNoAccount = context.createCallContextNoAccountId(createdBy, reason, comment, request);
          // Verify subscription exists..
          final Entitlement entitlement = entitlementApi.getEntitlementForId(json.getSubscriptionId(), false, callContextNoAccount);
          if (entitlement.getEffectiveEndDate() != null) {
              final DateTime highestRecordDate = getHighestRecordDate(json.getUnitUsageRecords());
              if (entitlement.getEffectiveEndDate().compareTo(highestRecordDate) < 0) {
                  return Response.status(Status.BAD_REQUEST).build();
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — jaxrs module (UsageResource)
  [Confidence score]: High

- [Kill-BR-0096] - Plan-phase price override must specify at least one price component
  [Description]: When overriding a plan phase's price at subscription-creation/change time, the override must include a fixed price, a recurring price, or at least one usage price - an empty override is rejected. For example: given A PhasePriceJson override with fixedPrice=null, recurringPrice=null, and an empty usagePrices list, when buildPlanPhasePriceOverrides processes the override list, then An IllegalArgumentException ('At least one fixed price, one recurring price or one usage price must be overridden') is thrown, blocking the subscription operation.
  [Line Numbers]: 89 to 97
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/SubscriptionResourceHelpers.java
  [Source Code]:
  ```java
      public static List<PlanPhasePriceOverride> buildPlanPhasePriceOverrides(final Iterable<PhasePriceJson> priceOverrides,
                                                                              final Currency currency,
                                                                              final PlanPhaseSpecifier planPhaseSpecifier) {
          final List<PlanPhasePriceOverride> overrides = new LinkedList<>();
          if (priceOverrides != null) {
              for (final PhasePriceJson input : priceOverrides) {
                  Preconditions.checkNotNull(input);
                  Preconditions.checkArgument(input.getFixedPrice() != null || input.getRecurringPrice() != null ||  (input.getUsagePrices() != null && !input.getUsagePrices().isEmpty()),
                                              "At least one fixed price, one recurring price or one usage price must be overridden");
  ```
  [Applies to]: Validation rule (Priority P1) — jaxrs module (SubscriptionResourceHelpers)
  [Confidence score]: High

- [Kill-BR-0097] - Dry-run subscription-event spec requires mutually-consistent fields per action type
  [Description]: For a dry-run invoice request simulating a subscription START_BILLING or CHANGE event: either a planName is given alone (productName/billingPeriod/productCategory must then be null), or productName+billingPeriod+productCategory are given together (and if the category is ADD_ON, a bundleId is also required); for CHANGE or STOP_BILLING dry-runs, both subscriptionId and bundleId are required. For example: given A dry-run request with dryRunAction=CHANGE, planName='premium-monthly', and productName='premium' both set, when triggerDryRunInvoiceGeneration validates the DryRunArguments, then An IllegalArgumentException ('DryRun subscription productName should not be set when planName is specified') is thrown and no dry-run invoice is generated.
  [Line Numbers]: 469 to 488
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java
  [Source Code]:
  ```java
          if (dryRunSubscriptionSpec != null && dryRunSubscriptionSpec.getDryRunAction() != null) {
              if (SubscriptionEventType.START_BILLING.equals(dryRunSubscriptionSpec.getDryRunAction()) || SubscriptionEventType.CHANGE.equals(dryRunSubscriptionSpec.getDryRunAction())) {
                  if (dryRunSubscriptionSpec.getPlanName() == null) {
                      verifyNonNullOrEmpty(dryRunSubscriptionSpec.getProductName(), "DryRun subscription product category should be specified when no planName is specified");
                      verifyNonNullOrEmpty(dryRunSubscriptionSpec.getBillingPeriod(), "DryRun subscription billingPeriod should be specified when no planName is specified");
                      verifyNonNullOrEmpty(dryRunSubscriptionSpec.getProductCategory(), "DryRun subscription product category should be specified when no planName is specified");
                      if (dryRunSubscriptionSpec.getProductCategory().equals(ProductCategory.ADD_ON)) {
                          verifyNonNullOrEmpty(dryRunSubscriptionSpec.getBundleId(), "DryRun bundleID should be specified when product category is ADD_ON");
                      }
                  } else {
                      Preconditions.checkArgument(dryRunSubscriptionSpec.getProductName() == null, "DryRun subscription productName should not be set when planName is specified");
                      Preconditions.checkArgument(dryRunSubscriptionSpec.getBillingPeriod() == null, "DryRun subscription billing period should not be set when planName is specified");
                      Preconditions.checkArgument(dryRunSubscriptionSpec.getProductCategory() == null, "DryRun subscription product category should not be set when planName is specified");
                  }
              }
              if (SubscriptionEventType.CHANGE.equals(dryRunSubscriptionSpec.getDryRunAction()) || SubscriptionEventType.STOP_BILLING.equals(dryRunSubscriptionSpec.getDryRunAction())) {
                  verifyNonNullOrEmpty(dryRunSubscriptionSpec.getSubscriptionId(), "DryRun subscriptionID should be specified");
                  verifyNonNullOrEmpty(dryRunSubscriptionSpec.getBundleId(), "DryRun bundleID should be specified");
              }
          }
  ```
  [Applies to]: Validation rule (Priority P1) — jaxrs module (InvoiceResource)
  [Confidence score]: High

- [Kill-BR-0098] - External invoice payments cannot specify an explicit payment method
  [Description]: When triggering a payment against an invoice with externalPayment=true, the request must not include a paymentMethodId (external/manual payments bypass the payment-method system entirely); for normal payments, the explicit paymentMethodId is used if given, otherwise the account's default payment method is used. For example: given A createInstantPayment request with externalPayment=true and paymentMethodId set to some UUID, when createInstantPayment validates the request, then An IllegalArgumentException is thrown before any payment is attempted.
  [Line Numbers]: 716 to 739
  [Source Code File Name]: jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/InvoiceResource.java
  [Source Code]:
  ```java
      public Response createInstantPayment(@PathParam("invoiceId") final UUID invoiceId,
                                           final InvoicePaymentJson payment,
                                           @QueryParam(QUERY_PAYMENT_EXTERNAL) @DefaultValue("false") final Boolean externalPayment,
                                           @QueryParam(QUERY_PAYMENT_CONTROL_PLUGIN_NAME) final List<String> paymentControlPluginNames,
                                           @QueryParam(QUERY_PLUGIN_PROPERTY) final List<String> pluginPropertiesString,
                                           @HeaderParam(HDR_CREATED_BY) final String createdBy,
                                           @HeaderParam(HDR_REASON) final String reason,
                                           @HeaderParam(HDR_COMMENT) final String comment,
                                           @jakarta.ws.rs.core.Context final HttpServletRequest request,
                                           @jakarta.ws.rs.core.Context final UriInfo uriInfo) throws AccountApiException, PaymentApiException {
          verifyNonNullOrEmpty(payment, "InvoicePaymentJson body should be specified");
          verifyNonNullOrEmpty(payment.getAccountId(), "InvoicePaymentJson accountId needs to be set");
          Preconditions.checkArgument(!externalPayment || payment.getPaymentMethodId() == null, "InvoicePaymentJson should not contain a paymentMethodId when this is an external payment");

          final Iterable<PluginProperty> pluginProperties = extractPluginProperties(pluginPropertiesString);
          final CallContext callContext = context.createCallContextNoAccountId(createdBy, reason, comment, request);

          final Account account = accountUserApi.getAccountById(payment.getAccountId(), callContext);
          final UUID paymentMethodId = externalPayment ? null :
                                       (payment.getPaymentMethodId() != null ? payment.getPaymentMethodId() : account.getPaymentMethodId());

          final PaymentOptions paymentOptions = createControlPluginApiPaymentOptions(externalPayment, paymentControlPluginNames);
          final InvoicePayment result = createPurchaseForInvoice(account, invoiceId, payment.getPurchasedAmount(), paymentMethodId,
                                                                 payment.getPaymentExternalKey(), null, pluginProperties, paymentOptions, callContext);
  ```
  [Applies to]: Validation rule (Priority P1) — jaxrs module (InvoiceResource)
  [Confidence score]: High

- [Kill-BR-0099] - Catalog usage sections must define pricing structures appropriate to their billing mode
  [Description]: In the product catalog, an IN_ADVANCE CAPACITY usage section must define at least one limit, an IN_ADVANCE CONSUMABLE usage section must define at least one block, and any IN_ARREAR usage section must define at least one tier - otherwise the catalog fails validation and cannot be loaded. For example: given A catalog XML with a phase's usage element: billingMode=IN_ADVANCE, usageType=CONSUMABLE, and no blocks defined, when The catalog is validated on load, then A ValidationError 'Usage [IN_ADVANCE CONSUMABLE] section of phase <phase> needs to define some blocks' is added, and the catalog is rejected. Parameters: BillingMode {IN_ADVANCE, IN_ARREAR}, UsageType {CAPACITY, CONSUMABLE}. Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (The code at DefaultUsage.java:212-229 is faithfully described: it's a catalog-load-time XML validation that rejects a Usage section if (IN_ADVANCE, CAPACITY) has zero limits, (IN_ADVANCE, CONSUMABLE) has zero blocks, or (IN_ARREAR, any type) has zero tiers, ap.
  [Line Numbers]: 212 to 229
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (billingMode == BillingMode.IN_ADVANCE && usageType == UsageType.CAPACITY && limits.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ADVANCE CAPACITY] section of phase %s needs to define some limits",
                                                           phase.toString()), DefaultUsage.class, ""));
          }
          if (billingMode == BillingMode.IN_ADVANCE && usageType == UsageType.CONSUMABLE && blocks.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ADVANCE CONSUMABLE] section of phase %s needs to define some blocks",
                                                           phase.toString()), DefaultUsage.class, ""));
          }

          if (billingMode == BillingMode.IN_ARREAR && tiers.length == 0) {
              errors.add(new ValidationError(String.format("Usage [IN_ARREAR] section of phase %s needs to define some tiers",
                                                           phase.toString()), DefaultUsage.class, ""));
          }
          validateCollection(catalog, errors, limits);
          validateCollection(catalog, errors, tiers);
          return errors;
      }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultUsage)
  [Confidence score]: Medium

- [Kill-BR-0100] - Plan-change and plan-cancellation catalog rules require a catch-all default case
  [Description]: The catalog's plan-change and plan-cancellation rule tables must each include one 'default' rule (a case with every matching field left unset, i.e. it matches everything) so there is always a fallback policy; duplicate rule entries (identical match criteria) are also rejected. For example: given A catalog whose changePolicy section only defines rules scoped to specific products, with no catch-all changePolicyCase, when The catalog is validated on load, then A ValidationError 'Missing default rule case for plan change' is added and the catalog fails to load; an identical duplicate rule entry instead produces 'Duplicate rule for change plan <rule>'. Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (The rule text is faithful to the cited code: DefaultPlanRules.validate() (catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java:188-236) does exactly what's described — for the changePolicy table (lines 194-215) it walks every DefaultC.
  [Line Numbers]: 188 to 236
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          //
          // Validate that there is a default policy for change AND cancel rules and check unicity of rules
          //
          final HashSet<DefaultCaseChangePlanPolicy> caseChangePlanPoliciesSet = new HashSet<DefaultCaseChangePlanPolicy>();
          boolean foundDefaultCase = false;
          for (final DefaultCaseChangePlanPolicy cur : changeCase) {
              if (caseChangePlanPoliciesSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for change plan %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  caseChangePlanPoliciesSet.add(cur);
              }
              if (cur.getPhaseType() == null &&
                  cur.getFromProduct() == null &&
                  cur.getFromProductCategory() == null &&
                  cur.getFromBillingPeriod() == null &&
                  cur.getFromPriceList() == null &&
                  cur.getToProduct() == null &&
                  cur.getToProductCategory() == null &&
                  cur.getToBillingPeriod() == null &&
                  cur.getToPriceList() == null) {
                  foundDefaultCase = true;
              }
              cur.validate(catalog, errors);
          }
          if (!foundDefaultCase) {
              errors.add(new ValidationError("Missing default rule case for plan change", DefaultPlanRules.class, ""));
          }

          final HashSet<DefaultCaseCancelPolicy> defaultCaseCancelPoliciesSet = new HashSet<DefaultCaseCancelPolicy>();
          foundDefaultCase = false;
          for (final DefaultCaseCancelPolicy cur : cancelCase) {
              if (defaultCaseCancelPoliciesSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for plan cancellation %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  defaultCaseCancelPoliciesSet.add(cur);
              }
              if (cur.getPhaseType() == null &&
                  cur.getProduct() == null &&
                  cur.getProductCategory() == null &&
                  cur.getBillingPeriod() == null &&
                  cur.getPriceList() == null) {
                  foundDefaultCase = true;
              }
              cur.validate(catalog, errors);
          }
          if (!foundDefaultCase) {
              errors.add(new ValidationError("Missing default rule case for plan cancellation", DefaultPlanRules.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPlanRules)
  [Confidence score]: Medium

- [Kill-BR-0101] - Plan phase ordering constraints: no EVERGREEN initial phase, no TRIAL/DISCOUNT final phase
  [Description]: A catalog plan's initial phases (before the last one) cannot be of type EVERGREEN, and the plan's final phase cannot be of type TRIAL or DISCOUNT - a plan must end in a phase intended to run indefinitely (EVERGREEN or FIXEDTERM), not a temporary one. For example: given A plan definition where the only phase listed is a TRIAL phase acting as the final phase, when The catalog is validated on load, then A ValidationError 'Final Phase <name> of plan <plan> cannot be of type TRIAL' is added and the catalog fails to load. Parameters: PhaseType {TRIAL, DISCOUNT, EVERGREEN, FIXEDTERM}.
  [Line Numbers]: 286 to 319
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (effectiveDateForExistingSubscriptions != null &&
              catalog.getEffectiveDate().getTime() > effectiveDateForExistingSubscriptions.getTime()) {
              errors.add(new ValidationError(String.format("Price effective date %s is before catalog effective date '%s'",
                                                           effectiveDateForExistingSubscriptions,
                                                           catalog.getEffectiveDate()),
                                             DefaultPlan.class, ""));
          }

          // Pure usage based plans would not have a recurringBillingMode
          if (!BillingPeriod.NO_BILLING_PERIOD.equals(getRecurringBillingPeriod()) && recurringBillingMode == null) {
              errors.add(new ValidationError(String.format("Invalid recurring billingMode for plan '%s'", name), DefaultPlan.class, ""));
          }

          if (product == null) {
              errors.add(new ValidationError(String.format("Invalid product for plan '%s'", name), DefaultPlan.class, ""));
          }

          for (final DefaultPlanPhase cur : initialPhases) {
              cur.validate(catalog, errors);
              if (cur.getPhaseType() == PhaseType.EVERGREEN) {
                  errors.add(new ValidationError(String.format("Initial Phase %s of plan %s cannot be of type %s",
                                                               cur.getName(), name, cur.getPhaseType()),
                                                 DefaultPlan.class, ""));
              }
          }

          finalPhase.validate(catalog, errors);

          if (finalPhase.getPhaseType() == PhaseType.TRIAL || finalPhase.getPhaseType() == PhaseType.DISCOUNT) {
              errors.add(new ValidationError(String.format("Final Phase %s of plan %s cannot be of type %s",
                                                           finalPhase.getName(), name, finalPhase.getPhaseType()),
                                             DefaultPlan.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPlan)
  [Confidence score]: High

- [Kill-BR-0102] - Base-entitlement bundle creation requires at least one base entitlement specifier
  [Description]: When creating base entitlements with add-ons, the request must include at least one base-entitlement specifier; a null or empty list is rejected before any subscription is created. For example: given A createBaseEntitlementsWithAddOns call with an empty specifier list, when getFirstBaseEntitlementWithAddOnsSpecifier is invoked internally, then A SubscriptionBaseApiException with ErrorCode.SUB_CREATE_INVALID_ENTITLEMENT_SPECIFIER ('empty base entitlement specifier') is thrown.
  [Line Numbers]: 453 to 466
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java
  [Source Code]:
  ```java
      private DefaultBaseEntitlementWithAddOnsSpecifier getFirstBaseEntitlementWithAddOnsSpecifier(final Iterable<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifiers) throws SubscriptionBaseApiException {
          if (baseEntitlementWithAddOnsSpecifiers == null) {
              log.warn("getFirstBaseEntitlementWithAddOnsSpecifier: baseEntitlementWithAddOnsSpecifiers is null");
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_INVALID_ENTITLEMENT_SPECIFIER, "no base entitlement specifier");
          }

          final Iterator<BaseEntitlementWithAddOnsSpecifier> iterator = baseEntitlementWithAddOnsSpecifiers.iterator();
          if (!iterator.hasNext()) {
              log.warn("getFirstBaseEntitlementWithAddOnsSpecifier: baseEntitlementWithAddOnsSpecifiers is empty");
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_INVALID_ENTITLEMENT_SPECIFIER, "empty base entitlement specifier");
          }

          return (DefaultBaseEntitlementWithAddOnsSpecifier) iterator.next();
      }
  ```
  [Applies to]: Validation rule (Priority P1) — entitlement module (DefaultEntitlementApi)
  [Confidence score]: High

- [Kill-BR-0103] - Chargeback amount bounded by remaining paid amount
  [Description]: A chargeback must be for a positive amount and cannot exceed what remains paid (original payment minus prior refunds/chargebacks) on that invoice payment; currency must match the original invoice payment currency. For example: given An invoice payment of $100 in USD with $20 already refunded/charged back (remaining $80), when A chargeback for $90 USD is posted, then the call is rejected with CHARGE_BACK_AMOUNT_TOO_HIGH (requested 90 > remaining 80); a chargeback for <=0 is rejected with CHARGE_BACK_AMOUNT_IS_NEGATIVE, and a currency mismatch is rejected via a Precondition check. Edge cases: amount == null means 'charge back everything remaining'.
  [Line Numbers]: 933 to 954
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
              // We expect the code to correctly pass the account currency -- the payment code, more generic accept chargeBack in different currencies,
              // but this is only for direct payment (no invoice)
              Preconditions.checkArgument(invoicePayment.getCurrency() == currency, String.format("Invoice payment currency %s doesn't match chargeback currency %s", invoicePayment.getCurrency(), currency));

              final UUID invoicePaymentId = invoicePayment.getId();
              final BigDecimal maxChargedBackAmount = invoiceDaoHelper.getRemainingAmountPaidFromTransaction(invoicePaymentId, entitySqlDaoWrapperFactory, context);
              final BigDecimal requestedChargedBackAmount = (amount == null) ? maxChargedBackAmount : amount;
              if (requestedChargedBackAmount.compareTo(BigDecimal.ZERO) <= 0) {
                  throw new InvoiceApiException(ErrorCode.CHARGE_BACK_AMOUNT_IS_NEGATIVE);
              }
              if (requestedChargedBackAmount.compareTo(maxChargedBackAmount) > 0) {
                  throw new InvoiceApiException(ErrorCode.CHARGE_BACK_AMOUNT_TOO_HIGH, requestedChargedBackAmount, maxChargedBackAmount);
              }

              final InvoicePaymentModelDao payment = entitySqlDaoWrapperFactory.become(InvoicePaymentSqlDao.class).getById(invoicePaymentId.toString(), context);
              if (payment == null) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_PAYMENT_NOT_FOUND, invoicePaymentId.toString());
              }
              final InvoicePaymentModelDao chargeBack = new InvoicePaymentModelDao(UUIDs.randomUUID(), context.getCreatedDate(), InvoicePaymentType.CHARGED_BACK,
                                                                                   payment.getInvoiceId(), payment.getPaymentId(), context.getCreatedDate(),
                                                                                   requestedChargedBackAmount.negate(), payment.getCurrency(), payment.getProcessedCurrency(),
                                                                                   chargebackTransactionExternalKey, payment.getId(), InvoicePaymentStatus.SUCCESS);
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0104] - Refund amount must not exceed original payment and must match specified item adjustments
  [Description]: A refund cannot be for more than the original payment amount, and if specific invoice items are being adjusted as part of the refund, their total must not exceed the requested refund amount. For example: given An original payment of $60.00 and a refund request specifying item adjustments totaling $70.00, when computePositiveRefundAmount is evaluated, then REFUND_AMOUNT_DONT_MATCH_ITEMS_TO_ADJUST is thrown because item total ($70) exceeds requested amount; a requested refund of $61 against a $60 payment throws REFUND_AMOUNT_TOO_HIGH. Edge cases: Comment in code admits check doesn't account for prior refunds and relies on the payment layer for that.
  [Line Numbers]: 155 to 175
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceDaoHelper.java
  [Source Code]:
  ```java
      public BigDecimal computePositiveRefundAmount(final InvoicePaymentModelDao payment, final BigDecimal requestedRefundAmount, final Map<UUID, BigDecimal> invoiceItemIdsWithAmounts) throws InvoiceApiException {
          final BigDecimal maxRefundAmount = payment.getAmount() == null ? BigDecimal.ZERO : payment.getAmount();
          final BigDecimal requestedPositiveAmount = requestedRefundAmount == null ? maxRefundAmount : requestedRefundAmount;
          // This check is good but not enough, we need to also take into account previous refunds
          // (But that should have been checked in the payment call already)
          if (requestedPositiveAmount.compareTo(maxRefundAmount) > 0) {
              throw new InvoiceApiException(ErrorCode.REFUND_AMOUNT_TOO_HIGH, requestedPositiveAmount, maxRefundAmount);
          }

          // Verify if the requested amount matches the invoice items to adjust, if specified
          BigDecimal amountFromItems = BigDecimal.ZERO;
          for (final BigDecimal itemAmount : invoiceItemIdsWithAmounts.values()) {
              amountFromItems = amountFromItems.add(itemAmount);
          }

          // Sanity check: if some items were specified, then the sum should be equal to specified refund amount, if specified
          if (amountFromItems.compareTo(BigDecimal.ZERO) != 0 && requestedPositiveAmount.compareTo(amountFromItems) < 0) {
              throw new InvoiceApiException(ErrorCode.REFUND_AMOUNT_DONT_MATCH_ITEMS_TO_ADJUST, requestedPositiveAmount, amountFromItems);
          }
          return requestedPositiveAmount;
      }
  ```
  [Applies to]: Validation rule (Priority P0) — invoice module (InvoiceDaoHelper)
  [Confidence score]: High

- [Kill-BR-0105] - Deleting a used-credit (CBA) item requires an active, non-migrated, COMMITTED invoice
  [Description]: You can only delete a CBA adjustment item from an invoice that belongs to the given account, is not a migrated (legacy-imported) invoice, and is in COMMITTED status; otherwise the operation is rejected as if the invoice doesn't exist. For example: given A migrated invoice with a CBA item, or a DRAFT invoice with a CBA item, when deleteCBA is called, then INVOICE_NOT_FOUND is thrown in both cases.
  [Line Numbers]: 1172 to 1196
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
      @Override
      public void deleteCBA(final UUID accountId, final UUID invoiceId, final UUID invoiceItemId, final InternalCallContext context) throws InvoiceApiException {
          final List<CustomField> invoiceCustomFields = getInvoiceCustomFields(context);
          final List<Tag> invoicesTags = getInvoicesTags(context);
          final Set<UUID> invoiceIds = new HashSet<>();

          transactionalSqlDao.execute(false, InvoiceApiException.class, entitySqlDaoWrapperFactory -> {
              final InvoiceSqlDao invoiceSqlDao = entitySqlDaoWrapperFactory.become(InvoiceSqlDao.class);

              // Retrieve the invoice and make sure it belongs to the right account
              final InvoiceModelDao invoice = invoiceSqlDao.getById(invoiceId.toString(), context);
              if (invoice == null ||
                  !invoice.getAccountId().equals(accountId) ||
                  invoice.isMigrated() ||
                  invoice.getStatus() != InvoiceStatus.COMMITTED) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_NOT_FOUND, invoiceId);
              }
              invoiceDaoHelper.populateChildren(invoice, invoiceCustomFields, invoicesTags, false, entitySqlDaoWrapperFactory, context);

              // Retrieve the invoice item and make sure it belongs to the right invoice
              final InvoiceItemSqlDao invoiceItemSqlDao = entitySqlDaoWrapperFactory.become(InvoiceItemSqlDao.class);
              final InvoiceItemModelDao cbaItem = invoiceItemSqlDao.getById(invoiceItemId.toString(), context);
              if (cbaItem == null || !cbaItem.getInvoiceId().equals(invoice.getId())) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_NOT_FOUND, invoiceItemId);
              }
  ```
  [Applies to]: Validation rule (Priority P1) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0106] - Catalog validation requires an explicit wildcard/default rule for change and cancel policies
  [Description]: A catalog XML is rejected at load time unless it defines at least one change-plan-policy rule and one cancel-policy rule with every matching field left blank (i.e. a true catch-all default); duplicate identical rule cases are also flagged as validation errors. For example: given A catalog XML whose changePolicy cases all specify a product or category (no catch-all case), when The catalog is loaded/validated, then validation fails with 'Missing default rule case for plan change'. Edge cases: Same requirement independently enforced for cancel policy ('Missing default rule case for plan cancellation'); Duplicate rule cases (by equals()) across change/cancel/alignment/price-list rule sets are also validation errors.
  [Line Numbers]: 188 to 236
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          //
          // Validate that there is a default policy for change AND cancel rules and check unicity of rules
          //
          final HashSet<DefaultCaseChangePlanPolicy> caseChangePlanPoliciesSet = new HashSet<DefaultCaseChangePlanPolicy>();
          boolean foundDefaultCase = false;
          for (final DefaultCaseChangePlanPolicy cur : changeCase) {
              if (caseChangePlanPoliciesSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for change plan %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  caseChangePlanPoliciesSet.add(cur);
              }
              if (cur.getPhaseType() == null &&
                  cur.getFromProduct() == null &&
                  cur.getFromProductCategory() == null &&
                  cur.getFromBillingPeriod() == null &&
                  cur.getFromPriceList() == null &&
                  cur.getToProduct() == null &&
                  cur.getToProductCategory() == null &&
                  cur.getToBillingPeriod() == null &&
                  cur.getToPriceList() == null) {
                  foundDefaultCase = true;
              }
              cur.validate(catalog, errors);
          }
          if (!foundDefaultCase) {
              errors.add(new ValidationError("Missing default rule case for plan change", DefaultPlanRules.class, ""));
          }

          final HashSet<DefaultCaseCancelPolicy> defaultCaseCancelPoliciesSet = new HashSet<DefaultCaseCancelPolicy>();
          foundDefaultCase = false;
          for (final DefaultCaseCancelPolicy cur : cancelCase) {
              if (defaultCaseCancelPoliciesSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for plan cancellation %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  defaultCaseCancelPoliciesSet.add(cur);
              }
              if (cur.getPhaseType() == null &&
                  cur.getProduct() == null &&
                  cur.getProductCategory() == null &&
                  cur.getBillingPeriod() == null &&
                  cur.getPriceList() == null) {
                  foundDefaultCase = true;
              }
              cur.validate(catalog, errors);
          }
          if (!foundDefaultCase) {
              errors.add(new ValidationError("Missing default rule case for plan cancellation", DefaultPlanRules.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P1) — catalog module (DefaultPlanRules)
  [Confidence score]: High

- [Kill-BR-0107] - START_OF_TERM is not a legal default cancellation policy in the base catalog
  [Description]: A catalog-defined cancel-policy rule cannot use START_OF_TERM as its billing action policy; that option is only usable via an explicit per-call override at cancellation time, not as a catalog default. For example: given A catalog cancelPolicy case with policy=START_OF_TERM, when The catalog is validated, then a validation error 'Default catalog START_OF_TERM has not been implemented...' is recorded.
  [Line Numbers]: 51 to 55
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseCancelPolicy.java
  [Source Code]:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          if (policy == BillingActionPolicy.START_OF_TERM) {
              errors.add(new ValidationError("Default catalog START_OF_TERM has not been implemented, such policy can be used during cancellation by overriding policy",
                                             DefaultCaseCancelPolicy.class, ""));
          }
  ```
  [Applies to]: Validation rule (Priority P2) — catalog module (DefaultCaseCancelPolicy)
  [Confidence score]: High

- [Kill-BR-0108] - Security user, role, and role-permission uniqueness on create
  [Description]: A new admin/API user cannot be created with a username that already exists, and a new role definition cannot be created with a name that already exists; operations that act on a user (fetch roles, update password, update roles, invalidate) first verify the username exists and fail otherwise. For example: given A user 'admin' already exists, when insertUser('admin', ...) is called again, then SECURITY_USER_ALREADY_EXISTS is thrown and no duplicate row is created; similarly addRoleDefinition for an existing role name throws SECURITY_ROLE_ALREADY_EXISTS. Edge cases: Password hashed with a per-user random salt using the configured Shiro hash-iteration count before checking uniqueness.
  [Line Numbers]: 57 to 64
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java
  [Source Code]:
  ```java
       * Validate if username exist.
       * @param username to validate
       * @param usersSqlDao usersSqlDao instance
       * @throws SecurityApiException if {@link UsersSqlDao#getByUsername(String)} return null
       */
      private void validateUser(final String username, final UsersSqlDao usersSqlDao) throws SecurityApiException {
          final UserModelDao userModelDao = usersSqlDao.getByUsername(username);
          if (userModelDao == null) {
  ```
  [Line Numbers]: 72 to 89
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java
  [Source Code]:
  ```java
          final String hashedPasswordBase64 = new SimpleHash(KillbillCredentialsMatcher.HASH_ALGORITHM_NAME,
                                                             password, salt.toBase64(), securityConfig.getShiroNbHashIterations()).toBase64();

          final DateTime createdDate = clock.getUTCNow();
          inTransactionWithExceptionHandling(new TransactionCallback<Void>() {
              @Override
              public Void inTransaction(final Handle handle, final TransactionStatus status) throws Exception {
                  final UserRolesSqlDao userRolesSqlDao = handle.attach(UserRolesSqlDao.class);
                  for (final String role : roles) {
                      userRolesSqlDao.create(new UserRolesModelDao(username, role, createdDate, createdBy));
                  }

                  final UsersSqlDao usersSqlDao = handle.attach(UsersSqlDao.class);
                  final UserModelDao userModelDao = usersSqlDao.getByUsername(username);
                  if (userModelDao != null) {
                      throw new SecurityApiException(ErrorCode.SECURITY_USER_ALREADY_EXISTS, username);
                  }
                  usersSqlDao.create(new UserModelDao(username, hashedPasswordBase64, salt.toBase64(), createdDate, createdBy));
  ```
  [Line Numbers]: 105 to 113
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java
  [Source Code]:
  ```java

      @Override
      public void addRoleDefinition(final String role, final List<String> permissions, final String createdBy)  throws SecurityApiException {
          final DateTime createdDate = clock.getUTCNow();
          inTransactionWithExceptionHandling((handle, status) -> {
              final RolesPermissionsSqlDao rolesPermissionsSqlDao = handle.attach(RolesPermissionsSqlDao.class);
              final List<RolesPermissionsModelDao> existingRole = rolesPermissionsSqlDao.getByRoleName(role);
              if (!existingRole.isEmpty()) {
                  throw new SecurityApiException(ErrorCode.SECURITY_ROLE_ALREADY_EXISTS, role);
  ```
  [Applies to]: Validation rule (Priority P1) — util module (DefaultUserDao)
  [Confidence score]: High

- [Kill-BR-0109] - Overdue state trigger conditions (all must hold)
  [Description]: An account moves into (or stays in) a given overdue state only when every configured trigger for that state is simultaneously true: minimum number of unpaid invoices, minimum total unpaid balance, minimum elapsed time since the oldest unpaid invoice, the payment-decline reason matching a configured list, and account control-tag inclusion/exclusion rules. For example: given an overdue state configured with numberOfUnpaidInvoicesEqualsOrExceeds=2, totalUnpaidInvoiceBalanceEqualsOrExceeds=$100.00, timeSinceEarliestUnpaidInvoiceEqualsOrExceeds=30 days, and an account with 2 unpaid invoices totaling $150.00 where the oldest is 35 days old, when the overdue condition is evaluated against current billing state, then the condition evaluates true (2>=2 AND 150>=100 AND 35days>=30days) and the account transitions into that overdue state. Edge cases: If there are no unpaid invoices, dateOfEarliestUnpaidInvoice is null and the time-based condition can never trigger even if configured (line 72, 79-80); Control-tag inclusion/exclusion checks use exact tag-definition-id matches, not tag names (line 96-112). Parameters: All thresholds are per-tenant XML catalog config (no universal default); any unset threshold is treated as automatically satisfied (null-check short-circuits to true, line 77-83). Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (The Given/When/Then is faithful to overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84.
  [Line Numbers]: 69 to 84
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java
  [Source Code]:
  ```java
      @Override
      public boolean evaluate(final BillingState state, final LocalDate date) {
          LocalDate unpaidInvoiceTriggerDate = null;
          if (timeSinceEarliestUnpaidInvoiceEqualsOrExceeds != null && state.getDateOfEarliestUnpaidInvoice() != null) {  // no date => no unpaid invoices
              unpaidInvoiceTriggerDate = state.getDateOfEarliestUnpaidInvoice().plus(timeSinceEarliestUnpaidInvoiceEqualsOrExceeds.toJodaPeriod());
          }

          return
                  (numberOfUnpaidInvoicesEqualsOrExceeds == null || state.getNumberOfUnpaidInvoices() >= numberOfUnpaidInvoicesEqualsOrExceeds) &&
                  (totalUnpaidInvoiceBalanceEqualsOrExceeds == null || totalUnpaidInvoiceBalanceEqualsOrExceeds.compareTo(state.getBalanceOfUnpaidInvoices()) <= 0) &&
                  (timeSinceEarliestUnpaidInvoiceEqualsOrExceeds == null ||
                   (unpaidInvoiceTriggerDate != null && !unpaidInvoiceTriggerDate.isAfter(date))) &&
                  (responseForLastFailedPayment == null || responseIsIn(state.getResponseForLastFailedPayment(), responseForLastFailedPayment)) &&
                  (controlTagInclusion == null || isTagIn(controlTagInclusion, state.getTags())) &&
                  (controlTagExclusion == null || isTagNotIn(controlTagExclusion, state.getTags()));
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — overdue module (DefaultOverdueCondition)
  [Confidence score]: Medium

- [Kill-BR-0110] - Entitlement state derivation precedence
  [Description]: Entitlement state is recomputed each time in strict priority order: cancelled beats expired, beats pending, beats blocked or active. For example: given An entitlement cancelled effective 2026-08-01 that also has an active BLOCKED tag, when State is computed at 2026-08-31, then State returns CANCELLED, not BLOCKED. Edge cases: Cancel date equal to now counts as already cancelled; EXPIRED only applies to FIXEDTERM plans. Parameters: Order: CANCELLED then EXPIRED then PENDING then BLOCKED/ACTIVE. Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (Faithful: `computeStateForEntitlement()` (DefaultEventsStream.java:494-510) does implement exactly the precedence claimed: if `entitlementEffectiveEndDateTime <= utcNow` -> CANCELLED (line 496-497, checked first and short-circuits everything else via the else.
  [Line Numbers]: 494 to 510
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  [Source Code]:
  ```java
      private void computeStateForEntitlement() {
          // Current state for the ENTITLEMENT_SERVICE_NAME is set to cancelled
          if (entitlementEffectiveEndDateTime != null && entitlementEffectiveEndDateTime.compareTo(utcNow) <= 0) {
              entitlementState = EntitlementState.CANCELLED;
          } else {
              final SubscriptionBaseTransition expiryTransition = subscription.getAllTransitions(false).stream().filter(transition -> transition.getTransitionType().equals(SubscriptionBaseTransitionType.EXPIRED)).findFirst().orElse(null);
              if (expiryTransition != null && expiryTransition.getEffectiveTransitionTime() != null && expiryTransition.getEffectiveTransitionTime().compareTo(utcNow) <= 0) {
                  entitlementState = EntitlementState.EXPIRED;
              }
              else if (entitlementEffectiveStartDateTime.compareTo(utcNow) > 0) {
                  entitlementState = EntitlementState.PENDING;
              } else {
                  // Gather states across all services and check if one of them is set to 'blockEntitlement'
                  entitlementState = (currentStateBlockingAggregator != null && currentStateBlockingAggregator.isBlockEntitlement() ? EntitlementState.BLOCKED : EntitlementState.ACTIVE);
              }
          }
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — entitlement module (DefaultEventsStream)
  [Confidence score]: Medium

- [Kill-BR-0111] - Overdue transition is a no-op on unchanged state name
  [Description]: If overdue re-evaluation lands on the same state, nothing is persisted or published. For example: given Account currently in OD1, when Re-evaluation again computes OD1, then storeNewState and the change event are both skipped.
  [Line Numbers]: 127 to 132
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
          if (previousOverdueState.getName().equals(nextOverdueState.getName())) {
              log.debug("OverdueStateApplicator is no-op: previousState={}, nextState={}", previousOverdueState, nextOverdueState);
              return;
          } else {
              log.debug("OverdueStateApplicator has new state: previousState={}, nextState={}", previousOverdueState, nextOverdueState);
          }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — overdue module (OverdueStateApplicator)
  [Confidence score]: High

- [Kill-BR-0112] - Overdue-driven subscription cancellation policy
  [Description]: If the new overdue state specifies a cancellation policy other than NONE, all base-plan entitlements are cancelled at the overdue effective date, immediate or end-of-term as configured. For example: given Account enters OD3 with overdueCancellationPolicy=IMMEDIATE, when apply() processes the new state, then All non-add-on entitlements are cancelled immediately; add-ons cascade. Edge cases: Add-ons created in the future are missed by this filter (line 323 comment). Parameters: OverdueCancellationPolicy: NONE, END_OF_TERM, IMMEDIATE.
  [Line Numbers]: 285 to 329
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      private void cancelSubscriptionsIfRequired(final DateTime effectiveDate, final ImmutableAccountData account, final OverdueState nextOverdueState, final InternalCallContext context) throws OverdueException {
          if (nextOverdueState.getOverdueCancellationPolicy() == OverdueCancellationPolicy.NONE) {
              return;
          }

          final CallContext callContext = internalCallContextFactory.createCallContext(context);
          try {
              final BillingActionPolicy actionPolicy;
              switch (nextOverdueState.getOverdueCancellationPolicy()) {
                  case END_OF_TERM:
                      actionPolicy = BillingActionPolicy.END_OF_TERM;
                      break;
                  case IMMEDIATE:
                      actionPolicy = BillingActionPolicy.IMMEDIATE;
                      break;
                  default:
                      throw new IllegalStateException("Unexpected OverdueCancellationPolicy " + nextOverdueState.getOverdueCancellationPolicy());
              }
              final List<Entitlement> toBeCancelled = new LinkedList<Entitlement>();
              computeEntitlementsToCancel(account, toBeCancelled, callContext);

              try {
                  entitlementInternalApi.cancel(toBeCancelled, context.toLocalDate(effectiveDate), actionPolicy, Collections.emptyList(), context);
              } catch (final EntitlementApiException e) {
                  throw new OverdueException(e);
              }
          } catch (final EntitlementApiException e) {
              throw new OverdueException(e);
          }
      }

      private void computeEntitlementsToCancel(final ImmutableAccountData account, final List<Entitlement> result, final CallContext context) throws EntitlementApiException {
          final List<Entitlement> allEntitlementsForAccountId = entitlementApi.getAllEntitlementsForAccountId(account.getId(), context);
          // Entitlement is smart enough and will cancel the associated add-ons. See also discussion in https://github.com/killbill/killbill/issues/94

          final Collection<Entitlement> allEntitlementsButAddonsForAccountId = allEntitlementsForAccountId
                  .stream()
                  .filter(entitlement -> {
                      // Note: this would miss add-ons created in the future. We should expose a new API to do something similar to EventsStreamBuilder#findBaseSubscription
                      return !ProductCategory.ADD_ON.equals(entitlement.getLastActiveProductCategory());
                  })
                  .collect(Collectors.toUnmodifiableList());

          result.addAll(allEntitlementsButAddonsForAccountId);
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — overdue module (OverdueStateApplicator)
  [Confidence score]: High

- [Kill-BR-0113] - Add-on cascade cancellation on base plan change or cancel
  [Description]: Cancelling or changing a base subscription auto-cancels its add-ons unless the new product still includes or allows them. For example: given Base plan changes to a product whose included/available lists exclude the active add-on, when The CHANGE transition is processed, then A cancellation blocking-state is created for that add-on at the same effective date. Edge cases: Base cancellation always cascades to all active add-ons.
  [Line Numbers]: 389 to 423
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  [Source Code]:
  ```java
      private boolean isAddOnsNeedToBeBlocked(final SubscriptionBase subscription, final Product baseTransitionTriggerNextProduct) {
          // Compute included and available addons for the new product
          final Collection<String> includedAddonsForProduct;
          final Collection<String> availableAddonsForProduct;
          if (baseTransitionTriggerNextProduct == null) {
              includedAddonsForProduct = Collections.emptyList();
              availableAddonsForProduct = Collections.emptyList();
          } else {
              includedAddonsForProduct = baseTransitionTriggerNextProduct.getIncluded().stream()
                                                                         .map(Product::getName)
                                                                         .collect(Collectors.toUnmodifiableSet());

              availableAddonsForProduct = baseTransitionTriggerNextProduct.getAvailable()
                                                                          .stream()
                                                                          .map(Product::getName)
                                                                          .collect(Collectors.toUnmodifiableSet());
          }

          final Plan lastActivePlan = subscription.getLastActivePlan();

          return ProductCategory.ADD_ON.equals(subscription.getCategory()) &&
                 // Check the subscription started, if not we don't want it, and that way we avoid doing NPE a few lines below.
                 lastActivePlan != null &&
                 // Check the entitlement for that add-on hasn't been cancelled yet
                 getEntitlementCancellationEvent(subscription.getId()) == null &&
                 (
                         // Base subscription cancelled
                         baseTransitionTriggerNextProduct == null ||
                         (
                                 // Change plan - check which add-ons to cancel
                                 includedAddonsForProduct.contains(lastActivePlan.getProduct().getName()) ||
                                 !availableAddonsForProduct.contains(subscription.getLastActivePlan().getProduct().getName())
                         )
                 );
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — entitlement module (DefaultEventsStream)
  [Confidence score]: High

- [Kill-BR-0114] - Invoice status lifecycle is one-way: DRAFT to COMMITTED to VOID
  [Description]: Invoice status only moves forward; VOID is terminal; setting the same status again is rejected. For example: given Invoice already VOID, when changeInvoiceStatus to COMMITTED is called, then INVOICE_INVALID_STATUS exception thrown. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: Re-derived the code at DefaultInvoiceDao.java:1372-1417 directly.
  [Line Numbers]: 1372 to 1417
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
      @Override
      public void changeInvoiceStatus(final UUID invoiceId, final InvoiceStatus newStatus, final InternalCallContext context) throws InvoiceApiException {
          final List<CustomField> invoiceCustomFields = getInvoiceCustomFields(context);
          final List<Tag> invoicesTags = getInvoicesTags(context);

          transactionalSqlDao.execute(false, InvoiceApiException.class, entitySqlDaoWrapperFactory -> {
              final InvoiceSqlDao transactional = entitySqlDaoWrapperFactory.become(InvoiceSqlDao.class);

              // Retrieve the invoice and make sure it belongs to the right account
              final InvoiceModelDao invoice = transactional.getById(invoiceId.toString(), context);

              if (invoice == null) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_NOT_FOUND, invoiceId);
              }

              if (invoice.getStatus().equals(newStatus) || invoice.getStatus().equals(InvoiceStatus.VOID)) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_INVALID_STATUS, newStatus, invoiceId, invoice.getStatus());
              }

              transactional.updateStatusAndTargetDate(invoiceId.toString(), newStatus.toString(), invoice.getTargetDate(), context);

              // Run through all invoices
              // Current invoice could be a credit item that needs to be rebalanced
              cbaDao.doCBAComplexityFromTransaction(invoiceCustomFields, invoicesTags, entitySqlDaoWrapperFactory, context);

              // Invoice creation event sent on COMMITTED
              if (InvoiceStatus.COMMITTED.equals(newStatus)) {
                  notifyBusOfInvoiceCreation(entitySqlDaoWrapperFactory, invoice, context);
              } else if (InvoiceStatus.VOID.equals(newStatus)) {
                  // https://github.com/killbill/killbill/issues/1448
                  notifyBusOfInvoiceAdjustment(entitySqlDaoWrapperFactory, invoiceId, invoice.getAccountId(), context.getUserToken(), context);

                  // Deactivate any usage trackingIds if necessary
                  final InvoiceTrackingSqlDao trackingSqlDao = entitySqlDaoWrapperFactory.become(InvoiceTrackingSqlDao.class);
                  final List<InvoiceTrackingModelDao> invoiceTrackingModelDaos = trackingSqlDao.getTrackingsForInvoices(List.of(invoiceId.toString()), context);
                  if (!invoiceTrackingModelDaos.isEmpty()) {
                      final Collection<String> invoiceTrackingIdsToDeactivate = invoiceTrackingModelDaos.stream()
                              .map(input -> input.getId().toString())
                              .collect(Collectors.toUnmodifiableList());

                      trackingSqlDao.deactivateByIds(invoiceTrackingIdsToDeactivate, context);
                  }
              }
              return null;
          });
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — invoice module (DefaultInvoiceDao)
  [Confidence score]: Medium

- [Kill-BR-0115] - Committing fires invoice-creation event; voiding fires adjustment event and deactivates usage tracking
  [Description]: Committing an invoice publishes an invoice-creation event; voiding publishes an adjustment event and deactivates any usage tracking IDs on that invoice. For example: given A DRAFT invoice with usage tracking IDs recorded, when changeInvoiceStatus to VOID is called, then notifyBusOfInvoiceAdjustment fires and tracking rows are deactivated.
  [Line Numbers]: 1397 to 1414
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
              // Invoice creation event sent on COMMITTED
              if (InvoiceStatus.COMMITTED.equals(newStatus)) {
                  notifyBusOfInvoiceCreation(entitySqlDaoWrapperFactory, invoice, context);
              } else if (InvoiceStatus.VOID.equals(newStatus)) {
                  // https://github.com/killbill/killbill/issues/1448
                  notifyBusOfInvoiceAdjustment(entitySqlDaoWrapperFactory, invoiceId, invoice.getAccountId(), context.getUserToken(), context);

                  // Deactivate any usage trackingIds if necessary
                  final InvoiceTrackingSqlDao trackingSqlDao = entitySqlDaoWrapperFactory.become(InvoiceTrackingSqlDao.class);
                  final List<InvoiceTrackingModelDao> invoiceTrackingModelDaos = trackingSqlDao.getTrackingsForInvoices(List.of(invoiceId.toString()), context);
                  if (!invoiceTrackingModelDaos.isEmpty()) {
                      final Collection<String> invoiceTrackingIdsToDeactivate = invoiceTrackingModelDaos.stream()
                              .map(input -> input.getId().toString())
                              .collect(Collectors.toUnmodifiableList());

                      trackingSqlDao.deactivateByIds(invoiceTrackingIdsToDeactivate, context);
                  }
              }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0116] - New invoice DRAFT vs COMMITTED driven by account auto-invoice-draft setting
  [Description]: A new invoice starts DRAFT if the account is tagged auto-invoice-draft; otherwise it is COMMITTED immediately and payable right away. For example: given Account tagged AUTO_INVOICE_DRAFT with charges due, when Invoice generation runs, then Invoice is created as DRAFT, no payment attempt triggers.
  [Line Numbers]: 91 to 91
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java
  [Source Code]:
  ```java
          final InvoiceStatus invoiceStatus = events.isAccountAutoInvoiceDraft() ? InvoiceStatus.DRAFT : InvoiceStatus.COMMITTED;
  ```
  [Applies to]: Lifecycle rule (Priority P0) — invoice module (DefaultInvoiceGenerator)
  [Confidence score]: High

- [Kill-BR-0117] - Reused draft invoices auto-promote to COMMITTED but never demote
  [Description]: When new billing items are added to an existing DRAFT invoice and the incoming write wants COMMITTED, the status is upgraded; the reverse never happens. For example: given Existing DRAFT invoice on disk, new invoice model for same id with status COMMITTED, when createInvoices() processes the existing invoice, then newStatus set to COMMITTED and persisted. Edge cases: Target date upgraded only if the new target date is later.
  [Line Numbers]: 513 to 536
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
                          } else {

                              // Allow transition from DRAFT to COMMITTED or keep current status
                              InvoiceStatus newStatus = invoiceOnDisk.getStatus();
                              boolean statusUpdated = false;
                              if (InvoiceStatus.COMMITTED == invoiceModelDao.getStatus() && InvoiceStatus.DRAFT == invoiceOnDisk.getStatus()) {
                                  statusUpdated = true;
                                  newStatus = InvoiceStatus.COMMITTED;
                              }

                              // Update if target date is specified and prev targetDate was either null or prior to new date
                              LocalDate newTargetDate = invoiceOnDisk.getTargetDate();
                              boolean targetDateUpdated = false;
                              if (invoiceModelDao.getTargetDate() != null &&
                                  (invoiceOnDisk.getTargetDate() == null || invoiceOnDisk.getTargetDate().compareTo(invoiceModelDao.getTargetDate()) < 0)) {
                                  targetDateUpdated = true;
                                  newTargetDate = invoiceModelDao.getTargetDate();
                              }

                              if (statusUpdated || targetDateUpdated) {
                                  invoiceSqlDao.updateStatusAndTargetDate(invoiceModelDao.getId().toString(), newStatus.toString(), newTargetDate, context);
                                  committedReusedInvoiceId.add(invoiceModelDao.getId());
                              }
                          }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0118] - Parent-invoice adjustment mirroring only happens once parent invoice is COMMITTED
  [Description]: When a child-account invoice is item-adjusted, the parent invoice summary line is mirrored with a new adjustment item only if the parent invoice is already COMMITTED. For example: given Child account invoice item-adjusted, parent invoice already COMMITTED, when The adjustment is propagated to the parent invoice, then A new ItemAdjInvoiceItem mirroring the amount is added to the parent invoice. Note (extraction review flagged this rule for SME confirmation): What happens to the mirrored parent adjustment if the parent invoice is still DRAFT at the time of the child adjustment?.
  [Line Numbers]: 1481 to 1499
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  [Source Code]:
  ```java
          if (parentInvoiceModelDao.getStatus().equals(InvoiceStatus.COMMITTED)) {
              final ItemAdjInvoiceItem adj = new ItemAdjInvoiceItem(UUIDs.randomUUID(),
                                                                    lastChildInvoiceItemAdjustment.getCreatedDate(),
                                                                    parentSummaryInvoiceItem.getInvoiceId(),
                                                                    parentSummaryInvoiceItem.getAccountId(),
                                                                    lastChildInvoiceItemAdjustment.getStartDate(),
                                                                    description,
                                                                    childInvoiceAdjustmentAmount,
                                                                    parentInvoiceModelDao.getCurrency(),
                                                                    parentSummaryInvoiceItem.getId(),
                                                                    null);
              parentInvoiceModelDao.addInvoiceItem(new InvoiceItemModelDao(adj));
              invoiceDao.createInvoices(List.of(parentInvoiceModelDao), null, Collections.emptySet(), null, null, false,parentContext);
              return;
          }

          // update item amount
          final BigDecimal newParentInvoiceItemAmount = childInvoiceAdjustmentAmount.add(parentSummaryInvoiceItem.getAmount());
          invoiceDao.updateInvoiceItemAmount(parentSummaryInvoiceItem.getId(), newParentInvoiceItemAmount, parentContext);
  ```
  [Applies to]: Lifecycle rule (Priority P1) — invoice module (InvoiceDispatcher)
  [Confidence score]: Medium

- [Kill-BR-0119] - Payment transaction state machine per transaction type
  [Description]: Each payment transaction starts INIT and resolves to SUCCESS, FAILED, PENDING, or ERRORED; from PENDING it can still resolve to SUCCESS, FAILED, or ERRORED. CHARGEBACK has no PENDING state. For example: given AUTHORIZE transaction in AUTH_PENDING, when Gateway confirms success, then State moves to AUTH_SUCCESS, unlocking CAPTURE and VOID. Edge cases: CHARGEBACK has no PENDING transition.
  [Line Numbers]: 38 to 409
  [Source Code File Name]: payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml
  [Source Code]:
  ```xml
          <stateMachine name="AUTHORIZE">
              <states>
                  <state name="AUTH_INIT"/>
                  <state name="AUTH_PENDING"/>
                  <state name="AUTH_SUCCESS"/>
                  <state name="AUTH_FAILED"/>
                  <state name="AUTH_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>AUTH_INIT</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>AUTH_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_INIT</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>AUTH_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_INIT</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>AUTH_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_PENDING</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>AUTH_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_PENDING</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>AUTH_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_PENDING</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>AUTH_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>AUTH_INIT</initialState>
                      <operation>OP_AUTHORIZE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>AUTH_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_AUTHORIZE"/>
              </operations>
          </stateMachine>

          <stateMachine name="CAPTURE">
              <states>
                  <state name="CAPTURE_INIT"/>
                  <state name="CAPTURE_PENDING"/>
                  <state name="CAPTURE_SUCCESS"/>
                  <state name="CAPTURE_FAILED"/>
                  <state name="CAPTURE_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>CAPTURE_INIT</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>CAPTURE_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_INIT</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>CAPTURE_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_INIT</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>CAPTURE_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_PENDING</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>CAPTURE_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_PENDING</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>CAPTURE_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_PENDING</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>CAPTURE_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>CAPTURE_INIT</initialState>
                      <operation>OP_CAPTURE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>CAPTURE_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_CAPTURE"/>
              </operations>
          </stateMachine>

          <stateMachine name="PURCHASE">
              <states>
                  <state name="PURCHASE_INIT"/>
                  <state name="PURCHASE_PENDING"/>
                  <state name="PURCHASE_SUCCESS"/>
                  <state name="PURCHASE_FAILED"/>
                  <state name="PURCHASE_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>PURCHASE_INIT</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>PURCHASE_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_INIT</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>PURCHASE_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_INIT</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>PURCHASE_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_PENDING</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>PURCHASE_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_PENDING</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>PURCHASE_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_PENDING</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>PURCHASE_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>PURCHASE_INIT</initialState>
                      <operation>OP_PURCHASE</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>PURCHASE_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_PURCHASE"/>
              </operations>
          </stateMachine>

          <stateMachine name="REFUND">
              <states>
                  <state name="REFUND_INIT"/>
                  <state name="REFUND_PENDING"/>
                  <state name="REFUND_SUCCESS"/>
                  <state name="REFUND_FAILED"/>
                  <state name="REFUND_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>REFUND_INIT</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>REFUND_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_INIT</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>REFUND_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_INIT</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>REFUND_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_PENDING</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>REFUND_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_PENDING</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>REFUND_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_PENDING</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>REFUND_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>REFUND_INIT</initialState>
                      <operation>OP_REFUND</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>REFUND_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_REFUND"/>
              </operations>
          </stateMachine>

          <stateMachine name="CREDIT">
              <states>
                  <state name="CREDIT_INIT"/>
                  <state name="CREDIT_PENDING"/>
                  <state name="CREDIT_SUCCESS"/>
                  <state name="CREDIT_FAILED"/>
                  <state name="CREDIT_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>CREDIT_INIT</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>CREDIT_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_INIT</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>CREDIT_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_INIT</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>CREDIT_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_PENDING</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>CREDIT_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_PENDING</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>CREDIT_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_PENDING</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>CREDIT_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>CREDIT_INIT</initialState>
                      <operation>OP_CREDIT</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>CREDIT_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_CREDIT"/>
              </operations>
          </stateMachine>

          <stateMachine name="VOID">
              <states>
                  <state name="VOID_INIT"/>
                  <state name="VOID_PENDING"/>
                  <state name="VOID_SUCCESS"/>
                  <state name="VOID_FAILED"/>
                  <state name="VOID_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>VOID_INIT</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>VOID_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_INIT</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>VOID_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_INIT</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>PENDING</operationResult>
                      <finalState>VOID_PENDING</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_PENDING</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>VOID_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_PENDING</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>VOID_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_PENDING</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>VOID_ERRORED</finalState>
                  </transition>
                  <transition>
                      <initialState>VOID_INIT</initialState>
                      <operation>OP_VOID</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>VOID_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_VOID"/>
              </operations>
          </stateMachine>
          <stateMachine name="CHARGEBACK">
              <states>
                  <state name="CHARGEBACK_INIT"/>
                  <state name="CHARGEBACK_SUCCESS"/>
                  <state name="CHARGEBACK_FAILED"/>
                  <state name="CHARGEBACK_ERRORED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>CHARGEBACK_INIT</initialState>
                      <operation>OP_CHARGEBACK</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>CHARGEBACK_SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>CHARGEBACK_INIT</initialState>
                      <operation>OP_CHARGEBACK</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>CHARGEBACK_FAILED</finalState>
                  </transition>
                  <transition>
                      <initialState>CHARGEBACK_INIT</initialState>
                      <operation>OP_CHARGEBACK</operation>
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>CHARGEBACK_ERRORED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_CHARGEBACK"/>
              </operations>
          </stateMachine>
  ```
  [Applies to]: Lifecycle rule (Priority P0) — payment module (PaymentStates)
  [Confidence score]: High

- [Kill-BR-0120] - Cross-transaction-type payment workflow linkage
  [Description]: Successful AUTHORIZE unlocks CAPTURE or VOID; successful CAPTURE unlocks REFUND, more CAPTURE, or CHARGEBACK; successful PURCHASE unlocks REFUND or CHARGEBACK directly; CHARGEBACK can be re-chargebacked or fall back to REFUND on failure. For example: given Payment with only a successful PURCHASE transaction, when A CAPTURE is attempted, then Not a valid transition; PURCHASE_SUCCESS only links to REFUND and CHARGEBACK.
  [Line Numbers]: 412 to 497
  [Source Code File Name]: payment/src/main/resources/org/killbill/billing/payment/PaymentStates.xml
  [Source Code]:
  ```xml
      <linkStateMachines>
          <linkStateMachine>
              <initialStateMachine>BIG_BANG</initialStateMachine>
              <initialState>BIG_BANG_INIT</initialState>
              <finalStateMachine>AUTHORIZE</finalStateMachine>
              <finalState>AUTH_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>BIG_BANG</initialStateMachine>
              <initialState>BIG_BANG_INIT</initialState>
              <finalStateMachine>PURCHASE</finalStateMachine>
              <finalState>PURCHASE_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>BIG_BANG</initialStateMachine>
              <initialState>BIG_BANG_INIT</initialState>
              <finalStateMachine>CREDIT</finalStateMachine>
              <finalState>CREDIT_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>AUTHORIZE</initialStateMachine>
              <initialState>AUTH_SUCCESS</initialState>
              <finalStateMachine>CAPTURE</finalStateMachine>
              <finalState>CAPTURE_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>AUTHORIZE</initialStateMachine>
              <initialState>AUTH_SUCCESS</initialState>
              <finalStateMachine>VOID</finalStateMachine>
              <finalState>VOID_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>CAPTURE</initialStateMachine>
              <initialState>CAPTURE_SUCCESS</initialState>
              <finalStateMachine>REFUND</finalStateMachine>
              <finalState>REFUND_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>CAPTURE</initialStateMachine>
              <initialState>CAPTURE_SUCCESS</initialState>
              <finalStateMachine>CAPTURE</finalStateMachine>
              <finalState>CAPTURE_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>CAPTURE</initialStateMachine>
              <initialState>CAPTURE_SUCCESS</initialState>
              <finalStateMachine>CHARGEBACK</finalStateMachine>
              <finalState>CHARGEBACK_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>REFUND</initialStateMachine>
              <initialState>REFUND_SUCCESS</initialState>
              <finalStateMachine>REFUND</finalStateMachine>
              <finalState>REFUND_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>REFUND</initialStateMachine>
              <initialState>REFUND_SUCCESS</initialState>
              <finalStateMachine>CHARGEBACK</finalStateMachine>
              <finalState>CHARGEBACK_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>PURCHASE</initialStateMachine>
              <initialState>PURCHASE_SUCCESS</initialState>
              <finalStateMachine>REFUND</finalStateMachine>
              <finalState>REFUND_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>PURCHASE</initialStateMachine>
              <initialState>PURCHASE_SUCCESS</initialState>
              <finalStateMachine>CHARGEBACK</finalStateMachine>
              <finalState>CHARGEBACK_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>CHARGEBACK</initialStateMachine>
              <initialState>CHARGEBACK_SUCCESS</initialState>
              <finalStateMachine>CHARGEBACK</finalStateMachine>
              <finalState>CHARGEBACK_INIT</finalState>
          </linkStateMachine>
          <linkStateMachine>
              <initialStateMachine>CHARGEBACK</initialStateMachine>
              <initialState>CHARGEBACK_FAILED</initialState>
              <finalStateMachine>REFUND</finalStateMachine>
              <finalState>REFUND_INIT</finalState>
          </linkStateMachine>
      </linkStateMachines>
  ```
  [Applies to]: Lifecycle rule (Priority P0) — payment module (PaymentStates)
  [Confidence score]: High

- [Kill-BR-0121] - Payment retry loop is unbounded at the state-machine level
  [Description]: A retry attempt starts INIT, moves to SUCCESS on success, or RETRIED on failure where it can self-loop indefinitely until success or an exception moves it to ABORTED. For example: given Retry attempt in RETRIED after several failures, when The next retry also fails, then State stays RETRIED; the state machine itself enforces no maximum retry count. Note (extraction review flagged this rule for SME confirmation): Where is the actual maximum retry count and backoff schedule enforced for payment retries?.
  [Line Numbers]: 22 to 84
  [Source Code File Name]: payment/src/main/resources/org/killbill/billing/payment/retry/RetryStates.xml
  [Source Code]:
  ```xml
          <stateMachine name="PAYMENT_RETRY">
              <states>
                  <state name="INIT"/>
                  <state name="SUCCESS"/>
                  <state name="RETRIED"/>
                  <state name="ABORTED"/>
              </states>
              <transitions>
                  <transition>
                      <initialState>INIT</initialState>
                      <operation>OP_RETRY</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>INIT</initialState>
                      <operation>OP_RETRY</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>RETRIED</finalState>
                  </transition>
                  <transition>
                      <initialState>INIT</initialState>
                      <operation>OP_RETRY</operation>
                      <!-- We are using EXCEPTION operation result to get out of the RETRIED state and transition to  ABORTED -->
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>ABORTED</finalState>
                  </transition>
                  <transition>
                      <initialState>RETRIED</initialState>
                      <operation>OP_RETRY</operation>
                      <operationResult>SUCCESS</operationResult>
                      <finalState>SUCCESS</finalState>
                  </transition>
                  <transition>
                      <initialState>RETRIED</initialState>
                      <operation>OP_RETRY</operation>
                      <operationResult>FAILURE</operationResult>
                      <finalState>RETRIED</finalState>
                  </transition>
                  <transition>
                      <initialState>RETRIED</initialState>
                      <operation>OP_RETRY</operation>
                      <!-- We are using EXCEPTION operation result to get out of the RETRIED state and transition to  ABORTED -->
                      <operationResult>EXCEPTION</operationResult>
                      <finalState>ABORTED</finalState>
                  </transition>
              </transitions>
              <operations>
                  <operation name="OP_RETRY"/>
              </operations>
          </stateMachine>
      </stateMachines>

      <linkStateMachines>
          <linkStateMachine>
              <initialStateMachine>PAYMENT_RETRY</initialStateMachine>
              <initialState>ABORTED</initialState>
              <finalStateMachine>PAYMENT_RETRY</finalStateMachine>
              <finalState>INIT</finalState>
          </linkStateMachine>
      </linkStateMachines>

  </stateMachineConfig>
  ```
  [Applies to]: Lifecycle rule (Priority P1) — payment module (RetryStates)
  [Confidence score]: Medium

- [Kill-BR-0122] - Billing-blocked periods insert a zero-charge 'disable' event and restore pricing on re-enable
  [Description]: When a subscription enters a billing-blocked state (e.g. via overdue or manual block), a synthetic billing event is inserted at the block's start with no fixed price, no recurring price, and billing period NO_BILLING_PERIOD so the invoice engine stops charging; when the block ends, a matching synthetic event restores the exact plan, phase, prices, usages and billing period that were in effect immediately before the block. For example: given A subscription actively billing $29.99/month gets a billing-blocking state effective 2026-05-10, later lifted 2026-05-25, when insertBlockingEvents runs for that subscription, then A START_BILLING_DISABLED billing event is added at 2026-05-10 with fixedPrice=null, recurringPrice=null, billingPeriod=NO_BILLING_PERIOD; an END_BILLING_DISABLED event is added at 2026-05-25 restoring recurringPrice=$29.99 and the original billingPeriod.
  [Line Numbers]: 247 to 317
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      protected BillingEvent createNewDisableEvent(final DateTime disabledDurationStart,
                                                   final BillingEvent previousEvent) {
          final int billCycleDay = previousEvent.getBillCycleDayLocal();
          final int quantity = previousEvent.getQuantity();
          final DateTime effectiveDate = disabledDurationStart;
          final PlanPhase planPhase = previousEvent.getPlanPhase();
          final Plan plan = previousEvent.getPlan();

          // Make sure to set the fixed price to null and the billing period to NO_BILLING_PERIOD,
          // which makes invoice disregard this event
          final BigDecimal fixedPrice = null;
          final BigDecimal recurringPrice = null;
          final BillingPeriod billingPeriod = BillingPeriod.NO_BILLING_PERIOD;

          final Currency currency = previousEvent.getCurrency();
          final String description = "";
          final SubscriptionBaseTransitionType type = SubscriptionBaseTransitionType.START_BILLING_DISABLED;
          final Long totalOrdering = globaltotalOrder.getAndIncrement();

          return new DefaultBillingEvent(previousEvent.getSubscriptionId(),
                                         previousEvent.getBundleId(),
                                         effectiveDate,
                                         plan,
                                         planPhase,
                                         fixedPrice,
                                         recurringPrice,
                                         Collections.emptyList(),
                                         currency,
                                         billingPeriod,
                                         billCycleDay,
                                         quantity,
                                         description,
                                         totalOrdering,
                                         type
          );
      }

      protected BillingEvent createNewReenableEvent(final DateTime odEventTime,
                                                    final BillingEvent previousEvent) throws CatalogApiException {
          // All fields are populated with the event state from before the blocking period, for invoice to resume invoicing
          final int billCycleDay = previousEvent.getBillCycleDayLocal();
          final int quantity = previousEvent.getQuantity();
          final DateTime effectiveDate = odEventTime;
          final PlanPhase planPhase = previousEvent.getPlanPhase();
          final BigDecimal fixedPrice = previousEvent.getFixedPrice();
          final BigDecimal recurringPrice = previousEvent.getRecurringPrice();
          final List<Usage> usages = previousEvent.getUsages();
          final Plan plan = previousEvent.getPlan();
          final Currency currency = previousEvent.getCurrency();
          final String description = "";
          final BillingPeriod billingPeriod = previousEvent.getBillingPeriod();
          final SubscriptionBaseTransitionType type = SubscriptionBaseTransitionType.END_BILLING_DISABLED;
          final Long totalOrdering = globaltotalOrder.getAndIncrement();

          return new DefaultBillingEvent(previousEvent.getSubscriptionId(),
                                         previousEvent.getBundleId(),
                                         effectiveDate,
                                         plan,
                                         planPhase,
                                         fixedPrice,
                                         recurringPrice,
                                         usages,
                                         currency,
                                         billingPeriod,
                                         billCycleDay,
                                         quantity,
                                         description,
                                         totalOrdering,
                                         type
          );
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — junction module (BlockingCalculator)
  [Confidence score]: High

- [Kill-BR-0123] - Account BCD is set exactly once via merge, then locked
  [Description]: When merging incoming account data with existing account data, the bill-cycle-day is only taken from the new data if the account doesn't already have one set (and the new value isn't the 'unset' sentinel); once set, subsequent merges keep the existing BCD. For example: given currentAccount.billCycleDayLocal is unset (equals DEFAULT_BILLING_CYCLE_DAY_LOCAL) and the incoming account has billCycleDayLocal=15, when mergeWithDelegate is invoked, then The merged account gets billCycleDayLocal=15; on any later merge, the existing value of 15 is preserved regardless of what's supplied. Parameters: DEFAULT_BILLING_CYCLE_DAY_LOCAL sentinel (0).
  [Line Numbers]: 317 to 323
  [Source Code File Name]: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  [Source Code]:
  ```java
          if (currentAccount.getBillCycleDayLocal() == DEFAULT_BILLING_CYCLE_DAY_LOCAL && // There is *not* already a BCD set
              billCycleDayLocal != null && // and the value proposed is not null
              billCycleDayLocal != DEFAULT_BILLING_CYCLE_DAY_LOCAL) {  // and the proposed date is not 0
              accountData.setBillCycleDayLocal(billCycleDayLocal);
          } else {
              accountData.setBillCycleDayLocal(currentAccount.getBillCycleDayLocal());
          }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — account module (DefaultAccount)
  [Confidence score]: High

- [Kill-BR-0124] - Pause bundle fully blocks; resume bundle fully clears
  [Description]: Pausing a bundle sets a blocking state that blocks change, entitlement use, and billing all at once for that bundle; resuming sets a 'clear' blocking state that unblocks all three simultaneously. There's no partial pause (e.g., billing-only). For example: given An active bundle, when pause(bundleId, effectiveDate) is called, then A BlockingState ENT_STATE_BLOCKED is recorded with blockChange=blockEntitlement=blockBilling=true, effective as of effectiveDate. Parameters: ENT_STATE_BLOCKED / ENT_STATE_CLEAR: hardcoded state names.
  [Line Numbers]: 168 to 198
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java
  [Source Code]:
  ```java
      public void pause(final UUID bundleId, @Nullable final LocalDate localEffectiveDate, final Iterable<PluginProperty> properties, final InternalCallContext internalCallContext) throws EntitlementApiException {
          final DateTime effectiveDateTime = dateHelper.fromLocalDateAndReferenceTime(localEffectiveDate, internalCallContext.getCreatedDate(), internalCallContext);

          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = List.of(
                  new DefaultBaseEntitlementWithAddOnsSpecifier(
                          bundleId,
                          null,
                          null,
                          effectiveDateTime,
                          effectiveDateTime,
                          false));

          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.PAUSE_BUNDLE,
                                                                                 null,
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 internalCallContextFactory.createCallContext(internalCallContext));
          final WithEntitlementPlugin<Void> pauseWithPlugin = new WithEntitlementPlugin<Void>() {
              @Override
              public Void doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  try {
                      final SubscriptionBase baseSubscription = subscriptionInternalApi.getBaseSubscription(bundleId, internalCallContext);
                      blockUnblockBundle(bundleId, DefaultEntitlementApi.ENT_STATE_BLOCKED, KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName(), localEffectiveDate, true, true, true, baseSubscription, internalCallContext);
                  } catch (final SubscriptionBaseApiException e) {
                      throw new EntitlementApiException(e);
                  }
                  return null;
              }
          };
  ```
  [Line Numbers]: 202 to 232
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java
  [Source Code]:
  ```java
      public void resume(final UUID bundleId, @Nullable final LocalDate localEffectiveDate, final Iterable<PluginProperty> properties, final InternalCallContext internalCallContext) throws EntitlementApiException {

          final DateTime effectiveDateTime = dateHelper.fromLocalDateAndReferenceTime(localEffectiveDate, internalCallContext.getCreatedDate(), internalCallContext);
          final BaseEntitlementWithAddOnsSpecifier baseEntitlementWithAddOnsSpecifier = new DefaultBaseEntitlementWithAddOnsSpecifier(
                  bundleId,
                  null,
                  null,
                  effectiveDateTime,
                  effectiveDateTime,
                  false);
          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = new ArrayList<BaseEntitlementWithAddOnsSpecifier>();
          baseEntitlementWithAddOnsSpecifierList.add(baseEntitlementWithAddOnsSpecifier);
          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.RESUME_BUNDLE,
                                                                                 null,
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 internalCallContextFactory.createCallContext(internalCallContext));
          final WithEntitlementPlugin<Void> resumeWithPlugin = new WithEntitlementPlugin<Void>() {
              @Override
              public Void doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  try {
                      final SubscriptionBase baseSubscription = subscriptionInternalApi.getBaseSubscription(bundleId, internalCallContext);
                      blockUnblockBundle(bundleId, DefaultEntitlementApi.ENT_STATE_CLEAR, KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName(), localEffectiveDate, false, false, false, baseSubscription, internalCallContext);
                  } catch (final SubscriptionBaseApiException e) {
                      throw new EntitlementApiException(e);
                  }
                  return null;
              }
          };
  ```
  [Applies to]: Lifecycle rule (Priority P1) — entitlement module (DefaultEntitlementApiBase)
  [Confidence score]: High

- [Kill-BR-0125] - Subscription entitlement-state derivation from event stream
  [Description]: A subscription's current state (ACTIVE, PENDING, CANCELLED, EXPIRED) is not stored directly; it is derived by replaying the subscription's event stream: CREATE/TRANSFER events set state ACTIVE, CANCEL events set state CANCELLED, an EXPIRED event sets state EXPIRED, and if only a future transition exists the state is PENDING. For example: given A subscription whose only recorded transition is a future-dated CREATE event, when getState() is called before that future date, then The subscription reports EntitlementState.PENDING (no ACTIVE state exists yet); once a CANCEL API event is applied, the previous transition's next-state becomes CANCELLED and getState() returns CANCELLED going forward. Parameters: States: ACTIVE, PENDING, CANCELLED, EXPIRED (Entitlement.EntitlementState). Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (The cited code is faithfully described: getState() (lines 192-205) derives EntitlementState by inspecting the previous/pending SubscriptionBaseTransition, and the event-stream-to-transition mapping (lines 1007-1050) confirms CREATE/TRANSFER set nextState=ACTIV.
  [Line Numbers]: 192 to 205
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  [Source Code]:
  ```java
      @Override
      public EntitlementState getState() {

          final SubscriptionBaseTransition previousTransition = getPreviousTransition();
          if (previousTransition != null) {
              return previousTransition.getNextState();
          }

          final SubscriptionBaseTransition pendingTransition = getPendingTransition();
          if (pendingTransition != null) {
              return EntitlementState.PENDING;
          }
          throw new IllegalStateException("Should return a valid EntitlementState");
      }
  ```
  [Line Numbers]: 1007 to 1050
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  [Source Code]:
  ```java
                  case API_USER:
                      final ApiEvent userEV = (ApiEvent) cur;
                      apiEventType = userEV.getApiEventType();
                      isFromDisk = userEV.isFromDisk();

                      switch (apiEventType) {
                          case TRANSFER:
                          case CREATE:
                              prevEventId = null;
                              prevCreatedDate = null;
                              previousState = null;
                              previousPlan = null;
                              previousPhase = null;
                              previousPriceList = null;
                              nextState = EntitlementState.ACTIVE;
                              nextPlanName = userEV.getEventPlan();
                              nextPhaseName = userEV.getEventPlanPhase();
                              lastPlanChangeTime = cur.getEffectiveDate();
                              break;

                          case CHANGE:
                              nextPlanName = userEV.getEventPlan();
                              nextPhaseName = userEV.getEventPlanPhase();
                              lastPlanChangeTime = cur.getEffectiveDate();
                              break;

                          case CANCEL:
                              nextState = EntitlementState.CANCELLED;
                              nextPlanName = null;
                              nextPhaseName = null;
                              break;
                          case UNCANCEL:
                          case UNDO_CHANGE:
                          default:
                              throw new SubscriptionBaseError(String.format(
                                      "Unexpected UserEvent type = %s", userEV
                                              .getApiEventType().toString()));
                      }
                      break;
                  case EXPIRED:
                      nextState = EntitlementState.EXPIRED;
                      nextPlanName = null;
                      nextPhaseName = null;
                      break;
  ```
  [Applies to]: Lifecycle rule (Priority P1) — subscription module (DefaultSubscriptionBase)
  [Confidence score]: Medium

- [Kill-BR-0126] - Add-on cascade cancellation on base plan cancel/change (subscription-level)
  [Description]: When a base plan is cancelled or changed and takes effect now or in the past, every non-cancelled/non-expired add-on in the bundle is auto-cancelled if the base plan is gone entirely, if the add-on is now included for free in the new base product, or if the add-on is no longer listed as available for the new base product. The add-on's cancellation date is the later of the base event's effective date and the add-on's own alignment start date. Cascade cancellation is skipped entirely for base plan changes/cancels scheduled in the future (it is recomputed when the event actually fires). For example: given A base subscription changing today from Product A (which allows add-on X) to Product B (which does not list X as available), when The plan change event is processed, then Add-on X receives an auto-generated CANCEL event effective at the change date (or its own start date if later). Parameters: None (driven by catalog available/included add-on lists per product).
  [Line Numbers]: 773 to 821
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private List<DefaultSubscriptionBase> computeAddOnsToCancel(final Collection<SubscriptionBaseEvent> cancelEvents, final Product baseProduct, final UUID bundleId, final DateTime effectiveDate, final SubscriptionCatalog catalog, final InternalCallContext internalCallContext) throws CatalogApiException {
          // If cancellation/change occur in the future, there is nothing to do
          if (effectiveDate.compareTo(internalCallContext.getCreatedDate()) > 0) {
              return Collections.emptyList();
          } else {
              return addCancellationAddOnForEventsIfRequired(cancelEvents, baseProduct, bundleId, effectiveDate, catalog, internalCallContext);
          }
      }

      private List<DefaultSubscriptionBase> addCancellationAddOnForEventsIfRequired(final Collection<SubscriptionBaseEvent> events, final Product baseProduct, final UUID bundleId,
                                                                                    final DateTime effectiveDate, final SubscriptionCatalog catalog, final InternalTenantContext internalTenantContext) throws CatalogApiException {

          final List<DefaultSubscriptionBase> subscriptionsToBeCancelled = new ArrayList<DefaultSubscriptionBase>();

          final List<DefaultSubscriptionBase> subscriptions = dao.getSubscriptions(bundleId, Collections.emptyList(), catalog, internalTenantContext);

          for (final SubscriptionBase subscription : subscriptions) {
              final DefaultSubscriptionBase cur = (DefaultSubscriptionBase) subscription;
              if (cur.getState() == EntitlementState.CANCELLED ||
                  cur.getState() == EntitlementState.EXPIRED ||
                  cur.getCategory() != ProductCategory.ADD_ON) {
                  continue;
              }

              final Plan addonCurrentPlan = cur.getCurrentOrPendingPlan();

              if (baseProduct == null ||
                  addonUtils.isAddonIncluded(baseProduct, addonCurrentPlan) ||
                  !addonUtils.isAddonAvailable(baseProduct, addonCurrentPlan)) {

                  final SubscriptionBaseEvent cancelEvent;
                  final DateTime cancellationDateTime;

                  if (effectiveDate.isAfter(cur.getAlignStartDate())) {
                      cancellationDateTime = effectiveDate;
                  } else {
                      cancellationDateTime = cur.getAlignStartDate();
                  }

                  cancelEvent = new ApiEventCancel(new ApiEventBuilder()
                                                           .setSubscriptionId(cur.getId())
                                                           .setEffectiveDate(cancellationDateTime)
                                                           .setFromDisk(true));
                  subscriptionsToBeCancelled.add(cur);
                  events.add(cancelEvent);
              }
          }
          return subscriptionsToBeCancelled;
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0127] - Bundle pause blocks everything; resume clears everything
  [Description]: Pausing a bundle records a bundle-level ENT_BLOCKED blocking state with blockBilling, blockEntitlement, and blockChange all set to true. Resuming records an ENT_STATE_CLEAR blocking state with all three flags set to false, unblocking the bundle. For example: given An active bundle, when pause(bundleId, effectiveDate) is called, then All billing, entitlement use, and plan changes are blocked for every subscription in the bundle from that date; a subsequent resume() call fully clears the block. Parameters: State names: ENT_BLOCKED, ENT_STATE_CLEAR (DefaultEntitlementApi.ENT_STATE_BLOCKED/ENT_STATE_CLEAR).
  [Line Numbers]: 168 to 234
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java
  [Source Code]:
  ```java
      public void pause(final UUID bundleId, @Nullable final LocalDate localEffectiveDate, final Iterable<PluginProperty> properties, final InternalCallContext internalCallContext) throws EntitlementApiException {
          final DateTime effectiveDateTime = dateHelper.fromLocalDateAndReferenceTime(localEffectiveDate, internalCallContext.getCreatedDate(), internalCallContext);

          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = List.of(
                  new DefaultBaseEntitlementWithAddOnsSpecifier(
                          bundleId,
                          null,
                          null,
                          effectiveDateTime,
                          effectiveDateTime,
                          false));

          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.PAUSE_BUNDLE,
                                                                                 null,
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 internalCallContextFactory.createCallContext(internalCallContext));
          final WithEntitlementPlugin<Void> pauseWithPlugin = new WithEntitlementPlugin<Void>() {
              @Override
              public Void doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  try {
                      final SubscriptionBase baseSubscription = subscriptionInternalApi.getBaseSubscription(bundleId, internalCallContext);
                      blockUnblockBundle(bundleId, DefaultEntitlementApi.ENT_STATE_BLOCKED, KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName(), localEffectiveDate, true, true, true, baseSubscription, internalCallContext);
                  } catch (final SubscriptionBaseApiException e) {
                      throw new EntitlementApiException(e);
                  }
                  return null;
              }
          };
          pluginExecution.executeWithPlugin(pauseWithPlugin, pluginContext);
      }

      public void resume(final UUID bundleId, @Nullable final LocalDate localEffectiveDate, final Iterable<PluginProperty> properties, final InternalCallContext internalCallContext) throws EntitlementApiException {

          final DateTime effectiveDateTime = dateHelper.fromLocalDateAndReferenceTime(localEffectiveDate, internalCallContext.getCreatedDate(), internalCallContext);
          final BaseEntitlementWithAddOnsSpecifier baseEntitlementWithAddOnsSpecifier = new DefaultBaseEntitlementWithAddOnsSpecifier(
                  bundleId,
                  null,
                  null,
                  effectiveDateTime,
                  effectiveDateTime,
                  false);
          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = new ArrayList<BaseEntitlementWithAddOnsSpecifier>();
          baseEntitlementWithAddOnsSpecifierList.add(baseEntitlementWithAddOnsSpecifier);
          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.RESUME_BUNDLE,
                                                                                 null,
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 internalCallContextFactory.createCallContext(internalCallContext));
          final WithEntitlementPlugin<Void> resumeWithPlugin = new WithEntitlementPlugin<Void>() {
              @Override
              public Void doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  try {
                      final SubscriptionBase baseSubscription = subscriptionInternalApi.getBaseSubscription(bundleId, internalCallContext);
                      blockUnblockBundle(bundleId, DefaultEntitlementApi.ENT_STATE_CLEAR, KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName(), localEffectiveDate, false, false, false, baseSubscription, internalCallContext);
                  } catch (final SubscriptionBaseApiException e) {
                      throw new EntitlementApiException(e);
                  }
                  return null;
              }
          };
          pluginExecution.executeWithPlugin(resumeWithPlugin, pluginContext);
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — entitlement module (DefaultEntitlementApiBase)
  [Confidence score]: High

- [Kill-BR-0128] - Blocked periods are aggregated into disabled billing durations and injected as billing on/off events
  [Description]: For each subscription, all active blocking states from subscription, bundle, and account level (that occur before the subscription's cancel/expire date, if any) are merged into contiguous 'disabled' date ranges per service. For each disabled range, a synthetic START_BILLING_DISABLED billing event (zero fixed/recurring price, NO_BILLING_PERIOD) is inserted at the start, and an END_BILLING_DISABLED event restoring the prior price/plan is inserted at the end (if the block has ended); overlapping/adjacent disabled ranges from different services are merged into one. For example: given A subscription blocked for billing (overdue) from day 10 to day 20, with no other blocking state, when Billing events are computed for invoicing, then A START_BILLING_DISABLED event is inserted on day 10 (nulling fixed/recurring price) and an END_BILLING_DISABLED event restoring the previous plan/price is inserted on day 20. Parameters: Event types: START_BILLING_DISABLED, END_BILLING_DISABLED. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 is justified: this logic decides whether recurring/fixed charges are generated (or entirely suppressed) for a subscription during overdue/blocking periods — it directly controls whether the customer is billed, so it is squarely "moves money" territory.
  [Line Numbers]: 82 to 161
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      public boolean insertBlockingEvents(final SortedSet<BillingEvent> billingEvents,
                                          final Set<UUID> skippedSubscriptions,
                                          final Map<UUID, List<SubscriptionBase>> subscriptionsForAccount,
                                          final VersionedCatalog catalog,
                                          @Nullable final LocalDate cutoffDt,
                                          final InternalTenantContext context) throws CatalogApiException {
          if (billingEvents.size() <= 0) {
              return false;
          }

          final Collection<BillingEvent> billingEventsToAdd = new TreeSet<>();
          final Collection<BillingEvent> billingEventsToRemove = new TreeSet<>();

          final List<BlockingState> blockingEvents = blockingApi.getBlockingActiveForAccount(catalog, cutoffDt, context);

          // Group blocking states per type
          final Collection<BlockingState> accountBlockingEvents = new LinkedList<>();
          final Map<UUID, List<BlockingState>> perBundleBlockingEvents = new HashMap<>();
          final Map<UUID, List<BlockingState>> perSubscriptionBlockingEvents = new HashMap<>();
          for (final BlockingState blockingEvent : blockingEvents) {
              if (blockingEvent.getType() == BlockingStateType.ACCOUNT) {
                  accountBlockingEvents.add(blockingEvent);
              } else if (blockingEvent.getType() == BlockingStateType.SUBSCRIPTION_BUNDLE) {
                  perBundleBlockingEvents.putIfAbsent(blockingEvent.getBlockedId(), new LinkedList<BlockingState>());
                  perBundleBlockingEvents.get(blockingEvent.getBlockedId()).add(blockingEvent);
              } else if (blockingEvent.getType() == BlockingStateType.SUBSCRIPTION) {
                  perSubscriptionBlockingEvents.putIfAbsent(blockingEvent.getBlockedId(), new LinkedList<BlockingState>());
                  perSubscriptionBlockingEvents.get(blockingEvent.getBlockedId()).add(blockingEvent);
              }
          }

          // Group billing events per subscriptionId
          final Map<UUID, SortedSet<BillingEvent>> perSubscriptionBillingEvents = new HashMap<UUID, SortedSet<BillingEvent>>();
          for (final BillingEvent event : billingEvents) {
              if (!perSubscriptionBillingEvents.containsKey(event.getSubscriptionId())) {
                  perSubscriptionBillingEvents.put(event.getSubscriptionId(), new TreeSet<BillingEvent>());
              }
              perSubscriptionBillingEvents.get(event.getSubscriptionId()).add(event);
          }

          for (final Entry<UUID, List<SubscriptionBase>> entry : subscriptionsForAccount.entrySet()) {
              final UUID bundleId = entry.getKey();

              final List<BlockingState> bundleBlockingEvents = perBundleBlockingEvents.get(bundleId) != null ? perBundleBlockingEvents.get(bundleId) : Collections.emptyList();

              for (final SubscriptionBase subscription : entry.getValue()) {
                  // Avoid inserting additional events for subscriptions that don't even have a START event
                  if (skippedSubscriptions.contains(subscription.getId())) {
                      continue;
                  }

                  final List<BlockingState> subscriptionBlockingEvents = perSubscriptionBlockingEvents.get(subscription.getId()) != null ? perSubscriptionBlockingEvents.get(subscription.getId()) : Collections.emptyList();
                  final SortedSet<BillingEvent> subscriptionBillingEvents = perSubscriptionBillingEvents.getOrDefault(subscription.getId(), Collections.emptySortedSet());
                  // Subscription#getEndDate() is only set for CANCELLED subscriptions, so we need to use the last billing event to determine the termination date
                  final BillingEvent lastBillingEvent = !subscriptionBillingEvents.isEmpty() ? subscriptionBillingEvents.last() : null;
                  final DateTime terminationDate = lastBillingEvent != null &&
                                                   (lastBillingEvent.getTransitionType() == SubscriptionBaseTransitionType.CANCEL ||
                                                   lastBillingEvent.getTransitionType() == SubscriptionBaseTransitionType.EXPIRED)
                                                   ? lastBillingEvent.getEffectiveDate() : null;

                  final List<BlockingState> aggregateSubscriptionBlockingEvents = getAggregateBlockingEventsPerSubscription(terminationDate, subscriptionBlockingEvents, bundleBlockingEvents, accountBlockingEvents);
                  final List<DisabledDuration> aggregateBlockingDurations = createBlockingDurations(aggregateSubscriptionBlockingEvents);


                  final SortedSet<BillingEvent> newEvents = createNewEvents(aggregateBlockingDurations, subscriptionBillingEvents, context);
                  billingEventsToAdd.addAll(newEvents);

                  final SortedSet<BillingEvent> removedEvents = eventsToRemove(aggregateBlockingDurations, subscriptionBillingEvents);
                  billingEventsToRemove.addAll(removedEvents);
              }
          }

          billingEvents.addAll(billingEventsToAdd);

          for (final BillingEvent eventToRemove : billingEventsToRemove) {
              billingEvents.remove(eventToRemove);
          }

          return !(billingEventsToAdd.isEmpty() && billingEventsToRemove.isEmpty());
      }
  ```
  [Line Numbers]: 196 to 282
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      protected SortedSet<BillingEvent> createNewEvents(final Iterable<DisabledDuration> disabledDuration,
                                                        final Iterable<BillingEvent> subscriptionBillingEvents,
                                                        final InternalTenantContext context) throws CatalogApiException {
          Preconditions.checkState(context.getAccountRecordId() != null);

          final SortedSet<BillingEvent> result = new TreeSet<BillingEvent>();

          for (final DisabledDuration duration : disabledDuration) {
              // The first one before the blocked duration
              final BillingEvent precedingInitialEvent = precedingActiveBillingEventForSubscription(duration.getStart(), subscriptionBillingEvents);
              // The last one during of before the duration
              final BillingEvent precedingFinalEvent = precedingActiveBillingEventForSubscription(duration.getEnd(), subscriptionBillingEvents);

              if (precedingInitialEvent != null) { // there is a preceding billing event
                  result.add(createNewDisableEvent(duration.getStart(), precedingInitialEvent));
                  if (duration.getEnd() != null && precedingFinalEvent != null) { // no second event in the pair means they are still disabled (no re-enable)
                      result.add(createNewReenableEvent(duration.getEnd(), precedingFinalEvent));
                  }
              } else if (precedingFinalEvent != null) { // can happen - e.g. phase event
                  result.add(createNewReenableEvent(duration.getEnd(), precedingFinalEvent));
              }
              // N.B. if there's no precedingInitial and no precedingFinal then there's nothing to do
          }
          return result;
      }

      protected BillingEvent precedingActiveBillingEventForSubscription(final DateTime disabledDurationStart,
                                                                        final Iterable<BillingEvent> subscriptionBillingEvents) {
          if (disabledDurationStart == null) {
              return null;
          }

          // We look for the first billingEvent strictly prior our disabledDurationStart or null if none
          BillingEvent prev = null;
          for (final BillingEvent event : subscriptionBillingEvents) {
              if (!event.getEffectiveDate().isBefore(disabledDurationStart)) {
                  return prev;
              } else {
                  prev = event;
              }
          }

          // We ignore anything beyond the final event
          // TODO 0.23.x
          if (prev == null || prev.getTransitionType() == SubscriptionBaseTransitionType.CANCEL /*|| prev.getTransitionType() == SubscriptionBaseTransitionType.EXPIRED */) {
              return null;
          }

          return prev;
      }

      protected BillingEvent createNewDisableEvent(final DateTime disabledDurationStart,
                                                   final BillingEvent previousEvent) {
          final int billCycleDay = previousEvent.getBillCycleDayLocal();
          final int quantity = previousEvent.getQuantity();
          final DateTime effectiveDate = disabledDurationStart;
          final PlanPhase planPhase = previousEvent.getPlanPhase();
          final Plan plan = previousEvent.getPlan();

          // Make sure to set the fixed price to null and the billing period to NO_BILLING_PERIOD,
          // which makes invoice disregard this event
          final BigDecimal fixedPrice = null;
          final BigDecimal recurringPrice = null;
          final BillingPeriod billingPeriod = BillingPeriod.NO_BILLING_PERIOD;

          final Currency currency = previousEvent.getCurrency();
          final String description = "";
          final SubscriptionBaseTransitionType type = SubscriptionBaseTransitionType.START_BILLING_DISABLED;
          final Long totalOrdering = globaltotalOrder.getAndIncrement();

          return new DefaultBillingEvent(previousEvent.getSubscriptionId(),
                                         previousEvent.getBundleId(),
                                         effectiveDate,
                                         plan,
                                         planPhase,
                                         fixedPrice,
                                         recurringPrice,
                                         Collections.emptyList(),
                                         currency,
                                         billingPeriod,
                                         billCycleDay,
                                         quantity,
                                         description,
                                         totalOrdering,
                                         type
          );
      }
  ```
  [Line Numbers]: 319 to 357
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingCalculator.java
  [Source Code]:
  ```java
      // In ascending order
      protected List<DisabledDuration> createBlockingDurations(final Iterable<BlockingState> inputBundleEvents) {
          final List<DisabledDuration> result = new LinkedList<DisabledDuration>();

          final Map<String, BlockingStateService> svcBlockedMap = new HashMap<String, BlockingStateService>();
          for (final BlockingState bs : inputBundleEvents) {
              final String service = bs.getService();
              svcBlockedMap.putIfAbsent(service, new BlockingStateService());
              svcBlockedMap.get(service).addBlockingState(bs);
          }

          final Collection<DisabledDuration> unorderedDisabledDuration = new LinkedList<DisabledDuration>();
          for (final Entry<String, BlockingStateService> entry : svcBlockedMap.entrySet()) {
              unorderedDisabledDuration.addAll(entry.getValue().build());
          }
          final List<DisabledDuration> sortedDisabledDuration = unorderedDisabledDuration.stream()
                  .sorted()
                  .collect(Collectors.toUnmodifiableList());

          DisabledDuration prevDuration = null;
          for (final DisabledDuration d : sortedDisabledDuration) {
              // isDisjoint
              if (prevDuration == null) {
                  prevDuration = d;
              } else {
                  if (prevDuration.isDisjoint(d)) {
                      result.add(prevDuration);
                      prevDuration = d;
                  } else {
                      prevDuration = DisabledDuration.mergeDuration(prevDuration, d);
                  }
              }
          }
          if (prevDuration != null) {
              result.add(prevDuration);
          }

          return result;
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — junction module (BlockingCalculator)
  [Confidence score]: Medium

- [Kill-BR-0129] - Tagging an invoice as written off excludes it from balance and triggers overdue re-evaluation
  [Description]: Marking an invoice as written off (or reversing that) adds/removes the WRITTEN_OFF control tag on the invoice and fires an invoice-adjustment bus event, which overdue processing listens to in order to recompute the account's overdue/blocking status. For example: given A committed, unpaid invoice contributing to an account being overdue, when tagInvoiceAsWrittenOff is called on that invoice, then The WRITTEN_OFF tag is added, the invoice's balance becomes $0 for aggregation purposes, and an adjustment notification is published so overdue state can be re-evaluated. Parameters: Tag: ControlTagType.WRITTEN_OFF.
  [Line Numbers]: 313 to 334
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  [Source Code]:
  ```java
      @Override
      public void tagInvoiceAsWrittenOff(final UUID invoiceId, final CallContext context) throws TagApiException, InvoiceApiException {
          // Note: the tagApi is audited
          final InternalCallContext internalContext = internalCallContextFactory.createInternalCallContext(invoiceId, ObjectType.INVOICE, context);
          tagApi.addTag(invoiceId, ObjectType.INVOICE, ControlTagType.WRITTEN_OFF.getId(), internalContext);

          // Retrieve the invoice for the account id
          final Invoice invoice = new DefaultInvoice(dao.getById(invoiceId, internalContext));
          // This is for overdue
          notifyBusOfInvoiceAdjustment(invoiceId, invoice.getAccountId(), internalContext);
      }

      @Override
      public void tagInvoiceAsNotWrittenOff(final UUID invoiceId, final CallContext context) throws TagApiException, InvoiceApiException {
          // Note: the tagApi is audited
          final InternalCallContext internalContext = internalCallContextFactory.createInternalCallContext(invoiceId, ObjectType.INVOICE, context);
          tagApi.removeTag(invoiceId, ObjectType.INVOICE, ControlTagType.WRITTEN_OFF.getId(), internalContext);

          // Retrieve the invoice for the account id
          final Invoice invoice = new DefaultInvoice(dao.getById(invoiceId, internalContext));
          // This is for overdue
          notifyBusOfInvoiceAdjustment(invoiceId, invoice.getAccountId(), internalContext);
  ```
  [Applies to]: Lifecycle rule (Priority P1) — invoice module (DefaultInvoiceUserApi)
  [Confidence score]: High

- [Kill-BR-0130] - Payment plugin status mapped to internal transaction status
  [Description]: A payment plugin's reported status is translated to Kill Bill's internal transaction status: PROCESSED to SUCCESS, PENDING to PENDING, ERROR to PAYMENT_FAILURE (the transaction reached the processor but was declined), CANCELED to PLUGIN_FAILURE (the processor confirms the transaction never happened), and UNDEFINED or a null/unrecognized status to UNKNOWN (left for the Janitor to reconcile later). For example: given A plugin returns PaymentPluginStatus.CANCELED for a charge attempt, when toTransactionStatus is invoked, then The transaction is recorded with TransactionStatus.PLUGIN_FAILURE (distinct from a declined-card PAYMENT_FAILURE). Parameters: Mapping: PROCESSED->SUCCESS, PENDING->PENDING, ERROR->PAYMENT_FAILURE, CANCELED->PLUGIN_FAILURE, UNDEFINED/null->UNKNOWN.
  [Line Numbers]: 33 to 56
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentTransactionInfoPluginConverter.java
  [Source Code]:
  ```java
      public static TransactionStatus toTransactionStatus(final PaymentTransactionInfoPlugin paymentTransactionInfoPlugin) {
          final PaymentPluginStatus status = Objects.requireNonNullElse(paymentTransactionInfoPlugin.getStatus(), PaymentPluginStatus.UNDEFINED);
          switch (status) {
              case PROCESSED:
                  return TransactionStatus.SUCCESS;
              case PENDING:
                  return TransactionStatus.PENDING;
              // The naming is a bit inconsistent, but ERROR on the plugin side means PAYMENT_FAILURE (that is a case where transaction went through but did not
              // return successfully (e.g: CC denied, ...)
              case ERROR:
                  return TransactionStatus.PAYMENT_FAILURE;
              //
              // The plugin is trying to tell us that it knows for sure that payment transaction did not happen (connection failure,..)
              case CANCELED:
                  return TransactionStatus.PLUGIN_FAILURE;
              //
              // This will be picked up by Janitor to figure out what really happened and correct the state if needed
              // Note that the default case includes the null status
              //
              case UNDEFINED:
              default:
                  return TransactionStatus.UNKNOWN;
          }
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — payment module (PaymentTransactionInfoPluginConverter)
  [Confidence score]: High

- [Kill-BR-0131] - Janitor only repairs PENDING/UNKNOWN transactions, never terminal ones
  [Description]: The background Janitor process only attempts to re-query the plugin and repair a payment transaction's stored status if it is currently PENDING or UNKNOWN; SUCCESS, PAYMENT_FAILURE, and PLUGIN_FAILURE are treated as terminal and are left untouched even if re-checked. For example: given A payment transaction stored with status PAYMENT_FAILURE, when The Janitor task runs against that transaction, then It short-circuits and returns the current status unchanged, without contacting the plugin. Parameters: TRANSACTION_STATUSES_TO_CONSIDER = {PENDING, UNKNOWN}.
  [Line Numbers]: 63 to 63
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  [Source Code]:
  ```java
      static final List<TransactionStatus> TRANSACTION_STATUSES_TO_CONSIDER = List.of(TransactionStatus.PENDING, TransactionStatus.UNKNOWN);
  ```
  [Line Numbers]: 112 to 155
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  [Source Code]:
  ```java
      TransactionStatus updatePaymentAndTransactionIfNeeded2(final UUID accountId,
                                                            final UUID paymentTransactionId,
                                                            @Nullable final TransactionStatus currentTransactionStatus,
                                                            @Nullable final PaymentTransactionInfoPlugin paymentTransactionInfoPlugin,
                                                            final boolean isApiPayment,
                                                            final InternalTenantContext internalTenantContext) throws LockFailedException {
          // In the GET case, make sure we bail as early as possible (see PaymentRefresher)
          if (currentTransactionStatus != null && !TRANSACTION_STATUSES_TO_CONSIDER.contains(currentTransactionStatus)) {
              // Nothing to do, so we return the currentTransactionStatus to indicate that nothing has changed
              return currentTransactionStatus;
          }

          return tryToDoJanitorOperationWithAccountLock(new JanitorIterationCallback() {
              @Override
              public TransactionStatus doIteration() {
                  return updatePaymentAndTransactionIfNeeded(accountId,
                                                             paymentTransactionId,
                                                             paymentTransactionInfoPlugin,
                                                             isApiPayment,
                                                             internalTenantContext);
              }
          }, internalTenantContext);
      }

      private TransactionStatus updatePaymentAndTransactionIfNeeded(final UUID accountId,
                                                                    final UUID paymentTransactionId,
                                                                    @Nullable final PaymentTransactionInfoPlugin paymentTransactionInfoPlugin,
                                                                    final boolean isApiPayment,
                                                                    final InternalTenantContext internalTenantContext) {
          // State may have changed since we originally retrieved with no lock
          final PaymentTransactionModelDao paymentTransaction = paymentDao.getPaymentTransaction(paymentTransactionId, internalTenantContext);
          if (!TRANSACTION_STATUSES_TO_CONSIDER.contains(paymentTransaction.getTransactionStatus())) {
              // Nothing to do, so we return the currentTransactionStatus to indicate that nothing has changed
              return paymentTransaction.getTransactionStatus();
          }

          // On-the-fly Janitor already has the latest state, avoid a round-trip to the plugin
          final PaymentTransactionInfoPlugin latestPaymentTransactionInfoPlugin = paymentTransactionInfoPlugin != null ? paymentTransactionInfoPlugin : getLatestPaymentTransactionInfoPlugin(paymentTransaction, internalTenantContext);
          return updatePaymentAndTransactionInternal(accountId,
                                                     paymentTransaction,
                                                     latestPaymentTransactionInfoPlugin,
                                                     isApiPayment,
                                                     internalTenantContext);
      }
  ```
  [Line Numbers]: 167 to 221
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  [Source Code]:
  ```java
          final TransactionStatus transactionStatus = computeNewTransactionStatusFromPaymentTransactionInfoPlugin(paymentTransactionInfoPlugin, paymentTransaction.getTransactionStatus());
          final String newPaymentState;
          switch (transactionStatus) {
              case PENDING:
                  newPaymentState = paymentStateMachineHelper.getPendingStateForTransaction(paymentTransaction.getTransactionType());
                  break;
              case SUCCESS:
                  newPaymentState = paymentStateMachineHelper.getSuccessfulStateForTransaction(paymentTransaction.getTransactionType());
                  break;
              case PAYMENT_FAILURE:
                  newPaymentState = paymentStateMachineHelper.getFailureStateForTransaction(paymentTransaction.getTransactionType());
                  break;
              case PLUGIN_FAILURE:
                  newPaymentState = paymentStateMachineHelper.getErroredStateForTransaction(paymentTransaction.getTransactionType());
                  break;
              case UNKNOWN:
              default:
                  // We can't get anything interesting from the plugin...
                  log.info("Unable to repair paymentId='{}', paymentTransactionId='{}', currentTransactionStatus='{}', newTransactionStatus='{}'",
                           paymentId, paymentTransaction.getId(), paymentTransaction.getTransactionStatus(), transactionStatus);
                  return transactionStatus;
          }

          // Our status did not change, so we just insert a new notification (attemptNumber will be incremented)
          if (transactionStatus == paymentTransaction.getTransactionStatus()) {
              log.info("Unable to repair paymentId='{}', paymentTransactionId='{}', currentTransactionStatus='{}', newTransactionStatus='{}'",
                       paymentId, paymentTransaction.getId(), paymentTransaction.getTransactionStatus(), transactionStatus);
              return transactionStatus;
          }

          final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(internalTenantContext.getTenantRecordId(),
                                                                                                               internalTenantContext.getAccountRecordId(),
                                                                                                               "IncompletePaymentTransactionTask",
                                                                                                               CallOrigin.INTERNAL,
                                                                                                               UserType.SYSTEM,
                                                                                                               UUIDs.randomUUID());
          final PaymentAutomatonDAOHelper paymentAutomatonDAOHelper = new PaymentAutomatonDAOHelper(paymentDao,
                                                                                                    internalCallContext,
                                                                                                    paymentStateMachineHelper);

          log.info("Repairing paymentId='{}', paymentTransactionId='{}', currentTransactionStatus='{}', newTransactionStatus='{}'",
                   paymentId, paymentTransaction.getId(), paymentTransaction.getTransactionStatus(), transactionStatus);
          paymentAutomatonDAOHelper.processPaymentInfoPlugin(transactionStatus,
                                                             paymentTransactionInfoPlugin,
                                                             newPaymentState,
                                                             paymentTransaction.getProcessedAmount(),
                                                             paymentTransaction.getProcessedCurrency(),
                                                             accountId,
                                                             paymentTransaction.getAttemptId(),
                                                             paymentId,
                                                             paymentTransaction.getId(),
                                                             paymentTransaction.getTransactionType(),
                                                             isApiPayment);

          return null;
  ```
  [Applies to]: Lifecycle rule (Priority P0) — payment module (IncompletePaymentTransactionTask)
  [Confidence score]: High

- [Kill-BR-0132] - Overdue state evaluation order: first matching configured state wins
  [Description]: An account's overdue state is computed by testing each configured overdue state's condition, in the order the states are declared in overdue.xml, and returning the first one whose condition evaluates true; if none match, the account is in the CLEAR state. For example: given An overdue.xml with states declared in order OD1, OD2, OD3, where both OD1's and OD2's conditions currently evaluate true for the account's billing state, when calculateOverdueState() runs for that account, then OD1 is returned (the first match in declaration order), even though OD2 would also match. Edge cases: If getStates() is empty or no condition matches, the account resolves to the built-in CLEAR state. Parameters: None (order is purely XML declaration order). Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67 — calculateOverdueState() iterates getStates() in array order and returns the first state whose conditionEvaluation.evaluate(billingState, now).
  [Line Numbers]: 59 to 67
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java
  [Source Code]:
  ```java
      @Override
      public DefaultOverdueState calculateOverdueState(final BillingState billingState, final LocalDate now) throws OverdueApiException {
          for (final DefaultOverdueState overdueState : getStates()) {
              if (overdueState.getConditionEvaluation() != null && overdueState.getConditionEvaluation().evaluate(billingState, now)) {
                  return overdueState;
              }
          }
          return getClearState();
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — overdue module (DefaultOverdueStateSet)
  [Confidence score]: Medium

- [Kill-BR-0133] - Refund idempotency by transaction cookie id
  [Description]: If a refund for the same payment-system transaction key already exists, don't create a duplicate refund record — just update its status/date and skip re-running CBA/adjustment side effects; the amount and payment id must match exactly or the call is rejected. For example: given An existing PaymentPayment DAO refund record for transactionExternalKey 'refund-123' with amount -50.00 and paymentId P1, when createRefund is invoked again for the same transactionExternalKey with a computed refund amount of 50.00 and paymentId P1, then the existing refund record is updated (status/date only) and returned; no new refund row, adjustment, or CBA recompute is created. If the amount or paymentId differs from the existing record, an IllegalStateException/Precondition failure is raised instead. Edge cases: Called multiple times by payment state machine retries for same cookie id; Mismatched amount/paymentId on repeat call throws Precondition failure. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 is justified: this code path in DefaultInvoiceDao.createRefund (invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java:806-917, cited region 842-880) is squarely money-movement and data-integrity logic — it creates/dedupes refund records against payments, enforce.
  [Line Numbers]: 842 to 880
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
              // Before we go further, check if that refund already got inserted -- the payment system keeps a state machine
              // and so this call may be called several time for the same  paymentCookieId (which is really the refundId)
              final InvoicePaymentModelDao existingRefund = transactional.getPaymentForCookieId(transactionExternalKey, context);
              if (existingRefund != null) {

                  Preconditions.checkState(existingRefund.getAmount().compareTo(requestedPositiveAmount.negate()) == 0,
                                           "Found refund for transactionExternalKey=" + transactionExternalKey + ", amount=" + existingRefund.getAmount() +
                                           "and does not match input amount=" + requestedPositiveAmount.negate());

                  Preconditions.checkState(existingRefund.getPaymentId().compareTo(paymentId) == 0,
                                           "Found refund for transactionExternalKey=" + transactionExternalKey + ", paymentId=" + existingRefund.getPaymentId() +
                                           "and does not match input paymentId=" + paymentId);

                  // The entry already exists, bail out (no need to send events, or compute cba logic)
                  if (existingRefund.getStatus() == status) {
                      return existingRefund;
                  }

                  // We only update date and the status
                  existingRefund.setStatus(status);
                  existingRefund.setPaymentDate(context.getCreatedDate());
                  transactional.updateAttempt(existingRefund.getId().toString(),
                                              existingRefund.getPaymentId().toString(),
                                              existingRefund.getPaymentDate().toDate(),
                                              existingRefund.getAmount(),
                                              existingRefund.getCurrency(),
                                              existingRefund.getProcessedCurrency(),
                                              existingRefund.getPaymentCookieId(),
                                              existingRefund.getLinkedInvoicePaymentId().toString(),
                                              existingRefund.getStatus().toString(),
                                              context);
                  result = existingRefund;
              } else {
                  final InvoicePaymentModelDao refund = new InvoicePaymentModelDao(UUIDs.randomUUID(), context.getCreatedDate(), InvoicePaymentType.REFUND,
                                                                                   payment.getInvoiceId(), paymentId,
                                                                                   context.getCreatedDate(), requestedPositiveAmount.negate(),
                                                                                   payment.getCurrency(), payment.getProcessedCurrency(), transactionExternalKey, payment.getId(), status);
                  result = createAndRefresh(transactional, refund, context);
              }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — invoice module (DefaultInvoiceDao)
  [Confidence score]: Medium

- [Kill-BR-0134] - Chargeback reversal resets invoice payment status to INIT
  [Description]: Reversing a chargeback does not delete the chargeback record; it flips its status back to INIT (effectively voiding its effect on balance) and re-triggers CBA recompute for the invoice. For example: given A CHARGED_BACK invoice-payment row in SUCCESS status for transactionExternalKey 'cb-1', when postChargebackReversal is called for 'cb-1', then the row's status is set to INIT (no longer counted against invoice balance), CBA logic is re-run for that invoice, and a bus notification is sent. Edge cases: Unknown cookie id throws PAYMENT_NO_SUCH_PAYMENT.
  [Line Numbers]: 970 to 1003
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
      public InvoicePaymentModelDao postChargebackReversal(final UUID paymentId, final UUID paymentAttemptId, final String chargebackTransactionExternalKey, final InternalCallContext context) throws InvoiceApiException {
          final List<CustomField> invoiceCustomFields = getInvoiceCustomFields(context);
          final List<Tag> invoicesTags = getInvoicesTags(context);

          return transactionalSqlDao.execute(false, InvoiceApiException.class, entitySqlDaoWrapperFactory -> {
              final InvoicePaymentSqlDao transactional = entitySqlDaoWrapperFactory.become(InvoicePaymentSqlDao.class);

              final InvoicePaymentModelDao invoicePayment = transactional.getPaymentForCookieId(chargebackTransactionExternalKey, context);
              if (invoicePayment == null) {
                  throw new InvoiceApiException(ErrorCode.PAYMENT_NO_SUCH_PAYMENT, paymentId);
              }

              transactional.updateAttempt(invoicePayment.getId().toString(),
                                          invoicePayment.getPaymentId().toString(),
                                          invoicePayment.getPaymentDate().toDate(),
                                          invoicePayment.getAmount(),
                                          invoicePayment.getCurrency(),
                                          invoicePayment.getProcessedCurrency(),
                                          invoicePayment.getPaymentCookieId(),
                                          invoicePayment.getLinkedInvoicePaymentId() == null ? null : invoicePayment.getLinkedInvoicePaymentId().toString(),
                                          InvoicePaymentStatus.INIT.toString(),
                                          context);
              final InvoicePaymentModelDao chargebackReversed = transactional.getByRecordId(invoicePayment.getRecordId(), context);

              // Notify the bus since the balance of the invoice changed
              final UUID accountId = transactional.getAccountIdFromInvoicePaymentId(chargebackReversed.getId().toString(), context);

              final CBALogicWrapper cbaWrapper = new CBALogicWrapper(accountId, invoiceCustomFields, invoicesTags, context, entitySqlDaoWrapperFactory);
              cbaWrapper.runCBALogicWithNotificationEvents(Set.of(chargebackReversed.getInvoiceId()));

              notifyBusOfInvoicePayment(entitySqlDaoWrapperFactory, chargebackReversed, accountId, paymentAttemptId, context.getUserToken(), context);

              return chargebackReversed;
          });
  ```
  [Applies to]: Lifecycle rule (Priority P0) — invoice module (DefaultInvoiceDao)
  [Confidence score]: High

- [Kill-BR-0135] - Consecutive duplicate blocking states are pruned when inserting a new one
  [Description]: When a new blocking state is recorded for a blockable object/service, the full chronological history for that object+service is re-sorted, and any state entry that is immediately followed by another entry with the identical state name is deleted as redundant — only the transition points that actually change the state are kept. For example: given History for a subscription/service: t0=S1, t1=S2, t3=S3, and a new insert of S2 at t1' where t0<t1'<t1, when setBlockingStatesAndPostBlockingTransitionEvent runs, then the redundant old S2 at t1 is unactivated/deleted, leaving t0=S1, t1'=S2, t3=S3 (never two consecutive rows with the same state name). Edge cases: Also cleans up legacy duplicate rows from pre-existing bad data per an explicit code comment.
  [Line Numbers]: 236 to 265
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java
  [Source Code]:
  ```java

                  // Go through the (ordered) stream of blocking states for that blocked id and service and check
                  // if there is one or more blocking states for the same state following each others.
                  // If there are, delete them, as they are not needed anymore. A picture being worth a thousand words,
                  // if the current stream is: t0 S1 t1 S2 t3 S3 and we want to insert S2 at t0 < t1' < t1,
                  // the final stream should be: t0 S1 t1' S2 t3 S3 (and not t0 S1 t1' S2 t1 S2 t3 S3)
                  // Note that we also take care of the use case t0 S1 t1 S2 t2 S2 t3 S3 to cleanup legacy systems, although
                  // it shouldn't happen anymore
                  final Collection<UUID> blockingStatesToRemove = new HashSet<UUID>();
                  BlockingStateModelDao prevBlockingStateModelDao = null;
                  for (final BlockingStateModelDao blockingStateModelDao : allForBlockedItAndServiceOrdered) {
                      if (prevBlockingStateModelDao != null && prevBlockingStateModelDao.getState().equals(blockingStateModelDao.getState())) {
                          blockingStatesToRemove.add(blockingStateModelDao.getId());
                      }
                      prevBlockingStateModelDao = blockingStateModelDao;
                  }

                  // Delete unnecessary states (except newBlockingStateModelDao, which doesn't exist in the database)
                  for (final UUID blockedId : blockingStatesToRemove) {
                      if (!newBlockingStateModelDao.getId().equals(blockedId)) {
                          sqlDao.unactiveEvent(blockedId.toString(), context);
                      }
                  }

                  boolean inserted = false;
                  // Create the state, if needed
                  if (!blockingStatesToRemove.contains(newBlockingStateModelDao.getId())) {
                      createAndRefresh(sqlDao, newBlockingStateModelDao, context);
                      inserted = true;
                  }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — entitlement module (DefaultBlockingStateDao)
  [Confidence score]: High

- [Kill-BR-0136] - Blocking-state notification/bus-event batching by aggregation mode
  [Description]: When multiple blocking-state changes are applied together and event aggregation is enabled, only the first bus-eligible (already-effective) transition in the batch records a notification/event; future-dated transitions always get their own scheduled notification. For example: given A batch of 3 blocking-state changes, 2 of which are effective immediately and grouping is enabled, when setBlockingStatesAndPostBlockingTransitionEvent processes the batch, then only the first immediately-effective change triggers a bus notification computation; the second immediately-effective one is skipped for notification purposes. Note (extraction review flagged this rule for SME confirmation): Confirm this event-coalescing does not suppress externally-visible entitlement webhooks that downstream integrators depend on for each individual state change.
  [Line Numbers]: 205 to 286
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java
  [Source Code]:
  ```java
      public void setBlockingStatesAndPostBlockingTransitionEvent(final Map<BlockingState, Optional<UUID>> states, final InternalCallContext context) {
          final boolean groupBusEvents = eventBus.shouldAggregateSubscriptionEvents(context);

          transactionalSqlDao.execute(false, entitySqlDaoWrapperFactory -> {
              final BlockingStateSqlDao sqlDao = entitySqlDaoWrapperFactory.become(BlockingStateSqlDao.class);

              int seqId = 0;
              for (final Entry<BlockingState, Optional<UUID>> entry : states.entrySet()) {
                  final BlockingState state = entry.getKey();
                  final DateTime upToDate = state.getEffectiveDate();
                  final UUID bundleId = entry.getValue().orElse(null);

                  final boolean isBusEvent = state.getEffectiveDate().compareTo(context.getCreatedDate()) <= 0;
                  final boolean shouldRecordNotification = (!isBusEvent || !groupBusEvents || seqId == 0);

                  final BlockingAggregator previousState = shouldRecordNotification ?
                                                           getBlockedStatus(sqlDao, entitySqlDaoWrapperFactory.getHandle(), state.getBlockedId(), state.getType(), bundleId, upToDate, context) :
                                                           null;

                  final BlockingStateModelDao newBlockingStateModelDao = new BlockingStateModelDao(state, context);

                  // Get all blocking states for that blocked id and service
                  final List<BlockingStateModelDao> allForBlockedItAndService = sqlDao.getBlockingHistoryForService(state.getBlockedId(), state.getService(), context);

                  // Add the new one (we rely below on the fact that the ID for newBlockingStateModelDao is now set)
                  allForBlockedItAndService.add(newBlockingStateModelDao);

                  // Re-order what should be the final list (allForBlockedItAndService is ordered by record_id in the SQL and we just added a new state)
                  final List<BlockingStateModelDao> allForBlockedItAndServiceOrdered = allForBlockedItAndService.stream()
                          .sorted(BLOCKING_STATE_MODEL_DAO_ORDERING)
                          .collect(Collectors.toUnmodifiableList());

                  // Go through the (ordered) stream of blocking states for that blocked id and service and check
                  // if there is one or more blocking states for the same state following each others.
                  // If there are, delete them, as they are not needed anymore. A picture being worth a thousand words,
                  // if the current stream is: t0 S1 t1 S2 t3 S3 and we want to insert S2 at t0 < t1' < t1,
                  // the final stream should be: t0 S1 t1' S2 t3 S3 (and not t0 S1 t1' S2 t1 S2 t3 S3)
                  // Note that we also take care of the use case t0 S1 t1 S2 t2 S2 t3 S3 to cleanup legacy systems, although
                  // it shouldn't happen anymore
                  final Collection<UUID> blockingStatesToRemove = new HashSet<UUID>();
                  BlockingStateModelDao prevBlockingStateModelDao = null;
                  for (final BlockingStateModelDao blockingStateModelDao : allForBlockedItAndServiceOrdered) {
                      if (prevBlockingStateModelDao != null && prevBlockingStateModelDao.getState().equals(blockingStateModelDao.getState())) {
                          blockingStatesToRemove.add(blockingStateModelDao.getId());
                      }
                      prevBlockingStateModelDao = blockingStateModelDao;
                  }

                  // Delete unnecessary states (except newBlockingStateModelDao, which doesn't exist in the database)
                  for (final UUID blockedId : blockingStatesToRemove) {
                      if (!newBlockingStateModelDao.getId().equals(blockedId)) {
                          sqlDao.unactiveEvent(blockedId.toString(), context);
                      }
                  }

                  boolean inserted = false;
                  // Create the state, if needed
                  if (!blockingStatesToRemove.contains(newBlockingStateModelDao.getId())) {
                      createAndRefresh(sqlDao, newBlockingStateModelDao, context);
                      inserted = true;
                  }

                  final BlockingAggregator currentState = shouldRecordNotification ?
                                                          getBlockedStatus(sqlDao, entitySqlDaoWrapperFactory.getHandle(), state.getBlockedId(), state.getType(), bundleId, upToDate, context) :
                                                          null;
                  if (shouldRecordNotification &&
                      previousState != null &&
                      currentState != null) {
                      recordBusOrFutureNotificationFromTransaction(entitySqlDaoWrapperFactory,
                                                                   state.getId(),
                                                                   state.getEffectiveDate(),
                                                                   state.getBlockedId(),
                                                                   state.getType(),
                                                                   state.getStateName(),
                                                                   state.getService(),
                                                                   inserted,
                                                                   previousState,
                                                                   currentState,
                                                                   context);
                      seqId = isBusEvent ? seqId + 1 : seqId;
                  }
              }
  ```
  [Applies to]: Lifecycle rule (Priority P2) — entitlement module (DefaultBlockingStateDao)
  [Confidence score]: Medium

- [Kill-BR-0137] - Idempotent get-or-create for the external/manual-pay payment method
  [Description]: Requesting the external (manual) payment method for an account returns the existing one if the account already has one, otherwise creates a new external payment method — it is never duplicated. For example: given An account with no external payment method yet, when createOrGetExternalPaymentMethod is called twice in a row, then the first call creates the external payment method; the second call returns the same payment method id without creating a second one.
  [Line Numbers]: 475 to 491
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  [Source Code]:
  ```java
      public UUID createOrGetExternalPaymentMethod(final String paymentMethodExternalKey, final Account account, final Iterable<PluginProperty> properties, final CallContext callContext, final InternalCallContext context) throws PaymentApiException {
          // Check if this account has already used the external payment plugin
          // If not, it's the first time - add a payment method for it
          final PaymentMethod externalPaymentMethod = getExternalPaymentMethod(properties, callContext, context);
          if (externalPaymentMethod != null) {
              return externalPaymentMethod.getId();
          }
          final DefaultNoOpPaymentMethodPlugin props = new DefaultNoOpPaymentMethodPlugin(UUIDs.randomUUID().toString(), false, properties);
          return addPaymentMethod(paymentMethodExternalKey, ExternalPaymentProviderPlugin.PLUGIN_NAME, account, false, props, properties, callContext, context);
      }

      public ExternalPaymentProviderPlugin createPaymentMethodAndGetExternalPaymentProviderPlugin(final String paymentMethodExternalKey, final Account account, final Iterable<PluginProperty> properties, final CallContext callContext, final InternalCallContext internalContext) throws PaymentApiException {
          // Check if this account has already used the external payment plugin
          // If not, it's the first time - add a payment method for it
          createOrGetExternalPaymentMethod(paymentMethodExternalKey, account, properties, callContext, internalContext);
          return (ExternalPaymentProviderPlugin) getPaymentPluginApi(ExternalPaymentProviderPlugin.PLUGIN_NAME);
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — payment module (PaymentMethodProcessor)
  [Confidence score]: High

- [Kill-BR-0138] - Subscription events occurring at or after a CANCEL/EXPIRED event are discarded
  [Description]: Once a subscription has a CANCEL or EXPIRED event, that is treated as the definitive end of the subscription's event stream: any other active event dated on or after that cancellation/expiration date is removed from consideration, except the CREATE/TRANSFER 'start of time' event and the cancellation/expiration event itself, which are always kept. This guards against data-integrity bugs that produced out-of-order or duplicate cancellations. For example: given An event stream with a CANCEL event on 2026-01-15 and a stray CHANGE event dated 2026-01-20 (after cancellation) due to a data bug, when removeEverythingPastCancelEvent runs while rebuilding subscription state, then the 2026-01-20 CHANGE event is dropped from the effective stream; only events up to and including the 2026-01-15 CANCEL remain. Edge cases: Explicitly references known bugs killbill/killbill#897 and #619 in comments as the reason for this hardening.
  [Line Numbers]: 1093 to 1123
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  [Source Code]:
  ```java
      // Skip any event after a CANCELor EXPIRED event:
      //
      //  * DefaultSubscriptionDao#buildBundleSubscriptions may have added an out-of-order cancellation event (https://github.com/killbill/killbill/issues/897)
      //  * Hardening against data integrity issues where we have multiple active CANCEL (https://github.com/killbill/killbill/issues/619)
      //
      private void removeEverythingPastCancelEvent(final List<SubscriptionBaseEvent> inputEvents) {
          final SubscriptionBaseEvent canceledOrExpiredEvent = inputEvents.stream()
                                                                     .filter(input -> input.getType() == EventType.EXPIRED ||
                                                                                      (input.getType() == EventType.API_USER && ((ApiEvent) input).getApiEventType() == ApiEventType.CANCEL))
                                                                     .findFirst().orElse(null);
          if (canceledOrExpiredEvent == null) {
              return;
          }

          final Iterator<SubscriptionBaseEvent> it = inputEvents.iterator();
          while (it.hasNext()) {
              final SubscriptionBaseEvent input = it.next();
              if (!input.isActive()) {
                  continue;
              }

              if (input.getId().compareTo(canceledOrExpiredEvent.getId()) == 0) {
                  // Keep the cancellation event
              } else if (input.getType() == EventType.API_USER && (((ApiEvent) input).getApiEventType() == ApiEventType.TRANSFER || ((ApiEvent) input).getApiEventType() == ApiEventType.CREATE)) {
                  // Keep the initial event (SOT use-case)
              } else if (input.getEffectiveDate().compareTo(canceledOrExpiredEvent.getEffectiveDate()) >= 0) {
                  // Event to ignore past cancellation date
                  it.remove();
              }
          }
      }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — subscription module (DefaultSubscriptionBase)
  [Confidence score]: High

- [Kill-BR-0139] - Accounts are auto-parked on unrecoverable invoice-generation errors
  [Description]: When invoice generation for an account fails with a catalog, account, subscription, or unexpected runtime error (outside of a dry-run), the account is automatically tagged as 'parked', which then blocks further automatic invoicing until an operator investigates and un-parks it. Errors from an API call, or from a dry-run, do not auto-park unless the system is explicitly configured to always park on exceptions, with one exception: an InvoiceApiException carrying UNEXPECTED_ERROR always parks the account (outside dry-run) even from an API call. For example: given An account whose invoice generation throws a CatalogApiException during the nightly batch run, when processAccount catches the exception, then the account is tagged PARKED (idempotently) via ParkedAccountsManager.parkAccount, and an empty invoice list is returned instead of propagating the error. Edge cases: Plugin-aborted invoice generation (INVOICE_PLUGIN_API_ABORTED) does not park the account — treated as an intentional stop. Parameters: Config flag: invoiceConfig.isParkAccountsOnAllExceptions().
  [Line Numbers]: 443 to 487
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  [Source Code]:
  ```java
          } catch (final CatalogApiException e) {
              log.warn("Failed to generate invoice for accountId='{}'", accountId, e);
              if (!isDryRun && !isApiCall && invoiceConfig.isParkAccountsOnAllExceptions(context)) {
                  parkAccount(accountId, context);
              }
              return Collections.emptyList();
          } catch (final AccountApiException e) {
              log.warn("Failed to generate invoice for accountId='{}'", accountId, e);
              if (!isDryRun && !isApiCall && invoiceConfig.isParkAccountsOnAllExceptions(context)) {
                  parkAccount(accountId, context);
              }
              return Collections.emptyList();
          } catch (final SubscriptionBaseApiException e) {
              log.warn("Failed to generate invoice for accountId='{}'", accountId, e);
              if (!isDryRun && !isApiCall && invoiceConfig.isParkAccountsOnAllExceptions(context)) {
                  parkAccount(accountId, context);
              }
              return Collections.emptyList();
          } catch (final InvoiceApiException e) {
              if (e.getCode() == ErrorCode.INVOICE_PLUGIN_API_ABORTED.getCode()) {
                  log.info("Invoice generation aborted by plugin for accountId='{}', targetDate='{}'", accountId, inputTargetDate);
                  return Collections.emptyList();
              }
              log.warn("Failed to generate invoice for accountId='{}'", accountId, e);
              // Only case where we park even if this is from an API call and isParkAccountsOnAllExceptions is not explicitly set.
              if (e.getCode() == ErrorCode.UNEXPECTED_ERROR.getCode() && !isDryRun) {
                  parkAccount(accountId, context);
              } else if (!isDryRun && !isApiCall && invoiceConfig.isParkAccountsOnAllExceptions(context)) {
                  parkAccount(accountId, context);
              }
              // For non-API, non-dry-run paths the exception has been fully handled above (logged and parked if applicable).
              // Rethrowing would only cause duplicate logging in the async callers that catch and swallow it anyway.
              if (!isApiCall && !isDryRun) {
                  return Collections.emptyList();
              }
              throw e;
          } catch (RuntimeException e) { // The case of LockFailedException was handled prior we enter this method.
              log.warn("Failed to generate invoice for accountId='{}'", accountId, e);
              if (!isDryRun && !isApiCall && invoiceConfig.isParkAccountsOnAllExceptions(context)) {
                  parkAccount(accountId, context);
              }
              throw e;
          } catch (final NoSuchNotificationQueue e) { /* Dry run use cases */
              throw new InvoiceApiException(ErrorCode.UNEXPECTED_ERROR, "Failed to retrieve future notifications from notificationQ");
          }
  ```
  [Applies to]: Lifecycle rule (Priority P0) — invoice module (InvoiceDispatcher)
  [Confidence score]: High

- [Kill-BR-0140] - Account parking/unparking implemented via idempotent system tag
  [Description]: Parking and unparking an account is just adding/removing a well-known system tag on the account; parking is idempotent (adding the tag twice is not an error). For example: given An already-parked account, when parkAccount is called again, then the duplicate-tag error is silently swallowed and the account remains parked (no exception surfaces to the caller). Note (extraction review flagged this rule for SME confirmation): Citation was corrected by referee (The cited range (lines 28-42) covers only import statements, the class declaration, and the constructor (ParkedAccountsManager.java:35-44) — none of which implement parking/unparking logic.
  [Line Numbers]: 46 to 61
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/ParkedAccountsManager.java
  [Source Code]:
  ```java
      // Idempotent
      public void parkAccount(final UUID accountId, final InternalCallContext internalCallContext) throws TagApiException {
          log.warn("Parking account for accountId='{}'", accountId);
          try {
              tagApi.addTag(accountId, ObjectType.ACCOUNT, PARK_TAG_DEFINITION_ID, internalCallContext);
          } catch (final TagApiException e) {
              if (ErrorCode.TAG_ALREADY_EXISTS.getCode() != e.getCode()) {
                  throw e;
              }
          }
      }

      public void unparkAccount(final UUID accountId, final InternalCallContext internalCallContext) throws TagApiException {
          log.warn("Unparking account for accountId='{}'", accountId);
          tagApi.removeTag(accountId, ObjectType.ACCOUNT, PARK_TAG_DEFINITION_ID, internalCallContext);
      }
  ```
  [Applies to]: Lifecycle rule (Priority P1) — invoice module (ParkedAccountsManager)
  [Confidence score]: Medium

- [Kill-BR-0141] - Role permission update computes and applies a diff (add new, deactivate removed)
  [Description]: Updating a role's permission list is not a wholesale replace: existing permissions not present in the new list are individually deactivated, and only genuinely new permissions are inserted; supplying an empty permission list removes all existing permissions for that role. For example: given Role 'support' currently has permissions [A, B, C], when updateRoleDefinition('support', [B, D]) is called, then permission A and C are deactivated, permission D is newly created, and B is left untouched (not re-inserted). Edge cases: Empty incoming permissions list deactivates all existing permissions for the role.
  [Line Numbers]: 117 to 140
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/shiro/dao/DefaultUserDao.java
  [Source Code]:
  ```java
          });
      }

      @Override
      public void updateRoleDefinition(final String role, final List<String> permissions, final String createdBy) throws SecurityApiException {
          final DateTime createdDate = clock.getUTCNow();
          inTransactionWithExceptionHandling((handle, status) -> {
              final RolesPermissionsSqlDao rolesPermissionsSqlDao = handle.attach(RolesPermissionsSqlDao.class);
              final List<RolesPermissionsModelDao> existingPermissions = rolesPermissionsSqlDao.getByRoleName(role);
              // A empty list of permissions means we should remove all current permissions
              final Iterable<RolesPermissionsModelDao> toBeDeleted = existingPermissions.isEmpty() ?
                                                                     existingPermissions :
                                                                     existingPermissions.stream()
                                                                             .filter(input -> !permissions.contains(input.getPermission()))
                                                                             .collect(Collectors.toUnmodifiableList());

              final List<String> toBeAdded = permissions.stream()
                      .filter(s -> existingPermissions.stream().noneMatch(e -> e.getPermission().equals(s)))
                      .collect(Collectors.toUnmodifiableList());

              for (final RolesPermissionsModelDao d : toBeDeleted) {
                  rolesPermissionsSqlDao.unactiveEvent(d.getRecordId(), createdDate, createdBy);
              }
  ```
  [Applies to]: Lifecycle rule (Priority P2) — util module (DefaultUserDao)
  [Confidence score]: High

- [Kill-BR-0142] - In-arrear greedy billing mode
  [Description]: For arrears billing, the system normally waits until the end of a billing period before invoicing it. An optional 'greedy' mode lets the system bill a not-yet-complete period as soon as the target (as-of) date falls anywhere within it, rather than waiting for the period's end. For example: given InArrearMode=GREEDY, a monthly IN_ARREAR plan with firstBillingCycleDate=2024-01-01, and targetDate=2024-01-15 (mid-period), when the effective end date for invoicing is computed, then billing occurs for the period ending on the next proposed billing cycle date (e.g. 2024-02-01) rather than being deferred until that date is reached in a later invoice run. Edge cases: targetDate exactly equals endDate (subscription cancellation date) → always bills immediately regardless of greedy mode, to handle CHANGE/CANCELLATION events (BillingIntervalDetail.java:143-146, referencing issue #1907). Parameters: org.killbill.invoice.inArrear.mode / InArrearMode enum {DEFAULT, GREEDY} (util/src/main/java/org/killbill/billing/util/config/definition/InvoiceConfig.java, enum at InvoiceConfig.java:58-61).
  [Line Numbers]: 134 to 181
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/BillingIntervalDetail.java
  [Source Code]:
  ```java
      private void calculateInArrearEffectiveEndDate() {

          //
          // If we have an event mid-billing period (CHANGE, CANCELLATION) that aligns
          // with the target date, we bill immediately for the period instead of waiting for
          // the next billing cycle date, a.k.a firstBillingCycleDate. See #1907
          //
          // The following condition may be even more generic, but targetDate will typically align with the event so perhaps unnecessary:
          // if (endDate != null && targetDate.compareTo(endDate) >= 0 && targetDate.isBefore(cutoffStartDt)) { ...}
          if (endDate != null && targetDate.compareTo(endDate) == 0) {
              effectiveEndDate = targetDate;
              return;
          }

          final LocalDate cutoffStartDt = inArrearGreedy ? startDate : firstBillingCycleDate;
          if (targetDate.isBefore(cutoffStartDt)) {
              // Nothing to bill for, hasSomethingToBill will return false
              effectiveEndDate = null;
              return;

          }

          if (endDate != null && endDate.isBefore(firstBillingCycleDate)) {
              effectiveEndDate = endDate;
              return;
          }

          int numberOfPeriods = 0;
          LocalDate proposedDate = firstBillingCycleDate;
          LocalDate nextProposedDate = getFutureBillingDateFor(numberOfPeriods);
          while (!nextProposedDate.isAfter(targetDate)) {
              proposedDate = nextProposedDate;
              numberOfPeriods += 1;
              nextProposedDate = getFutureBillingDateFor(numberOfPeriods);
          }

          if (inArrearGreedy && !proposedDate.isEqual(targetDate)) {
              proposedDate = nextProposedDate;
          }

          final LocalDate cutoffEndDt = inArrearGreedy ? nextProposedDate : targetDate;
          // We honor the endDate as long as it does not go beyond our targetDate (by construction this cannot be after the nextProposedDate neither.
          if (endDate != null && !endDate.isAfter(cutoffEndDt)) {
              effectiveEndDate = endDate;
          } else {
              effectiveEndDate = proposedDate;
          }
      }
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (BillingIntervalDetail)
  [Confidence score]: High

- [Kill-BR-0143] - AUTO_PAY_OFF blocks only system-triggered payments
  [Description]: If an account is tagged AUTO_PAY_OFF, the system will not attempt to automatically collect payment against it (e.g. via retries or invoice-driven auto-pay) — but a payment explicitly initiated by the customer/API through the payment API still goes through. For example: given an account tagged AUTO_PAY_OFF and a system-initiated (non-API) payment attempt for a $75.00 invoice, when the prior-payment-control check runs, then the payment attempt is recorded in a pending 'auto pay off' queue and aborted (isApiPayment=false AND tag present → abort); if the same payment were instead flagged isApiPayment=true, this check is skipped entirely and payment proceeds. Parameters: ControlTagType.AUTO_PAY_OFF tag (see util/src/main/java/org/killbill/billing/util/tag/ControlTagType.java, referenced at InvoicePaymentControlPluginApi.java:747-749).
  [Line Numbers]: 730 to 745
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
      private boolean insert_AUTO_PAY_OFF_ifRequired(final PaymentControlContext paymentControlContext, final BigDecimal computedAmount) {
          if (paymentControlContext.isApiPayment() || !isAccountAutoPayOff(paymentControlContext.getAccountId(), paymentControlContext)) {
              return false;
          }
          final PluginAutoPayOffModelDao data = new PluginAutoPayOffModelDao(paymentControlContext.getAttemptPaymentId(), paymentControlContext.getPaymentExternalKey(), paymentControlContext.getTransactionExternalKey(),
                                                                             paymentControlContext.getAccountId(), PLUGIN_NAME,
                                                                             paymentControlContext.getPaymentId(),
                                                                             computedAmount, paymentControlContext.getCurrency(), CREATED_BY, paymentControlContext.getCreatedDate());
          controlDao.insertAutoPayOff(data);
          return true;
      }

      private boolean isAccountAutoPayOff(final UUID accountId, final CallContext callContext) {
          final List<Tag> accountTags = tagApi.getTagsForAccount(accountId, false, callContext);
          return ControlTagType.isAutoPayOff(accountTags.stream().map(Tag::getTagDefinitionId).collect(Collectors.toUnmodifiableList()));
      }
  ```
  [Line Numbers]: 376 to 380
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
              // Are we in auto-payoff (do the check as soon as possible -- https://github.com/killbill/killbill/issues/812)?
              if (insert_AUTO_PAY_OFF_ifRequired(paymentControlPluginContext, requestedAmount)) {
                  log.info("Aborting payment: invoiceId='{}' is AUTO_PAY_OFF", invoice.getId());
                  return new DefaultPriorPaymentControlResult(true);
              }
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: High

- [Kill-BR-0144] - Payment failure retry schedule
  [Description]: After a real payment decline (e.g. card declined, insufficient funds), the system retries on a fixed schedule of days after the failure, and gives up once the configured list of retry intervals is exhausted. For example: given org.killbill.payment.retry.days default '8,8,8' and this is the 2nd consecutive PAYMENT_FAILURE for a transaction (attemptsInState=2, so retryCount=1), when the next retry date is computed, then next retry = now + 8 days (the 2nd entry in the list, index 1); after the 3rd failure (retryCount=2, still < list size 3) another retry is scheduled +8 days; after the 4th failure (retryCount=3 >= list size 3) no further retry is scheduled (payment permanently fails). Edge cases: retryCount computed as max(attemptsInState-1, 0) (line 597). Parameters: org.killbill.payment.retry.days, default '8,8,8' (util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java:31-39). Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: InvoicePaymentControlPluginApi.java:592-609 computes retryCount = attemptsInState-1 against paymentConfig.getPaymentFailureRetryDays() (default "8,8,8" from PaymentConfig.java:31-39), returns now+retryDays[retryCount] while retryCount.
  [Line Numbers]: 592 to 609
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
      private DateTime getNextRetryDateForPaymentFailure(final List<PaymentTransactionModelDao> purchasedTransactions, final InternalCallContext internalContext) {

          DateTime result = null;
          final List<Integer> retryDays = paymentConfig.getPaymentFailureRetryDays(internalContext);
          final int attemptsInState = getNumberAttemptsInState(purchasedTransactions, TransactionStatus.PAYMENT_FAILURE);
          final int retryCount = (attemptsInState - 1) >= 0 ? (attemptsInState - 1) : 0;
          if (retryCount < retryDays.size()) {
              final int retryInDays;
              final DateTime nextRetryDate = internalContext.getCreatedDate();
              try {
                  retryInDays = retryDays.get(retryCount);
                  result = nextRetryDate.plusDays(retryInDays);
                  log.debug("Next retryDate={}, retryInDays={}, retryCount={}, now={}", result, retryInDays, retryCount, internalContext.getCreatedDate());
              } catch (final NumberFormatException ex) {
                  log.error("Could not get retry day for retry count {}", retryCount);
              }
          }
          return result;
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: Medium

- [Kill-BR-0145] - Plugin-failure retry backoff (suspected off-by-one)
  [Description]: After a transient plugin/gateway failure (not a real decline), the system retries with an exponentially growing delay starting from an initial wait, doubling (by default) on each subsequent attempt, up to a maximum number of attempts. For example: given initial retry = 300s, multiplier = 2, max attempts = 8 (all defaults), and this is the 4th consecutive PLUGIN_FAILURE (attemptsInState=4, retryAttempt=3), when the next retry delay is computed by the while(--remainingAttempts > 0) loop, then nbSec starts at 300 and the loop runs (retryAttempt-1)=2 times, doubling each time → 300 → 600 → 1200, so next retry = now + 1200s. Edge cases: retryAttempt=0 (1st failure) and retryAttempt=1 (2nd failure) BOTH resolve to the unmultiplied initial delay (300s) because the doubling loop's `--remainingAttempts > 0` guard skips execution for remainingAttempts in {0,1} — i.e. the first two retries occur at the same interval instead of following a clean 300/600/1200 progression. This looks like an off-by-one defect in the loop bound, not an intentional design.; retryAttempt >= maxAttempts (8) → no further retry, permanent failure; Suspected defect: The while(--remainingAttempts > 0) loop causes retry attempt #1 (0-indexed 0) and retry attempt #2 (0-indexed 1) to both use the un-multiplied initial delay, effectively wasting one exponential step versus the likely intended 300/600/1200/2400... sequence.. Parameters: org.killbill.payment.failure.retry.start.sec=300, org.killbill.payment.failure.retry.multiplier=2, org.killbill.payment.failure.retry.max.attempts=8 (PaymentConfig.java:41-59,82-90). Note (extraction review flagged this rule for SME confirmation): Is it intentional that the first two plugin-failure retries both wait exactly the initial 300s before the delay starts doubling, or should the second retry already be 600s?.
  [Line Numbers]: 612 to 628
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
      private DateTime getNextRetryDateForPluginFailure(final List<PaymentTransactionModelDao> purchasedTransactions, final InternalCallContext internalContext) {

          DateTime result = null;
          final int attemptsInState = getNumberAttemptsInState(purchasedTransactions, TransactionStatus.PLUGIN_FAILURE);
          final int retryAttempt = (attemptsInState - 1) >= 0 ? (attemptsInState - 1) : 0;

          if (retryAttempt < paymentConfig.getPluginFailureRetryMaxAttempts(internalContext)) {
              int nbSec = paymentConfig.getPluginFailureInitialRetryInSec(internalContext);
              int remainingAttempts = retryAttempt;
              while (--remainingAttempts > 0) {
                  nbSec = nbSec * paymentConfig.getPluginFailureRetryMultiplier(internalContext);
              }
              result = internalContext.getCreatedDate().plusSeconds(nbSec);
              log.debug("Next retryDate={}, retryAttempt={}, now={}", result, retryAttempt, internalContext.getCreatedDate());
          }
          return result;
      }
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: Medium

- [Kill-BR-0146] - Billing-blocking overdue state auto-disables invoicing
  [Description]: Entering an overdue state that disables entitlement auto-tags the account AUTO_INVOICING_OFF; clearing it removes the tag. For example: given Account moves to an overdue state with disableEntitlementAndChangesBlocked=true, when OverdueStateApplicator.apply runs, then The account is tagged AUTO_INVOICING_OFF. Edge cases: Tag removal tolerates TAG_DOES_NOT_EXIST. Parameters: blockBilling = isDisableEntitlementAndChangesBlocked().
  [Line Numbers]: 176 to 183
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      private void avoid_extra_credit_by_toggling_AUTO_INVOICE_OFF(final ImmutableAccountData account, final OverdueState previousOverdueState,
                                                                   final OverdueState nextOverdueState, final InternalCallContext context) throws OverdueApiException {
          if (isBlockBillingTransition(previousOverdueState, nextOverdueState)) {
              set_AUTO_INVOICE_OFF_on_blockedBilling(account.getId(), context);
          } else if (isUnblockBillingTransition(previousOverdueState, nextOverdueState)) {
              remove_AUTO_INVOICE_OFF_on_clear(account.getId(), context);
          }
      }
  ```
  [Line Numbers]: 237 to 253
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      private void set_AUTO_INVOICE_OFF_on_blockedBilling(final UUID accountId, final InternalCallContext context) throws OverdueApiException {
          try {
              tagApi.addTag(accountId, ObjectType.ACCOUNT, ControlTagType.AUTO_INVOICING_OFF.getId(), context);
          } catch (final TagApiException e) {
              throw new OverdueApiException(e);
          }
      }

      private void remove_AUTO_INVOICE_OFF_on_clear(final UUID accountId, final InternalCallContext context) throws OverdueApiException {
          try {
              tagApi.removeTag(accountId, ObjectType.ACCOUNT, ControlTagType.AUTO_INVOICING_OFF.getId(), context);
          } catch (final TagApiException e) {
              if (e.getCode() != ErrorCode.TAG_DOES_NOT_EXIST.getCode()) {
                  throw new OverdueApiException(e);
              }
          }
      }
  ```
  [Line Numbers]: 267 to 269
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      private boolean blockBilling(final OverdueState nextOverdueState) {
          return nextOverdueState.isDisableEntitlementAndChangesBlocked();
      }
  ```
  [Applies to]: Policy rule (Priority P0) — overdue module (OverdueStateApplicator)
  [Confidence score]: High

- [Kill-BR-0147] - Default overdue escalation tiers
  [Description]: Default config escalates by age of earliest unpaid invoice: 30 days blocks changes (OD1), 40 days disables entitlement/billing (OD2), 50 days same with distinct message (OD3); rechecked every 5 days. For example: given Earliest unpaid invoice is 40 days old, when Overdue check runs, then Account moves to OD2 with entitlement and billing blocked. Edge cases: Overridable per tenant. Parameters: OD1=30d, OD2=40d, OD3=50d, recheck=5d. Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (The Given/When/Then is mechanically accurate against overdue.xml:19-61 (OD1=30d blockChanges only; OD2=40d and OD3=50d add disableEntitlementAndChangesBlocked=true; autoReevaluationInterval=5d for all three), but the "Plain English" framing that this is the "d.
  [Line Numbers]: 19 to 61
  [Source Code File Name]: profiles/killbill/src/main/resources/overdue.xml
  [Source Code]:
  ```xml
  <overdueConfig>
     <accountOverdueStates>
         <state name="OD3">
             <condition>
                 <timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
                     <unit>DAYS</unit><number>50</number>
                 </timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
             </condition>
             <externalMessage>Reached OD3</externalMessage>
             <blockChanges>true</blockChanges>
             <disableEntitlementAndChangesBlocked>true</disableEntitlementAndChangesBlocked>
             <autoReevaluationInterval>
                 <unit>DAYS</unit><number>5</number>
             </autoReevaluationInterval>
         </state>
         <state name="OD2">
             <condition>
                 <timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
                     <unit>DAYS</unit><number>40</number>
                 </timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
             </condition>
             <externalMessage>Reached OD2</externalMessage>
             <blockChanges>true</blockChanges>
             <disableEntitlementAndChangesBlocked>true</disableEntitlementAndChangesBlocked>
             <autoReevaluationInterval>
                 <unit>DAYS</unit><number>5</number>
             </autoReevaluationInterval>
         </state>
         <state name="OD1">
             <condition>
                 <timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
                     <unit>DAYS</unit><number>30</number>
                 </timeSinceEarliestUnpaidInvoiceEqualsOrExceeds>
             </condition>
             <externalMessage>Reached OD1</externalMessage>
             <blockChanges>true</blockChanges>
             <disableEntitlementAndChangesBlocked>false</disableEntitlementAndChangesBlocked>
             <autoReevaluationInterval>
                 <unit>DAYS</unit><number>5</number>
             </autoReevaluationInterval>
         </state>
     </accountOverdueStates>
  </overdueConfig>
  ```
  [Applies to]: Policy rule (Priority P1) — profiles module (overdue)
  [Confidence score]: Medium

- [Kill-BR-0148] - Plan-change alignment determines the effective phase start date
  [Description]: When a customer changes plans, the catalog's alignment rule decides whether the new plan's phase clock restarts at the original subscription start, the bundle start, or the moment of the change itself. For example: given A subscription changing from Plan A to Plan B where the matching CaseChangePlanAlignment rule resolves to CHANGE_OF_PLAN, when the change becomes effective, then The new plan's phase timeline starts at the change's effective date (lastOrCurrentChangeEffectiveDate) rather than the original subscription/bundle start date; for START_OF_SUBSCRIPTION/START_OF_BUNDLE alignment the new plan instead re-uses the original subscription or bundle start date, and CHANGE_OF_PRICELIST is explicitly unimplemented (throws). Edge cases: CHANGE_OF_PRICELIST alignment throws SubscriptionBaseError('Not implemented yet').
  [Line Numbers]: 239 to 282
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java
  [Source Code]:
  ```java
      private TimedPhase getTimedPhaseOnChange(final DateTime subscriptionStartDate,
                                               final DateTime bundleStartDate,
                                               final PlanPhase currentPhase,
                                               final Plan currentPlan,
                                               final Plan nextPlan,
                                               final DateTime effectiveDate,
                                               final DateTime catalogEffectiveDate,
                                               final DateTime lastOrCurrentChangeEffectiveDate,
                                               final PhaseType originalInitialPhase,
                                               @Nullable final PhaseType newPlanInitialPhaseType,
                                               final WhichPhase which,
                                               final SubscriptionCatalog catalog,
                                               final InternalTenantContext context) throws CatalogApiException, SubscriptionBaseApiException {
          final PlanPhaseSpecifier fromPlanPhaseSpecifier = new PlanPhaseSpecifier(currentPlan.getName(),
                                                                                   currentPhase.getPhaseType());

          final PlanSpecifier toPlanSpecifier = new PlanSpecifier(nextPlan.getName());
          final PhaseType initialPhase;
          final DateTime planStartDate;
          final PlanAlignmentChange alignment = catalog.getPlanChangeResult(fromPlanPhaseSpecifier, toPlanSpecifier, catalogEffectiveDate).getAlignment();
          switch (alignment) {
              case START_OF_SUBSCRIPTION:
                  planStartDate = subscriptionStartDate;
                  initialPhase = newPlanInitialPhaseType != null ? newPlanInitialPhaseType :
                                 (isPlanContainPhaseType(nextPlan, originalInitialPhase) ? originalInitialPhase : null);
                  break;
              case START_OF_BUNDLE:
                  planStartDate = bundleStartDate;
                  initialPhase = newPlanInitialPhaseType != null ? newPlanInitialPhaseType :
                                 (isPlanContainPhaseType(nextPlan, originalInitialPhase) ? originalInitialPhase : null);
                  break;
              case CHANGE_OF_PLAN:
                  planStartDate = lastOrCurrentChangeEffectiveDate;
                  initialPhase = newPlanInitialPhaseType;
                  break;
              case CHANGE_OF_PRICELIST:
                  throw new SubscriptionBaseError(String.format("Not implemented yet %s", alignment));
              default:
                  throw new SubscriptionBaseError(String.format("Unknown PlanAlignmentChange %s", alignment));
          }

          final List<TimedPhase> timedPhases = getPhaseAlignments(nextPlan, initialPhase, planStartDate, context);
          return getTimedPhase(timedPhases, effectiveDate, which);
      }
  ```
  [Applies to]: Policy rule (Priority P0) — subscription module (PlanAligner)
  [Confidence score]: High

- [Kill-BR-0149] - Account credit is distributed across unpaid invoices oldest-first
  [Description]: When applying spare account credit across multiple unpaid invoices, the oldest invoice (by invoice date) is paid down first, then the next oldest, until the credit runs out. For example: given Account credit of $50.00 and two unpaid COMMITTED invoices: Inv#1 dated 2026-01-01 for $30, Inv#2 dated 2026-02-01 for $40, when useExistingCBAFromTransaction runs, then Inv#1 is fully paid with $30 of credit first, leaving $20, which is then applied to Inv#2 (partially settling it); processing stops as soon as remaining credit reaches zero.
  [Line Numbers]: 185 to 217
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  [Source Code]:
  ```java
      // Distribute account CBA across all COMMITTED unpaid invoices
      private List<InvoiceItemModelDao> useExistingCBAFromTransaction(final BigDecimal accountCBA,
                                                                      final List<CustomField> invoiceCustomFields,
                                                                      final List<Tag> invoicesTags,
                                                                      final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                                      final InternalCallContext context) throws InvoiceApiException, EntityPersistenceException {
          if (accountCBA.compareTo(BigDecimal.ZERO) <= 0) {
              return Collections.emptyList();
          }

          final List<InvoiceItemModelDao> result = new ArrayList<>();

          // PERF: Computing the invoice balance is difficult to do in the DB, so we effectively need to retrieve all invoices on the account and filter the unpaid ones in memory.
          // This should be infrequent though because of the account CBA check above.
          final List<InvoiceModelDao> allInvoices = invoiceDaoHelper.getAllInvoicesByAccountFromTransaction(false, true, invoiceCustomFields, invoicesTags, entitySqlDaoWrapperFactory, context);
          final List<InvoiceModelDao> unpaidInvoices = invoiceDaoHelper.getUnpaidInvoicesByAccountFromTransaction(allInvoices, null, null);
          // We order the same os BillingStateCalculator-- should really share the comparator
          final List<InvoiceModelDao> orderedUnpaidInvoices = unpaidInvoices.stream()
                  .sorted(Comparator.comparing(InvoiceModelDao::getInvoiceDate))
                  .collect(Collectors.toUnmodifiableList());

          BigDecimal remainingAccountCBA = accountCBA;
          for (final InvoiceModelDao unpaidInvoice : orderedUnpaidInvoices) {
              final InvoiceItemModelDao cbaItem = computeCBAComplexityAndCreateCBAItem(remainingAccountCBA, unpaidInvoice, entitySqlDaoWrapperFactory, context);
              if (cbaItem != null) {
                  result.add(cbaItem);
                  remainingAccountCBA = remainingAccountCBA.add(cbaItem.getAmount());
              }
              if (remainingAccountCBA.compareTo(BigDecimal.ZERO) <= 0) {
                  break;
              }
          }
          return result;
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (CBADao)
  [Confidence score]: High

- [Kill-BR-0150] - Account-level BCD alignment falls back to subscription alignment until the account BCD is set
  [Description]: A catalog rule saying 'align billing to the account's bill-cycle-day' is treated as 'align to the subscription's own bill-cycle-day' for as long as the account doesn't have a bill-cycle-day yet (e.g. the very first subscription on a brand-new account). For example: given BillingAlignment.ACCOUNT and accountBillCycleDayLocal == 0 (unset), when resolveEffectiveBillingAlignment is called, then BillingAlignment.SUBSCRIPTION is returned instead of ACCOUNT.
  [Line Numbers]: 50 to 54
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  [Source Code]:
  ```java
      public static BillingAlignment resolveEffectiveBillingAlignment(final BillingAlignment alignment, final int accountBillCycleDayLocal) {
          if (alignment == BillingAlignment.ACCOUNT && accountBillCycleDayLocal == 0) {
              return BillingAlignment.SUBSCRIPTION;
          }
          return alignment;
  ```
  [Applies to]: Policy rule (Priority P1) — util module (BillCycleDayCalculator)
  [Confidence score]: High

- [Kill-BR-0151] - Blocked billing periods shorter than one day are not disabled
  [Description]: If a billing-block is toggled on and back off again within less than a full day, it's ignored — no billing gap is created for it. For example: given A BlockingState becomes block-billing at 2026-06-01T10:00 and un-blocks at 2026-06-01T18:00 (same calendar day, less than 1 day apart), when addDisabledDuration evaluates whether to record this window, then No DisabledDuration is recorded (Days.daysBetween(...) < 1), so no billing gap/disable event is created for that period; an open-ended block (disableDurationEndDate == null) is always recorded regardless of length. Parameters: Minimum disable duration threshold: 1 day (hardcoded).
  [Line Numbers]: 62 to 68
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java
  [Source Code]:
  ```java
      private void addDisabledDuration(final BlockingState firstBlockingState, @Nullable final DateTime disableDurationEndDate) {

          if (disableDurationEndDate == null || Days.daysBetween(firstBlockingState.getEffectiveDate(), disableDurationEndDate).getDays() >= 1) {
              // Don't disable for periods less than a day (see https://github.com/killbill/killbill/issues/267)
              result.add(new DisabledDuration(firstBlockingState.getEffectiveDate(), disableDurationEndDate));
          }
      }
  ```
  [Applies to]: Policy rule (Priority P2) — junction module (BlockingStateService)
  [Confidence score]: High

- [Kill-BR-0152] - Add-on creation eligibility against base subscription
  [Description]: An add-on can only be attached to a base subscription that is active (or pending-active as of the request date), whose base product doesn't already include that add-on for free, and whose base product's catalog entry explicitly allows that add-on. For example: given A base subscription in state CANCELLED, or PENDING with a start date after the requested add-on date, when A caller tries to create an add-on subscription against that base, then The call is rejected with SUB_CREATE_AO_BP_NON_ACTIVE; separately, if the add-on's product is already in the base product's 'included' list it is rejected with SUB_CREATE_AO_ALREADY_INCLUDED, and if it's not in the base product's 'available' list it is rejected with SUB_CREATE_AO_NOT_AVAILABLE. Parameters: Included/available add-on product lists are catalog-defined (per base Product), not hardcoded. Note (extraction review flagged this rule for SME confirmation): P0 panel doubts spec fidelity: P0 classification is justified: this gate is the sole guard preventing an add-on from being attached to a base subscription that is cancelled/not-yet-active, already includes the add-on for free, or isn't catalog-permitted for that base product — all of which directly affect what gets inv.
  [Line Numbers]: 36 to 77
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java
  [Source Code]:
  ```java
      public void checkAddonCreationRights(final SubscriptionBase baseSubscription, final Plan targetAddOnPlan, final DateTime requestedDate, final InternalTenantContext context)
              throws SubscriptionBaseApiException {
          if (baseSubscription.getState() == EntitlementState.CANCELLED ||
              (baseSubscription.getState() == EntitlementState.PENDING && context.toLocalDate(baseSubscription.getStartDate()).compareTo(context.toLocalDate(requestedDate)) < 0)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_BP_NON_ACTIVE, targetAddOnPlan.getName());
          }

          final Plan currentOrPendingPlan = baseSubscription.getCurrentOrPendingPlan();
          final Product baseProduct = currentOrPendingPlan.getProduct();
          if (isAddonIncluded(baseProduct, targetAddOnPlan)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_ALREADY_INCLUDED,
                                                     targetAddOnPlan.getName(), currentOrPendingPlan.getProduct().getName());
          }

          if (!isAddonAvailable(baseProduct, targetAddOnPlan)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_AO_NOT_AVAILABLE,
                                                     targetAddOnPlan.getName(), currentOrPendingPlan.getProduct().getName());
          }
      }

      public boolean isAddonAvailable(final Product baseProduct, final Plan targetAddOnPlan) {
          final Product targetAddonProduct = targetAddOnPlan.getProduct();
          final Collection<Product> availableAddOns = baseProduct.getAvailable();

          for (final Product curAv : availableAddOns) {
              if (curAv.getName().equals(targetAddonProduct.getName())) {
                  return true;
              }
          }
          return false;
      }

      public boolean isAddonIncluded(final Product baseProduct, final Plan targetAddOnPlan) {
          final Product targetAddonProduct = targetAddOnPlan.getProduct();
          final Collection<Product> includedAddOns = baseProduct.getIncluded();
          for (final Product curAv : includedAddOns) {
              if (curAv.getName().equals(targetAddonProduct.getName())) {
                  return true;
              }
          }
          return false;
      }
  ```
  [Applies to]: Policy rule (Priority P0) — subscription module (AddonUtils)
  [Confidence score]: Medium

- [Kill-BR-0153] - Subscription state guard for plan changes
  [Description]: A plan change is blocked if the subscription is CANCELLED, EXPIRED, has a future cancellation already scheduled, has a future expiry before the requested effective date, or the requested date is before the subscription's start date. For example: given A subscription with a pending future-dated cancellation already scheduled, when changePlanWithRequestedDate or changePlanWithPolicy is called, then The call is rejected with SUB_CHANGE_NON_ACTIVE (bad state / date-before-start) or SUB_CHANGE_FUTURE_CANCELLED or SUB_CHANGE_FUTURE_EXPIRED as appropriate.
  [Line Numbers]: 835 to 850
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      private void validateSubscriptionStateForChangePlan(final DefaultSubscriptionBase subscription, @Nullable final DateTime effectiveDate) throws SubscriptionBaseApiException {

          final EntitlementState currentState = subscription.getState();
          if (currentState == EntitlementState.CANCELLED ||
              currentState == EntitlementState.EXPIRED ||
              // We don't look for PENDING because as long as change is after startDate, we want to allow it.
              effectiveDate != null && effectiveDate.compareTo(subscription.getStartDate()) < 0) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_NON_ACTIVE, subscription.getId(), currentState);
          }
          if (subscription.isFutureCancelled()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_FUTURE_CANCELLED, subscription.getId());
          }
          if (effectiveDate != null && subscription.getFutureExpiryDate() != null && subscription.getFutureExpiryDate().isBefore(effectiveDate)) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CHANGE_FUTURE_EXPIRED, subscription.getId());
          }
      }
  ```
  [Applies to]: Policy rule (Priority P0) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0154] - Cancellation of an EXPIRED subscription is blocked
  [Description]: A subscription that has already reached the EXPIRED state cannot be cancelled again through any of the three cancel entry points (default policy, requested date, or explicit policy). For example: given A subscription in state EXPIRED, when cancel(), cancelWithRequestedDate(), or cancelWithPolicy() is called on it, then The call is rejected with SUB_CANCEL_BAD_STATE.
  [Line Numbers]: 191 to 194
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      public boolean cancel(final DefaultSubscriptionBase subscription, final CallContext context) throws SubscriptionBaseApiException {
          if (subscription.getState() == EntitlementState.EXPIRED) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), subscription.getState());
          }
  ```
  [Line Numbers]: 213 to 217
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      @Override
      public boolean cancelWithRequestedDate(final DefaultSubscriptionBase subscription, final DateTime requestedDateWithMs, final CallContext context) throws SubscriptionBaseApiException {
          if (subscription.getState() == EntitlementState.EXPIRED) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), subscription.getState());
          }
  ```
  [Line Numbers]: 230 to 234
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  [Source Code]:
  ```java
      @Override
      public boolean cancelWithPolicy(final DefaultSubscriptionBase subscription, final BillingActionPolicy policy, final CallContext context) throws SubscriptionBaseApiException {
          if (subscription.getState() == EntitlementState.EXPIRED) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_CANCEL_BAD_STATE, subscription.getId(), subscription.getState());
          }
  ```
  [Applies to]: Policy rule (Priority P1) — subscription module (DefaultSubscriptionBaseApiService)
  [Confidence score]: High

- [Kill-BR-0155] - Illegal plan-change policy blocks the change
  [Description]: The catalog can define change-plan rules that mark specific from-plan/to-plan combinations as ILLEGAL; such combinations are rejected outright regardless of any billing/alignment considerations. For example: given A catalog change-plan rule maps a given (from,to) combination to policy ILLEGAL, when getPlanChangeResult() is invoked for that combination, then An IllegalPlanChange exception is thrown before alignment is even computed. Parameters: changeCase rules are catalog-XML defined per tenant/catalog version.
  [Line Numbers]: 143 to 164
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      @Override
      public PlanChangeResult getPlanChangeResult(final PlanPhaseSpecifier from, final PlanSpecifier to) throws CatalogApiException {

          final DefaultPriceList toPriceList = to.getPriceListName() != null ?
                                               (DefaultPriceList) root.findPriceList(to.getPriceListName()) :
                                               findPriceList(from);

          // If we use old scheme {product, billingPeriod, pricelist}, ensure pricelist is correct
          // (Pricelist may be null because if it is unspecified this is the principal use-case)
          final PlanSpecifier toWithPriceList = to.getPlanName() == null ?
                                                new PlanSpecifier(to.getProductName(), to.getBillingPeriod(), toPriceList.getName()) :
                                                to;

          final BillingActionPolicy policy = getPlanChangePolicy(from, toWithPriceList);
          if (policy == BillingActionPolicy.ILLEGAL) {
              throw new IllegalPlanChange(from, toWithPriceList);
          }

          final PlanAlignmentChange alignment = getPlanChangeAlignment(from, toWithPriceList);

          return new PlanChangeResult(toPriceList, policy, alignment);
      }
  ```
  [Applies to]: Policy rule (Priority P0) — catalog module (DefaultPlanRules)
  [Confidence score]: High

- [Kill-BR-0156] - Blocking-state cascade blocks change/entitlement/billing actions (OR across levels)
  [Description]: An action (plan change, entitlement change, or billing) is blocked if ANY of the account-level, bundle-level, or subscription-level blocking states for that subscription say to block it; blocking flags are combined with logical OR across all three levels, so a block at any level propagates down. For example: given An account-level BlockingState with isBlockBilling()=true and a subscription with no subscription- or bundle-level blocks, when checkBlockedBilling is called for that subscription, then A BlockingApiException BLOCK_BLOCKED_ACTION is thrown for the subscription, purely because of the inherited account-level block. Parameters: None (boolean OR aggregation).
  [Line Numbers]: 44 to 91
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  [Source Code]:
  ```java
      public static class DefaultBlockingAggregator implements BlockingAggregator {

          private boolean blockChange = false;
          private boolean blockEntitlement = false;
          private boolean blockBilling = false;

          public void or(final BlockingState state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }

          public void or(final DefaultBlockingAggregator state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }

          @Override
          public boolean isBlockChange() {
              return blockChange;
          }

          @Override
          public boolean isBlockEntitlement() {
              return blockEntitlement;
          }

          @Override
          public boolean isBlockBilling() {
              return blockBilling;
          }

          @Override
          public String toString() {
              final StringBuilder sb = new StringBuilder("DefaultBlockingAggregator{");
              sb.append("blockChange=").append(blockChange);
              sb.append(", blockEntitlement=").append(blockEntitlement);
              sb.append(", blockBilling=").append(blockBilling);
              sb.append('}');
              return sb.toString();
          }
  ```
  [Line Numbers]: 137 to 188
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  [Source Code]:
  ```java
      private DefaultBlockingAggregator getBlockedStateSubscriptionId(final UUID subscriptionId, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          try {
              final UUID bundleId = subscriptionApi.getBundleIdFromSubscriptionId(subscriptionId, context);
              return getBlockedStateSubscription(bundleId, subscriptionId, upToDate, context);
          } catch (final SubscriptionBaseApiException e) {
              throw new BlockingApiException(e, ErrorCode.fromCode(e.getCode()));
          }
      }

      private DefaultBlockingAggregator getBlockedStateSubscription(@Nullable final UUID bundleId, @Nullable final UUID subscriptionId, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          final DefaultBlockingAggregator result = new DefaultBlockingAggregator();
          if (subscriptionId != null) {
              final DefaultBlockingAggregator subscriptionState = getBlockedStateForId(subscriptionId, BlockingStateType.SUBSCRIPTION, upToDate, context);
              if (subscriptionState != null) {
                  result.or(subscriptionState);
              }
              if (bundleId != null) {
                  // Recursive call to also fetch account state
                  result.or(getBlockedStateBundleId(bundleId, upToDate, context));
              }
          }
          return result;
      }

      private DefaultBlockingAggregator getBlockedStateBundleId(final UUID bundleId, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          try {
              final UUID accountId = subscriptionApi.getAccountIdFromBundleId(bundleId, context);
              return getBlockedStateBundle(accountId, bundleId, upToDate, context);
          } catch (final SubscriptionBaseApiException e) {
              throw new BlockingApiException(e, ErrorCode.fromCode(e.getCode()));
          }
      }

      private DefaultBlockingAggregator getBlockedStateBundle(final UUID accountId, final UUID bundleId, final DateTime upToDate, final InternalTenantContext context) {
          final DefaultBlockingAggregator result = getBlockedStateAccountId(accountId, upToDate, context);
          final DefaultBlockingAggregator bundleState = getBlockedStateForId(bundleId, BlockingStateType.SUBSCRIPTION_BUNDLE, upToDate, context);
          if (bundleState != null) {
              result.or(bundleState);
          }
          return result;
      }

      private DefaultBlockingAggregator getBlockedStateAccount(final Account account, final DateTime upToDate, final InternalTenantContext context) {
          if (account != null) {
              return getBlockedStateForId(account.getId(), BlockingStateType.ACCOUNT, upToDate, context);
          }
          return new DefaultBlockingAggregator();
      }

      private DefaultBlockingAggregator getBlockedStateAccountId(final UUID accountId, final DateTime upToDate, final InternalTenantContext context) {
          return getBlockedStateForId(accountId, BlockingStateType.ACCOUNT, upToDate, context);
      }
  ```
  [Line Numbers]: 218 to 248
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  [Source Code]:
  ```java
      public void checkBlockedChange(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }

      @Override
      public void checkBlockedEntitlement(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }

      @Override
      public void checkBlockedBilling(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }
  ```
  [Line Numbers]: 25 to 38
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java
  [Source Code]:
  ```java
      public DefaultBlockingAggregator getBlockedState(final Iterable<BlockingState> accountEntitlementStates,
                                                       final Iterable<BlockingState> bundleEntitlementStates,
                                                       final Iterable<BlockingState> subscriptionEntitlementStates) {
          final DefaultBlockingAggregator result = getBlockedState(subscriptionEntitlementStates);
          result.or(getBlockedState(bundleEntitlementStates));
          result.or(getBlockedState(accountEntitlementStates));
          return result;
      }

      public DefaultBlockingAggregator getBlockedState(final Iterable<BlockingState> currentBlockableStatePerService) {
          final DefaultBlockingAggregator result = new DefaultBlockingAggregator();
          for (final BlockingState cur : currentBlockableStatePerService) {
              result.or(cur);
          }
  ```
  [Applies to]: Policy rule (Priority P0) — entitlement module (DefaultBlockingChecker, StatelessBlockingChecker)
  [Confidence score]: High

- [Kill-BR-0157] - Add-on creation blocked when base subscription is cancelled, not-yet-pending, or itself blocked
  [Description]: Before creating an add-on entitlement, the system verifies the base subscription is not cancelled, not still-pending-in-the-future relative to the requested add-on date, and not itself blocked for change or entitlement actions. For example: given A base entitlement that is PENDING with an effective start date after the requested add-on's effective date, when A caller tries to add an add-on entitlement to that bundle, then The call fails with SUB_GET_NO_SUCH_BASE_SUBSCRIPTION (treated as if there's no valid base), or with a BLOCK_BLOCKED_ACTION exception if the base is change/entitlement-blocked.
  [Line Numbers]: 530 to 544
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java
  [Source Code]:
  ```java
      private void preCheckAddEntitlement(final UUID bundleId, final DateTime entitlementRequestedDate, final BaseEntitlementWithAddOnsSpecifier baseEntitlementWithAddOnsSpecifier, final EventsStream eventsStreamForBaseSubscription) throws EntitlementApiException {
          if (eventsStreamForBaseSubscription.isEntitlementCancelled() ||
              (eventsStreamForBaseSubscription.isEntitlementPending() &&
               (baseEntitlementWithAddOnsSpecifier.getEntitlementEffectiveDate() == null ||
                baseEntitlementWithAddOnsSpecifier.getEntitlementEffectiveDate().compareTo(eventsStreamForBaseSubscription.getEntitlementEffectiveStartDateTime()) < 0))) {
              throw new EntitlementApiException(ErrorCode.SUB_GET_NO_SUCH_BASE_SUBSCRIPTION, bundleId);
          }

          // Check the base entitlement state is not blocked
          if (eventsStreamForBaseSubscription.isBlockChange(entitlementRequestedDate)) {
              throw new EntitlementApiException(new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, BlockingChecker.ACTION_CHANGE, BlockingChecker.TYPE_SUBSCRIPTION, eventsStreamForBaseSubscription.getEntitlementId().toString()));
          } else if (eventsStreamForBaseSubscription.isBlockEntitlement(entitlementRequestedDate)) {
              throw new EntitlementApiException(new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, BlockingChecker.ACTION_ENTITLEMENT, BlockingChecker.TYPE_SUBSCRIPTION, eventsStreamForBaseSubscription.getEntitlementId().toString()));
          }
      }
  ```
  [Applies to]: Policy rule (Priority P0) — entitlement module (DefaultEntitlementApi)
  [Confidence score]: High

- [Kill-BR-0158] - Default payment method cannot be deleted without explicit override
  [Description]: Deleting an account's default payment method is refused unless the caller explicitly passes deleteDefaultPaymentMethodWithAutoPayOff or forceDefaultPaymentMethodDeletion; if deleted via the auto-pay-off option and the account isn't already on AUTO_PAY_OFF, the system automatically switches the account to AUTO_PAY_OFF before removing it. For example: given An account whose default payment method equals the one being deleted, called with both override flags = false, when deletedPaymentMethod is invoked, then The call fails with PAYMENT_DEL_DEFAULT_PAYMENT_METHOD; if deleteDefaultPaymentMethodWithAutoPayOff=true instead and the account isn't already AUTO_PAY_OFF, the account is switched to AUTO_PAY_OFF as a side effect before the payment method is removed. Parameters: None (two boolean override flags).
  [Line Numbers]: 503 to 528
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  [Source Code]:
  ```java
      public void deletedPaymentMethod(final Account account, final UUID paymentMethodId,
                                       final boolean deleteDefaultPaymentMethodWithAutoPayOff,
                                       final boolean forceDefaultPaymentMethodDeletion,
                                       final Iterable<PluginProperty> properties, final CallContext callContext, final InternalCallContext context)
              throws PaymentApiException {
          try {
              new WithAccountLock<Void, PaymentApiException>(paymentConfig).processAccountWithLock(locker, account.getId(), new DispatcherCallback<PluginDispatcherReturnType<Void>, PaymentApiException>() {

                  @Override
                  public PluginDispatcherReturnType<Void> doOperation() throws PaymentApiException {
                      @SuppressWarnings("unused")
                      final PaymentMethodModelDao paymentMethodModel = getPaymentMethodById(paymentMethodId, false, context);

                      try {
                          // Note: account.getPaymentMethodId() may be null
                          if (paymentMethodId.equals(account.getPaymentMethodId())) {
                              if (!deleteDefaultPaymentMethodWithAutoPayOff && !forceDefaultPaymentMethodDeletion) {
                                  throw new PaymentApiException(ErrorCode.PAYMENT_DEL_DEFAULT_PAYMENT_METHOD, account.getId());
                              } else {
                                  if (deleteDefaultPaymentMethodWithAutoPayOff && !isAccountAutoPayOff(account.getId(), context)) {
                                      log.info("Setting AUTO_PAY_OFF on accountId='{}' because of default payment method deletion", account.getId());
                                      setAccountAutoPayOff(account.getId(), context);
                                  }
                                  accountInternalApi.removePaymentMethod(account.getId(), context);
                              }
                          }
  ```
  [Applies to]: Policy rule (Priority P0) — payment module (PaymentMethodProcessor)
  [Confidence score]: High

- [Kill-BR-0159] - Blocking-state flags OR-aggregate across account, bundle, and subscription levels
  [Description]: Whether a change, entitlement action, or billing action is blocked for a subscription is computed by OR-ing the blockChange/blockEntitlement/blockBilling flags of the most recent blocking state at the subscription level, its bundle level, and its account level — if any one of the three levels blocks the action, the action is blocked, regardless of the other levels. For example: given A subscription itself is not blocked, but its account carries a blocking state with blockBilling=true (e.g. from overdue), when checkBlockedBilling is called for that subscription, then BlockingApiException BLOCK_BLOCKED_ACTION is thrown, because the account-level block propagates down.
  [Line Numbers]: 44 to 124
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  [Source Code]:
  ```java
      public static class DefaultBlockingAggregator implements BlockingAggregator {

          private boolean blockChange = false;
          private boolean blockEntitlement = false;
          private boolean blockBilling = false;

          public void or(final BlockingState state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }

          public void or(final DefaultBlockingAggregator state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }

          @Override
          public boolean isBlockChange() {
              return blockChange;
          }

          @Override
          public boolean isBlockEntitlement() {
              return blockEntitlement;
          }

          @Override
          public boolean isBlockBilling() {
              return blockBilling;
          }

          @Override
          public String toString() {
              final StringBuilder sb = new StringBuilder("DefaultBlockingAggregator{");
              sb.append("blockChange=").append(blockChange);
              sb.append(", blockEntitlement=").append(blockEntitlement);
              sb.append(", blockBilling=").append(blockBilling);
              sb.append('}');
              return sb.toString();
          }

          @Override
          public boolean equals(final Object o) {
              if (this == o) {
                  return true;
              }
              if (!(o instanceof DefaultBlockingAggregator)) {
                  return false;
              }

              final DefaultBlockingAggregator that = (DefaultBlockingAggregator) o;

              if (blockBilling != that.blockBilling) {
                  return false;
              }
              if (blockChange != that.blockChange) {
                  return false;
              }
              if (blockEntitlement != that.blockEntitlement) {
                  return false;
              }

              return true;
          }

          @Override
          public int hashCode() {
              int result = (blockChange ? 1 : 0);
              result = 31 * result + (blockEntitlement ? 1 : 0);
              result = 31 * result + (blockBilling ? 1 : 0);
              return result;
          }
      }
  ```
  [Line Numbers]: 161 to 248
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  [Source Code]:
  ```java
      private DefaultBlockingAggregator getBlockedStateBundleId(final UUID bundleId, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          try {
              final UUID accountId = subscriptionApi.getAccountIdFromBundleId(bundleId, context);
              return getBlockedStateBundle(accountId, bundleId, upToDate, context);
          } catch (final SubscriptionBaseApiException e) {
              throw new BlockingApiException(e, ErrorCode.fromCode(e.getCode()));
          }
      }

      private DefaultBlockingAggregator getBlockedStateBundle(final UUID accountId, final UUID bundleId, final DateTime upToDate, final InternalTenantContext context) {
          final DefaultBlockingAggregator result = getBlockedStateAccountId(accountId, upToDate, context);
          final DefaultBlockingAggregator bundleState = getBlockedStateForId(bundleId, BlockingStateType.SUBSCRIPTION_BUNDLE, upToDate, context);
          if (bundleState != null) {
              result.or(bundleState);
          }
          return result;
      }

      private DefaultBlockingAggregator getBlockedStateAccount(final Account account, final DateTime upToDate, final InternalTenantContext context) {
          if (account != null) {
              return getBlockedStateForId(account.getId(), BlockingStateType.ACCOUNT, upToDate, context);
          }
          return new DefaultBlockingAggregator();
      }

      private DefaultBlockingAggregator getBlockedStateAccountId(final UUID accountId, final DateTime upToDate, final InternalTenantContext context) {
          return getBlockedStateForId(accountId, BlockingStateType.ACCOUNT, upToDate, context);
      }

      private DefaultBlockingAggregator getBlockedStateForId(@Nullable final UUID blockableId, final BlockingStateType blockingStateType, final DateTime upToDate, final InternalTenantContext context) {
          // Last states across services
          final List<BlockingState> blockableState;
          if (blockableId != null) {
              blockableState = dao.getBlockingState(blockableId, blockingStateType, upToDate, context);
          } else {
              blockableState = Collections.emptyList();
          }
          return statelessBlockingChecker.getBlockedState(blockableState);
      }

      @Override
      public BlockingAggregator getBlockedStatus(final UUID blockableId, final BlockingStateType type, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (type == BlockingStateType.SUBSCRIPTION) {
              return getBlockedStateSubscriptionId(blockableId, upToDate, context);
          } else if (type == BlockingStateType.SUBSCRIPTION_BUNDLE) {
              return getBlockedStateBundleId(blockableId, upToDate, context);
          } else { // BlockingStateType.ACCOUNT {
              return getBlockedStateAccountId(blockableId, upToDate, context);
          }
      }

      @Override
      public BlockingAggregator getBlockedStatus(final List<BlockingState> accountEntitlementStates, final List<BlockingState> bundleEntitlementStates, final List<BlockingState> subscriptionEntitlementStates, final InternalTenantContext internalTenantContext) {
          return statelessBlockingChecker.getBlockedState(accountEntitlementStates, bundleEntitlementStates, subscriptionEntitlementStates);
      }

      @Override
      public void checkBlockedChange(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockChange()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_CHANGE, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }

      @Override
      public void checkBlockedEntitlement(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockEntitlement()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_ENTITLEMENT, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }

      @Override
      public void checkBlockedBilling(final Blockable blockable, final DateTime upToDate, final InternalTenantContext context) throws BlockingApiException {
          if (blockable instanceof SubscriptionBase && getBlockedStateSubscription(((SubscriptionBase) blockable).getBundleId(), blockable.getId(), upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_SUBSCRIPTION, blockable.getId().toString());
          } else if (blockable instanceof SubscriptionBaseBundle && getBlockedStateBundle(((SubscriptionBaseBundle) blockable).getAccountId(), blockable.getId(), upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_BUNDLE, blockable.getId().toString());
          } else if (blockable instanceof Account && getBlockedStateAccount((Account) blockable, upToDate, context).isBlockBilling()) {
              throw new BlockingApiException(ErrorCode.BLOCK_BLOCKED_ACTION, ACTION_BILLING, TYPE_ACCOUNT, blockable.getId().toString());
          }
      }
  ```
  [Applies to]: Policy rule (Priority P0) — entitlement module (DefaultBlockingChecker)
  [Confidence score]: High

- [Kill-BR-0160] - Sub-one-day blocking periods are not disabled for billing
  [Description]: A blocking period is only turned into a billing-disabled duration if it lasts at least one full day; blocking states that are opened and cleared within the same day are ignored for billing purposes (per GitHub issue #267). For example: given A blocking state that sets blockBilling=true and is cleared 6 hours later, when createBlockingDurations computes disabled ranges for the subscription, then No START_BILLING_DISABLED/END_BILLING_DISABLED event pair is generated for that sub-day window. Parameters: Minimum blocked duration: 1 day.
  [Line Numbers]: 62 to 68
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/BlockingStateService.java
  [Source Code]:
  ```java
      private void addDisabledDuration(final BlockingState firstBlockingState, @Nullable final DateTime disableDurationEndDate) {

          if (disableDurationEndDate == null || Days.daysBetween(firstBlockingState.getEffectiveDate(), disableDurationEndDate).getDays() >= 1) {
              // Don't disable for periods less than a day (see https://github.com/killbill/killbill/issues/267)
              result.add(new DisabledDuration(firstBlockingState.getEffectiveDate(), disableDurationEndDate));
          }
      }
  ```
  [Applies to]: Policy rule (Priority P1) — junction module (BlockingStateService)
  [Confidence score]: High

- [Kill-BR-0161] - AUTO_INVOICING_OFF tag (account or bundle) fully excludes subscriptions from billing
  [Description]: If the account carries an AUTO_INVOICING_OFF control tag, the entire account's billing event set is marked off. If only a specific bundle carries the AUTO_INVOICING_OFF tag, every subscription in that bundle is added to a skip list and excluded from billing-event generation, while other bundles on the account continue to be billed normally. For example: given A bundle tagged AUTO_INVOICING_OFF, on an account with two other untagged bundles, when Billing events are computed for the account, then Subscriptions in the tagged bundle are recorded in subscriptionIdsWithAutoInvoiceOff and produce no billing events; the other bundles bill normally. Parameters: Tags: AUTO_INVOICING_OFF, AUTO_INVOICING_DRAFT, AUTO_INVOICING_REUSE_DRAFT.
  [Line Numbers]: 99 to 115
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DefaultInternalBillingApi.java
  [Source Code]:
  ```java
          // Check to see if billing is off for the account
          final List<Tag> tagsForAccount = tagApi.getTagsForAccount(false, context);
          final List<Tag> accountTags = getTagsForObjectType(ObjectType.ACCOUNT, tagsForAccount, null);
          final boolean found_AUTO_INVOICING_OFF = is_AUTO_INVOICING_OFF(accountTags);
          final boolean found_INVOICING_DRAFT = is_AUTO_INVOICING_DRAFT(accountTags);
          final boolean found_INVOICING_REUSE_DRAFT = is_AUTO_INVOICING_REUSE_DRAFT(accountTags);

          final Set<UUID> skippedSubscriptions = new HashSet<>();
          final DefaultBillingEventSet result;

          long subsIniTs = System.nanoTime();
          final Map<UUID, List<SubscriptionBase>> subscriptionsForAccount = subscriptionApi.getSubscriptionsForAccount(fullCatalog, cutoffDt, context);
          long subsAfterTs = System.nanoTime();

          final ImmutableAccountData account = accountApi.getImmutableAccountDataById(accountId, context);
          result = new DefaultBillingEventSet(found_AUTO_INVOICING_OFF, found_INVOICING_DRAFT, found_INVOICING_REUSE_DRAFT);
          addBillingEventsForBundles(account, dryRunArguments, context, result, skippedSubscriptions, subscriptionsForAccount, fullCatalog, tagsForAccount);
  ```
  [Line Numbers]: 208 to 214
  [Source Code File Name]: junction/src/main/java/org/killbill/billing/junction/plumbing/billing/DefaultInternalBillingApi.java
  [Source Code]:
  ```java
              // Check if billing is off for the bundle
              final List<Tag> bundleTags = getTagsForObjectType(ObjectType.BUNDLE, tagsForAccount, bundleId);
              final boolean found_AUTO_INVOICING_OFF = is_AUTO_INVOICING_OFF(bundleTags);
              if (found_AUTO_INVOICING_OFF) {
                  for (final SubscriptionBase subscription : subscriptions) { // billing is off so list sub ids in set to be excluded
                      result.getSubscriptionIdsWithAutoInvoiceOff().add(subscription.getId());
                  }
  ```
  [Applies to]: Policy rule (Priority P0) — junction module (DefaultInternalBillingApi)
  [Confidence score]: High

- [Kill-BR-0162] - Janitor retry schedule differs for UNKNOWN vs PENDING transactions and terminates after exhausting the list
  [Description]: When a payment transaction remains unresolved, the delay before the next Janitor recheck attempt is looked up from one of two separately configured retry-interval lists depending on whether the transaction's status is UNKNOWN or PENDING; once the attempt number exceeds the configured list's length, no further recheck is scheduled and the transaction is left in its unresolved state permanently. For example: given A transaction stuck in PENDING with paymentConfig.getPendingTransactionsRetries() containing 3 entries, when The Janitor schedules its 4th retry attempt, then getNextNotificationTime returns null and no further notification/recheck is scheduled. Parameters: paymentConfig.getUnknownTransactionsRetries(...) and paymentConfig.getPendingTransactionsRetries(...) — configurable TimeSpan lists, values not hardcoded in this file. Note (extraction review flagged this rule for SME confirmation): What are the configured default retry-interval lists for UNKNOWN vs PENDING transactions in production (paymentConfig defaults), and is it acceptable that a transaction permanently stays unresolved once the list is exhausted with no alerting?.
  [Line Numbers]: 401 to 418
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java
  [Source Code]:
  ```java
      @VisibleForTesting
      DateTime getNextNotificationTime(final TransactionStatus transactionStatus, final Integer attemptNumber, final InternalTenantContext internalTenantContext) {
          final List<TimeSpan> retries;
          if (TransactionStatus.UNKNOWN.equals(transactionStatus)) {
              retries = paymentConfig.getUnknownTransactionsRetries(internalTenantContext);
          } else if (TransactionStatus.PENDING.equals(transactionStatus)) {
              retries = paymentConfig.getPendingTransactionsRetries(internalTenantContext);
          } else {
              retries = Collections.emptyList();
              log.warn("Unexpected transactionStatus='{}' from janitor, ignore...", transactionStatus);
          }

          if (attemptNumber > retries.size()) {
              return null;
          }
          final TimeSpan nextDelay = retries.get(attemptNumber - 1);
          return clock.getUTCNow().plusMillis((int) nextDelay.getMillis());
      }
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (IncompletePaymentAttemptTask)
  [Confidence score]: Medium

- [Kill-BR-0163] - Bundle transfer: old subscription cancellation date honors charged-through date
  [Description]: When a bundle is transferred to another account (not an immediate cancel), the source subscription isn't cancelled on the transfer date if the customer already paid through a later date — it is cancelled at the later of the charged-through date or the earliest valid transition date instead. For example: given A subscription's charged-through date is 2026-09-15, transfer requested for 2026-09-01, cancelImmediately=false, and the subscription's earliest valid transition date is 2026-08-01, when transferBundle() computes the cancellation event for the old subscription, then The old subscription's cancel event is scheduled for 2026-09-15 (charged-through date), not 2026-09-01; if that computed date were somehow before the earliest valid transition date, it is pulled forward to that earliest date instead.
  [Line Numbers]: 250 to 268
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/api/transfer/DefaultSubscriptionBaseTransferApi.java
  [Source Code]:
  ```java
                  // For future add-on cancellations, don't add a cancellation on disk right away (mirror the behavior
                  // on base plan cancellations, even though we don't support un-transfer today)
                  if (productCategory != ProductCategory.ADD_ON || cancelImmediately) {
                      // Create the cancelWithRequestedDate event on effectiveCancelDate
                      final DateTime candidateCancelDate = !cancelImmediately &&
                                                           oldSubscription.getChargedThroughDate() != null &&
                                                           effectiveTransferDate.isBefore(oldSubscription.getChargedThroughDate()) ?
                                                           oldSubscription.getChargedThroughDate() : effectiveTransferDate;

                      //
                      // We are checking that if the subscription is PENDING (start date in the future) and if the requestedDate
                      // for the transfer is prior to the startDate, then it gets realigned with the startDate.
                      // The code goes further (reuse logic from validateEffectiveDate) and also checks that we if we have a subscription with multiple
                      // change plans (that we want to transfer), we don't end up cancelling prior a previous transition as this would create some
                      // weird REPAIR scenarios.
                      //
                      final SubscriptionBaseTransition previousTransition = oldSubscription.getPreviousTransition();
                      final DateTime earliestValidDate = previousTransition != null ? previousTransition.getEffectiveTransitionTime() : oldSubscription.getStartDate();
                      final DateTime effectiveCancelDate = (candidateCancelDate.isBefore(earliestValidDate)) ? earliestValidDate : candidateCancelDate;
  ```
  [Applies to]: Policy rule (Priority P1) — subscription module (DefaultSubscriptionBaseTransferApi)
  [Confidence score]: High

- [Kill-BR-0164] - Chargeback is idempotent — no partial/duplicate chargebacks
  [Description]: A payment can only be charged back once; if a chargeback invoice-payment record already exists for that payment, a second successful chargeback transaction is treated as a no-op instead of recording another chargeback. For example: given A payment that already has a recorded chargeback, when onSuccessCall() is invoked again for a CHARGEBACK transaction on the same payment, then No new chargeback invoice-payment is recorded; the system logs that the chargeback was already completed.
  [Line Numbers]: 209 to 232
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
                  case CHARGEBACK:
                      existingInvoicePayment = invoiceApi.getInvoicePaymentForChargeback(paymentControlContext.getPaymentId(), internalContext);
                      if (existingInvoicePayment != null) {
                          // We don't support partial chargebacks (yet?)
                          log.info("onSuccessCall was already completed for chargeback paymentId='{}'", paymentControlContext.getPaymentId());
                      } else {
                          final InvoicePayment linkedInvoicePayment = invoiceApi.getInvoicePaymentForAttempt(paymentControlContext.getPaymentId(), internalContext);

                          final BigDecimal amount;
                          final Currency currency;
                          if (linkedInvoicePayment.getCurrency().equals(paymentControlContext.getProcessedCurrency()) && paymentControlContext.getProcessedAmount() != null) {
                              amount = paymentControlContext.getProcessedAmount();
                              currency = paymentControlContext.getProcessedCurrency();
                          } else if (linkedInvoicePayment.getCurrency().equals(paymentControlContext.getCurrency()) && paymentControlContext.getAmount() != null) {
                              amount = paymentControlContext.getAmount();
                              currency = paymentControlContext.getCurrency();
                          } else {
                              amount = linkedInvoicePayment.getAmount();
                              currency = linkedInvoicePayment.getCurrency();
                          }

                          invoiceApi.recordChargeback(paymentControlContext.getPaymentId(), paymentControlContext.getAttemptPaymentId(), paymentControlContext.getTransactionExternalKey(), amount, currency, internalContext);
                      }
                      break;
  ```
  [Applies to]: Policy rule (Priority P0) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: High

- [Kill-BR-0165] - REFUND and CREDIT transactions are never retried on failure
  [Description]: If a refund or credit transaction fails, the system does not schedule an automatic retry (unlike failed purchases, which do get a computed retry date). For example: given A REFUND transaction that fails at the gateway, when onFailureCall() is invoked for that transaction, then No nextRetryDate is computed or returned; only a PURCHASE failure computes computeNextRetryDate(...).
  [Line Numbers]: 294 to 297
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  [Source Code]:
  ```java
              case CREDIT:
              case REFUND:
                  // We don't retry REFUND
                  break;
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (InvoicePaymentControlPluginApi)
  [Confidence score]: High

- [Kill-BR-0166] - Chargeable invoice item type classification
  [Description]: Only TAX, EXTERNAL_CHARGE, FIXED, USAGE, and RECURRING invoice item types count as billable 'charges' for calculator purposes; credits, adjustments, and CBA items do not. For example: given An invoice with a RECURRING item and a CBA_ADJ item, when isCharge() is evaluated on each item, then The RECURRING item returns true; the CBA_ADJ item returns false.
  [Line Numbers]: 87 to 94
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/calculator/InvoiceCalculatorUtils.java
  [Source Code]:
  ```java
      // Regular line item (charges)
      public static boolean isCharge(final InvoiceItem invoiceItem) {
          return InvoiceItemType.TAX.equals(invoiceItem.getInvoiceItemType()) ||
                 InvoiceItemType.EXTERNAL_CHARGE.equals(invoiceItem.getInvoiceItemType()) ||
                 InvoiceItemType.FIXED.equals(invoiceItem.getInvoiceItemType()) ||
                 InvoiceItemType.USAGE.equals(invoiceItem.getInvoiceItemType()) ||
                 InvoiceItemType.RECURRING.equals(invoiceItem.getInvoiceItemType());
      }
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (InvoiceCalculatorUtils)
  [Confidence score]: High

- [Kill-BR-0167] - Catalog version selection by effective date
  [Description]: When looking up the catalog to use for a given date, the system picks the most recent catalog version whose effective date is on or before that date; if every version is dated after the given date, it falls back to using the very first (oldest) version rather than failing. For example: given Catalog versions effective 2025-01-01, 2025-06-01, 2026-01-01, and a lookup date of 2025-08-01, when getVersion(date) is called, then The 2025-06-01 version is returned (latest version with effectiveDate <= 2025-08-01). Edge cases: If lookup date is before all versions (e.g. due to clock manipulation in tests), the oldest version is returned instead of throwing, per killbill/killbill#760; Throws IllegalStateException only if there are zero catalog versions at all.
  [Line Numbers]: 91 to 107
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java
  [Source Code]:
  ```java
      private int indexOfVersionForDate(final Date date) {

          for (int i = versions.size() - 1; i >= 0; i--) {
              final StaticCatalog c = versions.get(i);
              if (c.getEffectiveDate().getTime() <= date.getTime()) {
                  return i;
              }
          }
          // If the only version we have are after the input date, we return the first version
          // This is not strictly correct from an api point of view, but there is no real good use case
          // where the system would ask for the catalog for a date prior any catalog was uploaded and
          // yet time manipulation could end of inn that state -- see https://github.com/killbill/killbill/issues/760
          if (!versions.isEmpty()) {
              return 0;
          }
          throw new IllegalStateException(String.format("No existing versions in the VersionedCatalog catalog for input date %s", date));
      }
  ```
  [Applies to]: Policy rule (Priority P0) — catalog module (DefaultVersionedCatalog)
  [Confidence score]: High

- [Kill-BR-0168] - Existing-subscription catalog grandfathering
  [Description]: New subscriptions always use the latest catalog version as of the request date. But for an existing subscription undergoing a plan change/renewal, a newer catalog version's pricing/terms only kick in once the plan's configured 'effective date for existing subscriptions' has passed; until then, the subscription keeps using the older catalog version's terms for that plan (or, if no such grandfathering date is configured for a newer version, that version simply doesn't apply to existing subscriptions and the search continues to older versions). For example: given Plan 'gold-monthly' is repriced in catalog version effective 2026-01-01, with effectiveDateForExistingSubscriptions=2026-04-01; an existing subscription had its plan chosen on 2025-01-01, when The system resolves which plan/pricing to use for that existing subscription on a change dated 2026-02-15, then The 2026-01-01 catalog version's new pricing is NOT applied (still before 2026-04-01); the previous catalog version's plan definition is used instead. Parameters: effectiveDateForExistingSubscriptions (per-plan, optional; null means grandfathering never applies and existing subs never see that version's changes for this plan).
  [Line Numbers]: 168 to 222
  [Source Code File Name]: subscription/src/main/java/org/killbill/billing/subscription/catalog/SubscriptionCatalog.java
  [Source Code]:
  ```java
      private CatalogPlanEntry findCatalogPlanEntry(final PlanRequestWrapper wrapper,
                                                    final DateTime requestedDate,
                                                    final DateTime subscriptionChangePlanDate) throws CatalogApiException {
          final List<StaticCatalog> catalogs = versionsBeforeDate(requestedDate);
          if (catalogs.isEmpty()) {
              throw new CatalogApiException(ErrorCode.CAT_NO_CATALOG_FOR_GIVEN_DATE, requestedDate.toDate().toString());
          }

          CatalogPlanEntry candidateInSubsequentCatalog = null;
          for (int i = catalogs.size() - 1; i >= 0; i--) { // Working backwards to find the latest applicable plan
              final StaticCatalog c = catalogs.get(i);

              final Plan plan;
              try {
                  plan = wrapper.findPlan(c);
              } catch (final CatalogApiException e) {
                  if (e.getCode() != CAT_NO_SUCH_PLAN.getCode() &&
                      e.getCode() != ErrorCode.CAT_PLAN_NOT_FOUND.getCode()) {
                      throw e;
                  } else {
                      // If we can't find an entry it probably means the plan has been retired so we keep looking...
                      continue;
                  }
              }

              final boolean oldestCatalog = (i == 0);
              final DateTime catalogEffectiveDate = CatalogDateHelper.toUTCDateTime(c.getEffectiveDate());
              final boolean catalogOlderThanSubscriptionChangePlanDate = !subscriptionChangePlanDate.isBefore(catalogEffectiveDate);
              if (oldestCatalog || // Prevent issue with time granularity -- see #760
                  catalogOlderThanSubscriptionChangePlanDate) { // It's a new subscription, this plan always applies
                  return new CatalogPlanEntry(c, plan);
              } else { // It's an existing subscription
                  if (plan.getEffectiveDateForExistingSubscriptions() != null) { // If it is null, any change to this catalog does not apply to existing subscriptions
                      final DateTime existingSubscriptionDate = CatalogDateHelper.toUTCDateTime(plan.getEffectiveDateForExistingSubscriptions());
                      if (requestedDate.compareTo(existingSubscriptionDate) >= 0) { // This plan is now applicable to existing subs
                          return new CatalogPlanEntry(c, plan);
                      }
                  } else if (candidateInSubsequentCatalog == null) {
                      // Keep the most recent one
                      candidateInSubsequentCatalog = new CatalogPlanEntry(c, plan);
                  }
              }
          }

          if (candidateInSubsequentCatalog != null) {
              return candidateInSubsequentCatalog;
          }

          final PlanSpecifier spec = wrapper.getSpec();
          throw new CatalogApiException(ErrorCode.CAT_PLAN_NOT_FOUND,
                                        spec.getPlanName() != null ? spec.getPlanName() : "undefined",
                                        spec.getProductName() != null ? spec.getProductName() : "undefined",
                                        spec.getBillingPeriod() != null ? spec.getBillingPeriod() : "undefined",
                                        spec.getPriceListName() != null ? spec.getPriceListName() : "undefined");
      }
  ```
  [Applies to]: Policy rule (Priority P0) — subscription module (SubscriptionCatalog)
  [Confidence score]: High

- [Kill-BR-0169] - Default billing alignment falls back to ACCOUNT
  [Description]: If the catalog's billing-alignment rules don't match a specific case for a given plan phase, the system defaults to aligning that subscription's billing cycle day to the account level. For example: given A catalog with no <billingAlignmentCase> matching a particular plan/phase combination, when getBillingAlignment() is called for that plan phase, then BillingAlignment.ACCOUNT is returned.
  [Line Numbers]: 138 to 140
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      public BillingAlignment getBillingAlignment(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingAlignment result = DefaultCasePhase.getResult(billingAlignmentCase, planPhase, root);
          return (result != null) ? result : BillingAlignment.ACCOUNT;
  ```
  [Applies to]: Policy rule (Priority P1) — catalog module (DefaultPlanRules)
  [Confidence score]: High

- [Kill-BR-0170] - OVERDUE_ENFORCEMENT_OFF tag fully bypasses overdue evaluation
  [Description]: If an account is tagged with OVERDUE_ENFORCEMENT_OFF, the overdue state machine is never re-evaluated for it — the account effectively can never enter an overdue state regardless of unpaid balance. For example: given An account tagged OVERDUE_ENFORCEMENT_OFF with a large unpaid balance past all overdue day thresholds, when The overdue refresh job runs for that account, then refreshWithLock returns immediately with no state transition and no evaluation of billing state at all.
  [Line Numbers]: 109 to 113
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java
  [Source Code]:
  ```java
      private void refreshWithLock(final DateTime effectiveDate, final InternalCallContext context) throws OverdueException, OverdueApiException {
          if (overdueStateApplicator.isAccountTaggedWith_OVERDUE_ENFORCEMENT_OFF(context)) {
              log.debug("OverdueStateApplicator: apply returns because account (recordId={}) is set with OVERDUE_ENFORCEMENT_OFF", context.getAccountRecordId());
              return;
          }
  ```
  [Applies to]: Policy rule (Priority P0) — overdue module (OverdueWrapper)
  [Confidence score]: High

- [Kill-BR-0171] - Catalog rule-case matching (first match, wildcard nulls)
  [Description]: Catalog rule tables (change policy, change alignment, cancel policy, create alignment, billing alignment, price-list transition) are evaluated as an ordered list of cases; a case's product/productCategory/billingPeriod/priceList fields act as wildcards when left unset (null), and the array is scanned in declaration order, returning the result of the first case whose non-null fields all match the plan being evaluated. For example: given A changeCase array [Case1: fromProduct=null,toProduct=null (a catch-all default), Case2: fromProduct=Gold,toProduct=Silver->policy=IMMEDIATE] declared in that order in the catalog XML, when A plan change from Gold to Silver is evaluated, then Case1 is checked first; since all of its fields are null (wildcard) it matches immediately and its result is returned, even though Case2 is a more specific match further down the list. Edge cases: Because case-order controls precedence, placing a wildcard/default case before a specific case silently makes the specific case unreachable; Suspected defect: Rule matching is purely positional (first-match-wins) rather than most-specific-match-wins; catalog authors must manually order rules from most specific to least specific or a broad default rule can shadow a narrower one.. Parameters: None (order-dependent, driven entirely by XML declaration order). Note (extraction review flagged this rule for SME confirmation): Should catalog rule-case matching pick the most specific match instead of the first matching entry in XML order, to avoid a default/wildcard rule accidentally shadowing a more specific one?.
  [Line Numbers]: 47 to 87
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java
  [Source Code]:
  ```java
      public T getResult(final PlanSpecifier planPhase, final StaticCatalog c) throws CatalogApiException {
          if (satisfiesCase(planPhase, c)) {
              return getResult();
          }
          return null;
      }

      protected boolean satisfiesCase(final PlanSpecifier planPhase, final StaticCatalog c) throws CatalogApiException {
          final Product product;
          final BillingPeriod billingPeriod;
          final ProductCategory productCategory;
          final PriceList priceList;
          if (planPhase.getPlanName() != null) {
              final Plan plan = c.findPlan(planPhase.getPlanName());
              product = plan.getProduct();
              billingPeriod = plan.getRecurringBillingPeriod();
              productCategory = plan.getProduct().getCategory();
              priceList =  plan.getPriceList();
          } else {
              product = c.findProduct(planPhase.getProductName());
              billingPeriod = planPhase.getBillingPeriod();
              productCategory = product.getCategory();
              priceList = getPriceList() != null ? c.findPriceList(planPhase.getPriceListName()) : null;
          }
          return (getProduct() == null || getProduct().equals(product)) &&
                 (getProductCategory() == null || getProductCategory().equals(productCategory)) &&
                 (getBillingPeriod() == null || getBillingPeriod().equals(billingPeriod)) &&
                 (getPriceList() == null || getPriceList().equals(priceList));
      }

      public static <K> K getResult(final DefaultCase<K>[] cases, final PlanSpecifier planSpec, final StaticCatalog catalog) throws CatalogApiException {
          if (cases != null) {
              for (final DefaultCase<K> c : cases) {
                  final K result = c.getResult(planSpec, catalog);
                  if (result != null) {
                      return result;
                  }
              }
          }
          return null;
  ```
  [Applies to]: Policy rule (Priority P1) — catalog module (DefaultCase)
  [Confidence score]: Medium

- [Kill-BR-0172] - Default plan-creation alignment and cancellation policy
  [Description]: If no catalog rule case matches a plan for creation alignment, new subscriptions default to aligning to the start of the bundle; if no case matches for cancellation, the plan defaults to cancelling at END_OF_TERM (i.e. non-immediate). For example: given A catalog with no explicit <createAlignmentCase> or <cancelPolicyCase> matching a given plan, when A new subscription is created on that plan, or later cancelled, then Plan creation aligns to PlanAlignmentCreate.START_OF_BUNDLE and cancellation defaults to BillingActionPolicy.END_OF_TERM. Parameters: Defaults: START_OF_BUNDLE (creation), END_OF_TERM (cancellation).
  [Line Numbers]: 126 to 135
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      public PlanAlignmentCreate getPlanCreateAlignment(final PlanSpecifier specifier) throws CatalogApiException {
          final PlanAlignmentCreate result = DefaultCase.getResult(createAlignmentCase, specifier, root);
          return (result != null) ? result : PlanAlignmentCreate.START_OF_BUNDLE;
      }

      @Override
      public BillingActionPolicy getPlanCancelPolicy(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingActionPolicy result = DefaultCasePhase.getResult(cancelCase, planPhase, root);
          return (result != null) ? result : BillingActionPolicy.END_OF_TERM;
      }
  ```
  [Applies to]: Policy rule (Priority P1) — catalog module (DefaultPlanRules)
  [Confidence score]: High

- [Kill-BR-0173] - Usage billing in IN_ADVANCE mode is unimplemented for next-notification scheduling
  [Description]: When updating the per-subscription next-usage-notification date after generating usage invoice items, the system only supports IN_ARREAR usage billing; if it is ever asked to do this for IN_ADVANCE usage billing it throws an unimplemented-state error instead of scheduling anything. For example: given A catalog usage section configured with billingMode=IN_ADVANCE, when updatePerSubscriptionNextNotificationUsageDate is invoked for that usage's billing mode, then An IllegalStateException('Not implemented Yet)') is thrown, so usage-in-advance invoicing cannot currently complete its next-notification bookkeeping. Edge cases: This is a hard runtime failure, not a graceful validation error — any live catalog with IN_ADVANCE usage sections billed via this path will crash invoice generation. Note (extraction review flagged this rule for SME confirmation): P0 panel split on whether this moves money / is regulatory (Code confirmed faithful: UsageInvoiceItemGenerator.java:231-234 does exactly what the rule claims — updatePerSubscriptionNextNotificationUsageDate throws `new IllegalStateException("Not implemented Yet)")` (typo and all) when usageBillingMode == BillingMode.IN.
  [Line Numbers]: 231 to 234
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/generator/UsageInvoiceItemGenerator.java
  [Source Code]:
  ```java
      private void updatePerSubscriptionNextNotificationUsageDate(final UUID subscriptionId, final Map<String, LocalDate> nextBillingCycleDates, final BillingMode usageBillingMode, final Map<UUID, SubscriptionFutureNotificationDates> perSubscriptionFutureNotificationDates) {
          if (usageBillingMode == BillingMode.IN_ADVANCE) {
              throw new IllegalStateException("Not implemented Yet)");
          }
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (UsageInvoiceItemGenerator)
  [Confidence score]: Medium

- [Kill-BR-0174] - Zero-amount recurring and fixed items are excluded from the repair tree
  [Description]: When rebuilding the invoice-item repair tree for a subscription, existing RECURRING items with a $0 amount are set aside and never repaired/adjusted, and existing FIXED items are always set aside untouched — only non-zero RECURRING and REPAIR_ADJ items participate in the repair/proration tree logic. For example: given An existing invoice item of type RECURRING with amount=$0.00 for a subscription being re-invoiced, when The subscription's item tree is rebuilt for the new invoice, then That $0 RECURRING item is added to the 'existingIgnoredItems' list and is never split, repaired, or replaced by the tree logic. Edge cases: FIXED items are always ignored by the tree regardless of amount.
  [Line Numbers]: 91 to 115
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java
  [Source Code]:
  ```java
          switch (invoiceItem.getInvoiceItemType()) {
              case RECURRING:
                  if (invoiceItem.getAmount().compareTo(BigDecimal.ZERO) == 0) {
                      // Nothing to repair -- https://github.com/killbill/killbill/issues/783
                      existingIgnoredItems.add(invoiceItem);
                  } else {
                      root.addExistingItem(new ItemsNodeInterval(root, new Item(invoiceItem, targetInvoiceId, ItemAction.ADD, prorationFixedDays), prorationFixedDays));
                  }
                  break;

              case REPAIR_ADJ:
                  root.addExistingItem(new ItemsNodeInterval(root, new Item(invoiceItem, targetInvoiceId, ItemAction.CANCEL, prorationFixedDays), prorationFixedDays));
                  break;

              case FIXED:
                  existingIgnoredItems.add(invoiceItem);
                  break;

              case ITEM_ADJ:
                  pendingItemAdj.add(invoiceItem);
                  break;

              default:
                  break;
          }
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (SubscriptionItemTree)
  [Confidence score]: High

- [Kill-BR-0175] - Item adjustments linked to an ignored item are themselves dropped
  [Description]: When building the repair tree, any pending ITEM_ADJ adjustment whose linked item was previously set aside as ignored (e.g. a $0 recurring item) is discarded rather than applied, since there is nothing left in the tree for it to adjust. For example: given A $0 RECURRING item (ignored) that later received an ITEM_ADJ adjustment on disk, when The tree is built via SubscriptionItemTree.build(), then That ITEM_ADJ is skipped entirely (root.addAdjustment is never called for it).
  [Line Numbers]: 121 to 133
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java
  [Source Code]:
  ```java
      public void build() {
          Preconditions.checkState(!isBuilt);

          for (final InvoiceItem item : pendingItemAdj) {
              // If the linked item was ignored, ignore this adjustment too
              final boolean isLinkedItemExist = existingIgnoredItems
                      .stream()
                      .anyMatch(input -> input.getId().equals(item.getLinkedItemId()));
              if (!isLinkedItemExist) {
                  root.addAdjustment(item);
              }
          }
          pendingItemAdj.clear();
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (SubscriptionItemTree)
  [Confidence score]: High

- [Kill-BR-0176] - Overdue re-evaluation notification scheduling
  [Description]: After applying a new overdue state, the system schedules a future re-check: if the account just moved to CLEAR, it uses the overdue config's initialReevaluationInterval (only if the account still has an unpaid invoice and at least one overdue state is configured); otherwise it uses the new state's own configured autoReevaluationInterval. If no reevaluation interval is configured at all, no future check is scheduled (the assumption being the overdue conditions aren't time-based). For example: given An account transitions into overdue state OD2, which has autoReevaluationInterval=P7D configured, when OverdueStateApplicator.apply() runs at effectiveDate=2026-06-01, then A future notification is scheduled for 2026-06-08 (2026-06-01 + 7 days) to re-run the overdue evaluation. Edge cases: If nextOverdueState is CLEAR and there is no unpaid invoice (or no overdue states configured at all), no future notification is created and the existing one is cleared instead; OVERDUE_NO_REEVALUATION_INTERVAL exception from config is treated as 'no reschedule needed', not an error. Parameters: initialReevaluationInterval (account-level default), autoReevaluationInterval (per-state, e.g. P7D).
  [Line Numbers]: 104 to 125
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      public void apply(final DateTime effectiveDate, final OverdueStateSet overdueStateSet, final BillingState billingState,
                        final ImmutableAccountData account, final OverdueState previousOverdueState,
                        final OverdueState nextOverdueState, final InternalCallContext context) throws OverdueException, OverdueApiException {
          log.debug("OverdueStateApplicator: time={}, previousState={}, nextState={}, billingState={}", effectiveDate, previousOverdueState, nextOverdueState, billingState);

          final OverdueState firstOverdueState = overdueStateSet.getFirstState();
          final boolean conditionForNextNotfication = !nextOverdueState.isClearState() ||
                                                      // We did not reach the first state yet but we have an unpaid invoice
                                                      (firstOverdueState != null && billingState != null && billingState.getDateOfEarliestUnpaidInvoice() != null);

          if (conditionForNextNotfication) {
              final Period reevaluationInterval = getReevaluationInterval(overdueStateSet, nextOverdueState);
              // If there is no configuration in the config, we assume this is because the overdue conditions are not time based and so there is nothing to retry
              if (reevaluationInterval == null) {
                  log.debug("OverdueStateApplicator <notificationQ>: missing InitialReevaluationInterval from config, NOT inserting notification for account {}", account.getId());
              } else {
                  log.debug("OverdueStateApplicator <notificationQ>: inserting notification for account={}, time={}", account.getId(), effectiveDate.plus(reevaluationInterval));
                  createFutureNotification(account, effectiveDate.plus(reevaluationInterval), context);
              }
          } else if (nextOverdueState.isClearState()) {
              clearFutureNotification(account, context);
          }
  ```
  [Line Numbers]: 160 to 174
  [Source Code File Name]: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  [Source Code]:
  ```java
      private Period getReevaluationInterval(final OverdueStateSet overdueStateSet, final OverdueState nextOverdueState) throws OverdueException {
          try {
              if (nextOverdueState.isClearState()) {
                  return overdueStateSet.getInitialReevaluationInterval();
              } else {
                  return nextOverdueState.getAutoReevaluationInterval().toJodaPeriod();
              }
          } catch (final OverdueApiException e) {
              if (e.getCode() == ErrorCode.OVERDUE_NO_REEVALUATION_INTERVAL.getCode()) {
                  return null;
              } else {
                  throw new OverdueException(e);
              }
          }
      }
  ```
  [Applies to]: Policy rule (Priority P1) — overdue module (OverdueStateApplicator)
  [Confidence score]: High

- [Kill-BR-0177] - RBAC permission check supports AND/OR logic across required permissions
  [Description]: When an API call requires a list of permissions, the caller must satisfy all of them (AND) or at least one of them (OR), depending on how the endpoint is annotated; a single-permission check is a simple pass/fail. For example: given A JAX-RS endpoint requires permissions [ACCOUNT_CAN_VIEW, INVOICE_CAN_VIEW] with Logical.OR, when The current subject holds only INVOICE_CAN_VIEW, then The permission check passes because at least one of the OR'd permissions is held; had Logical.AND been used instead, the same subject would fail with SECURITY_NOT_ENOUGH_PERMISSIONS. Parameters: Logical.AND / Logical.OR enum, ErrorCode.SECURITY_NOT_ENOUGH_PERMISSIONS.
  [Line Numbers]: 177 to 204
  [Source Code File Name]: util/src/main/java/org/killbill/billing/util/security/api/DefaultSecurityApi.java
  [Source Code]:
  ```java
      @Override
      public void checkCurrentUserPermissions(final List<Permission> permissions, final Logical logical, final TenantContext context) throws SecurityApiException {
          final String[] permissionsString = permissions.stream().map(Permission::toString).toArray(String[]::new);

          try {
              final Subject subject = SecurityUtils.getSubject();
              if (permissionsString.length == 1) {
                  subject.checkPermission(permissionsString[0]);
              } else if (Logical.AND.equals(logical)) {
                  subject.checkPermissions(permissionsString);
              } else if (Logical.OR.equals(logical)) {
                  boolean hasAtLeastOnePermission = false;
                  for (final String permission : permissionsString) {
                      if (subject.isPermitted(permission)) {
                          hasAtLeastOnePermission = true;
                          break;
                      }
                  }

                  // Cause the exception if none match
                  if (!hasAtLeastOnePermission) {
                      subject.checkPermission(permissionsString[0]);
                  }
              }
          } catch (final AuthorizationException e) {
              throw new SecurityApiException(e, ErrorCode.SECURITY_NOT_ENOUGH_PERMISSIONS);
          }
      }
  ```
  [Applies to]: Policy rule (Priority P0) — util module (DefaultSecurityApi)
  [Confidence score]: High

- [Kill-BR-0178] - Bundle transfer requires the bundle to actually belong to the declared source account
  [Description]: When transferring a bundle (by external key) between accounts, the system looks up the bundle's actual owning account and rejects the transfer if it doesn't match the caller-supplied source account id. For example: given A transfer request with sourceAccountId=A and bundleExternalKey 'bike-club', but the active subscription for that key actually belongs to account B, when transferEntitlementsOverrideBillingPolicy executes the transfer, then An EntitlementApiException wrapping SUB_GET_INVALID_BUNDLE_KEY is thrown and no transfer occurs.
  [Line Numbers]: 313 to 323
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java
  [Source Code]:
  ```java
                  try {

                      final UUID activeSubscriptionIdForKey = entitlementUtils.getFirstActiveSubscriptionIdForKeyOrNull(bundleExternalKey, contextWithSourceAccountRecordId);
                      final UUID bundleId = activeSubscriptionIdForKey != null ?
                                            subscriptionBaseInternalApi.getBundleIdFromSubscriptionId(activeSubscriptionIdForKey, contextWithSourceAccountRecordId) : null;
                      final UUID baseBundleAccountId = bundleId != null ?
                                                       subscriptionBaseInternalApi.getAccountIdFromBundleId(bundleId, contextWithSourceAccountRecordId) : null;

                      if (baseBundleAccountId == null || !baseBundleAccountId.equals(sourceAccountId)) {
                          throw new EntitlementApiException(new SubscriptionBaseApiException(ErrorCode.SUB_GET_INVALID_BUNDLE_KEY, bundleExternalKey));
                      }
  ```
  [Applies to]: Policy rule (Priority P0) — entitlement module (DefaultEntitlementApi)
  [Confidence score]: High

- [Kill-BR-0179] - Bundle transfer billing policy determines whether the source subscription is cancelled immediately or at end of term
  [Description]: A bundle transfer's billing policy (IMMEDIATE or END_OF_TERM) directly controls whether the original subscription is cancelled right away or allowed to run to the end of its current billed term; any other policy value is treated as a programming error. For example: given A transfer request with billingPolicy=IMMEDIATE, when transferEntitlementsOverrideBillingPolicy runs, then cancelImm is set true and the source subscription's cancellation is immediate rather than deferred to end-of-term. Parameters: BillingActionPolicy {IMMEDIATE, END_OF_TERM}.
  [Line Numbers]: 298 to 311
  [Source Code File Name]: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java
  [Source Code]:
  ```java
          final WithEntitlementPlugin<UUID> transferWithPlugin = new WithEntitlementPlugin<UUID>() {
              @Override
              public UUID doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  final boolean cancelImm;
                  switch (billingPolicy) {
                      case IMMEDIATE:
                          cancelImm = true;
                          break;
                      case END_OF_TERM:
                          cancelImm = false;
                          break;
                      default:
                          throw new RuntimeException("Unexpected billing policy " + billingPolicy);
                  }
  ```
  [Applies to]: Policy rule (Priority P1) — entitlement module (DefaultEntitlementApi)
  [Confidence score]: High

- [Kill-BR-0180] - Deleting a system-generated credit is blocked; deleting an invoice-linked credit reclaims outstanding account credit first
  [Description]: Deleting a used-credit consumption line just zeroes it out. Deleting a credit-generation line only works if it can be tied back to an explicit CREDIT_ADJ item on a credit invoice; if the account has already spent down some of that credit elsewhere, the shortfall is reclaimed from other invoices before the delete proceeds. Credit generated automatically by the system (e.g. from a repair) cannot be deleted at all. For example: given A $30 system-generated CBA credit item from a repair invoice with no matching CREDIT_ADJ item, when deleteCBA is called on that item, then INVOICE_CBA_DELETED error is thrown; deletion is refused. Edge cases: Account CBA balance lower than the item amount triggers reclaim from other invoices before delete completes. Note (extraction review flagged this rule for SME confirmation): Is it intended that reclaiming credit from other invoices as a side effect of a CBA delete can alter balances on invoices unrelated to the one being edited?.
  [Line Numbers]: 1198 to 1227
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  [Source Code]:
  ```java
              if (cbaItem.getAmount().compareTo(BigDecimal.ZERO) < 0) { /* Credit consumption */

                  invoiceItemSqlDao.updateItemFields(cbaItem.getId().toString(), BigDecimal.ZERO, null,"Delete used credit", null, null, context);
                  invoiceIds.add(invoice.getId());
              } else if (cbaItem.getAmount().compareTo(BigDecimal.ZERO) > 0) {  /* Credit generation */
                  final InvoiceItemModelDao creditItem = invoice.getInvoiceItems().stream()
                                                                .filter(targetItem -> targetItem.getType() == InvoiceItemType.CREDIT_ADJ &&
                                                                                      targetItem.getAmount().negate().compareTo(cbaItem.getAmount()) >= 0)
                                                                .findFirst()
                                                                .orElse(null);

                  // In case this is a credit invoice (pure credit, or mixed), we allow to 'delete' credit generation
                  if (creditItem != null) { /* Credit Invoice */
                      final BigDecimal accountCBA = cbaDao.getAccountCBAFromTransaction(entitySqlDaoWrapperFactory, context);
                      // If we don't have enough credit left on the account, we reclaim what is necessary
                      if (accountCBA.compareTo(cbaItem.getAmount()) < 0) {
                          final BigDecimal amountToReclaim = cbaItem.getAmount().subtract(accountCBA);
                          final BigDecimal reclaimed = reclaimCreditFromTransaction(amountToReclaim, invoiceIds, entitySqlDaoWrapperFactory, context);
                          Preconditions.checkState(reclaimed.compareTo(amountToReclaim) == 0,
                                                   String.format("Unexpected state, reclaimed used credit [%s/%s]", reclaimed, amountToReclaim));
                      }

                      invoiceItemSqlDao.updateItemFields(cbaItem.getId().toString(), BigDecimal.ZERO, null, "Delete gen credit", null, null, context);
                      final BigDecimal adjustedCreditAmount = creditItem.getAmount().add(cbaItem.getAmount());
                      invoiceItemSqlDao.updateItemFields(creditItem.getId().toString(), adjustedCreditAmount, null,null,null, "Delete gen credit", context);
                      invoiceIds.add(invoice.getId());
                  } else /* System generated credit, e.g Repair invoice */ {
                      throw new InvoiceApiException(ErrorCode.INVOICE_CBA_DELETED, cbaItem.getId());
                  }
              }
  ```
  [Applies to]: Policy rule (Priority P1) — invoice module (DefaultInvoiceDao)
  [Confidence score]: Medium

- [Kill-BR-0181] - Refreshing payment methods from a plugin never un-sets the KB default
  [Description]: When syncing payment methods from a gateway plugin, Kill Bill only updates its own notion of the account's default payment method if the plugin reports a default AND the account's current default already belongs to that same plugin; a plugin reporting 'no default' never clears an existing Kill Bill default. For example: given Account's current default payment method belongs to plugin A; refreshPaymentMethods is run for plugin B and plugin B reports no default, when updateDefaultPaymentMethodIfNeeded runs, then the account default payment method is left unchanged (still plugin A's), because the current default's plugin (A) doesn't match the plugin being refreshed (B). Edge cases: If account has no default payment method set at all, the plugin-reported default is always applied.
  [Line Numbers]: 602 to 670
  [Source Code File Name]: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  [Source Code]:
  ```java
          try {
              final PluginDispatcherReturnType<List<PaymentMethod>> result = new WithAccountLock<List<PaymentMethod>, PaymentApiException>(paymentConfig).processAccountWithLock(locker, account.getId(), new DispatcherCallback<PluginDispatcherReturnType<List<PaymentMethod>>, PaymentApiException>() {
                  @Override
                  public PluginDispatcherReturnType<List<PaymentMethod>> doOperation() throws PaymentApiException {

                      UUID defaultPaymentMethodId = null;

                      final List<PaymentMethodInfoPlugin> pluginPmsWithId = new ArrayList<PaymentMethodInfoPlugin>();
                      final List<PaymentMethodModelDao> finalPaymentMethods = new ArrayList<PaymentMethodModelDao>();
                      for (final PaymentMethodInfoPlugin cur : pluginPms) {
                          // If the kbPaymentId is NULL, the plugin does not know about it, so we create a new UUID
                          final UUID paymentMethodId = cur.getPaymentMethodId() != null ? cur.getPaymentMethodId() : UUIDs.randomUUID();
                          final String externalKey = cur.getExternalPaymentMethodId() != null ? cur.getExternalPaymentMethodId() : paymentMethodId.toString();
                          final PaymentMethod input = new DefaultPaymentMethod(paymentMethodId, externalKey, account.getId(), pluginName);
                          final PaymentMethodModelDao pmModel = new PaymentMethodModelDao(input.getId(), input.getExternalKey(), input.getCreatedDate(), input.getUpdatedDate(),
                                                                                          input.getAccountId(), input.getPluginName(), input.isActive());
                          finalPaymentMethods.add(pmModel);

                          pluginPmsWithId.add(new DefaultPaymentMethodInfoPlugin(cur, paymentMethodId));

                          // Note: we do not unset the default payment method in Kill Bill even if isDefault is false here.
                          // Some gateways don't support the concept of "default" payment methods, in that case the plugin
                          // will always return false - it's Kill Bill in that case which is responsible to manage default payment methods
                          if (cur.isDefault()) {
                              defaultPaymentMethodId = paymentMethodId;
                          }
                      }

                      final List<PaymentMethodModelDao> refreshedPaymentMethods = paymentDao.refreshPaymentMethods(pluginName,
                                                                                                                   finalPaymentMethods,
                                                                                                                   context);

                      try {
                          pluginApi.resetPaymentMethods(account.getId(), pluginPmsWithId, properties, callContext);
                      } catch (final PaymentPluginApiException e) {
                          throw new PaymentApiException(e, ErrorCode.PAYMENT_REFRESH_PAYMENT_METHOD, account.getId(), e.getErrorMessage());
                      }
                      try {
                          updateDefaultPaymentMethodIfNeeded(pluginName, account, defaultPaymentMethodId, context);
                      } catch (final AccountApiException e) {
                          throw new PaymentApiException(e);
                      }
                      final List<PaymentMethod> result = refreshedPaymentMethods.stream()
                              .map(input -> new DefaultPaymentMethod(input, null))
                              .collect(Collectors.toUnmodifiableList());
                      return PluginDispatcher.createPluginDispatcherReturnType(result);
                  }
              });
              return result.getReturnType();
          } catch (final Exception e) {
              throw new PaymentApiException(e, ErrorCode.PAYMENT_INTERNAL_ERROR, Objects.requireNonNullElse(e.getMessage(), ""));
          }
      }

      private void updateDefaultPaymentMethodIfNeeded(final String pluginName, final Account account, @Nullable final UUID defaultPluginPaymentMethodId, final InternalCallContext context) throws PaymentApiException, AccountApiException {

          // Some gateways have the concept of default payment methods. Kill Bill has also its own default payment method
          // and is authoritative on this matter. However, if the default payment method is associated with a given plugin,
          // and if the default payment method in that plugin has changed, we will reflect this change in Kill Bill as well.

          boolean shouldUpdateDefaultPaymentMethod = true;
          if (account.getPaymentMethodId() != null) {
              final PaymentMethodModelDao currentDefaultPaymentMethod = getPaymentMethodById(account.getPaymentMethodId(), true, context);
              shouldUpdateDefaultPaymentMethod = pluginName.equals(currentDefaultPaymentMethod.getPluginName());
          }
          if (shouldUpdateDefaultPaymentMethod) {
              accountInternalApi.updatePaymentMethod(account.getId(), defaultPluginPaymentMethodId, context);
          }
      }
  ```
  [Applies to]: Policy rule (Priority P1) — payment module (PaymentMethodProcessor)
  [Confidence score]: High

- [Kill-BR-0182] - Catalog case-rule matching: first matching rule wins, null fields act as wildcards
  [Description]: Catalog rules (change policy, cancel policy, alignment, price-list rules, etc.) are evaluated as an ordered list of cases; a case matches if every field it specifies (product, product category, billing period, price list) equals the plan being evaluated, and any field left unset in the rule matches anything. The first matching case in declaration order determines the result. For example: given Two change-policy cases: (1) productCategory=BASE -> IMMEDIATE, (2) no fields set (wildcard) -> END_OF_TERM, declared in that order, when A plan change is evaluated for a BASE product, then case (1) matches first and IMMEDIATE is returned, even though the wildcard case (2) would also match.
  [Line Numbers]: 54 to 88
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java
  [Source Code]:
  ```java
      protected boolean satisfiesCase(final PlanSpecifier planPhase, final StaticCatalog c) throws CatalogApiException {
          final Product product;
          final BillingPeriod billingPeriod;
          final ProductCategory productCategory;
          final PriceList priceList;
          if (planPhase.getPlanName() != null) {
              final Plan plan = c.findPlan(planPhase.getPlanName());
              product = plan.getProduct();
              billingPeriod = plan.getRecurringBillingPeriod();
              productCategory = plan.getProduct().getCategory();
              priceList =  plan.getPriceList();
          } else {
              product = c.findProduct(planPhase.getProductName());
              billingPeriod = planPhase.getBillingPeriod();
              productCategory = product.getCategory();
              priceList = getPriceList() != null ? c.findPriceList(planPhase.getPriceListName()) : null;
          }
          return (getProduct() == null || getProduct().equals(product)) &&
                 (getProductCategory() == null || getProductCategory().equals(productCategory)) &&
                 (getBillingPeriod() == null || getBillingPeriod().equals(billingPeriod)) &&
                 (getPriceList() == null || getPriceList().equals(priceList));
      }

      public static <K> K getResult(final DefaultCase<K>[] cases, final PlanSpecifier planSpec, final StaticCatalog catalog) throws CatalogApiException {
          if (cases != null) {
              for (final DefaultCase<K> c : cases) {
                  final K result = c.getResult(planSpec, catalog);
                  if (result != null) {
                      return result;
                  }
              }
          }
          return null;

      }
  ```
  [Applies to]: Policy rule (Priority P1) — catalog module (DefaultCase)
  [Confidence score]: High

- [Kill-BR-0183] - Default fallbacks when no catalog rule case matches
  [Description]: If catalog rule evaluation finds no matching case (which validation is supposed to prevent, see wildcard-rule requirement), Kill Bill falls back to hardcoded defaults: START_OF_BUNDLE for creation and change-plan alignment, END_OF_TERM for cancel and change-plan billing policy. For example: given A catalog with a valid default rule present so this path is a safety net, when getPlanCreateAlignment / getPlanCancelPolicy / getBillingAlignment / getPlanChangeAlignment / getPlanChangePolicy find no matching case, then PlanAlignmentCreate.START_OF_BUNDLE, BillingActionPolicy.END_OF_TERM, BillingAlignment.ACCOUNT, PlanAlignmentChange.START_OF_BUNDLE, and BillingActionPolicy.END_OF_TERM are returned respectively. Parameters: Defaults: START_OF_BUNDLE (create/change alignment), END_OF_TERM (cancel/change policy), ACCOUNT (billing alignment).
  [Line Numbers]: 126 to 176
  [Source Code File Name]: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  [Source Code]:
  ```java
      public PlanAlignmentCreate getPlanCreateAlignment(final PlanSpecifier specifier) throws CatalogApiException {
          final PlanAlignmentCreate result = DefaultCase.getResult(createAlignmentCase, specifier, root);
          return (result != null) ? result : PlanAlignmentCreate.START_OF_BUNDLE;
      }

      @Override
      public BillingActionPolicy getPlanCancelPolicy(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingActionPolicy result = DefaultCasePhase.getResult(cancelCase, planPhase, root);
          return (result != null) ? result : BillingActionPolicy.END_OF_TERM;
      }

      @Override
      public BillingAlignment getBillingAlignment(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingAlignment result = DefaultCasePhase.getResult(billingAlignmentCase, planPhase, root);
          return (result != null) ? result : BillingAlignment.ACCOUNT;
      }

      @Override
      public PlanChangeResult getPlanChangeResult(final PlanPhaseSpecifier from, final PlanSpecifier to) throws CatalogApiException {

          final DefaultPriceList toPriceList = to.getPriceListName() != null ?
                                               (DefaultPriceList) root.findPriceList(to.getPriceListName()) :
                                               findPriceList(from);

          // If we use old scheme {product, billingPeriod, pricelist}, ensure pricelist is correct
          // (Pricelist may be null because if it is unspecified this is the principal use-case)
          final PlanSpecifier toWithPriceList = to.getPlanName() == null ?
                                                new PlanSpecifier(to.getProductName(), to.getBillingPeriod(), toPriceList.getName()) :
                                                to;

          final BillingActionPolicy policy = getPlanChangePolicy(from, toWithPriceList);
          if (policy == BillingActionPolicy.ILLEGAL) {
              throw new IllegalPlanChange(from, toWithPriceList);
          }

          final PlanAlignmentChange alignment = getPlanChangeAlignment(from, toWithPriceList);

          return new PlanChangeResult(toPriceList, policy, alignment);
      }

      private PlanAlignmentChange getPlanChangeAlignment(final PlanPhaseSpecifier from,
                                                         final PlanSpecifier to) throws CatalogApiException {
          final PlanAlignmentChange result = DefaultCaseChange.getResult(changeAlignmentCase, from, to, root);
          return (result != null) ? result : PlanAlignmentChange.START_OF_BUNDLE;
      }

      private BillingActionPolicy getPlanChangePolicy(final PlanPhaseSpecifier from,
                                                      final PlanSpecifier to) throws CatalogApiException {
          final BillingActionPolicy result = DefaultCaseChange.getResult(changeCase, from, to, root);
          return (result != null) ? result : BillingActionPolicy.END_OF_TERM;
      }
  ```
  [Applies to]: Policy rule (Priority P1) — catalog module (DefaultPlanRules)
  [Confidence score]: High

- [Kill-BR-0184] - Parked accounts are skipped for automatic invoicing
  [Description]: An account tagged as 'parked' (auto-suspended due to a prior invoicing error) has all non-API-triggered (i.e. scheduled/system) invoice generation runs silently skipped; API-initiated invoice generation still proceeds even for a parked account. For example: given Account X has the PARK system tag set from a previous failed invoicing run, when The nightly invoicing job (isApiCall=false) processes account X, then invoice generation is skipped entirely and an empty invoice list is returned; a direct client API call to generate an invoice for account X, however, still runs. Edge cases: If checking the parked-tag itself fails (TagApiException), processing proceeds as if not parked.
  [Line Numbers]: 303 to 320
  [Source Code File Name]: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  [Source Code]:
  ```java
      public List<Invoice> processAccount(final boolean isApiCall,
                                          final UUID accountId,
                                          @Nullable final LocalDate targetDate,
                                          @Nullable final DryRunArguments dryRunArguments,
                                          final boolean isRescheduled,
                                          final boolean allowSplitting,
                                          final Iterable<PluginProperty> properties,
                                          final InternalCallContext context) throws InvoiceApiException {
          boolean parkedAccount = false;
          try {
              parkedAccount = parkedAccountsManager.isParked(context);
              if (parkedAccount && !isApiCall) {
                  log.warn("Ignoring invoice generation process for accountId='{}', targetDate='{}', account is parked", accountId.toString(), targetDate);
                  return Collections.emptyList();
              }
          } catch (final TagApiException e) {
              log.warn("Unable to determine parking state for accountId='{}'", accountId);
          }
  ```
  [Applies to]: Policy rule (Priority P0) — invoice module (InvoiceDispatcher)
  [Confidence score]: High
