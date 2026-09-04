# Pacesetter CMS — Initial Architecture Plan

**Status:** Draft / pre-implementation — Phase 1 (Auth & Accounts) finalized
**Last updated:** 2026-09-04

## 1. Overview

A web-based Content Management System for a college student journalism
publication ("Pacesetter"), allowing writers to author articles through an
editorial review process before public publication. Layout and content
experience are inspired by Inquirer.net (section-based navigation, featured
articles) and Medium (clean article reading experience).

## 2. Tech Stack

| Layer | Choice |
|---|---|
| Frontend | React (Vite) |
| Backend | Node.js + Express |
| Database | MongoDB (Atlas) |
| Rich text editing | WYSIWYG editor (writers compose visually, Medium-style) |
| Media storage | Cloud storage (e.g. Cloudinary) — not local disk |
| Transactional email | Nodemailer via Gmail SMTP (App Password auth) |
| API documentation | Swagger (OpenAPI) |
| Hosting | Free/low-cost tier (e.g. Render/Railway for backend, Atlas free tier for DB) |

## 3. User Roles & Permissions

**Reader is not an account — it's the public/anonymous state of the site.**
Anyone can read published articles without logging in; there is no
registration, session, or User record for readers. The three roles below
(Writer, Admin, Super Admin) are the only ones with actual accounts, and
all three are provisioned exclusively by Super Admin (see Section 13).

This has a direct architectural consequence: authentication only needs to
exist for staff. Public routes (homepage, category pages, article pages,
search) are unauthenticated by default and require no session at all — the
only place `optionalAuth`-style logic matters is on read routes that might
be hit by logged-in staff previewing content, not by the general public.

A clear split remains between **content authority** and **people (account)
authority** among the three real roles:

| Capability | Writer | Admin | Super Admin |
|---|---|---|---|
| Read published articles | ✅ | ✅ | ✅ |
| Create / edit own drafts | ✅ | ✅ | ✅ |
| Submit draft for review | ✅ | ✅ | ✅ |
| Approve / reject submissions | — | ✅ | ✅ |
| Edit other writers' articles | — | ✅ | ✅ |
| Archive published articles | — | ✅ | ✅ |
| Manage categories | — | ✅ | ✅ |
| Create / disable user accounts | — | — | ✅ |
| Promote / demote roles | — | — | ✅ |
| Hard delete (articles / accounts) | — | — | ✅ |

**Rationale:** Admins run day-to-day editorial operations (review, edit,
organize content) but cannot control who has access to the system. Only
Super Admin (e.g. EIC, faculty adviser, org president) can change account
access or roles. This matters for an org with annual turnover — outgoing
Admins should never have had the power to mint new Admins.

## 4. Editorial Workflow

"Writers may publish articles" was reframed as a reviewed workflow rather
than direct self-publishing, since unreviewed publication under the
organization's name carries real editorial/reputational risk (factual
errors, no correction gate, etc.).

```
DRAFT → SUBMITTED → PUBLISHED
           ↓
        REJECTED → (writer revises, back to DRAFT)

PUBLISHED → ARCHIVED (admin can unpublish without deleting)
```

- Writers own their drafts, submit when ready, and can see the outcome of
  review on their own articles.
- Admins/Super Admins review the submission queue and approve or reject.
- **Rejection reason is optional** — a reviewer may leave a reason or reject
  without one. (Note: this is a deliberate flexibility tradeoff; an
  editorial norm of always giving a reason is a team-culture matter, not a
  system-enforced rule.)
- Nothing reaches the public site without passing through review.

**Provisioning model (resolved):** there is no self-registration anywhere
in the system. Readers require no account at all, and every real account
(Writer, Admin, or Super Admin) is created exclusively by a Super Admin.
This keeps org membership tightly controlled — no public sign-up surface
for staff roles exists to be abused or left open by accident.

## 5. Phase 1 Detail: Auth & Account Management

This is the foundation phase — nothing else in the app is meaningful
without it, since every later feature is permission-gated.

### 5.1 Account States

An account has a lifecycle status distinct from its role, since "created
but not yet activated" and "created and active" behave differently at
login:

| Status | Meaning | Can log in? |
|---|---|---|
| `pending` | Super Admin created the record; invite sent; no password set yet | No |
| `active` | Password has been set via invite or reset | Yes |
| `disabled` | Super Admin has revoked access | No — existing sessions are also invalidated immediately, not just blocked going forward |

### 5.2 Provisioning Flow

1. Super Admin creates an account: name, email, role (Writer or Admin —
   Super Admin accounts are provisioned outside the app; see 5.5).
2. Account is created with status `pending` and no password.
3. An invite email is sent (via Nodemailer/Gmail SMTP) containing a
   single-use, expiring token link.
4. The invited user opens the link, sets their own password, and the
   account becomes `active`.

No self-registration exists anywhere in the system. Readers require no
account at all (see Section 3).

### 5.3 Shared Token Mechanism (Invite + Password Reset)

Account activation and "forgot password" are treated as **one underlying
mechanism** rather than two separate systems, since both are fundamentally
"prove you control this email address, then set a password":

- A random, unguessable token is generated and emailed to the user as a
  link.
- Only a **hash** of the token is stored in the database — never the raw
  token — mirroring how passwords themselves are stored, so a database
  leak doesn't expose usable tokens.
- Tokens are **single-use**: successfully setting a password immediately
  invalidates the token.
- Requesting a new token (resending an invite, requesting another reset)
  invalidates any previously issued, still-outstanding token for that
  user — only one valid token exists per user at a time.
- **Expiry differs by purpose:** invite links live longer (e.g. 3–7 days,
  since a new hire may not check email immediately), while reset links are
  short-lived (e.g. 1 hour, since a live reset request is time-sensitive
  and a stale one is more likely to be someone else's mistake or an
  attack).

### 5.4 Session Handling

- JWT access + refresh tokens in httpOnly cookies (see Section 8 for the
  full rationale).
- Standard login/logout.
- **Logout-all / session revocation:** disabling an account or forcing a
  password change invalidates all of that user's outstanding sessions
  immediately, not just future logins.

### 5.5 Bootstrap: First Super Admin

Since every account is provisioned by a Super Admin, the very first
Super Admin account cannot be created through the app itself. This is
handled as a **one-time, ops-only manual step** — directly inserting a
user record into the database with a bcrypt-hashed password, using the
same hashing logic the application uses — and is explicitly **not**
application code or a seed script. This should be documented as a
deployment runbook step, not built as a feature.

### 5.6 Super Admin Account Management Capabilities

- Create account (name, email, role) → triggers invite flow
- View list of accounts (with status: pending / active / disabled)
- Disable / re-enable an account
- Change a user's role
- Resend invite (for accounts stuck in `pending`)

### 5.7 Email Dependency & Constraints

Nodemailer requires an actual SMTP server to send through — it is a
client library, not a hosted email service. This phase uses **Gmail SMTP**
with an App Password (requires 2FA enabled on the sending account).

**Known constraints, accepted for this phase:**
- Free Gmail accounts cap around 500 sends/day; Google Workspace accounts
  around 2,000/day. Fine for invite/reset volume at organization scale.
- Gmail SMTP is more prone to rate-limiting/flagging under bursty or
  automated-looking send patterns than a dedicated transactional provider.
- If later phases (Notification email channel in Phase 5, Email Template
  Generator in Phase 7) need higher volume, the SMTP transport can be
  swapped (e.g. to Resend or SendGrid's SMTP endpoints) without changing
  application code, since all mail is routed through one Nodemailer
  transport configuration.

Phase 1 only sends two plain system emails — invite and password reset.
Polished, branded templates are explicitly out of scope here and belong to
the Email Template Generator (Section 10).

### 5.8 Notification Event Hooks (Phase 1.5 tie-in)

Per the Notification Module architecture (Section 9), Phase 1 fires events
into the shared Notification Service at these points, even though no
notification UI exists yet:
- Account role changed
- Account disabled / re-enabled

### 5.9 Phase 1 Feature Checklist

1. User model: name, email, password hash, role, status, refresh-token
   version
2. Super Admin: create account → triggers invite email
3. Shared token mechanism: generate, email, validate, consume (invite +
   reset)
4. Login (only `active` accounts succeed; generic error message — never
   reveal whether an email exists or which check failed)
5. Forgot password flow (request → email → consume token → set new
   password)
6. Super Admin: view accounts, disable/re-enable, change role, resend
   invite
7. Session handling: httpOnly cookie JWT (access + refresh), logout,
   logout-all/session revocation
8. Notification event hooks fired on role change and disable (no UI yet)

## 6. Data Model (Conceptual)

- **User** — name, email, password hash, role, active/disabled flag,
  refresh-token version (for session revocation).
- **Article** — title, slug, structured rich-text content (not raw HTML —
  see security note below), excerpt, cover image, author, category, tags,
  status, rejection reason (optional), reviewer, publish date, view count.
- **Category** — name, slug, description. Maps to section-style navigation
  (News, Opinion, Features, Sports, etc.).

**Security note:** Article content is stored as structured editor data
rather than raw HTML, so rendering an article never means trusting
arbitrary markup from a writer — this closes a stored-XSS risk that would
otherwise let a malicious or compromised writer account inject scripts
that execute in every reader's browser.

## 7. Public Reading Experience

- Homepage with featured/latest articles and a section rail
- Category landing pages (News, Opinion, Features, Sports, etc.)
- Tag-based filtering
- Search across title/excerpt
- Article detail page: byline, date, category, related articles

## 8. Authentication Strategy

JWT access + refresh tokens stored in **httpOnly cookies**, not
`localStorage`. A rich-text CMS carries elevated XSS exposure (writers
pasting content/embeds), and tokens in `localStorage` are readable by any
injected script; httpOnly cookies close that attack surface. This requires
standard CSRF mitigations in exchange, which is an acceptable tradeoff for
a publicly-facing publication.

## 9. Notification Module (Core Feature)

Users can independently enable/disable **in-app** and **email**
notifications, per event type (e.g. article approved, article rejected,
account role changed).

**Architectural principle:** every module that produces a notification-worthy
event (account changes in Phase 1, review decisions in Phase 2, etc.) calls
into a single shared **Notification Service** — no module implements its own
notification logic. This lets the *hook points* (the actual event triggers)
get built alongside Phase 1/2 code, while the *notification center UI* and
*email sending* are built later as their own phase, without retrofitting
earlier phases.

The email channel of this module reuses the same rendering engine as the
Email Template Generator (Section 10) — one templating system, two callers:
a human clicking "generate" for outreach, or the system auto-filling a
template on an event.

## 10. Email Template Generator (Future Module)

A general-purpose internal tool (not a newsletter/subscriber system) for
org members to generate ready-to-send HTML emails:

- Use cases: outreach, official announcements, etc.
- A member selects a template and fills in variable content (subject,
  body text, etc.), then generates a rendered HTML preview.
- Templates may start hardcoded (a handful of HTML/CSS files with
  placeholder tokens) and later move into the CMS as DB-managed, editable
  templates once real usage patterns are known.
- Scope excludes actual sending infrastructure — output is
  preview/copy/download; sending happens via the member's own email client.

## 11. Future Feature: Social Auto-Posting

On an article's `PUBLISHED` transition, a hook/event triggers a job (queue
worker or direct API call) that posts to Facebook, Instagram, etc. via
their respective APIs. Deliberately designed as a listener on the existing
publish event rather than logic embedded in the article controller, so it
can be added later without touching core article logic.

## 12. Phase Plan

| Phase | Scope |
|---|---|
| **1** | Auth & Accounts — see Section 5 for full detail. Account states (pending/active/disabled), Super-Admin-only provisioning via invite email, shared invite/reset token mechanism, session handling, Nodemailer/Gmail SMTP for system email |
| **1.5** | Notification event hooks defined inside Phase 1/2 code (no UI yet) |
| **2** | Article authoring & editorial workflow (draft → submit → review → publish/archive) |
| **3** | Public reading experience (homepage, categories, search/filter, article pages) |
| **4** | Media handling (Cloudinary integration for cover/inline images) |
| **5** | Notification module (in-app center, preferences UI, email channel) |
| **6** | Polish & hardening (rate limiting, accessibility, performance, Swagger completeness, deployment) |
| **7** | Email Template Generator (outreach/announcements) |
| **Future** | Social auto-posting |

## 13. Open Questions

1. Editorial norms around rejection reasons (team culture, not a system
   constraint, but worth defining for consistency).