# NetSuite 360 view

A skill that maps your NetSuite account in a few minutes, using only Oracle's NetSuite
MCP connector with a read-only role. Ask your assistant "give me a 360 view of my NetSuite" and it
writes one report:

- your subsidiaries, accounting books, tax nexuses and the currencies you use;
- the record types you really use, with monthly volumes (orders, invoices, payments...);
- your custom records, custom lists and custom fields, and how much each field is filled;
- the scripts deployed on each key record, and the ones that log errors;
- your saved searches and the reports your role can run;
- who and what creates your records (people, imports, integrations, scripts);
- your main business flows, for example sales order to fulfilment to invoice to payment;
- data-quality flags (missing emails, duplicates, future-dated or long-open documents);
- what your role cannot see, and why.

It ends with a short "context" block you can paste before asking your assistant to build anything on
your account (a script, an app, a report), so it starts from your real setup.

The skill only reads. It never creates or updates a record, even though the connector offers
write tools.

## Prerequisites

1. **The NetSuite MCP connector** connected in your assistant (the "MCP Standard Tools" SuiteApp
   installed in NetSuite, and the NetSuite connector added in your assistant).
2. **A read-only NetSuite role** for that connection: View permissions only. What the report
   shows depends on that role: the more record types it can view, the more complete the map.
   Useful View permissions: Customers, Items, transactions you use (Sales Order, Invoice,
   Customer Payment, Purchase Order...), Find Transaction, Perform Search, custom record types
   and lists, SuiteScript. Optional: Subsidiaries, Currency, Workflow.
3. Two guides, shared after the webinar, walk your team through it:
   - "How to convince your IT to activate the NetSuite MCP"
   - "Activate the NetSuite MCP, read-only"
4. An AI assistant that supports skills: Claude or ChatGPT (see the install steps).

Try it on a sandbox first. On a sandbox, activity stops at the last refresh: the skill detects
it and says so.

## Install

See the [install steps](../../README.md#install) for Claude Code, claude.ai and ChatGPT.

## What it can and cannot do

| Topic | What the skill gives you |
|---|---|
| Record types, volumes, flows | Yes, for what your role can view |
| Custom records and lists | Yes |
| Custom fields | Names, labels and fill rates |
| Scripts | What is deployed where, and which ones log errors |
| Workflows, saved-search criteria | Counts at most, often not visible through the MCP |
| Who writes each field | Who creates documents, partly |
| Output | One report and a context block, in the chat |

## Privacy

The report is written in your conversation, from your NetSuite data, under your AI
account's data settings. The skill counts fields that hold personal or banking data, and does
not copy their values. It lists names of people only where they create records; ask your assistant to
remove them before you share the report.

## Limits to keep in mind

- Results reflect your role. A record type with no rows may simply be hidden from it.
- SuiteQL through the MCP returns a few thousand rows per query and heavy queries can time out:
  the skill works with counts and recent windows (90 days, 12 months).
- Script code, workflow logic, form layouts and enabled features are not readable through the
  MCP.
- Figures are read at one point in time; run it again after big changes.
