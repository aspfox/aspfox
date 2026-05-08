# Changelog

All notable changes to AspFox are documented here.

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)  
Versioning: [Semantic Versioning](https://semver.org/spec/v2.0.0.html)

---

## [Unreleased]

---

## [0.1.0] — 2026-05-08

### Added

**Authentication**

- JWT RS256 authentication with asymmetric key signing — the API signs tokens with a private key; any service with the public key can verify them without the secret
- 15-minute access tokens paired with 7-day rotating refresh tokens; every successful refresh issues a new refresh token and invalidates the previous one
- Refresh token reuse detection — submitting a revoked token immediately revokes the entire token family and forces re-login for all sessions
- Email/password registration with mandatory email verification before the first login is permitted
- Passwordless magic link authentication with a 10-minute expiry
- Google and GitHub OAuth with automatic account creation for first-time social logins
- Forgot password and reset password flows with a 1-hour token expiry
- All authentication endpoints protected against email enumeration — success and failure responses are indistinguishable to an outside observer

**Multi-Tenancy**

- Row-level tenant isolation via EF Core global query filters — cross-tenant data access is architecturally impossible without modifying the code itself
- Tenant creation with automatic slug generation, collision handling, and unique constraint enforcement
- JWT-based tenant context — the active tenant is a first-class JWT claim (`tenant_id`), resolved on every request by `TenantResolutionMiddleware`
- Tenant switching without re-authentication — issues a new token pair scoped to the requested tenant
- Complete team invitation flow: invite by email → signed secure token → 72-hour expiry → accept endpoint → automatic account creation for new users
- Pending and expired invitation management with cancellation support

**Role-Based Access Control**

- Three built-in roles per tenant: Owner, Admin, and Member, each with a defined, non-configurable permission set
- Ten granular permission strings covering settings read/edit, members read/invite/remove, billing read/manage, roles read/manage, and ownership transfer
- Dynamic ASP.NET Core policy provider — permission policies are created on demand; there is no startup registration step for each permission string
- Custom role creation per tenant with a fully configurable permission set
- Role assignment with ownership transfer protection — the Owner role cannot be assigned through the normal role endpoint

**Stripe Billing**

- Full subscription lifecycle: checkout session creation, customer portal, plan upgrades, downgrades, cancellation scheduling, and reactivation
- 14-day free trial automatically applied to Pro and Business plan checkouts
- Idempotent Stripe webhook processing — the same event delivered twice produces the same database state as once, with each event ID recorded to prevent double-processing
- Active subscription status cached in Redis with a 5-minute TTL to avoid a database round-trip on every authenticated request
- Customer billing portal integration for self-serve plan and payment management
- Admin manual subscription toggle for support use when Stripe and local state diverge

**Email**

- Transactional email via Resend with 12 responsive HTML templates built on a shared base layout
- Templates: welcome, email verification, password reset, magic link, password changed, tenant invitation, trial expiry (7-day warning), trial expiry (1-day warning), payment failed, payment recovered, cancellation scheduled, cancellation confirmed, upgrade confirmed
- Email failures are logged at Warning level and swallowed — a broken email service never causes an authentication or billing operation to fail

**Background Jobs (Hangfire)**

- Trial expiry job: sends the 7-day warning email, the 1-day warning email, and on expiry downgrades the subscription to Free automatically
- Subscription sync job: reconciles local subscription state with Stripe every 2 hours to recover from missed or failed webhooks
- Token cleanup job: purges expired refresh tokens, password reset tokens, email verification tokens, and magic link tokens daily
- Invitation cleanup job: removes expired invitations older than 30 days while preserving recent ones so admins can see the recent invite history
- Hangfire dashboard secured behind an `is_admin` claim check; no admin JWT, no dashboard access

**Admin Panel**

- User management: paginated list with search, soft delete, and user impersonation
- Tenant management: paginated list with search, detail view including audit log and subscription status
- Subscription overview: active, trialing, past due counts and MRR calculation
- User impersonation with a full audit trail — every impersonation event is recorded with the admin's user ID
- Manual subscription status override for support use without touching the Stripe dashboard
- All admin-initiated actions written to the per-tenant audit log

**User Profile**

- Display name, avatar URL, and timezone preference management
- Email change flow with verification sent to the new address — the change only takes effect after the new address is confirmed
- Password change with current password confirmation
- Account deletion with password confirmation, Stripe subscription cancellation, and cascade cleanup

**Frontend (React 18 + TypeScript)**

- Complete authentication pages: login, register, forgot password, reset password, email verification, magic link confirmation, invitation acceptance
- Workspace onboarding flow for new users who have no tenant yet
- Tenant settings page with permission-gated form controls — fields are disabled when the user lacks the required permission
- Member management page: member table with role badges, role-change dropdowns, remove-member actions, and an invite modal
- Billing page: plan comparison grid, trial countdown banner, PastDue alert with portal link, and subscription management
- Profile page with email change flow and account deletion confirmation dialog
- Admin dashboard: metrics cards, searchable user table, searchable tenant table, impersonation controls, subscription override panel
- Impersonation banner displayed during admin impersonation sessions with a stop-impersonating action
- Concurrent-safe JWT refresh token interceptor in Axios — parallel 401 responses are queued and resolved after a single refresh, rather than triggering multiple simultaneous refresh attempts
- All components handle the loading skeleton, error state, and empty state correctly

**In-App Notifications**

- Notification bell in the top bar with unread count badge, updated every 30 seconds via polling
- Notification dropdown showing recent notifications with mark-as-read and mark-all-as-read actions
- Notifications created automatically by billing events (trial expiring, payment failed, payment recovered, upgrade confirmed, cancellation), membership events (member invited, member joined, member removed), and ownership transfer

**Command Palette**

- Cmd+K / Ctrl+K keyboard shortcut opens the command palette from anywhere in the application
- Permission-aware navigation and action items — the palette only shows items the current user can actually access
- Fuzzy search across all registered commands with keyboard navigation

**Dark Mode**

- System preference detection on first load, with manual override via a toggle in the top bar
- Preference persisted across sessions via `next-themes` writing to localStorage
- Tailwind `darkMode: 'class'` — the `dark` class on `<html>` controls all color variables

**Infrastructure**

- Docker Compose for local development: PostgreSQL 15, Redis 7, API on port 5000, Vite dev server on port 5173
- Production multi-stage Dockerfiles with Alpine base images and dedicated non-root users for both API and frontend
- Production Compose file (`docker/docker-compose.prod.yml`) with memory limits, health checks, and a single nginx ingress point
- GitHub Actions CI: parallel backend (with PostgreSQL and Redis service containers) and frontend (type check, lint, build, test) jobs
- GitHub Actions deploy workflow: builds Docker images on every merge to main; deployment step is a commented scaffold for Railway, Render, or VPS via SSH
- Integration tests using TestContainers with a real PostgreSQL instance (no SQLite or in-memory mocks)
- Complete environment variable documentation in `.env.example` with descriptions, example values, and setup instructions

[0.1.0]: https://github.com/aspfox/aspfox/releases/tag/v0.1.0

> **Note for maintainer**: this file belongs in the `aspfox/aspfox` public repo as `CHANGELOG.md`. Do not commit it to `aspfox-private`.
