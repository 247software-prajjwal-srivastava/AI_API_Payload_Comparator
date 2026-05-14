# AI_API_Payload_Comparator
Vibe coded an api payload comparartor tool which cuts the manual testing effort of payload comparison by 80%.
# API Payload Comparator

An automated testing tool that compares API payloads between two versions (v1 and v2) of a React/Next.js application. It reads your source code, understands what CRUD operations exist, auto-generates test scenarios, drives both UIs in a real browser, intercepts every mutation request, and produces a visual diff report — all without writing a single test manually.

---

## Why This Exists

When migrating or rewriting a frontend (v1 → v2), the backend contract often shifts in subtle ways: a field gets renamed from `firstName` to `first_name`, an extra wrapper object appears around the payload, a `DELETE` body that used to send `{ id }` now sends nothing, or a date format changes from ISO to Unix timestamp. These differences cause silent bugs that only surface in production.

This tool catches all of them automatically by comparing what each frontend version actually sends over the wire.

---

## Architecture Overview

The tool runs in four sequential phases. Each phase produces intermediate output (saved as JSON in `reports/`) so you can inspect, debug, or re-run from any stage.

```
┌─────────────────────┐     ┌─────────────────────┐
│   v1 Source Code     │     │   v2 Source Code     │
└────────┬────────────┘     └────────┬────────────┘
         │                           │
         ▼                           ▼
╔═══════════════════════════════════════════════════╗
║  PHASE 1: Code Analysis (Babel AST)              ║
║  Extracts API calls, forms, inline edits,        ║
║  and maps them to routes                         ║
╚═══════════════════════════╤═══════════════════════╝
                            │
                            ▼
╔═══════════════════════════════════════════════════╗
║  PHASE 2: Scenario Generation                    ║
║  Matches v1↔v2 operations, builds executable     ║
║  step sequences (navigate, fill, submit)         ║
╚═══════════════════════════╤═══════════════════════╝
                            │
                            ▼
╔═══════════════════════════════════════════════════╗
║  PHASE 3: Browser Execution (Playwright)         ║
║  Logs in, replays steps on both UIs,             ║
║  intercepts POST/PUT/PATCH/DELETE requests        ║
╚═══════════════════════════╤═══════════════════════╝
                            │
                            ▼
╔═══════════════════════════════════════════════════╗
║  PHASE 4: Diff & Report                         ║
║  Deep payload comparison, rename detection,       ║
║  HTML report generation                          ║
╚═══════════════════════════════════════════════════╝
                            │
                            ▼
                    reports/report.html
```

---

## Phase 1 — Code Analysis

**File:** `analyze.js`
**Input:** Two directories containing React/Next.js source code
**Output:** `reports/analysis.json`

The analyzer uses Babel's AST parser to walk every `.js`, `.jsx`, `.ts`, and `.tsx` file in both codebases. It extracts:

**API Calls (mutations only)**
- `fetch()` calls with `method: "POST"`, `"PUT"`, `"PATCH"`, or `"DELETE"`
- `axios.post()`, `axios.put()`, `axios.patch()`, `axios.delete()` — also recognizes custom instances named `api`, `http`, or `client`
- Custom hook invocations matching patterns like `useMutation`, `useCreate`, `useUpdate`, `useDelete`, `useSWRMutation`, `useSubmit`, `useSave`

For each API call it captures the HTTP method, URL (including template literals), payload field names extracted from the request body object, and the raw code snippet around the call.

**Forms**
- HTML `<form>` elements and their child `<input>`, `<select>`, and `<textarea>` fields
- For each field: `name`, `type`, `placeholder`, `id`, `data-testid`, and `aria-label` attributes
- The form's `action` and `method` attributes

**Inline Edit Patterns**
- `onBlur` handlers (common for save-on-blur inline editing)
- `onDoubleClick` handlers (click-to-edit patterns)
- `contentEditable` attributes

**Route Mapping**
Every detected element is mapped to a route using Next.js file-system conventions:
- App Router: `app/users/page.tsx` → `/users`
- Pages Router: `pages/users/index.tsx` → `/users`
- Dynamic segments: `[id]` → `:id`

This means the tool knows which page to navigate to in order to trigger each API call.

---

## Phase 2 — Scenario Generation

**File:** `generate-scenarios.js`
**Input:** Phase 1 analysis results for both versions
**Output:** `reports/scenarios.json`

The generator takes the extracted API calls from both codebases and produces executable test scenarios.

**Matching v1 ↔ v2 operations**

It runs a three-pass matching algorithm to pair up API calls between versions:

1. **Exact URL match** — After stripping version prefixes (`/api/v1/` → `/api/`), if two calls have the same normalized URL and HTTP method, they're paired. This catches the common case where v2 just bumps the API version.

2. **Same route match** — If two calls live on the same page route and use the same HTTP method, they're paired. This catches cases where the URL changed but the page didn't.

3. **Similar endpoint match** — If the last meaningful URL segment matches (e.g., both end in `/users`) and the method is the same, they're paired. This catches deeper URL restructuring.

Calls that don't match anything are flagged as "v1-only" or "v2-only" in the report.

**Building interaction steps**

For each matched pair, the generator builds a sequence of browser actions:

- **Navigate** to the correct route
- **Click into a record** (for PUT/PATCH/DELETE — clicks the first table row, list item, or edit link)
- **Click edit/delete buttons** using selector heuristics (looks for `data-testid`, `aria-label`, class names, and text content)
- **Fill form fields** using the field metadata from Phase 1 — each field gets a test value based on its type and name (email fields get an email, phone fields get a phone number, etc.)
- **Submit** by clicking the submit button

When no form is found for a POST operation, it falls back to an auto-fill strategy: click a "Create/Add/New" button, then fill every visible input on the page.

Test values are configurable in `config.json` under `options.testData`.

---

## Phase 3 — Browser Execution

**File:** `execute.js`
**Input:** Generated scenarios + config with URLs and credentials
**Output:** `reports/execution-results.json` + screenshots

The execution engine uses Playwright to run each scenario against both v1 and v2 staging environments.

**Login**
Before running scenarios, the engine logs into each version using form-based authentication. It fills the username/password fields using the CSS selectors from your config, clicks submit, and waits for the success indicator (a URL pattern or network idle).

**Network Interception**
A `page.on('request')` listener captures every `POST`, `PUT`, `PATCH`, and `DELETE` request. For each one it records:
- The full URL and path
- The HTTP method
- The request body (parsed as JSON when possible)
- Request headers
- Timestamp

**Step Execution**
Each scenario step is executed in order. The engine supports these action types:

| Action | Behavior |
|--------|----------|
| `navigate` | Goes to the specified route on the current base URL |
| `wait` | Pauses for a specified duration (ms) |
| `click` | Tries multiple CSS selectors until one matches, including `:has-text()` for text-based matching |
| `click_first_record` | Clicks the first row/item in a table or list |
| `fill` | Fills an input with a test value; handles checkboxes (toggle), selects (pick first option), and text inputs |
| `fill_all_visible` | Auto-detects every visible input on the page and fills it with an appropriate value |

**Screenshots**
When `screenshotOnDiff` is enabled in the config, the engine takes a full-page screenshot after each scenario completes, saved to `reports/screenshots/`.

**Error Handling**
Each scenario runs in an isolated browser context. If a scenario fails on one version, the other version still executes. Errors are captured and included in the report rather than stopping the run.

---

## Phase 4 — Diff & Report

**Files:** `diff-engine.js` + `report-generator.js`
**Input:** Execution results with captured request payloads
**Output:** `reports/diff-report.json` + `reports/report.html`

### Diff Engine

**Request Pairing**
Captured requests from v1 and v2 are paired by HTTP method and URL similarity. The similarity algorithm normalizes URLs (strips version prefixes, replaces numeric IDs and UUIDs with placeholders) and scores by segment overlap. Pairs scoring above 0.3 similarity are matched; the rest appear as unmatched requests.

**Deep Comparison**
Paired payloads are recursively compared field by field. The diff engine categorizes every difference:

| Diff Type | Meaning |
|-----------|---------|
| `added` | Field exists in v2 but not v1 |
| `removed` | Field exists in v1 but not v2 |
| `value_changed` | Same field, different value |
| `type_changed` | Same field, different type (e.g., string → number) |
| `array_length_changed` | Same array field, different length |

Paths are dot-notation for nested fields: `user.address.zipCode`.

**Field Rename Detection**
When a field is removed in v1 and a new field is added in v2, the engine checks whether it might be a rename rather than a true add/remove. It scores candidates on:

- **Value matching** (0.6 weight) — same value is the strongest signal (`firstName: "John"` removed, `first_name: "John"` added)
- **Name similarity** (0.3 weight) — one name contains the other, or Levenshtein distance is low
- **Same parent** (0.1 weight) — both fields live at the same nesting level

Pairs scoring above 0.5 confidence are reported as probable renames.

**Ignored Fields**
Fields listed in `config.options.ignoredPayloadFields` (e.g., `_csrf`, `timestamp`, `requestId`) are excluded from the diff.

### Report Generator

The HTML report is fully self-contained (inline CSS, inline JS, Google Fonts link) and opens in any browser. It includes:

- **Summary dashboard** — cards showing total scenarios, identical count, diffs found, version-only count, and errors
- **Expandable scenario sections** — each scenario has a header with HTTP method badge and status badge; scenarios with diffs are expanded by default
- **Side-by-side payload view** — v1 and v2 JSON payloads displayed in parallel columns
- **Diff table** — color-coded rows for each difference (green for added, red for removed, amber for changed, purple for type changes)
- **Rename detection panel** — shows detected field renames with confidence percentages
- **Unmatched request warnings** — lists API calls that only appeared in one version
- **Expand/Collapse All** toggle

---

## File Structure

```
api-payload-comparator/
├── config.json              # Your URLs, code paths, credentials, options
├── package.json             # Dependencies and npm scripts
├── run.js                   # CLI entry point — orchestrates all 4 phases
├── analyze.js               # Phase 1: Babel AST code scanner
├── generate-scenarios.js    # Phase 2: Scenario matcher and step builder
├── execute.js               # Phase 3: Playwright browser automation
├── diff-engine.js           # Phase 4a: Deep payload comparison
├── report-generator.js      # Phase 4b: HTML report builder
├── README.md                # This file
└── reports/                 # Generated outputs (created on first run)
    ├── analysis.json
    ├── scenarios.json
    ├── execution-results.json
    ├── diff-report.json
    ├── report.html
    └── screenshots/
```

---

## Setup

**Prerequisites**
- Node.js 18 or higher
- Both app codebases accessible on disk (cloned repos or local copies)
- Both app versions running at accessible URLs (local dev servers, staging, or production)

**Install**

```bash
npm install
```

This installs all dependencies and downloads Chromium for Playwright via the `postinstall` script.

**Configure**

Edit `config.json`:

```jsonc
{
  "v1": {
    "baseUrl": "https://staging-v1.yourapp.com",   // Where v1 is running
    "codePath": "./app-v1",                         // Path to v1 source code
    "login": {
      "url": "/login",                              // Login page path
      "usernameSelector": "input[name='email']",    // CSS selector for username field
      "passwordSelector": "input[type='password']", // CSS selector for password field
      "submitSelector": "button[type='submit']",    // CSS selector for login button
      "username": "testuser@example.com",           // Test account username
      "password": "testpass123",                    // Test account password
      "successIndicator": "/dashboard"              // URL pattern after successful login
    }
  },
  "v2": {
    // Same structure as v1, pointing to v2's URL and code
  },
  "options": {
    "headless": true,              // false to see the browser
    "timeout": 30000,              // Max wait time per action (ms)
    "waitAfterAction": 2000,       // Pause after submitting to catch async requests
    "screenshotOnDiff": true,      // Take screenshots after each scenario
    "ignoredPayloadFields": [      // Fields to exclude from comparison
      "_csrf", "timestamp", "requestId", "nonce"
    ],
    "testData": {                  // Values used to fill form fields
      "string": "Test Automation Data",
      "email": "autotest@example.com",
      "number": "12345",
      "phone": "+1234567890",
      "url": "https://example.com",
      "date": "2026-01-15",
      "textarea": "Automated test input."
    }
  }
}
```

---

## Usage

**Full run (all 4 phases)**

```bash
node run.js
```

**Override paths from the command line**

```bash
node run.js --v1-code ../old-app --v2-code ../new-app \
            --v1-url http://localhost:3000 --v2-url http://localhost:3001
```

**Watch the browser while it runs**

```bash
node run.js --headed
```

**Only analyze code (no browser needed)**

```bash
node run.js --analyze-only
```

This is useful to verify the tool found all your API calls before running the full suite. Check `reports/analysis.json`.

**Analyze + generate scenarios (no browser needed)**

```bash
node run.js --scenarios-only
```

Review `reports/scenarios.json` to see exactly what the tool plans to do before letting it loose on your staging environment.

**Use a different config file**

```bash
node run.js --config ./config-staging.json
```

**npm script shortcuts**

```bash
npm start              # node run.js
npm run analyze        # node run.js --analyze-only
npm run scenarios      # node run.js --scenarios-only
npm run headed         # node run.js --headed
```

---

## CLI Reference

| Flag | Description |
|------|-------------|
| `--config <path>` | Path to config file (default: `./config.json`) |
| `--v1-code <path>` | Override v1 codebase path |
| `--v2-code <path>` | Override v2 codebase path |
| `--v1-url <url>` | Override v1 staging URL |
| `--v2-url <url>` | Override v2 staging URL |
| `--analyze-only` | Run Phase 1 only |
| `--scenarios-only` | Run Phases 1–2 only |
| `--headed` | Show the browser window |
| `--help` | Print help and exit |

---

## What It Detects

**API patterns recognized by the code analyzer:**
- `fetch(url, { method: "POST", body: JSON.stringify({...}) })`
- `axios.post(url, payload)` — also `.put()`, `.patch()`, `.delete()`
- Custom HTTP client instances (`api.post()`, `http.put()`, `client.patch()`)
- React Query / SWR mutation hooks (`useMutation`, `useSWRMutation`)
- Custom hooks following the naming convention `useCreate*`, `useUpdate*`, `useDelete*`, `useSubmit*`, `useSave*`
- Server actions (detected as function calls with mutation semantics)

**Form patterns:**
- Standard HTML `<form>` elements with `<input>`, `<select>`, `<textarea>` children
- Field metadata extraction: name, type, placeholder, id, data-testid, aria-label

**Inline edit patterns:**
- `onBlur` save handlers
- `onDoubleClick` edit triggers
- `contentEditable` elements

**Framework support:**
- Next.js App Router (`app/*/page.tsx`)
- Next.js Pages Router (`pages/*.tsx`)
- Generic React (maps components to directory-based routes)

---

## Interpreting the Report

Open `reports/report.html` in any browser. Here's how to read it:

**Summary cards at the top**
- **Identical** — scenarios where v1 and v2 sent exactly the same payload structure and values (excluding ignored fields)
- **Diffs Found** — scenarios where payloads differ (these are expanded by default)
- **Version-Only** — API operations that exist in only one version
- **Errors** — scenarios that failed to execute (selector not found, timeout, crash)

**Scenario sections**
Each section represents one CRUD operation. The method badge shows the HTTP verb; the status badge shows the result.

**Diff table columns**
- **Type** — the kind of difference (ADDED, REMOVED, VALUE CHANGED, TYPE CHANGED, ARRAY LENGTH CHANGED)
- **Field Path** — dot-notation path to the differing field (e.g., `user.profile.avatar`)
- **v1 Value** — what v1 sent (red)
- **v2 Value** — what v2 sent (green)

**Rename detection**
When the tool suspects a field was renamed rather than removed-and-added, it shows a purple panel with the old name, new name, and confidence score. High confidence (>80%) usually means the values matched exactly and the names are similar.

**Unmatched requests**
Amber warnings at the bottom of a scenario indicate API calls that fired in one version but couldn't be paired with anything in the other. This often means v2 split one endpoint into two, or consolidated multiple calls into one.

---

## Troubleshooting

**"Code path not found"**
Make sure `codePath` in config.json points to the root of each app's source directory (the folder containing `src/`, `app/`, or `pages/`).

**"Could not find clickable element"**
The tool tries multiple CSS selector strategies but may miss custom component libraries. You have two options:
1. Add `data-testid` attributes to your components (recommended anyway for testing)
2. Review `reports/scenarios.json`, find the failing step, and check which selectors it tried

**Login fails**
Verify your login selectors by opening the login page in DevTools and testing the CSS selectors manually. The `successIndicator` should be a URL path that the browser navigates to after successful login.

**No API calls detected**
Run with `--analyze-only` and check `reports/analysis.json`. If it's empty, the analyzer might not recognize your API call pattern. The tool looks for `fetch`, `axios`-like clients, and hooks matching `use[Create|Update|Delete|Mutation|Submit|Save]*`. If you use a different convention, the analyzer would need to be extended.

**Playwright won't install**
On some Linux systems, Playwright needs additional system dependencies. Run:
```bash
npx playwright install-deps chromium
```

---

## Limitations

- The tool works with form-based login only (not OAuth/SSO redirects, magic links, or 2FA)
- It fills forms with synthetic test data — it won't replay real user sessions
- Complex multi-step workflows (step 1 creates a record, step 2 edits it) are handled as separate scenarios, not chained
- Client-side validation that blocks submission may prevent some scenarios from reaching the API call
- The code analyzer reads static source code — dynamically constructed API calls (e.g., URLs built entirely at runtime from state) will show as `(dynamic)` and can't be auto-tested
- WebSocket and GraphQL mutations are not currently intercepted (REST only)

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `@babel/parser` | Parses JS/TS/JSX/TSX into AST |
| `@babel/traverse` | Walks the AST to extract API calls and forms |
| `glob` | File discovery across both codebases |
| `playwright` | Browser automation and network interception |
