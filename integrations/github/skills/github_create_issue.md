---
id: github_create_issue
kind: capability
description: "Use this when someone asks to open, file or raise a GitHub issue on a repository."
template: standard
tools: ["github_create_issue"]
required_credentials: ["op://Engineering/GitHub/token"]
network_allowlist: ["api.github.com"]
needs_approval: true
model_tier: mid
trigger_patterns: ["open a github issue", "file a github issue", "raise a github bug"]
status: active
---

1. You need the repo (`owner/name`) and a title. Ask only if one of those two is missing; labels
   and assignees are optional and should be omitted unless the user named them.
2. Call `use_credential` with method POST to `/repos/<owner>/<name>/issues`, header
   `Accept: application/vnd.github+json`, and a body of
   `{"title":"<title>","body":"<body>"}`. Add `"labels":["..."]` or `"assignees":["..."]` only when
   the user explicitly asked for them.
3. In the `body`, include who asked for the issue and where it came from (the chat request), so the
   issue's own record says who wanted it — the API call is authenticated as the team's service
   token, so GitHub will show the token's identity as the author, not the person who asked.
4. On success read `number` and `html_url` from the response and reply with both.
5. On a non-2xx, surface the status and `message` and stop. A 410 means issues are disabled on that
   repo; a 403 means the token lacks Issues:write. Report which — do not retry.

Worked example — "open an issue on acme/api: healthcheck flaps under load" →
POST `/repos/acme/api/issues` with that title and a body naming the requester →
reply "Opened acme/api#212".

Does not do: edit, close, comment on, or label existing issues, and does not touch pull requests.
