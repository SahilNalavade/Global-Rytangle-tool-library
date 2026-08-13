---
id: notion_create_page
kind: capability
description: "Use this when someone asks to create a new Notion page, start a new doc, or write something up in Notion as its own page rather than adding to an existing one."
template: standard
tools: ["notion_create_page"]
required_credentials: ["op://Engineering/Notion/token"]
network_allowlist: ["api.notion.com"]
needs_approval: true
model_tier: mid
trigger_patterns: ["create a notion page", "make a page in notion", "start a new notion doc", "new page in notion"]
status: active
---

1. You need a PARENT page id and a title. If the user did not name a parent, ask which page the new
   one should live under — never pick a parent yourself, and never create at the workspace root.
   If they want the content added to a page that already exists, this is the wrong skill: use
   `notion_append_blocks` instead.
2. Call `use_credential` with method POST to `/v1/pages`, the header `Notion-Version: 2022-06-28`,
   and a body of
   `{"parent":{"page_id":"<parent_id>"},"properties":{"title":{"title":[{"text":{"content":"<title>"}}]}},"children":[]}`.
   For a page parent the property key is literally `title` — a workspace's own column names like
   `Name` apply to database rows, which this skill does not create.
3. Put the initial body content in `children`, using the same block vocabulary as
   `notion_append_blocks`: `paragraph`, `heading_2` and `bulleted_list_item`, each shaped
   `{"object":"block","type":"<type>","<type>":{"rich_text":[{"type":"text","text":{"content":"<text>"}}]}}`.
   Send the content in this one call rather than creating an empty page and appending afterwards.
4. A single text `content` value is capped at 2000 characters and one request accepts at most 100
   blocks. Split longer text across multiple blocks; if the content exceeds 100 blocks, create the
   page with the first 100 and follow up with `notion_append_blocks` for the rest.
5. `parent` must be a page id, not a database id. If the user names a database or a tracker, say
   that creating database rows is not supported here and stop — a `database_id` parent requires
   properties matching that database's schema and will fail validation.
6. On success, reply with the `url` from the response so the user can open what was created. On a
   non-2xx, surface the status and `message` and stop. A 404 on the parent almost always means the
   parent page was not shared with the integration with WRITE access (sharing read-only is the
   common mistake), not that the id is wrong.

Worked example — "start a new Notion page under Engineering called Q3 Retro with our three
takeaways" → POST `/v1/pages` with a `page_id` parent, a `title` of "Q3 Retro", and `children` of
one heading_2 block and three bulleted_list_item blocks → reply with the new page's URL.

Does not do: create database rows, edit or delete existing pages, change page properties after
creation, or move a page to a different parent.
