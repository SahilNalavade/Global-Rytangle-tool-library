---
id: notion_search
kind: capability
description: "Use this when someone asks to find a Notion page or doc, or asks what the team has written down about a topic in Notion."
template: standard
tools: ["notion_search"]
required_credentials: ["op://Engineering/Notion/token"]
network_allowlist: ["api.notion.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["notion", "find the doc", "our notes on", "the page about"]
status: active
---

1. Notion search is POST, not GET. Call `use_credential` with method POST to `/v1/search`, the
   header `Notion-Version: 2022-06-28`, and a body of
   `{"query":"<what they're looking for>","page_size":10}`.
   To restrict to pages only, add `"filter":{"value":"page","property":"object"}`.
2. Notion's search matches on TITLE, not full page text. If the user described content rather than
   a title, search the most likely title words rather than their whole sentence.
3. Read `results[]`. For each, the title is at
   `properties.title.title[0].plain_text` for a page in a database, or
   `properties.Name.title[0].plain_text` in many workspaces — check both and fall back to the
   page's `url`. Report the title, the `url`, and `last_edited_time`.
4. An empty result set usually means the page was never shared with the integration, not that it
   does not exist. Say that explicitly — it is the single most common cause of a Notion miss, and
   the fix is for a human to share the page in Notion.
5. On a non-2xx, surface the status and the `message` field, then stop. A 401 means the token needs
   refreshing; do not retry.

Worked example — "find our onboarding doc in Notion" →
POST `/v1/search` with `{"query":"onboarding","page_size":10}` and the Notion-Version header →
reply listing each matching page title with its URL.

Does not do: create or edit pages, and does not query database rows with filters.
