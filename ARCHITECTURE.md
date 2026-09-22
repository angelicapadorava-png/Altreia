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

## 13. Entitlement Overrides

Admin can temporarily or permanently override entitlements:

```
business_id, feature_key, granted, reason, expires_at, created_by
```

Use cases: beta access, free trial, support workaround, temporary promotion, special customer agreement. The original plan configuration remains unchanged.

## 14. Authentication

Platform-level. Recommended: Better Auth or equivalent — secure sessions, business membership checks, role-based authorization, server-side authorization enforcement. Frontend checks are UX only; access control is enforced server-side.

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

## 35. Repository Direction

Recommended monorepo:

```
altreia/
├── apps/
│   ├── platform/
│   └── admin/
├── platform/
│   ├── auth/
│   ├── businesses/
│   ├── branding/
│   ├── billing/
│   ├── entitlements/
│   ├── permissions/
│   ├── files/
│   ├── storage/
│   ├── audit/
│   └── ui/
├── products/
│   ├── nail/
│   │   ├── domain/
│   │   ├── data/
│   │   ├── ui/
│   │   └── tests/
│   └── car-rental/
│       ├── domain/
│       ├── data/
│       ├── ui/
│       └── tests/
├── workers/
│   ├── api/
│   └── jobs/
├── database/
│   ├── control/
│   │   └── migrations/
│   └── operational/
│       └── migrations/
└── tests/
    ├── integration/
    └── security/
```

Exact folder names may change during setup, but the boundary between platform / products / admin must remain.

## 36. Security Requirements (pre-production)

Secure authentication, secure sessions, role authorization, tenant isolation, cross-tenant security tests, file authorization, rate limiting, upload validation, sensitive document handling, audit logging, data deletion, retention policies, account suspension, account recovery, basic abuse prevention.

Car Rental OS may store driver's licenses and IDs — these require especially careful access controls.

## 37. Backup & Recovery

Must exist before production launch: database restore process, accidental-deletion recovery, file recovery strategy, business data export, migration rollback strategy, recovery documentation. Cloud provider recovery capabilities may be used but do not replace our own operational recovery procedure.

## 38. Migration Strategy

Every schema change uses versioned migrations. Separate migration streams for control plane vs. operational schema. Product-specific schema changes are tracked and tested. Migrations run against staging before production. No manual production schema edits.

## 39. Environment Strategy

Local Development, Staging (production-like), Production. Secrets and production credentials never live in source code.

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
| 1 | Platform Skeleton (monorepo, apps, Cloudflare envs, Worker API skeleton, control/operational D1, migration + testing infra, deployment foundation — no Nail/Car Rental workflows) |
| 2 | Authentication & Tenancy (accounts, login, sessions, businesses, memberships, roles, tenant context, authorization middleware, cross-tenant tests) |
| 3 | Product & Subscription Foundation (products, plans, subscriptions, states, product assignment, entitlements, overrides, feature guards, expiration/read-only behavior) |
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

## 47. Remaining Items Before Phase 1 Implementation

The high-level architecture is locked. Before Phase 1 implementation begins, the next specification must define:

1. Exact Phase 1 repository structure
2. Exact Cloudflare resources
3. Exact database table schemas for Phase 1
4. Migration naming/versioning convention
5. Development/staging/production configuration
6. API response/error conventions
7. Logging conventions
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
