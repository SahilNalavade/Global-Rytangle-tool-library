---
id: gdrive_read_doc
kind: capability
description: "Use this when someone asks what a Google Doc says, or to summarize or answer a question from a Drive file."
template: standard
tools: ["gdrive_read_doc"]
required_credentials: ["op://Engineering/GoogleDrive/service_account_json"]
network_allowlist: ["www.googleapis.com"]
needs_approval: false
model_tier: mid
trigger_patterns: ["what does the doc say", "summarize the google doc", "read the drive file"]
status: active
---

1. You need a file id. If the user pasted a Drive URL, the id is the segment between `/d/` and the
   next `/`. If they named a file instead, find it first with the gdrive_search skill — never guess
   an id.
2. **The endpoint depends on the file type**, and getting this wrong is the most common failure:
   - A Google Doc, Sheet or Slide is a Google-native file with no bytes to download. Export it:
     `GET /files/<id>/export?mimeType=text/plain`
   - Any other file (uploaded .txt, .md, .csv, .json) has real bytes:
     `GET /files/<id>?alt=media`
   If you do not know the type, call `GET /files/<id>?fields=mimeType` first and branch on whether
   it starts with `application/vnd.google-apps.`.
3. Call `use_credential` with method GET and the path from step 2.
4. Answer the user's actual question. For a long document, summarize — do not paste the whole
   export back into chat.
5. Export tops out around 10 MB. If the response is truncated or the call fails on size, say the
   document was too large to read in full rather than answering from a partial read as if complete.
6. On 404, say the file was not found OR is not shared with the service account — the API does not
   distinguish them, so do not claim it does not exist. On a 403 for a Google-native file, check you
   used `/export` and not `alt=media`: that mismatch returns
   `fileNotDownloadable`, which reads like a permission problem but is not.

Worked example — "summarize the Q3 planning doc" (user pasted a Docs URL) →
extract the id → `GET /files/<id>/export?mimeType=text/plain` → summarize the returned text.

Does not do: write or edit files, and does not read Sheets cell-by-cell — export gives flat text.
