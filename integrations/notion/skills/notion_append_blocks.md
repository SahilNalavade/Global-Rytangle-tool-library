---
id: notion_append_blocks
kind: capability
description: "Use this when someone asks to add notes, a summary or a section to an existing Notion page."
template: standard
tools: ["notion_append_blocks"]
required_credentials: ["op://Engineering/Notion/token"]
network_allowlist: ["api.notion.com"]
needs_approval: true
model_tier: mid
trigger_patterns: ["add this to notion", "append to the notion page", "write this up in notion"]
status: active
---

1. You need the target page id and the content to add. If the user did not name a page, ask which
   one — never pick a page yourself, and never create a new one from this skill.
2. Call `use_credential` with method PATCH to `/v1/blocks/<page_id>/children`, the header
   `Notion-Version: 2022-06-28`, and a body of
   `{"children":[{"object":"block","type":"paragraph","paragraph":{"rich_text":[{"type":"text","text":{"content":"<text>"}}]}}]}`.
3. One block per paragraph. For a heading use `"type":"heading_2"` with a `heading_2` key of the
   same shape; for a bullet use `"type":"bulleted_list_item"`. Send several blocks in one call
   rather than one call per line.
4. A single text `content` value is capped at 2000 characters. Split longer text across multiple
   blocks — a single oversized block fails the whole request with a validation error.
5. Append only. This adds to the END of the page and cannot edit or delete what is already there,
   which is deliberate: the agent never rewrites human-written content.
6. On success, reply with the page URL so the user can see what landed. On a non-2xx, surface the
   status and `message` and stop — a 404 here usually means the page was not shared with the
   integration with WRITE access (sharing read-only is the common mistake).

Worked example — "add the decision summary to the Q3 planning page" →
PATCH `/v1/blocks/<id>/children` with one heading_2 block and two paragraph blocks →
reply with the page URL.

Does not do: create pages, edit or delete existing blocks, or change page properties.
