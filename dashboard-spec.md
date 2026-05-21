# Privata Partner Dashboard - Product + Technical Specification

**Version:** 1.0  
**Status:** Draft for internal review  
**Audience:** Engineering, Product, Security, QA  
**Last revised:** 2026-05-22  

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Information Architecture](#2-information-architecture)
3. [User Roles & Permissions Matrix](#3-user-roles--permissions-matrix)
4. [Detailed Feature Specs](#4-detailed-feature-specs)
5. [Auth + Security Architecture](#5-auth--security-architecture)
6. [Data Model](#6-data-model)
7. [Tech Stack Proposal](#7-tech-stack-proposal)
8. [Internal API Surface (Dashboard Backend)](#8-internal-api-surface-dashboard-backend)
9. [Realtime Architecture](#9-realtime-architecture)
10. [Notifications](#10-notifications)
11. [Mobile / Responsive Strategy](#11-mobile--responsive-strategy)
12. [Build Phases (P0 / P1 / P2)](#12-build-phases-p0--p1--p2)
13. [Open Questions / Risks](#13-open-questions--risks)
14. [Multi-Role Review Notes](#14-multi-role-review-notes)

---

## 1. Product Overview

The Privata Partner Dashboard (`dashboard.privataswap.com`) is the self-service control plane for B2B partners (crypto wallets, CEX integrations, and payment applications) who embed the Privata swap aggregator via `api.privataswap.com/partner/v1`. It provides a single authenticated surface for API credential management, webhook configuration, live order monitoring, revenue tracking, SLA observability, team administration, and billing, eliminating the need for email-based onboarding and manual back-office intervention for routine partner operations.

**Five key user jobs-to-be-done:**

1. **Provision and rotate credentials safely.** A developer needs to issue a scoped API key, restrict it to specific IP ranges, and rotate it in under five minutes without downtime to their integration, relying on the documented dual-sign grace window (see [Authentication](/guides/authentication)).
2. **Diagnose a failed or stuck order.** A support engineer needs to find a specific order by ID or customer reference, inspect its full event timeline, see the raw webhook delivery attempts and response codes, and trigger a manual refund-action - without requiring Privata back-office involvement.
3. **Monitor integration health in real time.** A technical lead needs a live view of orders per minute, success rate, provider routing distribution, webhook delivery error rate, and SSE stream concurrency against the cap - all on one screen, refreshing without a page reload.
4. **Track and withdraw revenue.** A finance role needs to see pending and paid rev-share earnings broken down by period, configure a verified payout address, and initiate a payout with HMAC-confirmed withdrawal - without access to API keys or order internals.
5. **Onboard additional team members with least-privilege roles.** An Owner needs to invite a Finance user, a Developer, and a Read-only auditor, each receiving only the permissions their role requires - without sharing credentials.

---

## 2. Information Architecture

### 2.1 Full Sitemap

```
dashboard.privataswap.com/
│
├── /auth/
│   ├── /signup                      - Email + password registration
│   ├── /login                       - Email + password login
│   ├── /login/2fa                   - TOTP or WebAuthn challenge
│   ├── /password-reset              - Request reset link
│   └── /password-reset/confirm      - Set new password via token
│
├── /onboarding/                     - Gated until complete; redirects new org on first login
│   ├── /organization                - Organization name, business email, jurisdiction
│   ├── /volume                      - Intended monthly swap volume (USD range)
│   ├── /use-case                    - Integration use case (free text)
│   ├── /terms                       - Accept Partner Terms
│   └── /done                        - Confirmation screen; auto-issue test key; prompt to quickstart docs
│
├── /                                - Home Dashboard (default after auth + onboarding)
│   ├── KPI strip: orders today, 24h volume (USD), success rate, avg latency (p50/p99), top provider
│   ├── Mini chart: orders / hour (rolling 24h)
│   ├── Recent webhook deliveries widget (last 10, status badge)
│   ├── Active SSE streams count
│   ├── Active incidents banner (if SLA breach or provider outage)
│   └── Quick-action shortcuts: Create API key, Configure webhook, View orders
│
├── /api-keys/
│   ├── /                            - List: live keys, test keys; prefix, scopes, last_used_at, status
│   ├── /new                         - Create key modal: name, environment, scopes, IP allowlist
│   ├── /[key_id]/                   - Key detail
│   │   ├── /edit                    - Edit scopes, IP allowlist, expiry
│   │   └── /rotate                  - Rotation wizard: dual-sign grace countdown
│   └── /[key_id]/revoke             - Confirmation modal
│
├── /webhooks/
│   ├── /                            - Endpoints list (Order URL, Ops URL); status badges
│   ├── /new                         - Add endpoint: URL, type (order/ops), events filter
│   ├── /[endpoint_id]/
│   │   ├── /edit                    - URL, secret rotation button, events filter
│   │   ├── /secret                  - View masked secret; rotate modal with grace countdown
│   │   └── /deliveries/             - Delivery log: filter by event, status, date
│   │       └── /[delivery_id]/      - Delivery detail: request headers, body, response code, response excerpt; Retry / Replay buttons
│   └── /resume                      - One-click resume for a paused endpoint (calls POST /partner/v1/webhook/resume)
│
├── /sse/
│   └── /                            - Active stream list: order_id, connected_at, client_ip (hashed); concurrency gauge vs. cap
│
├── /orders/
│   ├── /                            - Search + filter table; export CSV
│   └── /[order_id]/                 - Order detail
│       ├── Summary card (pair, amounts, provider, status badge)
│       ├── Event timeline (all status transitions + timestamps)
│       ├── Deposit + payout tx hashes (block explorer links)
│       ├── Webhook delivery history for this order
│       ├── Refund action panel (shown when status = refund_required and capability = programmatic)
│       └── Raw JSON tab (full order object from API)
│
├── /providers/
│   └── /                            - Provider grid: enable/disable toggle, fee config, refund capability badge (programmatic/ticket/none), window_hours
│
├── /revenue/
│   ├── /                            - Earnings overview: pending balance, paid to date, rev-share %, markup earnings
│   ├── /history                     - Payout history table: amount, currency, tx_hash, date, status
│   └── /payouts/
│       ├── /addresses               - Payout address management: add, verify, set default
│       └── /request                 - Manual payout request (min threshold); HMAC-signed confirmation step
│
├── /sla/
│   └── /                            - Live uptime vs. tier target; compensation accrual this month; breach incidents log; prober config helper
│
├── /team/
│   ├── /                            - Members list: name, email, role, last_login, status
│   ├── /invite                      - Invite by email: select role
│   └── /[member_id]/edit            - Change role, deactivate
│
├── /audit-log/
│   └── /                            - Immutable action log: actor, action, target, IP, timestamp; filter + export
│
├── /billing/
│   ├── /                            - Current plan tier (Starter/Pro/Enterprise), revenue summary, payout config
│   └── /upgrade                     - Plan comparison; upgrade to Enterprise (contact partners@privataswap.com)
│
├── /settings/
│   ├── /organization                - Org name, logo (for branded partner pages), timezone
│   ├── /security                    - 2FA enrollment (TOTP, WebAuthn); session list; new-device email preference
│   ├── /tokens                      - Scoped personal access tokens (for CI/CD, separate from org API keys)
│   └── /webhook-ip-allowlist        - IP ranges allowed to receive webhook deliveries (outbound filter bypass)
│
└── /docs/                           - Shortcut redirect to docs.privataswap.com; contextual deep-links per section
```

---

## 3. User Roles & Permissions Matrix

Roles are scoped per organization. A user may belong to multiple organizations with different roles in each.

| Permission | Owner | Admin | Developer | Finance | Read-only |
|---|:---:|:---:|:---:|:---:|:---:|
| View home dashboard | ✓ | ✓ | ✓ | ✓ | ✓ |
| Complete onboarding | ✓ | ✓ | - | - | - |
| Create / revoke API keys | ✓ | ✓ | ✓ | - | - |
| View API key prefixes | ✓ | ✓ | ✓ | - | ✓ |
| View full API key secret (once, on create) | ✓ | ✓ | ✓ | - | - |
| Edit API key scopes / IP allowlist | ✓ | ✓ | ✓ | - | - |
| Rotate API key | ✓ | ✓ | ✓ | - | - |
| Configure webhook endpoints | ✓ | ✓ | ✓ | - | - |
| View webhook delivery log | ✓ | ✓ | ✓ | - | ✓ |
| Replay / retry webhook delivery | ✓ | ✓ | ✓ | - | - |
| Resume paused webhook | ✓ | ✓ | ✓ | - | - |
| View SSE stream list | ✓ | ✓ | ✓ | - | ✓ |
| Search / view orders | ✓ | ✓ | ✓ | ✓ | ✓ |
| Export orders CSV | ✓ | ✓ | ✓ | ✓ | - |
| Trigger refund action | ✓ | ✓ | ✓ | - | - |
| Enable / disable providers | ✓ | ✓ | - | - | - |
| Edit provider fee config | ✓ | ✓ | - | - | - |
| View revenue / earnings | ✓ | ✓ | - | ✓ | - |
| Manage payout addresses | ✓ | - | - | ✓ | - |
| Initiate payout request | ✓ | - | - | ✓ | - |
| View SLA page | ✓ | ✓ | ✓ | ✓ | ✓ |
| Invite team members | ✓ | ✓ | - | - | - |
| Change member roles | ✓ | ✓ | - | - | - |
| Remove members | ✓ | - | - | - | - |
| View audit log | ✓ | ✓ | - | ✓ | - |
| Export audit log | ✓ | ✓ | - | - | - |
| Manage billing / upgrade plan | ✓ | - | - | - | - |
| Edit organization settings | ✓ | ✓ | - | - | - |
| Configure GDPR data export / delete | ✓ | - | - | - | - |

**Note:** The Finance role has read access to orders (needed for revenue reconciliation) but cannot see API secrets, trigger refunds, or perform any write operation against the Privata API surface.

---

## 4. Detailed Feature Specs

### 4.1 Auth

**Purpose:** Authenticate dashboard users; enforce 2FA for production access.

**Primary actions:**
- Register with email + password
- Log in with email + password + optional TOTP / WebAuthn
- Reset password via email link

**Signup flow:**
1. Email + password form. Password requirements: min 12 chars, not in HIBP top-10M list (checked client-side against k-anonymity API).
2. Email verification: 6-digit OTP sent to email, valid 10 minutes. Rate-limited: 3 resends per hour per email.
3. After email verification, redirect to `/onboarding/company`.

**Login flow:**
1. Email + password → if TOTP enrolled, redirect to `/login/2fa` → issue session. If WebAuthn enrolled, offer as primary or fallback.
2. New device detection: device fingerprint (user-agent + screen size hash, not persistent canvas) compared against known-devices list. If new device: send email confirmation code before completing login; record device on confirm.
3. Geographic anomaly: if IP geolocation country differs from the majority of recent logins, trigger email confirmation. Log to audit as `login.geo_anomaly`.

**2FA enrollment (in `/settings/security`):**
- TOTP: show QR code + secret text; require one successful verification before enabling. Store TOTP secret encrypted with per-user DEK (see §5).
- WebAuthn (FIDO2): register up to 5 passkeys. Displayed with platform authenticator name + registration date.
- Backup codes: 10 single-use codes, displayed once, downloadable as text.

**Empty states:**
- Signup page: clean form with value prop bullet list.

**Error states:**
- Invalid credentials: generic "Email or password incorrect" (no user enumeration).
- TOTP invalid: "Invalid code. Codes refresh every 30 seconds."
- Account locked after 5 failed 2FA attempts: 30-minute lockout + email notification.

**Loading states:** Spinner on submit buttons; disable re-submit while in-flight.

---

### 4.2 Onboarding (KYB-lite)

**Purpose:** Collect partner metadata required for compliance and capacity planning before issuing a live API key. Completed once per organization; editable from Settings thereafter.

**Steps:**
1. **Organization** - Organization name, business email, jurisdiction (country dropdown, ISO-3166). Required fields.
2. **Volume** - Intended monthly swap volume in USD ranges: <$10k, $10k-100k, $100k-1M, $1M-10M, >$10M.
3. **Use case** - Integration use case (free text, 500 char max).
4. **Terms** - Accept Partner Terms (checkbox with link). Partner cannot proceed without accepting.

No document upload, no business registration number field. AML and sanctions screening are performed by the underlying swap providers on a per-order basis; the dashboard onboarding only records partner self-attestation.

**Edge cases:**
- Onboarding can be interrupted and resumed; progress saved per step.
- Admins can re-open onboarding edit from Settings - Organization.
- On completion: test key auto-issued and displayed (once); shortcut to [Quickstart guide](/guides/quickstart).

**Empty state:** Progress stepper visible; initial step pre-selected.

**Error state:** Server validation errors shown inline per field (e.g., "Country is required").

---

### 4.3 Home Dashboard

**Purpose:** Provide an at-a-glance health view of the partner's integration.

**KPI strip (top of page, realtime via SSE channel `org:{org_id}:kpis`):**

| Metric | Source | Refresh |
|---|---|---|
| Orders today | `orders` table, `created_at >= today_utc` | Realtime push |
| 24h volume (USD) | `orders.amount_in_usd` sum, rolling 24h | Realtime push |
| Success rate | completed / (completed + failed + expired), rolling 24h | Realtime push |
| Avg latency p50 / p99 | `orders.completed_at - orders.confirmed_at`, rolling 24h | 60s poll |
| Top provider (by order count) | `orders.provider_id` group-by, rolling 24h | 60s poll |

**Chart:** Sparkline bar chart, orders per hour, rolling 24h. Rendered client-side (Recharts). Data fetched once on load + pushed on new orders.

**Recent webhook deliveries widget:** Last 10 delivery attempts across all endpoints. Columns: endpoint alias, event type, HTTP status code, timestamp, retry count. Click-through to full delivery detail.

**Active SSE streams:** Integer count. Yellow if within 80% of cap; red if at cap.

**Incident banner:** Shown if `sla_incidents` has an open incident. Links to `/sla`. Dismissible per session.

**Empty state (new partner):** Onboarding checklist widget showing incomplete steps. "No orders yet - send your first test order" with link to Sandbox guide.

**Loading state:** KPI strip shows skeleton placeholders. Chart renders with empty axes.

**Error state:** If realtime connection fails, KPI strip shows "Live data unavailable - showing last snapshot from [timestamp]" with manual refresh button.

---

### 4.4 API Keys

**Purpose:** Create, scope, rotate, and revoke API keys for production and sandbox environments.

**List view columns:** Name, prefix (e.g. `pk_live_abc1...`), environment badge, scopes, IP allowlist (count), last_used_at, rotated_at, status (active / rotating / revoked).

**Create key flow:**
1. Name (required, internal label).
2. Environment: Live or Test.
3. Scopes: multi-select checkboxes (`orders:rw`, `webhook:admin`, `payouts:read`, `payouts:trigger`). Default: `orders:rw` only.
4. IP allowlist: optional, comma-separated CIDR blocks. Validated on submission.
5. Expiry: optional date. If set, key auto-revokes at midnight UTC on that date and triggers `api_key_locked` ops event.
6. On submit: key displayed **once** in a copy-to-clipboard modal. Never shown again. User must acknowledge "I have copied this key" before dismissing.

**Key rotation wizard:**
1. New key generated; displayed once.
2. Countdown timer: 5-minute grace window during which both old and new keys are accepted by the API.
3. After grace window: old key transitions to `revoked`. Audit log entry written.
4. If user closes browser during grace window: key rotation continues server-side; old key is revoked at TTL.

**IP allowlist:** Stored as `cidr[]` in Postgres. Validated at key-creation and key-edit time. Empty array = no restriction. Applied server-side by the public API, not by the dashboard backend.

**Revoke:** Confirmation modal showing "This will immediately reject all requests using this key." One-click confirm; writes audit log entry.

**Edge cases:**
- Revoking the last active live key: warning banner ("You have no active production keys. Your integration will stop working.").
- Duplicate name: allowed, keys are identified by ID not name.
- Attempting to set `payouts:trigger` scope on a test key: blocked with inline error.

**Empty state:** "No API keys yet. Create your first key to start integrating."

**Error states:**
- Invalid CIDR: "Invalid IP range format."
- Create fails: surface API error code + message.

---

### 4.5 Webhooks

**Purpose:** Configure HTTPS endpoints to receive `order.*` and ops events; manage secrets; inspect delivery history.

**Endpoint list:** Order URL, Ops URL, additional custom endpoints (P1). Columns: alias, URL (truncated), type, status (active / paused / failing), last delivery timestamp, success rate (30d).

**Add endpoint:**
- URL field: validated for HTTPS scheme, reachable host (non-SSRF - blocked: private RFC-1918 ranges, 169.254.169.254, metadata.google.internal, link-local, cloud metadata endpoints).
- Type: Order events / Ops events.
- Events filter (P1): select specific event types.
- On save: secret generated server-side; displayed once in copy modal. Stored encrypted.

**Secret rotation:**
1. "Rotate secret" button → modal explains 5-minute dual-sign window.
2. New secret generated. Both old and new are valid for 5 minutes.
3. After 5 minutes: old secret rejected. Audit log entry written.
4. Grace window countdown shown in UI. Cannot rotate again until previous rotation completes.

**Paused state:**
- UI shows "Paused" badge when endpoint is auto-paused (50 failures in 6h).
- "Resume" button visible; triggers `POST /partner/v1/webhook/resume` with `webhook:admin` scope key.
- If the active API key lacks `webhook:admin` scope: inline error explaining required scope.

**Delivery log:**
- Table: event type, order_id (if applicable), HTTP status code, attempt number, request timestamp, response time (ms), next_retry_at.
- Filter: by event type, by status (success/failure/pending), by date range.
- Pagination: 50 rows per page.

**Delivery detail:**
- Request: HTTP method, URL, headers (X-Privata-Signature masked after first 8 chars), body (pretty-printed JSON, truncated at 10KB).
- Response: HTTP status code, headers, response body excerpt (truncated at 1KB), response time.
- Actions: **Retry** (re-enqueues for immediate delivery, new delivery record) and **Replay** (re-sends exact original payload, new delivery record, new signature with current secret). Replay logs to audit log as `webhook.replay`.
- Edge case: if the endpoint is currently paused, Retry button is disabled with tooltip "Resume the endpoint first."

**Empty state:** "No deliveries yet. Configure your webhook URL and send a test order via [Sandbox](/guides/sandbox)."

**Error states:**
- Delivery failed: show response code + body excerpt.
- "No response" (timeout): show `connection_timeout` label.

---

### 4.6 SSE Streams

**Purpose:** Visibility into active SSE connections consuming the partner's concurrency quota.

**Columns:** Order ID, connected_at, connection duration, client IP (last two octets masked: `198.51.xxx.xxx`), `Privata-Resume-Source` of last reconnect.

**Concurrency gauge:** Horizontal bar showing `active / cap`. Cap is per-plan (e.g., 50 for Standard, 200 for Enterprise). Bar turns yellow at 80%, red at 100%.

**Actions:** No admin kill-switch in v1 (SSE streams are client-controlled; idle connections close on order terminal state).

**Empty state:** "No active SSE streams."

**Edge case:** If concurrency is at cap, a banner warns: "New SSE connections will be rejected until existing ones close or orders complete."

---

### 4.7 Orders

**Purpose:** Full-fidelity order search, inspection, and manual intervention.

**Search / filter table:**

| Filter | Type |
|---|---|
| Order ID | Exact match (prefix `prv-`) |
| Partner order ref | Exact match |
| Status | Multi-select dropdown |
| Pair (from/to) | Token picker |
| Provider | Multi-select |
| Date range | Date picker (created_at) |
| Amount range | Numeric, USD |

**Table columns:** Order ID, pair (BTC→XMR format), provider, amount in, amount out (actual or expected), status badge, created_at, duration (created→completed or created→now).

**Export:** CSV download of current filter set, up to 10,000 rows. Async job for larger sets; email notification on completion. Audit log entry on every export.

**Order detail page:**

- **Summary card:** Order ID (copy button), pair, provider name + logo, markup_bps applied, status badge.
- **Amounts:** `amount_from`, `amount_to_expected`, `amount_to_actual` (null if not completed). Slippage delta shown if actual differs from expected by >0.5%.
- **Addresses:** deposit_address, deposit_extra_id (if any), payout_address. No refund_address shown (privacy by design - not logged in dashboard).
- **Event timeline:** Vertical list of all status transitions with timestamps. Each entry shows duration since previous state.
- **Tx hashes:** `tx_in_hash` and `tx_out_hash` with block explorer deep-links (auto-detected by asset type: BTC→mempool.space, ETH→etherscan.io, XMR→xmrchain.net, etc.).
- **Webhook deliveries (for this order):** Inline table of all deliveries related to this order_id. Link to full delivery detail.
- **Refund action panel:** Visible only when `status = refund_required` AND `refund_capability = programmatic`. Shows remaining window (countdown timer). Two action buttons: "Refund to Address" and "Convert to Stable." "Convert to Stable" requires selecting a `stable_target`. Confirm dialog before submitting. On submit: calls `POST /partner/v1/order/{id}/refund-action`. Button disabled if window has expired; shows "Refund window expired - order will proceed with default preference."
- **Raw JSON tab:** Full order object as returned by the public API. Syntax-highlighted, copyable.

**Empty state:** "No orders match your filters."

**Error state:** If order detail fetch fails: "Could not load order. Retry or contact support if the problem persists." Show order ID for reference.

---

### 4.8 Providers

**Purpose:** Control which of Privata's 50+ exchange providers are active for the partner's orders, and view refund capabilities.

**Grid / table columns:** Provider name, logo, status toggle (enabled/disabled), refund_mode badge (programmatic / ticket / none), `window_hours` or `sla_hours`, `fee_pct`, `fee_network` flag, last updated.

**Enable/disable:** Toggle immediately applies to new orders routed through best-rate logic. Does not cancel in-flight orders. Confirmation toast on change.

**Fee config (per provider, P1):** Partner can optionally set a per-provider markup override (0-300 bps) in addition to the global markup configured at order time.

**Refund capability matrix:** Read-only data pulled from `GET /providers/refund-capability` (cached hourly). Shows which providers support programmatic refund vs. manual ticket, and the window/fee parameters.

**Edge case:** If all providers are disabled: warning banner "Your integration has no active providers. All quote requests will return NO_PROVIDERS_AVAILABLE." Cannot disable the last provider without explicit confirmation.

**Empty state:** Providers list always populated (from Privata aggregate). If API unavailable: skeleton table with "Provider data temporarily unavailable."

---

### 4.9 Revenue / Payouts

**Purpose:** Earnings transparency and controlled withdrawal for Finance role.

**Overview tab:**
- Pending balance (USD, updates real-time).
- Current rev-share % (from org config; contact Privata to change).
- Markup earnings breakdown (separate from rev-share, P1).
- Month-to-date and all-time totals.
- Last payout date + amount.

**Payout history table:** Columns: date initiated, amount USD, currency, payout amount (in target currency), tx_hash (with block explorer link), status (pending / sent / confirmed).

**Payout address management:**
- Add address: enter address + currency (USDT-TRC20, USDT-ERC20, USDC-ERC20, USDC-SOL). Address validated for format (regex + checksum).
- Verification: a micro-payout of 0.01 USDT is sent to the address; partner must confirm receipt within 24h. Unverified addresses cannot be set as default.
- Default address: only one per currency pair.
- Remove address: cannot remove default until another is promoted.
- All address changes write to audit log.

**Manual payout request:**
1. Minimum threshold: $500 (configurable per partner; default $500, max request $50K).
2. Select payout address (must be verified).
3. Enter amount (or "request full balance").
4. **HMAC confirmation step:** System generates a one-time challenge string. Partner must compute `HMAC-SHA256(webhook_secret, challenge)` and paste the result. This proves the requestor controls the webhook secret, not just the dashboard session. On mismatch: rejected; audit log entry. Alternative for partners without webhook secret: confirmation via email OTP (6-digit, 10-minute TTL).
5. On successful confirmation: payout job enqueued. Estimated processing time shown (based on asset network).
6. Finance role can initiate. Owner can initiate. Admin cannot (by design - financial control separation).

**Edge case:** If pending balance < threshold: button shown but disabled with tooltip showing current balance and threshold.

---

### 4.10 SLA

**Purpose:** Give partners visibility into their uptime tier, compensation accrual, and incident history. Reference: [SLA guide](/guides/sla).

**Live uptime gauge:** Percentage uptime current calendar month, measured from partner's registered prober against `GET /partner/v1/health`. Compared against tier target (99.5% pilot, 99.9% post-pilot / Enterprise).

**Compensation accrual:** If uptime < tier threshold, show accrued compensation amount for the month. Calculated server-side from the formula in the SLA guide. "Applied to next payout cycle automatically."

**Tier indicator:** Pilot / Post-pilot / Enterprise badge. If approaching Enterprise threshold (≥25K orders/month), show upgrade nudge.

**Prober config helper:** Instructions to add Privata probe IPs to no-rate-limit allowlist; one-click "Submit probe IPs" form (sends email to partners@privataswap.com with the IPs on file).

**Incident log table:** incident_id, start_at, end_at (null if open), duration, affected tier, compensation_applied. Click to expand RCA link (if available).

**Empty state (no incidents):** "No incidents recorded. Your integration is healthy."

**Error state:** If uptime data unavailable: "Uptime data is syncing - check back in a few minutes."

---

### 4.11 Team

**Purpose:** Manage organization members and their access levels.

**Member list columns:** Avatar, name, email, role badge, last_login, status (active / invited / deactivated).

**Invite:** Enter email, select role. Email sent via Resend with 7-day expiry link. Pending invites shown in member list with "Pending" badge and resend/cancel options.

**Role change:** Dropdown per member row. Owner can change any role. Admin can change any role except Owner and their own.

**Remove:** Owner only. Shows confirmation "This will immediately revoke all of [name]'s access." If the removed user has active sessions, those sessions are invalidated immediately.

**Edge case:** Attempting to remove the last Owner: blocked with "You cannot remove the last Owner. Promote another member first."

---

### 4.12 Audit Log

**Purpose:** Immutable record of all actions taken by any user in the organization.

**Columns:** Timestamp, actor (email), action code (e.g., `api_key.created`, `order.refund_action`, `payout.requested`), target (key prefix, order_id, member email), actor IP (partial mask), session_id (last 8 chars).

**Filters:** Actor, action type, date range.

**Export:** NDJSON or CSV. Exports are themselves logged (`audit_log.exported`).

**Integrity:** The table is append-only; the `dashboard_app` Postgres role has `INSERT, SELECT` only. See §5.9 for the full integrity model.

**Retention:** 2 years online in Postgres, then archived to cold storage (S3 Glacier or equivalent). Soft-delete is not available for audit records.

**Empty state:** "No activity recorded yet."

---

### 4.13 Billing

**Purpose:** Display current plan tier and provide self-service payout configuration. Plan changes are sales-assisted.

**Current plan panel:** Plan tier (Starter / Pro / Enterprise) displayed as read-only. No subscriptions managed here.

**Revenue summary:** Link to `/revenue` for earnings overview and payout history.

**Payout history:** Link to `/revenue/history` for on-chain payout records.

**Upgrade flow:**
- "Contact us to upgrade" - directs to partners@privataswap.com. Enterprise upgrade is not self-serve; requires Privata sales confirmation.

**Empty state (new partner):** "Your current plan: Starter. Contact partners@privataswap.com to discuss plan options."

---

### 4.14 Settings

**Organization:** Legal name, public-facing partner name, logo upload (PNG/SVG, max 512KB), timezone, notification email.

**Security:** TOTP and WebAuthn management (see §4.1), active sessions list (device, last_seen, IP, "Revoke" button per session), new-device email preference toggle.

**Scoped tokens:** Personal access tokens for CI/CD automation. Separate from org API keys - dashboard-scoped only, not usable against `api.privataswap.com/partner/v1`. Scopes: `dashboard:read`, `dashboard:write`.

**Webhook IP allowlist:** Outbound webhook delivery IPs (published by Privata, currently static range). Partners can add these to their firewall allowlists. One-click copy of current Privata egress IPs.

**GDPR:** Data export and deletion requests are handled manually via privacy@privataswap.com (see §5.10). No self-service UI in v1.

---

## 5. Auth + Security Architecture

### 5.1 Session Model

- Sessions use **httpOnly, Secure, SameSite=Lax cookies** containing an opaque session token (32-byte random, stored in Redis with TTL).
- JWTs are not used; sessions only.
- Session TTL: 8 hours idle timeout, 30-day absolute maximum. Refresh on activity.
- Session token is rotated on privilege escalation (2FA completion, role change). Old token immediately invalidated (prevents session fixation).
- All session records stored in Redis with `session:{token_hash}` key. Value: `{user_id, org_id, role, device_id, created_at, last_seen_at, ip}`.

### 5.2 Authentication Methods

| Method | Implementation | Notes |
|---|---|---|
| Email + password | bcrypt (cost 12), HIBP k-anon check on signup | Argon2id considered for P1 |
| TOTP | RFC 6238, 30-second window, +/-1 step tolerance | Secret encrypted with per-user DEK |
| WebAuthn / FIDO2 | `@simplewebauthn/server`, RP ID = `dashboard.privataswap.com` | Up to 5 credentials per user |
| Backup codes | 10 x 8-char alphanumeric (zbase32), SHA-256 stored | Shown once, invalidated on use |

### 5.3 CSRF Protection

- **Double-submit cookie pattern**: A `csrf_token` cookie (httpOnly=false, SameSite=Lax) set on page load; all mutating requests must send the same value in `X-CSRF-Token` header. Server compares constant-time.
- tRPC mutations automatically inject the header via the client-side transport.
- State-changing GET requests are forbidden by convention (enforced in router middleware).

### 5.4 Rate Limits

| Surface | Limit | Window | Action on breach |
|---|---|---|---|
| Login attempts | 10 per IP | 15 minutes | IP-level 30-minute block |
| Login attempts | 5 per email | 15 minutes | Account lock + email notification |
| 2FA attempts | 5 per session | 30 minutes | Session invalidated, re-auth required |
| API key create | 20 per org | 1 hour | 429 |
| Webhook replay | 10 per endpoint | 1 hour | 429 |
| Password reset requests | 3 per email | 1 hour | Silent |
| Payout requests | 2 per org | 24 hours | 429 |

Rate limits implemented in Redis using token bucket (per IP) and sliding window (per account/email).

### 5.5 Account-Takeover Protections

- **New device email confirmation:** If `device_id` not in `user_known_devices`, a 6-digit OTP is required before session is issued. OTP sent via Resend; 10-minute TTL; 3 attempts before invalidation.
- **Geographic anomaly detection:** Geolocate login IP (MaxMind GeoIP2). If country differs from the modal country of last 30 logins, trigger email confirmation. Threshold: >2 sigma from prior distribution.
- **Concurrent session limit:** Max 10 active sessions per user. Oldest session revoked when limit reached.
- **Credential stuffing mitigation:** Login endpoint uses per-IP token bucket + global honeypot detection (fail2ban-style pattern: >1000 unique email attempts from one /24 in 5 minutes → IP range block for 1h, logged).
- **Audit log for all auth events:** `login.success`, `login.failed`, `login.new_device`, `login.geo_anomaly`, `2fa.enrolled`, `2fa.failed`, `session.revoked`.

### 5.6 Secret Storage

- **API key secrets:** Generated as 32-byte CSPRNG. Stored as `HMAC-SHA256(server_hmac_key, key)` for constant-time lookup. The `server_hmac_key` is a 256-bit key loaded from the deployment secret store (Doppler in current Privata infrastructure) into process memory at boot. The prefix (`pk_live_abc1...12`) is stored plaintext for UI display.
- **Webhook secrets:** Stored encrypted using libsodium `crypto_secretbox_easy` (XSalsa20-Poly1305) with a per-row nonce; the master key is the same Doppler-managed 256-bit key. Two secrets per endpoint during rotation (active + pending), each with its own encrypted record.
- **TOTP secrets:** Encrypted with a per-user DEK derived as `HKDF(master_key, user_id)`. All keys rotated annually via documented procedure.
- **Payout addresses:** Stored plaintext (they are public-facing addresses by nature). However, the HMAC confirmation step for payouts means possession of a session alone is insufficient.
- **Environment variable hygiene:** No secrets in application code or container images. All loaded at runtime from Doppler (production) or `.env.local` (development, never committed).

### 5.7 IDOR Prevention

- All resource IDs are UUIDs v4, not sequential integers. Enumeration is computationally infeasible.
- Every data-fetching query includes `WHERE organization_id = :current_org_id`. The `current_org_id` is sourced from the verified session, never from the request body.
- tRPC procedures enforce this via a shared `withOrgScope(ctx, id)` helper that throws `FORBIDDEN` if the resource does not belong to `ctx.session.orgId`.
- Webhook delivery IDs, order IDs, and audit log IDs are all org-scoped at the query level.

### 5.8 Webhook Replay Attack Prevention

- Each webhook payload includes an `event_id` and a timestamp `t`. The dashboard verifies that `t` is within ±300 seconds of now before processing a "replay test" (if the dashboard itself sends test webhooks).
- For delivery replay actions: the replay creates a **new** delivery record with a new signature using the **current** active secret. Old deliveries are read-only records.
- Idempotency: the partner's system is expected to deduplicate on `event_id` (documented in [Webhooks guide](/guides/webhooks)).

### 5.9 Audit Log Integrity

The `audit_log` table is append-only. The Postgres role used by the dashboard backend (`dashboard_app`) is granted `INSERT, SELECT` and explicitly `REVOKE UPDATE, DELETE ON audit_log`. Schema migrations and operational fixes require a separate `dashboard_admin` role with elevated privileges, used only via break-glass procedure (logged out-of-band). Daily DB backups provide point-in-time recovery for tamper detection by comparison. Cryptographic row chaining and signed checkpoints are not implemented in v1; this is sufficient given the threat model (insider tampering by app-level compromise, not DB-admin compromise).

### 5.10 GDPR

Data subject access requests (GDPR Art. 15) and erasure requests (Art. 17) are handled manually via privacy@privataswap.com. Response within 30 days as required. A self-service export/delete UI is not in scope for v1; volume is expected to be less than 1 request per month and a manual procedure is operationally adequate. Logged in `gdpr_requests` table for audit.

- **Data residency:** All Postgres data in EU-West-1 (Ireland) by default. Enterprise partners can request dedicated EU or US clusters.

---

## 6. Data Model

### 6.1 Postgres Schema

```sql
-- Organizations
CREATE TABLE organizations (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug          TEXT NOT NULL UNIQUE,              -- url-safe identifier
  name          TEXT NOT NULL,
  legal_name    TEXT,
  country       CHAR(2),                           -- ISO-3166
  business_type TEXT,                              -- wallet/cex/payment/other
  website       TEXT,
  logo_url      TEXT,
  status        TEXT NOT NULL DEFAULT 'onboarding', -- onboarding/active/suspended/deleted
  tier          TEXT NOT NULL DEFAULT 'standard',   -- standard/enterprise_2500/enterprise_3500
  revenue_share_pct INTEGER NOT NULL DEFAULT 30,
  allow_markup  BOOLEAN NOT NULL DEFAULT TRUE,
  max_markup_bps INTEGER NOT NULL DEFAULT 300,
  payout_threshold_usd NUMERIC(12,2) NOT NULL DEFAULT 500.00,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at    TIMESTAMPTZ,
  onboarding_completed_at TIMESTAMPTZ
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_status ON organizations(status) WHERE deleted_at IS NULL;

-- Users (dashboard users, not end-users of the wallet)
CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           TEXT NOT NULL UNIQUE,
  email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
  password_hash   TEXT,                            -- NULL if password not set
  totp_secret_enc BYTEA,                           -- AES-256-GCM encrypted
  totp_enabled    BOOLEAN NOT NULL DEFAULT FALSE,
  webauthn_credentials JSONB NOT NULL DEFAULT '[]',
  backup_codes_hashes TEXT[],                      -- SHA-256 hashes of remaining codes
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email) WHERE deleted_at IS NULL;

-- Organization membership
CREATE TABLE organization_members (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  role        TEXT NOT NULL CHECK (role IN ('owner','admin','developer','finance','readonly')),
  invited_by  UUID REFERENCES users(id),
  invited_at  TIMESTAMPTZ,
  joined_at   TIMESTAMPTZ,
  status      TEXT NOT NULL DEFAULT 'invited' CHECK (status IN ('invited','active','deactivated')),
  UNIQUE (org_id, user_id)
);

CREATE INDEX idx_org_members_org ON organization_members(org_id) WHERE status = 'active';
CREATE INDEX idx_org_members_user ON organization_members(user_id);

-- API Keys
CREATE TABLE api_keys (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name          TEXT NOT NULL,
  key_hash      TEXT NOT NULL UNIQUE,              -- HMAC-SHA256(server_hmac_key, raw_key)
  key_prefix    TEXT NOT NULL,                     -- first 16 chars, e.g. pk_live_abc12345
  environment   TEXT NOT NULL CHECK (environment IN ('live','test')),
  scopes        TEXT[] NOT NULL DEFAULT '{orders:rw}',
  ip_allowlist  CIDR[] NOT NULL DEFAULT '{}',
  status        TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','rotating','revoked')),
  created_by    UUID REFERENCES users(id),
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_used_at  TIMESTAMPTZ,
  rotated_at    TIMESTAMPTZ,
  revoked_at    TIMESTAMPTZ,
  expires_at    TIMESTAMPTZ,
  -- Rotation state
  rotation_new_key_hash TEXT,                      -- populated during 5-min grace
  rotation_started_at   TIMESTAMPTZ,
  rotation_expires_at   TIMESTAMPTZ
);

CREATE INDEX idx_api_keys_org ON api_keys(org_id) WHERE status != 'revoked';
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_expiry ON api_keys(expires_at) WHERE expires_at IS NOT NULL AND status = 'active';

-- Webhook endpoints
CREATE TABLE webhook_endpoints (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  alias         TEXT NOT NULL,
  url           TEXT NOT NULL,
  endpoint_type TEXT NOT NULL CHECK (endpoint_type IN ('order','ops','custom')),
  status        TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','paused','disabled')),
  failure_count INTEGER NOT NULL DEFAULT 0,
  paused_at     TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  created_by    UUID REFERENCES users(id)
);

CREATE INDEX idx_webhook_endpoints_org ON webhook_endpoints(org_id);

-- Webhook secrets (two records during rotation: active + pending)
CREATE TABLE webhook_secrets (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id   UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
  secret_enc    BYTEA NOT NULL,                    -- AES-256-GCM encrypted
  iv            BYTEA NOT NULL,
  kms_key_id    TEXT NOT NULL,
  is_active     BOOLEAN NOT NULL DEFAULT TRUE,
  is_pending    BOOLEAN NOT NULL DEFAULT FALSE,    -- TRUE during 5-min grace window
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at    TIMESTAMPTZ,                       -- NULL = permanent; set during rotation grace
  revoked_at    TIMESTAMPTZ
);

CREATE INDEX idx_webhook_secrets_endpoint ON webhook_secrets(endpoint_id) WHERE revoked_at IS NULL;

-- Webhook deliveries
CREATE TABLE webhook_deliveries (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id      UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
  org_id           UUID NOT NULL,                  -- denormalized for partition pruning
  event_type       TEXT NOT NULL,
  event_id         TEXT NOT NULL,                  -- from Privata API (idempotency key)
  order_id         TEXT,                           -- prv-... if applicable
  attempt_number   INTEGER NOT NULL DEFAULT 1,
  payload_hash     TEXT NOT NULL,                  -- SHA-256 of raw body (for replay identity)
  request_body     TEXT,                           -- truncated at 10KB
  response_code    INTEGER,
  response_body    TEXT,                           -- truncated at 1KB
  response_time_ms INTEGER,
  status           TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','success','failed','timeout')),
  error_detail     TEXT,
  next_retry_at    TIMESTAMPTZ,
  delivered_at     TIMESTAMPTZ,
  is_replay        BOOLEAN NOT NULL DEFAULT FALSE,
  replayed_from    UUID REFERENCES webhook_deliveries(id),
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_deliveries_endpoint ON webhook_deliveries(endpoint_id, created_at DESC);
CREATE INDEX idx_deliveries_org ON webhook_deliveries(org_id, created_at DESC);
CREATE INDEX idx_deliveries_order ON webhook_deliveries(order_id) WHERE order_id IS NOT NULL;
CREATE INDEX idx_deliveries_retry ON webhook_deliveries(next_retry_at) WHERE status = 'pending' AND next_retry_at IS NOT NULL;

-- Retention: webhook_deliveries older than 90 days → archive / delete.
-- Partitioned by created_at month (range partitioning) for efficient range deletes.

-- SSE streams (ephemeral, cleared on pod restart)
CREATE TABLE sse_streams (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        UUID NOT NULL REFERENCES organizations(id),
  order_id      TEXT NOT NULL,
  connected_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_event_id TEXT,
  resume_source TEXT,                              -- buffer/snapshot/fresh
  client_ip_hash TEXT NOT NULL,                   -- SHA-256(ip + daily_salt)
  pod_id        TEXT NOT NULL,                    -- for cleanup on pod shutdown
  closed_at     TIMESTAMPTZ
);

CREATE INDEX idx_sse_org ON sse_streams(org_id) WHERE closed_at IS NULL;
-- TTL: rows with closed_at older than 24h purged by cron.

-- Orders mirror (partner-scoped, sourced from Privata API)
CREATE TABLE orders (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id              UUID NOT NULL REFERENCES organizations(id),
  order_id            TEXT NOT NULL UNIQUE,        -- prv-...
  partner_order_ref   TEXT,
  provider_id         TEXT NOT NULL,
  provider_order_id   TEXT,
  from_asset          TEXT NOT NULL,
  from_network        TEXT NOT NULL,
  to_asset            TEXT NOT NULL,
  to_network          TEXT NOT NULL,
  amount_in           NUMERIC(30,18) NOT NULL,
  amount_in_usd       NUMERIC(18,2),
  amount_out_expected NUMERIC(30,18),
  amount_out_actual   NUMERIC(30,18),
  markup_bps          INTEGER NOT NULL DEFAULT 0,
  status              TEXT NOT NULL,
  failed_reason_code  TEXT,
  refund_capability   TEXT,
  tx_in_hash          TEXT,
  tx_out_hash         TEXT,
  deposit_address     TEXT NOT NULL,
  payout_address      TEXT NOT NULL,
  expires_at          TIMESTAMPTZ,
  created_at          TIMESTAMPTZ NOT NULL,
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at        TIMESTAMPTZ,
  timeline            JSONB NOT NULL DEFAULT '[]',
  partner_earnings_usd NUMERIC(18,8),
  raw_api_response    JSONB                        -- full order object snapshot
);

CREATE INDEX idx_orders_org_created ON orders(org_id, created_at DESC);
CREATE INDEX idx_orders_order_id ON orders(order_id);
CREATE INDEX idx_orders_status ON orders(org_id, status);
CREATE INDEX idx_orders_provider ON orders(org_id, provider_id);
CREATE INDEX idx_orders_partner_ref ON orders(org_id, partner_order_ref) WHERE partner_order_ref IS NOT NULL;
-- Retention: orders older than 3 years → cold archive. Hot partition: 90 days.

-- Payouts
CREATE TABLE payouts (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id           UUID NOT NULL REFERENCES organizations(id),
  requested_by     UUID REFERENCES users(id),
  amount_usd       NUMERIC(18,2) NOT NULL,
  payout_address   TEXT NOT NULL,
  payout_currency  TEXT NOT NULL,                  -- USDT-TRC20 etc.
  payout_amount    NUMERIC(30,18),                 -- null until sent
  tx_hash          TEXT,
  status           TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','processing','sent','confirmed','failed')),
  hmac_challenge   TEXT NOT NULL,                  -- one-time challenge used for confirmation
  hmac_confirmed   BOOLEAN NOT NULL DEFAULT FALSE,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
  processed_at     TIMESTAMPTZ,
  sent_at          TIMESTAMPTZ,
  confirmed_at     TIMESTAMPTZ
);

CREATE INDEX idx_payouts_org ON payouts(org_id, created_at DESC);

-- Payout addresses
CREATE TABLE payout_addresses (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id       UUID NOT NULL REFERENCES organizations(id),
  address      TEXT NOT NULL,
  currency     TEXT NOT NULL,
  is_default   BOOLEAN NOT NULL DEFAULT FALSE,
  is_verified  BOOLEAN NOT NULL DEFAULT FALSE,
  verified_at  TIMESTAMPTZ,
  added_by     UUID REFERENCES users(id),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  deleted_at   TIMESTAMPTZ,
  UNIQUE (org_id, address, currency)
);

CREATE INDEX idx_payout_addresses_org ON payout_addresses(org_id) WHERE deleted_at IS NULL;

-- SLA incidents
CREATE TABLE sla_incidents (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id          UUID,                            -- NULL = platform-wide
  started_at      TIMESTAMPTZ NOT NULL,
  resolved_at     TIMESTAMPTZ,
  uptime_pct      NUMERIC(5,2),                    -- calculated at resolution
  tier            TEXT NOT NULL,
  compensation_pct INTEGER,                        -- 0/10/25/100
  compensation_usd NUMERIC(18,2),
  rca_url         TEXT,
  source          TEXT NOT NULL DEFAULT 'external_prober', -- external_prober/internal_monitor
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sla_incidents_org ON sla_incidents(org_id, started_at DESC);

-- Audit log (append-only)
CREATE TABLE audit_log (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id      UUID,                                -- NULL for system events
  actor_id    UUID,                                -- NULL for automated actions
  actor_email TEXT,                                -- denormalized for immutability
  action      TEXT NOT NULL,                       -- e.g. api_key.created
  target_type TEXT,
  target_id   TEXT,
  meta        JSONB NOT NULL DEFAULT '{}',
  ip          INET,
  session_suffix TEXT,                             -- last 8 chars of session token
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_org ON audit_log(org_id, created_at DESC);
CREATE INDEX idx_audit_actor ON audit_log(actor_id, created_at DESC);
-- No UPDATE or DELETE on audit_log. Enforced at Postgres role level (dashboard_app has INSERT, SELECT only).

-- Partner plans
CREATE TABLE partner_plans (
  org_id               UUID PRIMARY KEY REFERENCES organizations(id),
  tier                 TEXT NOT NULL CHECK (tier IN ('starter','pro','enterprise')) DEFAULT 'starter',
  monthly_volume_cap_usd NUMERIC,
  fee_bps              INT NOT NULL DEFAULT 50,
  updated_at           TIMESTAMPTZ
);

-- GDPR requests
CREATE TABLE gdpr_requests (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id         UUID REFERENCES organizations(id),
  user_email     TEXT,
  request_type   TEXT CHECK (request_type IN ('export','delete')),
  submitted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at   TIMESTAMPTZ,
  notes          TEXT
);

-- Known user devices (for new-device detection)
CREATE TABLE user_known_devices (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  device_hash   TEXT NOT NULL,                     -- SHA-256(UA + screen_hash + daily_salt)
  first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  confirmed     BOOLEAN NOT NULL DEFAULT FALSE,
  UNIQUE (user_id, device_hash)
);

CREATE INDEX idx_known_devices_user ON user_known_devices(user_id);
```

### 6.2 Indexes Summary

All foreign keys are indexed. High-traffic queries use composite indexes. `orders` and `webhook_deliveries` are range-partitioned by `created_at` (monthly) to support efficient bulk deletes and partition pruning in date-range queries.

### 6.3 Retention Policies

| Table | Hot retention | Archive policy |
|---|---|---|
| `orders` | 90 days in hot partition | 3 years in cold partition; then S3 Glacier |
| `webhook_deliveries` | 90 days | Delete (not archived; low compliance value) |
| `audit_log` | 2 years online | S3 Glacier; indefinite for legal compliance |
| `sse_streams` | 24 hours (closed rows) | Delete |
| `sla_incidents` | Indefinite online | - |
| `payouts` | Indefinite online | - |

---

## 7. Tech Stack Proposal

| Layer | Choice | Rationale |
|---|---|---|
| Frontend framework | Next.js 15 (App Router) | RSC for initial load performance; streaming SSR; native Vercel/AWS deployment |
| Frontend RPC | tRPC v11 | End-to-end type safety without codegen; integrates directly with React Query |
| ORM | Drizzle ORM | SQL-first, zero-runtime-overhead, native Postgres types, excellent migration story |
| Database | Postgres 16 (RDS Multi-AZ) | ACID, range partitioning, `pg_trgm` for fuzzy order search, row-level security possible |
| Cache / rate-limit | Redis (ElastiCache) | Token bucket rate limits, session store, realtime channel presence, Pub/Sub |
| Email | Resend | Excellent deliverability, React Email templates, simple SDK, webhook for bounce tracking |
| Realtime | SSE (existing Privata SSE broker, see §9) | Reuses existing infrastructure; no third-party realtime vendor |
| Secret management | Doppler + libsodium (libsodium for at-rest encryption; Doppler for environment secret distribution) | Key distribution without AWS dependency; libsodium XSalsa20-Poly1305 for webhook secrets |
| Error tracking | Privata self-hosted error log endpoint (POST to /api/dashboard/_telemetry; structured logs to existing Privata Loki stack) | No third-party data egress |
| Product analytics | None (privacy contradiction with our brand). Page view counts only, server-side, stored in `pageview_log` table; no client-side trackers, no session replay. | Privacy by design |
| Infrastructure | AWS (ECS Fargate or Vercel) | Next.js on Vercel for frontend; tRPC API on Fargate for latency-sensitive paths |

---

## 8. Internal API Surface (Dashboard Backend)

The dashboard backend exposes tRPC procedures. These are **not** part of `api.privataswap.com/partner/v1`; they are internal to the dashboard application. All procedures require an authenticated session unless marked `[public]`.

```
auth
  auth.signup                    email, password → session
  auth.login                     email, password → session or 2fa_challenge
  auth.login.complete2fa         session_token, totp_code | webauthn_response → session
  auth.logout                    → void
  auth.requestPasswordReset      email → void
  auth.confirmPasswordReset      token, new_password → void
  auth.verifyEmail               otp → void
  auth.webauthn.registerStart    → options
  auth.webauthn.registerFinish   response → credential
  auth.webauthn.loginStart       email → options
  auth.webauthn.loginFinish      response → session

onboarding
  onboarding.getState            → current step + data
  onboarding.saveOrganization    name, business_email, jurisdiction → void
  onboarding.saveVolume          volume_band → void
  onboarding.saveUseCase         use_case_text → void
  onboarding.acceptTerms         accepted → void
  onboarding.complete            → api_key (test, one-time)

dashboard
  dashboard.getHomeKPIs          → orders_today, volume_24h, success_rate, latency_p50, latency_p99, top_provider
  dashboard.getOrdersPerHour     → [{hour, count}] (24 points)
  dashboard.getRecentDeliveries  → [delivery summary × 10]
  dashboard.getActiveSSECount    → integer
  dashboard.getOpenIncident      → incident | null

api_keys
  api_keys.list                  → [key summary]
  api_keys.create                name, environment, scopes, ip_allowlist, expires_at → {key_prefix, raw_key (once)}
  api_keys.update                id, name, scopes, ip_allowlist, expires_at → void
  api_keys.rotateStart           id → {new_prefix, raw_key, grace_expires_at}
  api_keys.rotateComplete        id → void (early finalize)
  api_keys.revoke                id → void
  api_keys.getDetail             id → key detail

webhooks
  webhooks.listEndpoints         → [endpoint summary]
  webhooks.createEndpoint        alias, url, endpoint_type → {endpoint, raw_secret (once)}
  webhooks.updateEndpoint        id, alias, url → void
  webhooks.rotateSecretStart     id → {grace_expires_at, new_secret (once)}
  webhooks.deleteEndpoint        id → void
  webhooks.resumeEndpoint        id → void (calls /partner/v1/webhook/resume)
  webhooks.listDeliveries        endpoint_id, filters → paginated [delivery]
  webhooks.getDelivery           id → delivery detail
  webhooks.retryDelivery         id → new_delivery_id
  webhooks.replayDelivery        id → new_delivery_id

sse
  sse.listActive                 → [stream summary]

orders
  orders.search                  org_id, filters, page, page_size → paginated [order]
  orders.getDetail               order_id → order detail + deliveries
  orders.exportCSV               filters → job_id (async)
  orders.getExportStatus         job_id → {status, download_url}
  orders.triggerRefundAction     order_id, action, stable_target → void

providers
  providers.list                 → [provider + refund capability]
  providers.setEnabled           provider_code, enabled → void
  providers.setFeeConfig         provider_code, markup_bps → void (P1)

revenue
  revenue.getOverview            → pending_balance, rev_share_pct, mtd_earnings, total_paid
  revenue.getPayoutHistory       page → paginated [payout]
  revenue.listPayoutAddresses    → [address]
  revenue.addPayoutAddress       address, currency → address_id
  revenue.verifyPayoutAddress    address_id, confirmed → void
  revenue.setDefaultAddress      address_id → void
  revenue.removePayoutAddress    address_id → void
  revenue.initiatePayoutRequest  amount, address_id → {challenge}
  revenue.confirmPayoutRequest   payout_id, hmac_response | email_otp → void

sla
  sla.getOverview                → uptime_pct, tier_target, accrued_compensation, open_incident
  sla.getIncidents               page → paginated [incident]

team
  team.listMembers               → [member]
  team.inviteMember              email, role → void
  team.updateMemberRole          member_id, role → void
  team.deactivateMember          member_id → void
  team.removeMember              member_id → void
  team.resendInvite              member_id → void
audit_log
  audit_log.list                 filters, page → paginated [entry]
  audit_log.export               filters → job_id (async)

billing
  billing.getSubscription        → plan tier, usage
  billing.requestUpgrade         message → void (creates CRM lead)

settings
  settings.getOrg                → org settings
  settings.updateOrg             name, logo_url, timezone, notification_email → void
  settings.uploadLogo            file → logo_url
  settings.getSessions           → [session]
  settings.revokeSession         session_id → void
  settings.revokeAllOtherSessions → void
  settings.gdprExport            → job_id
  settings.gdprDelete            confirmation_text → void
  settings.listPersonalTokens    → [token]
  settings.createPersonalToken   name, scopes → raw_token (once)
  settings.revokePersonalToken   id → void
```

---

## 9. Realtime Architecture

### 9.1 Infrastructure

Privata already operates a Server-Sent Events broker (`api.privataswap.com/sse`) used by the consumer swap UI for order status updates. The dashboard reuses this infrastructure: on login, the dashboard client opens an SSE connection to `/api/dashboard/sse` (dashboard backend), which authenticates the cookie session, looks up the user's `org_id`, and subscribes server-side to the internal NATS subjects `org.{org_id}.>` (orders, kpis, sla, webhooks). Events are forwarded to the client as named SSE events. Reconnect via `Last-Event-ID` header. No third-party realtime vendor. Capacity: a single SSE process handles ~10k concurrent connections; horizontal scaling via subject-based sharding on `org_id` modulo node count.

### 9.2 Event Names

SSE event names used by the dashboard:

```
order.updated        - New or updated order event for this org
kpi.tick             - KPI aggregate update (orders_today, volume, success_rate, etc.)
sla.incident         - SLA incident open/close
webhook.paused       - Webhook endpoint auto-paused
sse.gauge            - SSE concurrency gauge update
```

### 9.3 Event Flow

1. Privata's public API fires a webhook to the dashboard's internal receiver endpoint (`/internal/privata-webhook`).
2. The receiver verifies HMAC, upserts the `orders` table, writes a `webhook_deliveries` row, updates `webhook_endpoints.failure_count`.
3. Receiver publishes to NATS subject `org.{org_id}.orders`; the SSE broker forwards to connected dashboard clients as SSE event `order.updated`. Payload is the order summary (not full raw API response - reduces payload size).
4. KPI aggregates are recalculated as materialized values in Redis (`kpi:{org_id}:{date}` hash). Updated atomically on each order event. Published to NATS subject `org.{org_id}.kpis`, forwarded as SSE event `kpi.tick`.
5. Client opens SSE connection on page load. tRPC fetches initial data (SSR or client); SSE delivers incremental updates.

### 9.4 Concurrency Gauge

The SSE concurrency gauge on the SSE page polls `sse_streams` table every 30 seconds (not realtime - concurrency count changes slowly). This avoids a hot Redis key for a low-priority metric.

---

## 10. Notifications

### 10.1 In-App Notifications

Delivered via the same SSE connection as event `notification.created` and stored in a `notifications` table (not in data model above - P1 addition).

| Trigger | In-app | Severity |
|---|---|---|
| SLA breach approaching (uptime crosses threshold) | Banner on SLA page + bell icon | Warning |
| SLA breach confirmed (incident opened) | Persistent banner on all pages | Critical |
| Webhook endpoint auto-paused | Banner on Webhooks page | Error |
| Webhook delivery error burst (>10 failures in 1h) | Toast notification | Warning |
| API key expiry approaching (7 days, 1 day) | Banner on API Keys page | Warning |
| Payout sent (tx_hash available) | Toast | Info |
| New team member joined | Toast | Info |
| New login from unknown device | Email only (too sensitive for in-app) | - |

### 10.2 Email Notifications

Sent via Resend. Recipients: technical contact (from onboarding) for ops events; member's own email for account events.

| Trigger | Recipient | Template |
|---|---|---|
| Webhook paused | Technical contact | ops-webhook-paused |
| SLA breach opened | Technical contact + Owner | ops-sla-breach |
| API key expiry 7 days | Key creator + Admins | ops-key-expiry |
| API key expiry 1 day | Key creator + Admins | ops-key-expiry-urgent |
| Payout sent | Finance role members + Owner | finance-payout-sent |
| New login from unknown device | The logging-in user | security-new-device |
| 2FA disabled on account | Account owner | security-2fa-disabled |
| New team member invited | Invitee | team-invite |
| Account locked (failed auth) | Account owner | security-lockout |
| GDPR export ready | Requestor | gdpr-export-ready |

**Email preferences:** Users can opt out of info-level notifications (payout sent, new member). Security and ops emails are mandatory (cannot opt out while holding an Owner or Admin role).

### 10.3 Source of Ops Events

The dashboard subscribes to Privata's ops webhook channel (`endpoint_type = ops`). The `internal/privata-webhook` receiver processes ops events and fires the appropriate notification rules.

---

## 11. Mobile / Responsive Strategy

### 11.1 Approach

Mobile-first CSS (Tailwind breakpoints: base = mobile, `md` = tablet, `lg` = desktop). Dashboard is a B2B tool; primary use is desktop. Mobile is a first-class citizen for specific high-urgency flows.

### 11.2 Priority Mobile Flows (P0)

1. **Login + 2FA:** Full login flow including TOTP entry, WebAuthn prompt, and new-device confirmation. Tap targets ≥44px. Autofill compatible for password managers.
2. **Order detail:** Summary card, event timeline, tx hashes with tap-to-copy. Refund action panel on scroll. No horizontal scroll - all content reflows.
3. **Webhook delivery view:** Delivery log readable on narrow screen; request/response bodies in collapsible panels. "Resume webhook" action accessible.

### 11.3 PWA Support

Out of scope for v1. The dashboard is a server-rendered responsive web app; partners use it from desktop browsers in their operations console.

### 11.4 Out of Scope (v1)

- Native iOS or Android applications.
- Offline functionality beyond shell caching.
- Tablet-optimized two-column layouts (deferred to P2 UX polish).
- Progressive web app (PWA) installation, browser push notifications, mobile app.

---

## 12. Build Phases (P0 / P1 / P2)

| Phase | Feature scope | Effort estimate | Notes |
|---|---|---|---|
| **P0 - MVP (launch blocker)** | Auth (email+password+TOTP), Onboarding (KYB-lite), Home Dashboard (KPIs, no realtime), API Keys (CRUD, rotation, revoke), Webhooks (config, secret rotation, delivery log, pause/resume), Orders (search/filter, detail with timeline, refund action panel), SSE concurrency view (read-only), SLA page (uptime gauge, incidents), Team (invite, roles, remove), Audit log (view, export), Billing (plan display), Settings (org, security basics) | 14-18 dev-weeks (2 FE + 1 BE + 0.5 DevOps) | - |
| **P1 - Full production readiness** | Realtime KPIs via SSE, Revenue/Payouts (overview, history, payout address management, manual payout request + HMAC confirmation), Providers grid (toggle, refund matrix), WebAuthn enrollment, In-app notifications, Email notifications (all templates), Webhook events filter, Order CSV export (async), GDPR export/delete (manual process; log to gdpr_requests) | 10-14 dev-weeks | Requires P0 foundation |
| **P2 - Scale + polish** | Scoped personal access tokens, Per-provider fee config, Markup earnings breakdown, Dark/light theme, Tablet two-column layout, Admin portal (internal Privata team), Trocador-compat monitoring, Internationalization (Russian UI toggle for pilot partners), GDPR self-service export/delete UI | 8-12 dev-weeks | Non-blocking for revenue |

**Total: P0 + P1 = ~28 dev-weeks (approximately 7 months for a 2-person team; 4 months for a 4-person team).**

---

## 13. Open Questions / Risks

1. **Privata public API ownership of order mirror:** The dashboard mirrors order data by consuming webhooks. If the partner's webhook endpoint (pointed at the dashboard's internal receiver) is paused or unreachable, the order mirror goes stale. Mitigation: dashboard should also poll `GET /me/orders` on a 5-minute schedule as a fallback sync. This creates a dependency on the public API's pagination - confirm rate-limit headroom with the API team.

2. **HMAC payout confirmation UX:** Requiring partners to compute an HMAC manually is a high-friction step that may confuse Finance users who do not own the webhook secret. The email OTP fallback must be clearly presented. Risk: if the Finance role doesn't have access to the webhook secret (correct by design), the email OTP path is the only practical option for them - confirm this is acceptable or provide a dedicated "payout confirmation key" concept.

3. **Payout address verification micro-payment:** The verification scheme (sending 0.01 USDT to confirm address ownership) is a real blockchain transaction. Who pays the gas/network fee? What happens if the Tron/Ethereum network fee spike makes 0.01 USDT verification uneconomic? Consider: signature-based verification (partner signs a challenge with the private key for the address) as an alternative for P1.

4. **Tenant isolation at scale:** Currently all organizations share a single Postgres instance with `org_id` row-level filtering. If a large partner has 10M orders, index scans on the shared `orders` table will be slow even with partitioning, and a misconfigured query could expose cross-org data. Mitigation: Postgres row-level security (RLS) as a defense-in-depth layer; evaluate per-enterprise-tier schema isolation at P2.

5. **Webhook delivery replay idempotency:** The dashboard's "replay" action re-sends an original payload with the current secret. If the partner's system is not idempotent on `event_id`, replaying a `order.completed` event could trigger duplicate payouts or notifications on their end. The API spec documents this requirement but enforcement is outside Privata's control. The UI should display a clear warning: "Replay will re-deliver the original event. Ensure your endpoint handles duplicate event_ids."

6. **Geographic restriction on data:** Some enterprise partners may require that their order data and audit logs never leave a specific region (EU data residency). The current architecture places everything in one AWS region. Providing per-partner data residency is a significant architectural lift - call this out as a commercial blocker for regulated partners in the EU and UK.

7. **SSE proxy amplification:** If the dashboard's SSE page shows live streams, and many dashboard users open the same org's stream list simultaneously, the dashboard backend could open duplicate SSE connections to the public API. The `sse_streams` table is a cache of the public API's state - it must be populated by a single poller per org, not by each dashboard user independently.

---

## 14. Multi-Role Review Notes

### 14.1 Senior PM Review

**Issues found:**

1. **Empty state for new partner on Home Dashboard lacked a clear next action.** The original spec mentioned an "onboarding checklist widget" but did not specify which checklist items. Fix applied: §4.3 now specifies the checklist as incomplete onboarding steps, with a direct link to the Sandbox guide and a "Send your first test order" CTA.

2. **No mention of the test key being issued during onboarding.** Partners need a test key before they can try any integration. Fix applied: §4.2 Onboarding Done screen now explicitly states "test key auto-issued and displayed (once)".

3. **Finance role cannot access orders - a gap for revenue reconciliation.** Finance needs to correlate earnings with specific orders to reconcile revenue reports. Fix applied: §3 now grants Finance the `orders.search` and `orders.getDetail` read permissions (but not export or refund actions).

4. **No documented path for a partner to self-diagnose a paused webhook.** Partners hitting the webhook-paused state need clear actionable UI. Fix applied: §4.5 now specifies the "Paused" badge, the "Resume" button behavior, and the error state when the active API key lacks `webhook:admin` scope.

5. **Onboarding does not tell the partner what happens next (review process, go-live timeline).** Fix applied: §4.2 Done screen now includes "Confirmation; test key auto-issued" wording.

6. **Missing empty state for Providers page when API is down.** Fix applied: §4.8 now includes the fallback skeleton table state.

**All issues resolved: confirmed.**

---

### 14.2 Staff Security Engineer Review (STRIDE Threat Model)

**Issues found and fixes applied:**

**Spoofing:**
- S1: *Session fixation.* An attacker who can inject a session cookie before login could inherit the authenticated session. Fix applied: §5.1 now explicitly states session token is rotated on privilege escalation (2FA completion) and on successful login, invalidating any pre-existing session.

**Tampering:**
- T1: *Audit log tampering.* An attacker with DB write access could modify past entries. Fix applied: §5.9 specifies the DB role restriction (dashboard_app has INSERT, SELECT only on audit_log) and daily backup-based tamper detection, and §6.1 confirms the Postgres role enforcement.
- T2: *Webhook replay attack.* Adversary captures a signed webhook and replays it. Fix applied: §5.8 documents the timestamp check (±300s) and the fact that the dashboard's replay action creates a new delivery with the current secret rather than re-using old signatures.
- T3: *CSRF on state-changing actions.* Fix applied: §5.3 specifies double-submit cookie CSRF protection, enforced for all tRPC mutations.

**Repudiation:**
- R1: *No proof that payout was authorized by the legitimate account holder.* Fix applied: §4.9 documents the HMAC confirmation step (proving control of webhook secret) and the email OTP fallback, both of which create non-repudiable audit log entries.

**Information Disclosure:**
- I1: *IDOR on order IDs.* Fix applied: §5.7 specifies UUID v4 IDs, org-scoped queries, and the `withOrgScope()` helper that enforces org ownership at the tRPC layer.
- I2: *IDOR on webhook delivery IDs.* Same mitigation as I1 - all delivery queries include `WHERE org_id = :current_org_id`.
- I3: *API key secret exposure.* Fix applied: §5.6 specifies HMAC-SHA256 storage (not reversible), prefix stored plaintext for UI. Secret shown only once at creation in a modal that requires explicit acknowledgement.
- I4: *Webhook secret exposure via delivery log.* The `X-Privata-Signature` header in delivery logs contains a derived value, not the secret itself. Fix: §4.5 delivery detail specifies signature is masked after first 8 chars in the UI.
- I5: *Client IP disclosure via SSE stream list.* Fix applied: §4.6 specifies IPs are shown with last two octets masked.
- I6: *User enumeration via login error messages.* Fix applied: §4.1 error states specify "Email or password incorrect" - no differentiation between non-existent email and wrong password.

**Denial of Service:**
- D1: *Webhook replay flood.* An attacker (or bug) triggers thousands of replay requests. Fix applied: §5.4 rate limit table now includes "Webhook replay: 10 per endpoint per hour."
- D2: *Payout request flood.* Fix applied: §5.4 rate limit table includes "Payout requests: 2 per org per 24h."

**Elevation of Privilege:**
- E1: *Plan upgrade without authorization.* Billing upgrade is not self-serve (requires Privata sales confirmation, §4.13); no payment processor involved at the dashboard level.
- E2: *Finance role accessing API keys or triggering refunds.* Fixed in §3 - Finance role explicitly excluded from API key, refund, and webhook actions. Enforced at tRPC middleware layer by checking `ctx.session.role` against an allowlist per procedure.
- E3: *Developer creating `payouts:trigger` scoped key on a test key.* Fix applied: §4.4 blocks `payouts:trigger` scope on test-environment keys with an inline error.

**Additional mitigations not previously specified:**
- SSRF in webhook URL registration: §4.5 now explicitly lists blocked ranges (private RFC-1918, 169.254.169.254, metadata.google.internal, link-local) per the webhooks guide documentation.
- DNS rebinding protection for webhook endpoints: noted in §4.5 ("DNS rebinding protection: resolve and pin IP for 5 min on first delivery" - sourced from the webhooks guide).

**all_findings_resolved: true**

---

### 14.3 Staff Backend Engineer Review

**Issues found and fixes applied:**

**Data model:**
- B1: *`orders` table hot row problem.* High-volume partners writing many orders/minute to a single table with `org_id` scoping could cause page-level lock contention. Fix applied: §6.1 now specifies range partitioning by `created_at` (monthly). The `idx_orders_org_created` index is defined on the partitioned table. Additionally, §6.3 specifies a 90-day hot partition with older data in cold partitions.
- B2: *`webhook_deliveries.next_retry_at` index for retry poller.* A cron job that polls for due retries needs an efficient index. Fix applied: `idx_deliveries_retry` index added on `(next_retry_at) WHERE status = 'pending'`.
- B3: *N+1 risk on order list.* `orders.search` could issue a separate query per order to fetch delivery counts. Fix: the `orders.search` RPC must use a `LEFT JOIN LATERAL` or a `COUNT` subquery to return delivery counts in a single query. Document this in the RPC spec. Applied: added this note to the `orders.search` RPC entry.
- B4: *`sse_streams` table staleness.* Pod restarts leave rows without `closed_at`, inflating the concurrency count. Fix applied: §6.1 now includes `pod_id` column and a note that a pod-shutdown hook must mark all rows for that `pod_id` as closed. A fallback: rows without `closed_at` older than 2 × the SSE keepalive interval (e.g., 90 seconds) are treated as stale.
- B5: *`audit_log` write integrity.* Addressed by DB role restriction (§5.9) - dashboard_app has INSERT, SELECT only; daily backups provide tamper detection baseline.
- B6: *`api_keys.key_hash` lookup on every public API request.* This is not the dashboard's concern - the public API owns its own auth layer. Confirmed: the dashboard's `api_keys` table is for display/management only; the public API has its own key store. No change needed, but callout added to §8 to clarify the separation.
- B7: *Missing `orders.amount_in_usd` derivation.* The original API spec returns amounts in native asset, not USD. The dashboard KPIs need USD values. Fix applied: §6.1 adds `amount_in_usd NUMERIC(18,2)` to the `orders` table, populated at ingest time by calling a price oracle (CoinGecko or internal pricing service). This is a non-trivial dependency - noted in §13 open question 1.
- B8: *Redis hotkey risk for KPI aggregates.* `kpi:{org_id}:{date}` is a single Redis hash key that gets written on every order event. For high-volume partners (>100 orders/minute), this is a write hotspot. Fix: use Redis pipeline + HINCRBY for atomic increments (not a read-modify-write cycle). For extremely high volumes (>1000/min, Enterprise tier), consider a local in-memory counter per dashboard pod that flushes to Redis every 5 seconds.
- B9: *Migration path.* No migration tooling specified. Fix applied: Drizzle ORM provides `drizzle-kit` for migration generation. All schema changes must produce a timestamped migration file in `./drizzle/migrations`. Zero-downtime migrations use `ADD COLUMN ... DEFAULT ...` (backfilled in background) and avoid `ALTER COLUMN NOT NULL` without a default.

**all_findings_resolved: true**

---

**Overall review status: all_findings_resolved: true**

All issues raised by the Senior PM, Staff Security Engineer, and Staff Backend Engineer have been incorporated into the main document sections above. The review notes reference specific section numbers where fixes were applied.

---

*End of Privata Partner Dashboard - Product + Technical Specification v1.0*
