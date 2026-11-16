# Describe your current process

## What it is

You run one of your real processes in NetSuite while recording a call, and you say out loud what
you do. Your assistant then reads the transcript, looks in NetSuite at what really happened in those
minutes (through Oracle's NetSuite MCP connector, read-only) and writes how the process actually
works: the steps in order, who does what, the records and fields touched, the forms, scripts,
workflows, approvals and integrations behind each step, what triggers what, and what would break
if you automated it.

Every statement in the report is marked:

- **[NS]** seen in NetSuite;
- **[SAID]** only said in the recording;
- **[INFERRED]** deduced, with what it is deduced from.

The report ends with open questions for the person who ran the process, the NetSuite admin or
your integrator. Your assistant can then turn it into a PDF or a shareable document.

## How to record a good run

1. **One process per recording.** "Rebill a customer after a VAT change", not "my month-end".
2. **Note the start time and your time zone** at the beginning ("it is 10:02, Paris time").
   Most recorders save it, but saying it removes any doubt.
3. **Use your own NetSuite login**, the one you use every day, so your actions can be told apart
   from other users' and from scripts.
4. **Say what you do, out loud, as you do it.** "I open sales order 1234 for customer X", "I
   change the tax code on line 2", "I click Save", "now I wait for the approval".
5. **Name the records**: type and number (invoice, credit memo, vendor bill...), and the
   customer or vendor.
6. **Say what you check and why**, even when you only look: "I look at this saved search to
   see what is still open". Looking leaves no trace in NetSuite, so the recording is the only
   evidence.
7. **Say what usually happens next** and what you do outside NetSuite (Excel, email, a bank
   portal): "normally finance approves it tomorrow", "I copy this into my spreadsheet".
8. **Run it for real, or in a sandbox that mirrors production.** Do not stop halfway.
9. **Run the skill within a few days.** NetSuite keeps script logs for about 30 days.

Any recorder that produces a transcript with timestamps works: Fireflies, Zoom, Microsoft
Teams, Google Meet, Otter and others.

## Prerequisites

- An AI assistant that supports skills: Claude or ChatGPT.
- Oracle's NetSuite MCP connector (the "MCP Standard Tools" SuiteApp) connected to your NetSuite
  account with a **read-only role**. The role should be able to read transactions, entities,
  system notes, scripts and script deployments, and the custom records your process uses.
  Two guides from SeamSet help your IT set it up: **"How to convince your IT to activate the
  NetSuite MCP"** and **"Activate the NetSuite MCP, read-only"**.
- The transcript of your recorded run: pasted, attached as a file, or reachable through a
  meeting-notes connector you already use in your assistant.

## Install

See the [install steps](../../README.md#install) for Claude Code, claude.ai and ChatGPT.

## What it will and will not do

- It only reads. It never creates, changes or deletes anything in NetSuite, even if asked.
- It reads only what your MCP role is allowed to read.
- It quotes the transcript sparingly and asks before keeping names, customers and amounts in a
  document you may share.

## Limits

- **Field history depends on your role.** NetSuite's system notes (who changed which field, from
  what to what) are readable through the MCP for some record types and not others, depending on
  the role. When they are not, the report relies on creation and modification times, script
  logs and what you said, and says so.
- **Workflows are inferred, not read.** The MCP does not expose workflow definitions. The assistant
  sees their effects (field changes made by a workflow, workflow action scripts, approval
  status changes) and lists the rest as questions for your admin.
- **Script logs** exist only if the script's developer logs something, and only for about 30
  days.
- **Looking leaves no trace.** Screens you opened, searches you ran and checks you made rest on
  the recording alone.
- **Time zones matter.** NetSuite shows record times in the MCP user's time zone and script logs
  in the data center's time zone. The skill calibrates both, but a wrong recording time zone
  makes it miss everything: confirm it when asked.
- **Busy accounts.** If several people work on the same records in the same minutes,
  attribution is harder. Saying record numbers aloud helps.
- **Large processes.** Very long recordings or windows with thousands of changes are slower and
  may need to be split.

Queries were tested on a NetSuite sandbox through the MCP Standard Tools on 2026-11-16. Each
account's customisations and role permissions differ, so the skill reports what it could not
read rather than assuming.
