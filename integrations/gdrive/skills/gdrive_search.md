---
id: gdrive_search
kind: capability
description: "Use this when someone asks to find a file, doc, sheet or folder in Google Drive."
template: standard
tools: ["gdrive_search"]
required_credentials: ["op://Engineering/GoogleDrive/service_account_json"]
network_allowlist: ["www.googleapis.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["google drive", "find the doc", "the spreadsheet", "shared folder"]
status: active
---

1. Build a Drive query. The qualifiers that matter: `name contains 'x'`,
   `fullText contains 'x'` (searches file CONTENT, not just the name),
   `mimeType = 'application/vnd.google-apps.document'`, and `modifiedTime > '2026-01-01T00:00:00'`.
   Join conditions with ` and `. Single-quote every literal.
2. Call `use_credential` with method GET and the relative path
   `/files?q=<url-encoded query>&pageSize=25&fields=files(id,name,mimeType,modifiedTime,webViewLink)&orderBy=modifiedTime desc`.
3. Read `files[]` and report `name`, `webViewLink`, and `modifiedTime`. Give the `id` too when the
   user is likely to want the contents next — gdrive_read_doc needs it.
4. Useful mimeTypes: Docs `application/vnd.google-apps.document`,
   Sheets `application/vnd.google-apps.spreadsheet`, folders
   `application/vnd.google-apps.folder`. Filter by type only when the user named one.
5. An empty result usually means the file was never shared with the service account, not that it
   does not exist. Say so — access here is granted by sharing a file or folder with the service
   account's email, and "nothing found" and "not shared with me" look identical from the API.
6. On a non-2xx, surface the status and `error.message` and stop. A 403 naming
   `insufficientPermissions` means the scope is wrong, not the query.

Worked example — "find the Q3 budget sheet" →
query `name contains 'Q3 budget' and mimeType = 'application/vnd.google-apps.spreadsheet'` →
`GET /files?q=...&pageSize=25&fields=files(id,name,mimeType,modifiedTime,webViewLink)&orderBy=modifiedTime desc`
→ reply with each name and link.

Does not do: create, edit, move or delete files — this connection is read-only by design.
