# Tool library — layout contract

The tool library is a repo of **pre-vetted integrations**. A tenant's agent installs one verbatim
instead of authoring it with the LLM.

This exists for one reason: every connector-authoring failure measured on the credval stack —
unknown tool ids, a stray `base_url` on an MCP connection, a missing `data_scope`, non-deterministic
transport choice — is an **authoring** error. None of them can happen to an artifact that is copied
rather than generated. The generation step is deleted, not improved.

## Layout

Entries are derived from directory structure. There is no index file, so there is nothing to drift.

```
integrations/
  <id>/
    entry.yaml          # REQUIRED — id, version, description (all strings)
    connection.yaml     # REQUIRED — the connection manifest, copied verbatim
    tools.yaml          # REQUIRED — tool verbs, merged into config/tools-registry.yaml
    skills/             # optional — a manifest-only entry is valid
      <skill_id>.md
```

An entry missing `entry.yaml`, `connection.yaml` or `tools.yaml` is **skipped with a warning**, not
a boot failure (`loadLibrary` degrades to empty, mirroring `loadConnections`).

## Rules

1. **`skills/*.md` frontmatter must satisfy `SkillFrontmatter`** (`src/kernel/capabilities/skill.ts`)
   — `id, kind, description, template, tools, required_credentials, network_allowlist,
   needs_approval, model_tier, trigger_patterns, status`. A skill whose frontmatter is malformed is
   skipped at load with only a `console.warn`, so it would install "successfully" and then not exist.
2. **Every tool a skill declares must appear in that entry's `tools.yaml`** or be a built-in. A skill
   naming an unknown tool is dropped by `loadSkillCatalog`.
3. **Any skill using a `write: true` tool must set `needs_approval: true`.** A write must never
   install un-gated.
4. **Tenant-specific hosts use `base_url_ref`, not `base_url`.** You cannot know a customer's
   Atlassian domain at authoring time; the value is collected through the credential form and
   resolved from the vault at runtime.
5. **Never put a credential VALUE anywhere.** `op://` references only.
6. **Every release must be tagged.** Deployments pin `LIBRARY_REF` to a tag or SHA — the kernel
   refuses to boot with `LIBRARY_REPO` set and `LIBRARY_REF` unset.

Rules 1–3 are enforced by `tests/unit/kernel/library_content.test.ts`, which runs the shipped
content through the real `parseSkill` and `loadConnections`. Add content, run that test.

## Install semantics

- **Copy-on-install.** Files are written into the tenant's brain repo. Nothing depends on the
  library being reachable afterwards.
- **Provenance.** The installed manifest carries `source: library/<id>@<version>`.
- **Tenant wins on collision.** If the brain repo already has `config/connections/<id>.yaml`, the
  install is refused. An install is a fork, not a subscription — a library bump never clobbers a
  tenant's own edits.
- **Idempotent registry merge.** Re-installing cannot duplicate tool verbs.
- **Gated.** `install_from_library` requires the `capabilityBuild` scope and is disabled under a
  frozen license, exactly like `create_skill`.

## Deployment

| Env var | Meaning |
|---|---|
| `LIBRARY_REPO` | `owner/repo` (a full GitHub URL is accepted and canonicalized). **Unset ⇒ the feature is dormant** and nothing changes. |
| `LIBRARY_REF` | Tag or SHA. Required whenever `LIBRARY_REPO` is set. |
| `LIBRARY_DIR` | Clone destination, default `/data/library`. Must be outside the kernel repo. |

Current test library — `SahilNalavade/Global-Rytangle-tool-library` @ `v1.0.0` (public). It is cloned
with `GIT_TOKEN` via `authUrl()`, the same credential the brain repo uses, so a library hosted
outside github.com would need that path revisited.

The clone is read-only, refreshed on the existing 60s brain poll (no new timer), and deliberately
does **not** take the brain repo's single-writer lock — see decisions.md #214.

## Publishing

`library/` in this repo is the authoring source. Publish by copying `library/integrations/` to the
library repo and tagging:

```bash
git tag v1.0.0 && git push origin v1.0.0
```

Then bump `LIBRARY_REF` on the deployment. A library edit does **not** reach a running tenant until
that bump — which is the point.
