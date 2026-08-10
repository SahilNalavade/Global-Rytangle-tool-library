---
id: github_search_issues
kind: capability
description: "Use this when someone asks what issues or pull requests are open, assigned, stale or recently merged on a GitHub repo."
template: standard
tools: ["github_search_issues"]
required_credentials: ["op://Engineering/GitHub/token"]
network_allowlist: ["api.github.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["github", "pull request", "PR", "open issues", "merged"]
status: active
---

1. Build a GitHub search query from what the user asked. The qualifiers that matter most:
   `repo:owner/name`, `is:issue` or `is:pr`, `is:open` / `is:closed`, `author:`, `assignee:`,
   `label:`, and `created:>YYYY-MM-DD`. Use only what the user actually named.
2. Call `use_credential` with method GET and the relative path
   `/search/issues?q=<url-encoded query>&per_page=25&sort=updated`, passing the header
   `Accept: application/vnd.github+json`.
3. Read `items[]`. For each, report `number`, `title`, `state`, `user.login`, and `html_url`.
   An item with a `pull_request` key is a PR, not an issue — say which it is.
4. `total_count` is the full match count. If it exceeds the number returned, say how many more
   matched so the user knows the list was capped.
5. On a non-2xx, surface the status and the `message` field, then stop. A 422 almost always means
   the query had a malformed qualifier — show the query you built so the user can see it. Do not
   silently retry with a different query.

Worked example — "what PRs are open on acme/api":
query `repo:acme/api is:pr is:open` becomes
`GET /search/issues?q=repo%3Aacme%2Fapi+is%3Apr+is%3Aopen&per_page=25&sort=updated`
and the reply lists each number, title and author.

Note: search only covers repositories the team's token can see. If a repo the user named returns
nothing, say the token may not have access rather than reporting "no results" as fact.

Does not do: create, comment on, merge, or close anything.
