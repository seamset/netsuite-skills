---
name: describe-your-process
description: Reverse-engineer how a business process really runs in NetSuite from a recorded walkthrough. Use when the user has a transcript (with a date and timestamps) of someone running a real process live in NetSuite (Fireflies, Zoom, Teams, Google Meet or similar) and wants a documented description of that process, checked against what NetSuite actually recorded in the same time window. Reads NetSuite only through Oracle's NetSuite MCP connector (MCP Standard Tools), read-only. Produces a structured report of steps, records, fields, scripts, workflows and risks, separating what NetSuite proves from what was only said or inferred. Do not use without a NetSuite MCP connection or without a timestamped transcript.
---

# Describe your current process

Someone ran one of their real processes in NetSuite while recording a call and saying out loud
what they were doing. This skill turns that recording into a precise description of the
process: it reads what the person said, step by step, then looks in NetSuite at what really
happened in the same minutes, by the same person, and matches the two.

The result is a report that a finance lead, a NetSuite admin or an integrator can read and
challenge: steps in order, records and fields touched, the technical building blocks behind
each step (forms, custom fields, scripts, workflows, approvals, integrations), what triggers
what, and what would break if the process were automated.

## Ground rules (read before any step)

1. **Read-only, always.** Use only these MCP tools: `ns_runCustomSuiteQL`,
   `ns_getSuiteQLMetadata`, `ns_getRecordTypeMetadata`, `ns_getRecord`, `ns_listSavedSearches`,
   `ns_runSavedSearch`, `ns_listAllReports`, `ns_runReport`, `ns_report_filters_app`,
   `ns_getSubsidiaries`, `ns_getAccountingBooks`, `ns_getAccountingContexts`, `ns_getNexusIds`,
   `ns_selector_app`. **Never call `ns_createRecord`, `ns_updateRecord` or any other tool that
   creates, changes or deletes data**, even if the user asks, even "just to test". If a check
   seems to need a write, say so in the report as an open question.
2. **SuiteQL is SELECT only.** One statement, no `WITH`, no `LIMIT`/`TOP` (use `ROWNUM` or the
   tool's pagination), dates through `TO_TIMESTAMP(...)`, labels through `BUILTIN.DF(...)`.
3. **Paginate.** The tool returns at most about 5,000 rows per query. For any query on
   `transaction`, `transactionline`, `systemnote` or `scriptnote`, pass `pageSize` (500 to 1000)
   and `pageIndex` 0, read `numberOfPages`, and fetch the remaining pages. Never assume a first
   page is the whole answer.
4. **Keep the window tight.** Every query on activity tables filters on the time window and,
   where possible, on the user. Never scan a whole table.
5. **Privacy.** Transcripts and NetSuite data contain personal and business data (names,
   customers, amounts, emails). Quote the transcript only as much as a step needs (one short
   sentence at most), never paste it whole into the report. Show customer names, amounts and
   record numbers only when they explain a step; otherwise write "customer A", "invoice 1".
   Do not send transcript or NetSuite content to any other tool or service than the ones the
   user asked for. Note that every MCP query is itself logged by NetSuite in the account's script
   execution logs.
6. **Never invent.** Every statement in the report carries one of three marks (see below). When
   NetSuite does not show something, say "not visible with this role", not "did not happen".

## Evidence marks

Use them on every step, field and building block in the report:

- **[NS]** Evidenced in NetSuite: a row returned by a query, with the record and time.
- **[SAID]** Only said in the transcript: no matching NetSuite trace found or readable.
- **[INFERRED]** Deduced: for example a script deployed on the record type that probably fired,
  or a field change implied by a status change. Say what it is inferred from.

## Step 1. Get the transcript

Accept the transcript in any of these forms:

- pasted in the conversation;
- an attached file (txt, docx, pdf, vtt, srt, json export);
- fetched from a meeting-notes connector the user already has (Fireflies, Zoom, Teams, Google
  Meet, Otter...). Ask which meeting if more than one fits; fetch only that one.

Extract and show the user, before going further:

| Item | How to find it |
|---|---|
| Date | Meeting metadata, file name, or ask |
| Start time and time zone of the recording | Metadata first. If only "10:02" is given, ask for the time zone. Never guess it |
| End time | Start time plus the last timestamp, or metadata |
| Who ran the process in NetSuite | The speaker who says "I click", "I open", "I save". Ask for the full name as it appears in NetSuite |
| Steps said | One line per action: timestamp, what they say they do, record named (type, number, customer), fields named, buttons clicked, "then normally..." remarks |

Transcript timestamps are often relative ("00:12:40" from the start). Convert each to an
absolute clock time: start time plus offset.

Ask the user to confirm the date, start time, time zone and the person's name. A wrong time zone
makes every later query miss.

## Step 2. Calibrate time with NetSuite

NetSuite returns dates in two different clocks. Run this first:

```sql
SELECT TO_CHAR(CURRENT_DATE, 'YYYY-MM-DD HH24:MI:SS') AS session_now,
       TO_CHAR(SYSDATE, 'YYYY-MM-DD HH24:MI:SS')      AS server_now,
       SESSIONTIMEZONE                                AS session_tz
FROM DUAL
```

- `session_tz` is the time zone of the NetSuite user behind the MCP connection (their
  preference). Record dates (`transaction.createddate`, `lastmodifieddate`,
  `entity.datecreated`, `systemnote.date`) are shown and filtered in this time zone.
- `server_now` is the data center clock. **Script execution logs (`scriptnote.date`) are in
  this server clock, not in the session time zone.** Compute the offset
  `session_now - server_now` (for example 9 hours between Europe/Paris and US Pacific in
  winter) and apply it to every script log time.

Then convert the recording window into the session time zone.

## Step 3. Set the time window and the margins

- **Process window** = recording start minus 5 minutes to recording end plus 15 minutes, in the
  session time zone. The 5 minutes cover clock drift between the recorder and NetSuite and
  actions taken just before recording; the 15 minutes catch scripts, workflows and integrations
  that react after a save.
- **Step match** = a NetSuite event matches a spoken step when it falls between the step's
  timestamp and the next step's timestamp, plus or minus 2 minutes. People often speak before or
  after they click: when an event sits between two steps, attach it to the step whose words
  name the record or field.
- **Delayed effects**: scheduled scripts, map/reduce jobs, scheduled workflows and approvals by
  someone else can land hours later. If the person says "this goes to approval" or "it runs
  tonight", widen to the end of the day (or the next morning) for those record ids only.
- If nothing is found in the window at all, first suspect the time zone (try the window shifted
  by the difference between the recorder's zone and the session zone, and by one hour for
  daylight saving), then the user, then the environment (sandbox versus production).

## Step 4. Identify the user in NetSuite

The person's NetSuite user id is needed to separate their actions from other users and from
scripts. The employee table may not be readable with a read-only role, so find the user through
what they created or changed in the window:

```sql
SELECT t.createdby, BUILTIN.DF(t.createdby) AS user_name, t.type, COUNT(*) AS n
FROM transaction t
WHERE t.createddate >= TO_TIMESTAMP('2026-01-15 09:55:00', 'YYYY-MM-DD HH24:MI:SS')
  AND t.createddate <  TO_TIMESTAMP('2026-01-15 10:45:00', 'YYYY-MM-DD HH24:MI:SS')
GROUP BY t.createdby, BUILTIN.DF(t.createdby), t.type
```

Repeat with `lastmodifiedby` and `lastmodifieddate`, and with `systemnote.name` (Step 5). Pick
the id whose name matches the person. Keep other users' ids: they matter when someone else
approves or edits during the window. Ids below zero (for example -4) are system or script
users, not people.

## Step 5. Collect the NetSuite evidence

Run these in order. Replace the timestamps with the process window and `:user` with the user
id. Paginate every activity query.

### 5.1 Transactions created or changed in the window

```sql
SELECT t.id, t.type, t.tranid,
       TO_CHAR(t.createddate, 'YYYY-MM-DD HH24:MI:SS')      AS created,
       TO_CHAR(t.lastmodifieddate, 'YYYY-MM-DD HH24:MI:SS') AS modified,
       BUILTIN.DF(t.createdby)      AS created_by,
       BUILTIN.DF(t.lastmodifiedby) AS modified_by,
       BUILTIN.DF(t.entity)         AS entity,
       BUILTIN.DF(t.customform)     AS form,
       BUILTIN.DF(t.status)         AS status,
       BUILTIN.DF(t.approvalstatus) AS approval_status
FROM transaction t
WHERE (t.createddate >= TO_TIMESTAMP(:start, 'YYYY-MM-DD HH24:MI:SS')
       AND t.createddate < TO_TIMESTAMP(:end, 'YYYY-MM-DD HH24:MI:SS'))
   OR (t.lastmodifieddate >= TO_TIMESTAMP(:start, 'YYYY-MM-DD HH24:MI:SS')
       AND t.lastmodifieddate < TO_TIMESTAMP(:end, 'YYYY-MM-DD HH24:MI:SS'))
ORDER BY t.createddate
```

Notes: `lastmodifieddate` only keeps the latest change. A record edited in the window and again
later will not show up here: system notes (5.3) are the way to catch it, when readable.

### 5.2 Entities (customers, vendors, contacts...) created or changed

```sql
SELECT e.id, e.entityid, e.type,
       TO_CHAR(e.datecreated, 'YYYY-MM-DD HH24:MI:SS')      AS created,
       TO_CHAR(e.lastmodifieddate, 'YYYY-MM-DD HH24:MI:SS') AS modified
FROM entity e
WHERE e.lastmodifieddate >= TO_TIMESTAMP(:start, 'YYYY-MM-DD HH24:MI:SS')
  AND e.lastmodifieddate <  TO_TIMESTAMP(:end, 'YYYY-MM-DD HH24:MI:SS')
```

Do the same on `item` if the person touched items, and on custom records (5.6).

### 5.3 System notes: field-level history (when readable)

```sql
SELECT TO_CHAR(sn.date, 'YYYY-MM-DD HH24:MI:SS') AS at,
       BUILTIN.DF(sn.name)         AS user_name,
       BUILTIN.DF(sn.role)         AS role,
       BUILTIN.DF(sn.recordtypeid) AS record_type,
       sn.recordid, sn.record, sn.field, sn.oldvalue, sn.newvalue,
       sn.type, sn.context, BUILTIN.DF(sn.context) AS context_label
FROM systemnote sn
WHERE sn.date >= TO_TIMESTAMP(:start, 'YYYY-MM-DD HH24:MI:SS')
  AND sn.date <  TO_TIMESTAMP(:end, 'YYYY-MM-DD HH24:MI:SS')
ORDER BY sn.date
```

- Do not wrap `sn.field` or `sn.type` in `BUILTIN.DF`: NetSuite rejects it ("Static field is
  not supported"). `sn.field` is a code such as `ENTITY.SNAME` or a custom field id such as
  `CUSTBODY_...`; translate it with `ns_getRecordTypeMetadata` or `ns_getSuiteQLMetadata` on the
  record type.
- `sn.type`: 1 appears on the creation of a record, 2 on field changes (observed, not
  documented by Oracle).
- `sn.context` tells who made the change: `UIF` user interface, `CSV` import, `RWS`/`WSS` web
  services (integration), `UES` user event script, `SCH` scheduled script, `MPR` map/reduce,
  `WFW` workflow, `WFA` workflow action script, `SWF` scheduled workflow. The full list:
  `SELECT * FROM systemnotecontexttype`. A change in the window with a script or workflow
  context, on a record the person just saved, is strong evidence of automation.
- **Coverage depends on the role.** On some accounts the read-only role sees system notes for
  some record types (entities, roles) and none for transactions. If the query returns nothing
  for a transaction you know was saved, write in the report: "field history not readable with
  this role" and rely on 5.1, 5.4 and the transcript.

### 5.4 Script execution logs in the window (what actually fired)

Convert the window to the server clock first (Step 2).

```sql
SELECT TO_CHAR(n.date, 'YYYY-MM-DD HH24:MI:SS') AS at_server_time,
       s.scriptid, s.name AS script_name, s.scripttype,
       n.type AS log_level, n.title, n.detail
FROM scriptnote n
JOIN script s ON s.id = n.scripttype
WHERE n.date >= TO_TIMESTAMP(:start_server, 'YYYY-MM-DD HH24:MI:SS')
  AND n.date <  TO_TIMESTAMP(:end_server, 'YYYY-MM-DD HH24:MI:SS')
ORDER BY n.date
```

- `scriptnote.scripttype` is the script's internal id (join to `script.id`).
- A script only logs what its developer chose to log, at the deployment's log level. No log
  does not mean the script did not run.
- Ignore the MCP connector's own scripts (names such as "SuiteQL Tools" or "Record Tools"):
  they log your own queries.
- Logs are kept for a limited period (about 30 days). Run the skill soon after the recording.
- Keep `n.detail` out of the report unless it explains a step: it can hold record data.

### 5.5 Scripts that could fire on the record types touched

For each record type found in 5.1 to 5.3 (use the NetSuite record id in upper case, for example
`SALESORDER`, `INVOICE`, `VENDORBILL`, `CUSTOMER`, `CUSTOMRECORD_...`):

```sql
SELECT s.scriptid, s.name, s.scripttype,
       d.scriptid AS deployment_id, d.status, d.loglevel,
       BUILTIN.DF(d.recordtype) AS record_type
FROM scriptdeployment d
JOIN script s ON s.id = d.script
WHERE d.isdeployed = 'T'
  AND d.recordtype = 'SALESORDER'
```

- Do not select `d.executioncontext`: it is a long list and adds nothing to the report.
- `scripttype`: `USEREVENT` runs on load/save on the server, `CLIENT` runs in the browser while
  the user types, `ACTION` is a workflow action script (a sign a workflow exists on that record),
  `SCHEDULED` and `MAPREDUCE` run in the background, `SUITELET` and `RESTLET` are custom pages
  and endpoints, often integrations.
- A deployed script is **[INFERRED]** until a log (5.4) or a system note with a script context
  (5.3) shows it ran in the window, then it becomes **[NS]**.

### 5.6 Workflows, custom records, forms, saved searches

- **Workflows** are not exposed as a SuiteQL table through the MCP. Evidence of workflows comes
  from system notes with context `WFW`, `WFA` or `SWF`, from `ACTION` scripts deployed on the
  record type, and from an `approvalstatus` that changes. List them as [INFERRED] with that
  evidence, and add an open question for the admin ("which workflows run on this record?").
- **Custom records**: list the types with
  `SELECT crt.scriptid, crt.name FROM customrecordtype crt`. Each custom record table (for
  example `customrecord_xyz`) is readable only if the role has permission on that type; "Record
  not found" means "not readable with this role", not "absent". When readable, filter on
  `created` / `lastmodified`.
- **Forms**: `BUILTIN.DF(t.customform)` in 5.1 names the form used. A custom form is a building
  block: it decides which fields the user sees.
- **Fields on a record**: `ns_getRecord` with `fields` as a comma-separated list **without
  spaces** (`customform,orderstatus,createdfrom`); with spaces, it returns nothing. Use
  `ns_getRecordTypeMetadata` for field labels and custom fields (`custbody_`, `custcol_`,
  `custentity_`).
- **Transaction chains** (quote to order to invoice, credit memo from invoice):
  ```sql
  SELECT BUILTIN.DF(l.previousdoc) AS previous_doc, BUILTIN.DF(l.nextdoc) AS next_doc, l.linktype
  FROM previoustransactionlink l
  WHERE l.previousdoc = :id OR l.nextdoc = :id
  ```
- **Lines**: `SELECT tl.linesequencenumber, tl.mainline, BUILTIN.DF(tl.item) AS item,
  BUILTIN.DF(tl.createdfrom) AS created_from FROM transactionline tl WHERE tl.transaction = :id`
- **Saved searches and reports** the person opened: `ns_listSavedSearches` and
  `ns_listAllReports` to find them by name, `ns_runSavedSearch` only if its result helps describe
  the step (it may return many rows: paginate, do not paste them).

## Step 6. Match spoken steps with evidence

Build one table, in time order, merging both sources:

| # | Time (session tz) | Said in the transcript | NetSuite evidence | Mark |
|---|---|---|---|---|

Rules:

- One row per spoken action. Add rows for NetSuite events with no spoken step (often scripts,
  workflows or integrations), marked "not mentioned".
- A spoken step with no NetSuite trace stays [SAID]. Check first whether it is a read-only action
  (opening a search, looking at a report, checking a screen): those leave no trace, which is
  normal.
- When the person says "normally I would also...", record it as a variant, not as a step done.
- Flag every contradiction (said "I approve it", but the approval status did not change; said
  "the invoice is created automatically", but it was created by the user in the UI).

## Step 7. Write the report

Use this template. Write in the user's language. Keep it factual and short per item; the detail
lives in the tables.

```markdown
# [Process name]: how it runs today

Recorded on [date], [start]-[end] ([time zone]), run by [role, not necessarily the name],
NetSuite account [sandbox / production]. Report produced on [date].

## Summary
Five lines: what the process does, how many steps, how many records touched, the main
automation found, the main risk.

## Steps
For each step:
### Step N. [Verb + object]  [NS] / [SAID] / [INFERRED]
- Who: [role] (user / script / workflow / integration / other user)
- Where in NetSuite: [record type, form, screen or saved search]
- Records: [type and number, anonymised if not needed]
- Fields set or changed: [field label (id): old -> new]
- What fires: [scripts, workflows, emails, approvals, with evidence]
- Manual effort / judgment: [what the person decides or types by hand]
- Said: "[one short quote]" (mm:ss)

## Records and fields touched
Table: record type | count | created / changed | fields changed | by whom.

## Technical building blocks
Table: type (form, custom field, custom record, user event script, client script, workflow,
saved search, approval, integration, report) | name and id | where it acts | evidence mark |
what it does (known or inferred).

## What triggers what
Ordered chains, e.g. "Save sales order (user) -> user event X sets field Y [NS] -> workflow
action Z [INFERRED] -> invoice created next morning by scheduled script [SAID]".

## Implications
- Manual steps and re-typing.
- Risks: errors, missing controls, single person dependency, timing.
- What would break or need care if this process were automated or rebuilt (scripts that expect
  the UI context, client scripts that do not run through APIs or imports, approvals, numbering,
  period locks, integrations listening to the same records).

## Evidence coverage
What was readable and what was not with this role (system notes per record type, script logs,
custom records, workflows). Time zone and margins used.

## Open questions
Numbered, each addressed to someone (the person who ran it, the NetSuite admin, the
integrator).

## Appendix: queries run
The SuiteQL queries with the time window, without result data.
```

## Step 8. Offer a clean document

Once the report is written in the conversation, offer to turn it into a nice-looking document:
a PDF or Word file, or a document or page in the user's tool of choice, using whatever
document capability is available in this assistant. Keep the three evidence marks visible (for
example as small coloured labels) and the open questions at the end. Before producing it, ask
whether names, customers and amounts should stay or be anonymised, since the document may be
shared.

## Known limits

- System notes coverage depends on the role: on some accounts transactions have no readable
  field history through the MCP.
- Workflow definitions are not readable through the MCP; they are inferred from their effects.
- Employee records and some custom records may not be readable with a read-only role.
- Script logs exist only if the scripts log, and only for about 30 days.
- Read-only actions (looking at a screen, running a search) leave no trace: they rest on the
  transcript alone.
- Several people working in the same minutes on the same records make attribution harder: rely
  on the user id and the record numbers said aloud.
