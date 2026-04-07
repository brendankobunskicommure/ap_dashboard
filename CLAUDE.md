# AP Dashboard — Build Plan

**Goal:** Build an Accounts Payable aging dashboard that mirrors the AR dashboard (`cmr-ar-dashboard.web.app`) exactly in structure and UX, but for vendors instead of customers.

**Stack:** Vite + React 18 + Firebase Hosting + Firebase Cloud Functions (Node 20) + BigQuery (`cmr-finance-data-lake`)

**Reference repo:** `/Users/bkobunski/Desktop/CoWork/ar_dashboard` — this is the source of truth. All work below is a direct AR → AP translation unless stated otherwise.

---

## AR → AP Terminology Mapping

| AR Concept | AP Equivalent | Notes |
|---|---|---|
| Customer | Vendor | `customer_name` → `vendor_name`, `customer_id` → `vendor_id` |
| Open Invoice | Open Bill | NetSuite type `VendBill`, status `A` = Open, `B` = Paid In Full |
| Credit Memo | Vendor Credit | NetSuite type `VendCred`, status `A` = Open, `B` = Fully Applied |
| Customer Payment (received) | Vendor Payment (sent) | NetSuite type `VendPymt`, status `B` = In Transit, `C` = Cleared |
| Net AR (asset) | Net AP (liability) | Net = open bills − vendor credits − unapplied prepayments |
| `ar_notes` / `ar_saved_filters` | `ap_notes` / `ap_saved_filters` | Firestore collections |
| `arData` Cloud Function | `apData` Cloud Function | Firebase function name |
| `cmr-ar-dashboard` hosting target | `cmr-ap-dashboard` hosting target | Firebase hosting alias |
| `OTC_Dev` dataset | `ptp_dev` dataset | Purchase-to-Pay dataset in BigQuery |
| `vw_Invoices` | `vw_Bills` (confirm name) | Need to verify actual view name in `ptp_dev` |
| `vw_CreditMemos` | `vw_VendorCredits` (confirm name) | Need to verify actual view name in `ptp_dev` |
| `vw_CustomerPayments` | `vw_VendorPayments` (confirm name) | Need to verify actual view name in `ptp_dev` |

> **Note on ptp_dev view names:** Before writing SQL, run `SELECT table_name FROM cmr-finance-data-lake.ptp_dev.INFORMATION_SCHEMA.TABLES` to confirm the actual `vw_*` view names. The AR dashboard uses `vw_Invoices`, `vw_CreditMemos`, `vw_CustomerPayments` from `OTC_Dev`. The AP equivalent likely lives in `ptp_dev` with similar naming but may differ.

---

## Step-by-Step Delegation Plan

Each step below is a discrete agent task. Steps 1–3 can be executed in parallel once the `ptp_dev` view names are confirmed. Steps 4–6 depend on Step 3 being deployed. Step 7 is final.

---

### Step 0 — Discover ptp_dev Schema

**Delegate to:** `Acting as data-engineer:`

**What to do:**
1. Query `cmr-finance-data-lake.ptp_dev.INFORMATION_SCHEMA.TABLES` to list all views
2. For each `vw_*` view found, run `SELECT * FROM ... LIMIT 1` to inspect column names
3. Identify which views correspond to: vendor bills, vendor credits, vendor payments
4. Map AP field names from the schema CSV (`/Users/bkobunski/Desktop/CoWork/Fingerprinting Journal Entries/Instructions/NetSuite_Live - Schema v2.csv`) to confirm `entity` = vendor on AP transactions
5. Return: confirmed view names, key column names for amounts (`foreignamountunpaid`, `foreignpaymentamountunused`, etc.), and any AP-specific quirks vs. AR

**Output needed:** Confirmed table/view names and field mappings before proceeding to Steps 1–3.

---

### Step 1 — BigQuery SQL Views (5 views)

**Delegate to:** `Acting as data-engineer:`

**Reference:** `/Users/bkobunski/Desktop/CoWork/ar_dashboard/sql/views/` — copy all 5 AR views and translate to AP

**Create in `cmr-finance-data-lake.ptp_dev`:**

| # | View Name | Source (AR equivalent) | Purpose |
|---|-----------|------------------------|---------|
| 1 | `ap_open_bills` | `ar_open_invoices` | Open vendor bills with aging buckets. Filter: `mainline=true, status='A', voided=false, void=false, posting=true` |
| 2 | `ap_open_bills_by_vendor` | `ar_open_invoices_by_customer` | Vendor-level aggregation of open bills |
| 3 | `ap_unapplied_vendor_credits` | `ar_unapplied_credit_memos` | Unapplied vendor credits. Filter: `SAFE_CAST(foreignpaymentamountunused AS FLOAT64) > 0` |
| 4 | `ap_unapplied_prepayments` | `ar_unapplied_payments` | Unapplied vendor prepayments. Same filter. |
| 5 | `ap_vendor_net_position` | `ar_customer_net_position` | Net AP = open bills − vendor credits − prepayments. 3-CTE pattern, LEFT JOINs on `vendor_id` |

**Key translation rules:**
- `customer_id` → `vendor_id`, `customer_name` → `vendor_name`, `customer_entityid` → `vendor_entityid`
- `customer_email` → `vendor_email` (confirm field exists)
- Aging logic is identical: `DATE_DIFF(CURRENT_DATE(), COALESCE(duedate, trandate), DAY)` with same 5 buckets
- Same `SAFE_CAST(foreignpaymentamountunused AS FLOAT64)` pattern for credits/payments
- Status decode for bills: `'A'='Open', 'B'='Paid In Full', 'V'='Voided'` (same as AR)
- Status decode for vendor credits: `'A'='Open (Unapplied)', 'B'='Fully Applied'` (same as AR)
- Net AP label: `net_ap_usd` (not `net_ar_usd`)

**Output files to create:**
- `sql/views/ap_open_bills.sql`
- `sql/views/ap_open_bills_by_vendor.sql`
- `sql/views/ap_unapplied_vendor_credits.sql`
- `sql/views/ap_unapplied_prepayments.sql`
- `sql/views/ap_vendor_net_position.sql`
- `sql/ap_dashboard_sql_layer.md` (same structure as AR equivalent, translated)

**Deployment order:** Views 1, 3, 4 first (no dependencies) → Views 2, 5 last (depend on Views 1, 3, 4)

---

### Step 2 — Firebase Cloud Function (`apData`)

**Delegate to:** `Acting as engineer:`

**Reference:** `/Users/bkobunski/Desktop/CoWork/ar_dashboard/functions/index.js`

**What to build:** A new Cloud Function `apData` co-located in the same Firebase project (`cmr-finance-data-lake`). This function is **added to the existing `functions/index.js`** — do NOT replace `arData`.

```javascript
// Pattern to follow — translate from arData:
const DATASET = "ptp_dev";
const QUERIES = {
  vendors:  `SELECT * FROM \`cmr-finance-data-lake.${DATASET}.ap_vendor_net_position\``,
  bills:    `SELECT * FROM \`cmr-finance-data-lake.${DATASET}.ap_open_bills\``,
  credits:  `SELECT * FROM \`cmr-finance-data-lake.${DATASET}.ap_unapplied_vendor_credits\``,
  payments: `SELECT * FROM \`cmr-finance-data-lake.${DATASET}.ap_unapplied_prepayments\``,
};
// Returns JSON: { vendors, bills, credits, payments }
// Same serializeRow, same CORS/timeout/memory settings
```

**firebase.json change:** Add AP rewrite rule:
```json
{ "source": "/api/ap/**", "function": "apData" }
```
(Keep existing `{ "source": "/api/**", "function": "arData" }` — do not remove it)

**Deploy command:** `firebase deploy --only functions:apData` (never redeploy arData inadvertently)

---

### Step 3 — React Frontend (`APDashboard.jsx`)

**Delegate to:** `Acting as frontend:`

**Reference:** `/Users/bkobunski/Desktop/CoWork/ar_dashboard/src/ARDashboard.jsx` (689 lines) — this is the full source. Mirror it exactly, making only the AP translation changes below.

**File to create:** `src/APDashboard.jsx`

**Translation checklist:**
- [ ] Fetch endpoint: `/api/arData` → `/api/ap/apData`
- [ ] All state/variable names: `customers` → `vendors`, `invoices` → `bills`, `creditMemos` → `vendorCredits`, `payments` → `prepayments`
- [ ] Field mappings: `customer_name` → `vendor_name`, `customer_id` → `vendor_id`, `net_ar_usd` → `net_ap_usd`, `open_invoices_usd` → `open_bills_usd`, `unapplied_credits_usd` (same), `unapplied_payments_usd` → `unapplied_prepayments_usd`
- [ ] Tab labels: "Open Invoices" → "Open Bills", "Credit Memos" → "Vendor Credits", "Payments" → "Prepayments"
- [ ] KPI labels: "Total Outstanding" → "Total Payable", "Net AR" → "Net AP"
- [ ] Column headers in tables: "Customer" → "Vendor", "Invoice" → "Bill", etc.
- [ ] Firestore collections: `ar_notes` → `ap_notes`, `ar_saved_filters` → `ap_saved_filters`
- [ ] Page title / header: "AR Aging Dashboard" → "AP Aging Dashboard"
- [ ] Keep all styling, colors, fonts, branding, and layout identical — no UI changes

**Do not change:**
- Auth logic (stays in `main.jsx`, no changes needed)
- Firebase config (`firebase.js`, no changes)
- Aging bucket colors, KPI card styles, sort logic, CSV export, filter architecture
- Commure branding

---

### Step 4 — Firestore Rules Update

**Delegate to:** `Acting as engineer:`

**Reference:** `/Users/bkobunski/Desktop/CoWork/ar_dashboard/firestore.rules`

**Add to existing `firestore.rules`** (do not remove AR rules):
```
match /ap_notes/{noteId} {
  allow read, write: if request.auth != null
    && request.auth.token.email.matches('.*@commure\\.com');
}
match /ap_saved_filters/{filterId} {
  allow read, write: if request.auth != null
    && request.auth.token.email.matches('.*@commure\\.com');
}
```

---

### Step 5 — Project Scaffolding & Config Files

**Delegate to:** `Acting as engineer:`

**Scaffold the `ap_dashboard` repo** at `/Users/bkobunski/Desktop/CoWork/ap_dashboard` to mirror `ar_dashboard` structure:

```
ap_dashboard/
├── src/
│   ├── main.jsx          # Copy from ar_dashboard (auth logic unchanged)
│   ├── APDashboard.jsx   # Created in Step 3
│   └── firebase.js       # Copy from ar_dashboard (same Firebase project)
├── functions/
│   ├── index.js          # apData function from Step 2
│   └── package.json      # Copy from ar_dashboard functions/package.json
├── sql/
│   └── views/            # 5 SQL files from Step 1
├── public/
│   ├── Commure-logo_full-horizontal_black.jpg   # Copy from ar_dashboard
│   └── commure-logo.png                          # Copy from ar_dashboard
├── .github/
│   └── workflows/
│       ├── firebase-hosting-merge.yml          # Copy + update target
│       └── firebase-hosting-pull-request.yml   # Copy + update target
├── CLAUDE.md
├── index.html            # Copy from ar_dashboard, update title to "AP Dashboard"
├── package.json          # Copy from ar_dashboard, update name to "ap-dashboard"
├── firebase.json         # AP-specific (see below)
├── .firebaserc           # AP-specific (see below)
├── firestore.rules       # AP rules from Step 4
└── vite.config.js        # Copy from ar_dashboard (unchanged)
```

**`firebase.json`** (AP-specific):
```json
{
  "firestore": { "rules": "firestore.rules" },
  "functions": { "source": "functions" },
  "hosting": {
    "target": "ap-dashboard",
    "public": "dist",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"],
    "rewrites": [
      { "source": "/api/ap/**", "function": "apData" },
      { "source": "**", "destination": "/index.html" }
    ]
  }
}
```

**`.firebaserc`** (AP-specific):
```json
{
  "projects": { "default": "cmr-finance-data-lake" },
  "targets": {
    "cmr-finance-data-lake": {
      "hosting": {
        "ap-dashboard": ["cmr-ap-dashboard"]
      }
    }
  }
}
```

> **Note:** The AP dashboard is a separate repo/deploy from AR. The `apData` function is deployed from this repo. Firestore rules need to be deployed from one place — coordinate with the AR dashboard deployment to avoid overwriting each other's rules. Safest: manage Firestore rules from the AR repo (which already has them) and add the AP rules there, then `firebase deploy --only firestore:rules` from that repo.

---

### Step 6 — Firebase Hosting Setup

**Delegate to:** `Acting as engineer:`

**What to do (manual steps, requires Firebase CLI):**
1. Create new hosting site in `cmr-finance-data-lake` project:
   ```bash
   firebase hosting:sites:create cmr-ap-dashboard --project cmr-finance-data-lake
   ```
2. Apply hosting target locally:
   ```bash
   firebase target:apply hosting ap-dashboard cmr-ap-dashboard
   ```
3. Verify `.firebaserc` was updated correctly

**Deploy sequence:**
```bash
# 1. Install deps
npm install
cd functions && npm install && cd ..

# 2. Build React app
npm run build

# 3. Deploy function only first, verify it works
firebase deploy --only functions:apData

# 4. Test function endpoint (should return JSON)
curl https://us-central1-cmr-finance-data-lake.cloudfunctions.net/apData

# 5. Deploy hosting
firebase deploy --only hosting:ap-dashboard

# 6. Deploy Firestore rules (coordinate with AR dashboard)
firebase deploy --only firestore:rules
```

---

### Step 7 — QA & Reconciliation

**Delegate to:** `Acting as qa:`

**Acceptance criteria:**
- [ ] App loads at `cmr-ap-dashboard.web.app` with Google auth (only @commure.com)
- [ ] All 3 tabs render: Open Bills, Vendor Credits, Prepayments
- [ ] KPI strip shows: Total Payable, Over 90 Days, Vendor Count, Net AP
- [ ] Vendor summary table is sortable by all columns
- [ ] Aging histogram renders per vendor
- [ ] Clicking a vendor row filters the detail table below
- [ ] Aging bucket filter (Current / 1-30 / 31-60 / 61-90 / 90+) works
- [ ] Notes can be added and persist via Firestore (`ap_notes`)
- [ ] Filters can be saved and loaded via Firestore (`ap_saved_filters`)
- [ ] CSV export works for all 3 tabs
- [ ] Data matches BQ view totals: `SELECT SUM(amount_unpaid_usd) FROM ptp_dev.ap_open_bills`
- [ ] No AR-specific terminology visible anywhere in the UI

---

## Execution Notes

- **Steps 1, 2, 3, 4** are independent and can be delegated in parallel once Step 0 (schema discovery) is complete.
- **Step 5** (scaffolding) can start in parallel with Steps 1–4 since it's primarily file copying.
- **Step 6** (deploy) requires Steps 1–5 complete.
- **Step 7** (QA) requires Step 6 complete.
- When delegating, always open the prompt with the role from `/Users/bkobunski/Desktop/CoWork/team-profiles/` (e.g., `Acting as data-engineer:`).
- BigQuery project: `cmr-finance-data-lake`, AP dataset: `ptp_dev`
- AR dashboard reference: `/Users/bkobunski/Desktop/CoWork/ar_dashboard`
- Schema reference: `/Users/bkobunski/Desktop/CoWork/Fingerprinting Journal Entries/Instructions/NetSuite_Live - Schema v2.csv`
