---
id: notion_read_page
kind: capability
description: "Use this when someone asks what a specific Notion page says, or to summarize or answer a question from a Notion page."
template: standard
tools: ["notion_read_page"]
required_credentials: ["op://Engineering/Notion/token"]
network_allowlist: ["api.notion.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["what does the notion page say", "summarize the notion doc", "read the notion page"]
status: active
---

1. You need a page id. If the user gave a Notion URL, the id is the 32-hex-character string at the
   end of the URL slug — strip any dashes and any `?v=` query part. If they gave neither an id nor
   a URL, find the page first with the notion_search skill rather than guessing an id.
2. Call `use_credential` with method GET and the relative path
   `/v1/blocks/<page_id>/children?page_size=100`, passing the header `Notion-Version: 2022-06-28`.
3. The response is `results[]`, one entry per block. A block's text lives at
   `<block.type>.rich_text[]`, where each element has `plain_text` — e.g. a paragraph's text is at
   `paragraph.rich_text[*].plain_text`, a heading's at `heading_2.rich_text[*].plain_text`.
   Concatenate those to reconstruct the page text.
4. A block with `has_children: true` (toggles, nested lists, columns) holds its content one level
   down: fetch `/v1/blocks/<that block's id>/children` for the parts you actually need. Do not
   recurse through the whole tree by default.
5. If `has_more` is true, there are further blocks; fetch the next page with
   `?start_cursor=<next_cursor>` only when the answer plainly needs them.
6. On a 404, say the page was not found OR was never shared with the integration — the API cannot
   tell those apart, so do not claim the page does not exist. On other non-2xx, surface the status
   and `message` and stop.

Worked example — "summarize the Q3 planning page" (user pasted a URL) →
extract the id from the URL → `GET /v1/blocks/<id>/children?page_size=100` →
join the paragraph and heading text → summarize.

Does not do: write to the page, and does not read database row properties.
