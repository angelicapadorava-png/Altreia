# Altreia — Architecture

**Status: Phase 0 — Architecture Lock (COMPLETE)**

This document is the locked architecture reference for the Altreia platform. It governs all implementation decisions from Phase 1 onward. Changes to this document should be deliberate and reviewed — it is the source of truth for platform boundaries, tenancy, entitlements, and product separation.

## 1. Platform Definition

Altreia is a multi-product business operating-system platform. It is not a collection of unrelated SaaS applications.

The platform supports multiple vertical business systems that share core infrastructure while keeping industry-specific workflows independent.

**Initial products:**
- Nail OS
- Car Rental OS

**Future products may include:** ResortFlow, Salon OS, Lash OS, Pet Grooming OS, and other niche business operating systems.

The architecture must allow future OS products to be added without rebuilding authentication, subscriptions, branding, customer identity, files, storage, admin tooling, and other shared infrastructure.

## 2. Core Architecture Principle

> Share infrastructure aggressively. Share business logic only after repetition proves that it should be shared.

We must not force Nail OS and Car Rental OS into generic abstractions merely because their concepts look similar.

Examples:
- Nail appointments and car reservations both appear on calendars, but their business logic is different.
- Nail services and rental charges both create revenue, but their financial models differ.
- Vehicles, rooms, equipment, and service resources may eventually share patterns, but should not be forced into one generic "Asset" model until actual products prove that abstraction useful.

## 3. Platform Structure

- **Altreia Admin** — private internal control center used by Altreia administrators.
- **Altreia Platform App** — the customer-facing application. One product-aware customer application, not a separate frontend deployment per OS.
- **Shared Platform Infrastructure** — reusable capabilities belonging to the platform rather than any individual product.
- **Product-Specific Domains** — Nail OS business logic, Car Rental OS business logic, future vertical logic.

## 4. High-Level System

```
ALTREIA
│
├── ALTREIA ADMIN
│   ├── Businesses
│   ├── Products
│   ├── Plans
│   ├── Subscriptions
│   ├── Entitlements
│   ├── Storage
│   ├── Users
│   ├── Support Controls
│   └── Audit/System Controls
│
└── ALTREIA PLATFORM APP
    ├── Authentication
    ├── Business Context
    ├── Product Router
    ├── Nail OS
    └── Car Rental OS
```

Later products plug into the same product router.

## 5. One Customer-Facing Frontend

One customer-facing application with product-aware routing. We will **not** create separate web apps per product (`nail-app`, `car-rental-app`, `resortflow-app`, ...).

```
platform-app
├── shared shell
├── shared UI
├── shared authentication
├── shared settings
├── shared branding
└── products
    ├── nail
    └── car-rental
```

The authenticated business's assigned product determines which product routes and modules are mounted.

## 6. Admin Application

A separate internal application. Admin users can: create businesses, assign products, view/change subscriptions, view/change storage allowance, manage entitlements, enable beta features, suspend/reactivate accounts, inspect account metadata, manage products/plans, review audit activity, assist with support cases.

Admin must never expose normal customer UI permissions.

**[Amendment A]** Admin is used by multiple internal Altreia accounts with data-driven roles (SUPER_ADMIN, PRODUCT_ADMIN, SALES, and future roles). Authorized administrators can create and manage Altreia OS products through explicit product grants. Sales users get an OWN-scoped sales CRM and commission dashboard inside Admin. See §35c.

## 7. Business Model

Every business account has: a business record, an owner, optional members, an assigned product, subscription status, product configuration, entitlements, branding, storage quota, settings, and audit history.

V1 assumes one primary product per business. The architecture must not preclude multi-product businesses later, but they are not required for V1.

## 8. Pricing Architecture

Base OS pricing direction (configurable, never hardcoded):
- First year: ₱600
- Renewal: ₱1,000/year

The database must support promotional pricing, grandfathered pricing, per-product pricing, different renewal prices, and monthly/yearly Connected Services pricing.

## 9. Base OS vs Connected Services

Two commercial layers:

**Base OS** — the business-management software; must remain useful on its own.
- Nail OS Base: clients, client profiles, services, internal calendar, manual appointments, revenue records, expenses, inventory, profitability, reports, documents, files, branding, settings.
- Car Rental OS Base: customers, fleet, drivers, internal reservations, internal calendar, payment/deposit records, expenses, maintenance, inspections, incidents, contracts, documents, reports, branding, settings.

**Connected Services** — recurring integrations/automation: public booking, public reservation requests, email sending (confirmations, reminders, follow-ups), online payment/deposit collection, payment gateway integrations, Google Calendar sync, customer portals, scheduled automations, SMS (later).

Connected Services are implemented through entitlements. Cancelling Connected must never destroy or corrupt Base OS data.

## 11. Non-Negotiable Dependency Rule

**No Base feature may require Connected Services in order to function.** This rule must be tested.

Examples: a manually created appointment/reservation must work without public booking; an invoice must exist even if email sending is disabled; a payment record must exist even if online payments are unavailable.

## 12. Product Configuration vs Entitlements vs Permissions

Three separate systems — never collapsed into one feature-flag system:

- **Product Configuration** — what exists in a product (e.g. Car Rental has fleet/drivers/inspections/maintenance; Nail does not).
- **Entitlements** — what a business currently has access to (e.g. `booking.public`, `email.transactional`, `payments.online`, `calendar.sync`). Sourced from plan grants, promos, manual overrides, beta access.
- **Permissions** — what an individual user is allowed to do (e.g. owner: `business.settings.write`, `users.manage`; staff: `appointments.write`, `clients.read`).

**[Amendment A]** A fourth, separate system: **Internal Altreia permissions** — what an Altreia team member may do in Admin (e.g. `products.create`, `sales.crm.own`). These are never mixed with customer permissions, entitlements, or product configuration. See §35c.

## 13. Entitlement Overrides

Admin can temporarily or permanently override entitlements:

```
business_id, feature_key, granted, reason, expires_at, created_by
```

Use cases: beta access, free trial, support workaround, temporary promotion, special customer agreement. The original plan configuration remains unchanged.

## 14. Authentication

Platform-level. Recommended: Better Auth or equivalent — secure sessions, business membership checks, role-based authorization, server-side authorization enforcement. Frontend checks are UX only; access control is enforced server-side.

**[Amendment A]** Two separate account realms: customer accounts (`users`, used in `apps/app`) and internal Altreia accounts (`internal_users`, used in `apps/admin`), with separate sessions and authorization and no realm switching. MFA is mandatory for internal accounts. See §35c.

## 15. Tenant Isolation

Every business is a tenant. All operational records must be isolated by `business_id`.

- **Control Database** — platform-level data.
- **Shared Operational Database** — operational business data with strict tenant ownership.

No route handler writes unrestricted raw tenant queries. Tenant data access goes through a tenant-aware data-access layer:

```
tenantData.forBusiness(authenticatedBusinessId)
```

Automated tests must specifically attempt cross-tenant access.

## 16. Future Database Scaling

V1 will not use one D1 database per business by default. The data-access layer must avoid assuming a single shared database forever — sharding, product-specific databases, dedicated large-customer databases, or tenant-per-database must remain possible later without rewriting the domain layer.

## 17. Database Separation

At least two conceptual database domains:

**Control Plane** (platform data): `users`, `businesses`, `business_members`, `products`, `plans`, `subscriptions`, `subscription_periods`, `entitlements`, `plan_entitlements`, `entitlement_overrides`, `business_branding`, `storage_usage`, `audit_logs`. Potential additions: `support_notes`, `feature_flags`, `product_settings`.

**[Amendment A]** Control plane also holds internal identity/RBAC (`internal_users`, `internal_roles`, `internal_permissions`, `internal_role_permissions`, `internal_user_roles`), `product_admin_grants`, `business_profiles`, `plan_prices`, `platform_payments`, Altreia sales CRM (`sales_leads`, `sales_notes`, `sales_attributions`), and commissions (`commission_rules`, `commission_ledger_entries`, `commission_payouts`, `commission_payout_items`). Altreia CRM data lives here; tenant operational data never does. See §35b and §35c.

**Operational Data** (business operations): shared minimal customer identity where appropriate, product-specific domain data, product financial data, file metadata references, operational history.

## 18. Shared Customer Identity

No enormous generic customer model. Only genuinely universal fields:

```
customer_id, business_id, first_name, last_name, email, phone, address, notes, status, created_at, updated_at
```

Optional shared systems: tags, activity timeline, file relationships.

## 19. Product-Specific Customer Extensions

Products extend the shared customer record with typed, indexed extensions rather than arbitrary JSON:
- Nail OS: `nail_client_profile` (preferences, nail notes, service preferences, service notes).
- Car Rental OS: `rental_customer_profile` (driver's license metadata, emergency contact, verification fields).

## 20–23. Files & Document System

Platform-level, shared across products. Categories (extensible per-product): IDs, contracts, invoices, receipts, payment proof, forms, photos, business logos, inspection images, before/after photos, documents, other.

**Storage:** Cloudflare R2, layout `businesses/{business_id}/...`, shared private bucket (no per-business bucket in V1). Files are never publicly exposed by default; downloads are authorized by the application with time-limited/signed access where appropriate.

**Storage allowance:** 500 MB per business, database-configurable (`quota_bytes`, not hardcoded). Admin can adjust it. Customer UI shows used/allowance/percentage/warning status (80%, 90%, 100%).

**Upload rules:** per-file size limits, allowed MIME/file types, image optimization/compression, upload validation, safe filenames, file ownership, audit history, secure deletion. Videos are out of scope for normal V1 business storage unless specifically required later.

## 24. Branding

Shared platform capability. Configurable per business: logo, business name, tagline, primary/accent color (HEX picker), light/dark preference, contact info, invoice/document branding, optional cover image. Branding influences navigation, buttons, calendar accents, documents, invoices, receipts, future emails, and public booking pages. Businesses can brand the experience but cannot redesign the UI structure.

## 25. UI Foundation

Shared: responsive desktop/mobile layout, navigation shell, mobile navigation, loaders/skeletons, toasts, modals, confirmation dialogs, unsaved-change warnings, empty states, accessibility basics, theme support, shared form controls/tables, search/filter patterns. The UI shell stays consistent across products.

## 26. Calendar Architecture

No generic appointment/reservation model. Shared: date utilities, month/week/day views, responsive layout, drag/drop utilities, timezone handling, controls, filters. Product logic stays independent (Nail appointment domain vs. Car Rental date-range reservation domain). Extract shared calendar behavior only after both products are implemented and duplication is proven.

## 27. Finance Architecture

No single generic financial object. Shared: money utilities, currency formatting, expense UI patterns, payment-method infrastructure, invoice rendering, report primitives. Product-specific finance differs freely (Nail: service revenue/material cost/profitability/ROI/payback; Car Rental: reservation charges/deposits/balance/damages/financing/fees).

## 28. Nail OS (Product-Specific)

Services & pricing/durations, appointments, availability, working hours, conflict detection, appointment statuses, nail inventory/consumables/hygiene items, cost-per-service, revenue, expenses, profitability, ROI, payback calculator, purchase recovery, customer photos/documents. Connected later: public booking, confirmation emails, reminders, online deposits, customer self-service.

## 29. Car Rental OS (Product-Specific)

Fleet (cars, motorcycles, scooters, SUVs, vans, pickups, trucks, minibuses, buses), drivers, inquiries, reservations, date-range availability, renter profile, pickup/drop-off/destination, deposits, payments, maintenance, inspections, incidents, financing, contracts, invoices, receipts, photos/documents.

Core workflow: `Inquiry → Review → Availability → Customer → Reservation → Contract/Invoice → Payment → Pickup → Return → Completed`

## 30–31. Deferred Generic Abstractions

No universal `Asset` system in V1 (Car Rental owns its vehicle/fleet model; extraction only after a product like ResortFlow proves it). No arbitrary "add any field" custom-field capability in V1; product schemas stay structured and typed. If limited custom fields become necessary later, use controlled product/entity-specific metadata, not a global EAV database.

## 32–34. OS Builder

A controlled internal OS Builder is part of the long-term strategy, but **will not be built before Nail OS V2 and Car Rental OS V2 prove the framework**. Initial product configuration lives in code plus database configuration.

Builder V1 (future) may eventually control: product name/code/icon, module registration, terminology, navigation, branding defaults, entitlement defaults, plan assignments, default permissions, dev/published state, clone-from-template.

The Builder will **never** contain: arbitrary coding, arbitrary database creation, generic workflow programming, drag-and-drop page building, custom JS execution, arbitrary logic engines.

Configuration split:
- **Database:** display names, icons, terminology, navigation visibility, default branding, plan mappings, entitlement definitions.
- **Code:** conflict detection, availability logic, pricing rules, profitability calculations, vehicle workflows, inspection logic, domain transitions, specialized reports. Business logic stays typed, tested, and version-controlled.

## 35. Repository Direction — LOCKED (§47 Item #1)

**Scope note:** this locks the repository structure for **Altreia Cloud** — the Cloudflare-hosted, multi-tenant SaaS platform (Altreia Admin + Altreia Platform App + product modules). It does not apply to, and does not change, any Google Sheets / Apps Script one-time systems, which remain separate, standalone deliverables outside this monorepo.

One Altreia monorepo. No independent application or repository per product.

```
altreia/
├── apps/
│   ├── admin/                    # Altreia Admin — internal control center
│   └── app/                      # Altreia Platform App — product-aware customer app
│
├── platform/                     # Shared platform infrastructure (genuinely platform-owned)
│   ├── database/                 # control + operational DB clients, tenant-aware data-access layer
│   ├── auth/                     # authentication, sessions
│   ├── tenancy/                  # business context, tenant isolation enforcement
│   ├── ui/                       # design system / shared UI components
│   ├── branding/
│   ├── files/                    # upload/storage infra (R2), file metadata, categories
│   ├── entitlements/
│   ├── permissions/
│   ├── billing/                  # products, plans, subscriptions
│   ├── audit/
│   ├── internal-auth/            # [Amendment A] internal accounts, internal RBAC, product grants
│   ├── sales/                    # [Amendment A] Altreia sales CRM + attribution
│   ├── commissions/              # [Amendment A] rules, ledger, payouts
│   └── utils/                    # shared utilities (money, dates, formatting, validation)
│
├── products/                     # Isolated product modules — one per OS
│   ├── nail-tech-os/
│   │   ├── domain/
│   │   ├── data/
│   │   ├── ui/
│   │   └── tests/
│   ├── car-rental-os/
│   │   ├── domain/
│   │   ├── data/
│   │   ├── ui/
│   │   └── tests/
│   └── resortflow/                # future product, same shape, added when built
│       ├── domain/
│       ├── data/
│       ├── ui/
│       └── tests/
│
├── connected/                     # Connected Services modules — separate from Base product logic
│   ├── email/
│   ├── payments/
│   ├── booking/                   # public booking/reservation + automation infra
│   └── calendar-sync/
│
├── workers/
│   ├── api/
│   └── jobs/
│
├── database/
│   ├── control/
│   │   └── migrations/
│   └── operational/
│       └── migrations/
│
└── tests/
    ├── integration/
    └── security/
```

**Binding rules locked with this structure:**

- Product-specific business logic stays inside its `products/<product>/` module. It is only extracted into `platform/` after genuine repetition across two or more products proves the abstraction (per the Phase 0 principle — see below).
- `connected/*` modules implement Connected Services only. A `products/*` module may call into `connected/*` through an entitlement-gated interface, but must remain fully functional with any `connected/*` module absent or disabled (Base OS never depends on Connected Services — §11).
- No product gets its own app, repo, or deployment. All products are mounted into `apps/app/` via the product router (§5), each as an isolated module under `products/`.
- Exact file/folder names inside each module may evolve during setup, but the top-level boundaries (`apps/` vs `platform/` vs `products/` vs `connected/`) are locked and must remain.

Preserves the Phase 0 principle: *"Share infrastructure aggressively. Share logic only after repetition."*

## 35a. Cloudflare Resources — LOCKED (§47 Item #2)

**Scope note:** this locks the Cloudflare resource architecture for **Altreia Cloud** only, not the Google Sheets / Apps Script one-time systems.

**Frontend deployments — two separate surfaces**
- **Altreia Admin** (`apps/admin`) — its own Cloudflare Pages/Workers deployment.
- **Altreia Customer App** (`apps/app`) — its own Cloudflare Pages/Workers deployment.
- Both are separate deployment surfaces but consume the same shared backend/API architecture (`workers/api`). Two deployments, one API.

**Data**
- **Control D1** — one database for platform/control-plane data: users, businesses, memberships, subscriptions, entitlements, platform configuration, audit information (§17).
- **Shared Operational D1** — one database for V1 tenant/product operational data across all products. No per-product operational databases in V1.
  - Every operational record is tenant-scoped via `business_id`.
  - All access is enforced server-side through the tenant-aware data-access layer (§15–16) — no route handler queries the operational D1 directly.
  - The data-access layer is the seam that allows operational storage to later be split or sharded (per product, per large customer, etc.) without redesigning the control plane.
- **R2** — one shared private bucket, `businesses/{business_id}/...` layout (§21), no per-business buckets in V1.

**Admin protection — both layers required**
- **Cloudflare Access / Zero Trust** in front of `apps/admin`, as an additional network-level boundary.
- **Application-level authentication and authorization** inside `apps/admin`, independent of Access.
- **[Amendment A]** Access policies must admit every internal role that uses Admin, including Sales. Access still only adds a boundary; internal permissions decide what each person can do.
- Cloudflare Access is additive, not a substitute: application permissions (roles, entitlements) are enforced regardless of whether Access is reachable, misconfigured, or bypassed at the edge.

**Compute, jobs, and supporting resources**
- **API Worker** (`workers/api`) — the single backend API, shared by both frontend deployments.
- **Background/Jobs Worker** (`workers/jobs`) — scheduled and async work (Connected Services automation, reminders, cleanup).
- **Cloudflare Queues** — decouples `workers/api` from `workers/jobs` for async work (email, webhooks, reminders, file post-processing).
- **Cron Triggers** — drives scheduled Connected Services jobs.
- **Turnstile** — bot/abuse protection where appropriate (public booking forms, login, other public-facing endpoints).
- **Wrangler/environment secrets** — no secrets or production credentials in source (§39).
- **Isolated development / staging / production resources** — separate D1, R2, KV, Queues, and Worker environments per environment tier (§39).

**KV — deferred, not locked**
- KV is **not** locked as the session-token/auth-persistence store. Where and how authentication sessions are persisted is deferred to the authentication implementation architecture (§14, to be specified with Phase 2).
- KV **may** be used now for appropriate low-latency cache/config/rate-limiting use cases where eventual consistency is acceptable (e.g. entitlement cache, rate-limit counters, config lookups) — just not as the source of truth for sessions.

**Base vs. Connected dependency rule (reaffirmed at the infrastructure level)**
- Base product functionality must remain operational even if Queues, Cron jobs, email, payment integrations, or other Connected Services are unavailable or unreachable.
- Connected Services (anything routed through `connected/*` modules, Queues, Cron-driven jobs, third-party integrations) must never become a runtime dependency for Base functionality — consistent with §11 and §9.

## 35b. Phase 1 Database Schema — LOCKED (§47 Item #3)

**Scope note:** Altreia Cloud only. Phase 1 provisions Control D1 fully and provisions/migrates Operational D1 with only tenancy scaffolding — no Nail Tech OS, Car Rental OS, or ResortFlow domain tables. Those arrive with their respective product phases (7, 8, future).

**Conventions**
- All application-generated primary IDs are **ULIDs stored as TEXT**. ULIDs are chosen for sortability and distributed-friendly generation, not for secrecy — tenant isolation is enforced by server-side authorization and `business_id` scoping, never by IDs being hard to guess.
- All timestamps are **Unix epoch INTEGER** (seconds).
- Every tenant-owned table carries `business_id` and is only ever accessed through the tenant-aware data-access layer (§15–16).

### Control D1

```sql
users
  id                    TEXT PK                 -- ULID
  email                 TEXT NOT NULL UNIQUE
  password_hash         TEXT
  created_at            INTEGER NOT NULL
  updated_at            INTEGER NOT NULL

businesses
  id                    TEXT PK                 -- ULID
  name                  TEXT NOT NULL            -- canonical business name (see business_profiles note)
  product_code          TEXT NOT NULL REFERENCES products(code)
  status                TEXT NOT NULL            -- ACTIVE | SUSPENDED | ... (platform/admin state — independent of billing)
  created_at            INTEGER NOT NULL
  updated_at            INTEGER NOT NULL
  CHECK (status IN ('ACTIVE','SUSPENDED','CLOSED'))

-- No owner_user_id column. Ownership is derived from business_members
-- (role = 'OWNER'), not duplicated on businesses. This allows ownership
-- transfer and, later, multiple owners without redesigning this table.

business_members
  id                    TEXT PK                 -- ULID
  business_id           TEXT NOT NULL REFERENCES businesses(id)
  user_id               TEXT NOT NULL REFERENCES users(id)
  role                  TEXT NOT NULL            -- OWNER | STAFF | ...
  created_at            INTEGER NOT NULL
  UNIQUE (business_id, user_id)
  INDEX idx_business_members_business (business_id)
  INDEX idx_business_members_user (user_id)
-- Authoritative business <-> user relationship. A business must have
-- at least one member with role = 'OWNER' (enforced in application
-- logic at write time in Phase 2+; not expressible as a single CHECK
-- across rows in SQLite).

business_profiles
  business_id           TEXT PK REFERENCES businesses(id)
  display_name          TEXT                     -- optional override for public-facing display; falls back to businesses.name if null
  contact_name          TEXT                     -- [Amendment A] main contact person
  email                 TEXT
  phone                 TEXT
  address_line_1        TEXT
  address_line_2        TEXT
  city                  TEXT
  region                TEXT
  postal_code           TEXT
  country_code          TEXT
  updated_at            INTEGER NOT NULL

business_branding
  business_id           TEXT PK REFERENCES businesses(id)
  logo_file_id          TEXT
  tagline               TEXT
  primary_color         TEXT
  accent_color          TEXT
  theme_preference      TEXT                     -- LIGHT | DARK | SYSTEM
  updated_at            INTEGER NOT NULL

products
  code                  TEXT PK                  -- 'nail_tech_os', 'car_rental_os', 'resortflow'
  name                  TEXT NOT NULL
  is_published          INTEGER NOT NULL DEFAULT 0
  created_by            TEXT REFERENCES internal_users(id)   -- [Amendment A] provenance only, never authorization
  created_at            INTEGER NOT NULL                     -- [Amendment A]
  updated_at            INTEGER NOT NULL                     -- [Amendment A]

plans
  id                    TEXT PK                  -- ULID
  product_code          TEXT NOT NULL REFERENCES products(code)
  name                  TEXT NOT NULL
  is_active             INTEGER NOT NULL DEFAULT 1
  created_at            INTEGER NOT NULL
  updated_at            INTEGER NOT NULL
  INDEX idx_plans_product (product_code)
-- Plans define product capability/tier only. No pricing fields here.

plan_prices
  id                    TEXT PK                  -- ULID
  plan_id               TEXT NOT NULL REFERENCES plans(id)
  billing_type          TEXT NOT NULL            -- MONTHLY | YEARLY | LIFETIME | FOUNDING | ...
  amount                INTEGER NOT NULL         -- minor units (centavos)
  currency              TEXT NOT NULL DEFAULT 'PHP'
  introductory_amount   INTEGER
  introductory_periods  INTEGER
  is_active             INTEGER NOT NULL DEFAULT 1
  created_at            INTEGER NOT NULL
  updated_at            INTEGER NOT NULL
  INDEX idx_plan_prices_plan (plan_id)
-- Billing/pricing is fully decoupled from plan capability. New billing
-- models (monthly, yearly, lifetime, founding, introductory) are added
-- as rows here without altering `plans`.

subscriptions
  id                    TEXT PK                  -- ULID
  business_id           TEXT NOT NULL REFERENCES businesses(id)
  plan_id               TEXT NOT NULL REFERENCES plans(id)
  plan_price_id         TEXT NOT NULL REFERENCES plan_prices(id)
  status                TEXT NOT NULL            -- TRIAL|ACTIVE|PAST_DUE|EXPIRED|SUSPENDED|CANCELLED
  current_period_start  INTEGER NOT NULL
  current_period_end    INTEGER NOT NULL
  created_at            INTEGER NOT NULL
  updated_at            INTEGER NOT NULL
  INDEX idx_subscriptions_business (business_id)
  CHECK (status IN ('TRIAL','ACTIVE','PAST_DUE','EXPIRED','SUSPENDED','CANCELLED'))
-- subscriptions.status is billing/subscription state, independent of
-- businesses.status (platform/admin state). An ACTIVE subscription
-- does not prevent Admin from independently suspending a business,
-- and a SUSPENDED business does not require touching billing state.

subscription_periods
  id                    TEXT PK                  -- ULID
  subscription_id       TEXT NOT NULL REFERENCES subscriptions(id)
  period_start          INTEGER NOT NULL
  period_end            INTEGER NOT NULL
  amount_paid           INTEGER
  created_at            INTEGER NOT NULL
  INDEX idx_subscription_periods_subscription (subscription_id)

plan_entitlements
  id                    TEXT PK                  -- ULID
  plan_id               TEXT NOT NULL REFERENCES plans(id)
  feature_key           TEXT NOT NULL            -- 'booking.public', 'email.transactional', ...
  UNIQUE (plan_id, feature_key)
  INDEX idx_plan_entitlements_plan (plan_id)

entitlement_overrides
  id                    TEXT PK                  -- ULID
  business_id           TEXT NOT NULL REFERENCES businesses(id)
  feature_key           TEXT NOT NULL
  granted               INTEGER NOT NULL         -- 0/1
  reason                TEXT
  expires_at            INTEGER
  created_by            TEXT NOT NULL REFERENCES internal_users(id)   -- [Amendment A] was users(id)
  created_at            INTEGER NOT NULL
  INDEX idx_entitlement_overrides_business (business_id)

storage_usage
  business_id           TEXT PK REFERENCES businesses(id)
  quota_bytes           INTEGER NOT NULL DEFAULT 524288000   -- 500MB default, admin-configurable
  used_bytes            INTEGER NOT NULL DEFAULT 0
  updated_at            INTEGER NOT NULL

audit_logs
  id                    TEXT PK                  -- ULID
  business_id           TEXT REFERENCES businesses(id)   -- nullable for platform-level events
  actor_type            TEXT NOT NULL            -- [Amendment A] CUSTOMER_USER | INTERNAL_USER | SYSTEM
  actor_id              TEXT                     -- [Amendment A] users.id or internal_users.id per actor_type; null for SYSTEM
  action                TEXT NOT NULL
  target_type           TEXT
  target_id             TEXT
  metadata              TEXT                     -- JSON
  created_at            INTEGER NOT NULL
  INDEX idx_audit_logs_business (business_id)
  INDEX idx_audit_logs_actor (actor_type, actor_id)
  CHECK (actor_type IN ('CUSTOMER_USER','INTERNAL_USER','SYSTEM'))
-- [Amendment A] replaces actor_user_id so both account realms and
-- automated jobs can be recorded as actors.
```

**[Amendment A] additions to Control D1:** `business_profiles` gains `contact_name TEXT` (the business's main contact person). The internal identity/RBAC tables in §35c are also part of the Phase 1 Control D1 schema. The remaining §35c tables are designed now but created by migrations in their own phases.

**Canonical naming, defined:**
- `businesses.name` is the canonical business name (system of record, used in control-plane logic, billing, admin listings).
- `business_profiles.display_name` is optional and display-specific only — used when a business wants a different public-facing name than its canonical `businesses.name`. If null, display falls back to `businesses.name`.
- `business_branding` holds no name field at all — it is presentation-only (logo, tagline, colors, theme).

### Operational D1 (Phase 1 scaffolding only)

```sql
schema_migrations
  version               TEXT PK
  checksum              TEXT NOT NULL
  applied_at            INTEGER NOT NULL
```

Control D1 carries an identical `schema_migrations` table (see §38a).

No product domain tables. Phase 1 deliverables here are the provisioned/migrated database and the tenant-aware access layer (`tenantData.forBusiness(business_id)`) plus its cross-tenant access test suite — not product schemas, which belong to Phases 6–8.

**Concept boundary (reaffirmed):** `plan_entitlements`/`entitlement_overrides` (access), `plans`/`plan_prices` (billing), and `business_members.role` + a future `permissions` layer (what a member can do) are three distinct systems and are not collapsed, per §12. Internal Altreia permissions (§35c) are a fourth, separate system.

## 35c. Phase 0 Amendment A — Internal Accounts, Product Management, Sales Attribution & Commissions — LOCKED

**Scope note:** Altreia Cloud only. This is an explicit Phase 0 amendment approved after §47 Items #1–#5 were locked. Where it changes a locked section, that section carries an `[Amendment A]` marker.

### A1. Two separate account realms

| | Customer realm | Internal Altreia realm |
|---|---|---|
| Who | People who use a business's OS | Altreia team: platform operators, product builders, sales |
| Identity table | `users` | `internal_users` |
| Authority from | `business_members.role` (+ future customer permissions) | internal roles, permissions, and product grants |
| App | `apps/app` | `apps/admin` |

- The realms never share a table, role, permission, or session.
- The customer API rejects internal sessions; the admin API rejects customer sessions. Tests must attempt both crossings.
- One person who is both an Altreia team member and an Altreia customer has two separate accounts. There is no realm switching.
- A customer account can never be promoted into an internal account.
- Support impersonation (staff acting as a customer business) is not part of this amendment. If ever needed, it is a separate, explicit, audited feature.

### A2. Internal roles and permissions (data-driven RBAC)

- Roles are data. Seeded system roles: `SUPER_ADMIN`, `PRODUCT_ADMIN`, `SALES`. New roles and permissions can be added later without a schema change.
- Application code checks **permissions**, never role names. No individual person's name appears in any role or rule.
- Each permission has a scope:
  - **GLOBAL** — applies everywhere.
  - **PRODUCT** — applies only to products where the user holds an active `product_admin_grant`.
  - **OWN** — applies only to records attributed to that user.
- A request is allowed only when one of the user's roles grants the permission **and** its scope is satisfied.

**Seeded role defaults**

| Permission (examples) | Scope | SUPER_ADMIN | PRODUCT_ADMIN | SALES |
|---|---|---|---|---|
| `internal_users.manage`, `internal_roles.manage` | GLOBAL | ✓ | | |
| `platform.config.sensitive.write` | GLOBAL | ✓ | | |
| `products.create` | GLOBAL | ✓ | ✓ | |
| `product.config.write`, `product.plans.write` | PRODUCT | ✓ | ✓ (granted products) | |
| `product.grants.manage` | PRODUCT | ✓ | ✓ (as PRODUCT_OWNER) | |
| `businesses.read`, `businesses.suspend` | PRODUCT | ✓ | ✓ (granted products) | |
| `entitlements.override`, `subscriptions.manage` | PRODUCT | ✓ | ✓ (granted products) | |
| `platform_payments.record` | GLOBAL | ✓ | | |
| `sales.attribution.manage` | GLOBAL | ✓ | | |
| `commission_rules.manage` | GLOBAL | ✓ | | |
| `payouts.manage`, `payouts.approve` | GLOBAL | ✓ | | |
| `commissions.ledger.read.all` | GLOBAL | ✓ | | |
| `sales.crm.own` (leads, clients, notes) | OWN | | | ✓ |
| `commissions.ledger.read.own`, `payouts.read.own` | OWN | | | ✓ |
| `audit.read` | GLOBAL / PRODUCT | ✓ | ✓ (granted products) | |
| `platform.logs.read.production` (added by §35e) | GLOBAL | ✓ | | never |

- **PRODUCT_ADMIN does not automatically receive financial or sales-management authority**: recording platform payments, managing sales attribution, managing commission rules, and managing/approving payouts are separate permissions held by SUPER_ADMIN by default. An individual PRODUCT_ADMIN may be explicitly granted any of them later. PRODUCT_ADMIN is never equivalent to SUPER_ADMIN.

**Guardrails**
- Only a SUPER_ADMIN can grant SUPER_ADMIN. The last active SUPER_ADMIN cannot be removed or disabled.
- Nobody can change their own roles, grants, or attribution.
- Sales users cannot attribute clients to themselves; attribution is set by a holder of `sales.attribution.manage`.
- Nobody can approve a payout to themselves. Payout creation and approval should be done by different people where staffing allows.
- Internal accounts require MFA (method decided with the Phase 2 authentication design).

### A3. Product management

- Any holder of `products.create` can create a new Altreia OS product using the same internal accounts, admin app, and permission system. New products need no new authentication or admin infrastructure.
- `products.created_by` records provenance only. It is never used for authorization.
- Authority over a product comes from `product_admin_grants` (`PRODUCT_OWNER | PRODUCT_MANAGER | PRODUCT_VIEWER`). Creating a product automatically grants its creator `PRODUCT_OWNER`. Multiple administrators can hold grants on one product.

### A4. Altreia CRM data vs. tenant operational data

Two different kinds of customer information exist, and the boundary between them is strict:

| Altreia CRM / customer-relationship data | Tenant product / operational data |
|---|---|
| Altreia's relationship **with** a business | The business's own private operations |
| Lives in **Control D1** | Lives in **Operational D1** (and tenant files in R2) |
| Business name, contact person, business email/phone, product/OS, plan, lead/customer status, date acquired, sales notes/follow-ups, subscription status, payments made to Altreia, commission | Nail clients, appointments, service history, inventory, the business's financial books, renters, reservations, driver info, incidents, operational payments, uploaded operational files |
| Visible to authorized internal users, and to Sales for **their own** attributed leads/clients | Never visible to Sales. Internal access only through a future explicit, audited support feature |

- Internal roles, including Sales, have **no access path** to Operational D1 or tenant files. The Sales dashboard reads Control D1 only.
- Phase 6's "no generic CRM" refers to tenant customer records inside a product. Altreia's own lightweight sales CRM described here is a separate, platform-level concern.

### A5. Sales CRM and attribution

- Attribution is platform-level and attached to the control-plane business, so it works across every product (Nail Tech OS, Car Rental OS, ResortFlow, and future products).
- A Sales user works in a lightweight, OWN-scoped internal CRM inside `apps/admin` (no third frontend): their leads, their clients, contacts, status, notes and follow-ups, subscription status, qualifying Altreia revenue, commission, and payout history.
- Leads exist before a business account does. When a lead converts, it is linked to the new business and attribution is recorded.
- Attribution history is never overwritten: changing it closes the current row and opens a new one.
- Split attribution between multiple reps is supported by `share_bps` (UI can come later).
- Adding another salesperson later only requires creating a new internal account with the SALES role and giving them their own attributions and, if needed, commission rules.

### A6. Commission engine

**What earns commission:** only a SUCCEEDED `platform_payments` row of type PAYMENT — a customer paying **Altreia** for its subscription. Creating a lead or a business account never earns commission. Payments a business's own clients make to that business (operational data) never earn commission.

**Rules**
- Support percentage and fixed amounts, one-time and recurring (optionally capped by `max_periods`), product-specific and plan-specific rules, rep-specific rules, and effective dates.
- Rule selection uses the rule in effect when the payment occurred, most specific first: rep + plan → rep + product → rep → plan → product → global default.
- Once a rule has produced a ledger entry it is not edited; changes close the rule (`effective_to`) and create a new one.
- `hold_days` defaults to **0**, so under a zero-day rule a successful qualifying payment creates commission that is immediately earned and available. Any rule may set a hold (7, 14, 30, …) without schema changes.

**Ledger (append-only)**
- Each entry copies the rule, rate/fixed amount, share, basis amount, product, and plan used at the time. Historical commission is never recalculated from a rep's current rate.
- Calculation: PERCENTAGE = `basis × rate_bps / 10000 × share_bps / 10000`; FIXED = `fixed_amount × share_bps / 10000`. Integer minor units with one documented rounding rule.
- Refunds and chargebacks add a negative REVERSAL entry linked to the original, per the rule's `reversal_policy`. Already-paid amounts carry into the next payout; paid payouts are never modified.
- Corrections are MANUAL_ADJUSTMENT entries with a required reason, permission-gated and audited.
- `UNIQUE (payment_id, sales_user_id, entry_type)` prevents a retried job or duplicate webhook from paying twice.
- No UPDATE or DELETE on ledger entries or paid payouts in the data layer, backed by database triggers that reject them.

**Lifecycle and displayed states** (derived from data, not a stored status)
```
Lead/client attributed to Sales user        (no money)
  → Altreia payment SUCCEEDED
  → EARNED ledger entry written
  → PENDING while now < payable_after        (zero time under a 0-day hold)
  → AVAILABLE (earned, not yet paid)
  → included in a payout: DRAFT → APPROVED → PAID
Refund/chargeback at any point → REVERSAL entry
```
Earned/available is always shown separately from PAID.

### A7. Tables (Control D1)

```sql
-- Phase 1 schema -------------------------------------------------------

internal_users
  id                TEXT PK                  -- ULID
  email             TEXT NOT NULL UNIQUE
  password_hash     TEXT
  display_name      TEXT NOT NULL
  status            TEXT NOT NULL            -- ACTIVE | DISABLED
  created_at        INTEGER NOT NULL
  updated_at        INTEGER NOT NULL

internal_roles
  id                TEXT PK
  code              TEXT NOT NULL UNIQUE     -- SUPER_ADMIN, PRODUCT_ADMIN, SALES, ...
  name              TEXT NOT NULL
  is_system         INTEGER NOT NULL DEFAULT 0
  created_at        INTEGER NOT NULL

internal_permissions
  code              TEXT PK
  description       TEXT NOT NULL
  scope_type        TEXT NOT NULL            -- GLOBAL | PRODUCT | OWN

internal_role_permissions
  role_id           TEXT NOT NULL REFERENCES internal_roles(id)
  permission_code   TEXT NOT NULL REFERENCES internal_permissions(code)
  PRIMARY KEY (role_id, permission_code)

internal_user_roles
  internal_user_id  TEXT NOT NULL REFERENCES internal_users(id)
  role_id           TEXT NOT NULL REFERENCES internal_roles(id)
  granted_by        TEXT REFERENCES internal_users(id)   -- null only for the bootstrap SUPER_ADMIN
  created_at        INTEGER NOT NULL
  PRIMARY KEY (internal_user_id, role_id)
  INDEX (role_id)

-- Phase 3 --------------------------------------------------------------

product_admin_grants
  id                TEXT PK
  product_code      TEXT NOT NULL REFERENCES products(code)
  internal_user_id  TEXT NOT NULL REFERENCES internal_users(id)
  access_level      TEXT NOT NULL            -- PRODUCT_OWNER | PRODUCT_MANAGER | PRODUCT_VIEWER
  granted_by        TEXT NOT NULL REFERENCES internal_users(id)
  created_at        INTEGER NOT NULL
  revoked_at        INTEGER
  INDEX (internal_user_id), INDEX (product_code)
  UNIQUE INDEX (product_code, internal_user_id) WHERE revoked_at IS NULL

platform_payments
  id                     TEXT PK
  business_id            TEXT NOT NULL REFERENCES businesses(id)
  subscription_id        TEXT NOT NULL REFERENCES subscriptions(id)
  subscription_period_id TEXT REFERENCES subscription_periods(id)
  type                   TEXT NOT NULL      -- PAYMENT | REFUND | CHARGEBACK
  related_payment_id     TEXT REFERENCES platform_payments(id)
  amount                 INTEGER NOT NULL   -- minor units, positive; type gives direction
  currency               TEXT NOT NULL
  status                 TEXT NOT NULL      -- PENDING | SUCCEEDED | FAILED
  source                 TEXT NOT NULL      -- MANUAL | PROVIDER
  provider_ref           TEXT               -- unique when present
  recorded_by            TEXT REFERENCES internal_users(id)
  occurred_at            INTEGER NOT NULL
  created_at             INTEGER NOT NULL
  INDEX (business_id), INDEX (subscription_id)
  UNIQUE INDEX (provider_ref) WHERE provider_ref IS NOT NULL

sales_leads
  id                TEXT PK
  sales_user_id     TEXT NOT NULL REFERENCES internal_users(id)
  business_name     TEXT NOT NULL
  contact_name      TEXT
  email             TEXT
  phone             TEXT
  product_code      TEXT REFERENCES products(code)   -- product of interest
  status            TEXT NOT NULL            -- NEW | CONTACTED | TRIAL | WON | LOST
  converted_business_id TEXT REFERENCES businesses(id)
  created_at        INTEGER NOT NULL
  updated_at        INTEGER NOT NULL
  INDEX (sales_user_id), INDEX (converted_business_id)

sales_notes
  id                TEXT PK
  author_id         TEXT NOT NULL REFERENCES internal_users(id)
  lead_id           TEXT REFERENCES sales_leads(id)
  business_id       TEXT REFERENCES businesses(id)
  body              TEXT NOT NULL
  follow_up_at      INTEGER
  created_at        INTEGER NOT NULL
  INDEX (lead_id), INDEX (business_id), INDEX (author_id, follow_up_at)
  CHECK (lead_id IS NOT NULL OR business_id IS NOT NULL)

sales_attributions
  id                TEXT PK
  business_id       TEXT NOT NULL REFERENCES businesses(id)
  sales_user_id     TEXT NOT NULL REFERENCES internal_users(id)
  lead_id           TEXT REFERENCES sales_leads(id)
  share_bps         INTEGER NOT NULL DEFAULT 10000
  effective_from    INTEGER NOT NULL
  effective_to      INTEGER
  attributed_by     TEXT NOT NULL REFERENCES internal_users(id)
  reason            TEXT
  created_at        INTEGER NOT NULL
  INDEX (sales_user_id), INDEX (business_id)
  -- active shares per business must total <= 10000 (checked at write time)

-- Sales & Commissions phase --------------------------------------------

commission_rules
  id                TEXT PK
  name              TEXT NOT NULL
  sales_user_id     TEXT REFERENCES internal_users(id)
  product_code      TEXT REFERENCES products(code)
  plan_id           TEXT REFERENCES plans(id)
  calc_type         TEXT NOT NULL            -- PERCENTAGE | FIXED
  rate_bps          INTEGER
  fixed_amount      INTEGER
  currency          TEXT
  recurrence        TEXT NOT NULL            -- ONE_TIME | RECURRING
  max_periods       INTEGER
  hold_days         INTEGER NOT NULL DEFAULT 0
  reversal_policy   TEXT NOT NULL            -- FULL_ON_REFUND | PRORATED | NONE_AFTER_HOLD
  effective_from    INTEGER NOT NULL
  effective_to      INTEGER
  created_by        TEXT NOT NULL REFERENCES internal_users(id)
  created_at        INTEGER NOT NULL
  CHECK ((calc_type='PERCENTAGE' AND rate_bps IS NOT NULL) OR (calc_type='FIXED' AND fixed_amount IS NOT NULL))

commission_ledger_entries
  id                TEXT PK
  sales_user_id     TEXT NOT NULL REFERENCES internal_users(id)
  business_id       TEXT NOT NULL REFERENCES businesses(id)
  payment_id        TEXT REFERENCES platform_payments(id)    -- null only for MANUAL_ADJUSTMENT
  attribution_id    TEXT REFERENCES sales_attributions(id)
  rule_id           TEXT REFERENCES commission_rules(id)     -- null only for MANUAL_ADJUSTMENT
  entry_type        TEXT NOT NULL            -- EARNED | REVERSAL | MANUAL_ADJUSTMENT
  reverses_entry_id TEXT REFERENCES commission_ledger_entries(id)
  product_code      TEXT NOT NULL            -- snapshot
  plan_id           TEXT                     -- snapshot
  basis_amount      INTEGER NOT NULL         -- snapshot
  calc_type         TEXT                     -- snapshot
  rate_bps          INTEGER                  -- snapshot
  fixed_amount      INTEGER                  -- snapshot
  share_bps         INTEGER NOT NULL         -- snapshot
  amount            INTEGER NOT NULL         -- signed
  currency          TEXT NOT NULL
  payable_after     INTEGER NOT NULL
  reason            TEXT                     -- required for MANUAL_ADJUSTMENT
  created_by        TEXT REFERENCES internal_users(id)       -- null = system
  created_at        INTEGER NOT NULL
  UNIQUE (payment_id, sales_user_id, entry_type)
  INDEX (sales_user_id, created_at), INDEX (business_id)

commission_payouts
  id                TEXT PK
  sales_user_id     TEXT NOT NULL REFERENCES internal_users(id)
  total_amount      INTEGER NOT NULL
  currency          TEXT NOT NULL
  status            TEXT NOT NULL            -- DRAFT | APPROVED | PAID | VOID
  approved_by       TEXT REFERENCES internal_users(id)
  approved_at       INTEGER
  paid_at           INTEGER
  payment_reference TEXT
  created_by        TEXT NOT NULL REFERENCES internal_users(id)
  created_at        INTEGER NOT NULL
  INDEX (sales_user_id)
  CHECK (approved_by IS NULL OR approved_by <> sales_user_id)

commission_payout_items
  payout_id         TEXT NOT NULL REFERENCES commission_payouts(id)
  ledger_entry_id   TEXT NOT NULL UNIQUE REFERENCES commission_ledger_entries(id)
  PRIMARY KEY (payout_id, ledger_entry_id)
```

### A8. Security additions

- Realm separation enforced by separate tables, session types, and middleware, with cross-realm tests.
- MFA mandatory for all internal accounts.
- Cloudflare Access policies for `apps/admin` cover every internal role, including Sales, and remain additive to application authorization (§35a).
- Sales users have no access path to Operational D1 or tenant files; CRM data they see is limited to their own attributed leads/clients.
- Manually recorded platform payments create commission, so recording is a separate permission, audited, and the recorder should not be the rep attributed to that business (enforced where practical).
- Provider payment webhooks (Phase 13) must be signature-verified before a payment can become SUCCEEDED.
- Payee bank/tax details are not stored in V1; payouts keep only an external `payment_reference`. If stored later, they are sensitive data (§36).
- All changes to internal users, roles, grants, leads, attributions, platform payments, commission rules, manual adjustments, and payouts are written to `audit_logs` with `actor_type = INTERNAL_USER`.

## 35d. API Response & Error Conventions — LOCKED (§47 Item #6)

**Scope note:** Altreia Cloud only. Applies to shared platform APIs and product APIs alike.

**Versioning and realms**
- Customer APIs live under `/v1/`. Internal Altreia APIs live under `/admin/v1/`.
- Session realms stay strictly separate (§35c A1). A session from the wrong realm gets **401 `INVALID_SESSION_REALM`**, and the response never reveals that the other realm exists.

**Field names and values**
- JSON field names are `snake_case` everywhere, platform and product APIs alike.
- IDs are ULID strings; timestamps are Unix epoch integers (§35b).
- Money is always integer minor units plus an explicit currency code, e.g. `{ "amount": 60000, "currency": "PHP" }`. Floating-point values are never used for stored or calculated money.

**Success responses**
```json
{ "data": { ... } }
{ "data": [ ... ], "page": { "next_cursor": "01J...", "has_more": true } }
```
- Lists use cursor pagination. Default page size 25, maximum 100.
- Offset pagination is not the default for large, changing datasets.

**Error responses**
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message",
    "details": {},
    "request_id": "ULID"
  }
}
```
- This custom Altreia envelope is the V1 error format. RFC 9457 is not the primary format.
- Application and frontend behavior depends only on the stable `code`, never on `message`.
- `details` is optional, structured data (e.g. per-field validation errors).
- Errors never expose stack traces, SQL/database errors, provider internals, secrets, or sensitive submitted values.

**HTTP status mapping**

| Status | Used for |
|---|---|
| 400 | Malformed request |
| 401 | Not authenticated, or `INVALID_SESSION_REALM` |
| 403 | Authenticated but not permitted (incl. suspended business) |
| 403 + `ENTITLEMENT_REQUIRED` | Feature not included in the business's entitlements |
| 403 + `SUBSCRIPTION_READ_ONLY` | Write attempted on an expired Base subscription (§41) |
| 404 | Not found, or hidden because revealing it would leak unauthorized information |
| 409 | Conflict (duplicate, state-transition clash) |
| 422 | Validation failed |
| 429 | Rate limited, with `Retry-After` |
| 500 / 503 | Server error / dependency unavailable |

**Authorization and scope filtering**
- All authorization and scope filtering happens server-side. The API never returns cross-tenant, cross-sales-rep, or unauthorized product data and relies on the frontend to hide it.
- Information hiding:
  - another tenant's record → **404**
  - a Sales user requesting another Sales user's lead/client → **404**
  - a Product Admin requesting an ungranted product or its resources → **404** where revealing existence would leak unauthorized information
  - wrong account realm → **401 `INVALID_SESSION_REALM`**

**Request IDs**
- Every response includes `request_id` in the body (for errors) and the `X-Request-Id` header (always).
- The same ID is carried into structured logs and, where practical, into downstream and background operation context (queue messages, jobs), so a failed operation can be traced end to end.

**Error-code registry**
- One shared platform registry of error codes lives in `platform/`.
- Products add their own codes only under a product prefix (`NAIL_`, `RENTAL_`, …).
- Products may not redefine the meaning of shared platform codes.

**Idempotency**
- `Idempotency-Key` is **not** required for every mutation. It is required for operations where a retry or duplicate submission could cause a meaningful duplicate side effect, at minimum:
  - recording platform payments
  - payment-provider operations
  - commission/payout creation or approval
  - outbound email sends
  - external Connected Services actions
  - bulk imports and other operations that could create duplicate records or actions
  - any other mutation explicitly classified as high-risk or retry-prone
- Ordinary Base CRUD (updating a business profile, branding, a note, and similar) does not require a key unless that specific operation carries duplicate-side-effect risk.
- **Idempotency does not rely only on HTTP headers.** Queued/background jobs and provider-webhook processing use stable operation/event identifiers plus deduplication. No queue, Worker, webhook, network, or user retry may charge twice, record the same payment twice, generate or pay commission twice, send the same transactional action twice, or repeat any other protected external side effect.
- The database-level protections already locked remain: unique `platform_payments.provider_ref` and unique `(payment_id, sales_user_id, entry_type)` on commission ledger entries (§35c A7).

## 35e. Logging Conventions — LOCKED (§47 Item #7)

**Scope note:** Altreia Cloud only.

**Two separate systems**
- **Operational logs** serve engineering: debugging, errors, performance, tracing. Short-lived.
- **`audit_logs`** (§35b) record who did what for security and business accountability. Permanent, stored in the database, never sampled.
- Neither substitutes for the other.

**Destination**
- V1 uses **Cloudflare Workers Logs only**. No Logpush destination or external logging provider yet.
- The logging design stays provider-neutral so Logpush or external observability can be added later without rewriting application logging.

**Format and logger**
- One structured JSON object per log entry.
- All logging goes through the shared platform logger in `platform/`. No scattered `console.log` in committed code.

**Standard fields**
```json
{
  "ts": 1790000000123,
  "level": "info",
  "env": "production",
  "service": "api",
  "request_id": "01J...",
  "realm": "customer",
  "actor_id": "01J...",
  "business_id": "01J...",
  "product_code": "nail_tech_os",
  "route": "POST /v1/appointments",
  "event": "appointment.created",
  "status": 201,
  "duration_ms": 42,
  "error_code": null
}
```
- `service`: `api | jobs | admin | app`. `realm`: `customer | internal | system`.
- `event` is a stable dot-separated name. Dashboards and alerts key on `event` and `error_code`, never on free text.
- Request/operation IDs propagate into logs (§35d). Background jobs also log `job`, `attempt`, and `idempotency_key`.

**Levels**

| Level | Use | Enabled in |
|---|---|---|
| `debug` | Detailed troubleshooting | Local and DEV only |
| `info` | Normal meaningful events | Hosted environments |
| `warn` | Recoverable problems (retry, slow dependency, rejected login) | All |
| `error` | Failed operations needing attention | All |

**Sampling**
- V1 keeps **100%** of request summary logs. No routine production sampling.
- If traffic, retention, or cost later justify it, routine successful requests may be sampled.
- The following are **never** intentionally sampled away: errors, warnings, security-relevant events, authentication failures, wrong-realm attempts, permission denials, failed/dead-letter jobs, Connected Services failures, startup/configuration failures, and operationally relevant duplicate/idempotency protection events.
- Audit logs are never sampled.

**Redaction and privacy**
- Never logged in any environment: passwords, password hashes, MFA codes, session IDs, cookies, authorization headers, tokens, API keys, secrets, complete login request bodies, full request/response bodies, payment card or bank data, government ID or driver's license numbers, uploaded file contents.
- Customer operational data and Sales CRM contact information are never logged.
- Raw email addresses, phone numbers, and other personal identifiers are avoided when an internal ID or minimized metadata is enough.
- Authentication and security logs record the minimum useful diagnostic information.
- The shared logger strips known sensitive fields (deny-list) as a safety net; bodies are never logged whole, only selected fields.
- Error logs may include stack traces internally, after redaction.

**Always logged**
- A request summary line (route, status, duration, IDs, `error_code`).
- Every 5xx, with stack trace.
- Authentication events: success, failure, MFA failure, wrong-realm attempts.
- Permission denials and scope rejections.
- Job attempts, retries, and idempotency duplicate catches.
- Dependency failures (payments, email, Connected Services).
- Startup binding/secret validation failures.

**Denials vs. information hiding**
- Security-relevant denials are logged internally without weakening the API's information-hiding rules (§35d). A request for another tenant's record still returns 404 to the caller, while the internal log records that tenant authorization rejected it.

**Log failure behavior**
- Operational logging is never a hard dependency of Base product functionality. If routine log delivery is temporarily unavailable, business operations do not fail because of it.
- Audit events are different: operations defined as requiring an audit record keep that requirement under the audit architecture.

**Access**
- Raw production log access requires the explicit permission `platform.logs.read.production`, not a role-name check.
- SUPER_ADMIN holds it by default. No other internal role does by default; it can be granted to an authorized technical administrator later without changing the authorization design.
- Sales users never receive it. Product Admin users do not receive it by default.

**Alerts**
- V1 sends operator **email** alerts.
- Alert definitions are independent of delivery, so another destination (e.g. Slack) can be added later without changing them.
- Initial alert set: 5xx rate spike, dead-letter jobs, Worker startup validation failures, suspicious spikes in authentication failures or wrong-realm attempts.
- Log retention follows the Workers Logs default for V1.

## 36. Security Requirements (pre-production)

Secure authentication, secure sessions, role authorization, tenant isolation, cross-tenant security tests, file authorization, rate limiting, upload validation, sensitive document handling, audit logging, data deletion, retention policies, account suspension, account recovery, basic abuse prevention.

Car Rental OS may store driver's licenses and IDs — these require especially careful access controls.

**[Amendment A]** Also required: customer/internal realm separation with cross-realm tests, internal MFA, append-only commission ledger protected at the database level, separation of duties for payouts, audited and permission-gated manual platform payments, and no internal Sales access path to tenant operational data. See §35c A8.

## 37. Backup & Recovery

Must exist before production launch: database restore process, accidental-deletion recovery, file recovery strategy, business data export, migration rollback strategy, recovery documentation. Cloud provider recovery capabilities may be used but do not replace our own operational recovery procedure.

## 38. Migration Strategy

Every schema change uses versioned migrations. Separate migration streams for control plane vs. operational schema. Product-specific schema changes are tracked and tested. Migrations run against staging before production. No manual production schema edits.

## 38a. Migration Naming & Versioning — LOCKED (§47 Item #4)

**Scope note:** Altreia Cloud only.

**Streams and layout**
```
database/
├── control/migrations/        # Control D1 stream
└── operational/migrations/    # Operational D1 stream
```
- Control D1 and Operational D1 have separate, independent migration streams. `control/0005` and `operational/0005` have no implied relationship.
- A rare change spanning both is documented as a pair sharing a description suffix (e.g. `control/0007_add_x.sql` + `operational/0004_add_x.sql`).

**Naming**
- `NNNN_snake_case_description.sql` — 4-digit, zero-padded, strictly increasing per stream, starting at `0001`.
- The description names the change, not just the table (e.g. `0004_split_plan_pricing_from_plans.sql`).
- Merged migration numbers are never reused or renumbered.

**Immutability**
- Merged/applied migrations are immutable. Mistakes are fixed forward with a new migration.
- Both Control D1 and Operational D1 maintain `schema_migrations` (`version`, `checksum`, `applied_at`).
- Migration tooling must refuse to proceed if it detects a previously-applied migration whose checksum has changed, or a gap in the sequence.

**Environment order**
- Apply development → staging → production. Production is never ahead of staging.
- No manual production schema edits (§38).

**Rollback and recovery**
- Rollback files are **optional**, and exist only where a migration can be safely and meaningfully reversed: `NNNN_snake_case_description.rollback.sql`.
- No artificial rollback SQL for migrations that cannot genuinely restore the previous state.
- Irreversible or destructive migrations must explicitly declare themselves **forward-only** in a header comment.
- Schema rollback is not data recovery. A rollback file restores structure, not lost data.
- Any destructive production migration must have a documented recovery strategy before production deployment.
- Where data loss is possible, an appropriate backup/recovery checkpoint (§37) is required before applying the migration to production.
- For significant schema changes, prefer **expand → migrate/backfill → verify → contract** over destructive one-step migrations where practical.
- Production rollback must never silently discard customer data.

## 39. Environment Strategy

Local Development, Staging (production-like), Production. Secrets and production credentials never live in source code.

## 39a. Environment Configuration — LOCKED (§47 Item #5)

**Scope note:** Altreia Cloud only.

**Tiers and flow**

`LOCAL → DEV → STAGING → PRODUCTION`

- **Development tier** has two parts:
  - **Local** is the primary, fast environment: `wrangler dev` with local D1/R2/KV emulation.
  - **DEV (hosted)** is a shared Cloudflare deployment for integration testing when local emulation isn't enough. It belongs to the Development tier and is not a formal release environment.
- **Staging** is the production-like check before release.
- **Production** serves real customers.

| | Local | DEV (hosted) | Staging | Production |
|---|---|---|---|---|
| Data | Synthetic only | Synthetic only | Synthetic only | Real |
| Cloudflare Access on Admin | Off | Per policy | **On** | **On** |
| Turnstile | Test keys | Environment keys | Environment keys | Environment keys |
| Connected Services | Stubbed | Sandboxed/stubbed | Provider sandbox/test modes | Live |
| Logging | Debug | Structured | Structured | Structured, personal/sensitive data redacted |

**Cloudflare account**
- V1 uses **one Cloudflare account**.
- Each hosted environment has its own separate Control D1, Operational D1, R2 bucket, KV namespaces, Queues, Worker deployments, secrets, bindings, and Cloudflare Access policies where they apply. No environment shares an application data resource with another.
- Resources are named `altreia-<resource>-<env>` (e.g. `altreia-control-staging`, `altreia-files-prod`).
- **Future hardening:** Production may move to its own Cloudflare account if Altreia's scale, team size, security requirements, or operator access model justify it.

**Configuration and secrets**
- One Wrangler config per Worker/app, with a block per environment. Resource IDs may live there; they are identifiers, not secrets.
- Non-secret settings (environment name, log level, public URLs) go in Wrangler `vars`.
- Secrets are set per environment with `wrangler secret put` and never committed.
- Local secrets live in `.dev.vars`, which is git-ignored. `.dev.vars.example` is committed with key names and no values.
- Production secrets are held by the smallest practical group of people.

**Staging and DEV data**
- Staging and hosted DEV use synthetic/test data only. Real customer or production data is never copied into them.
- Production-like QA datasets are generated as representative synthetic fixtures.
- Stripping names/emails from a production copy does **not** count as safe staging data.

**Environment safety guardrails**
- **No fallback between environments.** If a required binding, secret, database, bucket, queue, provider configuration, or other dependency is missing or invalid, the Worker fails safely and loudly at startup. It never silently uses another environment's resource.
- This applies in both directions: Production never falls back to DEV/Staging resources, and DEV/Staging can never reach Production bindings through a default or fallback configuration.
- Every Worker validates its required bindings and secrets at startup.
- Non-production interfaces, especially Altreia Admin, are clearly marked as **LOCAL**, **DEV**, or **STAGING** so operators always know which environment they are in.
- Any operation that can affect real customers (sending real email, charging real payments, contacting real people) is only available in Production, with Production authorization.

**Promotion**
- Approved merges to `main` deploy automatically to Staging.
- Production deploys only through an explicit release process (details in §47 Item #12).
- Migrations are promoted dev → staging → production, per §38a.

## 40–42. Subscription States & Lifecycle

Base subscription states: `TRIAL`, `ACTIVE`, `PAST_DUE`, `EXPIRED`, `SUSPENDED`, `CANCELLED`. Not every state needs full billing automation in V1.

**Expired Base:** account becomes read-only. Customer may still log in, view data/documents, download files, export records. Customer may not create records, edit operational data, or perform new transactions. Data is not deleted immediately (formal retention policy defined separately).

**Connected cancellation:** disables only Connected entitlements (`booking.public`, `email.send`, `email.reminders`, `payments.online`, `calendar.sync` → false). Existing Base OS data, appointments/reservations, payment records, and files remain intact.

## 43–44. Development Philosophy & Phase Order

Develop in small, reviewable slices, each with: exact scope, explicit exclusions, acceptance criteria, automated tests, security considerations, migration impact, QA checklist. Do not hand an AI coding tool the entire platform and ask it to "build everything."

**Phase order:**

| Phase | Focus |
|---|---|
| 0 | Architecture Lock — **CURRENT / COMPLETE** |
| 1 | Platform Skeleton (monorepo, apps, Cloudflare envs, Worker API skeleton, control/operational D1, migration + testing infra, deployment foundation — no Nail/Car Rental workflows). **[Amendment A]** includes internal identity tables and the internal RBAC/permissions schema foundation |
| 2 | Authentication & Tenancy (accounts, login, sessions, businesses, memberships, roles, tenant context, authorization middleware, cross-tenant tests). **[Amendment A]** includes internal authentication, internal sessions, mandatory internal MFA, and realm-separation enforcement |
| 3 | Product & Subscription Foundation (products, plans, subscriptions, states, product assignment, entitlements, overrides, feature guards, expiration/read-only behavior). **[Amendment A]** includes product management and product grants, sales attribution and lead foundation, and platform payment foundation (manual recording) |
| 3A | **[Amendment A] Sales & Commissions** — commission rules, ledger processing, reversals/adjustments, payouts, Sales dashboard/CRM UI, commission reporting. Scheduled any time after Phase 3 is complete; never folded into Phase 1 |
| 4 | Workspace Foundation (app shell, nav, branding, settings, dark mode, modals/toasts/loaders, empty states, unsaved-change protection) |
| 5 | Files & Storage (R2, uploads, quotas, metadata, categories, secure downloads, image optimization, usage, admin controls, audit history) |
| 6 | Shared Customer Identity (identity, contact info, notes, tags, timeline foundation, file relationships, product extension mechanism — no generic CRM) |
| 7 | Nail OS V2 Core |
| 8 | Car Rental OS V2 Core |
| 9 | Shared Pattern Extraction (only after both products exist; extract only proven reusable patterns) |
| 10 | Communication Infrastructure (email abstraction, transactional email, queues/jobs, retries, delivery logging, templates, reminders) |
| 11 | Public Booking / Reservation (product-specific public flows) |
| 12 | Automation (scheduled reminders, confirmations, follow-ups, triggered communication, Connected jobs) |
| 13 | Online Payments (provider adapter, payment links, deposits, webhooks, reconciliation, payment-status handling) |
| 14 | OS Builder V1 (only after platform patterns proven; codifies existing config, invents no new architecture) |
| 15 | Third OS Validation (e.g. ResortFlow — prove the platform/Builder reduce dev effort) |

## 45. Explicitly Not in V1

Visual page builder, Bubble/Retool clone, arbitrary workflows, scripting language, arbitrary custom database fields, marketplace, native mobile apps, advanced AI agents, multi-location enterprise architecture, SMS, universal asset abstraction, complicated multi-product accounts, excessive subscription tiers, complex white-label domains, Kubernetes, microservice sprawl.

## 46. Phase 0 Final Lock Checklist

- [x] Altreia defined as one platform
- [x] One customer-facing platform app
- [x] Separate internal Admin
- [x] Nail OS and Car Rental OS are product domains
- [x] Shared infrastructure vs. product logic principle defined
- [x] Base vs. Connected defined
- [x] Configurable annual pricing defined
- [x] Roles separated from entitlements
- [x] Product configuration separated from entitlements
- [x] Entitlement override concept defined
- [x] Branding platform-owned
- [x] File infrastructure platform-owned
- [x] 500 MB storage direction
- [x] Shared minimal customer identity
- [x] Product customer extensions
- [x] No generic Calendar domain
- [x] No generic Finance domain yet
- [x] No generic Assets module yet
- [x] Builder deferred
- [x] One shared operational D1 for initial release
- [x] Tenant-aware repository/data layer required
- [x] Future database sharding allowed
- [x] Read-only expired Base accounts
- [x] Connected cancellation preserves Base
- [x] Backups moved before production
- [x] Migration infrastructure required
- [x] Staging required
- [x] Security requirements defined
- [x] [Amendment A] Separate internal account realm with data-driven RBAC
- [x] [Amendment A] Product management through explicit product grants
- [x] [Amendment A] Platform-level sales attribution and OWN-scoped sales CRM
- [x] [Amendment A] Altreia CRM data separated from tenant operational data
- [x] [Amendment A] Commissions from payments to Altreia only, with append-only ledger and payouts

## 47. Remaining Items Before Phase 1 Implementation

The high-level architecture is locked. Before Phase 1 implementation begins, the next specification must define:

1. ~~Exact Phase 1 repository structure~~ — **RESOLVED / LOCKED** (see §35, updated)
2. ~~Exact Cloudflare resources~~ — **RESOLVED / LOCKED** (see §35a, new)
3. ~~Exact database table schemas for Phase 1~~ — **RESOLVED / LOCKED** (see §35b, new)
4. ~~Migration naming/versioning convention~~ — **RESOLVED / LOCKED** (see §38a, new)
   - **Phase 0 Amendment A** (internal accounts, product management, sales attribution, commissions) — **RESOLVED / LOCKED** after Item #5 (see §35c)
5. ~~Development/staging/production configuration~~ — **RESOLVED / LOCKED** (see §39a, new)
6. ~~API response/error conventions~~ — **RESOLVED / LOCKED** (see §35d, new)
7. ~~Logging conventions~~ — **RESOLVED / LOCKED** (see §35e, new)
8. Testing stack
9. Coding standards
10. Phase 1 acceptance criteria
11. Exclusions for Phase 1
12. Deployment/tagging procedure

## Final Architecture Rule

When deciding whether something belongs in Altreia Platform or an individual OS, ask:

> Is this necessary because the customer uses Altreia, or because the customer runs this particular kind of business?

- "Because they use Altreia" → probably platform infrastructure.
- "Because they operate a nail studio / car rental / resort" → probably belongs to that product.

When uncertain, keep it product-specific first. Extract later when real duplication proves it belongs in the platform.
