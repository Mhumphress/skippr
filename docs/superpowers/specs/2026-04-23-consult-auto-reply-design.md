# Consult Form Auto-Reply — Design Spec

**Date:** 2026-04-23
**Status:** Approved for implementation
**Owner:** M. Humphress

## Goal

Replace the current Formspree-backed consult form submission with a self-hosted pipeline that does three things automatically on every submission:

1. Appends the lead to the existing Skippr CRM spreadsheet (in a new 📥 Leads tab).
2. Sends the customer an auto-reply from `support@skippr.us` within seconds.
3. Sends an internal notification to `support@skippr.us` (a distribution group that fans out to the team).

## Constraints

- **Free to run.** No new SaaS subscriptions.
- **No new third parties.** Use existing Google Workspace + Cloudflare Pages only.
- **Isolated from existing CRM script.** The existing container-bound `Skippr_CRM.gs` stays untouched; its menu, triggers, and deploys must not change.
- **Keep `support@skippr.us` as the canonical support/sender address** across the codebase.

## Architecture

A **new standalone Google Apps Script project** (separate from the existing CRM script) hosts a single `doPost(e)` Web App endpoint. The form on `skippr.us/consult.html` posts directly to that endpoint. The Apps Script opens the existing CRM spreadsheet by ID, appends the lead, and sends both emails via Gmail.

No Cloudflare Pages Function is needed. No new email service is needed.

```
┌─────────────────────┐
│  consult.html form  │  (skippr.us, served by Cloudflare Pages)
│  fetch POST         │
└──────────┬──────────┘
           │
           ▼
┌───────────────────────────────────────────────────┐
│  Apps Script Web App — ConsultIntake              │
│  URL: script.google.com/macros/s/<ID>/exec        │
│                                                   │
│  doPost(e):                                       │
│    1. Validate fields                             │
│    2. Spam check (honeypot + origin)              │
│    3. Append row to 📥 Leads sheet                │
│    4. Send customer auto-reply via MailApp        │
│    5. Send internal notification via MailApp      │
│    6. Return { ok: true } / { ok: false, error }  │
└──────┬──────────────────┬─────────────┬───────────┘
       │                  │             │
       ▼                  ▼             ▼
   📥 Leads tab      Customer      support@skippr.us
   (same CRM         inbox         (distribution group)
    spreadsheet)
```

### Why standalone instead of container-bound

Google allows only one container-bound Apps Script per spreadsheet. The existing CRM script already owns that slot. A standalone script:

- Lives in Drive independently, opens the target sheet via `SpreadsheetApp.openById(SHEET_ID)`
- Fails in isolation — a bug here cannot break the CRM's menu, triggers, or UI
- Has a narrower permissions scope (this one spreadsheet + Gmail), not the full sheet-UI surface
- Can be redeployed on its own cadence without affecting the CRM

## Components

All code lives in one file: `skippr/CRM/ConsultIntake.gs.txt` (local mirror) → pasted into the "Skippr Consult Intake" Apps Script project in Drive.

| Symbol | Purpose |
|---|---|
| `CONFIG` | Constants: `SHEET_ID`, `SUPPORT_EMAIL`, `SENDER_NAME`, `TIMEZONE`, `ALLOWED_ORIGINS`, `LEADS_SHEET_NAME` |
| `doPost(e)` | Entry point. Parses form params, delegates to `handleConsultSubmission`, returns JSON. |
| `handleConsultSubmission(data)` | Orchestrator. Validates, spam-checks, writes row, sends both emails. |
| `appendLead(ss, data)` | Opens 📥 Leads tab, appends one row. |
| `sendAutoReply(data)` | Composes and sends the customer email. |
| `sendInternalNotification(data)` | Composes and sends the team email. |
| `setup()` | One-time bootstrap. Creates the 📥 Leads sheet with formatting. |
| `buildLeads(ss)` | Builds the 📥 Leads sheet (header, colors, widths, filter). Called by `setup`. |
| `test_happyPath()` | Manually-invoked test: simulates a valid submission. |
| `test_honeypot()` | Manually-invoked test: honeypot-tripped submission is silently dropped. |
| `test_missingField()` | Manually-invoked test: missing `email` returns `{ ok: false }`. |

## The 📥 Leads sheet

Created once by `setup()` in the existing CRM spreadsheet. Formatted to match the existing CRM theme (black header, gold accents, off-white row alternation).

| Col | Field | Example |
|---|---|---|
| A | Timestamp (America/Indianapolis) | `2026-04-23 14:32 ET` |
| B | Name | `Sarah Thompson` |
| C | Email | `sarah@gmail.com` |
| D | Phone | `(317) 555-0199` |
| E | Service Tier | `Tier 2 - White Glove Concierge ($2,500)` |
| F | Desired Vehicle | `2026 Lexus GX 550` |
| G | Message | `Trading in my 2021 Highlander…` |
| H | Status | `New` (default) |
| I | Assigned Rep | (blank — manually filled) |
| J | Origin / Notes | `referer: skippr.us; UA: Chrome/…` |

- Row 1 has the existing CRM's auto-filter enabled.
- Column widths tuned: A:160, B:160, C:200, D:130, E:240, F:200, G:360, H:100, I:140, J:260.
- The existing `setupCRM()` in `Skippr_CRM.gs` only deletes a sheet named `Sheet1`, so 📥 Leads survives future CRM re-runs.

## Email content

### Sender display name

All outbound mail uses:

```
From: "Skippr Concierge" <support@skippr.us>
```

Set explicitly via `MailApp.sendEmail({ name: "Skippr Concierge", ... })`. Without this, Gmail would display the owning account's default name.

### Customer auto-reply (same copy for all tiers)

```
Subject: Your Skippr request has been received
From:    "Skippr Concierge" <support@skippr.us>
To:      {{customer.email}}
Reply-To: support@skippr.us

{{firstName}},

Thank you for reaching out. We've received your consultation request and
a member of our team will be in touch within 48 hours.

If your inquiry is time-sensitive, simply reply to this message — it
lands directly in our inbox.

— The Skippr Team
support@skippr.us · skippr.us
```

`{{firstName}}` is derived by splitting the submitted full name on whitespace and taking the first token. `"Sarah Thompson"` → `"Sarah"`; `"Sarah"` → `"Sarah"`.

### Internal notification

```
Subject: New consult lead — {{name}} ({{serviceTier}})
From:    "Skippr Concierge" <support@skippr.us>
To:      support@skippr.us
Reply-To: {{customer.email}}

A new consultation request just came in.

Name:    {{name}}
Email:   {{email}}
Phone:   {{phone}}
Service: {{serviceTier}}
Vehicle: {{vehicle}}

Message:
{{message}}

(Submitted {{timestamp}} from {{origin}})
```

Hitting Reply from any inbox threads directly to the customer.

## Website changes

Exactly one file changes: `skippr/consult.html`.

**Line 233 — form action URL:**
```diff
- <form id="consultForm" action="https://formspree.io/f/mpqoqabj" method="POST">
+ <form id="consultForm" action="https://script.google.com/macros/s/<DEPLOYMENT_ID>/exec" method="POST">
```

**Add a honeypot input** just before the `</form>` close (position off-screen, hide from ATs):

```html
<input type="text" name="company_website" tabindex="-1" autocomplete="off"
       aria-hidden="true"
       style="position:absolute;left:-9999px;width:1px;height:1px;opacity:0;">
```

**Extend the fetch body** (around line 309) to include the origin for the Apps Script's origin check:

```diff
  const formData = new FormData(this);
+ formData.append('_origin', window.location.origin);
  const res = await fetch(this.action, {
```

No other HTML/CSS/JS changes. The existing `res.ok` check + redirect to `thank-you.html` continues to work.

## Spam protection

Two lightweight defenses inside `handleConsultSubmission`:

1. **Honeypot** — if `e.parameter.company_website` is non-empty, log and return `{ ok: true }` without writing to the sheet or sending email. Bots see a present field and fill it; humans never see it.
2. **Origin check** — `e.parameter._origin` must be in `CONFIG.ALLOWED_ORIGINS` (`https://skippr.us`, `https://skippr.pages.dev`). Otherwise return `{ ok: false, error: "Bad origin" }`.

No Cloudflare Turnstile, no shared-secret token, no IP rate limiting in v1. These are all easy to add later if real spam appears.

## Deliverability

Google Workspace already handles outbound mail from `@skippr.us` with DKIM signing via the `google._domainkey` DNS record. DMARC passes on DKIM alignment even though SPF does not include Google. No DNS changes are required for this project. (If auto-replies ever land in spam, the first fix is adding `include:_spf.google.com` to the existing SPF record — but this is out of scope for v1.)

## Deployment sequence

1. Write source locally: `skippr/CRM/ConsultIntake.gs.txt` + consult.html edits. Commit.
2. (User) Create new standalone Apps Script project in Drive: "Skippr Consult Intake". Paste source. Fill in `CONFIG.SHEET_ID` from the CRM spreadsheet URL.
3. (User) Run `setup()` once. Authorize Sheets + Gmail. Confirm 📥 Leads tab appears.
4. (User) Deploy → Test deployments → Web app → copy `/dev` URL.
5. Validation: curl test payloads against `/dev` URL — happy path, honeypot, missing field. Confirm rows appear and test emails arrive at a user-provided test email address (not a random one).
6. (User) Deploy → New deployment → Web app → Execute as: me, Who has access: Anyone → copy `/exec` URL.
7. Update `consult.html` line 233 with the real `/exec` URL. Commit.
8. (User) `git push`. Cloudflare Pages auto-deploys in ~60s.
9. Smoke test: one real submission from skippr.us. Verify Leads row, auto-reply delivery, internal notification.
10. (Optional, later) Delete the `mpqoqabj` form from Formspree's dashboard.

## Rollback plan

- **Fastest rollback** (~60s): `git revert <cutover-commit> && git push`. Cloudflare redeploys the Formspree version.
- **Apps Script misbehaving**: in Apps Script → Manage deployments → point the `/exec` URL at a previous version. ~15s.
- **Nuclear**: the `backup-2026-04-23-pre-auto-reply` git tag and the local zip backups under `Website/backups/skippr-backup-2026-04-23/`.

## Testing

### Pre-cutover (against `/dev` URL)

- `test_happyPath()` — inside the Apps Script editor. Simulates a full valid submission; asserts a new row appears in 📥 Leads and `MailApp` quota counter decrements by 2.
- `test_honeypot()` — simulates a submission with `company_website=spam`; asserts no row written, no emails sent, response `{ ok: true }`.
- `test_missingField()` — simulates a submission with no `email`; asserts response `{ ok: false, error: "Missing email" }`, no row, no emails.
- External curl from the workstation against `/dev` URL for a real HTTP round-trip.

### Post-cutover (against `/exec` URL on live site)

- One real form submission from skippr.us using a real customer-grade email address (test alias or user's personal Gmail). Verify all three effects: Leads row, auto-reply in customer inbox, internal notification to support@skippr.us.

Ongoing: check 📥 Leads weekly for any spam that slipped past the honeypot. If >1-2 spam submissions per week, add Cloudflare Turnstile.

## Out of scope for this project

- Cloudflare Turnstile / captcha.
- Shared-secret token for the Apps Script endpoint.
- Shared-secret rotation.
- Automated "move lead to Pipeline" menu action in the CRM.
- Error-alert emails when `doPost` throws (Apps Script Executions log is sufficient).
- Any edits to the existing `Skippr_CRM.gs` file or its behavior.
- SPF record update (non-blocking; skip unless deliverability complaints arise).

## Open inputs required from user at deploy time

- `CONFIG.SHEET_ID` — the CRM spreadsheet ID, pasted into the Apps Script.
- Test customer email address for step 5 validation curls.
- The `/dev` URL (after step 4) and the `/exec` URL (after step 6).
