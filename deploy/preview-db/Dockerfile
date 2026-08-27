# Preview-only Postgres image for Kill Bill.
#
# Kill Bill ships plain SQL schema files per module (*/src/main/resources/.../ddl.sql)
# plus a PostgreSQL compatibility prelude (util/.../ddl-postgresql.sql) that defines the
# MySQL-only types/functions the module DDLs rely on (datetime, mediumtext, blob domains,
# last_insert_id(), etc). bin/db-helper concatenates these the same way for a `--driver
# postgres` install; we bake the same concatenation into /docker-entrypoint-initdb.d so the
# official postgres image applies it automatically on first boot (against the POSTGRES_DB
# database), in the same module order used by the top-level pom.xml's <modules> list.
#
# This file is a preview-deploy addition only; it does not change how Kill Bill is built
# or run anywhere else.
FROM postgres:16

COPY util/src/main/resources/org/killbill/billing/util/ddl-postgresql.sql /docker-entrypoint-initdb.d/00-compat.sql
COPY account/src/main/resources/org/killbill/billing/account/ddl.sql /docker-entrypoint-initdb.d/10-account.sql
COPY beatrix/src/main/resources/org/killbill/billing/beatrix/ddl.sql /docker-entrypoint-initdb.d/20-beatrix.sql
COPY catalog/src/main/resources/org/killbill/billing/catalog/ddl.sql /docker-entrypoint-initdb.d/30-catalog.sql
COPY subscription/src/main/resources/org/killbill/billing/subscription/ddl.sql /docker-entrypoint-initdb.d/40-subscription.sql
COPY entitlement/src/main/resources/org/killbill/billing/entitlement/ddl.sql /docker-entrypoint-initdb.d/50-entitlement.sql
COPY invoice/src/main/resources/org/killbill/billing/invoice/ddl.sql /docker-entrypoint-initdb.d/60-invoice.sql
COPY payment/src/main/resources/org/killbill/billing/payment/ddl.sql /docker-entrypoint-initdb.d/70-payment.sql
COPY usage/src/main/resources/org/killbill/billing/usage/ddl.sql /docker-entrypoint-initdb.d/80-usage.sql
COPY util/src/main/resources/org/killbill/billing/util/ddl.sql /docker-entrypoint-initdb.d/90-util.sql
COPY tenant/src/main/resources/org/killbill/billing/tenant/ddl.sql /docker-entrypoint-initdb.d/95-tenant.sql
