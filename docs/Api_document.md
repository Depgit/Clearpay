# Property Finance Layer API: India Edition

**Built on:** Zoho Books API v3, India data centre (`.in`)\
**Currency:** INR **Status:** design draft

Zoho details come from the official Zoho Books API docs. Anything marked **\[verify\]** was not confirmed in the docs and must be tested on a real organization before you build on it. Zoho publishes an OpenAPI file for Books (`openapi-all.zip` on the API docs site), so generate your Zoho client from that instead of hand-writing it.

This document has two parts:

- **Part A:** the Zoho Books calls your backend makes.
- **Part B:** the API your layer exposes to your own frontend.

---

# Part A: Zoho Books API (India)

## A1. Base URL, auth, organization

| Item | Value |
| --- | --- |
| API base URL | `https://www.zohoapis.in/books/v3` |
| OAuth (accounts) base URL | `https://accounts.zoho.in` |
| Auth header | `Authorization: Zoho-oauthtoken {{zoho_oauth_token_0afv}}` |
| Organization | `organization_id={{organization_id}}` query parameter on **every** request |

The header prefix is `Zoho-oauthtoken`, **not** `Bearer`. A request without `organization_id` fails.

```bash
curl -X GET 'https://www.zohoapis.in/books/v3/organizations' \
  -H 'Authorization: Zoho-oauthtoken {{zoho_oauth_token_0afv}}'
```

Get `organization_id` once from `GET /organizations`. The response also gives `currency_code`, `fiscal_year_start_month` and `time_zone`. Read the fiscal year from here instead of hardcoding it, because Indian organizations normally run April to March.

**Response envelope:** every response has `"code": 0` and `"message": "success"` on success, plus the resource under its own key (`bills`, `contacts`, and so on).

## A2. OAuth for India

Zoho uses the authorization code grant.

**1. Register a client** in the Zoho API Console (use the `.in` console for India accounts). For a SaaS product choose *Server-based Application*. For your own company's first case study you can use *Self Client*, which needs no redirect URL.

**2. Send the user to consent**

```
GET https://accounts.zoho.in/oauth/v2/auth
  ?scope=ZohoBooks.bills.ALL,ZohoBooks.contacts.READ
  &client_id={{client_id}}
  &response_type=code
  &redirect_uri={{redirect_uri}}
  &access_type=offline
  &prompt=consent
```

The returned `code` is valid for 2 minutes.

**3. Exchange the code**

```
POST https://accounts.zoho.in/oauth/v2/token
  grant_type=authorization_code
  client_id=...  client_secret=...
  redirect_uri=...  code=...
```

The response contains `access_token`, `refresh_token`, `expires_in` and `api_domain`.

**4. Refresh**

```
POST https://accounts.zoho.in/oauth/v2/token
  grant_type=refresh_token
  refresh_token=...  client_id=...  client_secret=...
```

Rules that shape the design:

- An access token lasts **1 hour**. Refresh before it expires.
- A refresh token does not expire until revoked, so store it **encrypted** per tenant and never send it to the browser.
- Each new consent creates a new refresh token. The limit is **20 per user**, and the oldest is silently deleted. Do not send users through consent repeatedly.
- Use the `api_domain` value from the token response as the API host. For India it should be `https://www.zohoapis.in`.

## A3. Scopes

Format: `ZohoBooks.<scope>.<CREATE|READ|UPDATE|DELETE|ALL>`

| Scope | Used for |
| --- | --- |
| `contacts` | Owners and vendors (read, create) |
| `bills` | Read bills; create, submit and approve bills |
| `expenses` | Read and create expenses, recurring expenses |
| `invoices` | Read revenue |
| `customerpayments` | Read receipts |
| `vendorpayments` | Record owner payouts |
| `creditnotes`, `debitnotes` | Read adjustments (debit notes = vendor credits) |
| `settings` | Taxes, categories, reporting tags **\[verify\]** |
| `accountants` | Chart of accounts, journals **\[verify\]** |

**MVP rule:** start with read-only scopes (`*.READ`) to sync and report. Add write scopes only when you build assignment write-back, bills and payouts.

## A4. Limits and pagination

| Limit | Value |
| --- | --- |
| Per minute | 100 requests per organization, then HTTP `429`. One docs page mentions a 30 per minute threshold, so design for the lower number until you test **\[verify\]** |
| Per day | Free 1,000 · Standard 2,000 · Professional 5,000 · Premium, Elite, Ultimate 10,000 |
| Concurrent | Free 5 · paid 10 (soft limit) |

Daily limits are the real constraint for syncing. A customer on the Standard plan has 2,000 calls a day, so incremental sync and local caching are not optional.

**Pagination:** `?page=1&per_page=200` (200 is the default and the maximum). Keep fetching while `page_context.has_more_page` is `true`.

```json
{ "code": 0, "message": "success", "bills": [], "page_context": { "page": 1, "per_page": 200, "has_more_page": false } }
```

## A5. Zoho endpoints your layer uses

### Property dimension: three options

| Option | Notes |
| --- | --- |
| **Reporting Tag (recommended)** | Apply a tag at transaction and line-item level. A tag can be set `is_mandatory`, so Zoho itself can force staff to pick a property. Classic limit is 10 tags with 500 options each. Advanced Reporting Tags allow option hierarchies |
| **Location** | A built-in Zoho dimension. Check whether each property needs its own GSTIN or registered address. If so this may fit better **\[verify\]** |
| **Project** | Heavier, built for billable work. Not recommended for this |

The rest of this document assumes one reporting tag named **Property**, with one option per property.

### Endpoints

| Area | Method and path | Purpose |
| --- | --- | --- |
| Organization | `GET /organizations` | Get `organization_id`, currency, fiscal year |
| Contacts | `GET /contacts`, `GET /contacts/{contact_id}` | Read vendors and owners |
|  | `POST /contacts` | Create an owner as a vendor contact |
| Reporting tags | `GET /reportingtags` | List tags and `tag_id` |
|  | `POST /reportingtags` | Create the **Property** tag |
|  | "Update Reporting Tag Options" | Add an option when a property is created. Check the exact path in the OpenAPI file |
| Accounts | `GET /chartofaccounts` | Map accounts to P&L categories |
| Revenue | `GET /invoices`, `GET /invoices/{id}` | Revenue |
|  | `GET /customerpayments`, `GET /creditnotes` | Receipts and adjustments (cash basis, reversals) |
| Costs | `GET /bills`, `GET /bills/{bill_id}` | Vendor bills |
|  | `GET /expenses`, `GET /expenses/{id}` | Expenses |
|  | `GET /vendorcredits` **\[verify path\]** | Vendor credits |
| Create or edit | `POST /bills`, `PUT /bills/{bill_id}` | Create a bill with property tags |
|  | `POST /expenses`, `PUT /expenses/{id}` | Create expenses (for example recurring costs) |
| Bill workflow | `POST /bills/{bill_id}/status/open` | Mark a bill as open |
|  | `POST /bills/{bill_id}/submit` | Submit for Zoho's own approval |
|  | `POST /bills/{bill_id}/approve` | Approve in Zoho |
|  | `POST /bills/{bill_id}/status/void` | Void |
|  | `POST /bills/{bill_id}/attachment` | Attach the vendor invoice |
| Recurring | Recurring expenses and recurring bills endpoints | Native recurring costs. Confirm paths in OpenAPI **\[verify\]** |
| Payouts | `POST /vendorpayments` | Record a payout to an owner |
|  | Journals endpoints (create, publish) | Record the owner liability **\[verify\]** |
| Period lock | Transaction Locking endpoints | Lock a month after its statement is approved **\[verify\]** |
| Tax | `GET /settings/taxes` **\[verify path\]** | GST tax IDs |

Zoho's API navigation has **no general reports section** (P&L, balance sheet). Plan to compute per-property P&L yourself from synced transactions, then reconcile the totals against Zoho's own P&L report in the UI.

### India-specific bill fields (confirmed in the Bills docs)

| Field | Level | Meaning |
| --- | --- | --- |
| `gst_treatment` | bill | `business_gst`, `business_none`, `overseas` or `consumer` |
| `gst_no` | bill | 15 digit GSTIN of the vendor |
| `source_of_supply` | bill | Two-letter state code where goods or services originate (for example `TN`) |
| `destination_of_supply` | bill | Two-letter state code of delivery. Defaults to the organization's home state |
| `is_reverse_charge_applied` | bill | Reverse charge applies |
| `is_tds_applied` | bill | TDS applied |
| `is_pre_gst` | bill | For transactions before 1 July 2017 only |
| `hsn_or_sac` | line item | HSN or SAC code |
| `tax_id` | line item | GST tax or tax group, from the Taxes API |
| `tags` | bill and line item | `[{ "tag_id": "...", "tag_option_id": "..." }]` |

### A6. Worked example: create a bill tagged to a property

```
POST https://www.zohoapis.in/books/v3/bills?organization_id={{organization_id}}
Authorization: Zoho-oauthtoken {{zoho_oauth_token_0afv}}
Content-Type: application/json
```

```json
{
  "vendor_id": "460000000026049",
  "bill_number": "GL-2026-114",
  "date": "2026-09-14",
  "due_date": "2026-09-29",
  "gst_treatment": "business_gst",
  "gst_no": "29ABCDE1234F1Z5",
  "source_of_supply": "KA",
  "destination_of_supply": "KA",
  "line_items": [
    {
      "account_id": "460000000011111",
      "description": "Lawn care, September",
      "rate": 2000,
      "quantity": 1,
      "tax_id": "460000000027005",
      "hsn_or_sac": "998533",
      "tags": [{ "tag_id": "460000000038080", "tag_option_id": "460000000038091" }]
    }
  ]
}
```

Notes:

- Whether `POST /bills` saves a **draft** or an **open** bill by default must be tested on your org **\[verify\]**. A "Mark a bill as open" endpoint exists, which suggests drafts are possible.
- `PUT /bills/{id}` normally replaces `line_items`. Send the full line item list with the tags when you re-tag, or you may drop lines **\[verify\]**.
- Zoho's own approval endpoints (`submit`, `approve`) run as the OAuth user. A user usually cannot approve their own submission, so test this with two real users **\[verify\]**.
- Bank, GSTIN, tax IDs and account IDs in the example are placeholders.

### Webhooks

Zoho Books can call a webhook from workflow rules in its Automation settings, which you configure in the Zoho UI rather than through a public API **\[verify\]**. Plan for polling as the fallback.

---

# Part B: Your layer's API

## B1. Conventions

**Base URL:** `https://api.yourdomain.in/v1`

**Auth to your API:** your own login session, sent as `Authorization: Bearer <your_jwt>`. This is a different system from Zoho. **Zoho tokens never leave your server**, and they use the `Zoho-oauthtoken` header only on calls to `zohoapis.in`.

**Tenant:** one tenant is one Zoho organization (`organization_id`). Your backend adds `organization_id` to every Zoho call, so your own API never exposes it.

**Roles:** `admin`, `manager`, `approver`, `owner` (read-only plus consent).

**Money:** decimal strings in rupees with two decimals, with no currency field because the product is INR only.

```json
{ "amount": "850.00" }
```

**Dates:** `YYYY-MM-DD` in IST for dates, ISO 8601 UTC for timestamps. Financial year follows the organization's fiscal year setting.

**IDs:** prefixed strings: `prop_`, `own_`, `agr_`, `txn_`, `job_`, `stmt_`, `apr_`.

**Pagination:** `?limit=50&cursor=...` returning `{ data, next_cursor, has_more }`.

**Idempotency:** `Idempotency-Key` header on `POST /bills`, `/statements/generate` and `/payouts`.

**Errors**

```json
{ "error": { "code": "owner_consent_required", "message": "...", "details": {}, "request_id": "req_8f2a" } }
```

Statuses: `400` validation, `401` auth, `403` role or rule block, `404`, `409` conflict (period locked), `422` business rule (allocations do not sum), `429` rate limited, `502` Zoho failed or rate limited.

## B2. How your data maps to Zoho

| Your object | Zoho object |
| --- | --- |
| Tenant | Organization (`organization_id`) |
| Property | **Property** reporting tag option (`tag_option_id`) |
| Owner | Vendor contact (`contact_id`) |
| Transaction | Bill, expense, invoice, credit note, vendor credit, kept as a local mirror (`zoho_id`, `zoho_type`) |
| Category | Chart of accounts entry |
| Agreement | Yours only (fee %, reserve, owner spending limit) |
| Statement, approval request, budget | Yours only |

## B3. Endpoints

### Auth and connection

| Method and path | What it does | Calls Zoho |
| --- | --- | --- |
| `POST /auth/login`, `POST /auth/refresh`, `GET /me` | Your own sessions | None |
| `POST /zoho/connect` (admin) | Returns the consent URL | `accounts.zoho.in/oauth/v2/auth` |
| `GET /zoho/callback` | Stores tokens, reads organization | `POST /oauth/v2/token`, `GET /organizations` |
| `GET /zoho/status` | Connected, last sync, scopes granted | None |
| `DELETE /zoho/connection` (admin) | Disconnect and revoke | Token revoke |
| `POST /zoho/sync` | Starts a sync, returns `sync_run_id` (`202`) | Many (see B4) |
| `GET /zoho/sync-runs/{id}` | Status, counts, errors, Zoho calls used | None |

### Properties and owners

| Method and path | What it does | Calls Zoho |
| --- | --- | --- |
| `POST /properties` | Create a property | Adds a **Property** tag option (creates the tag first if missing) |
| `GET /properties`, `GET /properties/{id}`, `PATCH /properties/{id}` | Read, rename | Updates the option name |
| `POST /properties/{id}/archive` | Stop new assignments | Marks the option inactive |
| `POST /owners` | Create an owner | `POST /contacts` (vendor) |
| `GET /owners`, `GET /owners/{id}`, `PATCH /owners/{id}` | Read, edit | `GET/PUT /contacts` |

```json
POST /properties
{
  "name": "Lakeview Hotel",
  "type": "hotel",
  "state_code": "KA",
  "gstin": "29ABCDE1234F1Z5",
  "address": { "line1": "12 Lake Rd", "city": "Bengaluru", "pincode": "560001" }
}
```

`type` is `hotel`, `vacation_rental`, `apartment` or `other`.

### Agreements

| Method and path | What it does |
| --- | --- |
| `POST /properties/{id}/agreements` | Link an owner and set terms |
| `GET /properties/{id}/agreements`, `PATCH /agreements/{id}` | Read, version a change |

```json
{
  "owner_id": "own_01",
  "ownership_share_percent": "100.00",
  "management_fee_percent": "10.00",
  "management_fee_gst_percent": "18.00",
  "reserve_amount": "50000.00",
  "owner_approval_limit": "25000.00",
  "effective_from": "2026-04-01",
  "effective_to": null
}
```

`owner_approval_limit` is the most the operator may spend without asking this owner. Shares across owners of one property must total 100. The GST rate on the management fee is stored per agreement, but confirm the rate and any TDS with your CA before relying on it.

### Transactions and assignment

| Method and path | What it does | Calls Zoho |
| --- | --- | --- |
| `GET /transactions` | Filters: `property_id`, `unassigned=true`, `type`, `from`, `to`, `vendor_id`, `category_id` | None (local mirror) |
| `GET /transactions/unassigned/summary` | Count, total, oldest date | None |
| `PUT /transactions/{id}/assignment` | Assign to one property, or split | Writes the tag back with `PUT /bills/{id}` or `PUT /expenses/{id}` **\[verify\]** |
| `POST /transactions/assignment-rules` | Auto-assign by vendor or memo | None |
| `GET /categories`, `PUT /categories/{id}/mapping` | Map chart of accounts to P&L groups | `GET /chartofaccounts` |

```json
PUT /transactions/txn_77/assignment
{
  "allocations": [
    { "property_id": "prop_01", "category_id": "cat_lawn", "amount": "2000.00" },
    { "property_id": "prop_02", "category_id": "cat_lawn", "amount": "2000.00" }
  ]
}
```

Allocations must sum to the transaction total or the call returns `422`. Splits are held in your database, because a Zoho line item carries one tag option per tag. Split a line in Zoho into several lines if Zoho's own reports must agree **\[verify\]**. An approved statement period returns `409 period_locked`.

### Reports

| Method and path | Returns |
| --- | --- |
| `GET /reports/property-pnl?property_id=&from=&to=&group_by=month` | Revenue, expenses, net profit, `unassigned_warning` |
| `GET /reports/portfolio-pnl?from=&to=&owner_id=` | One row per property plus a portfolio total |
| `GET /reports/budget-vs-actual?property_id=&from=&to=` | Budget, actual, variance, `status` (`on_track`, `watch`, `over_budget`) |
| `PUT /properties/{id}/budgets/{fy}` | Twelve monthly values per category. `GET` on the same path reads it |

All reports are computed from your local mirror. Totals must reconcile with Zoho's P&L for the same dates and accounting basis (cash or accrual). Store the basis on the tenant.

### Recurring expenses

| Method and path | Calls Zoho |
| --- | --- |
| `POST /recurring-expenses` | Zoho recurring expense or bill, tagged to the property |
| `GET /recurring-expenses`, `PATCH`, `DELETE` | Update, stop |
| `GET /recurring-expenses/{id}/occurrences` | Child expenses created |

```json
{
  "property_id": "prop_01",
  "category_id": "cat_maintenance",
  "vendor_id": "ven_20",
  "amount": "8500.00",
  "frequency": "monthly",
  "day_of_month": 1,
  "start_date": "2026-04-01",
  "charge_to": "owner"
}
```

`charge_to` is `owner` (deducted on the statement) or `operator`. Prefer Zoho's native recurring feature so the entry exists in the books, and set the tag on the template **\[verify\]**.

### Owner statements and payouts

| Method and path | What it does | Calls Zoho |
| --- | --- | --- |
| `POST /statements/generate` | Builds a statement from tagged transactions and the agreement | None |
| `GET /statements`, `GET /statements/{id}`, `GET /statements/{id}/pdf` | Read | None |
| `POST /statements/{id}/submit-review`, `/approve`, `/send`, `/void` | `draft`, `in_review`, `approved`, `sent` | Optional period lock |
| `POST /payouts` | Create a payout from an approved statement | Journal for the liability **\[verify\]** |
| `PATCH /payouts/{id}` | `pending`, `scheduled`, `paid` | `POST /vendorpayments` on `paid` |

```json
{
  "id": "stmt_31",
  "status": "draft",
  "property_id": "prop_01",
  "owner_id": "own_01",
  "period": { "from": "2026-09-01", "to": "2026-09-30" },
  "gross_income": "420000.00",
  "operating_expenses": "318000.00",
  "management_fee": "42000.00",
  "management_fee_gst": "7560.00",
  "tds_deducted": "0.00",
  "reserve_held": "50000.00",
  "net_payout": "50440.00",
  "warnings": ["3 unassigned transactions in this period"]
}
```

Generation is blocked with `409` while unassigned transactions exist, unless `allow_unassigned` is true. The layer records payouts. It does not move money.

### Approval matrix

The matrix tests the **job total**, not the single bill, and checks two layers: your internal approver and the owner's delegated limit.

| Method and path | What it does |
| --- | --- |
| `POST /jobs`, `GET /jobs/{id}` | Group related bills. `committed_total` is what the rules test |
| `POST /approval-rules`, `GET`, `PATCH`, `DELETE` | Rules by property, amount range, approver role, `requires_owner_consent` |
| `POST /approvals/evaluate` | Dry run. Returns `auto_approved`, `needs_internal_approval`, `needs_owner_consent` or `blocked` with reasons |
| `POST /bills` | Submit a bill through the matrix |
| `GET /approval-requests` | Pending list |
| `POST /approval-requests/{id}/decision` | Approver decides |
| `POST /approval-requests/{id}/owner-consent` | Owner (or manager, with a mandatory note) records consent |

```json
POST /approvals/evaluate
{ "property_id": "prop_01", "job_id": "job_12", "amount": "15000.00", "category_id": "cat_repairs" }
```

```json
{
  "decision": "needs_owner_consent",
  "job_total_after": "30000.00",
  "reasons": ["Job total ₹30,000 exceeds owner limit ₹25,000", "Repairs budget 82% used"],
  "required_approvers": [{ "role": "approver" }],
  "owner_consent_required": true
}
```

### Webhooks and audit

| Method and path | What it does |
| --- | --- |
| `POST /webhook-endpoints` | Register an outbound endpoint (signed with an `X-Signature` HMAC header) |
| `GET /audit-log` | Assignments, approvals, agreement and statement changes. Append only |

Events: `sync.completed`, `transaction.unassigned`, `approval.requested`, `approval.decided`, `statement.approved`, `statement.sent`, `payout.paid`, `budget.over_threshold`.

## B4. Key flows

### Sync

1. **Backfill** on connect: list bills, expenses, invoices, credit notes, vendor credits, contacts, accounts, tags with `page` and `per_page=200`.
2. **Incremental** after that: filter by modified time where the list endpoint supports it **\[verify\]**, otherwise re-read the current and previous month.
3. Run one queue per tenant with a rate limiter below the per-minute limit, a counter for the daily limit, and back-off on `429`.
4. Every transaction without a property tag goes to the **unassigned queue** in your own database.

### Bill through the approval matrix

1. `POST /bills` with property, job, vendor and the vendor invoice.
2. Evaluate the matrix (property, job total, budget, owner limit).
3. **Hold the bill in your database until all approvals pass.** Create it in Zoho only after that (`POST /bills` on Zoho, tags included), then call `status/open`. This guarantees nothing posts early, whatever Zoho does by default.
4. If you use Zoho's own approval too, call `submit` and `approve` afterwards, using different Zoho users.

## B5. India notes

- **GST:** the layer stores and passes through Zoho tax IDs, GSTIN, `gst_treatment` and supply states. It does not calculate tax. Zoho does.
- **Management fee:** it is a taxable service for the operator. Keep fee and GST as separate lines on the statement.
- **TDS:** whether TDS applies to rent or fee payments to an owner depends on amount and type. Keep `tds_deducted` on the statement and confirm treatment with your CA.
- **Per-property GSTIN:** hotels often register separately by state. If so, test whether **Locations** suits you better than a tag.
- **Dates and year:** IST, and fiscal year from the organization (normally April to March).
- **Trust funds:** do not hold or move owner money in this product until you have advice on the rules that apply.

## B6. Things to test first

1. Does `POST /bills` create a draft or an open bill?
2. Can you edit tags on an existing bill or expense without dropping line items?
3. Can a bill be left unposted until your approval, and does Zoho's `submit` and `approve` allow the same user?
4. Exact paths and scopes for reporting tag options, vendor credits, recurring bills and expenses, journals, transaction locking and taxes.
5. The real per-minute limit on your plan (30 or 100).
6. Whether list endpoints support filtering by last modified time.
7. Whether workflow-rule webhooks give you what you need to avoid polling.