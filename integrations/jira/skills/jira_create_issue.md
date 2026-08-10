---
id: jira_create_issue
kind: capability
description: "Use this when someone asks to create, file, raise or open a new Jira issue or ticket."
template: standard
tools: ["jira_create_issue"]
required_credentials: ["op://Engineering/Jira/api_token", "op://Engineering/Jira/email"]
network_allowlist: ["*.atlassian.net"]
needs_approval: true
model_tier: mid
trigger_patterns: ["create a ticket", "file an issue", "raise a bug", "open a jira"]
status: active
---

1. You need a project key and a summary. Ask only if one of those two is genuinely missing —
   everything else is optional and should be omitted rather than prompted for.
2. Call `use_credential` with method POST to `/rest/api/3/issue` and a body of
   `{"fields":{"project":{"key":"<KEY>"},"summary":"<summary>","issuetype":{"name":"Task"},
   "description":{"type":"doc","version":1,"content":[{"type":"paragraph","content":[{"type":"text","text":"<description>"}]}]}}}`.
   Omit the `description` field entirely when the user gave none — an empty ADF document is rejected.
3. On success read `key` from the response and reply with the created issue key.
4. On a non-2xx, surface the status code and the `errors` object and stop. Do not retry. A 400
   naming `issuetype` usually means this project has no "Task" type — report that plainly rather
   than guessing another type.

Worked example — "file a bug in ENG: login times out on Safari" →
POST `/rest/api/3/issue` with project key ENG and summary "login times out on Safari" →
reply "Created ENG-482".

Does not do: modify, transition or comment on existing issues, and does not act on human-created
content unless an explicit issue key is supplied.
