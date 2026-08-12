---
id: jira_search
kind: capability
description: "Use this when someone asks to find, search, list or look up Jira issues by project, status, assignee or text."
template: standard
tools: ["jira_search"]
required_credentials: ["op://Engineering/Jira/base_url", "op://Engineering/Jira/api_token", "op://Engineering/Jira/email"]
network_allowlist: ["*.atlassian.net"]
needs_approval: false
model_tier: mid
trigger_patterns: ["jira", "issue", "ticket", "sprint", "backlog"]
status: active
---

1. Build a JQL query from what the user gave you. Use only the filters they named — never prompt
   for optional filters, and never invent a project key. If they named no project, search across
   all of them rather than guessing one.
2. Call `use_credential` with method GET and a relative path of
   `/rest/api/3/search/jql?jql=<url-encoded JQL>&maxResults=25&fields=summary,status,assignee,updated`.
3. Read `issues[]` from the response. For each, report `key`, `fields.summary`,
   `fields.status.name`, and `fields.assignee.displayName` (or "unassigned" when assignee is null).
4. This endpoint paginates by token, not by count: there is no `total`. If `isLast` is false (or a
   `nextPageToken` is present), say that more issues matched and the list was capped at 25. Never
   report a total count — you do not have one, and inventing it is worse than omitting it.
5. On a non-2xx, surface the status code and `errorMessages[0]` and stop. Do not retry — a 401 or
   403 means the credentials need refreshing, not that the query was wrong.

Worked example — "what's open in ENG assigned to Priya":
JQL `project = ENG AND status != Done AND assignee = "Priya"` becomes
`GET /rest/api/3/search/jql?jql=project%20%3D%20ENG%20AND%20status%20!%3D%20Done%20AND%20assignee%20%3D%20%22Priya%22&maxResults=25&fields=summary,status,assignee,updated`
and the reply lists each key, summary and status.

Does not do: create, edit, transition or comment on issues.
