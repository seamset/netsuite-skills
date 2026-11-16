---
name: netsuite-360-view
description: Builds a read-only "360 view" of the user's NetSuite account through Oracle's NetSuite MCP connector (MCP Standard Tools SuiteApp) and nothing else - subsidiaries, record types really used and their volumes, custom records and custom fields, scripts deployed per record, saved searches, who or what creates records, the main business flows (order to invoice to payment), data-quality flags, and what the role cannot see. Use it when the user asks to map, audit, explore, understand, document or "get a 360 view" of their NetSuite instance, or wants context about their NetSuite account before building a script, app, report or integration on it. Requires the NetSuite MCP tools (ns_runCustomSuiteQL and the other ns_ tools) to be connected. Never writes to NetSuite.
---

# NetSuite 360 view (read-only, MCP only)

This skill maps one NetSuite account with the tools of Oracle's NetSuite MCP connector, and
writes one report for the user plus a short context block to reuse before building anything on
that account.

It reads. It never writes, never changes a setting, never downloads a file from the File Cabinet.

## Guardrails (always)

1. **Never call `ns_createRecord` or `ns_updateRecord`.** Not for a test, not "to check
   permissions", not if the user asks in passing. If the user asks for a change, say this skill
   is read-only and stop there.
2. **SELECT only.** Every `ns_runCustomSuiteQL` call is one `SELECT` statement. No `WITH`, no
   `LIMIT`/`TOP`, no `OFFSET`: SuiteQL is Oracle-flavoured (`ROWNUM`, `NVL`, `TO_CHAR`,
   `TO_DATE`, `||`, `SYSDATE`, `ADD_MONTHS`).
3. **Always paginate.** Pass `pageSize: 1000` and `pageIndex: 0` on every query. Without them the
   tool returns pages of 100 rows by default, and a result can look complete while it is not.
   Read `numberOfPages` (or `hasNextPage`) and fetch the next pages only when you need the rows.
   Prefer `COUNT(*)` and `GROUP BY` over listing rows: this report needs counts, not records.
4. **Hard cap.** Stop paginating after 5 pages (5,000 rows) for any one query and say the list
   was cut. A 360 view never needs more.
5. **Small windows on big tables.** Joins on `transactionline` or `nexttransactionlinelink` over
   12 months can time out on a busy account. Start with 90 days, filter the transaction ids with
   an `IN (SELECT ...)` subquery, and widen only if the answer is fast.
6. **Errors are answers.** Never retry a failed query more than once. Record it in the "Not
   visible" list of the report, with the reason:
   - `Record 'x' was not found` = this role has no permission on that table (the table exists).
   - `Invalid search type: x` = this table is not exposed to SuiteQL through the MCP.
   - A timeout = the query is too heavy; narrow the window once, then give up.
7. **Everything is "as seen by this role".** The MCP runs under the user's NetSuite role. A
   transaction type with zero rows may simply be hidden. Never write "the account has no purchase
   orders"; write "no purchase orders visible to this role".
8. **Personal data.** Count fields that hold emails, phones, bank details or tax ids; never copy
   their values into the report. Do not list customer names or amounts unless the user asks for a
   specific record. People's names (who creates records) may appear, since the reader is the
   account's own team; the user can ask to remove them.
9. **No guessing.** Every figure in the report comes from a query run in this session. If a step
   failed, the section says so; it is not filled from general NetSuite knowledge.

## Steps

Run the steps in order. Several queries in one step can be sent in parallel. Keep a running list
of: query, OK / not visible / timeout, rows. It feeds the "Not visible" and "How this was read"
sections.

Tags on each query: **[tested 2026-11-16]** means it ran through the MCP on a NetSuite sandbox
with a read-only role on that date; **[untested]** means it was not run through the MCP yet.
Results on another account depend on its role and its features: a tested query can still be
refused there.

### Step 0. Check the connection and the role

- Run `SELECT COUNT(*) AS n FROM transaction` with `pageSize: 1000, pageIndex: 0`. If the
  `ns_runCustomSuiteQL` tool does not exist, stop and tell the user to connect the NetSuite MCP
  connector (see the README). **[tested 2026-11-16]**
- Ask the user, in one line, whether the account is production or a sandbox. On a sandbox, the
  activity stops at the date of the last refresh: Step 2 finds that date.

### Step 1. Company structure

Use the dedicated tools first: they work even when the role cannot read the tables.

| Read | How | Status |
|---|---|---|
| Subsidiaries | `ns_getSubsidiaries` (negative ids are consolidated views) | [tested 2026-11-16] |
| Accounting books | `ns_getAccountingBooks` | [tested 2026-11-16] |
| Tax nexuses | `ns_getNexusIds` | [tested 2026-11-16] |
| Accounting contexts | `ns_getAccountingContexts` | [untested] |

Then try the tables, which give more detail when the role allows it:

```sql
-- Subsidiaries with country and currency [tested 2026-11-16: refused on a read-only role without Subsidiaries permission]
SELECT s.id, s.name, s.parent, s.country, BUILTIN.DF(s.currency) AS currency, s.iselimination, s.isinactive
FROM subsidiary s ORDER BY s.id

-- Currencies [tested 2026-11-16: refused without Currency permission]
SELECT c.id, c.name, c.symbol, c.isbasecurrency, c.isinactive FROM currency c ORDER BY c.id
```

If the currency table is refused, read the currencies **in use** from transactions instead:

```sql
-- Currencies used, last 12 months [tested 2026-11-16]
SELECT BUILTIN.DF(t.currency) AS currency, t.type, COUNT(*) AS n
FROM transaction t
WHERE t.trandate >= ADD_MONTHS(SYSDATE, -12)
GROUP BY BUILTIN.DF(t.currency), t.type ORDER BY 3 DESC

-- Subsidiaries with activity, last 90 days [tested 2026-11-16]
SELECT BUILTIN.DF(tl.subsidiary) AS subsidiary, COUNT(*) AS docs
FROM transactionline tl
WHERE tl.mainline = 'T'
  AND tl.transaction IN (SELECT t.id FROM transaction t WHERE t.trandate >= SYSDATE - 90)
GROUP BY BUILTIN.DF(tl.subsidiary) ORDER BY 2 DESC
```

Note: the same subsidiary read joined over 12 months timed out on 2026-11-16. Keep 90 days.

### Step 2. Record types really used, with volumes

```sql
-- Transaction types, all time [tested 2026-11-16]
SELECT t.type, COUNT(*) AS n,
       TO_CHAR(MIN(t.trandate), 'YYYY-MM-DD') AS first_date,
       TO_CHAR(MAX(t.trandate), 'YYYY-MM-DD') AS last_date
FROM transaction t GROUP BY t.type ORDER BY 2 DESC

-- Monthly volume per type, last 12 months [tested 2026-11-16]
SELECT t.type, TO_CHAR(t.trandate, 'YYYY-MM') AS month, COUNT(*) AS n
FROM transaction t
WHERE t.trandate >= ADD_MONTHS(SYSDATE, -12) AND t.trandate <= SYSDATE
GROUP BY t.type, TO_CHAR(t.trandate, 'YYYY-MM') ORDER BY 2, 1

-- Status of each type, last 12 months [tested 2026-11-16]
SELECT t.type, t.status, BUILTIN.DF(t.status) AS status_label, COUNT(*) AS n
FROM transaction t
WHERE t.trandate >= ADD_MONTHS(SYSDATE, -12)
GROUP BY t.type, t.status, BUILTIN.DF(t.status) ORDER BY 1, 4 DESC

-- Lines per document, last 12 months (narrow to 90 days if it times out) [tested 2026-11-16]
SELECT t.type, COUNT(DISTINCT t.id) AS docs, COUNT(*) AS lines,
       SUM(CASE WHEN tl.mainline = 'F' AND tl.item IS NOT NULL THEN 1 ELSE 0 END) AS item_lines
FROM transaction t INNER JOIN transactionline tl ON tl.transaction = t.id
WHERE t.trandate >= ADD_MONTHS(SYSDATE, -12)
GROUP BY t.type

-- Entities by type [tested 2026-11-16]
SELECT e.type, COUNT(*) AS n, TO_CHAR(MAX(e.datecreated), 'YYYY-MM-DD') AS last_created
FROM entity e GROUP BY e.type ORDER BY 2 DESC

-- Items by type, active or not [tested 2026-11-16]
SELECT i.itemtype, COUNT(*) AS n, SUM(CASE WHEN i.isinactive = 'F' THEN 1 ELSE 0 END) AS active
FROM item i GROUP BY i.itemtype ORDER BY 2 DESC
```

How to read them:
- **Reference date.** Find the last month with normal volume in the monthly query. On a sandbox
  this is the refresh date; on production it should be the current month. Use that date, not
  today, as the end of "recent" in the report, and say so.
- **Used vs present.** A type is "in use" when it has rows in the 90 days before the reference
  date. Types with old rows only are "present, not used lately".
- `type` codes are NetSuite's internal ones (`SalesOrd`, `CustInvc`, `CustPymt`, `ItemShip`,
  `PurchOrd`, `VendBill`, `Journal`, `CustCred`, `RtnAuth`...). Give the plain name next to each.

### Step 3. Customisation: custom records, lists, fields

```sql
-- Custom record types [tested 2026-11-16]
SELECT c.internalid, c.scriptid, c.name, c.isinactive, c.lastmodifieddate
FROM customrecordtype c ORDER BY c.scriptid

-- Custom lists [tested 2026-11-16]
SELECT c.internalid, c.scriptid, c.name, c.isinactive FROM customlist c ORDER BY c.scriptid

-- Rows in one custom record type (one query per type; table name = scriptid in lower case) [tested 2026-11-16]
SELECT COUNT(*) AS n, TO_CHAR(MAX(r.lastmodified), 'YYYY-MM-DD') AS last_modified
FROM customrecord_example r
```

- Run the row count only for active custom record types whose `scriptid` does not look like an
  add-on (add-ons usually carry a vendor prefix or a number, for example `customrecord_2663_...`,
  `customrecord_taf_...`). Cap at 30 types; say how many were skipped.
- A custom record with rows modified in the last year is "in use".
- Separate the account's own customisation from add-ons (SuiteApps, bundles): own scripts have
  an empty `frombundle` (Step 4); own custom records usually have a short or company-specific
  prefix. Say it is an inference.

**Custom fields.** The custom field tables are usually not readable through the MCP:

```sql
-- [tested 2026-11-16: refused, "Record 'customfield' was not found"]
SELECT c.scriptid, c.name, c.fieldtype, BUILTIN.DF(c.recordtype) AS recordtype FROM customfield c
-- [tested 2026-11-16: "Invalid search type"] transactionbodycustomfield, entitycustomfield, customrecordcustomfield
```

Read them from the metadata instead:
1. Call `ns_getSuiteQLMetadata` with `recordType` set to each record in use: `customer`,
   `transaction`, `item`, `vendor`, `employee` as visible. **[tested 2026-11-16 on customer]**
   The answer is long: keep only the property names that start with `custentity`, `custbody`,
   `custcol`, `custitem`, `custrecord` or `custevent`, with their `title` and `type`, and
   `x-n:recordType` when present (the list or record the field points to).
2. Measure how much each one is filled, ten fields per query:

```sql
-- Fill rate of custom fields on active customers [tested 2026-11-16]
SELECT COUNT(*) AS total,
       COUNT(c.custentity_example_a) AS example_a_filled,
       COUNT(c.custentity_example_b) AS example_b_filled,
       SUM(CASE WHEN c.custentity_example_flag = 'T' THEN 1 ELSE 0 END) AS example_flag_true
FROM customer c WHERE c.isinactive = 'F'

-- Same on transactions, one type, recent window [untested]
SELECT COUNT(*) AS total, COUNT(t.custbody_example_a) AS example_a_filled
FROM transaction t
WHERE t.type = 'SalesOrd' AND t.trandate >= SYSDATE - 90
```

Report: per record, the number of custom fields, how many are filled on more than 50% of rows,
and how many are empty (candidates for clean-up). Booleans are never null: count `= 'T'`.

### Step 4. Scripts and automation

```sql
-- Scripts by type, active or not [tested 2026-11-16]
SELECT s.scripttype, s.isinactive, COUNT(*) AS n
FROM script s GROUP BY s.scripttype, s.isinactive ORDER BY 3 DESC

-- Own code vs add-ons (empty frombundle = written for this account or deployed by SDF) [tested 2026-11-16]
SELECT s.frombundle, COUNT(*) AS n FROM script s GROUP BY s.frombundle ORDER BY 2 DESC

-- Deployments per record [tested 2026-11-16]
SELECT d.recordtype, d.status, d.isdeployed, COUNT(*) AS n
FROM scriptdeployment d GROUP BY d.recordtype, d.status, d.isdeployed ORDER BY 4 DESC

-- What runs when one record is saved (one query per key record: SALESORDER, INVOICE, CUSTOMER...) [tested 2026-11-16]
SELECT s.scriptid, s.name, s.scripttype, s.frombundle, d.status,
       s.beforesubmitfunction, s.aftersubmitfunction
FROM scriptdeployment d INNER JOIN script s ON s.id = d.script
WHERE d.isdeployed = 'T' AND s.isinactive = 'F' AND d.recordtype = 'SALESORDER'
ORDER BY s.scripttype, s.name

-- Script log volume by level [tested 2026-11-16]
SELECT n.type, COUNT(*) AS n FROM scriptnote n GROUP BY n.type

-- Scripts that log errors [tested 2026-11-16]
SELECT s.name, s.scripttype AS script_type, COUNT(*) AS errors,
       TO_CHAR(MAX(n.date), 'YYYY-MM-DD') AS last_error
FROM scriptnote n LEFT JOIN script s ON s.id = n.scripttype
WHERE n.type = 'ERROR'
GROUP BY s.name, s.scripttype ORDER BY 3 DESC

-- Workflows [tested 2026-11-16: refused on a read-only role; may work with Workflow permission]
SELECT w.recordtype, w.releasestatus, w.isinactive, COUNT(*) AS n
FROM workflow w GROUP BY w.recordtype, w.releasestatus, w.isinactive ORDER BY 4 DESC
```

How to read them:
- `USEREVENT` scripts with a `beforesubmitfunction` or `aftersubmitfunction` change data when a
  record is saved: anything built on that record must expect them. `CLIENT` scripts only run in
  the browser form, not on API or CSV writes. `SCHEDULED` and `MAPREDUCE` run in batches.
- Scripts named after the MCP tools themselves (custom tools) may log errors from this very
  session; say so rather than flag them as a problem.
- The script **code** is not readable through the MCP: the report states what runs where, not
  what it does. Names and function names are clues, not proof.

### Step 5. Saved searches and reports

- `ns_listSavedSearches` with no query lists all searches visible to the role; with `query` it
  filters by title. **[tested 2026-11-16 with a query]** On a big account, prefer the count:

```sql
-- Saved searches by record type [tested 2026-11-16]
SELECT ss.searchtype, COUNT(*) AS n FROM savedsearch ss GROUP BY ss.searchtype ORDER BY 2 DESC
```

- `ns_listAllReports` lists the standard reports the role can run. **[tested 2026-11-16: returned
  an empty list for a read-only role without report permissions]** If empty, say the role has no
  report access.
- Do not run saved searches or reports in this skill (`ns_runSavedSearch`, `ns_runReport`
  **[untested]**): they can be slow and return rows, not structure. Mention the 10 searches whose
  titles look most business-critical (invoicing, collections, stock, commissions) as "worth
  knowing before building".

### Step 6. Who and what creates records

```sql
-- Who created transactions, last 12 months [tested 2026-11-16]
SELECT t.type, BUILTIN.DF(t.createdby) AS created_by, COUNT(*) AS n
FROM transaction t
WHERE t.createddate >= ADD_MONTHS(SYSDATE, -12)
GROUP BY t.type, BUILTIN.DF(t.createdby) ORDER BY 3 DESC

-- Through which channel, last 90 days (UI, CSV, web services, scripts...) [tested 2026-11-16]
SELECT t.type, BUILTIN.DF(t.source) AS source, COUNT(*) AS n
FROM transaction t
WHERE t.createddate >= SYSDATE - 90
GROUP BY t.type, BUILTIN.DF(t.source) ORDER BY 3 DESC

-- Changes by record type and context, last 90 days [tested 2026-11-16: readable, but only partly visible to a read-only role]
SELECT BUILTIN.DF(sn.recordtypeid) AS record_type, BUILTIN.DF(sn.context) AS context,
       COUNT(*) AS changes, COUNT(DISTINCT sn.name) AS users
FROM systemnote sn
WHERE sn.date >= SYSDATE - 90
GROUP BY BUILTIN.DF(sn.recordtypeid), BUILTIN.DF(sn.context) ORDER BY 3 DESC
```

How to read them:
- An empty `created_by` or `source` is common (imports, scripts, older records): report the
  share, do not guess.
- The system notes the role sees can be a small part of the real history. If the record types
  in use (Step 2) are missing from the system notes, say "the change history of transactions is
  not visible to this role".
- Login history (`loginaudit`), deleted records (`deletedrecord`) and transaction history
  (`transactionhistory`) were refused for a read-only role on 2026-11-16: list them as not
  visible unless they answer.

### Step 7. Key business flows

```sql
-- Document chains, last 90 days (which document leads to which) [tested 2026-11-16]
SELECT pt.type AS from_type, nt.type AS to_type, l.linktype, COUNT(*) AS pairs
FROM nexttransactionlink l
INNER JOIN transaction pt ON pt.id = l.previousdoc
INNER JOIN transaction nt ON nt.id = l.nextdoc
WHERE nt.trandate >= SYSDATE - 90
GROUP BY pt.type, nt.type, l.linktype ORDER BY 4 DESC

-- Line-level chains, last 12 months (includes types the role cannot open, e.g. fulfilments) [tested 2026-11-16]
SELECT l.previoustype, l.nexttype, l.linktype, COUNT(*) AS line_pairs
FROM nexttransactionlinelink l
WHERE l.nextdate >= ADD_MONTHS(SYSDATE, -12)
GROUP BY l.previoustype, l.nexttype, l.linktype ORDER BY 4 DESC

-- Orders with no next document yet, last 90 days [tested 2026-11-16]
SELECT COUNT(*) AS orders,
       SUM(CASE WHEN EXISTS (SELECT 1 FROM nexttransactionlink l WHERE l.previousdoc = so.id)
                THEN 0 ELSE 1 END) AS without_next
FROM transaction so
WHERE so.type = 'SalesOrd' AND so.trandate >= SYSDATE - 90 AND so.trandate <= SYSDATE
```

Draw the main chains as arrows with volumes, for example
`Sales order -> Item fulfilment -> Invoice -> Customer payment`, and the side branches (returns,
credit memos). If a step of a flow is not visible to the role but appears in the line links,
say "seen only through links". For a purchasing flow, repeat with `PurchOrd` if visible.

### Step 8. Data-quality flags

Run what applies to the record types in use. Report counts and shares, never record lists.

```sql
-- Customers: missing email, inactive, no terms, duplicates [tested 2026-11-16]
SELECT COUNT(*) AS customers,
       SUM(CASE WHEN c.email IS NULL THEN 1 ELSE 0 END) AS no_email,
       SUM(CASE WHEN c.isinactive = 'T' THEN 1 ELSE 0 END) AS inactive,
       SUM(CASE WHEN c.terms IS NULL THEN 1 ELSE 0 END) AS no_terms
FROM customer c

-- Active customers sharing the same company name [tested 2026-11-16]
SELECT COUNT(*) AS duplicate_groups FROM (
  SELECT UPPER(c.companyname) AS nm, COUNT(*) AS k FROM customer c
  WHERE c.companyname IS NOT NULL AND c.isinactive = 'F'
  GROUP BY UPPER(c.companyname) HAVING COUNT(*) > 1)

-- Transactions dated far in the future, without entity [tested 2026-11-16]
SELECT t.type,
       SUM(CASE WHEN t.trandate > SYSDATE + 30 THEN 1 ELSE 0 END) AS future_dated,
       SUM(CASE WHEN t.entity IS NULL THEN 1 ELSE 0 END) AS no_entity
FROM transaction t GROUP BY t.type

-- Open invoices by age of due date [tested 2026-11-16]
SELECT CASE WHEN t.duedate >= SYSDATE THEN 'not due'
            WHEN t.duedate >= SYSDATE - 30 THEN '1-30'
            WHEN t.duedate >= SYSDATE - 90 THEN '31-90' ELSE '90+' END AS bucket,
       COUNT(*) AS invoices
FROM transaction t WHERE t.type = 'CustInvc' AND t.status = 'A'
GROUP BY CASE WHEN t.duedate >= SYSDATE THEN 'not due'
              WHEN t.duedate >= SYSDATE - 30 THEN '1-30'
              WHEN t.duedate >= SYSDATE - 90 THEN '31-90' ELSE '90+' END

-- Items: inactive share, no sales in 12 months [untested]
SELECT COUNT(*) AS active_items,
       SUM(CASE WHEN NOT EXISTS (SELECT 1 FROM transactionline tl
            INNER JOIN transaction t ON t.id = tl.transaction
            WHERE tl.item = i.id AND t.trandate >= ADD_MONTHS(SYSDATE, -12)) THEN 1 ELSE 0 END) AS not_sold_12m
FROM item i WHERE i.isinactive = 'F'
```

Cautions to write next to the flags:
- A huge number of "open" invoices over 90 days can mean real overdue debt, or payments the role
  cannot see, or a sandbox whose payments were never copied. Say which is likely; do not conclude.
- Future-dated documents can be planned orders (subscriptions, standing orders) rather than errors.
- The item query above may be slow: run it last, and drop it if it times out.

### Step 9. Write the report

Write the report in the user's language, with the template below. Then offer, in one line, to
turn it into a formatted document or PDF (headings, tables, one diagram for the flows). If the
user accepts, produce it with the document or PDF capability available in this assistant
environment; keep the same sections.

## Report template

```markdown
# NetSuite 360 view: <account name or "your account"> (<production | sandbox>)

Read on <date> through the NetSuite MCP connector, role "<role name if known>", read-only.
Reference date for "recent": <date> (<why: today on production, last refresh on a sandbox>).
Everything below is what this role can see.

## 1. At a glance
- <N> subsidiaries (<names>), base currency <x>, <N> currencies in use.
- Main activity: <type> (<n>/month), <type> (<n>/month), ...
- <N> customers active, <N> items active, <N> custom record types (<n> in use).
- <N> active scripts (<n> own, <n> from add-ons); <n> run when a <key record> is saved.
- Top 3 points of attention: <one line each>.

## 2. Company structure
Table: subsidiary, country/currency if visible, documents in the last 90 days.
Accounting books, nexuses.

## 3. What the account really uses
Table: record type (plain name + code), total, last 12 months, per month on average, last date,
status: in use / present, not used lately.
Entities by type. Items by type (active / total).

## 4. Customisation
- Custom record types: total, active, in use (rows changed within a year), own vs add-on.
  Table of the 10 own custom records in use, with row count and last change.
- Custom lists: total, own vs add-on.
- Custom fields per record: total, filled > 50%, empty. Table of the most filled ones.

## 5. Automation
- Scripts by type, own vs add-on. Script errors (top 5 scripts, last error date).
- Per key record (sales order, invoice, customer...): the scripts that run on save, with
  before/after submit.
- Workflows: <counts, or "not visible to this role">.

## 6. Saved searches and reports
Counts by record type; the searches worth knowing; report access.

## 7. Who and what creates records
Per transaction type: share created by people (top creators), by system/scripts, by web services
or integrations, unknown. Change history visibility.

## 8. Key business flows
Diagram of the main chains with volumes. Orders without a next document. Flows seen only
through links.

## 9. Data-quality flags
Table: flag, count, share, what it may mean, who should look.

## 10. Not visible with this role
Table: what, why (permission missing / not exposed by the MCP / timeout), which permission or
tool would show it.

## 11. How this was read
Number of queries, number refused, windows used, tools used. No write was made.

## 12. Context to reuse before building
(the block below)
```

## The context block (section 12)

Keep it under 40 lines, plain text, so the user can paste it at the top of any future request
("build me a script / an app / a report on this account"). Facts only, from this session:

```text
NETSUITE CONTEXT (<account>, <prod|sandbox>, read <date>, role <role>, read-only)
- Subsidiaries: <list>. Base currency: <x>. Currencies used: <list>.
- Record types in use (90 days): <type code = plain name (n/month)>, ...
- Main flow: <SalesOrd -> ItemShip -> CustInvc -> CustPymt>, <other flows>.
- Custom records in use: <scriptid (rows)>, ...
- Key custom fields: <record: field id = label (fill %)>, ...
- Code that runs on save: <record: n user event scripts, own: <names>>, ...
- Creation channels: <type: UI x%, web services y%, scripts z%>.
- Known data issues: <flags>.
- Not visible to this role: <list>.
- SuiteQL notes for this account: <for example "join transactionline over 12 months times out;
  use 90 days", "BUILTIN.DF works", "status codes: SalesOrd F = Pending Billing">.
Rule for any build: read first, propose before writing, test in a sandbox, never write with this
read-only connection.
```

## What this skill cannot see with a read-only MCP, and why

Say this plainly in section 10, adapted to what actually failed:

- **Tables the role has no permission on** (often subsidiary, currency, account, employee,
  workflow, login audit, deleted records): the MCP runs as the user's role, and a read-only role
  rarely holds Setup permissions. The dedicated tools (`ns_getSubsidiaries`...) cover part of it.
- **Custom field definitions** (`customfield` and its variants): not exposed to SuiteQL through
  the MCP. The metadata tool gives the field names and labels, not who created them, nor their
  sourcing, defaults or formulas.
- **Script code, workflow states and actions, saved-search criteria, form layouts**: these live
  in the File Cabinet or in the account's configuration (SDF), which the MCP does not read.
  The report shows what is deployed where, not what it does.
- **Full change history**: system notes are only partly visible to a non-admin role, so "who
  writes each field" cannot be established reliably.
- **Enabled features and installed SuiteApps by name**: the bundle and feature tables are not
  readable by a read-only role; add-ons appear only as bundle numbers on scripts.
- **Anything the role cannot open** (for example purchases or payments when the role only has
  sales permissions): counts are therefore lower bounds.
- **Row limits**: SuiteQL through the MCP returns at most a few thousand rows per query and
  heavy joins time out, so this skill works with counts and windows, not full extracts.
