# CLAUDE.md

## Project Overview

killbill — the core engine of Kill Bill, an open-source subscription billing and payments
platform. This repo (`org.kill-bill.billing:killbill`) is a multi-module Maven project implementing
the billing domain logic: subscriptions, invoicing, payments, entitlements, usage, tax/currency
conversion, and the REST API exposed to clients and plugins. It's designed to be run standalone or
embedded, and is highly modular so integrators can swap out or disable pieces they don't need.

## Tech Stack

- **Language / runtime** — Java, built with Maven (multi-module `pom.xml`, parent artifact
  `killbill-oss-parent`)
- **Framework** — Guice for DI, JAX-RS (Jersey) for the REST API, Jetty for the embedded server
- **Data layer** — MySQL/relational DB via JDBC (see `bin/db-helper`), `log4jdbc` for query logging
- **Tooling** — Maven, CircleCI (`.circleci/config.yml`), SpotBugs (`spotbugs-exclude.xml`), TestNG
  for tests
- **External deps** — this repo depends on sibling Kill Bill repos resolved from Maven
  (`killbill-api`, `killbill-plugin-api`, `killbill-commons`, `killbill-platform`,
  `killbill-client-java`, etc.) — CI clones matching branches of those repos when building non-master
  branches

## Project Structure

```
/killbill
├── CLAUDE.md                 # This file
├── README.md                 # Project overview (upstream Kill Bill README)
├── CHANGELOG.md              # Local task-tracking changelog (separate from upstream NEWS)
├── NEWS                      # Upstream Kill Bill release notes
├── pom.xml                   # Parent/aggregator POM, lists all modules
├── account/                  # Account entity, CRUD, and account-level data (emails, custom fields)
├── api/                      # Core Kill Bill API interfaces shared across modules
├── beatrix/                  # Integration-test module wiring all modules together end-to-end
├── catalog/                  # Product catalog: plans, phases, pricing rules (XML-defined)
├── currency/                 # Currency conversion rates and lookups
├── entitlement/              # Entitlement/access-control layer sitting above subscriptions
├── invoice/                  # Invoice generation, invoice items, proration logic
├── jaxrs/                    # JAX-RS REST resources — the public HTTP API (Kill Bill Client API)
├── junction/                 # Bridges billing (invoice) and entitlement (subscription) views
├── overdue/                  # Overdue/dunning state machine for unpaid accounts
├── payment/                  # Payment processing, payment plugins integration, refunds
├── profiles/                 # Deployable server assemblies
│   ├── killbill/              # Main Kill Bill server profile (WAR/executable assembly)
│   └── killpay/                # Payment-only server profile
├── subscription/             # Subscription lifecycle: create, change plan, cancel
├── tenant/                   # Multi-tenancy support (per-tenant config, catalogs, overrides)
├── usage/                    # Usage-based billing (metered/consumable usage tracking)
├── util/                     # Shared utilities (caching, config, email, template rendering, etc.)
├── bin/                       # Dev scripts: start-server, db-helper, clean-and-install, import-account
└── planning/
    ├── design/                # Design docs and specs
    ├── research/              # Research notes
    ├── plan/                  # Implementation plans
    └── to do/                 # Active and completed tasks
        └── completed/
```

## Task Management

Tasks live in `planning/to do/todo.md` with unique identifiers (`KILL-001`, `KILL-002`, etc.).

When picking up a task:
1. Read `todo.md` to identify the next open item
2. Read its task plan if one exists in `planning/plan/`
3. Implement following the plan
4. Mark complete in `todo.md`
5. Update `CHANGELOG.md` with what changed

Note: `CHANGELOG.md` is a local addition for this task-tracking workflow — it's separate from the
upstream `NEWS` file, which records official Kill Bill release history and should not be conflated
with it.

## Development Workflow

```bash
# Build all modules, skip tests (matches CI's default build step)
mvn -DskipTests=true clean install

# Run the full test suite (TestNG)
mvn test

# Start the server locally (profiles/killbill)
bin/start-server

# Clean local Maven repo state for this project + reinstall
bin/clean-and-install
```

## Research > Plan > Implement

**Never jump straight to coding.** Always:
1. **Research** — Understand the existing codebase and patterns
2. **Plan** — Write a task plan and verify with the user
3. **Implement** — Execute the plan, then verify it works

## Working Together

- Clarity over cleverness — the simple solution is usually correct
- Match existing patterns before introducing new ones
- When stuck: stop, step back, simplify, ask
