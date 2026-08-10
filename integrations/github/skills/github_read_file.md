---
id: github_read_file
kind: capability
description: "Use this when someone asks what a specific file contains, or to check config, docs or code on a GitHub repo."
template: standard
tools: ["github_read_file"]
required_credentials: ["op://Engineering/GitHub/token"]
network_allowlist: ["api.github.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["read the file", "what's in", "show me the code", "check the config"]
status: active
---

1. You need the repo (`owner/name`) and the file path. Ask only if one is genuinely missing.
2. Call `use_credential` with method GET and the relative path
   `/repos/<owner>/<name>/contents/<path>`, passing the header
   `Accept: application/vnd.github.raw` — this returns the file body directly, so you do NOT have
   to base64-decode anything.
3. To read from a branch or tag other than the default, append `?ref=<branch-or-sha>`. If the user
   named a branch containing slashes, url-encode it.
4. Answer the user's actual question about the file. Quote only the relevant lines — do not paste
   an entire large file back into chat.
5. On 404, say the file or repo was not found and that the token may not have access to it — both
   look identical from the API, so do not assert the file does not exist. On a non-2xx, surface the
   status and `message` and stop.

Worked example — "what's in the Dockerfile on acme/api" →
`GET /repos/acme/api/contents/Dockerfile` with `Accept: application/vnd.github.raw` →
summarize the base image, exposed ports and entrypoint.

Does not do: write, commit, or open pull requests; and does not read a whole repository tree —
ask for a specific path.
