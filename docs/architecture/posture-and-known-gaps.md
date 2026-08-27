# Posture & Known Gaps

This document records two factual, evidence-bound observations about the current state of the
codebase, as observed at the pinned commit. See
[evidence-map.md](./evidence-map.md#cross-cutting-tenancy-audit-rbac-osgiplugin-loading) for the
underlying class/file evidence instead of restated paths here.

## RBAC enforcement gap

Kill Bill has a complete Shiro-based RBAC annotation mechanism: `@RequiresPermissions` is enforced, when
RBAC is enabled, by `PermissionAnnotationHandler`, wired through `KillBillShiroAopModule` (see the
evidence map's RBAC subsection). The mechanism itself is fully built.

`@RequiresPermissions` is applied to zero production JAX-RS resource endpoints. Its only appearance on
any method in this repository, outside its own handler, is on a test fixture method in
`TestPermissionAnnotationMethodInterceptor.java` (see the evidence map). No class under
`jaxrs/src/main/java/org/killbill/billing/jaxrs/resources/` carries the annotation.

The effective state of authorization on the JAX-RS API surface today is therefore **"authenticated tenant user"**, not per-operation permission checks: a request that successfully authenticates against
a tenant (via `TenantFilter`/`KillbillJdbcTenantRealm`, see the evidence map) is authorized to invoke
any JAX-RS endpoint that authentication reaches, regardless of which fine-grained permission that
operation might logically require. This is the observed state of the code today, as built.

## AccountResource payment-endpoint duplication

`AccountResource.getInvoicePayments` and `AccountResource.getPaymentsForAccount` implement closely
similar logic for retrieving an account's payments (one scoped through invoice payments, one direct),
and the author has flagged this in-code:

```
// STEPH should refactor code since very similar to @Path("/{accountId:" + UUID_PATTERN + "}/" + PAYMENTS)
```

(see the evidence map's cross-cutting section for the exact file and line). This is recorded here as a
smaller, evidence-bound maintenance note — a duplication the code itself already flags — rather than as
a defect requiring immediate remediation.
