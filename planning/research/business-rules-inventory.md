# Kill Bill — Business Rules & Unique-IP Inventory

## Methodology
Rules were identified using the graphify knowledge graph (`graphify-out/GRAPH_REPORT.md`) as an
orientation map (community list, God Nodes, Hyperedges), then verified against the actual source
in `account/`, `catalog/`, `subscription/`, `entitlement/`, `invoice/`, `payment/`, `overdue/`,
`usage/`, `tenant/`, and `currency/`. No rule below was scored from graph labels alone — each was
confirmed by reading the cited source file(s). Purely technical/infrastructure logic (DI wiring,
REST plumbing, generic caching) was excluded unless it encodes a business policy. Rules are
numbered sequentially in the order documented, which doubles as the module grouping order: account,
catalog, subscription, entitlement, invoice, payment, overdue, usage, tenant, currency/tax.

## Summary
- Total rules: 166
- By module: account 17, catalog 31, subscription 15, entitlement 11, invoice 35, payment 35,
  overdue 10, usage 5, tenant 6, currency/tax 1
- Confidence distribution: 161 High, 5 Medium, 0 Low
- Medium-confidence entries (none scored Low): Kill-BR-0017 (account — fixed-offset timezone rule's
  decision logic lives in the util module, not account, so account only calls/persists it),
  Kill-BR-0063 (subscription — add-on BCD interaction traced to a TODO-noted gap in
  calculateBcdForAlignment), Kill-BR-0074 (entitlement — overdue module's contribution to blocking
  state is inferred from the generic per-service mechanism, not read directly since no overdue
  source was in scope for that survey), Kill-BR-0088 (invoice — AUTO_INVOICING_REUSE_DRAFT reuse
  logic), Kill-BR-0136 (payment — isApiPayment retry-gating rationale documented in a comment but
  not independently re-traced through the full call graph).

## Business Rules

- [Kill-BR-0001] - Immutable account external key
  Description: An account's external key can never be changed once set. Enforced in AccountModelDao.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:184-189): if the current account already has a non-null externalKey and the proposed value differs, throws IllegalArgumentException. Duplicated (legacy path) in DefaultAccount.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:514-519).
  Line Numbers: 184 to 189
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          if ((ignoreNullInput || externalKey != null) &&
              currentAccount.getExternalKey() != null &&
              !currentAccount.getExternalKey().equals(externalKey)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account external key: new=%s, current=%s",
                                                               externalKey, currentAccount.getExternalKey()));
          }
  ```
  Line Numbers: 514 to 519
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  Source Code:
  ```java
          if ((ignoreNullInput || externalKey != null) &&
              currentAccount.getExternalKey() != null &&
              !currentAccount.getExternalKey().equals(externalKey)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account external key yet: new=%s, current=%s",
                                                               externalKey, currentAccount.getExternalKey()));
          }
  ```
  Applies to: Account update flow (DefaultAccountDao.update)
  Confidence score: High

- [Kill-BR-0002] - Immutable account currency once set
  Description: An account's currency cannot be changed once it has a non-null currency value. AccountModelDao.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:191-196) throws IllegalArgumentException "Killbill doesn't support updating the account currency" if the new currency differs from the existing non-null currency. Same rule duplicated in DefaultAccount.java:521-526.
  Line Numbers: 191 to 196
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          if ((ignoreNullInput || currency != null) &&
              currentAccount.getCurrency() != null &&
              !currentAccount.getCurrency().equals(currency)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account currency: new=%s, current=%s",
                                                               currency, currentAccount.getCurrency()));
          }
  ```
  Line Numbers: 521 to 526
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  Source Code:
  ```java
          if ((ignoreNullInput || currency != null) &&
              currentAccount.getCurrency() != null &&
              !currentAccount.getCurrency().equals(currency)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account currency yet: new=%s, current=%s",
                                                               currency, currentAccount.getCurrency()));
          }
  ```
  Applies to: Account update flow
  Confidence score: High

- [Kill-BR-0003] - Immutable billing-cycle-day (BCD) once set, feature-flag gated
  Description: Once an account has a non-default BCD (billingCycleDayLocal != 0, the sentinel meaning "unset"), it cannot be changed via a normal update unless the `allowAccountBCDUpdate` feature flag is enabled. See AccountModelDao.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:198-203) and the calling logic in DefaultAccountDao.update (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:260-274), which also decides whether to accept the proposed BCD or silently keep the current one based on `killbillFeatures.allowAccountBCDUpdate()`.
  Line Numbers: 198 to 203
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          if ((ignoreNullInput || (billingCycleDayLocal != DEFAULT_BILLING_CYCLE_DAY_LOCAL)) &&
              !allowAccountBCDUpdate && // BCD update is disallowed
              currentAccount.getBillingCycleDayLocal() != DEFAULT_BILLING_CYCLE_DAY_LOCAL && // There is already a BCD set
              !currentAccount.getBillingCycleDayLocal().equals(billingCycleDayLocal)) { // and it does not match what we have
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account BCD: new=%s, current=%s", billingCycleDayLocal, currentAccount.getBillingCycleDayLocal()));
          }
  ```
  Line Numbers: 260 to 274
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  Source Code:
  ```java
              specifiedAccount.validateAccountUpdateInput(currentAccount, treatNullValueAsReset, killbillFeatures.allowAccountBCDUpdate());

              if (!treatNullValueAsReset) {
                  if ((currentAccount.getBillingCycleDayLocal() == DEFAULT_BILLING_CYCLE_DAY_LOCAL && // There is *not* already a BCD set
                       specifiedAccount.getBillingCycleDayLocal() != DEFAULT_BILLING_CYCLE_DAY_LOCAL) || // and the proposed date is not 0
                      killbillFeatures.allowAccountBCDUpdate()) {
                      // Use the specified BCD
                  } else {
                      // Keep the current BCD
                      specifiedAccount.setBillingCycleDayLocal(currentAccount.getBillingCycleDayLocal());
                  }

                  // Set unspecified (null) fields to their current values
                  specifiedAccount.mergeWithDelegate(currentAccount);
              }
  ```
  Applies to: Account update / billing-cycle-day configuration, tied into invoice/junction billing alignment
  Confidence score: High

- [Kill-BR-0004] - BCD can only be set once via updateBCD API (separate from general update)
  Description: DefaultAccountInternalApi.updateBCD (account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java:86-99) throws AccountApiException(ACCOUNT_UPDATE_FAILED) if the account's BCD is already set (i.e., not equal to DEFAULT_BILLING_CYCLE_DAY_LOCAL). Only accounts with no BCD yet can have one assigned through this call.
  Line Numbers: 86 to 99
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java
  Source Code:
  ```java
      public void updateBCD(final String externalKey, final int bcd,
                            final InternalCallContext context) throws AccountApiException {
          final Account currentAccount = getAccountByKey(externalKey, context);
          if (currentAccount.getBillCycleDayLocal() != DefaultMutableAccountData.DEFAULT_BILLING_CYCLE_DAY_LOCAL) {
              throw new AccountApiException(ErrorCode.ACCOUNT_UPDATE_FAILED);
          }

          final MutableAccountData mutableAccountData = currentAccount.toMutableAccountData();
          mutableAccountData.setBillCycleDayLocal(bcd);
          final AccountModelDao accountToUpdate = new AccountModelDao(currentAccount.getId(), mutableAccountData);
          bcdCacheController.remove(currentAccount.getId());
          bcdCacheController.putIfAbsent(currentAccount.getId(), bcd);
          accountDao.update(accountToUpdate, true, context);
      }
  ```
  Applies to: Internal BCD-assignment flow, used to lazily compute/persist BCD the first time it's needed
  Confidence score: High

- [Kill-BR-0005] - Immutable account timezone once set
  Description: An account's timezone cannot be changed once set. AccountModelDao.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:205-210) throws IllegalArgumentException if currentAccount.getTimeZone() is non-null and differs from the proposed value.
  Line Numbers: 205 to 210
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          if ((ignoreNullInput || timeZone != null) &&
              currentAccount.getTimeZone() != null &&
              !currentAccount.getTimeZone().equals(timeZone)) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account timeZone: new=%s, current=%s",
                                                               timeZone, currentAccount.getTimeZone()));
          }
  ```
  Applies to: Account update flow; timezone drives billing date computations
  Confidence score: High

- [Kill-BR-0006] - Immutable account reference time (date-of-day only)
  Description: The account's referenceTime cannot be changed once set (day-of-month/time-of-day component compared via withMillisOfDay(0)). AccountModelDao.validateAccountUpdateInput (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:212-215) throws IllegalArgumentException if the proposed referenceTime's time-of-day differs from the current one.
  Line Numbers: 212 to 215
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          if (referenceTime != null && currentAccount.getReferenceTime().withMillisOfDay(0).compareTo(referenceTime.withMillisOfDay(0)) != 0) {
              throw new IllegalArgumentException(String.format("Killbill doesn't support updating the account referenceTime: new=%s, current=%s",
                                                               referenceTime, currentAccount.getReferenceTime()));
          }
  ```
  Applies to: Account update flow; referenceTime anchors billing cycle timing
  Confidence score: High

- [Kill-BR-0007] - External key uniqueness at account creation
  Description: DefaultAccountUserApi.createAccount (account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:85-89) checks getIdFromKey for an existing account with the same external key and throws AccountApiException(ACCOUNT_ALREADY_EXISTS) if found (with a code comment noting it's not transactional but backstopped by a DB constraint). A second check exists deeper in DefaultAccountDao.create via generateAlreadyExistsException (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:122-125).
  Line Numbers: 85 to 89
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java
  Source Code:
  ```java
      public Account createAccount(final AccountData data, final CallContext context) throws AccountApiException {
          // Not transactional, but there is a db constraint on that column
          if (data.getExternalKey() != null && getIdFromKey(data.getExternalKey(), context) != null) {
              throw new AccountApiException(ErrorCode.ACCOUNT_ALREADY_EXISTS, data.getExternalKey());
          }
  ```
  Line Numbers: 122 to 125
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  Source Code:
  ```java
      @Override
      protected AccountApiException generateAlreadyExistsException(final AccountModelDao account, final InternalCallContext context) {
          return new AccountApiException(ErrorCode.ACCOUNT_ALREADY_EXISTS, account.getExternalKey());
      }
  ```
  Applies to: Account creation
  Confidence score: High

- [Kill-BR-0008] - External key length limit (255 chars)
  Description: DefaultAccountUserApi.createAccount (account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:101-104) throws AccountApiException(EXTERNAL_KEY_LIMIT_EXCEEDED) if the external key exceeds 255 characters.
  Line Numbers: 101 to 104
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java
  Source Code:
  ```java
          final AccountModelDao account = new AccountModelDao(data);
          if (null != account.getExternalKey() && account.getExternalKey().length() > 255) {
              throw new AccountApiException(ErrorCode.EXTERNAL_KEY_LIMIT_EXCEEDED);
          }
  ```
  Applies to: Account creation
  Confidence score: High

- [Kill-BR-0009] - Parent account must exist before assignment
  Description: When creating an account with a non-null parentAccountId, DefaultAccountUserApi.createAccount (account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java:93-99) looks up the parent via ImmutableAccountInternalApi and throws AccountApiException(ACCOUNT_DOES_NOT_EXIST_FOR_ID) if the parent cannot be found, enforcing referential integrity for the parent/child account hierarchy (used for consolidated/delegated billing).
  Line Numbers: 93 to 99
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/user/DefaultAccountUserApi.java
  Source Code:
  ```java
          if (data.getParentAccountId() != null) {
              // verify that parent account exists if parentAccountId is not null
              final ImmutableAccountData immutableAccountData = immutableAccountInternalApi.getImmutableAccountDataById(data.getParentAccountId(), internalContext);
              if (immutableAccountData == null) {
                  throw new AccountApiException(ErrorCode.ACCOUNT_DOES_NOT_EXIST_FOR_ID, data.getParentAccountId());
              }
          }
  ```
  Applies to: Parent/child account creation flow
  Confidence score: High

- [Kill-BR-0010] - External key cannot resolve to null
  Description: DefaultAccountDao.getIdFromKey (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:240-247) throws AccountApiException(ACCOUNT_CANNOT_MAP_NULL_KEY) if a null external key is supplied for lookup, rather than silently returning null/no-match.
  Line Numbers: 240 to 247
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  Source Code:
  ```java
      public UUID getIdFromKey(final String externalKey, final InternalTenantContext context) throws AccountApiException {
          if (externalKey == null) {
              throw new AccountApiException(ErrorCode.ACCOUNT_CANNOT_MAP_NULL_KEY, "");
          }

          return transactionalSqlDao.execute(true, entitySqlDaoWrapperFactory ->
                  entitySqlDaoWrapperFactory.become(AccountSqlDao.class).getIdFromKey(externalKey, context));
      }
  ```
  Applies to: Account lookup-by-key flow
  Confidence score: High

- [Kill-BR-0011] - Account email must be unique by ID (no duplicate insert)
  Description: DefaultAccountDao.addEmail (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:330-341) throws AccountApiException(ACCOUNT_EMAIL_ALREADY_EXISTS) if an AccountEmailModelDao with the same id already exists before inserting a new one.
  Line Numbers: 330 to 341
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  Source Code:
  ```java
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
  Applies to: Account notification-email management
  Confidence score: High

- [Kill-BR-0012] - Payment-method update is a no-op when unchanged
  Description: DefaultAccountDao.updatePaymentMethod (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:296-327) explicitly checks whether the new paymentMethodId equals the current one (including both-null case) and bails out early without persisting or emitting an ACCOUNT_CHANGE bus event if there's no real change — a deliberate policy to avoid noisy change events for idempotent updates.
  Line Numbers: 296 to 327
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java
  Source Code:
  ```java
      public void updatePaymentMethod(final UUID accountId, final UUID paymentMethodId, final InternalCallContext context) throws AccountApiException {
          transactionalSqlDao.execute(false, AccountApiException.class, entitySqlDaoWrapperFactory -> {
              final AccountSqlDao transactional = entitySqlDaoWrapperFactory.become(AccountSqlDao.class);

              final AccountModelDao currentAccount = transactional.getById(accountId.toString(), context);
              if (currentAccount == null) {
                  throw new EntityPersistenceException(ErrorCode.ACCOUNT_DOES_NOT_EXIST_FOR_ID, accountId);
              }

              // Check if an update is really needed. If not, bail early to avoid sending an extra event on the bus
              if ((currentAccount.getPaymentMethodId() == null && paymentMethodId == null) ||
                  (currentAccount.getPaymentMethodId() != null && currentAccount.getPaymentMethodId().equals(paymentMethodId))) {
                  return null;
              }

              final String thePaymentMethodId = paymentMethodId != null ? paymentMethodId.toString() : null;
              final AccountModelDao account = (AccountModelDao) transactional.updatePaymentMethod(accountId.toString(), thePaymentMethodId, context);

              final AccountChangeInternalEvent changeEvent = new DefaultAccountChangeEvent(accountId, currentAccount, account,
                                                                                           context.getAccountRecordId(),
                                                                                           context.getTenantRecordId(),
                                                                                           context.getUserToken(),
                                                                                           context.getCreatedDate());

              try {
                  eventBus.postFromTransaction(changeEvent, entitySqlDaoWrapperFactory.getHandle().getConnection());
              } catch (final EventBusException e) {
                  log.warn("Failed to post account change event for accountId='{}'", accountId, e);
              }
              return null;
          });
      }
  ```
  Applies to: Payment method assignment/removal on an account
  Confidence score: High

- [Kill-BR-0013] - Update semantics — merge vs. reset (treatNullValueAsReset)
  Description: AccountModelDao.mergeWithDelegate (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:137-169) and its DefaultAccount counterpart (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:306-352) define the account "partial update" policy: most mutable fields (email, name, address, phone, etc.) take the new value if non-null, otherwise keep the current value; externalKey and currency are always forced back to the current value (immutable, see rules 1-2) regardless of what's supplied. This governs whether a null field in an update request means "clear it" or "leave unchanged," selected by the `treatNullValueAsReset` flag threaded from DefaultAccountUserApi.updateAccount (full-Account overload passes true/ignore-reset semantics; keyed-account/AccountData overloads pass false/merge semantics) down to DefaultAccountDao.update (account/src/main/java/org/killbill/billing/account/dao/DefaultAccountDao.java:250-274).
  Line Numbers: 137 to 169
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
      public void mergeWithDelegate(final AccountModelDao currentAccount) {
          setExternalKey(currentAccount.getExternalKey());

          setCurrency(currentAccount.getCurrency());

          // Note: the caller is responsible for setting the BCD

          // Set all updatable fields with the new values if non null, otherwise defaults to the current values
          setEmail(email != null ? email : currentAccount.getEmail());
          setName(name != null ? name : currentAccount.getName());
          final Integer firstNameLength = this.firstNameLength != null ? this.firstNameLength : currentAccount.getFirstNameLength();
          if (firstNameLength != null) {
              setFirstNameLength(firstNameLength);
          }
          setPaymentMethodId(paymentMethodId != null ? paymentMethodId : currentAccount.getPaymentMethodId());
          setTimeZone(timeZone != null ? timeZone : currentAccount.getTimeZone());
          setLocale(locale != null ? locale : currentAccount.getLocale());
          setAddress1(address1 != null ? address1 : currentAccount.getAddress1());
          setAddress2(address2 != null ? address2 : currentAccount.getAddress2());
          setCompanyName(companyName != null ? companyName : currentAccount.getCompanyName());
          setCity(city != null ? city : currentAccount.getCity());
          setStateOrProvince(stateOrProvince != null ? stateOrProvince : currentAccount.getStateOrProvince());
          setCountry(country != null ? country : currentAccount.getCountry());
          setPostalCode(postalCode != null ? postalCode : currentAccount.getPostalCode());
          setPhone(phone != null ? phone : currentAccount.getPhone());
          setNotes(notes != null ? notes : currentAccount.getNotes());
          setParentAccountId(parentAccountId != null ? parentAccountId : currentAccount.getParentAccountId());
          setIsPaymentDelegatedToParent(isPaymentDelegatedToParent != null ? isPaymentDelegatedToParent : currentAccount.getIsPaymentDelegatedToParent());
          final Boolean isMigrated = this.migrated != null ? this.migrated : currentAccount.getMigrated();
          if (isMigrated != null) {
              setMigrated(isMigrated);
          }
      }
  ```
  Line Numbers: 306 to 352
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  Source Code:
  ```java
      @Override
      @Deprecated // TODO Get rid of this in 0.22
      public Account mergeWithDelegate(final Account currentAccount) {
          final DefaultMutableAccountData accountData = new DefaultMutableAccountData(this);

          validateAccountUpdateInput(currentAccount, false);

          accountData.setExternalKey(currentAccount.getExternalKey());

          accountData.setCurrency(currentAccount.getCurrency());

          if (currentAccount.getBillCycleDayLocal() == DEFAULT_BILLING_CYCLE_DAY_LOCAL && // There is *not* already a BCD set
              billCycleDayLocal != null && // and the value proposed is not null
              billCycleDayLocal != DEFAULT_BILLING_CYCLE_DAY_LOCAL) {  // and the proposed date is not 0
              accountData.setBillCycleDayLocal(billCycleDayLocal);
          } else {
              accountData.setBillCycleDayLocal(currentAccount.getBillCycleDayLocal());
          }

          // Set all updatable fields with the new values if non null, otherwise defaults to the current values
          accountData.setEmail(email != null ? email : currentAccount.getEmail());
          accountData.setName(name != null ? name : currentAccount.getName());
          final Integer firstNameLength = this.firstNameLength != null ? this.firstNameLength : currentAccount.getFirstNameLength();
          if (firstNameLength != null) {
              accountData.setFirstNameLength(firstNameLength);
          }
          accountData.setPaymentMethodId(paymentMethodId != null ? paymentMethodId : currentAccount.getPaymentMethodId());
          accountData.setTimeZone(timeZone != null ? timeZone : currentAccount.getTimeZone());
          accountData.setLocale(locale != null ? locale : currentAccount.getLocale());
          accountData.setAddress1(address1 != null ? address1 : currentAccount.getAddress1());
          accountData.setAddress2(address2 != null ? address2 : currentAccount.getAddress2());
          accountData.setCompanyName(companyName != null ? companyName : currentAccount.getCompanyName());
          accountData.setCity(city != null ? city : currentAccount.getCity());
          accountData.setStateOrProvince(stateOrProvince != null ? stateOrProvince : currentAccount.getStateOrProvince());
          accountData.setCountry(country != null ? country : currentAccount.getCountry());
          accountData.setPostalCode(postalCode != null ? postalCode : currentAccount.getPostalCode());
          accountData.setPhone(phone != null ? phone : currentAccount.getPhone());
          accountData.setNotes(notes != null ? notes : currentAccount.getNotes());
          accountData.setParentAccountId(parentAccountId != null ? parentAccountId : currentAccount.getParentAccountId());
          accountData.setIsPaymentDelegatedToParent(isPaymentDelegatedToParent != null ? isPaymentDelegatedToParent : currentAccount.isPaymentDelegatedToParent());
          final Boolean isMigrated = this.isMigrated != null ? this.isMigrated : currentAccount.isMigrated();
          if (isMigrated != null) {
              accountData.setIsMigrated(isMigrated);
          }

          return new DefaultAccount(currentAccount.getId(), accountData);
      }
  ```
  Applies to: Account update API surface
  Confidence score: High

- [Kill-BR-0014] - BCD sentinel value and default-timezone-UTC on creation
  Description: DEFAULT_BILLING_CYCLE_DAY_LOCAL = 0 is treated as "no BCD set yet" throughout the module (comment: "0 has a special meaning in Junction" — account/src/main/java/org/killbill/billing/account/api/DefaultMutableAccountData.java:29-30). On account creation, AccountModelDao's constructor defaults externalKey to the account's UUID string and timeZone to DateTimeZone.UTC when not supplied (account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java:74-85, `withDefaults=true` path used by `new AccountModelDao(AccountData)`).
  Line Numbers: 29 to 30
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultMutableAccountData.java
  Source Code:
  ```java
      // 0 has a special meaning in Junction
      public static final int DEFAULT_BILLING_CYCLE_DAY_LOCAL = 0;
  ```
  Line Numbers: 74 to 85
  Source Code File Name: account/src/main/java/org/killbill/billing/account/dao/AccountModelDao.java
  Source Code:
  ```java
          super(id, createdDate, updatedDate);
          this.externalKey = !withDefaults ? externalKey : Objects.requireNonNullElse(externalKey, id.toString());
          this.email = email;
          this.name = name;
          this.firstNameLength = firstNameLength;
          this.currency = currency;
          this.parentAccountId = parentAccountId;
          this.isPaymentDelegatedToParent = isPaymentDelegatedToParent;
          this.billingCycleDayLocal = billingCycleDayLocal;
          this.paymentMethodId = paymentMethodId;
          this.referenceTime = referenceTime;
          this.timeZone = !withDefaults ? timeZone : Objects.requireNonNullElse(timeZone, DateTimeZone.UTC);
  ```
  Applies to: Account creation defaults; BCD/billing alignment sentinel semantics
  Confidence score: High

- [Kill-BR-0015] - isPaymentDelegatedToParent defaults to false
  Description: DefaultAccount's constructor coerces a null isPaymentDelegatedToParent input to false (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:145: `this.isPaymentDelegatedToParent = isPaymentDelegatedToParent != null ? isPaymentDelegatedToParent : false;`), i.e. by policy a child account does not delegate payment to its parent unless explicitly opted in.
  Line Numbers: 145 to 145
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  Source Code:
  ```java
          this.isPaymentDelegatedToParent = isPaymentDelegatedToParent != null ? isPaymentDelegatedToParent : false;
  ```
  Applies to: Parent/child account payment delegation
  Confidence score: High

- [Kill-BR-0016] - Cached BCD treats "unset" (0) as absent, forcing recompute
  Description: DefaultAccountInternalApi's BCD cache-loader callback (account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java:156-167) deliberately returns null (i.e., does not cache) when the stored BCD equals DEFAULT_BILLING_CYCLE_DAY_LOCAL, so an "unset" BCD is never cached as a real value and getBCD (line 101-108) falls back to returning the default (0) when nothing is cached.
  Line Numbers: 156 to 167
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java
  Source Code:
  ```java
      private CacheLoaderArgument createBCDCacheLoaderArgument(final InternalTenantContext context) {
          final AccountBCDCacheLoader.LoaderCallback loaderCallback = new AccountBCDCacheLoader.LoaderCallback() {
              @Override
              public Integer loadAccountBCD(final UUID accountId, final InternalTenantContext context) {
                  Integer result = accountDao.getAccountBCD(accountId, context);
                  if (result != null) {
                      // If the value is 0, then account BCD was not set so we don't want to create a cache entry
                      result = result.equals(DefaultMutableAccountData.DEFAULT_BILLING_CYCLE_DAY_LOCAL) ? null : result;
                  }
                  return result;
              }
          };
  ```
  Line Numbers: 101 to 108
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/svcs/DefaultAccountInternalApi.java
  Source Code:
  ```java
      @Override
      public int getBCD(final InternalTenantContext context) throws AccountApiException {
          final CacheLoaderArgument arg = createBCDCacheLoaderArgument(context);
          Preconditions.checkNotNull(context.getAccountRecordId(), "Context missing accountRecordId");
          final ImmutableAccountData account = immutableAccountInternalApi.getImmutableAccountDataByRecordId(context.getAccountRecordId(), context);
          final Integer result = bcdCacheController.get(account.getId(), arg);
          return result != null ? result : DefaultMutableAccountData.DEFAULT_BILLING_CYCLE_DAY_LOCAL;
      }
  ```
  Applies to: BCD read path / cache semantics tied to billing alignment
  Confidence score: High

- [Kill-BR-0017] - Fixed-offset timezone derived from account timezone + reference time (DST snapshot)
  Description: An account's "fixed offset timezone" (used for billing-date math so DST transitions don't shift billing dates) is computed once from the account's timezone and its immutable referenceTime: AccountDateTimeUtils.getFixedOffsetTimeZone checks whether DST was in effect at the referenceTime and pins either the DST offset or the standard offset (util/src/main/java/org/killbill/billing/util/account/AccountDateTimeUtils.java:36-44). It is invoked directly from the account module in DefaultAccount.getFixedOffsetTimeZone (account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java:354-357) and baked into the immutable account snapshot in DefaultImmutableAccountData's constructor (account/src/main/java/org/killbill/billing/account/api/DefaultImmutableAccountData.java:55-71). Note: rule's decision logic lives in the util module, not account module itself; account module only calls it and persists the derived value.
  Line Numbers: 36 to 44
  Source Code File Name: util/src/main/java/org/killbill/billing/util/account/AccountDateTimeUtils.java
  Source Code:
  ```java
      private static DateTimeZone getFixedOffsetTimeZone(final DateTimeZone referenceDateTimeZone, final DateTime referenceDateTime) {
          // Check if DST was in effect at the reference date time
          final boolean shouldUseDST = !referenceDateTimeZone.isStandardOffset(referenceDateTime.getMillis());
          if (shouldUseDST) {
              return DateTimeZone.forOffsetMillis(referenceDateTimeZone.getOffset(referenceDateTime.getMillis()));
          } else {
              return DateTimeZone.forOffsetMillis(referenceDateTimeZone.getStandardOffset(referenceDateTime.getMillis()));
          }
      }
  ```
  Line Numbers: 354 to 357
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultAccount.java
  Source Code:
  ```java
      @Override
      public DateTimeZone getFixedOffsetTimeZone() {
          return AccountDateTimeUtils.getFixedOffsetTimeZone(this);
      }
  ```
  Line Numbers: 55 to 71
  Source Code File Name: account/src/main/java/org/killbill/billing/account/api/DefaultImmutableAccountData.java
  Source Code:
  ```java
      public DefaultImmutableAccountData(final Account account) {
          this(account.getId(),
               account.getExternalKey(),
               account.getCurrency(),
               account.getTimeZone(),
               AccountDateTimeUtils.getFixedOffsetTimeZone(account),
               account.getReferenceTime());
      }

      public DefaultImmutableAccountData(final AccountModelDao account) {
          this(account.getId(),
               account.getExternalKey(),
               account.getCurrency(),
               account.getTimeZone(),
               AccountDateTimeUtils.getFixedOffsetTimeZone(account),
               account.getReferenceTime());
      }
  ```
  Applies to: Account timezone/date handling that feeds invoice and billing-cycle date calculations
  Confidence score: Medium

- [Kill-BR-0018] - Plan-change policy resolution via ordered rule matching
  Description: getPlanChangePolicy() walks the catalog's changeCase[] (DefaultCaseChange) array in declaration order and returns the BillingActionPolicy (IMMEDIATE/END_OF_TERM/ILLEGAL) of the first case whose from/to product, productCategory, billingPeriod and priceList (each optionally null = wildcard) match the requested change; falls back to END_OF_TERM if no case matches. See DefaultPlanRules.java:172-176 (getPlanChangePolicy) and DefaultCaseChange.java:86-137 (getResult/satisfies-match logic).
  Line Numbers: 172 to 176
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      private BillingActionPolicy getPlanChangePolicy(final PlanPhaseSpecifier from,
                                                      final PlanSpecifier to) throws CatalogApiException {
          final BillingActionPolicy result = DefaultCaseChange.getResult(changeCase, from, to, root);
          return (result != null) ? result : BillingActionPolicy.END_OF_TERM;
      }
  ```
  Line Numbers: 86 to 137
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseChange.java
  Source Code:
  ```java
      public T getResult(final PlanPhaseSpecifier from,
                         final PlanSpecifier to, final StaticCatalog catalog) throws CatalogApiException {

          final Product inFromProduct;
          final BillingPeriod inFromBillingPeriod;
          final ProductCategory inFromProductCategory;
          final PriceList inFromPriceList;
          if (from.getPlanName() != null) {
              final Plan plan = catalog.findPlan(from.getPlanName());
              inFromProduct = plan.getProduct();
              inFromBillingPeriod = plan.getRecurringBillingPeriod();
              inFromProductCategory = plan.getProduct().getCategory();
              inFromPriceList = plan.getPriceList();
          } else {
              inFromProduct = catalog.findProduct(from.getProductName());
              inFromBillingPeriod = from.getBillingPeriod();
              inFromProductCategory = inFromProduct.getCategory();
              inFromPriceList = from.getPriceListName() != null ? catalog.findPriceList(from.getPriceListName()) : null;
          }

          final Product inToProduct;
          final BillingPeriod inToBillingPeriod;
          final ProductCategory inToProductCategory;
          final PriceList inToPriceList;
          if (to.getPlanName() != null) {
              final Plan plan = catalog.findPlan(to.getPlanName());
              inToProduct = plan.getProduct();
              inToBillingPeriod = plan.getRecurringBillingPeriod();
              inToProductCategory = plan.getProduct().getCategory();
              inToPriceList =  plan.getPriceList();
          } else {
              inToProduct = catalog.findProduct(to.getProductName());
              inToBillingPeriod = to.getBillingPeriod();
              inToProductCategory = inToProduct.getCategory();
              inToPriceList = to.getPriceListName() != null ? catalog.findPriceList(to.getPriceListName()) : null;
          }

          if (
                  (phaseType == null || from.getPhaseType() == phaseType) &&
                  (fromProduct == null || fromProduct.equals(inFromProduct)) &&
                  (fromProductCategory == null || fromProductCategory.equals(inFromProductCategory)) &&
                  (fromBillingPeriod == null || fromBillingPeriod.equals(inFromBillingPeriod)) &&
                  (this.toProduct == null || this.toProduct.equals(inToProduct)) &&
                  (this.toProductCategory == null || this.toProductCategory.equals(inToProductCategory)) &&
                  (this.toBillingPeriod == null || this.toBillingPeriod.equals(inToBillingPeriod)) &&
                  (fromPriceList == null || fromPriceList.equals(inFromPriceList)) &&
                  (toPriceList == null || toPriceList.equals(inToPriceList))
          ) {
              return getResult();
          }
          return null;
      }
  ```
  Applies to: subscription plan-change (upgrade/downgrade) requests.
  Confidence score: High

- [Kill-BR-0019] - ILLEGAL plan-change rejection
  Description: getPlanChangeResult() throws IllegalPlanChange when the resolved BillingActionPolicy for a change is ILLEGAL, i.e. catalog authors can declare certain from-to transitions as forbidden. See DefaultPlanRules.java:156-159.
  Line Numbers: 156 to 159
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
          final BillingActionPolicy policy = getPlanChangePolicy(from, toWithPriceList);
          if (policy == BillingActionPolicy.ILLEGAL) {
              throw new IllegalPlanChange(from, toWithPriceList);
          }
  ```
  Applies to: subscription plan-change validation.
  Confidence score: High

- [Kill-BR-0020] - Plan-change realignment resolution
  Description: getPlanChangeAlignment() resolves a PlanAlignmentChange (e.g. START_OF_BUNDLE/START_OF_SUBSCRIPTION/CHANGE_OF_PLAN) via the same ordered CaseChange matching against changeAlignmentCase[], defaulting to START_OF_BUNDLE if unmatched. See DefaultPlanRules.java:166-170.
  Line Numbers: 166 to 170
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      private PlanAlignmentChange getPlanChangeAlignment(final PlanPhaseSpecifier from,
                                                         final PlanSpecifier to) throws CatalogApiException {
          final PlanAlignmentChange result = DefaultCaseChange.getResult(changeAlignmentCase, from, to, root);
          return (result != null) ? result : PlanAlignmentChange.START_OF_BUNDLE;
      }
  ```
  Applies to: recomputing billing-cycle-day/phase timeline on plan change.
  Confidence score: High

- [Kill-BR-0021] - Cancellation policy resolution
  Description: getPlanCancelPolicy() resolves BillingActionPolicy (IMMEDIATE vs END_OF_TERM) for subscription cancellation by matching cancelCase[] (DefaultCaseCancelPolicy, phase-aware matching via DefaultCasePhase) against the plan/phase being cancelled, defaulting to END_OF_TERM. See DefaultPlanRules.java:132-135 and DefaultCasePhase.java:43-49.
  Line Numbers: 132 to 135
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      public BillingActionPolicy getPlanCancelPolicy(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingActionPolicy result = DefaultCasePhase.getResult(cancelCase, planPhase, root);
          return (result != null) ? result : BillingActionPolicy.END_OF_TERM;
      }
  ```
  Line Numbers: 43 to 49
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCasePhase.java
  Source Code:
  ```java
      public T getResult(final PlanPhaseSpecifier specifier, final StaticCatalog c) throws CatalogApiException {
          if ((phaseType == null || specifier.getPhaseType() == phaseType)
              && satisfiesCase(new PlanSpecifier(specifier), c)) {
              return getResult();
          }
          return null;
      }
  ```
  Applies to: subscription cancellation.
  Confidence score: High

- [Kill-BR-0022] - Billing alignment resolution (ACCOUNT vs BUNDLE vs SUBSCRIPTION)
  Description: getBillingAlignment() resolves a BillingAlignment for a given plan/phase from billingAlignmentCase[], defaulting to ACCOUNT when no case matches. See DefaultPlanRules.java:137-141.
  Line Numbers: 137 to 141
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      @Override
      public BillingAlignment getBillingAlignment(final PlanPhaseSpecifier planPhase) throws CatalogApiException {
          final BillingAlignment result = DefaultCasePhase.getResult(billingAlignmentCase, planPhase, root);
          return (result != null) ? result : BillingAlignment.ACCOUNT;
      }
  ```
  Applies to: determining what billing-cycle-day a subscription's phase aligns to.
  Confidence score: High

- [Kill-BR-0023] - Create-alignment resolution (START_OF_BUNDLE default)
  Description: getPlanCreateAlignment() resolves PlanAlignmentCreate from createAlignmentCase[], defaulting to START_OF_BUNDLE if unmatched. See DefaultPlanRules.java:126-129.
  Line Numbers: 126 to 129
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      public PlanAlignmentCreate getPlanCreateAlignment(final PlanSpecifier specifier) throws CatalogApiException {
          final PlanAlignmentCreate result = DefaultCase.getResult(createAlignmentCase, specifier, root);
          return (result != null) ? result : PlanAlignmentCreate.START_OF_BUNDLE;
      }
  ```
  Applies to: new subscription creation, determines whether phase timeline aligns to bundle start or subscription start.
  Confidence score: High

- [Kill-BR-0024] - Price-list-transition rule on plan change
  Description: findPriceList()/getPlanChangeResult() determine the destination price list for a plan change by matching priceListCase[] (DefaultCasePriceList); if no rule matches, falls back to the plan's current price list (from-plan lookup) rather than an arbitrary default. See DefaultPlanRules.java:143-164, 178-185 and DefaultCasePriceList.java:56-83.
  Line Numbers: 143 to 164
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
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
  Line Numbers: 178 to 185
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
  ```java
      private DefaultPriceList findPriceList(final PlanSpecifier specifier) throws CatalogApiException {
          DefaultPriceList result = DefaultCasePriceList.getResult(priceListCase, specifier, root);
          if (result == null) {
              final String priceListName = specifier.getPlanName() != null ? root.findPlan(specifier.getPlanName()).getPriceList().getName() : specifier.getPriceListName();
              result = (DefaultPriceList) root.findPriceList(priceListName);
          }
          return result;
      }
  ```
  Line Numbers: 56 to 83
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCasePriceList.java
  Source Code:
  ```java
      @Override
      public DefaultProduct getProduct() {
          return fromProduct;
      }

      @Override
      public ProductCategory getProductCategory() {
          return fromProductCategory;
      }

      @Override
      public BillingPeriod getBillingPeriod() {
          return fromBillingPeriod;
      }

      @Override
      public DefaultPriceList getPriceList() {
          return fromPriceList;
      }

      @Override
      public PriceList getDestinationPriceList() {
          return toPriceList;
      }

      protected DefaultPriceList getResult() {
          return toPriceList;
      }
  ```
  Applies to: plan change, determines which price list the new plan is looked up in.
  Confidence score: High

- [Kill-BR-0025] - First-match-wins rule evaluation order
  Description: All catalog rule engines (DefaultCase.getResult, DefaultCaseChange.getResult, DefaultCasePhase.getResult) iterate the declared case array in order and return the first case whose (nullable = wildcard) criteria all match; this makes catalog XML rule ordering semantically significant (more specific rules must be declared before more general ones). See DefaultCase.java:77-88, DefaultCaseChange.java:139-151.
  Line Numbers: 77 to 88
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCase.java
  Source Code:
  ```java
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
  Line Numbers: 139 to 151
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultCaseChange.java
  Source Code:
  ```java
      public static <K> K getResult(final DefaultCaseChange<K>[] cases, final PlanPhaseSpecifier from,
                                    final PlanSpecifier to, final StaticCatalog catalog) throws CatalogApiException {
          if (cases != null) {
              for (final DefaultCaseChange<K> cc : cases) {
                  final K result = cc.getResult(from, to, catalog);
                  if (result != null) {
                      return result;
                  }
              }
          }
          return null;

      }
  ```
  Applies to: every rule category (change policy/alignment, cancel policy, billing alignment, create alignment, price-list transition).
  Confidence score: High

- [Kill-BR-0026] - Mandatory catch-all default rule + duplicate-rule rejection
  Description: DefaultPlanRules.validate() requires that changeCase[] and cancelCase[] each contain at least one fully-wildcard ("default") case, else validation fails with "Missing default rule case for plan change/cancellation"; it also rejects duplicate rule definitions (via HashSet+equals) across all six rule categories (change policy, change alignment, cancel policy, create alignment, billing alignment, price-list). See DefaultPlanRules.java:188-277.
  Line Numbers: 188 to 277
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/rules/DefaultPlanRules.java
  Source Code:
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

          final HashSet<DefaultCaseChangePlanAlignment> caseChangePlanAlignmentsSet = new HashSet<DefaultCaseChangePlanAlignment>();
          for (final DefaultCaseChangePlanAlignment cur : changeAlignmentCase) {
              if (caseChangePlanAlignmentsSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for plan change alignment %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  caseChangePlanAlignmentsSet.add(cur);
              }
              cur.validate(catalog, errors);
          }

          final HashSet<DefaultCaseCreateAlignment> caseCreateAlignmentsSet = new HashSet<DefaultCaseCreateAlignment>();
          for (final DefaultCaseCreateAlignment cur : createAlignmentCase) {
              if (caseCreateAlignmentsSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for create plan alignment %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  caseCreateAlignmentsSet.add(cur);
              }
              cur.validate(catalog, errors);
          }

          final HashSet<DefaultCaseBillingAlignment> caseBillingAlignmentsSet = new HashSet<DefaultCaseBillingAlignment>();
          for (final DefaultCaseBillingAlignment cur : billingAlignmentCase) {
              if (caseBillingAlignmentsSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for billing alignment %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  caseBillingAlignmentsSet.add(cur);
              }
              cur.validate(catalog, errors);
          }

          final HashSet<DefaultCasePriceList> casePriceListsSet = new HashSet<DefaultCasePriceList>();
          for (final DefaultCasePriceList cur : priceListCase) {
              if (casePriceListsSet.contains(cur)) {
                  errors.add(new ValidationError(String.format("Duplicate rule for price list transition %s", cur.toString()), DefaultPlanRules.class, ""));
              } else {
                  casePriceListsSet.add(cur);
              }
              cur.validate(catalog, errors);
          }
          return errors;
  ```
  Applies to: catalog XML authoring/validation — guarantees every plan/phase combination always resolves to a policy.
  Confidence score: High

- [Kill-BR-0027] - Effective-dated catalog version selection
  Description: DefaultVersionedCatalog.indexOfVersionForDate() picks, for a given request date, the latest version whose effectiveDate <= that date (scanning newest-to-oldest); if every version is still in the future relative to the date, it falls back to returning the earliest version rather than failing. See DefaultVersionedCatalog.java:91-107.
  Line Numbers: 91 to 107
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java
  Source Code:
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
  Applies to: resolving "which catalog version applies" for a subscription/invoice/plan lookup at a point in time.
  Confidence score: High

- [Kill-BR-0028] - Duplicate catalog-version effective-date rejection
  Description: DefaultVersionedCatalog.validate() rejects a set of versions where two StandaloneCatalog versions share the same effectiveDate, and rejects any version whose catalogName differs from the versioned catalog's name. See DefaultVersionedCatalog.java:134-149.
  Line Numbers: 134 to 149
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java
  Source Code:
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
  ```
  Applies to: multi-version catalog upload/validation.
  Confidence score: High

- [Kill-BR-0029] - Uniform plan-shape-across-versions constraint
  Description: validateUniformPlanShapeAcrossVersions()/validatePlanShape() enforce that if a plan with the same name exists in a later catalog version, it must have the same number of phases and the phases must appear in the same name order — i.e. an existing plan's phase structure can't be silently reshaped across catalog versions (though it can be re-priced). See DefaultVersionedCatalog.java:156-192.
  Line Numbers: 156 to 192
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultVersionedCatalog.java
  Source Code:
  ```java
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
  Applies to: catalog versioning/publishing new catalog XML for an existing tenant.
  Confidence score: High

- [Kill-BR-0030] - EVERGREEN-duration-must-be-UNLIMITED constraint
  Description: StandaloneCatalog.validatePlanDuration() enforces that a phase of type EVERGREEN must have duration UNLIMITED, and any non-EVERGREEN phase must NOT have duration UNLIMITED. See StandaloneCatalog.java:290-310.
  Line Numbers: 290 to 310
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java
  Source Code:
  ```java
      private ValidationErrors validatePlanDuration(final StaticCatalog newCatalogVersion, final ValidationErrors errors) {
          for (final Plan plan : newCatalogVersion.getPlans()) {
              PlanPhase[] planPhases = plan.getAllPhases();
              for (int i = 0; i < planPhases.length; i++) {
                  if (planPhases[i].getPhaseType().name().equals(PhaseType.EVERGREEN.name())
                      && !planPhases[i].getDuration().getUnit().name().equals(TimeUnit.UNLIMITED.name())) {
                      errors.add(new ValidationError(String.format(
                              "EVERGREEN Phase '%s' for plan '%s' in version '%s' must have duration as UNLIMITED'",
                              planPhases[i].getName(), plan.getName(), plan.getCatalog().getEffectiveDate()),
                                                     DefaultVersionedCatalog.class, ""));
                  } else if (!planPhases[i].getPhaseType().name().equals(PhaseType.EVERGREEN.name())
                             && planPhases[i].getDuration().getUnit().name().equals(TimeUnit.UNLIMITED.name())) {
                      errors.add(new ValidationError(String.format(
                              "'%s' Phase '%s' for plan '%s' in version '%s' must not have duration as UNLIMITED'",
                              planPhases[i].getPhaseType().name(), planPhases[i].getName(), plan.getName(),
                              plan.getCatalog().getEffectiveDate()), DefaultVersionedCatalog.class, ""));
                  }
              }
          }
          return errors;
      }
  ```
  Applies to: plan/phase authoring — trial/discount/fixed-term phases must end, evergreen phases must not.
  Confidence score: High

- [Kill-BR-0031] - Phase-type placement constraints on a plan
  Description: DefaultPlan.validate() rejects an initial phase of type EVERGREEN, and rejects a final phase of type TRIAL or DISCOUNT — i.e. EVERGREEN can only be the terminal (final) phase, and TRIAL/DISCOUNT can never be terminal. See DefaultPlan.java:304-319.
  Line Numbers: 304 to 319
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  Source Code:
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
  Applies to: plan phase-sequence definition.
  Confidence score: High

- [Kill-BR-0032] - Price-effective-date-can't-precede-catalog-version rule
  Description: DefaultPlan.validate() rejects a plan whose effectiveDateForExistingSubscriptions predates the containing catalog version's own effectiveDate — an existing-subscription price change can't be back-dated before the catalog version that declares it. See DefaultPlan.java:286-293.
  Line Numbers: 286 to 293
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  Source Code:
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
  Applies to: mid-term price changes applied to existing subscriptions.
  Confidence score: High

- [Kill-BR-0033] - Recurring-billing-mode requirement for recurring (non-usage-only) plans
  Description: DefaultPlan.validate() requires recurringBillingMode to be set for any plan whose recurring billing period is not NO_BILLING_PERIOD, but exempts pure usage-based plans from needing one; DefaultPlan.initialize() inherits the catalog-level recurringBillingMode when the plan doesn't declare its own. See DefaultPlan.java:295-298 and 276-278.
  Line Numbers: 295 to 298
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  Source Code:
  ```java
          // Pure usage based plans would not have a recurringBillingMode
          if (!BillingPeriod.NO_BILLING_PERIOD.equals(getRecurringBillingPeriod()) && recurringBillingMode == null) {
              errors.add(new ValidationError(String.format("Invalid recurring billingMode for plan '%s'", name), DefaultPlan.class, ""));
          }
  ```
  Line Numbers: 276 to 278
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  Source Code:
  ```java
          if (recurringBillingMode == null) {
              this.recurringBillingMode = catalog.getRecurringBillingMode();
          }
  ```
  Applies to: plan definition — distinguishes recurring vs. usage-only billing plans.
  Confidence score: High

- [Kill-BR-0034] - Currency-support enforcement on prices
  Description: DefaultInternationalPrice.validate() rejects any declared price whose Currency is not in the catalog's supportedCurrencies list, and rejects negative price values. See DefaultInternationalPrice.java:108-127.
  Line Numbers: 108 to 127
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  Source Code:
  ```java
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
          return errors;
      }
  ```
  Applies to: every priced element (fixed price, recurring price, tiered/usage block price).
  Confidence score: High

- [Kill-BR-0035] - Zero-price-in-all-currencies default rule
  Description: DefaultInternationalPrice.getPrice() returns BigDecimal.ZERO for any currency when the price element has no <price> entries at all (documented "No price is a zero cost plan in all currencies"); otherwise it throws CAT_NO_PRICE_FOR_CURRENCY if the specific currency isn't listed. See DefaultInternationalPrice.java:88-100.
  Line Numbers: 88 to 100
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultInternationalPrice.java
  Source Code:
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
  Applies to: fixed/recurring/usage pricing lookups.
  Confidence score: High

- [Kill-BR-0036] - Price-list plan lookup with DEFAULT-price-list fallback
  Description: DefaultPriceListSet.getPlanFrom() looks up a plan for (product, billingPeriod) first within the requested price list, and if no match is found there, falls back to searching the DEFAULT price list rather than failing immediately. See DefaultPriceListSet.java:64-83.
  Line Numbers: 64 to 83
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java
  Source Code:
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
  Applies to: plan selection when creating a subscription with product+billingPeriod+priceList.
  Confidence score: High

- [Kill-BR-0037] - Ambiguous-plan-for-pricelist rejection
  Description: The same getPlanFrom() throws CAT_MULTIPLE_MATCHING_PLANS_FOR_PRICELIST if more than one plan matches (product, billingPeriod) within a price list — a price list may not offer two plans with identical product+billing period. See DefaultPriceListSet.java:74-82.
  Line Numbers: 74 to 82
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java
  Source Code:
  ```java
          switch (plans.size()) {
              case 0:
                  return null;
              case 1:
                  return plans.iterator().next();
              default:
                  throw new CatalogApiException(ErrorCode.CAT_MULTIPLE_MATCHING_PLANS_FOR_PRICELIST,
                                                priceListName, product.getName(), period);
          }
  ```
  Applies to: price-list plan authoring/lookup.
  Confidence score: High

- [Kill-BR-0038] - Reserved "DEFAULT" price-list name protection
  Description: DefaultPriceListSet.validate() and PriceListDefault.validate() enforce that "DEFAULT" is a reserved price-list name: no child price list may be named DEFAULT, and the actual default price list's name must equal it. See DefaultPriceListSet.java:107-118 and PriceListDefault.java:36-45.
  Line Numbers: 107 to 118
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPriceListSet.java
  Source Code:
  ```java
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          defaultPricelist.validate(catalog, errors);
          //Check that the default pricelist name is not in use in the children
          for (final DefaultPriceList pl : childPriceLists) {
              if (pl.getName().equals(PriceListSet.DEFAULT_PRICELIST_NAME)) {
                  errors.add(new ValidationError("Pricelists cannot use the reserved name '" + PriceListSet.DEFAULT_PRICELIST_NAME + "'",
                                                 DefaultPriceListSet.class, pl.getName()));
              }
              pl.validate(catalog, errors); // and validate the individual pricelists
          }
          return errors;
      }
  ```
  Line Numbers: 36 to 45
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/PriceListDefault.java
  Source Code:
  ```java
      @Override
      public ValidationErrors validate(final StandaloneCatalog catalog, final ValidationErrors errors) {
          super.validate(catalog, errors);
          if (!getName().equals(PriceListSet.DEFAULT_PRICELIST_NAME)) {
              errors.add(new ValidationError("The name of the default pricelist must be 'DEFAULT'",
                                             DefaultPriceList.class, getName()));

          }
          return errors;
      }
  ```
  Applies to: catalog price-list authoring.
  Confidence score: High

- [Kill-BR-0039] - Usage/tier structural requirements by billing mode and usage type
  Description: DefaultUsage.validate() requires IN_ADVANCE+CAPACITY usage to declare limits and IN_ADVANCE+CONSUMABLE usage to declare blocks; IN_ARREAR usage must declare at least one tier. DefaultTier.validate() further requires IN_ARREAR+CAPACITY tiers to declare limits and IN_ARREAR+CONSUMABLE tiers to declare tiered blocks. See DefaultUsage.java:212-229 and DefaultTier.java:148-159.
  Line Numbers: 212 to 229
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultUsage.java
  Source Code:
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
  Line Numbers: 148 to 159
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultTier.java
  Source Code:
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
  Applies to: usage-based/metered pricing definitions.
  Confidence score: High

- [Kill-BR-0040] - Unit-limit compliance cascades from usage to product level
  Description: DefaultPlanPhase.compliesWithLimits() checks each phase-level Usage section's limits first, and only if all pass does it fall through to DefaultProduct.compliesWithLimits() for a product-wide limit on that unit — i.e. product-level limits act as an overall ceiling beneath any usage-section limit. See DefaultPlanPhase.java:129-138 and DefaultProduct.java:156-172.
  Line Numbers: 129 to 138
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlanPhase.java
  Source Code:
  ```java
      public boolean compliesWithLimits(final String unit, final BigDecimal value) {
          // First check usage section
          for (final DefaultUsage usage : usages) {
              if (!usage.compliesWithLimits(unit, value)) {
                  return false;
              }
          }
          // Second, check if there are limits defined at the product section.
          return product.compliesWithLimits(unit, value);
      }
  ```
  Line Numbers: 156 to 172
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultProduct.java
  Source Code:
  ```java
      protected Limit findLimit(String unit) {
          for (Limit limit : limits) {
              if (limit.getUnit().getName().equals(unit)) {
                  return limit;
              }
          }
          return null;
      }

      @Override
      public boolean compliesWithLimits(String unit, BigDecimal value) {
          Limit l = findLimit(unit);
          if (l == null) {
              return true;
          }
          return l.compliesWith(value);
      }
  ```
  Applies to: enforcing consumption caps for usage/metered billing.
  Confidence score: High

- [Kill-BR-0041] - Limit min/max bound validation
  Description: DefaultLimit.validate() rejects a Limit definition where max < min. At evaluation time, DefaultLimit.compliesWith(value) fails a value that exceeds max when max is set, and (as literally coded) additionally requires value <= min whenever a min bound is set. See DefaultLimit.java:84-89, 100-105.
  Line Numbers: 84 to 89
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultLimit.java
  Source Code:
  ```java
      public ValidationErrors validate(final StandaloneCatalog root, final ValidationErrors errors) {
          if (maxHasValue && minHasValue && max.compareTo(min) < 0) {
              errors.add(new ValidationError("max must be greater than min", Limit.class, ""));
          }
          return errors;
      }
  ```
  Line Numbers: 100 to 105
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultLimit.java
  Source Code:
  ```java
      public boolean compliesWith(final BigDecimal value) {
          if (maxHasValue && value.compareTo(max) > 0) {
              return false;
          }
          return !minHasValue || value.compareTo(min) <= 0;
      }
  ```
  Applies to: usage/product unit-limit enforcement.
  Confidence score: High

- [Kill-BR-0042] - Add-on availability/inclusion governs listing
  Description: StandaloneCatalog.getAvailableAddOnListings() only lists an add-on plan as available for a base product if that add-on's Product is present in the base product's "available" collection (DefaultProduct.getAvailable()), cross-referenced against every price list; base-plan listing (getAvailableBasePlanListings) is similarly restricted to BASE-category products present in a price list. See StandaloneCatalog.java:340-382 and DefaultProduct.java:99-103.
  Line Numbers: 340 to 382
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/StandaloneCatalog.java
  Source Code:
  ```java
      public List<Listing> getAvailableAddOnListings(final String baseProductName, @Nullable final String priceListName) {
          final List<Listing> availAddons = new ArrayList<Listing>();

          try {
              final Product product = findProduct(baseProductName);
              if (product != null) {
                  for (final Product availAddon : product.getAvailable()) {
                      for (final BillingPeriod billingPeriod : BillingPeriod.values()) {
                          for (final PriceList priceList : getPriceLists().getAllPriceLists()) {
                              if (priceListName == null || priceListName.equals(priceList.getName())) {
                                  final Collection<Plan> addonInList = priceList.findPlans(availAddon, billingPeriod);
                                  for (final Plan cur : addonInList) {
                                      availAddons.add(new DefaultListing(cur, priceList));
                                  }
                              }
                          }
                      }
                  }
              }
          } catch (final CatalogApiException e) {
              // No such product - just return an empty list
          }
          return availAddons;
      }

      @Override
      public List<Listing> getAvailableBasePlanListings() {
          final List<Listing> availBasePlans = new ArrayList<Listing>();

          for (final Plan plan : getPlans()) {
              if (plan.getProduct().getCategory().equals(ProductCategory.BASE)) {
                  for (final PriceList priceList : getPriceLists().getAllPriceLists()) {
                      for (final Plan priceListPlan : priceList.getPlans()) {
                          if (priceListPlan.getName().equals(plan.getName()) &&
                              priceListPlan.getProduct().getName().equals(plan.getProduct().getName())) {
                              availBasePlans.add(new DefaultListing(priceListPlan, priceList));
                          }
                      }
                  }
              }
          }
          return availBasePlans;
      }
  ```
  Line Numbers: 99 to 103
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultProduct.java
  Source Code:
  ```java
      @Override
      public Collection<Product> getAvailable() {
          // Workaround for existing catalogs which have self-referencing products. Shouldn't happen anymore.
          return available.getEntries().stream().filter(c -> c != this).collect(Collectors.toList());
      }
  ```
  Applies to: add-on/base-plan compatibility and discoverability for subscription creation.
  Confidence score: High

- [Kill-BR-0043] - First-non-zero-recurring-charge computation (trial/discount skip-through)
  Description: DefaultPlan.dateOfFirstRecurringNonZeroCharge() walks a plan's phases in order, skipping (advancing the date past) any phase whose duration isn't UNLIMITED and whose recurring price is null/zero, stopping at the first phase with a non-zero recurring charge or an UNLIMITED-duration phase — used to determine when a subscription actually starts being charged after trial/discount phases. See DefaultPlan.java:328-352.
  Line Numbers: 328 to 352
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/DefaultPlan.java
  Source Code:
  ```java
      @Override
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
      }
  ```
  Applies to: trial/discount phase handling, invoicing start date determination.
  Confidence score: High

- [Kill-BR-0044] - Per-tenant price-override validation (can't add a price dimension that doesn't exist)
  Description: DefaultPriceOverrideSvc.getOrCreateOverriddenPlan() throws CAT_INVALID_INVALID_PRICE_OVERRIDE if an override supplies a fixed price for a phase that has no fixed-price section in the base catalog, or a recurring price for a phase with no recurring section — overrides can only re-price an existing pricing dimension, not introduce a new one. See DefaultPriceOverrideSvc.java:104-119.
  Line Numbers: 104 to 119
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java
  Source Code:
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
  Applies to: per-tenant catalog price overrides (custom/negotiated pricing).
  Confidence score: High

- [Kill-BR-0045] - Deterministic overridden-plan naming/identity rule
  Description: getOrCreateOverriddenPlan() derives the overridden plan's name deterministically from the parent plan name plus either the persisted override-definition record id ("<plan>-<recordId>") for real overrides or a monotonically-incrementing dry-run index ("<plan>-dryrun-N") for preview/no-context calls, so identical overrides map to the same synthetic plan identity. See DefaultPriceOverrideSvc.java:121-134.
  Line Numbers: 121 to 134
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/override/DefaultPriceOverrideSvc.java
  Source Code:
  ```java
          final String planName;
          if (context != null) {
              final CatalogOverridePlanDefinitionModelDao overriddenPlan = overrideDao.getOrCreateOverridePlanDefinition(parentPlan, catalogEffectiveDate, resolvedOverride, context);
              planName = new StringBuffer(parentPlan.getName()).append("-").append(overriddenPlan.getRecordId()).toString();
          } else {
              planName = new StringBuffer(parentPlan.getName()).append("-dryrun-").append(DRY_RUN_PLAN_IDX.incrementAndGet()).toString();
          }

          final DefaultPlan result = new DefaultPlan(planName, (DefaultPlan) parentPlan, resolvedOverride);
          result.initialize(standaloneCatalog);
          if (context == null) {
              overriddenPlanCache.addDryRunPlan(planName, result);
          }
          return result;
  ```
  Applies to: per-tenant price overrides.
  Confidence score: High

- [Kill-BR-0046] - Simple-plan-descriptor trial/recurring compatibility rule
  Description: CatalogUpdater.validateExistingPlan() only allows the "simple plan" API to attach to an existing plan if that plan has no trial phase or exactly one $0 TRIAL phase, whose duration must match the descriptor's trial length/unit exactly if trial info is supplied; it also requires the final phase to be EVERGREEN with a matching recurring billing period and (if given) matching price in the target currency, else CAT_FAILED_SIMPLE_PLAN_VALIDATION is thrown. See CatalogUpdater.java:227-283.
  Line Numbers: 227 to 283
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java
  Source Code:
  ```java
      private void validateExistingPlan(final DefaultPlan plan, final SimplePlanDescriptor desc) throws CatalogApiException {

          boolean failedValidation = false;

          //
          // TRIAL VALIDATION
          //
          // We only support adding new Plan with NO TRIAL or $0 TRIAL. Existing Plan not matching such criteria are incompatible
          if (plan.getInitialPhases().length > 1 ||
              (plan.getInitialPhases().length == 1 &&
               (plan.getInitialPhases()[0].getPhaseType() != PhaseType.TRIAL || !plan.getInitialPhases()[0].getFixed().getPrice().isZero()))) {
              failedValidation = true;

          } else if (desc.getTrialLength() != null && desc.getTrialTimeUnit() != null) { // If desc includes trial info we verify this is valid
              final boolean isDescConfiguredWithTrial = desc.getTrialLength() > 0 && desc.getTrialTimeUnit() != TimeUnit.UNLIMITED;
              final boolean isPlanConfiguredWithTrial = plan.getInitialPhases().length == 1;
              // Current plan has trial and desc does not or reverse
              if ((isDescConfiguredWithTrial && !isPlanConfiguredWithTrial) ||
                  (!isDescConfiguredWithTrial && isPlanConfiguredWithTrial)) {
                  failedValidation = true;
                  // Both have trials , check they match
              } else if (isDescConfiguredWithTrial && isPlanConfiguredWithTrial) {
                  if (plan.getInitialPhases()[0].getDuration().getUnit() != desc.getTrialTimeUnit() ||
                      plan.getInitialPhases()[0].getDuration().getNumber() != desc.getTrialLength()) {
                      failedValidation = true;
                  }
              }
          }

          //
          // RECURRING VALIDATION
          //
          if (!failedValidation) {
              // Desc only supports EVERGREEN Phase
              if (plan.getFinalPhase().getPhaseType() != PhaseType.EVERGREEN) {
                  failedValidation = true;
              } else {

                  // Should be same recurring BillingPeriod
                  if (desc.getBillingPeriod() != null && plan.getFinalPhase().getRecurring().getBillingPeriod() != desc.getBillingPeriod()) {
                      failedValidation = true;
                  } else if (desc.getCurrency() != null && desc.getAmount() != null) {
                      try {
                          final BigDecimal currentAmount = plan.getFinalPhase().getRecurring().getRecurringPrice().getPrice(desc.getCurrency());
                          if (currentAmount.compareTo(desc.getAmount()) != 0) {
                              failedValidation = true;
                          }
                      } catch (CatalogApiException ignoreIfCurrencyIsCurrentlyUndefined) {
                      }
                  }
              }
          }

          if (failedValidation) {
              throw new CatalogApiException(ErrorCode.CAT_FAILED_SIMPLE_PLAN_VALIDATION, plan.toString(), desc.toString());
          }
      }
  ```
  Applies to: simple/programmatic plan creation API (e.g. plugin-driven catalog construction).
  Confidence score: High

- [Kill-BR-0047] - Add-on simple-plan creation prerequisite
  Description: CatalogUpdater.validateNewPlanDescriptor() requires that a new ADD_ON-category simple plan declare at least one existing base product it's available for, and rejects any listed base product that doesn't already exist in the catalog. See CatalogUpdater.java:296-314.
  Line Numbers: 296 to 314
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java
  Source Code:
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
      }
  ```
  Applies to: simple/programmatic add-on plan creation.
  Confidence score: High

- [Kill-BR-0048] - Hardcoded default rule set for programmatically-built ("simple") catalogs
  Description: CatalogUpdater.getSaneDefaultPlanRules() bootstraps a brand-new simple catalog with IMMEDIATE billing action policy for both plan change and cancellation, and START_OF_BUNDLE alignment for both change and create — encoding a specific default business policy (as opposed to catalog-author-chosen) for catalogs built via the simple-plan API rather than XML. See CatalogUpdater.java:332-365.
  Line Numbers: 332 to 365
  Source Code File Name: catalog/src/main/java/org/killbill/billing/catalog/CatalogUpdater.java
  Source Code:
  ```java
      private DefaultPlanRules getSaneDefaultPlanRules(final DefaultPriceList defaultPriceList) {

          final DefaultCaseChangePlanPolicy[] changePolicy = new DefaultCaseChangePlanPolicy[1];
          changePolicy[0] = new DefaultCaseChangePlanPolicy();
          changePolicy[0].setPolicy(BillingActionPolicy.IMMEDIATE);

          final DefaultCaseChangePlanAlignment[] changeAlignment = new DefaultCaseChangePlanAlignment[1];
          changeAlignment[0] = new DefaultCaseChangePlanAlignment();
          changeAlignment[0].setAlignment(PlanAlignmentChange.START_OF_BUNDLE);

          final DefaultCaseCancelPolicy[] cancelPolicy = new DefaultCaseCancelPolicy[1];
          cancelPolicy[0] = new DefaultCaseCancelPolicy();
          cancelPolicy[0].setPolicy(BillingActionPolicy.IMMEDIATE);

          final DefaultCaseCreateAlignment[] createAlignment = new DefaultCaseCreateAlignment[1];
          createAlignment[0] = new DefaultCaseCreateAlignment();
          createAlignment[0].setAlignment(PlanAlignmentCreate.START_OF_BUNDLE);

          final DefaultCaseBillingAlignment[] billingAlignmentCase = new DefaultCaseBillingAlignment[1];
          billingAlignmentCase[0] = new DefaultCaseBillingAlignment();
          billingAlignmentCase[0].setAlignment(BillingAlignment.ACCOUNT);

          final DefaultCasePriceList[] priceList = new DefaultCasePriceList[1];
          priceList[0] = new DefaultCasePriceList();
          priceList[0].setToPriceList(defaultPriceList);

          return new DefaultPlanRules()
                  .setChangeCase(changePolicy)
                  .setChangeAlignmentCase(changeAlignment)
                  .setCancelCase(cancelPolicy)
                  .setCreateAlignmentCase(createAlignment)
                  .setBillingAlignmentCase(billingAlignmentCase)
                  .setPriceListCase(priceList);
      }
  ```
  Applies to: simple/programmatic catalog bootstrap (used by e.g. some plugin-driven catalogs).
  Confidence score: High

- [Kill-BR-0049] - Cancellation effective-date policy resolution (IMMEDIATE / START_OF_TERM / END_OF_TERM)
  Description: Given a BillingActionPolicy, computes the concrete effective cancellation date. IMMEDIATE uses "now"; START_OF_TERM walks the charged-through-date backward by whole billing periods until it is not after "now" (or uses subscription start date if no CTD yet), then re-aligns to the BCD via BillCycleDayCalculator.alignProposedBillCycleDate; END_OF_TERM uses the charged-through-date if it's in the future, else "now". As a final sanity check, the result is clamped forward to the start of the current phase so a subscription can never be cancelled before its current phase began. Evidence: org.killbill.billing.subscription.api.user.DefaultSubscriptionBase.getEffectiveDateForPolicy, lines 811-895.
  Line Numbers: 811 to 895
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
      public DateTime getEffectiveDateForPolicy(final BillingActionPolicy policy, @Nullable final BillingAlignment alignment, final InternalTenantContext context) throws SubscriptionBaseApiException {

          final Integer accountBillCycleDayLocal = apiService != null ? apiService.getAccountBCD(context) : null;
          final DateTime candidateResult;
          switch (policy) {
              case IMMEDIATE:
                  candidateResult = clock.getUTCNow();
                  break;

              case START_OF_TERM:
                  if (chargedThroughDate == null) {
                      candidateResult = getStartDate();
                      // Will take care of billing IN_ARREAR or subscriptions that are not invoiced up to date
                  } else if (!chargedThroughDate.isAfter(clock.getUTCNow())) {
                      candidateResult = chargedThroughDate;
                  } else {

                      // In certain path (dryRun, or default catalog START_OF_TERM policy), the info is not easily available and as a result, such policy is not implemented
                      Preconditions.checkState(alignment != null && context != null && accountBillCycleDayLocal != null, "START_OF_TERM not implemented in dryRun use case");

                      Preconditions.checkState(alignment != BillingAlignment.BUNDLE || category != ProductCategory.ADD_ON, "START_OF_TERM not implemented for AO configured with a BUNDLE billing alignment");

                      // If BCD was overriden at the subscription level, we take its latest value (it should also be reflected in the chargedThroughDate) but still required for
                      // alignment purpose
                      Integer bcd = getBillCycleDayLocal();
                      if (bcd == null) {
                          bcd = BillCycleDayCalculator.calculateBcdForAlignment(null, this, this, alignment, context, accountBillCycleDayLocal);
                      }

                      final BillingPeriod billingPeriod = getLastActivePlan().getRecurringBillingPeriod();
                      DateTime proposedDate = chargedThroughDate;
                      if (billingPeriod.getPeriod().equals(Period.ZERO)) {
                          proposedDate = clock.getUTCNow();
                      } else {
                          while (proposedDate.isAfter(clock.getUTCNow())) {
                              proposedDate = proposedDate.minus(billingPeriod.getPeriod());
                          }
                      }

                      final LocalDate resultingLocalDate = BillCycleDayCalculator.alignProposedBillCycleDate(proposedDate, bcd, billingPeriod, context);
                      candidateResult = context.toUTCDateTime(resultingLocalDate);
                  }

                  break;

              case END_OF_TERM:
                  //
                  // If we have a chargedThroughDate that is 'up to date' we use it, if not default to now
                  // chargedThroughDate could exist and be less than now if:
                  // 1. account is not being invoiced, for e.g AUTO_INVOICING_OFF nis set
                  // 2. In the case if FIXED item CTD is set using startDate of the service period
                  //
                  candidateResult = (chargedThroughDate != null && chargedThroughDate.isAfter(clock.getUTCNow())) ? chargedThroughDate : clock.getUTCNow();
                  break;
              default:
                  throw new SubscriptionBaseError(String.format(
                          "Unexpected policy type %s", policy.toString()));
          }
          // Finally we verify we won't cancel prior the beginning of our current PHASE  -- mostly as a sanity or for test stability
          final DateTime lastTransitionTime = getCurrentPhaseStart();
          return (candidateResult.compareTo(lastTransitionTime) < 0) ? lastTransitionTime : candidateResult;
      }

      public DateTime getCurrentPhaseStart() {

          if (transitions == null) {
              throw new SubscriptionBaseError(String.format(
                      "No transitions for subscription %s", getId()));
          }
          final SubscriptionBaseTransitionDataIterator it = new SubscriptionBaseTransitionDataIterator(
                  clock, transitions, Order.DESC_FROM_FUTURE,
                  Visibility.ALL, TimeLimit.PAST_OR_PRESENT_ONLY);
          while (it.hasNext()) {
              final SubscriptionBaseTransitionData cur = (SubscriptionBaseTransitionData) it.next();

              if (cur.getTransitionType() == SubscriptionBaseTransitionType.PHASE
                  || cur.getTransitionType() == SubscriptionBaseTransitionType.TRANSFER
                  || cur.getTransitionType() == SubscriptionBaseTransitionType.CREATE
                  || cur.getTransitionType() == SubscriptionBaseTransitionType.CHANGE) {
                  return cur.getEffectiveTransitionTime();
              }
          }
          // If the subscription is not yet started we return the startDate
          return getStartDate();
      }
  ```
  Applies to: subscription cancellation (also reused by dryRunChangePlan for change-plan effective date)
  Confidence score: High

- [Kill-BR-0050] - Cancel is blocked on terminal/conflicting states
  Description: A cancel request throws SUB_CANCEL_BAD_STATE if the subscription is already CANCELLED or EXPIRED; it also throws SUB_CANCEL_BAD_STATE ("PENDING CANCELLED"/"PENDING EXPIRY") if a future CANCEL or EXPIRED transition is already pending at a date on/after the requested effective date — the caller must uncancel first. A pending future cancellation at an earlier-or-equal date than the new request is instead silently superseded. Evidence: DefaultSubscriptionBaseApiService.doCancelPlan, lines 264-316.
  Line Numbers: 264 to 316
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
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
                  cancelEvents.addAll(getEventsOnCancelPlan(subscription, effectiveDate, false, catalog, internalCallContext));

                  if (subscription.getCategory() == ProductCategory.BASE) {
                      subscriptionsToBeCancelled.addAll(computeAddOnsToCancel(cancelEvents, null, subscription.getBundleId(), effectiveDate, catalog, internalCallContext));
                  }
              }

              dao.cancelSubscriptions(subscriptionsToBeCancelled, cancelEvents, catalog, internalCallContext);

              boolean allSubscriptionsCancelled = true;
              for (final DefaultSubscriptionBase subscription : subscriptions.keySet()) {
                  subscription.rebuildTransitions(dao.getEventsForSubscription(subscription.getId(), false, internalCallContext), catalog); //false
                  allSubscriptionsCancelled = allSubscriptionsCancelled && (subscription.getState() == EntitlementState.CANCELLED);
              }

              return allSubscriptionsCancelled;
          } catch (final CatalogApiException e) {
              throw new SubscriptionBaseApiException(e);
          }
      }
  ```
  Applies to: subscription cancel (cancel/cancelWithDate/cancelWithPolicy paths)
  Confidence score: High

- [Kill-BR-0051] - Base-plan cancellation/expiry cascades to add-ons
  Description: Cancelling (or FIXEDTERM-expiring) a BASE subscription automatically cancels/expires every non-cancelled, non-expired ADD_ON subscription in the same bundle whose plan becomes "included" in the new/none base product or is no longer "available" for it. Cascading is skipped entirely when the triggering cancel/change is future-dated (only applied once it actually becomes effective). Each cascaded add-on's own cancellation date is clamped to not precede that add-on's alignStartDate. Evidence: DefaultSubscriptionBaseApiService.computeAddOnsToCancel/addCancellationAddOnForEventsIfRequired lines 773-821; computeAddOnsToExpire/handleExpiredEvent lines 736-771.
  Line Numbers: 773 to 821
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
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
  Line Numbers: 736 to 771
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
  ```java
      public int handleExpiredEvent(final DefaultSubscriptionBase subscription, final SubscriptionBaseEvent event, final SubscriptionCatalog catalog, final CallContext context) throws CatalogApiException {

          final InternalCallContext internalCallContext = createCallContextFromBundleId(subscription.getBundleId(), context);
          if (subscription.getCategory() == ProductCategory.BASE) {
              final List<SubscriptionBaseEvent> expireEvents = new LinkedList<SubscriptionBaseEvent>();
              final List<DefaultSubscriptionBase> subscriptionsToBeExpired = computeAddOnsToExpire(expireEvents, subscription.getBundleId(), event.getEffectiveDate(), catalog, internalCallContext);
              dao.cancelOrExpireSubscriptionOnNotification(subscription, event, subscriptionsToBeExpired, expireEvents, catalog, internalCallContext);
              return subscriptionsToBeExpired.size();
          } else { //ADD_ON and STANDALONE products
              final List<SubscriptionBaseEvent> expireEvents = new LinkedList<SubscriptionBaseEvent>();
              final List<DefaultSubscriptionBase> subscriptionsToBeExpired = new LinkedList<DefaultSubscriptionBase>();
              dao.cancelOrExpireSubscriptionOnNotification(subscription, event, subscriptionsToBeExpired, expireEvents, catalog, internalCallContext);
              return 1;
          }
      }

      private List<DefaultSubscriptionBase> computeAddOnsToExpire(final Collection<SubscriptionBaseEvent> expireEvents, final UUID bundleId, final DateTime effectiveDate, final SubscriptionCatalog catalog, final InternalCallContext internalCallContext) throws CatalogApiException {
          final List<DefaultSubscriptionBase> subscriptionsToBeExpired = new ArrayList<>();
          final List<DefaultSubscriptionBase> subscriptions = dao.getSubscriptions(bundleId, Collections.emptyList(), catalog, internalCallContext);
          for (final SubscriptionBase subscription : subscriptions) {
              final DefaultSubscriptionBase cur = (DefaultSubscriptionBase) subscription;
              if (cur.getCategory() != ProductCategory.ADD_ON || cur.getState() == EntitlementState.CANCELLED || cur.getState() == EntitlementState.EXPIRED) {
                  continue;
              }

              final SubscriptionBaseEvent expiredEvent = new ExpiredEventData(new ExpiredEventBuilder()
                                                                                      .setSubscriptionId(cur.getId())
                                                                                      .setEffectiveDate(effectiveDate)
                                                                                      .setActive(true));

              expireEvents.add(expiredEvent);
              subscriptionsToBeExpired.add(cur);
          }
          return subscriptionsToBeExpired;

      }
  ```
  Applies to: BASE subscription cancel/change flow, ADD_ON lifecycle
  Confidence score: High

- [Kill-BR-0052] - Uncancel only valid on a future-cancelled subscription
  Description: uncancel() throws SUB_UNCANCEL_BAD_STATE unless subscription.isFutureCancelled() is true (i.e. there is a pending future CANCEL transition); it then emits an ApiEventUncancel and recomputes the next phase transition (using the subscription start date instead of "now" as the alignment anchor when the subscription is still PENDING, to avoid the aligner picking CREATE instead of the correct next PHASE). Evidence: DefaultSubscriptionBaseApiService.uncancel, lines 318-356.
  Line Numbers: 318 to 356
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
  ```java
      @Override
      public boolean uncancel(final DefaultSubscriptionBase subscription, final CallContext context) throws SubscriptionBaseApiException {
          if (!subscription.isFutureCancelled()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_UNCANCEL_BAD_STATE, subscription.getId().toString());
          }
          try {

              final InternalCallContext internalCallContext = createCallContextFromBundleId(subscription.getBundleId(), context);
              final SubscriptionCatalog catalog = subscriptionCatalogApi.getFullCatalog(internalCallContext);
              final SubscriptionBaseEvent uncancelEvent = new ApiEventUncancel(new ApiEventBuilder()
                                                                                       .setSubscriptionId(subscription.getId())
                                                                                       .setEffectiveDate(context.getCreatedDate())
                                                                                       .setFromDisk(true));

              final List<SubscriptionBaseEvent> uncancelEvents = new ArrayList<SubscriptionBaseEvent>();
              uncancelEvents.add(uncancelEvent);

              //
              // Used to compute effective for next phase (which was set unactive during cancellation).
              // In case of a pending subscription we don't want to pass an effective date prior the CREATE event as we would end up with the wrong
              // transition in PlanAligner (next transition would be CREATE instead of potential next PHASE)
              //
              final DateTime planAlignerEffectiveDate = subscription.getState() == EntitlementState.PENDING ? subscription.getStartDate() : context.getCreatedDate();

              final TimedPhase nextTimedPhase = planAligner.getNextTimedPhase(subscription, planAlignerEffectiveDate, catalog, internalCallContext);
              final PhaseEvent nextPhaseEvent = (nextTimedPhase != null) ?
                                                PhaseEventData.createNextPhaseEvent(subscription.getId(), nextTimedPhase.getPhase().getName(), nextTimedPhase.getStartPhase()) :
                                                null;
              if (nextPhaseEvent != null) {
                  uncancelEvents.add(nextPhaseEvent);
              }

              dao.uncancelSubscription(subscription, uncancelEvents, internalCallContext);
              subscription.rebuildTransitions(dao.getEventsForSubscription(subscription.getId(), false, internalCallContext), catalog);
              return true;
          } catch (final CatalogApiException e) {
              throw new SubscriptionBaseApiException(e);
          }
      }
  ```
  Applies to: undo-cancel ("uncancel") flow
  Confidence score: High

- [Kill-BR-0053] - Undo-change-plan only valid with a pending plan change
  Description: undoChangePlan() throws SUB_UNDO_CHANGE_BAD_STATE unless subscription.isPendingChangePlan() (a future CHANGE transition exists); otherwise it emits ApiEventUndoChange and recomputes the resulting next-phase event the same way as uncancel. Evidence: DefaultSubscriptionBaseApiService.undoChangePlan lines 663-701; isPendingChangePlan in DefaultSubscriptionBase lines 796-809.
  Line Numbers: 663 to 701
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
  ```java
      @Override
      public boolean undoChangePlan(final DefaultSubscriptionBase subscription, final CallContext context) throws SubscriptionBaseApiException {
          if (!subscription.isPendingChangePlan()) {
              throw new SubscriptionBaseApiException(ErrorCode.SUB_UNDO_CHANGE_BAD_STATE, subscription.getId().toString());
          }
          try {

              final InternalCallContext internalCallContext = createCallContextFromBundleId(subscription.getBundleId(), context);
              final SubscriptionCatalog catalog = subscriptionCatalogApi.getFullCatalog(internalCallContext);
              final SubscriptionBaseEvent undoChangePlanEvent = new ApiEventUndoChange(new ApiEventBuilder()
                                                                                               .setSubscriptionId(subscription.getId())
                                                                                               .setEffectiveDate(context.getCreatedDate())
                                                                                               .setFromDisk(true));

              final List<SubscriptionBaseEvent> undoChangePlanEvents = new ArrayList<SubscriptionBaseEvent>();
              undoChangePlanEvents.add(undoChangePlanEvent);

              //
              // Used to compute effective for next phase (which was set unactive during cancellation).
              // In case of a pending subscription we don't want to pass an effective date prior the CREATE event as we would end up with the wrong
              // transition in PlanAligner (next transition would be CREATE instead of potential next PHASE)
              //
              final DateTime planAlignerEffectiveDate = subscription.getState() == EntitlementState.PENDING ? subscription.getStartDate() : context.getCreatedDate();

              final TimedPhase nextTimedPhase = planAligner.getNextTimedPhase(subscription, planAlignerEffectiveDate, catalog, internalCallContext);
              final PhaseEvent nextPhaseEvent = (nextTimedPhase != null) ?
                                                PhaseEventData.createNextPhaseEvent(subscription.getId(), nextTimedPhase.getPhase().getName(), nextTimedPhase.getStartPhase()) :
                                                null;
              if (nextPhaseEvent != null) {
                  undoChangePlanEvents.add(nextPhaseEvent);
              }

              dao.undoChangePlan(subscription, undoChangePlanEvents, internalCallContext);
              subscription.rebuildTransitions(dao.getEventsForSubscription(subscription.getId(), false, internalCallContext), catalog);
              return true;
          } catch (final CatalogApiException e) {
              throw new SubscriptionBaseApiException(e);
          }
      }
  ```
  Line Numbers: 796 to 809
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
      public boolean isPendingChangePlan() {

          final SubscriptionBaseTransitionDataIterator it = new SubscriptionBaseTransitionDataIterator(
                  clock, transitions, Order.ASC_FROM_PAST,
                  Visibility.ALL, TimeLimit.FUTURE_ONLY);

          while (it.hasNext()) {
              final SubscriptionBaseTransition next = it.next();
              if (next.getTransitionType() == SubscriptionBaseTransitionType.CHANGE) {
                  return true;
              }
          }
          return false;
      }
  ```
  Applies to: subscription change-plan undo flow
  Confidence score: High

- [Kill-BR-0054] - Change-plan state/date validation
  Description: A plan change is rejected (SUB_CHANGE_NON_ACTIVE) if the subscription is CANCELLED/EXPIRED, or if an explicit effective date is before the subscription start date; rejected (SUB_CHANGE_FUTURE_CANCELLED) if the subscription is already future-cancelled; rejected (SUB_CHANGE_FUTURE_EXPIRED) if a future FIXEDTERM expiry exists before the requested effective date. Separately, any requested effective date must not precede the last recorded transition (or subscription start), else SUB_INVALID_REQUESTED_DATE. Evidence: validateSubscriptionStateForChangePlan lines 835-850, validateEffectiveDate lines 823-833.
  Line Numbers: 835 to 850
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
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
  Line Numbers: 823 to 833
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
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
  Applies to: changePlan / changePlanWithDate / changePlanWithPolicy
  Confidence score: High

- [Kill-BR-0055] - Add-on plan-category and product eligibility on change
  Description: When changing a plan, the new plan's product category must equal the subscription's current category, else SUB_CHANGE_INVALID (an add-on can't become a base plan or vice versa via changePlan). If the new plan is itself an ADD_ON with a positive plansAllowedInBundle cap, the change is rejected (SUB_CHANGE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE) when the bundle already holds that many add-ons of the same plan name. Evidence: DefaultSubscriptionBaseApiService.doChangePlan, lines 473-486.
  Line Numbers: 473 to 486
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
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
  Applies to: changePlan for add-on subscriptions
  Confidence score: High

- [Kill-BR-0056] - Add-on creation eligibility (checkAddonCreationRights)
  Description: An add-on cannot be created (SUB_CREATE_AO_BP_NON_ACTIVE) if the base subscription is CANCELLED, or PENDING with a start date after the requested add-on date; cannot be created (SUB_CREATE_AO_ALREADY_INCLUDED) if the target add-on product is already included free in the base product; cannot be created (SUB_CREATE_AO_NOT_AVAILABLE) if the target add-on product isn't listed in the base product's "available" add-ons. Evidence: AddonUtils.checkAddonCreationRights, lines 36-54, called from SubscriptionApiBase.getBundleStartDateWithSanity line 244.
  Line Numbers: 36 to 54
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/engine/addon/AddonUtils.java
  Source Code:
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
  ```
  Applies to: add-on subscription creation
  Confidence score: High

- [Kill-BR-0057] - One base/standalone plan per bundle; add-on requires an active base
  Description: Creating a BASE plan when the bundle already has an ACTIVE or PENDING base subscription throws SUB_CREATE_BP_EXISTS; creating an ADD_ON with no base subscription in the bundle throws SUB_CREATE_NO_BP; creating an ADD_ON dated before the base subscription's start date throws SUB_INVALID_REQUESTED_DATE; creating a STANDALONE plan when a base already exists also throws SUB_CREATE_BP_EXISTS. An add-on's effective bundle start date is forced to the base subscription's start date, not the requested date. Evidence: SubscriptionApiBase.getBundleStartDateWithSanity, lines 223-258.
  Line Numbers: 223 to 258
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java
  Source Code:
  ```java
      protected DateTime getBundleStartDateWithSanity(final UUID bundleId,
                                                      @Nullable final SubscriptionBase baseSubscription,
                                                      final Plan plan,
                                                      final DateTime effectiveDate,
                                                      final AddonUtils addonUtils,
                                                      final InternalTenantContext context) throws SubscriptionBaseApiException, CatalogApiException {
          switch (plan.getProduct().getCategory()) {
              case BASE:
                  if (baseSubscription != null &&
                      (baseSubscription.getState() == EntitlementState.ACTIVE || baseSubscription.getState() == EntitlementState.PENDING)) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_BP_EXISTS, bundleId);
                  }
                  return effectiveDate;

              case ADD_ON:
                  if (baseSubscription == null) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_NO_BP, bundleId);
                  }
                  if (effectiveDate.isBefore(baseSubscription.getStartDate())) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, effectiveDate.toString(), baseSubscription.getStartDate().toString());
                  }
                  addonUtils.checkAddonCreationRights(baseSubscription, plan, effectiveDate, context);
                  return baseSubscription.getStartDate();

              case STANDALONE:
                  if (baseSubscription != null) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_BP_EXISTS, bundleId);
                  }
                  // Not really but we don't care, there is no alignment for STANDALONE subscriptions
                  return effectiveDate;

              default:
                  throw new SubscriptionBaseError(String.format("Can't create subscription of type %s",
                                                                plan.getProduct().getCategory().toString()));
          }
      }
  ```
  Applies to: subscription/bundle creation (BASE, ADD_ON, STANDALONE)
  Confidence score: High

- [Kill-BR-0058] - Per-bundle add-on quantity cap at creation time
  Description: When creating add-ons, for any plan with a finite plansAllowedInBundle (not -1/0), the sum of already-existing add-ons of that plan name in the bundle plus add-ons of that plan being created in the same batch cannot exceed the cap, else SUB_CREATE_AO_MAX_PLAN_ALLOWED_BY_BUNDLE. Evidence: DefaultSubscriptionBaseCreateApi, lines 234-246, using AddonUtils.countExistingAddOnsWithSamePlanName.
  Line Numbers: 234 to 246
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/svcs/DefaultSubscriptionBaseCreateApi.java
  Source Code:
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
  Applies to: bulk add-on creation (createPlansWithAddOns path)
  Confidence score: High

- [Kill-BR-0059] - Plan-phase alignment on create/change (trial -> discount -> evergreen sequencing)
  Description: Determines the anchor date a plan's phase sequence is computed from — START_OF_SUBSCRIPTION, START_OF_BUNDLE, or (on change) CHANGE_OF_PLAN using the last/current change effective date — per the catalog's alignment policy. From that anchor, getPhaseAlignments walks each PlanPhase in order (trial, discount, evergreen, etc.), accumulating each phase's start via Duration addition, until it locates the phase active "now" (CURRENT) and the one that starts next (NEXT); an EVERGREEN phase is treated as unbounded (no further phase after it). Evidence: PlanAligner.getTimedPhaseOnCreate/getTimedPhaseOnChange/getPhaseAlignments/getTimedPhase, lines 184-346.
  Line Numbers: 184 to 342
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/alignment/PlanAligner.java
  Source Code:
  ```java
      private List<TimedPhase> getTimedPhaseOnCreate(final DateTime subscriptionStartDate,
                                                     final DateTime bundleStartDate,
                                                     final Plan plan,
                                                     @Nullable final PhaseType initialPhase,
                                                     final SubscriptionCatalog catalog,
                                                     final DateTime catalogEffectiveDate,
                                                     final InternalTenantContext context)
              throws CatalogApiException, SubscriptionBaseApiException {
          final PlanSpecifier planSpecifier = new PlanSpecifier(plan.getName());

          final DateTime planStartDate;
          final PlanAlignmentCreate alignment = catalog.planCreateAlignment(planSpecifier, catalogEffectiveDate, subscriptionStartDate);
          switch (alignment) {
              case START_OF_SUBSCRIPTION:
                  planStartDate = subscriptionStartDate;
                  break;
              case START_OF_BUNDLE:
                  planStartDate = bundleStartDate;
                  break;
              default:
                  throw new SubscriptionBaseError(String.format("Unknown PlanAlignmentCreate %s", alignment));
          }

          return getPhaseAlignments(plan, initialPhase, planStartDate, context);
      }

      private TimedPhase getTimedPhaseOnChange(final DefaultSubscriptionBase subscription,
                                               final Plan nextPlan,
                                               final DateTime effectiveDate,
                                               final PhaseType newPlanInitialPhaseType,
                                               final WhichPhase which,
                                               final SubscriptionCatalog catalog,
                                               final InternalTenantContext context) throws CatalogApiException, SubscriptionBaseApiException {
          final SubscriptionBaseTransition pendingOrLastPlanTransition;
          if (subscription.getState() == EntitlementState.PENDING) {
              pendingOrLastPlanTransition = subscription.getPendingTransition();
          } else {
              pendingOrLastPlanTransition = subscription.getLastTransitionForCurrentPlan();
          }
          return getTimedPhaseOnChange(subscription.getAlignStartDate(),
                                       subscription.getBundleStartDate(),
                                       pendingOrLastPlanTransition.getNextPhase(),
                                       pendingOrLastPlanTransition.getNextPlan(),
                                       nextPlan,
                                       effectiveDate,
                                       effectiveDate,
                                       // This method is only called while doing the change, hence we want to pass the change effective date
                                       effectiveDate,
                                       subscription.getAllTransitions(false).get(0).getNextPhase().getPhaseType(),
                                       newPlanInitialPhaseType,
                                       which,
                                       catalog,
                                       context);
      }

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

          return result;
      }

      // STEPH check for non evergreen Plans and what happens
      private TimedPhase getTimedPhase(final List<TimedPhase> timedPhases, final DateTime effectiveDate, final WhichPhase which) {
          TimedPhase cur = null;
          TimedPhase next = null;
          for (final TimedPhase phase : timedPhases) {
              if (phase.getStartPhase().isAfter(effectiveDate)) {
                  next = phase;
                  break;
              }
              cur = phase;
          }

          switch (which) {
              case CURRENT:
                  return cur;
              case NEXT:
                  return next;
              default:
                  throw new SubscriptionBaseError(String.format("Unexpected %s TimedPhase", which));
          }
      }
  ```
  Applies to: subscription creation and plan-change phase transition scheduling
  Confidence score: High

- [Kill-BR-0060] - FIXEDTERM phase auto-expiry scheduling
  Description: If the phase a subscription starts on (at creation or after a plan change) is FIXEDTERM, an ExpiredEvent is scheduled at phase-start-date + phase duration, independent of any explicit cancellation — the subscription will auto-transition to EXPIRED at that computed date. Evidence: DefaultSubscriptionBaseApiService.getEventsOnCreation lines 554-563, getEventsOnChangePlan lines 615-622.
  Line Numbers: 554 to 563
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
  ```java
          if (nextPhaseEvent != null) {
              events.add(nextPhaseEvent);
          } else if (currentPhase.getPhase().getPhaseType() == PhaseType.FIXEDTERM) {
              final DateTime fixedTermExpiryDate = currentPhase.getPhase().getDuration().addToDateTime(effectiveDate);
              final SubscriptionBaseEvent expiredEvent = new ExpiredEventData(new ExpiredEventBuilder()
                                                                                      .setSubscriptionId(subscriptionId)
                                                                                      .setEffectiveDate(fixedTermExpiryDate)
                                                                                      .setActive(true));
              events.add(expiredEvent);
          }
  ```
  Line Numbers: 615 to 622
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBaseApiService.java
  Source Code:
  ```java
          if (currentTimedPhase.getPhase().getPhaseType() == PhaseType.FIXEDTERM) {
              final DateTime fixedTermExpiryDate = currentTimedPhase.getPhase().getDuration().addToDateTime(currentTimedPhase.getStartPhase());
              final SubscriptionBaseEvent expiredEvent = new ExpiredEventData(new ExpiredEventBuilder()
                                                                                      .setSubscriptionId(subscription.getId())
                                                                                      .setEffectiveDate(fixedTermExpiryDate)
                                                                                      .setActive(true));
              changeEvents.add(expiredEvent);
          }
  ```
  Applies to: subscription creation and change-plan event generation
  Confidence score: High

- [Kill-BR-0061] - Bill-cycle-day (BCD) resolution by billing alignment
  Description: A subscription's BCD is derived from the catalog's BillingAlignment: ACCOUNT alignment uses the account-level BCD (falling back to SUBSCRIPTION alignment if the account BCD isn't set yet, i.e. == 0); BUNDLE alignment derives BCD from the base subscription's date of first non-zero recurring charge; SUBSCRIPTION alignment derives it from the subscription's own first-charge date (day-of-month). This resolved BCD then drives both new BCD events and re-alignment of billing dates on phase/plan transitions. Evidence: util/BillCycleDayCalculator.resolveEffectiveBillingAlignment/calculateBcdForAlignment, invoked from subscription/DefaultSubscriptionBase.alignToNextBCDIfRequired lines 741-767 and getEffectiveDateForPolicy lines 835-838.
  Line Numbers: 50 to 72
  Source Code File Name: util/src/main/java/org/killbill/billing/util/bcd/BillCycleDayCalculator.java
  Source Code:
  ```java
      public static BillingAlignment resolveEffectiveBillingAlignment(final BillingAlignment alignment, final int accountBillCycleDayLocal) {
          if (alignment == BillingAlignment.ACCOUNT && accountBillCycleDayLocal == 0) {
              return BillingAlignment.SUBSCRIPTION;
          }
          return alignment;
      }

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
  Line Numbers: 741 to 767
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
      private DateTime alignToNextBCDIfRequired(final Plan curPlan, final PlanPhase curPlanPhase, final DateTime prevTransitionDate, final DateTime curTransitionDate, final SubscriptionCatalog catalog, final Integer bcdLocal, final InternalTenantContext context) throws SubscriptionBaseApiException, CatalogApiException {

          if (!apiService.isEffectiveDateForExistingSubscriptionsAlignedToBCD(context)) {
              return curTransitionDate;
          }

          // If there is no recurring phase, we can't align to any BCD and we don't create a billing transition for such phase
          // TODO It does not take into consideration possible usage sections that could be present
          if (curPlanPhase.getRecurring() == null) {
              return null;
          }

          final BillingAlignment rawBillingAlignment = catalog.billingAlignment(new PlanPhaseSpecifier(curPlan.getName(), curPlanPhase.getPhaseType()),
                                                                                curTransitionDate, prevTransitionDate);
          final int accountBillCycleDayLocal = apiService.getAccountBCD(context);
          final BillingAlignment billingAlignment = BillCycleDayCalculator.resolveEffectiveBillingAlignment(rawBillingAlignment, accountBillCycleDayLocal);
          Integer bcd = bcdLocal;
          if (bcd == null) {
              // TODO If we have an add-on subscription with a BUNDLE alignment, this is incorrect as we need access to the base subscription
              bcd = BillCycleDayCalculator.calculateBcdForAlignment(null, this, this, billingAlignment, context, accountBillCycleDayLocal);
          }

          final BillingPeriod billingPeriod = curPlanPhase.getRecurring() != null ? curPlanPhase.getRecurring().getBillingPeriod() : BillingPeriod.NO_BILLING_PERIOD;
          final LocalDate resultingLocalDate = BillCycleDayCalculator.alignToNextBillCycleDate(prevTransitionDate, curTransitionDate, bcd, billingPeriod, context);
          final DateTime candidateResult = context.toUTCDateTime(resultingLocalDate);
          return candidateResult;
      }
  ```
  Line Numbers: 835 to 838
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
                      Integer bcd = getBillCycleDayLocal();
                      if (bcd == null) {
                          bcd = BillCycleDayCalculator.calculateBcdForAlignment(null, this, this, alignment, context, accountBillCycleDayLocal);
                      }
  ```
  Applies to: subscription creation, plan change, catalog-version alignment
  Confidence score: High

- [Kill-BR-0062] - Catalog-version upgrade alignment to next BCD boundary
  Description: When a newer catalog version defines an effectiveDateForExistingSubscriptions for the subscription's current plan, a synthetic CHANGE billing event is generated at that date; if the tenant config isEffectiveDateForExistingSubscriptionsAlignedToBCD is enabled, that raw catalog date is pushed forward to the subscription's next bill-cycle boundary (via BillCycleDayCalculator.alignToNextBillCycleDate) rather than applied verbatim, and no event is generated at all if the phase has no recurring section. Evidence: DefaultSubscriptionBase.getSubscriptionBillingEvents lines 682-738 and alignToNextBCDIfRequired lines 740-767.
  Line Numbers: 682 to 738
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
                      boolean isPlanOverridden = false;
                      if (plan != null) {
                          lastActiveCatalog = plan.getCatalog();
                          isPlanOverridden = priceOverrideSvcStatus.isOverriddenPlan(plan.getName());
                      }
                      // Computed from lastActiveCatalog
                      final DateTime catalogEffectiveDate = isPlanOverridden ? cur.getEffectiveTransitionTime() : CatalogDateHelper.toUTCDateTime(lastActiveCatalog.getEffectiveDate());
                      final SubscriptionBillingEvent billingTransition = new DefaultSubscriptionBillingEvent(cur.getTransitionType(), plan, planPhase, cur.getEffectiveTransitionTime(),
                                                                                                             cur.getTotalOrdering(), cur.getNextBillingCycleDayLocal(), cur.getNextQuantity(),
                                                                                                             catalogEffectiveDate);
                      result.add(billingTransition);

                      if (isCreateOrTransfer || isChangeEvent || isPhaseEvent) {

                          // If we are moving to a new Plan, we use the latest active catalog version at the time this operation took place.
                          final DateTime billingTransitionEffectiveDate = isPhaseEvent ? lastPlanTransition.getEffectiveTransitionTime() : billingTransition.getEffectiveDate();
                          final StaticCatalog catalogVersion = catalog.versionForDate(billingTransitionEffectiveDate);

                          final Plan currentPlan = catalogVersion.findPlan(billingTransition.getPlan().getName());
                          final Integer bcdLocal = cur.getNextBillingCycleDayLocal();
                          // Iterate through all more recent version of the catalog to find possible effectiveDateForExistingSubscriptions transition for this Plan
                          Plan nextPlan = catalog.getNextPlanVersion(currentPlan);

                          while (nextPlan != null) {
                              if (nextPlan.getEffectiveDateForExistingSubscriptions() != null) {

                                  DateTime nextEffectiveDate = new DateTime(nextPlan.getEffectiveDateForExistingSubscriptions()).toDateTime(DateTimeZone.UTC);

                                  nextEffectiveDate = alignToNextBCDIfRequired(plan, planPhase, cur.getEffectiveTransitionTime(), nextEffectiveDate, catalog, bcdLocal, context);
                                  // Add the catalog change transition if it is for a date past our current transition
                                  if (nextEffectiveDate != null && !nextEffectiveDate.isBefore(cur.getEffectiveTransitionTime())) {
                                      // Computed from the nextPlan
                                      final PlanPhase nextPlanPhase = nextPlan.findPhase(planPhase.getName());
                                      final DateTime catalogEffectiveDateForNextPlan = CatalogDateHelper.toUTCDateTime(nextPlan.getCatalog().getEffectiveDate());
                                      final SubscriptionBillingEvent newBillingTransition = new DefaultSubscriptionBillingEvent(SubscriptionBaseTransitionType.CHANGE, nextPlan, nextPlanPhase, nextEffectiveDate,
                                                                                                                                cur.getTotalOrdering(), bcdLocal, cur.getNextQuantity(), catalogEffectiveDateForNextPlan);
                                      candidatesCatalogChangeEvents.add(newBillingTransition);
                                  }

                              }
                              // TODO not so optimized as we keep parsing catalogs from the start...
                              nextPlan = catalog.getNextPlanVersion(nextPlan);
                          }
                      }
                  }
              }
              SubscriptionBillingEvent prevCandidateForCatalogChangeEvents = candidatesCatalogChangeEvents.poll();
              while (prevCandidateForCatalogChangeEvents != null) {
                  result.add(prevCandidateForCatalogChangeEvents);
                  prevCandidateForCatalogChangeEvents = candidatesCatalogChangeEvents.poll();
              }

              return result;
          } catch (final CatalogApiException e) {
              throw new SubscriptionBaseApiException(e);
          }
      }
  ```
  Line Numbers: 740 to 767
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
      // Align to next BCD based on recurring section for the phase
      private DateTime alignToNextBCDIfRequired(final Plan curPlan, final PlanPhase curPlanPhase, final DateTime prevTransitionDate, final DateTime curTransitionDate, final SubscriptionCatalog catalog, final Integer bcdLocal, final InternalTenantContext context) throws SubscriptionBaseApiException, CatalogApiException {

          if (!apiService.isEffectiveDateForExistingSubscriptionsAlignedToBCD(context)) {
              return curTransitionDate;
          }

          // If there is no recurring phase, we can't align to any BCD and we don't create a billing transition for such phase
          // TODO It does not take into consideration possible usage sections that could be present
          if (curPlanPhase.getRecurring() == null) {
              return null;
          }

          final BillingAlignment rawBillingAlignment = catalog.billingAlignment(new PlanPhaseSpecifier(curPlan.getName(), curPlanPhase.getPhaseType()),
                                                                                curTransitionDate, prevTransitionDate);
          final int accountBillCycleDayLocal = apiService.getAccountBCD(context);
          final BillingAlignment billingAlignment = BillCycleDayCalculator.resolveEffectiveBillingAlignment(rawBillingAlignment, accountBillCycleDayLocal);
          Integer bcd = bcdLocal;
          if (bcd == null) {
              // TODO If we have an add-on subscription with a BUNDLE alignment, this is incorrect as we need access to the base subscription
              bcd = BillCycleDayCalculator.calculateBcdForAlignment(null, this, this, billingAlignment, context, accountBillCycleDayLocal);
          }

          final BillingPeriod billingPeriod = curPlanPhase.getRecurring() != null ? curPlanPhase.getRecurring().getBillingPeriod() : BillingPeriod.NO_BILLING_PERIOD;
          final LocalDate resultingLocalDate = BillCycleDayCalculator.alignToNextBillCycleDate(prevTransitionDate, curTransitionDate, bcd, billingPeriod, context);
          final DateTime candidateResult = context.toUTCDateTime(resultingLocalDate);
          return candidateResult;
      }
  ```
  Applies to: mid-subscription catalog version upgrades / billing-event generation
  Confidence score: High

- [Kill-BR-0063] - Add-on cannot be created/changed past its own start-date guard relative to base
  Description: Add-on subscriptions inherit alignment to the bundle/base plan's start date rather than their own requested creation date (getBundleStartDateWithSanity forces bundleStartDate = baseSubscription.getStartDate() for ADD_ON), tying add-on phase/BCD alignment ("START_OF_BUNDLE") to the base plan's lifecycle rather than the add-on's individual request time. Evidence: SubscriptionApiBase.getBundleStartDateWithSanity line 245, cross-checked against PlanAligner's START_OF_BUNDLE branch which consumes bundleStartDate — the full downstream BCD interaction for BUNDLE-aligned add-ons is not fully traced through calculateBcdForAlignment's TODO-noted gap at DefaultSubscriptionBase.java:759.
  Line Numbers: 237 to 245
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/SubscriptionApiBase.java
  Source Code:
  ```java
              case ADD_ON:
                  if (baseSubscription == null) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_CREATE_NO_BP, bundleId);
                  }
                  if (effectiveDate.isBefore(baseSubscription.getStartDate())) {
                      throw new SubscriptionBaseApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, effectiveDate.toString(), baseSubscription.getStartDate().toString());
                  }
                  addonUtils.checkAddonCreationRights(baseSubscription, plan, effectiveDate, context);
                  return baseSubscription.getStartDate();
  ```
  Line Numbers: 757 to 760
  Source Code File Name: subscription/src/main/java/org/killbill/billing/subscription/api/user/DefaultSubscriptionBase.java
  Source Code:
  ```java
          Integer bcd = bcdLocal;
          if (bcd == null) {
              // TODO If we have an add-on subscription with a BUNDLE alignment, this is incorrect as we need access to the base subscription
              bcd = BillCycleDayCalculator.calculateBcdForAlignment(null, this, this, billingAlignment, context, accountBillCycleDayLocal);
  ```
  Applies to: add-on creation/alignment relative to base plan
  Confidence score: Medium

- [Kill-BR-0064] - Cross-source blocking-state OR aggregation
  Description: An entity is considered blocked (for change/entitlement/billing) if ANY blocking-state source says so — there is no override/precedence between sources, only cumulative OR. `DefaultBlockingChecker.DefaultBlockingAggregator.or(BlockingState)` (entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:50-57) ORs `blockChange`/`blockEntitlement`/`blockBilling` in from each state. `StatelessBlockingChecker.getBlockedState(account, bundle, subscription)` (entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java:25-32) ORs the subscription-level result, then the bundle-level result, then the account-level result together — once any level sets a flag true, nothing at a "lower" level can clear it.
  Line Numbers: 50 to 57
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  Source Code:
  ```java
          public void or(final BlockingState state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }
  ```
  Line Numbers: 25 to 32
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java
  Source Code:
  ```java
      public DefaultBlockingAggregator getBlockedState(final Iterable<BlockingState> accountEntitlementStates,
                                                       final Iterable<BlockingState> bundleEntitlementStates,
                                                       final Iterable<BlockingState> subscriptionEntitlementStates) {
          final DefaultBlockingAggregator result = getBlockedState(subscriptionEntitlementStates);
          result.or(getBlockedState(bundleEntitlementStates));
          result.or(getBlockedState(accountEntitlementStates));
          return result;
      }
  ```
  Applies to: BlockingChecker.getBlockedStatus / checkBlockedChange / checkBlockedEntitlement / checkBlockedBilling, used by entitlement creation, change-plan, cancellation
  Confidence score: High

- [Kill-BR-0065] - Account/bundle/subscription blocking-state hierarchy (no downward override)
  Description: `DefaultBlockingChecker.getBlockedStateSubscription` fetches the subscription's own state, then recursively pulls in the bundle's state (`getBlockedStateBundleId`), which in turn recursively pulls in the account's state (`getBlockedStateBundle` calls `getBlockedStateAccountId`) and ORs them all together (entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java:146-177). This encodes the domain rule that account- and bundle-level blocks (e.g. account suspended, bundle paused) always propagate down and block every subscription in scope; a subscription cannot be "more open" than its bundle or account.
  Line Numbers: 146 to 177
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  Source Code:
  ```java
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
  ```
  Applies to: Subscription, bundle and account blocking checks
  Confidence score: High

- [Kill-BR-0066] - Per-service "latest wins", not global latest
  Description: For each service name (ENTITLEMENT_SERVICE, billing-service, or any 3rd-party service like overdue), only the most recent (highest effective-date <= upTo) `BlockingState` row is kept before OR-aggregation — multiple services can each contribute a currently-active state simultaneously. Implemented in `DefaultEventsStream.filterCurrentBlockableStatePerService` (entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:448-471), which builds a `Map<serviceName, BlockingState>` keeping the latest state per service before calling `blockingChecker.getBlockedStatus(...)`.
  Line Numbers: 448 to 471
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  Source Code:
  ```java
      private List<BlockingState> filterCurrentBlockableStatePerService(final BlockingStateType type, final UUID blockableId, @Nullable final DateTime upTo) {

          final DateTime resolvedUpTo = upTo != null ? upTo : utcNow;

          final Map<String, BlockingState> currentBlockingStatePerService = new HashMap<>();
          for (final BlockingState blockingState : blockingStates) {
              if (!blockingState.getBlockedId().equals(blockableId)) {
                  continue;
              }
              if (blockingState.getType() != type) {
                  continue;
              }
              if (blockingState.getEffectiveDate().isAfter(resolvedUpTo)) {
                  continue;
              }

              if (currentBlockingStatePerService.get(blockingState.getService()) == null ||
                  !currentBlockingStatePerService.get(blockingState.getService()).getEffectiveDate().isAfter(blockingState.getEffectiveDate())) {
                  currentBlockingStatePerService.put(blockingState.getService(), blockingState);
              }
          }

          return List.<BlockingState>copyOf(currentBlockingStatePerService.values());
      }
  ```
  Applies to: Computing the current effective blocking status of an account/bundle/subscription across multiple independent blocking sources (e.g. entitlement service + overdue service simultaneously)
  Confidence score: High

- [Kill-BR-0067] - Entitlement state derivation precedence (CANCELLED > EXPIRED > PENDING > BLOCKED/ACTIVE)
  Description: `DefaultEventsStream.computeStateForEntitlement()` (entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:494-510) applies state precedence: if the entitlement-cancel effective date is now-or-past, state is CANCELLED (checked first, overrides everything). Otherwise, if there's an EXPIRED subscription-base transition in the past, state is EXPIRED. Otherwise, if the entitlement's effective start date is in the future, state is PENDING. Only if none of those apply does it fall back to aggregating blocking states across all services: BLOCKED if `currentStateBlockingAggregator.isBlockEntitlement()`, else ACTIVE.
  Line Numbers: 494 to 510
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  Source Code:
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
  Applies to: Entitlement.getState() / EventsStream.getEntitlementState()
  Confidence score: High

- [Kill-BR-0068] - Cancellation date validation and idempotency guard
  Description: `DefaultEntitlement.cancelEntitlementWithDateInternal` (entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:331-390) rejects a cancellation whose entitlement-effective-date or billing-effective-date is before the entitlement's/subscription's start date (`ErrorCode.SUB_INVALID_REQUESTED_DATE`), and rejects cancelling an entitlement that is already cancelled (`eventsStream.isEntitlementCancelled()` -> `ErrorCode.SUB_CANCEL_BAD_STATE`, line 364-366). On success it writes a SUBSCRIPTION-level blocking state with all three flags true (`blockChange=true, blockEntitlement=true, blockBilling=false`) under stateName `ENT_CANCELLED` (line 378).
  Line Numbers: 331 to 390
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  Source Code:
  ```java
      private Entitlement cancelEntitlementWithDateInternal(final DateTime entitlementEffectiveDate, final DateTime billingEffectiveDate,
                                                            final Iterable<PluginProperty> properties, final CallContext callContext) throws EntitlementApiException {
          logCancelEntitlement(log, this, entitlementEffectiveDate, billingEffectiveDate, null, null, null);
          checkForPermissions(Permission.ENTITLEMENT_CAN_CANCEL, callContext);
          // Get the latest state from disk
          refresh(callContext);
          if (entitlementEffectiveDate == null || entitlementEffectiveDate.compareTo(getEffectiveStartDate()) < 0) {
              throw new EntitlementApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, entitlementEffectiveDate, getEffectiveStartDate());
          }
          if (billingEffectiveDate != null && billingEffectiveDate.compareTo(getSubscriptionBase().getStartDate()) < 0) {
              throw new EntitlementApiException(ErrorCode.SUB_INVALID_REQUESTED_DATE, entitlementEffectiveDate, getEffectiveStartDate());
          }

          final BaseEntitlementWithAddOnsSpecifier baseEntitlementWithAddOnsSpecifier = new DefaultBaseEntitlementWithAddOnsSpecifier(
                  getBundleId(),
                  getBundleExternalKey(),
                  List.of(new DefaultEntitlementSpecifier(null, null, null, getExternalKey(), null)),
                  entitlementEffectiveDate,
                  billingEffectiveDate,
                  false);
          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = new ArrayList<BaseEntitlementWithAddOnsSpecifier>();
          baseEntitlementWithAddOnsSpecifierList.add(baseEntitlementWithAddOnsSpecifier);

          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.CANCEL_SUBSCRIPTION,
                                                                                 getAccountId(),
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 callContext);
          final WithEntitlementPlugin<Entitlement> cancelEntitlementWithPlugin = new WithEntitlementPlugin<Entitlement>() {
              @Override
              public Entitlement doCall(final EntitlementApi entitlementApi, final DefaultEntitlementContext updatedPluginContext) throws EntitlementApiException {
                  if (eventsStream.isEntitlementCancelled()) {
                      throw new EntitlementApiException(ErrorCode.SUB_CANCEL_BAD_STATE, getId(), EntitlementState.CANCELLED);
                  }
                  final InternalCallContext contextWithValidAccountRecordId = internalCallContextFactory.createInternalCallContext(getAccountId(), callContext);
                  try {
                      if (billingEffectiveDate != null) {
                          getSubscriptionBase().cancelWithDate(billingEffectiveDate, callContext);
                      } else {
                          getSubscriptionBase().cancel(callContext);
                      }

                  } catch (final SubscriptionBaseApiException e) {
                      throw new EntitlementApiException(e);
                  }
                  final BlockingState newBlockingState = new DefaultBlockingState(getId(), BlockingStateType.SUBSCRIPTION, DefaultEntitlementApi.ENT_STATE_CANCELLED, KILLBILL_SERVICES.ENTITLEMENT_SERVICE.getServiceName(), true, true, false, entitlementEffectiveDate);
                  final Collection<NotificationEvent> notificationEvents = new ArrayList<NotificationEvent>();
                  final Collection<BlockingState> addOnsBlockingStates = computeAddOnBlockingStates(entitlementEffectiveDate, notificationEvents, callContext, contextWithValidAccountRecordId);
                  // Record the new state first, then insert the notifications to avoid race conditions
                  setBlockingStates(newBlockingState, addOnsBlockingStates, contextWithValidAccountRecordId);
                  for (final NotificationEvent notificationEvent : notificationEvents) {
                      recordFutureNotification(entitlementEffectiveDate, notificationEvent, contextWithValidAccountRecordId);
                  }
                  return entitlementApi.getEntitlementForId(getId(), false, callContext);
              }
          };
          return pluginExecution.executeWithPlugin(cancelEntitlementWithPlugin, pluginContext);
      }
  ```
  Applies to: Entitlement.cancelEntitlementWithDate* family
  Confidence score: High

- [Kill-BR-0069] - Cancellation/change cascades to compatible add-ons
  Description: When a base-plan subscription is cancelled or changed, add-ons are auto-cancelled if they are no longer compatible with the new/absent base product. `DefaultEventsStream.isAddOnsNeedToBeBlocked` (entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java:389-423) blocks an add-on if the base subscription was cancelled outright, OR if the new base product's `included` add-on list contains the add-on's product (mandatory bundling), OR if the new base product's `available` add-on list does NOT contain the add-on's product (no longer offered) — and only for add-ons not already entitlement-cancelled. `DefaultEntitlement.computeAddOnBlockingStates` (entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:852-877) defers this computation to a notification if the trigger is in the future, so the actual cascade fires at the effective date.
  Line Numbers: 389 to 423
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  Source Code:
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
  Line Numbers: 852 to 877
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  Source Code:
  ```java
      public Collection<BlockingState> computeAddOnBlockingStates(final DateTime effectiveDate, final Collection<NotificationEvent> notificationEvents, final TenantContext context, final InternalCallContext internalCallContext) throws EntitlementApiException {
          // Optimization - bail early
          if (!ProductCategory.BASE.equals(getSubscriptionBase().getCategory())) {
              // Only base subscriptions have add-ons
              return Collections.emptyList();
          }

          // Get the latest state from disk (we just got cancelled or changed plan)
          refresh(context);

          // If cancellation/change occurs in the future, do nothing for now but add a notification entry.
          // This is to distinguish whether a future cancellation was requested by the user, or was a side effect
          // (e.g. base plan cancellation): future entitlement cancellations for add-ons on disk always reflect
          // an explicit cancellation. This trick lets us determine what to do when un-cancelling.
          // This mirror the behavior in subscription base (see DefaultSubscriptionBaseApiService).
          if (effectiveDate.compareTo(internalCallContext.getCreatedDate()) > 0) {
              // Note that usually we record the notification from the DAO. We cannot do it here because not all calls
              // go through the DAO (e.g. change)
              final boolean isBaseEntitlementCancelled = eventsStream.isEntitlementCancelled();
              final NotificationEvent notificationEvent = new EntitlementNotificationKey(getId(), getBundleId(), isBaseEntitlementCancelled ? EntitlementNotificationKeyAction.CANCEL : EntitlementNotificationKeyAction.CHANGE, effectiveDate);
              notificationEvents.add(notificationEvent);
              return Collections.emptyList();
          }

          return eventsStream.computeAddonsBlockingStatesForNextSubscriptionBaseEvent(effectiveDate);
      }
  ```
  Applies to: Base-subscription cancel and change-plan flows, add-on entitlements in the same bundle
  Confidence score: High

- [Kill-BR-0070] - Un-cancellation (uncancel) rules
  Description: `DefaultEntitlement.uncancelEntitlement` (entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:392-460) requires the subscription not be already fully CANCELLED (`eventsStream.isSubscriptionCancelled()` -> `ErrorCode.SUB_UNCANCEL_BAD_STATE`). If the entitlement is currently cancelled it deactivates that cancellation blocking-state row; else if there are pending (future-dated) entitlement-cancellation events it deactivates all of them; else (no cancellation at all, past or future) it throws `ErrorCode.ENT_UNCANCEL_BAD_STATE`. Only if billing itself had a future end date does it also call `subscriptionBase.uncancel(...)` — i.e. entitlement-level and billing-level cancellation are un-done independently based on their own state.
  Line Numbers: 392 to 460
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  Source Code:
  ```java
      @Override
      public void uncancelEntitlement(final Iterable<PluginProperty> properties, final CallContext callContext) throws EntitlementApiException {

          logUncancelEntitlement(log, this);

          checkForPermissions(Permission.ENTITLEMENT_CAN_CANCEL, callContext);

          // Get the latest state from disk
          refresh(callContext);

          final BaseEntitlementWithAddOnsSpecifier baseEntitlementWithAddOnsSpecifier = new DefaultBaseEntitlementWithAddOnsSpecifier(
                  getBundleId(),
                  getBundleExternalKey(),
                  null,
                  null,
                  null,
                  false);
          final List<BaseEntitlementWithAddOnsSpecifier> baseEntitlementWithAddOnsSpecifierList = new ArrayList<BaseEntitlementWithAddOnsSpecifier>();
          baseEntitlementWithAddOnsSpecifierList.add(baseEntitlementWithAddOnsSpecifier);
          final EntitlementContext pluginContext = new DefaultEntitlementContext(OperationType.UNDO_PENDING_SUBSCRIPTION_OPERATION,
                                                                                 getAccountId(),
                                                                                 null,
                                                                                 baseEntitlementWithAddOnsSpecifierList,
                                                                                 null,
                                                                                 properties,
                                                                                 callContext);

          final WithEntitlementPlugin<Void> uncancelEntitlementWithPlugin = new WithEntitlementPlugin<Void>() {

              @Override
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
              }
          };

          pluginExecution.executeWithPlugin(uncancelEntitlementWithPlugin, pluginContext);
      }
  ```
  Applies to: Entitlement.uncancelEntitlement
  Confidence score: High

- [Kill-BR-0071] - Bundle-level pause/resume blocks/unblocks all three axes uniformly
  Description: `DefaultEntitlementApiBase.pause`/`resume` (entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java:168-234) call `blockUnblockBundle` which writes a single SUBSCRIPTION_BUNDLE-level `BlockingState`: pause sets `blockBilling=true, blockEntitlement=true, blockChange=true` with stateName `ENT_BLOCKED`; resume sets all three flags false with stateName `ENT_CLEAR`. Because this is bundle-level, by rule #2 it blocks every subscription in the bundle regardless of their own subscription-level state.
  Line Numbers: 168 to 234
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/svcs/DefaultEntitlementApiBase.java
  Source Code:
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
  Applies to: EntitlementApi.pause/resume(bundleId)
  Confidence score: High

- [Kill-BR-0072] - Change-plan and add-entitlement are gated by isBlockChange/isBlockEntitlement, not just isBlockBilling
  Description: `DefaultEntitlementApi.preCheckAddEntitlement` (entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java:530-544) blocks adding a new (add-on) entitlement to a bundle if the base subscription's events stream reports `isBlockChange(requestedDate)` or `isBlockEntitlement(requestedDate)` true. Similarly `DefaultEntitlement.changePlanWithDate` (entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java:727-731) calls `checker.checkBlockedChange(subscriptionBase, resultingEffectiveDate, context)` before allowing a plan change to proceed, throwing `BlockingApiException`/`EntitlementApiException` if change is blocked at any aggregation level.
  Line Numbers: 530 to 544
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlementApi.java
  Source Code:
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
  Line Numbers: 727 to 731
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/api/DefaultEntitlement.java
  Source Code:
  ```java
                  try {
                      checker.checkBlockedChange(getSubscriptionBase(), resultingEffectiveDate, context);
                  } catch (final BlockingApiException e) {
                      throw new EntitlementApiException(e, e.getCode(), e.getMessage());
                  }
  ```
  Applies to: Add-on entitlement creation, plan change
  Confidence score: High

- [Kill-BR-0073] - Blocking-state deduplication/history compaction on write
  Description: `DefaultBlockingStateDao.setBlockingStatesAndPostBlockingTransitionEvent` (entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java:205-290) re-sorts the full history of blocking states for a given (blockedId, service), and if the newly-inserted state has the same `state` value as its immediate predecessor in the ordered sequence, it deletes (soft-unactivates) the redundant intermediate rows rather than keeping duplicate consecutive states — collapsing "t0 S1 t1 S2 t3 S3" insertions of S2 to a single S2 entry. This encodes the policy that only actual state *transitions* are meaningful/kept, not every write.
  Line Numbers: 205 to 290
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/dao/DefaultBlockingStateDao.java
  Source Code:
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

              return null;
          });
      }
  ```
  Applies to: Blocking-state persistence for account/bundle/subscription
  Confidence score: High

- [Kill-BR-0074] - Overdue-driven (and any third-party service) blocking integrates via the same generic per-service OR mechanism, with no entitlement-side special-casing
  Description: The entitlement module has no overdue-specific code path; `filterCurrentBlockableStatePerService` and `StatelessBlockingChecker`/`DefaultBlockingChecker` treat every `BlockingState.getService()` value uniformly (rules #1 and #3 above) — a blocking-state row posted by any service (e.g. an overdue-service state with `blockEntitlement=true`/`blockBilling=true`) is picked up as "the latest state for that service" and OR'd into the account/bundle/subscription aggregate exactly like an entitlement-service state, thereby driving `EntitlementState.BLOCKED` in `computeStateForEntitlement`. No source code for the overdue module itself is in this module, so the overdue side of the contract (what state name/flags it writes) is not verified here — only that entitlement's blocking machinery is generic per-service and will honor whatever overdue writes.
  Line Numbers: 448 to 471
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  Source Code:
  ```java
      private List<BlockingState> filterCurrentBlockableStatePerService(final BlockingStateType type, final UUID blockableId, @Nullable final DateTime upTo) {

          final DateTime resolvedUpTo = upTo != null ? upTo : utcNow;

          final Map<String, BlockingState> currentBlockingStatePerService = new HashMap<>();
          for (final BlockingState blockingState : blockingStates) {
              if (!blockingState.getBlockedId().equals(blockableId)) {
                  continue;
              }
              if (blockingState.getType() != type) {
                  continue;
              }
              if (blockingState.getEffectiveDate().isAfter(resolvedUpTo)) {
                  continue;
              }

              if (currentBlockingStatePerService.get(blockingState.getService()) == null ||
                  !currentBlockingStatePerService.get(blockingState.getService()).getEffectiveDate().isAfter(blockingState.getEffectiveDate())) {
                  currentBlockingStatePerService.put(blockingState.getService(), blockingState);
              }
          }

          return List.<BlockingState>copyOf(currentBlockingStatePerService.values());
      }
  ```
  Line Numbers: 25 to 32
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/block/StatelessBlockingChecker.java
  Source Code:
  ```java
      public DefaultBlockingAggregator getBlockedState(final Iterable<BlockingState> accountEntitlementStates,
                                                       final Iterable<BlockingState> bundleEntitlementStates,
                                                       final Iterable<BlockingState> subscriptionEntitlementStates) {
          final DefaultBlockingAggregator result = getBlockedState(subscriptionEntitlementStates);
          result.or(getBlockedState(bundleEntitlementStates));
          result.or(getBlockedState(accountEntitlementStates));
          return result;
      }
  ```
  Line Numbers: 50 to 57
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/block/DefaultBlockingChecker.java
  Source Code:
  ```java
          public void or(final BlockingState state) {
              if (state == null) {
                  return;
              }
              blockChange = blockChange || state.isBlockChange();
              blockEntitlement = blockEntitlement || state.isBlockEntitlement();
              blockBilling = blockBilling || state.isBlockBilling();
          }
  ```
  Line Numbers: 494 to 510
  Source Code File Name: entitlement/src/main/java/org/killbill/billing/entitlement/engine/core/DefaultEventsStream.java
  Source Code:
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
  Applies to: Interaction between overdue module and entitlement blocking/state computation
  Confidence score: Medium

- [Kill-BR-0075] - Leading/trailing period proration with optional fixed-days-in-month mode
  Description: `InvoiceDateUtils.calculateProRationBeforeFirstBillingPeriod`/`calculateProRationAfterLastBillingCycleDate`/`calculateProrationBetweenDates` compute a fractional-period charge for partial billing periods (subscription start/cancel mid-cycle). When `prorationFixedDays` (config `InvoiceConfig.getProrationFixedDays()`) is non-zero, the days-in-month used for the denominator is a fixed constant instead of the actual calendar days, avoiding proration drift across months of different lengths (`daysBetweenWithFixedDaysInMonth`, lines 92-99).
  Line Numbers: 92 to 99
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java
  Source Code:
  ```java
      public static int daysBetweenWithFixedDaysInMonth(final LocalDate startDate, final LocalDate endDate, final int fixedDaysInMonth) {
          final int daysBetween = Days.daysBetween(startDate, endDate).getDays();
          if(startDate.getMonthOfYear() == endDate.getMonthOfYear()) { //same month, no need for extra logic
              return daysBetween;
          }
          final int lastDayOfMonth = startDate.dayOfMonth().withMaximumValue().getDayOfMonth();
          return daysBetween - (lastDayOfMonth - fixedDaysInMonth);
      }
  ```
  Applies to: RECURRING invoice item amount calculation for partial first/last billing periods.
  Confidence score: High

- [Kill-BR-0076] - Whole-billing-period counting for bulk period advancement
  Description: `InvoiceDateUtils.calculateNumberOfWholeBillingPeriods` divides the elapsed days/weeks/months/years between two dates by the billing period length to determine how many full periods have elapsed, used to fast-forward recurring billing without generating one item per period.
  Line Numbers: 34 to 51
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoiceDateUtils.java
  Source Code:
  ```java
      public static int calculateNumberOfWholeBillingPeriods(final LocalDate startDate, final LocalDate endDate, final BillingPeriod billingPeriod) {
          final int numberBetween;
          final int numberInPeriod;
          if (billingPeriod.getPeriod().getDays() != 0) {
              numberBetween = Days.daysBetween(startDate, endDate).getDays();
              numberInPeriod = billingPeriod.getPeriod().getDays();
          } else if (billingPeriod.getPeriod().getWeeks() != 0) {
              numberBetween = Weeks.weeksBetween(startDate, endDate).getWeeks();
              numberInPeriod = billingPeriod.getPeriod().getWeeks();
          } else if (billingPeriod.getPeriod().getMonths() != 0) {
              numberBetween = Months.monthsBetween(startDate, endDate).getMonths();
              numberInPeriod = billingPeriod.getPeriod().getMonths();
          } else {
              numberBetween = Years.yearsBetween(startDate, endDate).getYears();
              numberInPeriod = billingPeriod.getPeriod().getYears();
          }
          return numberBetween / numberInPeriod;
      }
  ```
  Applies to: Recurring item generation across multiple elapsed billing periods.
  Confidence score: High

- [Kill-BR-0077] - IN_ARREAR vs IN_ADVANCE billing-mode effective end date, with "greedy" early in-arrear billing
  Description: `BillingIntervalDetail.java` computes the effective end date for a recurring item differently depending on catalog `BillingMode` — in-advance bills for the upcoming period, in-arrear bills for the period just completed — and clamps/aligns to the billing-cycle-day (BCD). A "greedy" `InArrearMode.GREEDY` variant (from `InvoiceConfig.getInArrearMode()`) allows billing an in-arrear period slightly before it has fully elapsed.
  Line Numbers: 125 to 211
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/BillingIntervalDetail.java
  Source Code:
  ```java
      private void calculateEffectiveEndDate() {
          if (billingMode == BillingMode.IN_ADVANCE) {
              calculateInAdvanceEffectiveEndDate();
          } else {
              calculateInArrearEffectiveEndDate();
          }
      }


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

      private void calculateInAdvanceEffectiveEndDate() {

          // We have an endDate and the targetDate is greater or equal to our endDate => return it
          if (endDate != null && !targetDate.isBefore(endDate)) {
              effectiveEndDate = endDate;
              return;
          }

          if (targetDate.isBefore(firstBillingCycleDate)) {
              effectiveEndDate = firstBillingCycleDate;
              return;
          }

          int numberOfPeriods = 0;
          LocalDate proposedDate = firstBillingCycleDate;

          while (!proposedDate.isAfter(targetDate)) {
              proposedDate = getFutureBillingDateFor(numberOfPeriods);
              numberOfPeriods += 1;
          }
          proposedDate = BillCycleDayCalculator.alignProposedBillCycleDate(proposedDate, billingCycleDay, billingPeriod);

          // The proposedDate is greater to our endDate => return it
          if (endDate != null && endDate.isBefore(proposedDate)) {
              effectiveEndDate = endDate;
          } else {
              effectiveEndDate = proposedDate;
          }
      }
  ```
  Applies to: RECURRING item start/end date computation per subscription phase.
  Confidence score: High

- [Kill-BR-0078] - Usage billing is IN_ARREAR only
  Description: `UsageInvoiceItemGenerator.java` throws `IllegalStateException` if a usage-billed phase is configured `IN_ADVANCE` — usage/metered billing is only supported after consumption is known.
  Line Numbers: 231 to 234
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/UsageInvoiceItemGenerator.java
  Source Code:
  ```java
      private void updatePerSubscriptionNextNotificationUsageDate(final UUID subscriptionId, final Map<String, LocalDate> nextBillingCycleDates, final BillingMode usageBillingMode, final Map<UUID, SubscriptionFutureNotificationDates> perSubscriptionFutureNotificationDates) {
          if (usageBillingMode == BillingMode.IN_ADVANCE) {
              throw new IllegalStateException("Not implemented Yet)");
          }
  ```
  Applies to: USAGE invoice item generation.
  Confidence score: High

- [Kill-BR-0079] - Invoice items are append-only; retroactive changes produce REPAIR_ADJ items instead of editing history
  Description: `SubscriptionItemTree`/`ItemsNodeInterval` diff existing vs. proposed billing periods per subscription. When a subscription changes retroactively (plan change, cancellation, backdated event), the old RECURRING item is never mutated — instead a `REPAIR_ADJ` (`ItemAction.CANCEL`) item is generated to cancel out the un-owed portion, and new RECURRING items are added for the new plan (`ItemsNodeInterval.addProposedItem`/`mergeExistingAndProposed`, `SubscriptionItemTree.mergeProposedItem`/`buildForMerge`).
  Line Numbers: 166 to 199
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java
  Source Code:
  ```java
      public void mergeProposedItem(final InvoiceItem invoiceItem) {
          Preconditions.checkState(!isBuilt, "Tree already built, unable to add new invoiceItem=%s", invoiceItem);

          // Check if it was an existing item ignored for tree purposes (e.g. FIXED or $0 RECURRING, both of which aren't repaired)
          final boolean isItemExist = existingIgnoredItems.stream().anyMatch(input -> input.matches(invoiceItem));
          if (isItemExist) {
              return;
          }

          switch (invoiceItem.getInvoiceItemType()) {
              case RECURRING:
                  // merged means we've either matched the proposed to an existing, or triggered a repair
                  final List<ItemsNodeInterval> newNodes = root.addProposedItem(new ItemsNodeInterval(root, new Item(invoiceItem, targetInvoiceId, ItemAction.ADD, prorationFixedDays), prorationFixedDays));
                  for (final ItemsNodeInterval cur : newNodes) {
                      items.addAll(cur.getItems());
                  }
                  break;

              case FIXED:
                  remainingIgnoredItems.add(invoiceItem);
                  break;

              default:
                  Preconditions.checkState(false, "Unexpected proposed item " + invoiceItem);
          }
      }

      // Build tree post merge
      public void buildForMerge() {
          Preconditions.checkState(!isBuilt, "Tree already built");
          root.mergeExistingAndProposed(items, targetInvoiceId);
          isBuilt = true;
          isMerged = true;
      }
  ```
  Line Numbers: 161 to 262
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/tree/ItemsNodeInterval.java
  Source Code:
  ```java
      public List<ItemsNodeInterval> addProposedItem(final ItemsNodeInterval newNode) {
          Preconditions.checkState(newNode.getItems().size() == 1, "Invalid node=%s", newNode);

          final List<ItemsNodeInterval> newNodes = new LinkedList<>();

          addNode(newNode, new AddNodeCallback() {
              @Override
              public boolean onExistingNode(final NodeInterval existingNode, final ItemsNodeInterval updatedNewNode) {

                  final Item item = updatedNewNode.getItems().get(0);

                  // If we receive a new proposed that is the same kind as the reversed existing (current node),
                  // we match existing and proposed. If not, we keep the proposed item as-is outside of the tree.
                  if (isSameKind((ItemsNodeInterval) existingNode, item)) {
                      final ItemsInterval existingOrNewNodeItems = ((ItemsNodeInterval) existingNode).getItemsInterval();
                      existingOrNewNodeItems.cancelItems(item);
                      return true;
                  } else {
                      newNodes.add(updatedNewNode);
                      return false;
                  }
              }

              @Override
              public boolean shouldInsertNode(final NodeInterval insertionNode, final ItemsNodeInterval updatedNewNode) {

                  final Item item = updatedNewNode.getItems().get(0);

                  // At this stage, we're currently merging a proposed item that does not fit any of the existing intervals.
                  // If this new node is about to be inserted at the root level, this means the proposed item overlaps any
                  // existing item. We keep these as-is, outside of the tree: they will become part of the resulting list.
                  if (insertionNode.isRoot()) {
                      // If updatedNewNode was rebalanced and it is fully repaired by its children it just gets canceled out.
                      LocalDate curDate = updatedNewNode.start;
                      NodeInterval curChild = updatedNewNode.leftChild;
                      while (curChild != null &&
                             curChild.start.equals(curDate) &&
                             isSameKind((ItemsNodeInterval) curChild, item)) {
                          curDate = curChild.end;
                          curChild = curChild.rightSibling;
                      }

                      if (curDate.equals(updatedNewNode.end)) {
                          updatedNewNode.getItems().clear();
                          curChild = updatedNewNode.leftChild;
                          while (curChild != null) {
                              ((ItemsNodeInterval) curChild).getItems().clear();
                              curChild = curChild.rightSibling;
                          }
                          return false;
                      }

                      newNodes.add(updatedNewNode);
                      return false;
                  }

                  // If we receive a new proposed that is the same kind as the reversed existing (parent node),
                  // we want to insert it to generate a piece of repair (see SubscriptionItemTree#buildForMerge).
                  // If not, we keep the proposed item as-is outside of the tree.
                  final boolean result = isSameKind((ItemsNodeInterval) insertionNode, item);
                  if (!result) {
                      newNodes.add(updatedNewNode);
                  }
                  return result;
              }

              private boolean isSameKind(final ItemsNodeInterval insertionNode, final Item item) {
                  final List<Item> insertionNodeItems = insertionNode.getItems();
                  Preconditions.checkState(insertionNodeItems.size() == 1, "Expected existing node to have only one item");
                  final Item insertionNodeItem = insertionNodeItems.get(0);
                  return insertionNodeItem.isSameKind(item);
              }
          });

          return newNodes;
      }

      /**
       * The merge tree is initially constructed by flattening all the existing items and reversing them (CANCEL node).
       * <p/>
       * That means that if we were to not merge any new proposed items, we would end up with only those reversed existing
       * items, and they would all end up repaired-- which is what we want.
       * <p/>
       * However, if there are new proposed items, then we look to see if they are children one our existing reverse items
       * so that we can generate the repair pieces missing. For e.g, below is one scenario among so many:
       * <p/>
       * <pre>
       * D1                                                  D2
       * |---------------------------------------------------| (existing reversed (CANCEL) item
       *       D1'             D2'
       *       |---------------| (proposed same plan)
       * </pre>
       * In that case we want to generated a repair for [D1, D1') and [D2',D2)
       * <p/>
       * Note that this tree is never very deep, only 3 levels max, with exiting at the first level
       * and proposed that are the for the exact same plan but for different dates below.
       *
       * @param output result list of built items
       */
      public void mergeExistingAndProposed(final Collection<Item> output, final UUID targetInvoiceId) {
          build(output, targetInvoiceId, true);
      }
  ```
  Applies to: All RECURRING items on committed/existing invoices when a subscription is changed after invoicing.
  Confidence score: High

- [Kill-BR-0080] - $0 RECURRING items and all FIXED items are excluded from repair
  Description: `SubscriptionItemTree.addItem` explicitly ignores zero-amount RECURRING items (`existingIgnoredItems.add`, citing GH #783 "Nothing to repair") and all FIXED items, which are never candidates for the repair-diff tree.
  Line Numbers: 88 to 116
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java
  Source Code:
  ```java
      public void addItem(final InvoiceItem invoiceItem) {
          Preconditions.checkState(!isBuilt, "Tree already built, unable to add new invoiceItem=%s", invoiceItem);

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
      }
  ```
  Applies to: Repair/retroactive-change logic scope.
  Confidence score: High

- [Kill-BR-0081] - Over-repair / double-repair guard (no more can be repaired+adjusted than the original amount)
  Description: `ItemsNodeInterval.validateTree` and `InvoicePruner.AdjustedRecurringItem.build` enforce that the sum of REPAIR_ADJ + ITEM_ADJ against a RECURRING item never exceeds its original amount; violations throw `Preconditions.checkState` failures (surfaced as `InvoiceApiException` with `UNEXPECTED_ERROR`), except for a documented bypass when ITEM_ADJ is created directly by an invoice plugin (`InvoicePruner.java` lines 196-227, explicit comment referencing this).
  Line Numbers: 347 to 417
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/tree/ItemsNodeInterval.java
  Source Code:
  ```java
      private void validateTree() {
          final NodeInterval root = this;
          walkTree(new WalkCallback() {
              @Override
              public void onCurrentNode(final int depth, final NodeInterval curNode, final NodeInterval parent) {

                  if (curNode.isRoot()) {
                      return;
                  }

                  final ItemsInterval curNodeItems = ((ItemsNodeInterval) curNode).getItemsInterval();
                  for (final Item curCancelItem : curNodeItems.get_CANCEL_items()) {
                      // Sanity: cancelled items should only be in the same node or parents
                      if (curNode.getLeftChild() != null) {
                          curNode.getLeftChild()
                                 .walkTree(new WalkCallback() {
                                     @Override
                                     public void onCurrentNode(final int depth, final NodeInterval curNode, final NodeInterval parent) {
                                         final ItemsInterval curChildItems = ((ItemsNodeInterval) curNode).getItemsInterval();
                                         final Item cancelledItem = curChildItems.getCancelledItemIfExists(curCancelItem.getLinkedId());
                                         Preconditions.checkState(cancelledItem == null, "Invalid cancelledItem=%s for cancelItem=%s", cancelledItem, curCancelItem);
                                     }
                                 });
                      }

                      // Sanity: make sure the CANCEL item points to an ADD item
                      final NodeInterval nodeIntervalForCancelledItem = root.findNode(new SearchCallback() {
                          @Override
                          public boolean isMatch(final NodeInterval curNode) {
                              final ItemsInterval curChildItems = ((ItemsNodeInterval) curNode).getItemsInterval();
                              final Item cancelledItem = curChildItems.getCancelledItemIfExists(curCancelItem.getLinkedId());
                              return cancelledItem != null;
                          }
                      });
                      Preconditions.checkState(nodeIntervalForCancelledItem != null, "Missing cancelledItem for cancelItem=%s", curCancelItem);
                  }

                  for (final Item curAddItem : curNodeItems.get_ADD_items()) {
                      // Sanity: verify the item hasn't been repaired too much
                      if (curNode.getLeftChild() != null) {
                          final AtomicReference<BigDecimal> totalRepaired = new AtomicReference<BigDecimal>(BigDecimal.ZERO);
                          curNode.getLeftChild()
                                 .walkTree(new WalkCallback() {
                                     @Override
                                     public void onCurrentNode(final int depth, final NodeInterval curNode, final NodeInterval parent) {
                                         final ItemsInterval curChildItems = ((ItemsNodeInterval) curNode).getItemsInterval();
                                         final Item cancelledItem = curChildItems.getCancellingItemIfExists(curAddItem.getId());
                                         if (cancelledItem != null && curAddItem.getId().equals(cancelledItem.getLinkedId())) {
                                             totalRepaired.set(totalRepaired.get().add(cancelledItem.getAmount()));
                                         }
                                     }
                                 });
                          Preconditions.checkState(curAddItem.getNetAmount().compareTo(totalRepaired.get()) >= 0, "Item %s overly repaired", curAddItem);
                      }

                      // Old behavior compatibility for full item adjustment (Temp code should go away as move in time)
                      // If we see a fully adjusted item and an existing child (one ADD item), we discard the fully adjusted item
                      // in such a way that we are left with the child that will look like the proposed and nothing will be generated.
                      if (curAddItem.isFullyAdjusted()) {
                          final NodeInterval leftChild = curNode.getLeftChild();
                          if (leftChild != null) {
                              final ItemsInterval leftChildItems = ((ItemsNodeInterval) leftChild).getItemsInterval();
                              if (leftChildItems.getItems().size() == 1 && leftChildItems.getItems().get(0).getAction() == ItemAction.ADD) {
                                  curNodeItems.remove(curAddItem);
                              }
                          }
                      }
                  }
              }
          });
      }
  ```
  Line Numbers: 196 to 227
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java
  Source Code:
  ```java
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
  ```
  Applies to: RECURRING items with associated REPAIR_ADJ/ITEM_ADJ.
  Confidence score: High

- [Kill-BR-0082] - Fully-repaired item closure exclusion
  Description: `InvoicePruner.getFullyRepairedItemsClosure` computes the set of item IDs (original + all its repairs) that have been completely repaired, so they are pruned from the "existing items" view before the repair tree is (re-)built — preventing duplicate/overlapping node insertion in complex multi-generation repairs (GH #1205).
  Line Numbers: 89 to 99
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/InvoicePruner.java
  Source Code:
  ```java
      public Set<UUID> getFullyRepairedItemsClosure() throws InvoiceApiException {
          try {
              final Set<UUID> result = new HashSet();
              for (AdjustedRecurringItem cur : map.values()) {
                  result.addAll(cur.getFullyRepairedLinkedItems());
              }
              return result;
          } catch (final IllegalStateException e) {
              throw new InvoiceApiException(e, ErrorCode.UNEXPECTED_ERROR, String.format("ILLEGAL INVOICING STATE"));
          }
      }
  ```
  Applies to: Existing-invoice-item view construction prior to generating a new invoice.
  Confidence score: High

- [Kill-BR-0083] - No double-billing / no double-repair chronological invariant
  Description: `SubscriptionItemTree.checkItemsListState` walks the final flattened item list per subscription and asserts (via `Preconditions.checkState`) that consecutive RECURRING items and consecutive REPAIR_ADJ items never overlap in time — a structural correctness guarantee on the generated invoice.
  Line Numbers: 220 to 247
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/tree/SubscriptionItemTree.java
  Source Code:
  ```java
      private void checkItemsListState(final List<InvoiceItem> orderedList) {

          LocalDate prevRecurringEndDate = null;
          LocalDate prevRepairEndDate = null;
          for (final InvoiceItem cur : orderedList) {
              switch (cur.getInvoiceItemType()) {
                  case FIXED:
                      break;

                  case RECURRING:
                      if (prevRecurringEndDate != null) {
                          Preconditions.checkState(prevRecurringEndDate.compareTo(cur.getStartDate()) <= 0);
                      }
                      prevRecurringEndDate = cur.getEndDate();
                      break;

                  case REPAIR_ADJ:
                      if (prevRepairEndDate != null) {
                          Preconditions.checkState(prevRepairEndDate.compareTo(cur.getStartDate()) <= 0);
                      }
                      prevRepairEndDate = cur.getEndDate();
                      break;

                  default:
                      Preconditions.checkState(false, "Unexpected item type " + cur.getInvoiceItemType());
              }
          }
      }
  ```
  Applies to: Final invoice item list per subscription before being written to disk.
  Confidence score: High

- [Kill-BR-0084] - Account-level AUTO_INVOICING_OFF suppresses scheduled invoicing but not explicit API calls
  Description: `InvoiceDispatcher.processAccountInternal` (line 387): `if (!isApiCall && billingEvents.isAccountAutoInvoiceOff()) { return Collections.emptyList(); }` — background/notification-triggered invoice runs are skipped entirely for an account tagged with the auto-invoice-off control tag, but a direct API-triggered invoice run still proceeds.
  Line Numbers: 387 to 389
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  Source Code:
  ```java
              if (!isApiCall && billingEvents.isAccountAutoInvoiceOff()) {
                  return Collections.emptyList();
              }
  ```
  Applies to: Whole-account invoice generation triggering.
  Confidence score: High

- [Kill-BR-0085] - Per-subscription auto-invoice-off exclusion
  Description: `FixedAndRecurringInvoiceItemGenerator.java` excludes subscriptions listed in `eventSet.getSubscriptionIdsWithAutoInvoiceOff()` both from the existing-items tree and from the proposed billing events, so no items are generated or repaired for those subscriptions even while the rest of the account invoices normally.
  Line Numbers: 102 to 116
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  Source Code:
  ```java
          for (final Invoice invoice : existingInvoices.getInvoices()) {
              for (final InvoiceItem item : invoice.getInvoiceItems()) {
                  if (toBeIgnored.contains(item.getId())) {
                      continue;
                  }

                  if (item.getSubscriptionId() == null || // Always include migration invoices, credits, external charges etc.
                      !eventSet.getSubscriptionIdsWithAutoInvoiceOff()
                               .contains(item.getSubscriptionId())) { //don't add items with auto_invoice_off tag

                      accountItemTree.addExistingItem(item);
                      trackInvoiceItemCreatedDay(item, createdItemsPerDayPerSubscription, internalCallContext);
                  }
              }
          }
  ```
  Applies to: Per-subscription invoice item generation.
  Confidence score: High

- [Kill-BR-0086] - Removing AUTO_INVOICING_OFF triggers catch-up invoicing
  Description: `InvoiceTagHandler.java` listens for removal of the AUTO_INVOICING_OFF control tag and triggers invoice generation to catch up on unpaid/un-generated invoices that were withheld while the tag was set.
  Line Numbers: 72 to 84
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceTagHandler.java
  Source Code:
  ```java
          final SubscriberAction<ControlTagDeletionInternalEvent> action = new SubscriberAction<ControlTagDeletionInternalEvent>() {
              @Override
              public void run(final ControlTagDeletionInternalEvent event) {
                  if (event.getTagDefinition().getName().equals(ControlTagType.AUTO_INVOICING_OFF.toString()) && event.getObjectType() == ObjectType.ACCOUNT) {
                      final UUID accountId = event.getObjectId();
                      final InternalCallContext context = internalCallContextFactory.createInternalCallContext(event.getSearchKey2(), event.getSearchKey1(), "InvoiceTagHandler", CallOrigin.INTERNAL, UserType.SYSTEM, event.getUserToken());
                      processUnpaid_AUTO_INVOICING_OFF_invoices(accountId, context);
                  }
              }
          };
          subscriberQueueHandler.subscribe(ControlTagDeletionInternalEvent.class, action);
          this.retryableSubscriber = new RetryableSubscriber(clock, this, subscriberQueueHandler);
      }
  ```
  Applies to: Account-level control-tag transition handling.
  Confidence score: High

- [Kill-BR-0087] - AUTO_INVOICING_DRAFT tag governs generated-invoice status
  Description: `DefaultInvoiceGenerator.java` sets the new invoice's status to DRAFT instead of COMMITTED when `events.isAccountAutoInvoiceDraft()` is true; otherwise COMMITTED.
  Line Numbers: 91 to 94
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java
  Source Code:
  ```java
          final InvoiceStatus invoiceStatus = events.isAccountAutoInvoiceDraft() ? InvoiceStatus.DRAFT : InvoiceStatus.COMMITTED;
          final DefaultInvoice invoice = targetInvoiceId != null ?
                                         new DefaultInvoice(targetInvoiceId, account.getId(), null, invoiceDate, adjustedTargetDate, targetCurrency, false, invoiceStatus) :
                                         new DefaultInvoice(account.getId(), invoiceDate, adjustedTargetDate, targetCurrency, invoiceStatus);
  ```
  Applies to: Newly generated invoice status assignment.
  Confidence score: High

- [Kill-BR-0088] - AUTO_INVOICING_REUSE_DRAFT reuses an existing DRAFT invoice
  Description: `InvoiceDispatcher.java` (`isPropertyReuseDraftSet`/`getReuseDraftInvoiceId`, and generation logic around line 606-611 referencing GH #1313) causes new items to be merged into an existing DRAFT invoice rather than creating a new one when this flag/tag is active, keeping only the latest augmented invoice with all currently-generated items.
  Line Numbers: 606 to 611
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  Source Code:
  ```java
                  // In order to handle AUTO_INVOICING_REUSE_DRAFT where the invoice is being reused
                  // we need to only keep the latest invoice with all the items currently being generated
                  // See https://github.com/killbill/killbill/issues/1313
                  final UUID additionalInvoiceId = additionalInvoice.getId();
                  augmentedExistingInvoices.removeIf(input -> input.getId().equals(additionalInvoiceId));
                  augmentedExistingInvoices.add(additionalInvoice);
  ```
  Applies to: Invoice generation target selection (new vs. existing DRAFT invoice).
  Confidence score: Medium

- [Kill-BR-0089] - Target-date validation and clamping
  Description: `DefaultInvoiceGenerator.validateTargetDate` rejects target dates too far in the future (bounded by `InvoiceConfig.getNumberOfMonthsInFuture`), and `adjustTargetDate` prevents generating an invoice with a target date earlier than what has already been invoiced.
  Line Numbers: 120 to 154
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/DefaultInvoiceGenerator.java
  Source Code:
  ```java
      private void validateTargetDate(final LocalDate targetDate, final InternalTenantContext context) throws InvoiceApiException {
          final int maximumNumberOfMonths = config.getNumberOfMonthsInFuture(context);

          if (Months.monthsBetween(clock.getUTCToday(), targetDate).getMonths() > maximumNumberOfMonths) {
              throw new InvoiceApiException(ErrorCode.INVOICE_TARGET_DATE_TOO_FAR_IN_THE_FUTURE, targetDate.toString());
          }
      }

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
  Applies to: Invoice generation target date input (both API-driven and dry-run).
  Confidence score: High

- [Kill-BR-0090] - Dry-run invoice generation never commits to disk, three distinct modes
  Description: `InvoiceDispatcher.processAccountInternal`/`processDryRun_UPCOMING_INVOICE_Invoice`/`processDryRun_TARGET_DATE_Invoice` branch entirely away from the real-invoicing path for `DryRunType.UPCOMING_INVOICE`, `TARGET_DATE`, and `SUBSCRIPTION_ACTION`; results are computed and returned but `commitInvoiceAndSetFutureNotifications`/`invoiceDao.createInvoices` are never invoked for dry-run paths.
  Line Numbers: 403 to 439
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  Source Code:
  ```java
              } else /* Dry run use cases */ {
                  // Splitting invoices has not been implemented for dry-run use cases to we simply add the one invoice in the result list
                  result = new ArrayList<>();
                  final Invoice invoice;
                  final NotificationQueue notificationQueue = notificationQueueService.getNotificationQueue(KILLBILL_SERVICES.INVOICE_SERVICE.getServiceName(),
                                                                                                            DefaultNextBillingDateNotifier.NEXT_BILLING_DATE_NOTIFIER_QUEUE);
                  final Iterable<NotificationEventWithMetadata<NextBillingDateNotificationKey>> futureNotificationsIterable = notificationQueue.getFutureNotificationForSearchKeys(context.getAccountRecordId(), context.getTenantRecordId());

                  // Copy the results as retrieving the iterator will issue a query each time. This also makes sure the underlying JDBC connection is closed.
                  final List<NotificationEventWithMetadata<NextBillingDateNotificationKey>> futureNotifications = Iterables.toUnmodifiableList(futureNotificationsIterable);

                  final Map<UUID, DateTime> nextScheduledSubscriptionsEventMap = getNextTransitionsForSubscriptions(billingEvents);

                  // List of all existing invoice notifications
                  final Set<LocalDate> allCandidateTargetDates = getUpcomingInvoiceCandidateDates(futureNotifications, nextScheduledSubscriptionsEventMap, Collections.emptyList(), context);

                  if (dryRunArguments.getDryRunType() == DryRunType.UPCOMING_INVOICE) {

                      final Iterable<UUID> filteredSubscriptionIdsForDryRun = getFilteredSubscriptionIdsFor_UPCOMING_INVOICE_DryRun(dryRunArguments, billingEvents);

                      // List of existing invoice notifications associated to the filter set of subscriptionIds
                      final Set<LocalDate> filteredCandidateTargetDates = Iterables.isEmpty(filteredSubscriptionIdsForDryRun) ?
                                                                          allCandidateTargetDates :
                                                                          getUpcomingInvoiceCandidateDates(futureNotifications, nextScheduledSubscriptionsEventMap, filteredSubscriptionIdsForDryRun, context);

                      if (Iterables.isEmpty(filteredSubscriptionIdsForDryRun)) {
                          invoice = processDryRun_UPCOMING_INVOICE_Invoice(accountId, allCandidateTargetDates, billingEvents, accountInvoices, dryRunInfo, invoiceTimings, properties, context);
                      } else {
                          invoice = processDryRun_UPCOMING_INVOICE_FILTERING_Invoice(accountId, filteredCandidateTargetDates, allCandidateTargetDates, billingEvents, accountInvoices, dryRunInfo, invoiceTimings, properties, context);
                      }
                  } else /* DryRunType.TARGET_DATE, SUBSCRIPTION_ACTION */ {
                      invoice = processDryRun_TARGET_DATE_Invoice(accountId, inputTargetDate, allCandidateTargetDates, billingEvents, accountInvoices, dryRunInfo, invoiceTimings, properties, context);
                  }
                  if (invoice !=  null) {
                      result.add(invoice);
                  }
              }
  ```
  Applies to: Preview/what-if invoice computation (upcoming invoice preview, target-date preview, subscription-change preview).
  Confidence score: High

- [Kill-BR-0091] - Invoice voiding immutability rules
  Description: `DefaultInvoiceUserApi.voidInvoice` (lines 771-799) applies, for COMMITTED invoices: (a) `checkInvoiceNotRepaired` — throws `CAN_NOT_VOID_INVOICE_THAT_IS_REPAIRED` if `invoice.getIsRepaired()`; (b) `checkInvoiceDoesContainUsedGeneratedCredit` — throws `CAN_NOT_VOID_INVOICE_THAT_GENERATED_USED_CREDIT` if the invoice generated a CBA credit that has since been partly/fully consumed elsewhere; and always (c) `checkInvoiceNotPaid` — throws `CAN_NOT_VOID_INVOICE_THAT_IS_PAID` if `amountPaid + amountRefunded != 0`. This directly verifies/refines the prior architecture doc's immutability claims: voiding (not just repair) is itself gated by whether the invoice has been repaired, paid, or has already-consumed generated credit.
  Line Numbers: 771 to 799
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
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
  Applies to: `voidInvoice` API — governs which committed invoices can transition to VOID.
  Confidence score: High

- [Kill-BR-0092] - Invoice status transitions are one-directional and idempotency-guarded
  Description: `DefaultInvoiceDao.changeInvoiceStatus` (line 1372) throws `INVOICE_INVALID_STATUS` if the requested new status equals the current status, or if the invoice is already VOID (VOID is terminal — no transition out of VOID is permitted).
  Line Numbers: 1372 to 1389
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
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
  ```
  Applies to: All invoice status transitions (DRAFT→COMMITTED, COMMITTED→VOID, etc.).
  Confidence score: High

- [Kill-BR-0093] - Adjustable item-type restriction
  Description: `DefaultInvoiceDao.validateInvoiceItemToBeAdjusted` (line 1362) throws `INVOICE_ITEM_ADJUSTMENT_ITEM_INVALID` unless the linked target item's type is in `INVOICE_ITEM_TYPES_ADJUSTABLE` (EXTERNAL_CHARGE, FIXED, RECURRING, TAX, USAGE, PARENT_SUMMARY) — e.g. a CBA_ADJ or another ITEM_ADJ cannot itself be the target of an item adjustment.
  Line Numbers: 1362 to 1370
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
  ```java
      private void validateInvoiceItemToBeAdjusted(final InvoiceItemSqlDao invoiceItemSqlDao, final InvoiceItemModelDao invoiceItemModelDao, final InternalCallContext context) throws InvoiceApiException {
          Preconditions.checkNotNull(invoiceItemModelDao.getLinkedItemId(), "LinkedItemId cannot be null for ITEM_ADJ item: " + invoiceItemModelDao);
          // Note: this assumes the linked item has already been created in or prior to the transaction, which should almost always be the case
          // (unless some whacky plugin creates an out-of-order item adjustment on a subsequent external charge)
          final InvoiceItemModelDao invoiceItemToBeAdjusted = invoiceItemSqlDao.getById(invoiceItemModelDao.getLinkedItemId().toString(), context);
          if (!INVOICE_ITEM_TYPES_ADJUSTABLE.contains(invoiceItemToBeAdjusted.getType())) {
              throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_ADJUSTMENT_ITEM_INVALID, invoiceItemToBeAdjusted.getId());
          }
      }
  ```
  Applies to: ITEM_ADJ creation (`insertInvoiceItemAdjustment`, refund-driven adjustments).
  Confidence score: High

- [Kill-BR-0094] - Item-adjustment/refund amount cannot exceed remaining adjustable amount
  Description: `InvoiceDaoHelper.computeItemAdjustments`/`computeItemAdjustmentAmount` compute the max amount still adjustable on an item (original amount minus prior adjustments/repairs); `computePositiveRefundAmount` validates a requested refund against the amount actually paid. `DefaultInvoiceUserApi.insertInvoiceItemAdjustment` additionally rejects non-positive adjustment amounts (`INVOICE_ITEM_ADJUSTMENT_AMOUNT_SHOULD_BE_POSITIVE`) and currency mismatches against the invoice currency.
  Line Numbers: 86 to 175
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/InvoiceDaoHelper.java
  Source Code:
  ```java
      public Map<UUID, BigDecimal> computeItemAdjustments(final String invoiceId,
                                                          final List<CustomField> invoiceCustomFields,
                                                          final List<Tag> invoicesTags,
                                                          final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                          final Map<UUID, BigDecimal> invoiceItemIdsWithNullAmounts,
                                                          final InternalTenantContext context) throws InvoiceApiException {
          // Populate the missing amounts for individual items, if needed
          final Map<UUID, BigDecimal> outputItemIdsWithAmounts = new HashMap<>();
          // Retrieve invoice before the Refund
          final InvoiceModelDao invoice = entitySqlDaoWrapperFactory.become(InvoiceSqlDao.class).getById(invoiceId, context);
          if (invoice != null) {
              populateChildren(invoice, invoiceCustomFields, invoicesTags, false, entitySqlDaoWrapperFactory, context);
          } else {
              throw new IllegalStateException("Invoice shouldn't be null for id " + invoiceId);
          }

          //
          // If we have an item amount, we 'd like to use it, but we need to check first that it is lesser or equal than maximum allowed
          //If, not we compute maximum value we can adjust per item
          for (final UUID invoiceItemId : invoiceItemIdsWithNullAmounts.keySet()) {
              final List<InvoiceItemModelDao> adjustedOrRepairedItems = entitySqlDaoWrapperFactory.become(InvoiceItemSqlDao.class).getAdjustedOrRepairedInvoiceItemsByLinkedId(invoiceItemId.toString(), context);
              computeItemAdjustmentsForTargetInvoiceItem(getInvoiceItemForId(invoice, invoiceItemId), adjustedOrRepairedItems, invoiceItemIdsWithNullAmounts, outputItemIdsWithAmounts);
          }
          return outputItemIdsWithAmounts;
      }

      private static void computeItemAdjustmentsForTargetInvoiceItem(final InvoiceItemModelDao targetInvoiceItem,
                                                                     final List<InvoiceItemModelDao> adjustedOrRepairedItems,
                                                                     final Map<UUID, BigDecimal> inputAdjInvoiceItem,
                                                                     final Map<UUID, BigDecimal> outputAdjInvoiceItem) throws InvoiceApiException {
          final BigDecimal originalItemAmount = targetInvoiceItem.getAmount();
          final BigDecimal maxAdjLeftAmount = computeItemAdjustmentAmount(originalItemAmount, adjustedOrRepairedItems);

          final BigDecimal proposedItemAmount = inputAdjInvoiceItem.get(targetInvoiceItem.getId());
          if (proposedItemAmount != null && proposedItemAmount.compareTo(maxAdjLeftAmount) > 0) {
              throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_ADJUSTMENT_AMOUNT_INVALID, proposedItemAmount, maxAdjLeftAmount);
          }

          final BigDecimal itemAmountToAdjust = Objects.requireNonNullElse(proposedItemAmount, maxAdjLeftAmount);
          if (itemAmountToAdjust.compareTo(BigDecimal.ZERO) > 0) {
              outputAdjInvoiceItem.put(targetInvoiceItem.getId(), itemAmountToAdjust);
          }
      }

      /**
       * @param requestedPositiveAmountToAdjust amount we are adjusting for that item
       * @param adjustedOrRepairedItems         list of all adjusted or repaired linking to this item
       * @return the amount we should really adjust based on whether or not the item got repaired
       */
      private static BigDecimal computeItemAdjustmentAmount(final BigDecimal requestedPositiveAmountToAdjust, final List<InvoiceItemModelDao> adjustedOrRepairedItems) {

          BigDecimal positiveAdjustedOrRepairedAmount = BigDecimal.ZERO;

          for (final InvoiceItemModelDao cur : adjustedOrRepairedItems) {
              // Adjustment or repair items are negative so we negate to make it positive
              positiveAdjustedOrRepairedAmount = positiveAdjustedOrRepairedAmount.add(cur.getAmount().negate());
          }
          return (positiveAdjustedOrRepairedAmount.compareTo(requestedPositiveAmountToAdjust) >= 0) ? BigDecimal.ZERO : requestedPositiveAmountToAdjust.subtract(positiveAdjustedOrRepairedAmount);
      }

      private InvoiceItemModelDao getInvoiceItemForId(final InvoiceModelDao invoice, final UUID invoiceItemId) throws InvoiceApiException {
          for (final InvoiceItemModelDao invoiceItem : invoice.getInvoiceItems()) {
              if (invoiceItem.getId().equals(invoiceItemId)) {
                  return invoiceItem;
              }
          }
          throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_NOT_FOUND, invoiceItemId);
      }

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
  Line Numbers: 409 to 435
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
      @Override
      public InvoiceItem insertInvoiceItemAdjustment(final UUID accountId,
                                                     final UUID invoiceId,
                                                     final UUID invoiceItemId,
                                                     final LocalDate effectiveDate,
                                                     @Nullable final BigDecimal amount,
                                                     @Nullable final Currency currency,
                                                     final String description,
                                                     final String itemDetails,
                                                     final Iterable<PluginProperty> originalProperties,
                                                     final CallContext context) throws InvoiceApiException {
          if (amount != null && amount.compareTo(BigDecimal.ZERO) <= 0) {
              throw new InvoiceApiException(ErrorCode.INVOICE_ITEM_ADJUSTMENT_AMOUNT_SHOULD_BE_POSITIVE, amount);
          }

          final WithAccountLock withAccountLock = new WithAccountLock() {
              @Override
              public Iterable<DefaultInvoice> prepareInvoices() throws InvoiceApiException {

                  final DefaultInvoice invoice = getInvoiceInternal(invoiceId, context);
                  if (InvoiceStatus.VOID == invoice.getStatus()) {
                      throw new InvoiceApiException(ErrorCode.INVOICE_VOID_UPDATED, invoice.getId());
                  }

                  // Check the specified currency matches the one of the existing invoice
                  if (currency != null && invoice.getCurrency() != currency) {
                      throw new InvoiceApiException(ErrorCode.CURRENCY_INVALID, currency, invoice.getCurrency());
  ```
  Applies to: Item adjustments and payment refunds.
  Confidence score: High

- [Kill-BR-0095] - Adjustment/credit/external-charge items blocked on non-DRAFT invoices
  Description: `DefaultInvoiceUserApi.insertItems` (used by `insertExternalCharges`/`insertCredits`) throws `INVOICE_ALREADY_COMMITTED` when targeting a COMMITTED invoice, and rejects VOID invoices outright (`IllegalStateException`, TODO GH #1501); only DRAFT (or a brand-new) invoice may receive these items. `insertInvoiceItemAdjustment` separately blocks any insertion on a VOID invoice (`INVOICE_VOID_UPDATED`).
  Line Numbers: 588 to 604
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
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

                              newAndExistingInvoices.put(invoiceIdForItem, existingInvoiceForExternalCharge);
                          }
                          curInvoiceForItem = newAndExistingInvoices.get(invoiceIdForItem);
  ```
  Applies to: External charges, credits, and item adjustments insertion.
  Confidence score: High

- [Kill-BR-0096] - Credit items are stored as negated amounts
  Description: `DefaultInvoiceUserApi.insertItems` builds `CreditAdjInvoiceItem` with `inputItem.getAmount().negate()` — credits are always persisted as negative invoice-item amounts, and `insertCredits` calls `negateCreditItems` on the returned items to present them back as positive to the caller.
  Line Numbers: 633 to 647
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
                          case CREDIT_ADJ:

                              newInvoiceItem = new CreditAdjInvoiceItem(UUIDs.randomUUID(),
                                                                        context.getCreatedDate(),
                                                                        curInvoiceForItem.getId(),
                                                                        accountId,
                                                                        effectiveDate,
                                                                        inputItem.getDescription(),
                                                                        // Note! The amount is negated here!
                                                                        inputItem.getAmount().negate(),
                                                                        inputItem.getRate(),
                                                                        inputItem.getCurrency(),
                                                                        inputItem.getQuantity(),
                                                                        inputItem.getItemDetails());
                              break;
  ```
  Applies to: CREDIT_ADJ item generation.
  Confidence score: High

- [Kill-BR-0097] - External charge / credit amount and currency validation
  Description: `insertItems` rejects negative amounts for EXTERNAL_CHARGE (`EXTERNAL_CHARGE_AMOUNT_INVALID`) and CREDIT_ADJ (`CREDIT_AMOUNT_INVALID`), and rejects any item whose currency doesn't match the account currency (`CURRENCY_INVALID`).
  Line Numbers: 562 to 572
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
                      if (inputItem.getAmount() == null || inputItem.getAmount().compareTo(BigDecimal.ZERO) < 0) {
                          if (itemType == InvoiceItemType.EXTERNAL_CHARGE) {
                              throw new InvoiceApiException(ErrorCode.EXTERNAL_CHARGE_AMOUNT_INVALID, inputItem.getAmount());
                          } else if (itemType == InvoiceItemType.CREDIT_ADJ) {
                              throw new InvoiceApiException(ErrorCode.CREDIT_AMOUNT_INVALID, inputItem.getAmount());
                          }
                      }

                      if (inputItem.getCurrency() != null && !inputItem.getCurrency().equals(accountCurrency)) {
                          throw new InvoiceApiException(ErrorCode.CURRENCY_INVALID, inputItem.getCurrency(), accountCurrency);
                      }
  ```
  Applies to: External charge and credit item creation.
  Confidence score: High

- [Kill-BR-0098] - CBA credit generation on negative invoice balance
  Description: `CBADao.computeCBAComplexity` (lines 63-97): if a fully up-to-date invoice's balance is negative, a positive `CreditBalanceAdjInvoiceItem` (CBA_ADJ) is generated for the (negated) shortfall — this is how "the account overpaid/was over-credited" turns into usable account credit.
  Line Numbers: 63 to 97
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  Source Code:
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
  Applies to: Invoice finalization / payment-refund / chargeback flows that recompute balance.
  Confidence score: High

- [Kill-BR-0099] - CBA credit consumption on positive balance of a committed, non-written-off invoice with no pending payment
  Description: Same method: if balance is positive AND invoice status is COMMITTED AND not written off AND no `PENDING` payment attempt exists, existing account CBA (up to the invoice balance) is applied as a negative CBA_ADJ item to offset what's owed.
  Line Numbers: 63 to 97
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  Source Code:
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
  Applies to: Automatic application of existing account credit to new/updated invoices.
  Confidence score: High

- [Kill-BR-0100] - CBA distributed oldest-invoice-first across unpaid invoices
  Description: `CBADao.useExistingCBAFromTransaction` sorts unpaid invoices by `InvoiceModelDao::getInvoiceDate` ascending and applies remaining account CBA to each in turn until either CBA or unpaid invoices are exhausted.
  Line Numbers: 186 to 218
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  Source Code:
  ```java
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
  Applies to: `consumeExstingCBAOnAccountWithUnpaidInvoices`, and CBA rebalancing after status changes/refunds/chargebacks.
  Confidence score: High

- [Kill-BR-0101] - CBA deletion/reclaim rules distinguish credit consumption vs. credit generation, and block deleting system-generated credit
  Description: `DefaultInvoiceDao.deleteCBA` (lines 1173-1234): a negative CBA item ("credit consumption") is simply zeroed out. A positive CBA item ("credit generation") can only be deleted if it is linked to an existing CREDIT_ADJ item on the same invoice (a manually-issued credit); if the account has since spent some of that credit elsewhere, the shortfall is reclaimed from other invoices' consumed CBA (`reclaimCreditFromTransaction`) before the deletion proceeds. If no matching CREDIT_ADJ exists (i.e. the credit was system-generated, e.g. from a repair invoice), the whole deletion is rejected with `INVOICE_CBA_DELETED`. Also gated: `deleteCBA` overall requires the parent invoice be COMMITTED and not migrated.
  Line Numbers: 1173 to 1234
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
  ```java
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
              // renamed to 'invId' because: Variable 'invoiceId' is already defined in the scope
              for (final UUID invId : invoiceIds) {
                  notifyBusOfInvoiceAdjustment(entitySqlDaoWrapperFactory, invId, accountId, context.getUserToken(), context);
              }
              return null;
          });
      }
  ```
  Applies to: `deleteCBA` API — removal of previously generated/consumed account credit.
  Confidence score: High

- [Kill-BR-0102] - Written-off and zero-parent-balance child invoices excluded from account balance
  Description: `DefaultInvoiceDao.getAccountBalance` skips DRAFT/VOID invoices, and treats an invoice's balance contribution as zero if it is written off, or if it's a child invoice whose parent invoice is written off/DRAFT/VOID/zero-balance — while still including its CBA in the running total. This implements the WRITTEN_OFF and parent-child consolidation balance rules together.
  Line Numbers: 729 to 760
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
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
  Applies to: Account balance computation across parent/child accounts and written-off invoices.
  Confidence score: High

- [Kill-BR-0103] - Child invoice balance derived from parent PARENT_SUMMARY allocation when parent has zero raw balance
  Description: `CBADao.getInvoiceBalance`/`isParentExistAndRawBalanceIsZero`: when a child invoice's parent has already been fully paid/reduced to zero raw balance, the child's effective balance is recomputed as its own charged amount minus what the parent's PARENT_SUMMARY item allocated to that child account, rather than using the child invoice's own raw balance directly.
  Line Numbers: 99 to 124
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/CBADao.java
  Source Code:
  ```java
      @VisibleForTesting
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
  Applies to: Parent-child account invoice consolidation, CBA computation on child invoices.
  Confidence score: High

- [Kill-BR-0104] - Child-to-parent credit transfer requires positive child CBA and creates a linked charge/credit pair
  Description: `DefaultInvoiceUserApi.transferChildCreditToParent` requires the child account to have a parent (`ACCOUNT_DOES_NOT_HAVE_PARENT_ACCOUNT`) and positive CBA (`CHILD_ACCOUNT_MISSING_CREDIT`); `DefaultInvoiceDao.transferChildCreditToParent` then creates a COMMITTED external-charge invoice on the child account for the full CBA amount and a COMMITTED credit invoice on the parent account for the same amount, effectively migrating the credit.
  Line Numbers: 709 to 731
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
      @Override
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

      }
  ```
  Line Numbers: 1499 to 1592
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
  ```java
      public void transferChildCreditToParent(final Account childAccount, final InternalCallContext childAccountContext) throws InvoiceApiException {
          // Need to create an internalCallContext for parent account because it's needed to save the correct accountRecordId in Invoice tables.
          // Then it's used to load invoices by account.
          final InternalTenantContext internalTenantContext = internalCallContextFactory.createInternalTenantContext(childAccount.getParentAccountId(), childAccountContext);
          final InternalCallContext parentAccountContext = internalCallContextFactory.createInternalCallContext(internalTenantContext.getAccountRecordId(), childAccountContext);

          final List<CustomField> parentInvoiceCustomFields = getInvoiceCustomFields(parentAccountContext);
          final List<CustomField> childInvoiceCustomFields = getInvoiceCustomFields(childAccountContext);
          final List<Tag> parentInvoicesTags = getInvoicesTags(parentAccountContext);
          final List<Tag> childInvoicesTags = getInvoicesTags(childAccountContext);

          transactionalSqlDao.execute(false, entitySqlDaoWrapperFactory -> {
              final InvoiceSqlDao invoiceSqlDao = entitySqlDaoWrapperFactory.become(InvoiceSqlDao.class);
              final InvoiceItemSqlDao transInvoiceItemSqlDao = entitySqlDaoWrapperFactory.become(InvoiceItemSqlDao.class);

              // create child and parent invoices

              final DateTime childCreatedDate = childAccountContext.getCreatedDate();
              final BigDecimal accountCBA = getAccountCBA(childAccount.getId(), childAccountContext);

              // create external charge to child account
              final LocalDate childInvoiceDate = childAccountContext.toLocalDate(childAccountContext.getCreatedDate());
              final Invoice invoiceForExternalCharge = new DefaultInvoice(childAccount.getId(),
                                                                          childInvoiceDate,
                                                                          childCreatedDate.toLocalDate(),
                                                                          childAccount.getCurrency(),
                                                                          InvoiceStatus.COMMITTED);
              final String chargeDescription = "Charge to move credit from child to parent account";
              final InvoiceItem externalChargeItem = new ExternalChargeInvoiceItem(UUIDs.randomUUID(),
                                                                                   childCreatedDate,
                                                                                   invoiceForExternalCharge.getId(),
                                                                                   childAccount.getId(),
                                                                                   null,
                                                                                   chargeDescription,
                                                                                   childCreatedDate.toLocalDate(),
                                                                                   childCreatedDate.toLocalDate(),
                                                                                   accountCBA,
                                                                                   childAccount.getCurrency(),
                                                                                   null);
              invoiceForExternalCharge.addInvoiceItem(externalChargeItem);

              // create credit to parent account
              final LocalDate parentInvoiceDate = parentAccountContext.toLocalDate(parentAccountContext.getCreatedDate());
              final Invoice invoiceForCredit = new DefaultInvoice(childAccount.getParentAccountId(),
                                                                  parentInvoiceDate,
                                                                  childCreatedDate.toLocalDate(),
                                                                  childAccount.getCurrency(),
                                                                  InvoiceStatus.COMMITTED);
              final String creditDescription = "Credit migrated from child account " + childAccount.getId();
              final InvoiceItem creditItem = new CreditAdjInvoiceItem(UUIDs.randomUUID(),
                                                                      childCreatedDate,
                                                                      invoiceForCredit.getId(),
                                                                      childAccount.getParentAccountId(),
                                                                      childCreatedDate.toLocalDate(),
                                                                      creditDescription,
                                                                      // Note! The amount is negated here!
                                                                      accountCBA.negate(),
                                                                      childAccount.getCurrency(),
                                                                      null);
              invoiceForCredit.addInvoiceItem(creditItem);

              // save invoices and invoice items
              final InvoiceModelDao childInvoice = new InvoiceModelDao(invoiceForExternalCharge);
              createAndRefresh(invoiceSqlDao, childInvoice, childAccountContext);
              final InvoiceItemModelDao childExternalChargeItem = new InvoiceItemModelDao(externalChargeItem);
              createInvoiceItemFromTransaction(transInvoiceItemSqlDao, childExternalChargeItem, childAccountContext);
              // Keep invoice up-to-date for CBA below
              childInvoice.addInvoiceItem(childExternalChargeItem);

              final InvoiceModelDao parentInvoice = new InvoiceModelDao(invoiceForCredit);
              createAndRefresh(invoiceSqlDao, parentInvoice, parentAccountContext);
              final InvoiceItemModelDao parentCreditItem = new InvoiceItemModelDao(creditItem);
              createInvoiceItemFromTransaction(transInvoiceItemSqlDao, parentCreditItem, parentAccountContext);
              // Keep invoice up-to-date for CBA below
              parentInvoice.addInvoiceItem(parentCreditItem);

              // Create Mapping relation
              final InvoiceParentChildrenSqlDao transactional = entitySqlDaoWrapperFactory.become(InvoiceParentChildrenSqlDao.class);
              final InvoiceParentChildModelDao invoiceRelation = new InvoiceParentChildModelDao(parentInvoice.getId(), childInvoice.getId(), childInvoice.getAccountId());
              createAndRefresh(transactional, invoiceRelation, parentAccountContext);

              // Add child CBA complexity and notify bus on child invoice creation
              final CBALogicWrapper childCbaWrapper = new CBALogicWrapper(childAccount.getId(), childInvoiceCustomFields, childInvoicesTags, childAccountContext, entitySqlDaoWrapperFactory);
              childCbaWrapper.runCBALogicWithNotificationEvents(Collections.emptySet(), Set.of(childInvoice.getId()), List.of(childInvoice));
              notifyBusOfInvoiceCreation(entitySqlDaoWrapperFactory, childInvoice, childAccountContext);

              // Add parent CBA complexity and notify bus on child invoice creation
              final CBALogicWrapper cbaWrapper = new CBALogicWrapper(childAccount.getParentAccountId(), parentInvoiceCustomFields, parentInvoicesTags, parentAccountContext, entitySqlDaoWrapperFactory);
              cbaWrapper.runCBALogicWithNotificationEvents(Collections.emptySet(), Set.of(parentInvoice.getId()), List.of(parentInvoice));
              notifyBusOfInvoiceCreation(entitySqlDaoWrapperFactory, parentInvoice, parentAccountContext);

              return null;
          });
      }
  ```
  Applies to: Parent-child account credit consolidation.
  Confidence score: High

- [Kill-BR-0105] - Invoice numbering is plugin/custom-field driven, not a native sequence
  Description: `InvoiceDispatcher.INVOICE_SEQUENCE_NUMBER`/`InvoiceApiHelper.java` read an `INVOICE_SEQUENCE_NUMBER` plugin property (typically supplied by an invoice-numbering plugin) and persist it as a custom field on the invoice; `DefaultInvoiceDao.getByNumber` looks invoices up via this custom field rather than a DB auto-increment column — invoice numbering is optional/plugin-supplied, not core-enforced.
  Line Numbers: 785 to 803
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  Source Code:
  ```java
              // We should probably update InvoiceGroupingResult one day to allow more control to the caller
              final LinkedList<Object> invoiceSequenceNumbers = new LinkedList<>();
              StreamSupport.stream(pluginProperties.spliterator(), false)
                           .filter(pp -> INVOICE_SEQUENCE_NUMBER.equals(pp.getKey()))
                           .forEach(pp -> invoiceSequenceNumbers.add(pp.getValue()));

              // Transformation to Invoice -> InvoiceModelDao
              final List<InvoiceModelDao> invoicesModelDao = new ArrayList<>();
              for (final Invoice cur : splitInvoices) {
                  final InvoiceModelDao invoiceModelDao = new InvoiceModelDao(cur);
                  final List<InvoiceItemModelDao> invoiceItemModelDaos = transformToInvoiceModelDao(cur.getInvoiceItems());
                  invoiceModelDao.addInvoiceItems(invoiceItemModelDaos);
                  invoicesModelDao.add(invoiceModelDao);

                  final Object invoiceSequenceNumber = invoiceSequenceNumbers.isEmpty() ? null : invoiceSequenceNumbers.removeFirst();
                  if (invoiceSequenceNumber != null && !Strings.isNullOrEmpty(invoiceSequenceNumber.toString())) {
                      invoiceModelDao.setInvoiceNumber(Integer.valueOf(invoiceSequenceNumber.toString()));
                  }
              }
  ```
  Line Numbers: 123 to 129
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/InvoiceApiHelper.java
  Source Code:
  ```java
                      final PluginProperty invoiceSequenceNumber = StreamSupport.stream(pluginProperties.spliterator(), false)
                                                                                .filter(pp -> INVOICE_SEQUENCE_NUMBER.equals(pp.getKey()))
                                                                                .findFirst()
                                                                                .orElse(null);
                      if (invoiceSequenceNumber != null && !Strings.isNullOrEmpty(invoiceSequenceNumber.getValue().toString())) {
                          invoiceModelDao.setInvoiceNumber(Integer.valueOf(invoiceSequenceNumber.getValue().toString()));
                      }
  ```
  Line Numbers: 337 to 371
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
  ```java
      @Override
      public InvoiceModelDao getByNumber(final Integer number, final Boolean includeInvoiceChildren, final InternalTenantContext context) throws InvoiceApiException {
          if (number == null) {
              throw new InvoiceApiException(ErrorCode.INVOICE_INVALID_NUMBER, "(null)");
          }

          // Note: the context may not contain the account record id at this point
          final CustomField invoiceNumberCustomField = customFieldInternalApi.searchUniqueCustomField(INVOICE_SEQUENCE_NUMBER, String.valueOf(number), ObjectType.INVOICE, context);

          return transactionalSqlDao.execute(true, InvoiceApiException.class, entitySqlDaoWrapperFactory -> {
              final InvoiceSqlDao invoiceDao = entitySqlDaoWrapperFactory.become(InvoiceSqlDao.class);

              final InvoiceModelDao invoice;
              if (invoiceNumberCustomField != null) {
                  invoice = invoiceDao.getById(invoiceNumberCustomField.getObjectId().toString(), context); // TODO context is missing the account record id?
              } else {
                  invoice = invoiceDao.getByRecordId(number.longValue(), context);
              }
              if (invoice == null) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_NUMBER_NOT_FOUND, number.longValue());
              }

              // The context may not contain the account record id at this point - we couldn't do it in the API above
              // as we couldn't get access to the invoice object until now.
              final InternalTenantContext contextWithAccountRecordId = internalCallContextFactory.createInternalTenantContext(invoice.getAccountId(), context);
              final List<CustomField> invoiceCustomFields = getInvoiceCustomFields(invoice.getId(), contextWithAccountRecordId);
              final List<Tag> invoicesTags = getInvoiceTags(invoice.getId(), contextWithAccountRecordId);
              if (includeInvoiceChildren) {
                  invoiceDaoHelper.populateChildren(invoice, invoiceCustomFields, invoicesTags, false, entitySqlDaoWrapperFactory, contextWithAccountRecordId);
              } else {
                  invoiceDaoHelper.populateInvoiceModelDao(invoice, invoiceCustomFields, invoicesTags);
              }
              return invoice;
          });
      }
  ```
  Applies to: Human-readable invoice number assignment/lookup.
  Confidence score: High

- [Kill-BR-0106] - Migration invoices bypass the normal generation/repair pipeline
  Description: `DefaultInvoiceUserApi.createMigrationInvoice` builds an `InvoiceModelDao` with `isMigrated=true` directly from supplied items and calls `dao.createInvoices` directly, skipping billing-event-driven generation entirely; `DefaultInvoiceDao.deleteCBA` explicitly refuses to operate on migrated invoices (`invoice.isMigrated()` check, line 1185).
  Line Numbers: 502 to 532
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
      @Override
      public UUID createMigrationInvoice(final UUID accountId, final LocalDate targetDate, final Iterable<InvoiceItem> items, final CallContext context) {
          final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(accountId, context);
          final LocalDate invoiceDate = internalCallContext.toLocalDate(internalCallContext.getCreatedDate());
          final InvoiceModelDao migrationInvoice = new InvoiceModelDao(accountId, invoiceDate, targetDate, items.iterator().next().getCurrency(), true);
          final List<InvoiceItemModelDao> itemModelDaos = Iterables.toStream(items)
                  .map(input -> new InvoiceItemModelDao(internalCallContext.getCreatedDate(),
                                                        input.getInvoiceItemType(),
                                                        migrationInvoice.getId(),
                                                        accountId,
                                                        input.getBundleId(),
                                                        input.getSubscriptionId(),
                                                        input.getDescription(),
                                                        input.getProductName(),
                                                        input.getPlanName(),
                                                        input.getPhaseName(),
                                                        input.getUsageName(),
                                                        input.getCatalogEffectiveDate(),
                                                        input.getStartDate(),
                                                        input.getEndDate(),
                                                        input.getAmount(),
                                                        input.getRate(),
                                                        input.getCurrency(),
                                                        input.getLinkedItemId()))
                  .collect(Collectors.toUnmodifiableList());

          migrationInvoice.addInvoiceItems(itemModelDaos);

          dao.createInvoices(List.of(migrationInvoice), null, Collections.emptySet(), null, null, false, internalCallContext);
          return migrationInvoice.getId();
      }
  ```
  Line Numbers: 1183 to 1188
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/dao/DefaultInvoiceDao.java
  Source Code:
  ```java
              if (invoice == null ||
                  !invoice.getAccountId().equals(accountId) ||
                  invoice.isMigrated() ||
                  invoice.getStatus() != InvoiceStatus.COMMITTED) {
                  throw new InvoiceApiException(ErrorCode.INVOICE_NOT_FOUND, invoiceId);
              }
  ```
  Applies to: Bulk-imported/legacy invoice creation via `createMigrationInvoice`.
  Confidence score: High

- [Kill-BR-0107] - Charged-through-date (CTD) updated only on invoice commit
  Description: `DefaultInvoiceUserApi.commitInvoice` updates invoice status to COMMITTED before calling `dispatcher.setChargedThroughDates`, with an explicit comment: "we typically don't update CTD for non committed invoices" — CTD (which gates future billing eligibility per subscription) is only advanced once an invoice is finalized, not for DRAFT invoices.
  Line Numbers: 688 to 708
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/api/user/DefaultInvoiceUserApi.java
  Source Code:
  ```java
      @Override
      public void commitInvoice(final UUID invoiceId, final CallContext context) throws InvoiceApiException {
          final WithAccountLock withAccountLock = new WithAccountLock() {
              @Override
              public Iterable<DefaultInvoice> prepareInvoices() throws InvoiceApiException {
                  final InternalCallContext internalCallContext = internalCallContextFactory.createInternalCallContext(invoiceId, ObjectType.INVOICE, context);
                  // Update invoice status first prior we update CTD as we typically don't update CTD for non committed invoices.
                  dao.changeInvoiceStatus(invoiceId, InvoiceStatus.COMMITTED, internalCallContext);
                  final DefaultInvoice invoice = getInvoiceInternal(invoiceId, context);
                  dispatcher.setChargedThroughDates(invoice, internalCallContext);
                  return List.of(invoice);
              }
          };

          final UUID accountId = getInvoiceInternal(invoiceId, context).getAccountId();

          final LinkedList<PluginProperty> properties = new LinkedList<PluginProperty>();
          properties.add(new PluginProperty(INVOICE_OPERATION, "commit", false));

          invoiceApiHelper.dispatchToInvoicePluginsAndInsertItems(accountId, false, withAccountLock, properties, false, context);
      }
  ```
  Applies to: Subscription charged-through-date advancement tied to invoice commitment.
  Confidence score: High

- [Kill-BR-0108] - Account parking on unexpected/inconsistent invoicing state
  Description: `InvoiceDispatcher.processAccountInternal` calls `parkAccount` (which tags the account via `ParkedAccountsManager`) whenever an `UNEXPECTED_ERROR`-coded `InvoiceApiException` occurs during non-dry-run invoicing, or on other catalog/account/subscription exceptions when `InvoiceConfig.isParkAccountsOnAllExceptions()` is enabled — halting further automatic invoicing for that account until manually/automatically unparked (`unparkAccount` on next successful run).
  Line Numbers: 443 to 484
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/InvoiceDispatcher.java
  Source Code:
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
  ```
  Applies to: Account-level invoicing-integrity safeguard against corrupt/inconsistent invoice state.
  Confidence score: High

- [Kill-BR-0109] - Sanity safety-bound duplicate/overlap detection during item generation
  Description: `FixedAndRecurringInvoiceItemGenerator.safetyBounds`/`validateSafetyBoundsWithFixedInvoiceItem`/`validateSafetyBoundsWithRecurringInvoiceItem` (gated by `InvoiceConfig.isSanitySafetyBoundEnabled`/`getMaxDailyNumberOfItemsSafetyBound`) detect and reject generation of duplicate or overlapping FIXED/RECURRING items for the same subscription/date, referencing several historical bug numbers (GH #664, #993, #1241, #1907) as the motivating defects.
  Line Numbers: 477 to 548
  Source Code File Name: invoice/src/main/java/org/killbill/billing/invoice/generator/FixedAndRecurringInvoiceItemGenerator.java
  Source Code:
  ```java
      void safetyBounds(final Iterable<InvoiceItem> resultingItems, final MultiValueMap<UUID, LocalDate> createdItemsPerDayPerSubscription, final InternalCallContext internalCallContext) throws InvoiceApiException {
          // Trigger an exception if we detect the creation of similar items for a given subscription
          // See https://github.com/killbill/killbill/issues/664
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
  Applies to: Fixed and recurring invoice item generation, as a correctness backstop.
  Confidence score: High

- [Kill-BR-0110] - Control-plugin chain with abort-on-first-abort
  Description: When executing the configured payment control plugins before a payment, ControlPluginRunner.executePluginPriorCalls loops over paymentControlPluginNames in order, feeding each plugin's adjusted amount/currency/paymentMethodId/properties into the next plugin's input. If any plugin's PriorPaymentControlResult.isAborted() is true, the loop immediately throws PaymentControlApiAbortException(controlPluginName), aborting the whole payment attempt before it reaches the gateway. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:94-155.
  Line Numbers: 94 to 155
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java
  Source Code:
  ```java
          for (final String controlPluginName : paymentControlPluginNames) {

              final PaymentControlContext inputPaymentControlContext = new DefaultPaymentControlContext(accountId,
                                                                                                        inputPaymentMethodId,
                                                                                                        controlPluginName,
                                                                                                        paymentAttemptId,
                                                                                                        paymentId,
                                                                                                        paymentExternalKey,
                                                                                                        paymentTransactionId,
                                                                                                        paymentTransactionExternalKey,
                                                                                                        paymentApiType,
                                                                                                        transactionType,
                                                                                                        hppType,
                                                                                                        inputAmount,
                                                                                                        inputCurrency,
                                                                                                        processedAmount,
                                                                                                        processedCurrency,
                                                                                                        isApiPayment,
                                                                                                        callContext);

              final PaymentControlPluginApi plugin = paymentControlPluginRegistry.getServiceForName(controlPluginName);
              if (plugin == null) {
                  // First call to plugin, we log warn, if plugin is not registered
                  log.warn("Skipping unknown payment control plugin {} when fetching results", controlPluginName);
                  continue;
              }
              log.debug("Calling priorCall of plugin {}", controlPluginName);
              prevResult = plugin.priorCall(inputPaymentControlContext, inputPluginProperties);
              log.debug("Successful executed priorCall of plugin {}", controlPluginName);
              if (prevResult == null) {
                  // Nothing returned by the plugin
                  continue;
              }

              if (prevResult.getAdjustedPaymentMethodId() != null) {
                  // We only allow setting the paymentMethodId but disallow overwriting an existing paymentMethodId for a given Payment - See #1097
                  // unless the property isAllowedToOverwritePaymentMethodId was explicitly configured to allow this
                  if (!paymentConfig.isAllowedToOverwritePaymentMethodId() && paymentMethodId != null) {
                      throw new PaymentControlApiException(String.format("Not allowed to overwrite paymentMethodId '%s' for payment '%s'",
                                                                         paymentMethodId, paymentId));
                  }
                  inputPaymentMethodId = prevResult.getAdjustedPaymentMethodId();
              }
              if (prevResult.getAdjustedPluginName() != null) {
                  inputPaymentMethodName = prevResult.getAdjustedPluginName();
              }
              if (prevResult.getAdjustedAmount() != null) {
                  inputAmount = prevResult.getAdjustedAmount();
              }
              if (prevResult.getAdjustedCurrency() != null) {
                  inputCurrency = prevResult.getAdjustedCurrency();
              }
              if (prevResult.getAdjustedPluginProperties() != null) {
                  inputPluginProperties = prevResult.getAdjustedPluginProperties();
              }
              if (prevResult.isAborted()) {
                  throw new PaymentControlApiAbortException(controlPluginName);
              }
          }
          // Rebuild latest result to include inputPluginProperties
          prevResult = new DefaultPriorPaymentControlResult(prevResult != null && prevResult.isAborted(), inputPaymentMethodId, inputPaymentMethodName, inputAmount, inputCurrency, inputPluginProperties);
          return prevResult;
  ```
  Applies to: All payment-control-plugin-mediated transactions (purchase/refund/credit/chargeback/authorize/capture/void)
  Confidence score: High

- [Kill-BR-0111] - Earliest-retry-date-wins across failure-handling control plugins
  Description: ControlPluginRunner.executePluginOnFailureCalls calls onFailureCall on every configured control plugin and collects each plugin's suggested nextRetryDate; the final retry date chosen is the earliest (minimum) of all non-null suggestions. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:263-294.
  Line Numbers: 263 to 294
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java
  Source Code:
  ```java
          DateTime candidate = null;
          Iterable<PluginProperty> inputPluginProperties = pluginProperties;

          for (final String controlPluginName : paymentControlPluginNames) {
              final PaymentControlPluginApi plugin = paymentControlPluginRegistry.getServiceForName(controlPluginName);
              if (plugin != null) {
                  try {
                      log.debug("Calling onSuccessCall of plugin {}", controlPluginName);
                      final OnFailurePaymentControlResult result = plugin.onFailureCall(inputPaymentControlContext, inputPluginProperties);
                      log.debug("Successful executed onSuccessCall of plugin {}", controlPluginName);
                      if (result == null) {
                          // Nothing returned by the plugin
                          continue;
                      }

                      if (candidate == null) {
                          candidate = result.getNextRetryDate();
                      } else if (result.getNextRetryDate() != null) {
                          candidate = candidate.compareTo(result.getNextRetryDate()) > 0 ? result.getNextRetryDate() : candidate;
                      }

                      if (result.getAdjustedPluginProperties() != null) {
                          inputPluginProperties = result.getAdjustedPluginProperties();
                      }

                  } catch (final PaymentControlApiException e) {
                      log.warn("Error during onFailureCall for plugin='{}', paymentExternalKey='{}'", controlPluginName, inputPaymentControlContext.getPaymentExternalKey(), e);
                      return new DefaultFailureCallResult(candidate, inputPluginProperties);
                  }
              }
          }
          return new DefaultFailureCallResult(candidate, inputPluginProperties);
  ```
  Applies to: Retry scheduling for failed payment transactions with multiple control plugins configured
  Confidence score: High

- [Kill-BR-0112] - Invoice-driven purchase gated on COMMITTED invoice status
  Description: InvoicePaymentControlPluginApi.getPluginPurchaseResult aborts the payment (PriorPaymentControlResult with isAborted=true) if the target invoice's status is not COMMITTED (e.g. DRAFT invoices can never trigger a real charge). Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:341-351.
  Line Numbers: 341 to 351
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
      private PriorPaymentControlResult getPluginPurchaseResult(final PaymentControlContext paymentControlPluginContext, final Iterable<PluginProperty> pluginProperties, final InternalCallContext internalContext) throws PaymentControlApiException {
          try {
              final UUID invoiceId = getInvoiceId(pluginProperties);

              // Optimize case where we have a Draft invoice to avoid pulling the whole thing.
              final InvoiceStatus status = invoiceApi.getInvoiceStatus(invoiceId, internalContext);
              if (!InvoiceStatus.COMMITTED.equals(status)) {
                  // abort payment if the invoice status is not COMMITTED
                  log.info("Aborting payment: invoiceId='{}' is NOT COMMITTED", invoiceId);
                  return new DefaultPriorPaymentControlResult(true);
              }
  ```
  Applies to: Automatic/API purchase payments tied to an invoice
  Confidence score: High

- [Kill-BR-0113] - Child-account payments delegated to parent are blocked
  Description: In the same prior-call, if the account is a child account with isPaymentDelegatedToParent() set, or the invoice already has a parentAccountId, the payment is aborted — child accounts under parent payment delegation never get their own gateway charge. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:354-360.
  Line Numbers: 354 to 360
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
              // Get account and check if it is child and payment is delegated to parent => abort
              final AccountData accountData = accountApi.getAccountById(invoice.getAccountId(), internalContext);
              if (((accountData != null) && (accountData.getParentAccountId() != null) && accountData.isPaymentDelegatedToParent()) || // Valid when we initially create the child invoice (even if parent invoice does not exist yet)
                  (invoice.getParentAccountId() != null))  { // Valid after we have unparented the child
                  log.info("Aborting payment: invoiceId='{}' is delegated to parent", invoice.getId());
                  return new DefaultPriorPaymentControlResult(true);
              }
  ```
  Applies to: Parent/child account billing hierarchies
  Confidence score: High

- [Kill-BR-0114] - Payment amount is capped at, and defaults to, the invoice balance
  Description: validateAndComputePaymentAmount returns BigDecimal.ZERO if invoice balance <= 0; for API-initiated payments it throws (aborts) if the requested amount exceeds the invoice balance; otherwise the payment amount is min(invoice balance, requested amount) — the invoice balance is authoritative. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:710-728.
  Line Numbers: 710 to 728
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
  Applies to: Purchase payments against an invoice
  Confidence score: High

- [Kill-BR-0115] - Zero/negative computed amount aborts unless "allow empty invoice" is configured
  Description: If the computed requested payment amount is <= 0 (invoice already paid), the payment is aborted with a log "invoice has already been paid" — unless paymentConfig.allowEmptyInvoice() is true, in which case a zero-amount payment attempt is allowed through instead. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:362-374.
  Line Numbers: 362 to 374
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
              // Is remaining amount > 0?
              final BigDecimal requestedAmount = validateAndComputePaymentAmount(invoice, paymentControlPluginContext.getAmount(), paymentControlPluginContext.isApiPayment());
              if (requestedAmount.compareTo(BigDecimal.ZERO) <= 0) {
                  if (paymentConfig.allowEmptyInvoice()) {
                      log.info("Not aborting payment for zero amount invoice: invoiceId='{}' since allowEmptyInvoice is set", invoice.getId());
                      return new DefaultPriorPaymentControlResult(false, requestedAmount);
                  }
                  else {
                      log.info("Aborting payment: invoiceId='{}' has already been paid", invoice.getId());
                      return new DefaultPriorPaymentControlResult(true);

                  }
              }
  ```
  Applies to: Purchase payments against a fully/over paid invoice
  Confidence score: High

- [Kill-BR-0116] - AUTO_PAY_OFF suppresses only system-triggered (non-API) auto-charges
  Description: insert_AUTO_PAY_OFF_ifRequired only checks/aborts on the AUTO_PAY_OFF control tag when paymentControlContext.isApiPayment() is false; explicit API-initiated payments are never blocked by AUTO_PAY_OFF. When aborted, a PluginAutoPayOffModelDao row is persisted recording the deferred attempt. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:730-740.
  Line Numbers: 730 to 740
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
  ```
  Applies to: Automatic (invoice-creation-triggered) purchase payments
  Confidence score: High

- [Kill-BR-0117] - Removing AUTO_PAY_OFF replays all deferred payment attempts
  Description: process_AUTO_PAY_OFF_removal reads every PluginAutoPayOffModelDao entry recorded while the account was on AUTO_PAY_OFF and immediately schedules a retry (via RetryServiceScheduler) for each deferred attempt, then clears the entries — i.e. turning auto-pay back on replays the backlog rather than waiting for the next invoice. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:312-319.
  Line Numbers: 312 to 319
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
      public void process_AUTO_PAY_OFF_removal(final UUID accountId, final InternalCallContext internalCallContext) {
          final List<PluginAutoPayOffModelDao> entries = controlDao.getAutoPayOffEntry(accountId);
          for (final PluginAutoPayOffModelDao cur : entries) {
              // TODO In theory we should pass not only PLUGIN_NAME, but also all the plugin list associated which the original call
              retryServiceScheduler.scheduleRetry(ObjectType.ACCOUNT, accountId, cur.getAttemptId(), internalCallContext.getTenantRecordId(), List.of(PLUGIN_NAME), internalCallContext.getCreatedDate());
          }
          controlDao.removeAutoPayOffEntry(accountId);
      }
  ```
  Applies to: Account AUTO_PAY_OFF tag removal
  Confidence score: High

- [Kill-BR-0118] - Two-phase-commit attempt row to prevent double payment
  Description: Before calling the gateway, getPluginPurchaseResult inserts an INIT-status invoice payment attempt row (invoiceApi.recordPaymentAttemptInit) so that if the process crashes after the charge succeeds but before onSuccessCall runs, checkForIncompleteInvoicePaymentAndRepair can later detect the dangling ATTEMPT row and repair it from the matching successful PaymentTransactionModelDao rather than re-charging. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:410-424, 665-708.
  Line Numbers: 410 to 424
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
              //
              // Insert attempt row with a success = false status to implement a two-phase commit strategy and guard against scenario where payment would go through
              // but onSuccessCall callback never gets called (leaving the place for a double payment if user retries the operation)
              //
              invoiceApi.recordPaymentAttemptInit(invoice.getId(),
                                                  Objects.requireNonNullElse(paymentControlPluginContext.getAmount(), BigDecimal.ZERO),
                                                  paymentControlPluginContext.getCurrency(),
                                                  paymentControlPluginContext.getCurrency(),
                                                  // Likely to be null, but we don't care as we use the transactionExternalKey
                                                  // to match the operation in the checkForIncompleteInvoicePaymentAndRepair logic below
                                                  paymentControlPluginContext.getPaymentId(),
                                                  paymentControlPluginContext.getAttemptPaymentId(),
                                                  paymentControlPluginContext.getTransactionExternalKey(),
                                                  paymentControlPluginContext.getCreatedDate(),
                                                  internalContext);
  ```
  Line Numbers: 665 to 708
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
      private boolean checkForIncompleteInvoicePaymentAndRepair(final Invoice invoice, final UUID paymentAttemptId, final InternalCallContext internalContext) throws InvoiceApiException {

          final List<InvoicePayment> invoicePayments = invoice.getPayments();

          // Look for ATTEMPT matching that invoiceId that are not successful and extract matching paymentTransaction
          final InvoicePayment incompleteInvoicePayment = invoicePayments
                  .stream()
                  .filter(input -> input.getType() == InvoicePaymentType.ATTEMPT && input.getStatus() != InvoicePaymentStatus.SUCCESS)
                  .findFirst().orElse(null);

          // If such (incomplete) paymentTransaction exists, verify the state of the payment transaction
          if (incompleteInvoicePayment != null) {
              final String transactionExternalKey = incompleteInvoicePayment.getPaymentCookieId();
              final List<PaymentTransactionModelDao> transactions = paymentDao.getPaymentTransactionsByExternalKey(transactionExternalKey, internalContext);
              final PaymentTransactionModelDao successfulTransaction = transactions.stream()
                      .filter(input -> {
                          //
                          // In reality this is more tricky because the matching transaction could be an UNKNOWN or PENDING (unsupported by the plugin) state
                          // In case of UNKNOWN, we don't know what to do: fixing it could result in not paying, and not fixing it could result in double payment
                          // Current code ignores it, which means we might end up in doing a double payment in that very edgy scenario, and customer would have to request a refund.
                          //
                          return input.getTransactionStatus() == TransactionStatus.SUCCESS;
                      })
                      .findFirst().orElse(null);

              if (successfulTransaction != null) {
                  log.info(String.format("Detected an incomplete invoicePayment row for invoiceId='%s' and transactionExternalKey='%s', will correct status", invoice.getId(), successfulTransaction.getTransactionExternalKey()));

                  invoiceApi.recordPaymentAttemptCompletion(invoice.getId(),
                                                            successfulTransaction.getAmount(),
                                                            successfulTransaction.getCurrency(),
                                                            successfulTransaction.getProcessedCurrency(),
                                                            successfulTransaction.getPaymentId(),
                                                            paymentAttemptId,
                                                            successfulTransaction.getTransactionExternalKey(),
                                                            successfulTransaction.getCreatedDate(),
                                                            InvoicePaymentStatus.SUCCESS,
                                                            internalContext);
                  return true;

              }
          }
          return false;
      }
  ```
  Applies to: Purchase payments driven through the invoice control plugin
  Confidence score: High

- [Kill-BR-0119] - Existing UNKNOWN transaction blocks a new purchase attempt on the same invoice
  Description: getPluginPurchaseResult scans existing invoice payments' underlying transactions; if any is in UNKNOWN status, the new purchase is aborted rather than risking a double charge while the earlier attempt's true outcome is still unresolved (left for the Janitor to resolve). Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:399-408.
  Line Numbers: 399 to 408
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
              final List<InvoicePayment> existingInvoicePayments = invoiceApi.getInvoicePaymentsByInvoice(invoiceId, internalContext);
              for (final InvoicePayment existingInvoicePayment : existingInvoicePayments) {
                  final List<PaymentTransactionModelDao> existingTransactions = paymentDao.getPaymentTransactionsByExternalKey(existingInvoicePayment.getPaymentCookieId(), internalContext);
                  for (final PaymentTransactionModelDao existingTransaction : existingTransactions) {
                      if (existingTransaction.getTransactionStatus() == TransactionStatus.UNKNOWN) {
                          log.warn("Existing paymentTransactionId='{}' for invoiceId='{}' in UNKNOWN state", existingTransaction.getId(), invoiceId);
                          return new DefaultPriorPaymentControlResult(true);
                      }
                  }
              }
  ```
  Applies to: Repeated/retried purchase attempts against the same invoice
  Confidence score: High

- [Kill-BR-0120] - Refunds are never retried
  Description: In onFailureCall, the REFUND and CREDIT branches are explicit no-ops with the comment "We don't retry REFUND" — only PURCHASE failures compute a nextRetryDate; refund/credit failures simply surface as terminal failures. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:294-297.
  Line Numbers: 294 to 297
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
              case CREDIT:
              case REFUND:
                  // We don't retry REFUND
                  break;
  ```
  Applies to: Refund and credit transaction failures
  Confidence score: High

- [Kill-BR-0121] - Refund amount capped by invoice item sums, aborted if fully consumed
  Description: computeRefundAmount computes the refundable amount either from the caller-specified amount or by summing per-invoice-item amounts (rejecting any item amount that is <=0 or exceeds the item's original amount). getPluginRefundResult aborts (isAborted=true) if the computed refundable amount is zero, and additionally throws for API-initiated calls if aborted (rather than just silently skipping). Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:438-556.
  Line Numbers: 438 to 556
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
      private PriorPaymentControlResult getPluginRefundResult(final PaymentControlContext paymentControlPluginContext, final Iterable<PluginProperty> pluginProperties, final InternalCallContext internalContext) throws PaymentControlApiException {
          final Map<UUID, BigDecimal> idWithAmount = extractIdsWithAmountFromProperties(pluginProperties);
          if ((paymentControlPluginContext.getAmount() == null || paymentControlPluginContext.getAmount().compareTo(BigDecimal.ZERO) == 0) &&
              idWithAmount.size() == 0) {
              throw new PaymentControlApiException("Abort refund call: ", new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION,
                                                                                                  String.format("Refund for payment, key = %s, aborted: requested refund amount is = %s",
                                                                                                                paymentControlPluginContext.getPaymentExternalKey(),
                                                                                                                paymentControlPluginContext.getAmount())));
          }

          final PaymentModelDao payment = paymentDao.getPayment(paymentControlPluginContext.getPaymentId(), internalContext);
          if (payment == null) {
              throw new PaymentControlApiException("Unexpected null payment");
          }
          // This will calculate the upper bound on the refund amount based on the invoice items associated with that payment.
          // Note that we are not checking that other (partial) refund occurred, but if the refund ends up being greater than what is allowed
          // the call to the gateway would fail; it would need noce to validate on our side though...
          final BigDecimal amountToBeRefunded = computeRefundAmount(payment.getId(), paymentControlPluginContext.getAmount(), idWithAmount, internalContext);
          final boolean isAborted = amountToBeRefunded.compareTo(BigDecimal.ZERO) == 0;

          if (paymentControlPluginContext.isApiPayment() && isAborted) {
              throw new PaymentControlApiException("Abort refund call: ", new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION,
                                                                                                  String.format("Refund for payment %s aborted : invoice item sum amount is %s, requested refund amount is = %s",
                                                                                                                payment.getId(),
                                                                                                                amountToBeRefunded,
                                                                                                                paymentControlPluginContext.getAmount())));
          }

          final PluginProperty prop = getPluginProperty(pluginProperties, PROP_IPCD_REFUND_WITH_ADJUSTMENTS);
          final boolean isAdjusted = prop != null && prop.getValue() != null ? Boolean.valueOf(prop.getValue().toString()) : false;
          if (isAdjusted) {
              try {
                  invoiceApi.validateInvoiceItemAdjustments(paymentControlPluginContext.getPaymentId(), idWithAmount, internalContext);
              } catch (InvoiceApiException e) {
                  throw new PaymentControlApiException(String.format("Refund for payment %s aborted", payment.getId()),
                                                       new PaymentApiException(ErrorCode.PAYMENT_PLUGIN_EXCEPTION, e.getMessage()));
              }
          }

          return new DefaultPriorPaymentControlResult(isAborted, amountToBeRefunded);
      }

      private PriorPaymentControlResult getPluginCreditResult(final PaymentControlContext paymentControlPluginContext, final Iterable<PluginProperty> pluginProperties, final InternalCallContext internalContext) throws PaymentControlApiException {
          // TODO implement
          return new DefaultPriorPaymentControlResult(false, paymentControlPluginContext.getAmount());
      }

      private Map<UUID, BigDecimal> extractIdsWithAmountFromProperties(final Iterable<PluginProperty> properties) {
          final PluginProperty prop = getPluginProperty(properties, PROP_IPCD_REFUND_IDS_WITH_AMOUNT_KEY);
          if (prop == null) {
              return Collections.emptyMap();
          }
          // The deserialization may not recreate the map we expect, i.e Map<UUID, BigDecimal>, so we convert each key/value by hand
          // See https://github.com/killbill/killbill/issues/1453
          final Map<UUID, BigDecimal> res = new HashMap<>();
          final Map m = (Map) prop.getValue();
          for (final Object k : m.keySet()) {
              UUID uuid;
              if (k instanceof String) {
                  uuid = UUID.fromString((String) k);
              } else if (k instanceof UUID) {
                  uuid = (UUID) k;
              } else {
                  throw new IllegalStateException(String.format("Failed to deserialize plugin property map for adjustments: Invalid format for UUID, type=%s", k.getClass().getName()));
              }

              final Object v = m.get(k);
              BigDecimal val;
              if (v instanceof BigDecimal) {
                  val = (BigDecimal) v;
              } else if (v instanceof String) {
                  val = new BigDecimal((String) v);
              } else if (v instanceof Integer) {
                  val = new BigDecimal(((Integer) v).toString());
              } else if (v == null) {
                  // Null is allowed to default ot item#amount
                  val = null;
              } else {
                  throw new IllegalStateException(String.format("Failed to deserialize plugin property map for adjustments: Invalid format for BigDecimal, type=%s", v.getClass().getName()));
              }
              res.put(uuid, val);
          }
          return res;
      }

      private PluginProperty getPluginProperty(final Iterable<PluginProperty> properties, final String propertyName) {
          return Iterables.toStream(properties)
                          .filter(input -> input.getKey().equals(propertyName))
                          .findFirst().orElse(null);
      }

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
  ```
  Applies to: Partial/full refunds of a payment
  Confidence score: High

- [Kill-BR-0122] - No partial chargebacks
  Description: onSuccessCall's CHARGEBACK branch checks getInvoicePaymentForChargeback and, if one already exists for the payment, treats a second chargeback as already-completed/no-op ("We don't support partial chargebacks (yet?)") instead of recording another one. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:209-214.
  Line Numbers: 209 to 214
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
                  case CHARGEBACK:
                      existingInvoicePayment = invoiceApi.getInvoicePaymentForChargeback(paymentControlContext.getPaymentId(), internalContext);
                      if (existingInvoicePayment != null) {
                          // We don't support partial chargebacks (yet?)
                          log.info("onSuccessCall was already completed for chargeback paymentId='{}'", paymentControlContext.getPaymentId());
                      } else {
  ```
  Applies to: Chargeback transactions
  Confidence score: High

- [Kill-BR-0123] - Chargeback amount/currency fallback chain
  Description: When recording a chargeback, the amount/currency used is chosen by a priority chain: prefer processedAmount/processedCurrency if processedCurrency matches the linked invoice payment's currency; else fall back to the requested amount/currency if that currency matches; else fall back to the original linked invoice payment's amount/currency entirely. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:217-228.
  Line Numbers: 217 to 228
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
  ```
  Applies to: Chargeback recording against invoice payments
  Confidence score: High

- [Kill-BR-0124] - Currency-mismatch handling for successful invoice payments
  Description: onSuccessCall's PURCHASE branch normally records processedAmount as the invoice payment amount, but if the requested currency differs from the processed currency it logs a warning and instead assumes it was "a full payment," recording the originally requested amount instead of the processed amount. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:179-198.
  Line Numbers: 179 to 198
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
                          final BigDecimal invoicePaymentAmount;
                          if (paymentControlContext.getCurrency() == paymentControlContext.getProcessedCurrency()) {
                              invoicePaymentAmount = paymentControlContext.getProcessedAmount();
                          } else {
                              log.warn("processedCurrency='{}' of invoice paymentId='{}' doesn't match invoice currency='{}', assuming it is a full payment", paymentControlContext.getProcessedCurrency(), paymentControlContext.getPaymentId(), paymentControlContext.getCurrency());
                              invoicePaymentAmount = paymentControlContext.getAmount();
                          }

                          log.debug("Notifying invoice of paymentId='{}', amount='{}', currency='{}', invoiceId='{}', invoicePaymentStatus='{}'", paymentControlContext.getPaymentId(), invoicePaymentAmount, paymentControlContext.getCurrency(), invoiceId, status);

                          invoiceApi.recordPaymentAttemptCompletion(invoiceId,
                                                                    invoicePaymentAmount,
                                                                    paymentControlContext.getCurrency(),
                                                                    paymentControlContext.getProcessedCurrency(),
                                                                    paymentControlContext.getPaymentId(),
                                                                    paymentControlContext.getAttemptPaymentId(),
                                                                    paymentControlContext.getTransactionExternalKey(),
                                                                    paymentControlContext.getCreatedDate(),
                                                                    status,
                                                                    internalContext);
  ```
  Applies to: Successful purchase transactions where the plugin's processed currency diverges from the requested currency
  Confidence score: High

- [Kill-BR-0125] - Payment-level transaction currency must be consistent across a payment's transactions
  Description: PaymentAutomatonDAOHelper.createNewPaymentTransaction rejects (PaymentApiException PAYMENT_INVALID_PARAMETER) any new transaction whose currency differs from the currency of the payment's first existing transaction — with an explicit carve-out that CHARGEBACK transactions are allowed to use a different currency. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java:105-110.
  Line Numbers: 105 to 110
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java
  Source Code:
  ```java
              if (paymentStateContext.getCurrency() != null &&
                  existingTransactions.get(0).getCurrency() != paymentStateContext.getCurrency() &&
                  !TransactionType.CHARGEBACK.equals(paymentStateContext.getTransactionType())) {
                  // Note that we allow chargebacks in a different currency
                  throw new PaymentApiException(ErrorCode.PAYMENT_INVALID_PARAMETER, "currency", " should be " + existingTransactions.get(0).getCurrency() + " to match other existing transactions");
              }
  ```
  Applies to: Multi-transaction payments (e.g. capture after authorize, multiple purchases on one payment)
  Confidence score: High

- [Kill-BR-0126] - Automatic invoice payment always uses the account's currency
  Description: When an InvoiceCreationInternalEvent triggers an automatic payment, PaymentBusEventHandler passes account.getCurrency() (not the invoice's own currency field) as the payment currency to createPurchaseForInvoicePayment. Evidence: payment/src/main/java/org/killbill/billing/payment/bus/PaymentBusEventHandler.java:101-107.
  Line Numbers: 101 to 107
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/bus/PaymentBusEventHandler.java
  Source Code:
  ```java
                      invoicePaymentInternalApi.createPurchaseForInvoicePayment(false,
                                                                                account,
                                                                                event.getInvoiceId(),
                                                                                account.getPaymentMethodId(),
                                                                                null,
                                                                                amountToBePaid,
                                                                                account.getCurrency(),
  ```
  Applies to: System-triggered (auto-pay) purchase payments on new invoices
  Confidence score: High

- [Kill-BR-0127] - API-originated payment failures are not retried by default
  Description: InvoicePaymentControlPluginApi.computeNextRetryDate returns null (no retry scheduled) whenever the failed call was API-initiated (isApiPayment=true) and the "IPCD_RETRIES" plugin property was not explicitly set to true — retries only happen automatically for system-triggered (non-API) payment attempts unless a caller opts in. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:266-272, 567-572.
  Line Numbers: 266 to 272
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
          final PluginProperty ipcdRetriesProperty = StreamSupport.stream(pluginProperties.spliterator(), false)
                                                                  .filter(p -> PROP_IPCD_RETRIES.equals(p.getKey()))
                                                                  .findFirst()
                                                                  .orElse(null);
          final boolean ipcdRetries = ipcdRetriesProperty != null ? Boolean.parseBoolean(ipcdRetriesProperty.getValue().toString()) : false;
          DateTime nextRetryDate = null;
          switch (transactionType) {
  ```
  Line Numbers: 567 to 572
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
  ```java
      private DateTime computeNextRetryDate(final String paymentExternalKey, final boolean ipcdRetries, final boolean isApiAPayment, final InternalCallContext internalContext) {

          // Don't retry call that come from API.
          if (!ipcdRetries && isApiAPayment) {
              return null;
          }
  ```
  Applies to: Purchase transaction failure handling
  Confidence score: High

- [Kill-BR-0128] - Fixed-day-list retry backoff for PAYMENT_FAILURE
  Description: getNextRetryDateForPaymentFailure schedules the next retry `paymentConfig.getPaymentFailureRetryDays()` days from now, indexed by the count of prior consecutive PAYMENT_FAILURE attempts (default "8,8,8" days = up to 3 retries, 8 days apart); once the attempt count exceeds the configured list size, no further retry is scheduled (returns null = give up). Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:592-610; defaults in util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java:34-37.
  Line Numbers: 592 to 610
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
      }
  ```
  Line Numbers: 34 to 37
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java
  Source Code:
  ```java
      List<Integer> getPaymentFailureRetryDays();

      @Config("org.killbill.payment.retry.days")
      @Default("8,8,8")
  ```
  Applies to: Gateway-declined ("hard") payment failures (e.g. card declined)
  Confidence score: High

- [Kill-BR-0129] - Exponential-backoff retry for PLUGIN_FAILURE
  Description: getNextRetryDateForPluginFailure retries up to `pluginFailureRetryMaxAttempts` times (default 8), starting at `pluginFailureInitialRetryInSec` seconds (default 300s) and multiplying the delay by `pluginFailureRetryMultiplier` (default 2) on each subsequent attempt — a classic exponential backoff for infrastructure/connectivity-type failures as opposed to gateway declines. Evidence: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:612-628; defaults in PaymentConfig.java:41-59, 82-90.
  Line Numbers: 612 to 628
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
  Line Numbers: 41 to 59
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java
  Source Code:
  ```java
      @Config("org.killbill.payment.failure.retry.start.sec")
      @Default("300")
      @Description("Specify the interval of time in seconds before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureInitialRetryInSec();

      @Config("org.killbill.payment.failure.retry.start.sec")
      @Default("300")
      @Description("Specify the interval of time in seconds before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureInitialRetryInSec(@Param("dummy") final InternalTenantContext tenantContext);

      @Config("org.killbill.payment.failure.retry.multiplier")
      @Default("2")
      @Description("Specify the multiplier to apply between in retry before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureRetryMultiplier();

      @Config("org.killbill.payment.failure.retry.multiplier")
      @Default("2")
      @Description("Specify the multiplier to apply between in retry before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureRetryMultiplier(@Param("dummy") final InternalTenantContext tenantContext);
  ```
  Line Numbers: 82 to 90
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java
  Source Code:
  ```java
      @Config("org.killbill.payment.failure.retry.max.attempts")
      @Default("8")
      @Description("Specify the max number of attempts before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureRetryMaxAttempts();

      @Config("org.killbill.payment.failure.retry.max.attempts")
      @Default("8")
      @Description("Specify the max number of attempts before retrying a payment that failed due to a plugin failure (gateway is down, transient error, ...)")
      int getPluginFailureRetryMaxAttempts(@Param("dummy") final InternalTenantContext tenantContext);
  ```
  Applies to: Plugin/gateway connectivity failures ("soft" failures) during purchase
  Confidence score: High

- [Kill-BR-0130] - UNKNOWN-status transactions are never rescheduled by the retry state machine
  Description: DefaultControlCompleted.isUnknownTransaction guards retryServiceScheduler.scheduleRetry — a payment attempt entering the RETRIED state is only actually rescheduled if its current transaction status is not UNKNOWN. The comment explains this exists to avoid infinite retries from a badly-behaved plugin; UNKNOWN transactions are left exclusively to the Janitor to resolve. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/control/DefaultControlCompleted.java:56-106.
  Line Numbers: 56 to 106
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/control/DefaultControlCompleted.java
  Source Code:
  ```java
      @Override
      public void enteringState(final State state, final OperationCallback operationCallback, final OperationResult operationResult, final LeavingStateCallback leavingStateCallback) {
          final PaymentAttemptModelDao attempt = retryablePaymentAutomatonRunner.getPaymentDao().getPaymentAttempt(paymentStateContext.getAttemptId(), paymentStateContext.getInternalCallContext());
          final UUID transactionId = paymentStateContext.getCurrentTransaction() != null ?
                                     paymentStateContext.getCurrentTransaction().getId() :
                                     null;

          logger.debug("enteringState attemptId='{}', transactionId='{}', state='{}'", attempt.getId(), transactionId, state.getName());
          // At this stage we can update the paymentAttempt state AND serialize the plugin properties. Control plugins will have had the opportunity to erase sensitive data if required.
          retryablePaymentAutomatonRunner.getPaymentDao().updatePaymentAttemptWithProperties(attempt.getId(),
                                                                                             paymentStateContext.getPaymentMethodId(),
                                                                                             transactionId,
                                                                                             state.getName(),
                                                                                             // Use the amount (either specified in the API call or overridden by the priorCall),
                                                                                             // not the processed amount, as this will drive the retries in case of failure
                                                                                             paymentStateContext.getAmount(),
                                                                                             paymentStateContext.getCurrency(),
                                                                                             getSerializedProperties(),
                                                                                             paymentStateContext.getInternalCallContext());

          if (retriedState.getName().equals(state.getName()) && !isUnknownTransaction()) {
              retryServiceScheduler.scheduleRetry(ObjectType.PAYMENT_ATTEMPT, attempt.getId(), attempt.getId(), attempt.getTenantRecordId(),
                                                  paymentStateContext.getPaymentControlPluginNames(), paymentStateContext.getRetryDate());
          }
      }

      private byte[] getSerializedProperties() {
          try {
              return PluginPropertySerializer.serialize(paymentStateContext.getProperties());
          } catch (final PluginPropertySerializerException e) {
              throw new IllegalStateException(e);
          }
      }


      //
      // If we see an UNKNOWN transaction we prevent it to be rescheduled as the Janitor will *try* to fix it, and that could lead to infinite retries from a badly behaved plugin
      // (In other words, plugin should ONLY retry 'known' transaction)
      //
      private boolean isUnknownTransaction() {
          if (paymentStateContext.getCurrentTransaction() != null) {
              return paymentStateContext.getCurrentTransaction().getTransactionStatus() == TransactionStatus.UNKNOWN;
          } else {
              final List<PaymentTransactionModelDao> transactions = retryablePaymentAutomatonRunner.getPaymentDao().getPaymentTransactionsByExternalKey(paymentStateContext.getPaymentTransactionExternalKey(), paymentStateContext.getInternalCallContext());
              return transactions.stream()
                      .anyMatch(input -> input.getTransactionStatus() == TransactionStatus.UNKNOWN &&
                                         // Not strictly required
                                         // (Note, we don't match on AttemptId as it is risky, the row on disk would match the first attempt, not necessarily the current one)
                                         input.getAccountRecordId().equals(paymentStateContext.getInternalCallContext().getAccountRecordId()));
          }
      }
  ```
  Applies to: Attempt-level retry scheduling after a control-plugin-driven failure
  Confidence score: High

- [Kill-BR-0131] - Janitor only reconciles PENDING/UNKNOWN transactions
  Description: IncompletePaymentTransactionTask.TRANSACTION_STATUSES_TO_CONSIDER is hard-coded to {PENDING, UNKNOWN}; any transaction not in one of those two statuses is left untouched by the reconciliation logic (updatePaymentAndTransactionIfNeeded2 short-circuits immediately otherwise). Evidence: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java:63, 119-122.
  Line Numbers: 63 to 63
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  Source Code:
  ```java
      static final List<TransactionStatus> TRANSACTION_STATUSES_TO_CONSIDER = List.of(TransactionStatus.PENDING, TransactionStatus.UNKNOWN);
  ```
  Line Numbers: 119 to 122
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  Source Code:
  ```java
          if (currentTransactionStatus != null && !TRANSACTION_STATUSES_TO_CONSIDER.contains(currentTransactionStatus)) {
              // Nothing to do, so we return the currentTransactionStatus to indicate that nothing has changed
              return currentTransactionStatus;
          }
  ```
  Applies to: Payment-transaction Janitor reconciliation
  Confidence score: High

- [Kill-BR-0132] - Janitor state-repair mapping from re-queried plugin status
  Description: updatePaymentAndTransactionInternal re-derives the transaction's true status from a fresh call to the plugin (via PaymentTransactionInfoPluginConverter), then maps it to the corresponding payment state via PaymentStateMachineHelper (pending/successful/failure/errored-for-transaction-type); if the plugin still can't produce a definitive answer (stays UNKNOWN) or the status hasn't actually changed, no repair is made and the Janitor just logs and returns for another pass later. Evidence: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java:158-222.
  Line Numbers: 158 to 222
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentTransactionTask.java
  Source Code:
  ```java
      private TransactionStatus updatePaymentAndTransactionInternal(final UUID accountId,
                                                                    final PaymentTransactionModelDao paymentTransaction,
                                                                    final PaymentTransactionInfoPlugin paymentTransactionInfoPlugin,
                                                                    final boolean isApiPayment,
                                                                    final InternalTenantContext internalTenantContext) {
          final UUID paymentId = paymentTransaction.getPaymentId();

          // First obtain the new transactionStatus,
          // Then compute the new paymentState; this one is mostly interesting in case of success (to compute the lastSuccessPaymentState below)
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
      }
  ```
  Applies to: Stuck PENDING/UNKNOWN payment transactions
  Confidence score: High

- [Kill-BR-0133] - Gateway status -> internal TransactionStatus mapping (and its retry implications)
  Description: PaymentTransactionInfoPluginConverter.toTransactionStatus maps plugin PaymentPluginStatus values to internal statuses with specific, non-obvious semantics documented in-code: PROCESSED->SUCCESS, PENDING->PENDING, ERROR->PAYMENT_FAILURE ("transaction went through but did not return successfully, e.g. CC denied"), CANCELED->PLUGIN_FAILURE ("plugin knows for sure the transaction did not happen"), and UNDEFINED/null->UNKNOWN (picked up later by the Janitor). This mapping is what feeds both the retry-day-list (PAYMENT_FAILURE) vs exponential-backoff (PLUGIN_FAILURE) branching in rule #19/#20. Evidence: payment/src/main/java/org/killbill/billing/payment/core/PaymentTransactionInfoPluginConverter.java:33-56.
  Line Numbers: 33 to 56
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/PaymentTransactionInfoPluginConverter.java
  Source Code:
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
  Applies to: All payment-plugin gateway responses
  Confidence score: High

- [Kill-BR-0134] - Janitor attempt-completion: missing transaction => ABORTED
  Description: IncompletePaymentAttemptTask.doIteration, when a payment attempt has no matching PaymentTransactionModelDao at all, unconditionally moves that attempt to state "ABORTED" (keeping the original amount/currency so it can still drive future retries) rather than leaving it stuck in INIT. Evidence: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:229-242.
  Line Numbers: 229 to 242
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java
  Source Code:
  ```java
          final PaymentTransactionModelDao transaction = filteredTransactions.isEmpty() ? null : filteredTransactions.get(0);
          if (transaction == null) {
              log.info("Moving attemptId='{}' to ABORTED", attempt.getId());
              paymentDao.updatePaymentAttemptWithProperties(attempt.getId(),
                                                            attempt.getPaymentMethodId(),
                                                            attempt.getTransactionId(),
                                                            "ABORTED",
                                                            // Keep the initial amount, as this will drive the retries
                                                            attempt.getAmount(),
                                                            attempt.getCurrency(),
                                                            attempt.getPluginProperties(),
                                                            internalCallContext);
              return true;
          }
  ```
  Applies to: Payment attempts with a missing/never-created transaction
  Confidence score: High

- [Kill-BR-0135] - Janitor attempt-completion defers to the transaction Janitor for UNKNOWN
  Description: If the matched transaction's status is UNKNOWN, IncompletePaymentAttemptTask.doIteration deliberately returns false without acting — ownership of resolving UNKNOWN transactions belongs to IncompletePaymentTransactionTask, and the attempt-completion pass will only proceed once the transaction moves out of UNKNOWN. Evidence: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:244-248.
  Line Numbers: 244 to 248
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java
  Source Code:
  ```java
          // UNKNOWN transaction are handled by the Janitor IncompletePaymentTransactionTask and should eventually transition to something else,
          // at which point the attempt can also be transition to a different state.
          if (transaction.getTransactionStatus() == TransactionStatus.UNKNOWN) {
              return false;
          }
  ```
  Applies to: Payment attempts whose transaction status is UNKNOWN
  Confidence score: High

- [Kill-BR-0136] - isApiPayment must be true to guarantee retries happen
  Description: An explicit in-code note (and referenced GitHub issue #880) states that for a payment to actually be retried, isApiPayment must be set to true when re-running the attempt-completion Janitor task; this flag is also what feeds PaymentControlContext#isApiPayment for control plugins and gates whether a PaymentErrorInternalEvent is sent from PaymentEnteringStateCallback. Evidence: documented rationale in comment; effect on retry gating not independently re-traced here — payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:205-210.
  Line Numbers: 205 to 210
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java
  Source Code:
  ```java
      // Since the code is a bit tedious to follow, I'm adding some notes here on where isApiPayment is used (valid as of 09/19/2019 - might become stale!):
      //  * Passed through to plugins as PaymentControlContext#isApiPayment (e.g. InvoicePaymentControlPluginApi)
      //  * Used in PaymentEnteringStateCallback to decide whether to send a PaymentErrorInternalEvent
      // To ensure that payments are retried, isApiPayment must be true (see https://github.com/killbill/killbill/issues/880).
      @VisibleForTesting
      public boolean doIteration(final PaymentAttemptModelDao attempt, final boolean isApiPayment, final Iterable<PluginProperty> pluginProperties) {
  ```
  Applies to: Janitor-driven re-completion of payment attempts
  Confidence score: Medium

- [Kill-BR-0137] - Janitor re-check backoff differs by transaction status (UNKNOWN vs PENDING)
  Description: getNextNotificationTime picks between two independently configured backoff schedules — paymentConfig.getUnknownTransactionsRetries() (default "5m,1h,1d,1d,1d,1d,1d") for UNKNOWN transactions and getPendingTransactionsRetries() (default "1h,1d") for PENDING — indexed by attemptNumber; once attemptNumber exceeds the list size, null is returned and the Janitor stops re-checking that transaction. Evidence: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java:401-418; defaults in PaymentConfig.java:61-79.
  Line Numbers: 401 to 418
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/janitor/IncompletePaymentAttemptTask.java
  Source Code:
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
  Line Numbers: 61 to 79
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java
  Source Code:
  ```java
      @Config("org.killbill.payment.janitor.unknown.retries")
      @Default("5m,1h,1d,1d,1d,1d,1d")
      @Description("Delay before which unresolved transactions should be retried")
      List<TimeSpan> getUnknownTransactionsRetries();

      @Config("org.killbill.payment.janitor.unknown.retries")
      @Default("5m,1h,1d,1d,1d,1d,1d")
      @Description("Delay before which unresolved transactions should be retried")
      List<TimeSpan> getUnknownTransactionsRetries(@Param("dummy") final InternalTenantContext tenantContext);

      @Config("org.killbill.payment.janitor.pending.retries")
      @Default("1h, 1d")
      @Description("Delay before which unresolved transactions should be retried")
      List<TimeSpan> getPendingTransactionsRetries();

      @Config("org.killbill.payment.janitor.pending.retries")
      @Default("1h, 1d")
      @Description("Delay before which unresolved transactions should be retried")
      List<TimeSpan> getPendingTransactionsRetries(@Param("dummy") final InternalTenantContext tenantContext);
  ```
  Applies to: Janitor re-check notification scheduling
  Confidence score: High

- [Kill-BR-0138] - Control plugins may only overwrite paymentMethodId if explicitly allowed
  Description: In executePluginPriorCalls, if a control plugin returns an adjustedPaymentMethodId and a paymentMethodId was already set on the payment, the call is rejected with PaymentControlApiException unless paymentConfig.isAllowedToOverwritePaymentMethodId() (default false) — protects against a plugin silently redirecting an existing payment to a different payment method (see referenced issue #1097). Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java:128-136; config in PaymentConfig.java:133-136.
  Line Numbers: 128 to 136
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/control/ControlPluginRunner.java
  Source Code:
  ```java
              if (prevResult.getAdjustedPaymentMethodId() != null) {
                  // We only allow setting the paymentMethodId but disallow overwriting an existing paymentMethodId for a given Payment - See #1097
                  // unless the property isAllowedToOverwritePaymentMethodId was explicitly configured to allow this
                  if (!paymentConfig.isAllowedToOverwritePaymentMethodId() && paymentMethodId != null) {
                      throw new PaymentControlApiException(String.format("Not allowed to overwrite paymentMethodId '%s' for payment '%s'",
                                                                         paymentMethodId, paymentId));
                  }
                  inputPaymentMethodId = prevResult.getAdjustedPaymentMethodId();
              }
  ```
  Line Numbers: 133 to 136
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/PaymentConfig.java
  Source Code:
  ```java
      @Config("org.killbill.payment.method.overwrite")
      @Default("false")
      @Description("Ability to overwrite an existing payment method from a control plugin")
      boolean isAllowedToOverwritePaymentMethodId();
  ```
  Applies to: Control-plugin-adjusted payment method selection
  Confidence score: High

- [Kill-BR-0139] - Deleting the account's default payment method can auto-enable AUTO_PAY_OFF
  Description: PaymentMethodProcessor.deletedPaymentMethod, when deleting the account's current default payment method, requires either deleteDefaultPaymentMethodWithAutoPayOff or forceDefaultPaymentMethodDeletion to be set (otherwise throws PAYMENT_DEL_DEFAULT_PAYMENT_METHOD); if deleteDefaultPaymentMethodWithAutoPayOff is used and the account isn't already on AUTO_PAY_OFF, the processor automatically tags the account AUTO_PAY_OFF before removing the payment method reference, so future auto-charges don't fail outright with no payment method. Evidence: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:503-543.
  Line Numbers: 503 to 543
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  Source Code:
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
                          final PaymentPluginApi pluginApi = getPaymentProviderPlugin(paymentMethodId, false, context);
                          pluginApi.deletePaymentMethod(account.getId(), paymentMethodId, properties, callContext);
                          paymentDao.deletedPaymentMethod(paymentMethodId, context);
                          return PluginDispatcher.createPluginDispatcherReturnType(null);
                      } catch (final PaymentPluginApiException e) {
                          throw new PaymentApiException(e, ErrorCode.PAYMENT_DEL_PAYMENT_METHOD, account.getId(), e.getErrorMessage());
                      } catch (final AccountApiException e) {
                          throw new PaymentApiException(e);
                      }
                  }
              });
          } catch (final Exception e) {
              throw new PaymentApiException(e, ErrorCode.PAYMENT_INTERNAL_ERROR, Objects.requireNonNullElse(e.getMessage(), ""));
          }
      }
  ```
  Applies to: Payment method deletion for the account's default method
  Confidence score: High

- [Kill-BR-0140] - Kill Bill is authoritative for "default payment method," except when it isn't
  Description: updateDefaultPaymentMethodIfNeeded (called from refreshPaymentMethods) normally does NOT let the gateway's notion of a "default" payment method override Kill Bill's own — except if the account's *current* default payment method belongs to the same plugin being refreshed, in which case Kill Bill re-syncs its default to whatever that plugin now reports as default. Evidence: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:656-670.
  Line Numbers: 656 to 670
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  Source Code:
  ```java
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
  Applies to: Payment-method refresh/reconciliation from a gateway
  Confidence score: High

- [Kill-BR-0141] - Single external-payment-method-per-account
  Description: addPaymentMethod's validateUniqueExternalPaymentMethod rejects (PAYMENT_EXTERNAL_PAYMENT_METHOD_ALREADY_EXISTS) adding a second payment method backed by the ExternalPaymentProviderPlugin if the account already has one — an account can only have one "external/manual" payment method. Evidence: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java:155-162.
  Line Numbers: 155 to 162
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/PaymentMethodProcessor.java
  Source Code:
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
  Applies to: Payment method creation using the external/no-op plugin
  Confidence score: High

- [Kill-BR-0142] - Chargeback (and reversal) states are always treated as "success" states for last-success tracking
  Description: PaymentStateMachineHelper.isSuccessState returns true for any state name ending in "SUCCESS" OR starting with "CHARGEBACK" — meaning CHARGEBACK_FAILED and CHARGEBACK_PENDING are also classified as "success" states for purposes of updating a payment's lastSuccessPaymentState in PaymentAutomatonDAOHelper. This is a deliberate quirk (comment notes "a better way would be to change the xml to add attributes"), not a bug. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentStateMachineHelper.java:236-239, used at payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java:150.
  Line Numbers: 236 to 239
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentStateMachineHelper.java
  Source Code:
  ```java
      // A better way would be to change the xml to add attributes to the state (e.g isTerminal, isSuccess, isInit,...)
      public boolean isSuccessState(final String stateName) {
          return stateName.endsWith("SUCCESS") || stateName.startsWith("CHARGEBACK");
      }
  ```
  Line Numbers: 150 to 150
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/PaymentAutomatonDAOHelper.java
  Source Code:
  ```java
          final String lastSuccessPaymentState = paymentSMHelper.isSuccessState(currentPaymentStateName) ? currentPaymentStateName : null;
  ```
  Applies to: Payment last-success-state tracking, all chargeback transitions
  Confidence score: High

- [Kill-BR-0143] - Payment-plugin exceptions/timeouts degrade to UNKNOWN transaction + ERRORED payment state
  Description: PaymentOperation.convertToUnknownTransactionStatusAndErroredPaymentState converts any plugin exception/timeout into an OperationResult.EXCEPTION (drives payment state to *_ERRORED) plus a synthetic PaymentTransactionInfoPlugin with status UNDEFINED (converts to TransactionStatus.UNKNOWN) — deliberately choosing "unknown, needs Janitor" over guessing success or failure when the plugin call itself blew up. Evidence: payment/src/main/java/org/killbill/billing/payment/core/sm/payments/PaymentOperation.java:84-110.
  Line Numbers: 84 to 110
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/sm/payments/PaymentOperation.java
  Source Code:
  ```java
      //
      // In case of exceptions, timeouts, we don't really know what happened:
      // - Return an OperationResult.EXCEPTION to transition Payment State to Errored (see PaymentTransactionInfoPluginConverter#toOperationResult)
      // - Construct a PaymentTransactionInfoPlugin whose PaymentPluginStatus = UNDEFINED to end up with a paymentTransactionStatus = UNKNOWN and have a chance to
      //   be fixed by Janitor.
      //
      private OperationException convertToUnknownTransactionStatusAndErroredPaymentState(final Exception e) {

          final PaymentTransactionInfoPlugin paymentInfoPlugin = new DefaultNoOpPaymentInfoPlugin(paymentStateContext.getPaymentId(),
                                                                                                  paymentStateContext.getTransactionId(),
                                                                                                  paymentStateContext.getTransactionType(),
                                                                                                  paymentStateContext.getAmount(),
                                                                                                  paymentStateContext.getCurrency(),
                                                                                                  paymentStateContext.getCallContext().getCreatedDate(),
                                                                                                  paymentStateContext.getCallContext().getCreatedDate(),
                                                                                                  PaymentPluginStatus.UNDEFINED,
                                                                                                  null,
                                                                                                  null);
          paymentStateContext.setPaymentTransactionInfoPlugin(paymentInfoPlugin);
          if (e.getCause() instanceof OperationException) {
              return (OperationException) e.getCause();
          }
          if (e instanceof OperationException) {
              return (OperationException) e;
          }
          return new OperationException(e, OperationResult.EXCEPTION);
      }
  ```
  Applies to: All payment-plugin operation callbacks (authorize/capture/purchase/refund/credit/void/chargeback)
  Confidence score: High

- [Kill-BR-0144] - AUTO_PAY_OFF gate duplicated between ProcessorBase and InvoicePaymentControlPluginApi
  Description: The AUTO_PAY_OFF control tag is independently checked/read in two separate places rather than through one shared gate: ProcessorBase.isAccountAutoPayOff/setAccountAutoPayOff (tag-based check usable by any processor) and InvoicePaymentControlPluginApi's own isAccountAutoPayOff/insert_AUTO_PAY_OFF_ifRequired/process_AUTO_PAY_OFF_removal (control-plugin-specific enforcement with deferred-retry bookkeeping). Both must independently agree for the tag to be enforced consistently. Evidence: payment/src/main/java/org/killbill/billing/payment/core/ProcessorBase.java:87-99 and payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java:730-744; corroborated by docs/architecture/evidence-map.md's Seam-4 section.
  Line Numbers: 87 to 99
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/core/ProcessorBase.java
  Source Code:
  ```java
      protected boolean isAccountAutoPayOff(final UUID accountId, final InternalTenantContext context) {
          final List<Tag> accountTags = tagInternalApi.getTags(accountId, ObjectType.ACCOUNT, context);
          return ControlTagType.isAutoPayOff(accountTags.stream().map(Tag::getTagDefinitionId).collect(Collectors.toList()));
      }

      protected void setAccountAutoPayOff(final UUID accountId, final InternalCallContext context) throws PaymentApiException {
          try {
              tagInternalApi.addTag(accountId, ObjectType.ACCOUNT, ControlTagType.AUTO_PAY_OFF.getId(), context);
          } catch (TagApiException e) {
              log.error("Failed to add AUTO_PAY_OFF on account " + accountId, e);
              throw new PaymentApiException(ErrorCode.PAYMENT_INTERNAL_ERROR, "Failed to add AUTO_PAY_OFF on account " + accountId);
          }
      }
  ```
  Line Numbers: 730 to 744
  Source Code File Name: payment/src/main/java/org/killbill/billing/payment/invoice/InvoicePaymentControlPluginApi.java
  Source Code:
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
  ```
  Applies to: Auto-pay suppression logic
  Confidence score: High

- [Kill-BR-0145] - First-matching-state overdue evaluation (state-order priority)
  Description: An account's overdue state is computed by iterating configured states in declaration order and returning the first one whose condition evaluates true against the account's current BillingState; if none match, the account is in the CLEAR state. This means state ordering in config is a live business rule — states must be ordered most-severe-first (or however policy intends), since only the first match wins. Evidence: DefaultOverdueStateSet.calculateOverdueState, overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java:59-67.
  Line Numbers: 59 to 67
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStateSet.java
  Source Code:
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
  Applies to: Account overdue-state calculation (triggered on refresh)
  Confidence score: High

- [Kill-BR-0146] - Composite unpaid-invoice condition evaluation
  Description: A state's trigger condition ANDs together up to six independent sub-conditions, each optional (null = "don't care"): number of unpaid invoices >= threshold, total unpaid balance >= threshold, days since the earliest unpaid invoice >= a configured duration (computed as earliestUnpaidInvoiceDate + duration <= evaluation date), last failed payment response is in a configured set, account has a specific control tag, and/or account does NOT have a specific control tag. All present sub-conditions must hold for the state to trigger. Evidence: DefaultOverdueCondition.evaluate, overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java:69-84.
  Line Numbers: 69 to 84
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueCondition.java
  Source Code:
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
  Applies to: Per-state overdue trigger evaluation
  Confidence score: High

- [Kill-BR-0147] - Unified entitlement/billing block flag
  Description: A single config flag, disableEntitlementAndChangesBlocked, simultaneously drives blockEntitlement(), blockBilling(), and (together with blockChanges) blockChanges() when the state is applied — there is no way to block billing without also blocking entitlement in this model. blockChanges() alone is true if either blockChanges is set OR disableEntitlementAndChangesBlocked is set. Evidence: OverdueStateApplicator.blockChanges/blockBilling/blockEntitlement, overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:263-273, consumed at storeNewState lines 221-235.
  Line Numbers: 263 to 273
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
  ```java
      private boolean blockChanges(final OverdueState nextOverdueState) {
          return nextOverdueState.isBlockChanges() || nextOverdueState.isDisableEntitlementAndChangesBlocked();
      }

      private boolean blockBilling(final OverdueState nextOverdueState) {
          return nextOverdueState.isDisableEntitlementAndChangesBlocked();
      }

      private boolean blockEntitlement(final OverdueState nextOverdueState) {
          return nextOverdueState.isDisableEntitlementAndChangesBlocked();
      }
  ```
  Line Numbers: 221 to 235
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
  ```java
      protected void storeNewState(final DateTime effectiveDate, final ImmutableAccountData blockable, final OverdueState nextOverdueState, final InternalCallContext context) throws OverdueException {
          try {
              blockingApi.setBlockingState(new DefaultBlockingState(blockable.getId(),
                                                                    BlockingStateType.ACCOUNT,
                                                                    nextOverdueState.getName(),
                                                                    OverdueService.OVERDUE_SERVICE_NAME,
                                                                    blockChanges(nextOverdueState),
                                                                    blockEntitlement(nextOverdueState),
                                                                    blockBilling(nextOverdueState),
                                                                    effectiveDate),
                                           context);
          } catch (final Exception e) {
              throw new OverdueException(e, ErrorCode.OVERDUE_CAT_ERROR_ENCOUNTERED, blockable.getId(), blockable.getClass().getName());
          }
      }
  ```
  Applies to: BlockingState written for an account when entering an overdue state
  Confidence score: High

- [Kill-BR-0148] - AUTO_INVOICING_OFF toggled on billing block/unblock transitions
  Description: When a state transition newly blocks billing (previous state didn't block billing, next state does), the account gets the AUTO_INVOICING_OFF control tag added, to avoid generating extra invoices/credit while overdue. When a transition newly unblocks billing (leaving a billing-blocked state), the tag is removed. This runs only on an actual previous-state != next-state transition, computed via isBlockBillingTransition/isUnblockBillingTransition. Evidence: OverdueStateApplicator.avoid_extra_credit_by_toggling_AUTO_INVOICE_OFF + set_AUTO_INVOICE_OFF_on_blockedBilling/remove_AUTO_INVOICE_OFF_on_clear, overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:176-183, 237-253.
  Line Numbers: 176 to 183
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
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
  Line Numbers: 237 to 253
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
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
  Applies to: Account billing/invoicing behavior around overdue transitions
  Confidence score: High

- [Kill-BR-0149] - Subscription cancellation policy on entering an overdue state
  Description: A state can carry an OverdueCancellationPolicy of NONE, END_OF_TERM, or IMMEDIATE. On a real state transition (not a no-op), if the policy is not NONE, all of the account's base-plan entitlements (add-ons are excluded — cancelling the base automatically cancels associated add-ons) are cancelled effective at effectiveDate using the corresponding BillingActionPolicy. Evidence: OverdueStateApplicator.cancelSubscriptionsIfRequired + computeEntitlementsToCancel, overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:285-329.
  Line Numbers: 285 to 329
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
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
  Applies to: Subscription/entitlement cancellation as a consequence of overdue state
  Confidence score: High

- [Kill-BR-0150] - OVERDUE_ENFORCEMENT_OFF short-circuits all overdue processing
  Description: Before recalculating/applying any overdue state, refreshWithLock checks whether the account is tagged OVERDUE_ENFORCEMENT_OFF; if so, the refresh is a complete no-op (no state recalculation, no blocking changes, no notifications) regardless of billing state. Evidence: OverdueWrapper.refreshWithLock calling OverdueStateApplicator.isAccountTaggedWith_OVERDUE_ENFORCEMENT_OFF, overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:109-113 and overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:334-349.
  Line Numbers: 109 to 113
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java
  Source Code:
  ```java
      private void refreshWithLock(final DateTime effectiveDate, final InternalCallContext context) throws OverdueException, OverdueApiException {
          if (overdueStateApplicator.isAccountTaggedWith_OVERDUE_ENFORCEMENT_OFF(context)) {
              log.debug("OverdueStateApplicator: apply returns because account (recordId={}) is set with OVERDUE_ENFORCEMENT_OFF", context.getAccountRecordId());
              return;
          }
  ```
  Line Numbers: 334 to 349
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
  ```java
      public boolean isAccountTaggedWith_OVERDUE_ENFORCEMENT_OFF(final InternalCallContext context) throws OverdueException {

          try {
              final UUID accountId = accountApi.getByRecordId(context.getAccountRecordId(), context);

              final List<Tag> accountTags = tagApi.getTags(accountId, ObjectType.ACCOUNT, context);
              for (final Tag cur : accountTags) {
                  if (cur.getTagDefinitionId().equals(ControlTagType.OVERDUE_ENFORCEMENT_OFF.getId())) {
                      return true;
                  }
              }
              return false;
          } catch (final AccountApiException e) {
              throw new OverdueException(e);
          }
      }
  ```
  Applies to: Account-level opt-out of overdue enforcement
  Confidence score: High

- [Kill-BR-0151] - Auto-reevaluation notification scheduling (state-specific vs. initial interval)
  Description: After applying a state, the applicator decides whether/when to schedule the next automatic re-check: if the next state is not CLEAR, or is CLEAR but there's still an unpaid invoice with a first-state configured, a future notification is scheduled. The interval used is the target state's own autoReevaluationInterval if in a non-clear state, or the overdueStateSet's global initialReevaluationInterval if transitioning back to CLEAR. If the relevant interval is unconfigured (zero/unlimited/absent), no reevaluation is scheduled — those conditions are assumed to not be time-based. If the account fully clears with no scheduling condition met, any pending future notification is cancelled. Evidence: OverdueStateApplicator.apply + getReevaluationInterval, overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java:104-125, 160-174; interval sources in DefaultOverdueState.getAutoReevaluationInterval (overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueState.java:120-125) and DefaultOverdueStatesAccount.getInitialReevaluationInterval (overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStatesAccount.java:51-57).
  Line Numbers: 104 to 125
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
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
  Line Numbers: 160 to 174
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/applicator/OverdueStateApplicator.java
  Source Code:
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
  Line Numbers: 120 to 125
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueState.java
  Source Code:
  ```java
      public Duration getAutoReevaluationInterval() throws OverdueApiException {
          if (autoReevaluationInterval == null || autoReevaluationInterval.getUnit() == TimeUnit.UNLIMITED || autoReevaluationInterval.getNumber() == 0) {
              throw new OverdueApiException(ErrorCode.OVERDUE_NO_REEVALUATION_INTERVAL, name);
          }
          return autoReevaluationInterval;
      }
  ```
  Line Numbers: 51 to 57
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/config/DefaultOverdueStatesAccount.java
  Source Code:
  ```java
      @Override
      public Period getInitialReevaluationInterval() {
          if (initialReevaluationInterval == null || initialReevaluationInterval.getUnit() == TimeUnit.UNLIMITED || initialReevaluationInterval.getNumber() == 0) {
              return null;
          }
          return initialReevaluationInterval.toJodaPeriod();
      }
  ```
  Applies to: Automatic periodic re-evaluation of an account's overdue status
  Confidence score: High

- [Kill-BR-0152] - Balance-clearing / invoice/payment events trigger re-evaluation
  Description: Overdue re-evaluation is event-driven off billing activity: invoice creation, invoice payment info, invoice payment error, and invoice adjustment events (plus WRITTEN_OFF tag add/remove on an invoice, and OVERDUE_ENFORCEMENT_OFF tag add/remove on the account) all enqueue a REFRESH notification for the account, causing its BillingState (unpaid invoice count/balance/age) to be recomputed and its overdue state potentially recalculated — this is how clearing a balance (payment fully applied, invoice written off, etc.) drives the account back toward the CLEAR state on the next refresh. Evidence: OverdueListener.handlePaymentInfoEvent/handlePaymentErrorEvent/handleInvoiceAdjustmentEvent/handleInvoiceCreation/handleTagInsert/handleTagRemoval, overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java:96-157.
  Line Numbers: 96 to 157
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java
  Source Code:
  ```java
      @AllowConcurrentEvents
      @Subscribe
      public void handleTagInsert(final ControlTagCreationInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              if (event.getTagDefinition().getName().equals(ControlTagType.OVERDUE_ENFORCEMENT_OFF.toString()) && event.getObjectType() == ObjectType.ACCOUNT) {
                  final InternalCallContext internalCallContext = createCallContext(event.getUserToken(), event.getSearchKey1(), event.getSearchKey2());
                  insertBusEventIntoNotificationQueue(event.getObjectId(), OverdueAsyncBusNotificationAction.CLEAR, internalCallContext);
              } else if (event.getTagDefinition().getName().equals(ControlTagType.WRITTEN_OFF.toString()) && event.getObjectType() == ObjectType.INVOICE) {
                  final UUID accountId = nonEntityDao.retrieveIdFromObject(event.getSearchKey1(), ObjectType.ACCOUNT, objectIdCacheController);
                  insertBusEventIntoNotificationQueue(accountId, event);
              }
          }
      }

      @AllowConcurrentEvents
      @Subscribe
      public void handleTagRemoval(final ControlTagDeletionInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              if (event.getTagDefinition().getName().equals(ControlTagType.OVERDUE_ENFORCEMENT_OFF.toString()) && event.getObjectType() == ObjectType.ACCOUNT) {
                  insertBusEventIntoNotificationQueue(event.getObjectId(), event);
              } else if (event.getTagDefinition().getName().equals(ControlTagType.WRITTEN_OFF.toString()) && event.getObjectType() == ObjectType.INVOICE) {
                  final UUID accountId = nonEntityDao.retrieveIdFromObject(event.getSearchKey1(), ObjectType.ACCOUNT, objectIdCacheController);
                  insertBusEventIntoNotificationQueue(accountId, event);
              }
          }
      }

      @AllowConcurrentEvents
      @Subscribe
      public void handlePaymentInfoEvent(final InvoicePaymentInfoInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              log.debug("Received InvoicePaymentInfo event {}", event);
              insertBusEventIntoNotificationQueue(event.getAccountId(), event);
          }
      }

      @AllowConcurrentEvents
      @Subscribe
      public void handlePaymentErrorEvent(final InvoicePaymentErrorInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              log.debug("Received InvoicePaymentError event {}", event);
              insertBusEventIntoNotificationQueue(event.getAccountId(), event);
          }
      }

      @AllowConcurrentEvents
      @Subscribe
      public void handleInvoiceAdjustmentEvent(final InvoiceAdjustmentInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              log.debug("Received InvoiceAdjustment event {}", event);
              insertBusEventIntoNotificationQueue(event.getAccountId(), event);
          }
      }

      @AllowConcurrentEvents
      @Subscribe
      public void handleInvoiceCreation(final InvoiceCreationInternalEvent event) {
          if (busDispatcherOptimizer.shouldDispatch(event)) {
              log.debug("Received InvoiceCreation event {}", event);
              insertBusEventIntoNotificationQueue(event.getAccountId(), event);
          }
      }
  ```
  Applies to: Reactive overdue re-evaluation triggers
  Confidence score: High

- [Kill-BR-0153] - Parent-account payment delegation for billing state
  Description: When computing an account's BillingState (for overdue evaluation), if the account has a parent account and payment is delegated to the parent (isPaymentDelegatedToParent), the unpaid-invoice/balance data used is the parent account's, not the child's — overdue determination for a delegated child is effectively governed by the parent's payment status. Correspondingly, OverdueListener propagates refresh notifications to the parent and to sibling children when a payment/invoice event fires. Evidence: OverdueWrapper.billingState, overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java:151-159; propagation in OverdueListener.insertBusEventIntoNotificationQueue, overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java:168-204.
  Line Numbers: 151 to 159
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/wrapper/OverdueWrapper.java
  Source Code:
  ```java
      public BillingState billingState(final InternalCallContext context) throws OverdueException {
          if ((overdueable.getParentAccountId() != null) && (overdueable.isPaymentDelegatedToParent())) {
              // calculate billing state from parent account
              final InternalTenantContext internalTenantContext = internalCallContextFactory.createInternalTenantContext(overdueable.getParentAccountId(), context);
              final InternalCallContext parentAccountContext = internalCallContextFactory.createInternalCallContext(internalTenantContext.getAccountRecordId(), context);
              return billingStateCalcuator.calculateBillingState(overdueable, parentAccountContext);
          }
          return billingStateCalcuator.calculateBillingState(overdueable, context);
      }
  ```
  Line Numbers: 168 to 204
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/listener/OverdueListener.java
  Source Code:
  ```java
      private void insertBusEventIntoNotificationQueue(final UUID accountId, final OverdueAsyncBusNotificationAction action, final InternalCallContext callContext) {
          final boolean shouldInsertNotification = shouldInsertNotification(callContext);

          if (!shouldInsertNotification) {
              log.debug("OverdueListener: shouldInsertNotification=false");
              return;
          }

          OverdueAsyncBusNotificationKey notificationKey = new OverdueAsyncBusNotificationKey(accountId, action);
          asyncPoster.insertOverdueNotification(accountId, callContext.getCreatedDate(), OverdueAsyncBusNotifier.OVERDUE_ASYNC_BUS_NOTIFIER_QUEUE, notificationKey, callContext);

          try {
              // Refresh parent
              final Account account = accountApi.getAccountById(accountId, callContext);
              if (account.getParentAccountId() != null && account.isPaymentDelegatedToParent()) {
                  final InternalTenantContext parentAccountInternalTenantContext = internalCallContextFactory.createInternalTenantContext(account.getParentAccountId(), callContext);
                  final InternalCallContext parentAccountContext = internalCallContextFactory.createInternalCallContext(parentAccountInternalTenantContext.getAccountRecordId(), callContext);
                  notificationKey = new OverdueAsyncBusNotificationKey(account.getParentAccountId(), action);
                  asyncPoster.insertOverdueNotification(account.getParentAccountId(), callContext.getCreatedDate(), OverdueAsyncBusNotifier.OVERDUE_ASYNC_BUS_NOTIFIER_QUEUE, notificationKey, parentAccountContext);
              }

              // Refresh children
              final List<Account> childrenAccounts = accountApi.getChildrenAccounts(accountId, callContext);
              if (childrenAccounts != null) {
                  for (final Account childAccount : childrenAccounts) {
                      if (childAccount.isPaymentDelegatedToParent()) {
                          final InternalTenantContext internalTenantContext = internalCallContextFactory.createInternalTenantContext(childAccount.getId(), callContext);
                          final InternalCallContext accountContext = internalCallContextFactory.createInternalCallContext(internalTenantContext.getAccountRecordId(), callContext);
                          notificationKey = new OverdueAsyncBusNotificationKey(childAccount.getId(), action);
                          asyncPoster.insertOverdueNotification(childAccount.getId(), callContext.getCreatedDate(), OverdueAsyncBusNotifier.OVERDUE_ASYNC_BUS_NOTIFIER_QUEUE, notificationKey, accountContext);
                      }
                  }
              }
          } catch (final Exception e) {
              log.error("Error loading child accounts from accountId='{}'", accountId);
          }
      }
  ```
  Applies to: Parent/child account hierarchies with delegated payment
  Confidence score: High

- [Kill-BR-0154] - Unpaid-invoice set defines "overdue-relevant" invoices as of evaluation time
  Description: BillingState is derived only from invoices returned by invoiceApi.getUnpaidInvoicesByAccountId as of the current clock date; from that set it computes count, summed balance, and the earliest invoice date/id, which feed directly into the condition thresholds in rule #2. A last-failed-payment response is hardcoded to INSUFFICIENT_FUNDS (marked TODO in source), meaning the responseForLastFailedPaymentIn condition is not actually driven by real payment-failure data today. Evidence: BillingStateCalculator.calculateBillingState, overdue/src/main/java/org/killbill/billing/overdue/calculator/BillingStateCalculator.java:67-83, note line 79 hardcoded response.
  Line Numbers: 67 to 83
  Source Code File Name: overdue/src/main/java/org/killbill/billing/overdue/calculator/BillingStateCalculator.java
  Source Code:
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
  ```
  Applies to: Data feeding overdue condition evaluation
  Confidence score: High

- [Kill-BR-0155] - Usage-record idempotency via tracking ID
  Description: When a client submits a SubscriptionUsageRecord without a trackingId, the system auto-generates one (UUIDs.randomUUID()) so the batch is unconditionally accepted. When a trackingId IS supplied, the API first checks whether any row already exists for that (subscriptionId, trackingId) pair via rolledUpUsageDao.recordsWithTrackingIdExist(...); if so it aborts the whole write with UsageApiException(ErrorCode.USAGE_RECORD_TRACKING_ID_ALREADY_EXISTS) instead of inserting duplicate rows. Evidence: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java, method recordRolledUpUsage (lines 71-92) and recordsWithTrackingIdExist (lines 170-172); backing SQL in usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg, query recordsWithTrackingIdExist() (lines 26-35).
  Line Numbers: 71 to 92
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
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
  Line Numbers: 170 to 172
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      private boolean recordsWithTrackingIdExist(final SubscriptionUsageRecord record, final InternalCallContext context) {
          return rolledUpUsageDao.recordsWithTrackingIdExist(record.getSubscriptionId(), record.getTrackingId(), context);
      }
  ```
  Line Numbers: 26 to 35
  Source Code File Name: usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg
  Source Code:
  ```java
  recordsWithTrackingIdExist() ::= <<
  select
    1
  from <tableName()>
  where subscription_id = :subscriptionId
  and tracking_id = :trackingId
  <AND_CHECK_TENANT("")>
  limit 1
  ;
  >>
  ```
  Applies to: SubscriptionUsageRecord ingestion (recordRolledUpUsage), used to make repeated submissions of the same usage batch (e.g. plugin retries) safe/non-duplicating.
  Confidence score: High

- [Kill-BR-0156] - Plugin-sourced usage supersedes internally recorded usage
  Description: Before ever reading from Kill Bill's own rolled_up_usage table, the code polls each registered UsagePluginApi (via OSGIServiceRegistration) for the period; the FIRST plugin that returns a non-null list (even an empty one) wins and its data is used exclusively — the internal table is never consulted in that case. Only when no plugin is registered, or every plugin returns null, does the code fall back to the internally stored rolled-up usage. Evidence: usage/src/main/java/org/killbill/billing/usage/api/BaseUserApi.java, method getUsageFromPlugin (lines 63-94, especially the `if (result != null) { ... return result; }` at lines 78-90); consumed in DefaultUsageUserApi.getUsageForSubscription (lines 94-108) and getAllUsageForSubscription (lines 110-134), and DefaultInternalUserApi.getRawUsageForAccount (lines 63-84).
  Line Numbers: 63 to 94
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/BaseUserApi.java
  Source Code:
  ```java
      private List<RawUsageRecord> getUsageFromPlugin(@Nullable final UUID subscriptionId, final DateTime startDate, final DateTime endDate, final Iterable<PluginProperty> properties, final UsageContext usageContext) {
          Preconditions.checkNotNull(usageContext.getAccountId(), "UsageContext has no accountId");

          final Set<String> allServices = pluginRegistry.getAllServices();
          // No plugin registered
          if (allServices.isEmpty()) {
              return null;
          }
          for (final String service : allServices) {
              final UsagePluginApi plugin = pluginRegistry.getServiceForName(service);

              final List<RawUsageRecord> result = subscriptionId != null ?
                                                  plugin.getUsageForSubscription(subscriptionId, startDate, endDate, usageContext, properties) :
                                                  plugin.getUsageForAccount(startDate, endDate, usageContext, properties);
              // First plugin registered, returns result -- could be empty List if no usage was recorded.
              if (result != null) {

                  final DebugMap debugMap = new DebugMap(startDate, endDate, logger);
                  for (final RawUsageRecord cur : result) {
                      if (cur.getDate().compareTo(startDate) < 0 || cur.getDate().compareTo(endDate) >= 0) {
                          logger.warn("Usage plugin returned usage data with date {}, not in the specified range [{} -> {}[",
                                      cur.getDate(), startDate, endDate);
                      }
                      debugMap.add(cur);
                  }
                  debugMap.logDebug();
                  return result;
              }
          }
          // All registered plugins returned null
          return null;
      }
  ```
  Line Numbers: 94 to 108
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      @Override
      public RolledUpUsage getUsageForSubscription(final UUID subscriptionId, final String unitType, final DateTime startDate, final DateTime endDate, final Iterable<PluginProperty> properties, final TenantContext tenantContextNoAccountId) {
          final InternalTenantContext internalCallContext = internalCallContextFactory.createInternalTenantContext(subscriptionId, ObjectType.SUBSCRIPTION, tenantContextNoAccountId);
          final TenantContext tenantContext = internalCallContextFactory.createTenantContext(internalCallContext);
          final UsageContext usageContext = new DefaultUsageContext(null, null, tenantContext);
          final List<RawUsageRecord> rawUsage = getSubscriptionUsageFromPlugin(subscriptionId, startDate, endDate, properties, usageContext);
          if (rawUsage != null) {
              final List<RolledUpUnit> rolledUpAmount = getRolledUpUnitsForRawPluginUsage(subscriptionId, unitType, rawUsage);
              return new DefaultRolledUpUsage(subscriptionId, startDate, endDate, rolledUpAmount);
          }

          final List<RolledUpUsageModelDao> usageForSubscription = rolledUpUsageDao.getUsageForSubscription(subscriptionId, startDate, endDate, unitType, internalCallContextFactory.createInternalTenantContext(subscriptionId, ObjectType.SUBSCRIPTION, tenantContext));
          final List<RolledUpUnit> rolledUpAmount = getRolledUpUnits(usageForSubscription);
          return new DefaultRolledUpUsage(subscriptionId, startDate, endDate, rolledUpAmount);
      }
  ```
  Line Numbers: 110 to 134
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      @Override
      public List<RolledUpUsage> getAllUsageForSubscription(final UUID subscriptionId, final List<DateTime> transitionTimes, final Iterable<PluginProperty> properties, final TenantContext tenantContextNoAccountId) {
          final InternalTenantContext internalCallContext = internalCallContextFactory.createInternalTenantContext(subscriptionId, ObjectType.SUBSCRIPTION, tenantContextNoAccountId);
          final TenantContext tenantContext = internalCallContextFactory.createTenantContext(internalCallContext);
          final UsageContext usageContext = new DefaultUsageContext(null, null, tenantContext);

          final List<RolledUpUsage> result = new ArrayList<RolledUpUsage>();
          DateTime prevDate = null;
          for (final DateTime curDate : transitionTimes) {
              if (prevDate != null) {

                  final List<RawUsageRecord> rawUsage = getSubscriptionUsageFromPlugin(subscriptionId, prevDate, curDate, properties, usageContext);
                  if (rawUsage != null) {
                      final List<RolledUpUnit> rolledUpAmount = getRolledUpUnitsForRawPluginUsage(subscriptionId, null, rawUsage);
                      result.add(new DefaultRolledUpUsage(subscriptionId, prevDate, curDate, rolledUpAmount));
                  } else {
                      final List<RolledUpUsageModelDao> usageForSubscription = rolledUpUsageDao.getAllUsageForSubscription(subscriptionId, prevDate, curDate, internalCallContext);
                      final List<RolledUpUnit> rolledUpAmount = getRolledUpUnits(usageForSubscription);
                      result.add(new DefaultRolledUpUsage(subscriptionId, prevDate, curDate, rolledUpAmount));
                  }
              }
              prevDate = curDate;
          }
          return result;
      }
  ```
  Line Numbers: 63 to 84
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/svcs/DefaultInternalUserApi.java
  Source Code:
  ```java
      @Override
      public List<RawUsageRecord> getRawUsageForAccount(final DateTime startDate, final DateTime endDate, @Nullable final DryRunInfo dryRunInfo, final Iterable<PluginProperty> pluginProperties, final InternalTenantContext internalTenantContext) {

          log.info("GetRawUsageForAccount startDate='{}', endDate='{}'", startDate, endDate);

          final TenantContext tenantContext = internalCallContextFactory.createTenantContext(internalTenantContext);

          final DryRunType dryRunType = dryRunInfo != null ? dryRunInfo.getDryRunType() : null;
          final LocalDate inputTargetDate = dryRunInfo != null ? dryRunInfo.getInputTargetDate() : null;

          final UsageContext usageContext = new DefaultUsageContext(dryRunType, inputTargetDate, tenantContext);

          final List<RawUsageRecord> resultFromPlugin = getAccountUsageFromPlugin(startDate, endDate, pluginProperties, usageContext);
          if (resultFromPlugin != null) {
              return resultFromPlugin;
          }

          final List<RolledUpUsageModelDao> usage = rolledUpUsageDao.getRawUsageForAccount(startDate, endDate, internalTenantContext);
          return usage.stream()
                  .map(input -> new DefaultRawUsage(input.getSubscriptionId(), input.getRecordDate(), input.getUnitType(), input.getAmount(), input.getTrackingId()))
                  .collect(Collectors.toUnmodifiableList());
      }
  ```
  Applies to: Usage-data sourcing for both per-subscription rolled-up usage lookups and account-level raw usage feeding invoicing.
  Confidence score: High

- [Kill-BR-0157] - Per-unit-type usage aggregation (roll-up)
  Description: Raw/daily usage records for a subscription are summed into one total BigDecimal amount per unit type (e.g. "SMS", "MINUTES"), discarding the individual per-day granularity, to produce the RolledUpUnit values billing consumes. Two near-identical implementations do this: one for plugin-returned RawUsageRecords (filtering out records for the wrong subscriptionId or, if a unitType filter is given, the wrong unit type, then accumulating into a Map<String,BigDecimal>) and one for internally stored RolledUpUsageModelDao rows. Evidence: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java, methods getRolledUpUnitsForRawPluginUsage (lines 136-156) and getRolledUpUnits (lines 158-168).
  Line Numbers: 136 to 156
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      private List<RolledUpUnit> getRolledUpUnitsForRawPluginUsage(final UUID subscriptionId, @Nullable final String unitType, final List<RawUsageRecord> rawAccountUsage) {
          final Map<String, BigDecimal> tmp = new HashMap<>();
          for (final RawUsageRecord cur : rawAccountUsage) {
              // Filter out wrong subscriptionId
              if (cur.getSubscriptionId().compareTo(subscriptionId) != 0) {
                  continue;
              }

              // Filter out wrong unitType if specified.
              if (unitType != null && !unitType.equals(cur.getUnitType())) {
                  continue;
              }

              final BigDecimal currentAmount = tmp.get(cur.getUnitType());
              final BigDecimal updatedAmount = (currentAmount != null) ? currentAmount.add(cur.getAmount()) : cur.getAmount();
              tmp.put(cur.getUnitType(), updatedAmount);
          }
          return tmp.entrySet()
                    .stream().map(e -> new DefaultRolledUpUnit(e.getKey(), e.getValue()))
                    .collect(Collectors.toUnmodifiableList());
      }
  ```
  Line Numbers: 158 to 168
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      private List<RolledUpUnit> getRolledUpUnits(final List<RolledUpUsageModelDao> usageForSubscription) {
          final Map<String, BigDecimal> tmp = new HashMap<>();
          for (final RolledUpUsageModelDao cur : usageForSubscription) {
              final BigDecimal currentAmount = tmp.get(cur.getUnitType());
              final BigDecimal updatedAmount = (currentAmount != null) ? currentAmount.add(cur.getAmount()) : cur.getAmount();
              tmp.put(cur.getUnitType(), updatedAmount);
          }
          return tmp.entrySet()
                    .stream().map(e -> new DefaultRolledUpUnit(e.getKey(), e.getValue()))
                    .collect(Collectors.toUnmodifiableList());
      }
  ```
  Applies to: Building RolledUpUsage/RolledUpUnit results returned by getUsageForSubscription and getAllUsageForSubscription, which feed usage-based invoicing.
  Confidence score: High

- [Kill-BR-0158] - Usage periods segmented by subscription transition times
  Description: getAllUsageForSubscription does not roll up usage over one flat range; it walks the caller-supplied list of transitionTimes pairwise (prevDate -> curDate) and produces one independent RolledUpUsage per adjacent pair, aggregating usage separately within each sub-period. Since transitionTimes represent subscription phase/billing-period boundaries (supplied by the invoice/junction layer), this aligns usage roll-up boundaries with subscription billing-period transitions rather than using one uniform window. Evidence: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java, method getAllUsageForSubscription (lines 110-134).
  Line Numbers: 110 to 134
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/user/DefaultUsageUserApi.java
  Source Code:
  ```java
      @Override
      public List<RolledUpUsage> getAllUsageForSubscription(final UUID subscriptionId, final List<DateTime> transitionTimes, final Iterable<PluginProperty> properties, final TenantContext tenantContextNoAccountId) {
          final InternalTenantContext internalCallContext = internalCallContextFactory.createInternalTenantContext(subscriptionId, ObjectType.SUBSCRIPTION, tenantContextNoAccountId);
          final TenantContext tenantContext = internalCallContextFactory.createTenantContext(internalCallContext);
          final UsageContext usageContext = new DefaultUsageContext(null, null, tenantContext);

          final List<RolledUpUsage> result = new ArrayList<RolledUpUsage>();
          DateTime prevDate = null;
          for (final DateTime curDate : transitionTimes) {
              if (prevDate != null) {

                  final List<RawUsageRecord> rawUsage = getSubscriptionUsageFromPlugin(subscriptionId, prevDate, curDate, properties, usageContext);
                  if (rawUsage != null) {
                      final List<RolledUpUnit> rolledUpAmount = getRolledUpUnitsForRawPluginUsage(subscriptionId, null, rawUsage);
                      result.add(new DefaultRolledUpUsage(subscriptionId, prevDate, curDate, rolledUpAmount));
                  } else {
                      final List<RolledUpUsageModelDao> usageForSubscription = rolledUpUsageDao.getAllUsageForSubscription(subscriptionId, prevDate, curDate, internalCallContext);
                      final List<RolledUpUnit> rolledUpAmount = getRolledUpUnits(usageForSubscription);
                      result.add(new DefaultRolledUpUsage(subscriptionId, prevDate, curDate, rolledUpAmount));
                  }
              }
              prevDate = curDate;
          }
          return result;
      }
  ```
  Applies to: Multi-period usage roll-up during invoicing of subscriptions that changed phase/plan mid-period.
  Confidence score: High

- [Kill-BR-0159] - Cancellation-day inclusive boundary for invoiced usage
  Description: Two internal queries (getUsageForSubscription, getAllUsageForSubscription) use a half-open date range `record_date >= :startDate and record_date < :endDate`. But getRawUsageForAccount — explicitly documented in-code as "the only query used for invoicing" — instead uses a closed range `record_date >= :startDate and record_date <= :endDate`, specifically "to handle usage data at the cancellation day." This is a deliberate asymmetry: the invoicing path intentionally includes usage recorded exactly on the period's end date (e.g., the day a subscription is cancelled) so it isn't silently dropped from the final invoice, while other lookups use the standard exclusive-end convention. Evidence: usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg, queries getUsageForSubscription (lines 37-48), getAllUsageForSubscription (lines 50-60), and getRawUsageForAccount with its comment (lines 62-73); consumed by DefaultInternalUserApi.getRawUsageForAccount (usage/src/main/java/org/killbill/billing/usage/api/svcs/DefaultInternalUserApi.java, lines 63-84).
  Line Numbers: 37 to 48
  Source Code File Name: usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg
  Source Code:
  ```java
  getUsageForSubscription() ::= <<
  select
    <allTableFields("")>
  from <tableName()>
  where subscription_id = :subscriptionId
  and record_date >= :startDate
  and record_date \< :endDate
  and unit_type = :unitType
  <AND_CHECK_TENANT("")>
  <defaultOrderBy("")>
  ;
  >>
  ```
  Line Numbers: 50 to 60
  Source Code File Name: usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg
  Source Code:
  ```java
  getAllUsageForSubscription() ::= <<
  select
    <allTableFields("")>
  from <tableName()>
  where subscription_id = :subscriptionId
  and record_date >= :startDate
  and record_date \< :endDate
  <AND_CHECK_TENANT("")>
  <defaultOrderBy("")>
  ;
  >>
  ```
  Line Numbers: 62 to 73
  Source Code File Name: usage/src/main/resources/org/killbill/billing/usage/dao/RolledUpUsageSqlDao.sql.stg
  Source Code:
  ```java
  /** This is the only query used for invoicing, hence the <= :endDate (to handle usage data at the cancellation day) **/
  getRawUsageForAccount() ::= <<
  select
    <allTableFields("")>
  from <tableName()>
  where account_record_id = :accountRecordId
  and record_date >= :startDate
  and record_date \<= :endDate
  <AND_CHECK_TENANT("")>
  <defaultOrderBy("")>
  ;
  >>
  ```
  Line Numbers: 63 to 84
  Source Code File Name: usage/src/main/java/org/killbill/billing/usage/api/svcs/DefaultInternalUserApi.java
  Source Code:
  ```java
      @Override
      public List<RawUsageRecord> getRawUsageForAccount(final DateTime startDate, final DateTime endDate, @Nullable final DryRunInfo dryRunInfo, final Iterable<PluginProperty> pluginProperties, final InternalTenantContext internalTenantContext) {

          log.info("GetRawUsageForAccount startDate='{}', endDate='{}'", startDate, endDate);

          final TenantContext tenantContext = internalCallContextFactory.createTenantContext(internalTenantContext);

          final DryRunType dryRunType = dryRunInfo != null ? dryRunInfo.getDryRunType() : null;
          final LocalDate inputTargetDate = dryRunInfo != null ? dryRunInfo.getInputTargetDate() : null;

          final UsageContext usageContext = new DefaultUsageContext(dryRunType, inputTargetDate, tenantContext);

          final List<RawUsageRecord> resultFromPlugin = getAccountUsageFromPlugin(startDate, endDate, pluginProperties, usageContext);
          if (resultFromPlugin != null) {
              return resultFromPlugin;
          }

          final List<RolledUpUsageModelDao> usage = rolledUpUsageDao.getRawUsageForAccount(startDate, endDate, internalTenantContext);
          return usage.stream()
                  .map(input -> new DefaultRawUsage(input.getSubscriptionId(), input.getRecordDate(), input.getUnitType(), input.getAmount(), input.getTrackingId()))
                  .collect(Collectors.toUnmodifiableList());
      }
  ```
  Applies to: Account-level raw usage retrieval that feeds usage invoicing, vs. subscription-level rolled-up usage lookups.
  Confidence score: High

- [Kill-BR-0160] - Per-tenant config single-value / "latest wins" invariant
  Description: For system config keys that represent a single logical value per tenant (catalog XML, overdue config, per-tenant config JSON, invoice templates, plugin config, etc.), the tenant module enforces that exactly one value exists at read time and treats the most recent write as authoritative. `DefaultTenantInternalApi.getUniqueValue(...)` (tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenantInternalApi.java:132-141) throws `IllegalStateException` if more than one row is returned for a single-value key. On the write side, `DefaultTenantDao.addTenantKeyValue(...)` (DefaultTenantDao.java:126-137) soft-deletes prior rows for the key before inserting when `uniqueKey` is true, and `DefaultTenantDao.updateTenantLastKeyValue(...)` (DefaultTenantDao.java:140-157) explicitly updates the last-inserted row for that key rather than appending. Which keys are "unique" is decided by `DefaultTenantUserApi.isSingleValueKey(...)` (DefaultTenantUserApi.java:202-204), which delegates to `TenantKey.isSingleValue()`.
  Line Numbers: 132 to 141
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenantInternalApi.java
  Source Code:
  ```java
      private String getUniqueValue(final List<String> values, final String msg, final InternalTenantContext tenantContext) {
          if (values.isEmpty()) {
              return null;
          }
          if (values.size() > 1) {
              throw new IllegalStateException(String.format("Unexpected number of values %d for %s and tenant %d",
                                                            values.size(), msg, tenantContext.getTenantRecordId()));
          }
          return values.get(0);
      }
  ```
  Line Numbers: 126 to 137
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      public void addTenantKeyValue(final String key, final String value, final boolean uniqueKey, final InternalCallContext context) {
          transactionalSqlDao.execute(false, entitySqlDaoWrapperFactory -> {
              final TenantKVModelDao tenantKVModelDao = new TenantKVModelDao(UUIDs.randomUUID(), context.getCreatedDate(), context.getUpdatedDate(), key, value);
              final TenantKVSqlDao tenantKVSqlDao = entitySqlDaoWrapperFactory.become(TenantKVSqlDao.class);
              if (uniqueKey) {
                  deleteFromTransaction(key, entitySqlDaoWrapperFactory, context);
              }
              final TenantKVModelDao rehydrated = createAndRefresh(tenantKVSqlDao, tenantKVModelDao, context);
              broadcastConfigurationChangeFromTransaction(rehydrated.getRecordId(), key, entitySqlDaoWrapperFactory, context);
              return null;
          });
      }
  ```
  Line Numbers: 140 to 157
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      public void updateTenantLastKeyValue(final String key, final String value, final InternalCallContext context) {
          transactionalSqlDao.execute(false, entitySqlDaoWrapperFactory -> {
              final TenantKVModelDao tenantKVModelDao = new TenantKVModelDao(UUIDs.randomUUID(), context.getCreatedDate(), context.getUpdatedDate(), key, value);
              final TenantKVSqlDao tenantKVSqlDao = entitySqlDaoWrapperFactory.become(TenantKVSqlDao.class);

              // Retrieve all values for key ordered with recordId (last at the end)
              final List<TenantKVModelDao> tenantKV = tenantKVSqlDao.getTenantValueForKey(key, context);
              final TenantKVModelDao rehydrated;
              if (!tenantKV.isEmpty()) {
                  final String id = tenantKV.get(tenantKV.size() - 1).getId().toString();
                  rehydrated = (TenantKVModelDao) tenantKVSqlDao.updateTenantValueKey(id, value, context);
              } else {
                  rehydrated = createAndRefresh(tenantKVSqlDao, tenantKVModelDao, context);
              }
              broadcastConfigurationChangeFromTransaction(rehydrated.getRecordId(), key, entitySqlDaoWrapperFactory, context);
              return null;
          });
      }
  ```
  Line Numbers: 202 to 204
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
  ```java
      private boolean isSingleValueKey(final String key) {
          return Arrays.stream(TenantKey.values()).anyMatch(input -> input.isSingleValue() && key.startsWith(input.toString()));
      }
  ```
  Applies to: Per-tenant overrides of catalog, overdue config, invoice templates/translations, generic per-tenant config, plugin config.
  Confidence score: High

- [Kill-BR-0161] - System-key-only cross-node broadcast/cache-invalidation
  Description: A config change (add/update/delete of a tenant_kvs entry) only triggers a `tenant_broadcasts` row (and therefore cross-node cache invalidation + a `TenantConfigChange`/`TenantConfigDeletion` bus event) when the key matches one of the well-known `TenantKey` enum prefixes. `DefaultTenantDao.broadcastConfigurationChangeFromTransaction(...)` (DefaultTenantDao.java:190-197) calls `isSystemKey(key)` (DefaultTenantDao.java:202-204, `Arrays.stream(TenantKey.values()).anyMatch(input -> key.startsWith(input.toString()))`) and skips the broadcast entirely for arbitrary/custom keys that don't match a known system prefix. This means only recognized system config (catalog, overdue config, plugin config, push-notification callback, invoice templates, etc.) propagates to other nodes; free-form tenant KV entries do not.
  Line Numbers: 190 to 197
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      private void broadcastConfigurationChangeFromTransaction(final Long kvRecordId, final String key, final EntitySqlDaoWrapperFactory entitySqlDaoWrapperFactory,
                                                               final InternalCallContext context) throws EntityPersistenceException {
          if (isSystemKey(key)) {
              final TenantBroadcastModelDao broadcast = new TenantBroadcastModelDao(kvRecordId, key, context.getUserToken());
              final TenantBroadcastSqlDao tenantBroadcastSqlDao = entitySqlDaoWrapperFactory.become(TenantBroadcastSqlDao.class);
              createAndRefresh(tenantBroadcastSqlDao, broadcast, context);
          }
      }
  ```
  Line Numbers: 202 to 204
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      private boolean isSystemKey(final String key) {
          return Arrays.stream(TenantKey.values()).anyMatch(input -> key.startsWith(input.toString()));
      }
  ```
  Applies to: Multi-node cache coherence for tenant config, including per-tenant plugin config and push-notification callback (`PUSH_NOTIFICATION_CB`) precedence/propagation.
  Confidence score: High

- [Kill-BR-0162] - Selective server-side caching of tenant config keys
  Description: Only a hardcoded subset of `TenantKey`s (`CATALOG_TRANSLATION_`, `INVOICE_MP_TEMPLATE`, `INVOICE_TEMPLATE`, `INVOICE_TRANSLATION_`, `PLUGIN_CONFIG_`, `PLUGIN_PAYMENT_STATE_MACHINE_`, `PUSH_NOTIFICATION_CB`) are served from the `tenant-kv` cache; all other keys always hit the DAO/DB directly. This is implemented in `DefaultTenantUserApi.CACHED_TENANT_KEY` (DefaultTenantUserApi.java:63-69) and enforced in `getTenantValuesForKey`/`isCachedInTenantKVCache` (DefaultTenantUserApi.java:133-140, 206-208). The code comment explicitly states this exclusion (e.g. CATALOG's multi-value nature) is "an implementation choice," not an API contract — i.e. a deliberate precedence/caching policy for which per-tenant overrides get cached at this layer vs. reconstructed higher up (catalog/overdue modules).
  Line Numbers: 63 to 69
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
  ```java
      public static final Iterable<TenantKey> CACHED_TENANT_KEY = List.of(TenantKey.CATALOG_TRANSLATION_,
                                                                          TenantKey.INVOICE_MP_TEMPLATE,
                                                                          TenantKey.INVOICE_TEMPLATE,
                                                                          TenantKey.INVOICE_TRANSLATION_,
                                                                          TenantKey.PLUGIN_CONFIG_,
                                                                          TenantKey.PLUGIN_PAYMENT_STATE_MACHINE_,
                                                                          TenantKey.PUSH_NOTIFICATION_CB);
  ```
  Line Numbers: 133 to 140
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
  ```java
      public List<String> getTenantValuesForKey(final String key, final TenantContext context) throws TenantApiException {
          final InternalTenantContext internalContext = internalCallContextFactory.createInternalTenantContextWithoutAccountRecordId(context);
          if (!isCachedInTenantKVCache(key)) {
              return tenantDao.getTenantValueForKey(key, internalContext);
          } else {
              return getCachedTenantValuesForKey(key, internalContext);
          }
      }
  ```
  Line Numbers: 206 to 208
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
  ```java
      private boolean isCachedInTenantKVCache(final String key) {
          return Iterables.toUnmodifiableList(CACHED_TENANT_KEY).stream().anyMatch(input -> key.startsWith(input.toString()));
      }
  ```
  Applies to: Per-tenant plugin config and push-notification callback config precedence/lookup path; catalog/overdue config excluded from this cache layer.
  Confidence score: High

- [Kill-BR-0163] - Tenant external-key length limit
  Description: `DefaultTenantUserApi.createTenant(...)` rejects tenant creation when `externalKey.length() > 255`, throwing `TenantApiException(ErrorCode.EXTERNAL_KEY_LIMIT_EXCEEDED)` (DefaultTenantUserApi.java:89-91).
  Line Numbers: 89 to 91
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
  ```java
          if (null != tenant.getExternalKey() && tenant.getExternalKey().length() > 255) {
              throw new TenantApiException(ErrorCode.EXTERNAL_KEY_LIMIT_EXCEEDED);
          }
  ```
  Applies to: Tenant creation/onboarding.
  Confidence score: High

- [Kill-BR-0164] - API-key uniqueness enforced at tenant creation
  Description: `DefaultTenantUserApi.createTenant(...)` (DefaultTenantUserApi.java:93-104) looks up an existing tenant by the incoming API key before insert and throws `TenantApiException(ErrorCode.TENANT_ALREADY_EXISTS, ...)` if one is found, treating a lookup failure (`IllegalStateException` cause, i.e. "not found") as the expected non-collision case. This is a best-effort (non-transactional, comment notes reliance on a DB unique constraint as the real guarantee) business check that a tenant API key must be unique system-wide.
  Line Numbers: 93 to 104
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/user/DefaultTenantUserApi.java
  Source Code:
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
  Applies to: Tenant/API-key provisioning.
  Confidence score: High

- [Kill-BR-0165] - API secret is salted, iteratively hashed, and never stored/exposed in clear text
  Description: `DefaultTenantDao.create(...)` (DefaultTenantDao.java:87-103) generates a random salt via `SecureRandomNumberGenerator`, hashes the plaintext `apiSecret` with `SimpleHash` using the Shiro credentials-matcher algorithm and `securityConfig.getShiroNbHashIterations()` iterations, Base64-encodes it, and persists only the hash + salt — the model never stores the plaintext (`DefaultTenant.apiSecret` field is `transient` and `toString()`/`writeExternal()` explicitly omit it, DefaultTenant.java:43, 113-123, 174-185). Authentication then goes through Shiro's credential matcher against this hash (`getAuthenticationInfoForTenant`, DefaultTenantDao.java:105-115), i.e. tenant API secret validation is a hashed-credential match, not a plaintext comparison.
  Line Numbers: 87 to 103
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      @Override
      public void create(final TenantModelDao entity, final InternalCallContext context) throws TenantApiException {
          // Create the salt and password
          final ByteSource salt = rng.nextBytes();
          // Hash the plain-text password with the random salt and multiple iterations and then Base64-encode the value (requires less space than Hex)
          final String hashedPasswordBase64 = new SimpleHash(KillbillCredentialsMatcher.HASH_ALGORITHM_NAME,
                                                             entity.getApiSecret(), salt, securityConfig.getShiroNbHashIterations()).toBase64();

          transactionalSqlDao.execute(false, entitySqlDaoWrapperFactory -> {
              final TenantModelDao tenantModelDaoWithSecret = new TenantModelDao(entity.getId(), context.getCreatedDate(), context.getUpdatedDate(),
                                                                                 entity.getExternalKey(), entity.getApiKey(),
                                                                                 hashedPasswordBase64, salt.toBase64());
              final TenantSqlDao tenantSqlDao = entitySqlDaoWrapperFactory.become(TenantSqlDao.class);
              createAndRefresh(tenantSqlDao, tenantModelDaoWithSecret, context);
              return null;
          });
      }
  ```
  Line Numbers: 105 to 115
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/dao/DefaultTenantDao.java
  Source Code:
  ```java
      @VisibleForTesting
      AuthenticationInfo getAuthenticationInfoForTenant(final UUID id) {
          return transactionalSqlDao.execute(true, entitySqlDaoWrapperFactory -> {
              final TenantModelDao tenantModelDao = entitySqlDaoWrapperFactory.become(TenantSqlDao.class).getSecrets(id.toString());

              final SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(tenantModelDao.getApiKey(), tenantModelDao.getApiSecret().toCharArray(), getClass().getSimpleName());
              authenticationInfo.setCredentialsSalt(ByteSource.Util.bytes(Base64.decode(tenantModelDao.getApiSalt())));

              return authenticationInfo;
          });
      }
  ```
  Line Numbers: 43 to 43
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java
  Source Code:
  ```java
      private transient String apiSecret;
  ```
  Line Numbers: 113 to 123
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java
  Source Code:
  ```java
      public String toString() {
          final StringBuilder sb = new StringBuilder("DefaultTenant{");
          sb.append("id=").append(id);
          sb.append(", createdDate=").append(createdDate);
          sb.append(", updatedDate=").append(updatedDate);
          sb.append(", externalKey='").append(externalKey).append('\'');
          sb.append(", apiKey='").append(apiKey).append('\'');
          // Don't print the secret
          sb.append('}');
          return sb.toString();
      }
  ```
  Line Numbers: 174 to 185
  Source Code File Name: tenant/src/main/java/org/killbill/billing/tenant/api/DefaultTenant.java
  Source Code:
  ```java
      @Override
      public void writeExternal(final ObjectOutput oo) throws IOException {
          oo.writeLong(id.getMostSignificantBits());
          oo.writeLong(id.getLeastSignificantBits());
          oo.writeUTF(createdDate.toString());
          oo.writeUTF(updatedDate.toString());
          oo.writeBoolean(externalKey != null);
          if (externalKey != null) {
              oo.writeUTF(externalKey);
          }
          oo.writeUTF(apiKey);
      }
  ```
  Applies to: Tenant API key/secret validation and storage.
  Confidence score: High

- [Kill-BR-0166] - Single configured provider, no cross-provider fallback
  Description: `DefaultCurrencyConversionApi.getPluginApi()` (currency/src/main/java/org/killbill/billing/currency/api/DefaultCurrencyConversionApi.java, lines 43-49) looks up exactly one `CurrencyPluginApi` implementation from the OSGI plugin registry, keyed by the single provider name returned by `config.getDefaultCurrencyProvider()` (backed by `org.killbill.currency.provider.default`, default value `killbill-currency-plugin`, defined in util/src/main/java/org/killbill/billing/util/config/definition/CurrencyConfig.java lines 26-29). If no plugin is registered under that name, it throws `CurrencyConversionException(ErrorCode.CURRENCY_NO_SUCH_PAYMENT_PLUGIN, ...)` rather than trying any other registered provider. This is called from both `getCurrentCurrencyConversion(...)` and `getCurrencyConversion(..., dateConversion)` (lines 58-67), so every rate lookup — current or historical — is bound to this one configured provider with no precedence/fallback chain across multiple providers. Note on scope: actual exchange-rate lookup/date-based fallback/rounding logic is delegated to an externally registered CurrencyPluginApi plugin (outside this module); no tax logic was found in this module; no currency-support/validation or rate-rounding logic exists in currency/src/main/java itself.
  Line Numbers: 43 to 49
  Source Code File Name: currency/src/main/java/org/killbill/billing/currency/api/DefaultCurrencyConversionApi.java
  Source Code:
  ```java
      private CurrencyPluginApi getPluginApi() throws CurrencyConversionException {
          final CurrencyPluginApi result = registry.getServiceForName(config.getDefaultCurrencyProvider());
          if (result == null) {
              throw new CurrencyConversionException(ErrorCode.CURRENCY_NO_SUCH_PAYMENT_PLUGIN, config.getDefaultCurrencyProvider());
          }
          return result;
      }
  ```
  Line Numbers: 26 to 29
  Source Code File Name: util/src/main/java/org/killbill/billing/util/config/definition/CurrencyConfig.java
  Source Code:
  ```java
      @Config("org.killbill.currency.provider.default")
      @Default("killbill-currency-plugin")
      @Description("Default currency provider to use")
      public String getDefaultCurrencyProvider();
  ```
  Line Numbers: 58 to 67
  Source Code File Name: currency/src/main/java/org/killbill/billing/currency/api/DefaultCurrencyConversionApi.java
  Source Code:
  ```java
      public CurrencyConversion getCurrentCurrencyConversion(final Currency baseCurrency) throws CurrencyConversionException {
          final Set<Rate> allRates = getPluginApi().getCurrentRates(baseCurrency);
          return getCurrencyConversionInternal(baseCurrency, allRates);
      }

      @Override
      public CurrencyConversion getCurrencyConversion(final Currency baseCurrency, final DateTime dateConversion) throws CurrencyConversionException {
          final Set<Rate> allRates = getPluginApi().getRates(baseCurrency, dateConversion);
          return getCurrencyConversionInternal(baseCurrency, allRates);
      }
  ```
  Applies to: Currency conversion/rate lookup entry points (`getBaseRates`, `getCurrentCurrencyConversion`, `getCurrencyConversion`).
  Confidence score: High
