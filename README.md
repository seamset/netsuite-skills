# NetSuite skills

Skills that let your AI assistant (Claude or ChatGPT) understand your NetSuite account. They read
NetSuite only through Oracle's NetSuite MCP connector, with a read-only role, and never change a
record.

| Skill | What it does |
|---|---|
| [`netsuite-360-view`](skills/netsuite-360-view/) | Maps your account: subsidiaries, record types and volumes, custom records and fields, scripts, saved searches, who creates what, main business flows, data-quality flags. Ends with a context block to reuse before building anything. |
| [`describe-your-process`](skills/describe-your-process/) | From the recording of someone running a real process in NetSuite, matches each spoken step with what NetSuite recorded in the same minutes, then writes the process in detail: records, fields, scripts, workflows, approvals and their implications. |

## Before you start

1. **Connect the NetSuite MCP.** Your NetSuite admin installs Oracle's free "MCP Standard Tools"
   SuiteApp and gives you a role with View permissions only. Connector address:
   `https://<accountid>.suitetalk.api.netsuite.com/services/mcp/v1/suiteapp/com.netsuite.mcpstandardtools`
   (account ID in lowercase, `_` replaced by `-`, for example `1234567-sb1`).
2. **Try it on a sandbox first.**

## Install

**Claude Code**

```
/plugin marketplace add seamset/netsuite-skills
/plugin install netsuite-skills@netsuite-skills
```

**claude.ai and Claude desktop**

Download the skill's zip from [Releases](https://github.com/seamset/netsuite-skills/releases),
then Customize > Skills > + > Create skill > Upload a skill. Code execution must be on.

**ChatGPT**

Download the same zip from [Releases](https://github.com/seamset/netsuite-skills/releases), then
Skills > Create > Upload from your computer. Add the NetSuite connector as a custom app in
Developer mode (Settings > Apps > Advanced settings), on a paid plan. On Business or Enterprise
workspaces, an admin may need to allow it.

## Use

Open a new conversation with the NetSuite connector on, then ask:

- "Give me a 360 view of my NetSuite account."
- "Here is the transcript of the recording where I ran our credit memo process. Describe my
  current process."

## Read-only by design

- The skills only call read tools. They never call `ns_createRecord` or `ns_updateRecord`, even if
  asked.
- What they see is limited by your NetSuite role. A record type your role cannot view is reported
  as not visible, never as empty.
- Every SuiteQL query they run is logged by NetSuite.

## License

MIT. Made by [SeamSet](https://seamset.com).
