# Consult Form Auto-Reply Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the Formspree-backed `consult.html` form submission with a self-hosted pipeline that appends each lead to a new 📥 Leads tab in the existing Skippr CRM spreadsheet, sends the customer an auto-reply, and notifies the `support@skippr.us` distribution group.

**Architecture:** A single standalone Google Apps Script project ("Skippr Consult Intake") exposes a `doPost(e)` Web App endpoint. The form posts to it directly. The script opens the existing CRM spreadsheet via `SpreadsheetApp.openById(SHEET_ID)` and sends mail via `MailApp.sendEmail()`. No Cloudflare Pages Function is added. The existing container-bound `Skippr_CRM.gs` is not touched.

**Tech Stack:** Google Apps Script (V8 runtime), `SpreadsheetApp` + `MailApp` + `ContentService`, plain HTML + `fetch()` on the frontend. Git + Cloudflare Pages for deploy.

**Spec:** See `docs/superpowers/specs/2026-04-23-consult-auto-reply-design.md`.

---

## Preconditions

Before starting Task 1, verify:
- The local repo is at `skippr/`, branch `main`, clean (the untracked `services-preview.html` is unrelated and can be ignored).
- The backup tag `backup-2026-04-23-pre-auto-reply` exists locally (`git tag --list` should show it).
- The backup zips exist at `Website/backups/skippr-backup-2026-04-23/`.
- `support@skippr.us` is a real Google Workspace distribution group that fans out to the team (user confirmed).

---

## Testing note for this plan

Google Apps Script has no local test runner. Tests in this plan are **manually-invoked functions inside the Apps Script editor**. The TDD pattern here is: write the test function alongside the implementation in the same commit, then run all `test_*` functions once after deployment (Task 8). Failures are caught at that point, fixed, and redeployed.

Code tasks (1-6) do not have "run test" steps because you can't run Apps Script code locally. The "verify passes" action for all of those tasks happens collectively in Task 8, Step 4.

---

## File Structure

| Path | Action | Responsibility |
|---|---|---|
| `skippr/CRM/ConsultIntake.gs.txt` | **Create** | Full source for the new standalone Apps Script. Local mirror — user copies into script.google.com. |
| `skippr/consult.html` | **Modify** | Swap form action URL, add honeypot input, append `_origin` to fetch body. |
| `skippr/docs/superpowers/specs/2026-04-23-consult-auto-reply-design.md` | **(read only)** | Source of truth for design decisions. Already exists. |

Nothing else changes. The existing `Skippr_CRM.gs` / `skippr/CRM/CRM.gs.txt` remain untouched per the isolation requirement.

---

## Task 1: Scaffold ConsultIntake.gs.txt with CONFIG, setup(), buildLeads()

**Files:**
- Create: `skippr/CRM/ConsultIntake.gs.txt`

- [ ] **Step 1: Create the file with CONFIG, COLORS, setup(), buildLeads(), formatTimestamp().**

Create `skippr/CRM/ConsultIntake.gs.txt` with this exact content:

```javascript
// ============================================================
//  SKIPPR CONSULT INTAKE — Google Apps Script (Standalone)
//
//  This is a STANDALONE Apps Script project, separate from the
//  container-bound Skippr CRM script. It targets the same CRM
//  spreadsheet via openById, and exposes a doPost(e) Web App
//  endpoint that the skippr.us consult form posts to.
//
//  Deploy: Apps Script editor → Deploy → New deployment →
//          Web app → Execute as: me, Who has access: Anyone.
// ============================================================

// ── CONFIG — edit these before running setup() ──────────────
const CONFIG = {
  SHEET_ID: "REPLACE_WITH_CRM_SPREADSHEET_ID",
  SUPPORT_EMAIL: "support@skippr.us",
  SENDER_NAME: "Skippr Concierge",
  TIMEZONE: "America/Indianapolis",
  LEADS_SHEET_NAME: "📥 Leads",
  ALLOWED_ORIGINS: ["https://skippr.us", "https://skippr.pages.dev"],
  TEST_CUSTOMER_EMAIL: "REPLACE_WITH_YOUR_PERSONAL_EMAIL",
};

// ── COLORS (match existing CRM theme) ───────────────────────
const COLORS = {
  HEADER_BG: "#111111",
  HEADER_FG: "#C9A84C",
  GOLD:      "#C9A84C",
};

// ── SETUP & SHEET BUILDING ──────────────────────────────────
function setup() {
  const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);
  buildLeads(ss);
  Logger.log("✅ Setup complete. " + CONFIG.LEADS_SHEET_NAME + " tab is ready.");
}

function buildLeads(ss) {
  let sh = ss.getSheetByName(CONFIG.LEADS_SHEET_NAME);
  if (sh) {
    Logger.log(CONFIG.LEADS_SHEET_NAME + " already exists; refreshing formatting only.");
  } else {
    sh = ss.insertSheet(CONFIG.LEADS_SHEET_NAME);
  }
  sh.setTabColor(COLORS.GOLD);

  const headers = [
    "Timestamp", "Name", "Email", "Phone", "Service Tier",
    "Desired Vehicle", "Message", "Status", "Assigned Rep", "Origin / Notes"
  ];
  const widths = [160, 160, 200, 130, 240, 200, 360, 100, 140, 260];
  widths.forEach((w, i) => sh.setColumnWidth(i + 1, w));

  const headerRange = sh.getRange(1, 1, 1, headers.length);
  headerRange.setValues([headers]);
  headerRange.setBackground(COLORS.HEADER_BG);
  headerRange.setFontColor(COLORS.HEADER_FG);
  headerRange.setFontWeight("bold");
  headerRange.setHorizontalAlignment("left");
  sh.setFrozenRows(1);

  if (sh.getFilter()) sh.getFilter().remove();
  sh.getRange(1, 1, sh.getMaxRows(), headers.length).createFilter();
}

// ── HELPERS ─────────────────────────────────────────────────
function formatTimestamp(date) {
  return Utilities.formatDate(date, CONFIG.TIMEZONE, "yyyy-MM-dd HH:mm 'ET'");
}
```

- [ ] **Step 2: Commit.**

```bash
cd "C:/Users/Mchum/OneDrive/Desktop/Claude Projects/Website/skippr"
git add CRM/ConsultIntake.gs.txt
git commit -m "Scaffold ConsultIntake Apps Script (CONFIG, setup, buildLeads)"
```

---

## Task 2: Add firstNameOf() helper with test

**Files:**
- Modify: `skippr/CRM/ConsultIntake.gs.txt` (append)

- [ ] **Step 1: Append `firstNameOf()` after `formatTimestamp()` in the HELPERS section.**

Add this function immediately after `formatTimestamp`:

```javascript
function firstNameOf(fullName) {
  return String(fullName || "").trim().split(/\s+/)[0] || "Hi";
}
```

- [ ] **Step 2: Append `test_firstNameOf()` at the bottom of the file under a new TESTS header.**

Add to the bottom of the file:

```javascript
// ── TESTS (invoked manually from the Apps Script editor) ────

function test_firstNameOf() {
  const cases = [
    ["Sarah Thompson", "Sarah"],
    ["Sarah", "Sarah"],
    ["  Sarah  Thompson  ", "Sarah"],
    ["Dr. Sarah Thompson", "Dr."],
    ["", "Hi"],
    [null, "Hi"],
  ];
  cases.forEach(pair => {
    const input = pair[0];
    const expected = pair[1];
    const got = firstNameOf(input);
    if (got !== expected) {
      throw new Error("firstNameOf(" + JSON.stringify(input) + ") => " + got + ", want " + expected);
    }
  });
  Logger.log("✅ test_firstNameOf passed (" + cases.length + " cases)");
}
```

- [ ] **Step 3: Commit.**

```bash
git add CRM/ConsultIntake.gs.txt
git commit -m "Add firstNameOf helper + test"
```

---

## Task 3: Add appendLead() with test

**Files:**
- Modify: `skippr/CRM/ConsultIntake.gs.txt` (append)

- [ ] **Step 1: Add `appendLead()` after `buildLeads()` in the SETUP & SHEET BUILDING section.**

Append this function right after `buildLeads(ss)` closes (before the HELPERS section):

```javascript
// ── APPEND LEAD ─────────────────────────────────────────────
function appendLead(ss, data) {
  const sh = ss.getSheetByName(CONFIG.LEADS_SHEET_NAME);
  if (!sh) throw new Error("Leads sheet not found. Run setup() first.");
  const tsString = formatTimestamp(data.timestamp);
  const originCell = "origin: " + data.origin +
    (data.userAgent ? "; UA: " + data.userAgent : "");
  sh.appendRow([
    tsString,
    data.name,
    data.email,
    data.phone,
    data.service,
    data.vehicle,
    data.message,
    "New",
    "",
    originCell
  ]);
}
```

- [ ] **Step 2: Add `test_appendLead()` to the TESTS section, after `test_firstNameOf`.**

```javascript
function test_appendLead() {
  const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);
  const sh = ss.getSheetByName(CONFIG.LEADS_SHEET_NAME);
  if (!sh) throw new Error("Leads sheet missing; run setup() first.");
  const beforeRows = sh.getLastRow();

  const testData = {
    timestamp: new Date(),
    name: "_TEST_ Sarah Thompson",
    email: "test@example.com",
    phone: "(555) 000-0000",
    service: "Tier 1 - Expert Advocate",
    vehicle: "Test Vehicle",
    message: "Test message — safe to delete.",
    origin: "https://skippr.us",
    userAgent: "test",
  };
  appendLead(ss, testData);

  const afterRows = sh.getLastRow();
  if (afterRows !== beforeRows + 1) {
    throw new Error("Expected rows+1, before=" + beforeRows + " after=" + afterRows);
  }
  // Clean up: delete the test row.
  sh.deleteRow(afterRows);
  Logger.log("✅ test_appendLead passed (row added then cleaned up).");
}
```

- [ ] **Step 3: Commit.**

```bash
git add CRM/ConsultIntake.gs.txt
git commit -m "Add appendLead + test"
```

---

## Task 4: Add sendAutoReply() with test

**Files:**
- Modify: `skippr/CRM/ConsultIntake.gs.txt` (append)

- [ ] **Step 1: Add `sendAutoReply()` in a new EMAIL section after `appendLead()`.**

```javascript
// ── SEND AUTO-REPLY ─────────────────────────────────────────
function sendAutoReply(data) {
  const body =
    data.firstName + ",\n\n" +
    "Thank you for reaching out. We've received your consultation request and\n" +
    "a member of our team will be in touch within 48 hours.\n\n" +
    "If your inquiry is time-sensitive, simply reply to this message — it\n" +
    "lands directly in our inbox.\n\n" +
    "— The Skippr Team\n" +
    "support@skippr.us · skippr.us\n";

  MailApp.sendEmail({
    to: data.email,
    subject: "Your Skippr request has been received",
    body: body,
    name: CONFIG.SENDER_NAME,
    replyTo: CONFIG.SUPPORT_EMAIL,
  });
}
```

- [ ] **Step 2: Add `test_sendAutoReply()` in the TESTS section.**

```javascript
function test_sendAutoReply() {
  if (!CONFIG.TEST_CUSTOMER_EMAIL || CONFIG.TEST_CUSTOMER_EMAIL.indexOf("REPLACE") !== -1) {
    throw new Error("Set CONFIG.TEST_CUSTOMER_EMAIL first (top of file).");
  }
  sendAutoReply({
    firstName: "Sarah",
    email: CONFIG.TEST_CUSTOMER_EMAIL,
  });
  Logger.log("✅ test_sendAutoReply: email queued to " + CONFIG.TEST_CUSTOMER_EMAIL + ". Check inbox.");
}
```

- [ ] **Step 3: Commit.**

```bash
git add CRM/ConsultIntake.gs.txt
git commit -m "Add sendAutoReply + test"
```

---

## Task 5: Add sendInternalNotification() with test

**Files:**
- Modify: `skippr/CRM/ConsultIntake.gs.txt` (append)

- [ ] **Step 1: Add `sendInternalNotification()` after `sendAutoReply()`.**

```javascript
// ── SEND INTERNAL NOTIFICATION ──────────────────────────────
function sendInternalNotification(data) {
  const subject = "New consult lead — " + data.name + " (" + data.service + ")";
  const body =
    "A new consultation request just came in.\n\n" +
    "Name:    " + data.name + "\n" +
    "Email:   " + data.email + "\n" +
    "Phone:   " + data.phone + "\n" +
    "Service: " + data.service + "\n" +
    "Vehicle: " + data.vehicle + "\n\n" +
    "Message:\n" + data.message + "\n\n" +
    "(Submitted " + formatTimestamp(data.timestamp) + " from " + data.origin + ")\n";

  MailApp.sendEmail({
    to: CONFIG.SUPPORT_EMAIL,
    subject: subject,
    body: body,
    name: CONFIG.SENDER_NAME,
    replyTo: data.email,
  });
}
```

- [ ] **Step 2: Add `test_sendInternalNotification()` in the TESTS section.**

```javascript
function test_sendInternalNotification() {
  sendInternalNotification({
    name: "_TEST_ Sarah Thompson",
    email: "test@example.com",
    phone: "(555) 000-0000",
    service: "_TEST_ Tier 1",
    vehicle: "Test Vehicle",
    message: "Test internal notification — safe to delete.",
    timestamp: new Date(),
    origin: "https://skippr.us",
  });
  Logger.log("✅ test_sendInternalNotification: email queued to " + CONFIG.SUPPORT_EMAIL + ". Check inbox (subject starts with 'New consult lead — _TEST_').");
}
```

- [ ] **Step 3: Commit.**

```bash
git add CRM/ConsultIntake.gs.txt
git commit -m "Add sendInternalNotification + test"
```

---

## Task 6: Add doPost() and handleConsultSubmission() with integration tests

**Files:**
- Modify: `skippr/CRM/ConsultIntake.gs.txt` (append)

- [ ] **Step 1: Add the ENTRY POINT section at the top of the file, immediately after the COLORS block (and before `function setup()`).**

Insert this block:

```javascript
// ── ENTRY POINT ─────────────────────────────────────────────
function doPost(e) {
  try {
    const params = (e && e.parameter) || {};
    const result = handleConsultSubmission(params);
    return ContentService
      .createTextOutput(JSON.stringify(result))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    Logger.log("doPost error: " + err + "\n" + (err.stack || ""));
    return ContentService
      .createTextOutput(JSON.stringify({ ok: false, error: String(err) }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ── ORCHESTRATOR ────────────────────────────────────────────
function handleConsultSubmission(params) {
  // Honeypot: if bots filled the hidden field, silently discard.
  if (params.company_website && String(params.company_website).trim() !== "") {
    Logger.log("Honeypot triggered; submission silently dropped.");
    return { ok: true };
  }

  // Origin check.
  const origin = String(params._origin || "").trim();
  if (CONFIG.ALLOWED_ORIGINS.indexOf(origin) === -1) {
    return { ok: false, error: "Bad origin: " + origin };
  }

  // Validate required fields.
  const required = ["name", "email", "phone", "service", "vehicle", "message"];
  for (let i = 0; i < required.length; i++) {
    const field = required[i];
    if (!params[field] || String(params[field]).trim() === "") {
      return { ok: false, error: "Missing field: " + field };
    }
  }
  if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(String(params.email).trim())) {
    return { ok: false, error: "Invalid email" };
  }

  const data = {
    name: String(params.name).trim(),
    email: String(params.email).trim(),
    phone: String(params.phone).trim(),
    service: String(params.service).trim(),
    vehicle: String(params.vehicle).trim(),
    message: String(params.message).trim(),
    origin: origin,
    userAgent: String(params._ua || ""),
    timestamp: new Date(),
  };
  data.firstName = firstNameOf(data.name);

  const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);
  appendLead(ss, data);
  sendAutoReply(data);
  sendInternalNotification(data);

  return { ok: true };
}
```

- [ ] **Step 2: Add three integration tests in the TESTS section.**

Append:

```javascript
function test_doPost_happyPath() {
  if (!CONFIG.TEST_CUSTOMER_EMAIL || CONFIG.TEST_CUSTOMER_EMAIL.indexOf("REPLACE") !== -1) {
    throw new Error("Set CONFIG.TEST_CUSTOMER_EMAIL first (top of file).");
  }
  const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);
  const sh = ss.getSheetByName(CONFIG.LEADS_SHEET_NAME);
  const beforeRows = sh.getLastRow();

  const fakeEvent = {
    parameter: {
      name: "_TEST_ Sarah Thompson",
      email: CONFIG.TEST_CUSTOMER_EMAIL,
      phone: "(555) 000-0000",
      service: "Tier 1 - Expert Advocate",
      vehicle: "Test Vehicle",
      message: "Happy-path integration test.",
      _origin: "https://skippr.us",
      company_website: "",
    }
  };
  const response = doPost(fakeEvent);
  const result = JSON.parse(response.getContent());
  if (!result.ok) throw new Error("Expected ok:true, got " + JSON.stringify(result));

  const afterRows = sh.getLastRow();
  if (afterRows !== beforeRows + 1) {
    throw new Error("Expected rows+1, before=" + beforeRows + " after=" + afterRows);
  }
  sh.deleteRow(afterRows);
  Logger.log("✅ test_doPost_happyPath passed. Two emails sent — check " + CONFIG.TEST_CUSTOMER_EMAIL + " and " + CONFIG.SUPPORT_EMAIL);
}

function test_doPost_honeypot() {
  const ss = SpreadsheetApp.openById(CONFIG.SHEET_ID);
  const sh = ss.getSheetByName(CONFIG.LEADS_SHEET_NAME);
  const beforeRows = sh.getLastRow();

  const fakeEvent = {
    parameter: {
      name: "Bot Name",
      email: "bot@example.com",
      phone: "555",
      service: "Tier 1",
      vehicle: "Spam",
      message: "Spam",
      _origin: "https://skippr.us",
      company_website: "bot-filled-this",
    }
  };
  const response = doPost(fakeEvent);
  const result = JSON.parse(response.getContent());
  if (!result.ok) throw new Error("Expected silent ok:true; got " + JSON.stringify(result));

  const afterRows = sh.getLastRow();
  if (afterRows !== beforeRows) {
    throw new Error("Honeypot should not have written a row. before=" + beforeRows + " after=" + afterRows);
  }
  Logger.log("✅ test_doPost_honeypot passed (no row written, no emails sent).");
}

function test_doPost_missingField() {
  const fakeEvent = {
    parameter: {
      name: "Sarah",
      phone: "(555) 000-0000",
      service: "Tier 1",
      vehicle: "Test",
      message: "Test",
      _origin: "https://skippr.us",
      company_website: "",
    }
  };
  const response = doPost(fakeEvent);
  const result = JSON.parse(response.getContent());
  if (result.ok) throw new Error("Expected ok:false when email missing; got " + JSON.stringify(result));
  if (!result.error || result.error.indexOf("email") === -1) {
    throw new Error("Expected error about email; got " + JSON.stringify(result));
  }
  Logger.log("✅ test_doPost_missingField passed: " + result.error);
}

function test_doPost_badOrigin() {
  const fakeEvent = {
    parameter: {
      name: "Sarah",
      email: "test@example.com",
      phone: "(555) 000-0000",
      service: "Tier 1",
      vehicle: "Test",
      message: "Test",
      _origin: "https://evil.example.com",
      company_website: "",
    }
  };
  const response = doPost(fakeEvent);
  const result = JSON.parse(response.getContent());
  if (result.ok) throw new Error("Expected ok:false on bad origin; got " + JSON.stringify(result));
  if (!result.error || result.error.indexOf("origin") === -1) {
    throw new Error("Expected origin error; got " + JSON.stringify(result));
  }
  Logger.log("✅ test_doPost_badOrigin passed: " + result.error);
}
```

- [ ] **Step 3: Commit.**

```bash
git add CRM/ConsultIntake.gs.txt
git commit -m "Add doPost, handleConsultSubmission orchestrator, and integration tests"
```

---

## Task 7: Modify consult.html — honeypot, _origin, placeholder form action

**Files:**
- Modify: `skippr/consult.html:233` (form action)
- Modify: `skippr/consult.html` around line 272 (add honeypot input)
- Modify: `skippr/consult.html` around line 309 (fetch body)

- [ ] **Step 1: Change the form action URL on line 233.**

Use the Edit tool to change:

```
old_string:  <form id="consultForm" action="https://formspree.io/f/mpqoqabj" method="POST">
new_string:  <form id="consultForm" action="https://script.google.com/macros/s/REPLACE_WITH_DEPLOYMENT_ID/exec" method="POST">
```

The real deployment URL replaces `REPLACE_WITH_DEPLOYMENT_ID` in Task 11.

- [ ] **Step 2: Add the honeypot input just before `<button type="submit"`.**

Locate the submit button around line 272 (`<button type="submit" class="submit-btn">Request Invitation</button>`) and insert this line immediately before it:

```html
          <input type="text" name="company_website" tabindex="-1" autocomplete="off" aria-hidden="true" style="position:absolute;left:-9999px;width:1px;height:1px;opacity:0;">
```

The Edit call:

```
old_string:           <button type="submit" class="submit-btn">Request Invitation</button>
new_string:           <input type="text" name="company_website" tabindex="-1" autocomplete="off" aria-hidden="true" style="position:absolute;left:-9999px;width:1px;height:1px;opacity:0;">
          <button type="submit" class="submit-btn">Request Invitation</button>
```

- [ ] **Step 3: Modify the fetch block around line 307 to append `_origin`.**

Change:

```
old_string:      const res = await fetch(this.action, {
        method: 'POST',
        body: new FormData(this),
        headers: { 'Accept': 'application/json' }
      });
new_string:      const formData = new FormData(this);
      formData.append('_origin', window.location.origin);
      const res = await fetch(this.action, {
        method: 'POST',
        body: formData,
        headers: { 'Accept': 'application/json' }
      });
```

- [ ] **Step 4: Commit.**

```bash
git add consult.html
git commit -m "Wire consult form to Apps Script placeholder (honeypot + _origin)"
```

---

## Task 8: [USER ACTION] Deploy to Apps Script, configure, run tests

This task runs outside the code editor — the user (M. Humphress) does it in Google Drive / the Apps Script web editor. No git activity during this task.

- [ ] **Step 1: Create the standalone Apps Script project.**

   1. Open https://drive.google.com in the browser logged in as the Workspace owner.
   2. Click **New → More → Google Apps Script**. If "Google Apps Script" isn't in the menu: **New → More → Connect more apps**, search "Apps Script", connect, then retry.
   3. Name the project **"Skippr Consult Intake"** (top-left text field where it says "Untitled project").

- [ ] **Step 2: Paste the source.**

   1. Delete the default `Code.gs` contents.
   2. Open `skippr/CRM/ConsultIntake.gs.txt` locally, copy its full contents.
   3. Paste into `Code.gs` in the editor.
   4. **Save** (Ctrl+S or 💾 icon).

- [ ] **Step 3: Configure the CONFIG block.**

Get the CRM spreadsheet ID:

   1. Open the Skippr CRM spreadsheet in Google Sheets.
   2. Copy the ID from the URL — it's the long string between `/d/` and `/edit`. Example URL `https://docs.google.com/spreadsheets/d/1ABCdefGHIJklmNOP-qrs_tUV/edit#gid=0` → ID is `1ABCdefGHIJklmNOP-qrs_tUV`.

Back in the Apps Script editor, edit the `CONFIG` object at the top of the file:

   - Replace `"REPLACE_WITH_CRM_SPREADSHEET_ID"` with the copied ID.
   - Replace `"REPLACE_WITH_YOUR_PERSONAL_EMAIL"` with the personal email address you want test emails to land in (e.g., `mhumphress@gmail.com`).
   - Save (Ctrl+S).

- [ ] **Step 4: Run `setup()` and authorize.**

   1. In the editor's function dropdown (top toolbar), select `setup`.
   2. Click **Run**.
   3. Google prompts: "Authorization required." Click **Review permissions** → choose your Workspace account → **Advanced → Go to Skippr Consult Intake (unsafe)** (this warning is normal for unverified scripts you own) → **Allow**.
   4. The script re-runs automatically. View the execution log (**View → Executions** or `Ctrl+Enter`).
   5. Expected log output: `✅ Setup complete. 📥 Leads tab is ready.`
   6. Open the CRM spreadsheet in another tab. Confirm the new 📥 Leads tab is visible with formatted headers in row 1.

- [ ] **Step 5: Run every `test_*` function from the editor.**

In this order, select each from the function dropdown, click **Run**, and check the execution log for `✅` messages. Fix any errors by editing the script, saving, and re-running:

   - `test_firstNameOf`
   - `test_appendLead`
   - `test_sendAutoReply` — verify the email arrives in your TEST_CUSTOMER_EMAIL inbox
   - `test_sendInternalNotification` — verify the email arrives in `support@skippr.us`
   - `test_doPost_happyPath` — verify both emails arrive and the test row was auto-cleaned up
   - `test_doPost_honeypot` — verify no emails, no row
   - `test_doPost_missingField` — no emails expected
   - `test_doPost_badOrigin` — no emails expected

If all tests pass, proceed. If any test fails, report the failure to Claude (paste the log output) and **do not proceed** to step 6.

- [ ] **Step 6: Deploy as a Test Deployment.**

   1. Click **Deploy → Test deployments** (top-right).
   2. In the sidebar: Select type (gear icon) → **Web app**.
   3. Description: "v1-dev".
   4. Execute as: **Me (mhumphress@skippr.us or your Workspace account)**.
   5. Click **Done**.
   6. Copy the **Web app URL** — it ends in `/dev`.
   7. Paste the `/dev` URL into the chat for Claude to validate.

---

## Task 9: [CLAUDE ACTION] Validate /dev URL with curl

**Files:** None. Purely HTTP validation.

- [ ] **Step 1: Run curl against the /dev URL — happy path.**

Claude runs (after user provides the `<DEV_URL>`):

```bash
curl -L -X POST "<DEV_URL>" \
  -F "name=_CURL_ Sarah Thompson" \
  -F "email=<TEST_CUSTOMER_EMAIL>" \
  -F "phone=(555) 000-0000" \
  -F "service=Tier 1 - Expert Advocate" \
  -F "vehicle=Test Vehicle from curl" \
  -F "message=Curl happy path test" \
  -F "_origin=https://skippr.us"
```

Expected response: `{"ok":true}`

Expected side effects:
- A new row in 📥 Leads with name starting `_CURL_ Sarah Thompson`. (User deletes it manually after verification.)
- Auto-reply in TEST_CUSTOMER_EMAIL inbox.
- Internal notification in support@skippr.us.

**If `/dev` URL requires login** (it does, by design — only the script owner can hit it): the curl will return HTML that redirects to Google login. That's expected. In this case, skip curl and have the user simulate the POST using a browser extension or the Apps Script editor's `test_doPost_happyPath` function (already run in Task 8). Proceed to Task 10 once the editor-run tests pass.

- [ ] **Step 2: Run curl — honeypot.**

```bash
curl -L -X POST "<DEV_URL>" \
  -F "name=Bot" -F "email=bot@example.com" -F "phone=555" \
  -F "service=Tier 1" -F "vehicle=Spam" -F "message=Spam" \
  -F "_origin=https://skippr.us" \
  -F "company_website=bot-filled"
```

Expected: `{"ok":true}` + no new row + no emails.

- [ ] **Step 3: Run curl — missing field.**

```bash
curl -L -X POST "<DEV_URL>" \
  -F "name=Sarah" -F "phone=(555) 000-0000" \
  -F "service=Tier 1" -F "vehicle=Test" -F "message=Test" \
  -F "_origin=https://skippr.us"
```

Expected: `{"ok":false,"error":"Missing field: email"}`.

---

## Task 10: [USER ACTION] Promote to production deployment

**Files:** None.

- [ ] **Step 1: Create a versioned production deployment.**

   1. In the Apps Script editor: **Deploy → New deployment**.
   2. Select type (gear icon) → **Web app**.
   3. Description: "v1 — production".
   4. Execute as: **Me**.
   5. Who has access: **Anyone** (this is required — the form must be able to POST from skippr.us anonymously).
   6. Click **Deploy**.
   7. If prompted to re-authorize, click through (same flow as Task 8 Step 4).
   8. Copy the **Web app URL** — it ends in `/exec`. This is the stable, public URL.
   9. Paste the `/exec` URL into the chat for Claude.

---

## Task 11: Update consult.html with the real /exec URL and commit

**Files:**
- Modify: `skippr/consult.html:233`

- [ ] **Step 1: Replace the placeholder URL with the real one.**

Edit `skippr/consult.html` line 233:

```
old_string:  <form id="consultForm" action="https://script.google.com/macros/s/REPLACE_WITH_DEPLOYMENT_ID/exec" method="POST">
new_string:  <form id="consultForm" action="<EXEC_URL>" method="POST">
```

Where `<EXEC_URL>` is the exact URL pasted by the user in Task 10 — something like `https://script.google.com/macros/s/AKfycbx.../exec`.

- [ ] **Step 2: Commit.**

```bash
cd "C:/Users/Mchum/OneDrive/Desktop/Claude Projects/Website/skippr"
git add consult.html
git commit -m "Point consult form at production Apps Script /exec URL"
```

---

## Task 12: [USER ACTION] Push to GitHub and run live smoke test

**Files:** None.

- [ ] **Step 1: Push to `origin/main`.**

```bash
cd "C:/Users/Mchum/OneDrive/Desktop/Claude Projects/Website/skippr"
git push origin main
```

Cloudflare Pages auto-builds in ~60 seconds. Watch for the deploy to complete — either via the Cloudflare dashboard (**Workers & Pages → skippr → Deployments**) or by hitting `https://skippr.us/consult.html` and confirming the new markup loaded (right-click → View Source, find the form action URL; it should be the `/exec` URL).

- [ ] **Step 2: Submit one real form from the live site.**

   1. Open `https://skippr.us/consult.html` (in an incognito window — no login).
   2. Fill out the form with real-ish data. Use a real email you can check (the TEST_CUSTOMER_EMAIL used earlier is fine; flag the "name" field with a `_LIVE_SMOKE_` prefix so it's obvious in the Leads sheet).
   3. Click **Request Invitation**.
   4. Verify: browser redirects to `thank-you.html`.

- [ ] **Step 3: Verify all three effects.**

   - 📥 Leads sheet: new row with the `_LIVE_SMOKE_` prefix.
   - TEST_CUSTOMER_EMAIL inbox: auto-reply from `"Skippr Concierge" <support@skippr.us>` with subject "Your Skippr request has been received".
   - support@skippr.us inbox (anyone on the dist group): internal notification with subject starting "New consult lead — _LIVE_SMOKE_".

- [ ] **Step 4: Clean up the smoke-test row.**

   In the 📥 Leads sheet, right-click the `_LIVE_SMOKE_` row → Delete row. The form is live and production-ready.

---

## Rollback

If anything in Task 12 fails (wrong behavior, blank responses, customer errors visible on-site):

```bash
cd "C:/Users/Mchum/OneDrive/Desktop/Claude Projects/Website/skippr"
git revert HEAD
git push origin main
```

Cloudflare redeploys the pre-cutover version within ~60 seconds. Formspree is active again.

Alternate recovery without a revert: in the Apps Script editor, **Deploy → Manage deployments → edit the production deployment → select a previous version**. Restores the previous behavior in ~15 seconds without touching the website.

---

## Out of scope

- Editing `Skippr_CRM.gs` or changing the existing CRM's behavior.
- Cloudflare Pages Function, Cloudflare Turnstile, shared-secret tokens, IP rate limiting.
- Automatic promotion of Leads rows into the 🚗 Pipeline tab.
- Updating the Cloudflare DNS SPF record (DKIM handles deliverability; this is a separate issue).
- Deleting the Formspree `mpqoqabj` form. (Can be done manually from the Formspree dashboard any time after Task 12 passes.)

---

## Plan self-review notes

Reviewed against the spec (`docs/superpowers/specs/2026-04-23-consult-auto-reply-design.md`). Every spec section has a task or preconditions entry that implements it:

- Architecture (standalone Apps Script, opens CRM by ID) → Task 1, Task 8
- Components list (doPost, handleConsultSubmission, appendLead, sendAutoReply, sendInternalNotification, setup, buildLeads, firstNameOf, formatTimestamp, tests) → Tasks 1-6
- 📥 Leads sheet schema (columns, widths, formatting) → Task 1 `buildLeads()`
- Sender display name `"Skippr Concierge" <support@skippr.us>` → Task 4, Task 5
- America/Indianapolis timezone → Task 1 CONFIG + `formatTimestamp()`
- Customer auto-reply copy → Task 4 `sendAutoReply()` body
- Internal notification copy with Reply-To set to customer → Task 5 `sendInternalNotification()`
- consult.html changes (action URL, honeypot, _origin) → Task 7 and Task 11
- Spam defenses (honeypot + origin check) → Task 6 `handleConsultSubmission()`
- Deployment sequence → Tasks 8, 10, 12
- Rollback plan → Rollback section above
- Testing strategy (test_* functions run from Apps Script editor) → Tasks 1-6 (tests), Task 8 Step 5
- Out of scope items → Out of scope section above

Type / name consistency check: `CONFIG.LEADS_SHEET_NAME`, `CONFIG.SUPPORT_EMAIL`, `CONFIG.SENDER_NAME`, `CONFIG.TIMEZONE`, `CONFIG.SHEET_ID`, `CONFIG.ALLOWED_ORIGINS`, `CONFIG.TEST_CUSTOMER_EMAIL` used identically across all tasks. `appendLead(ss, data)`, `sendAutoReply(data)`, `sendInternalNotification(data)`, `firstNameOf(fullName)`, `formatTimestamp(date)` signatures match wherever called.
