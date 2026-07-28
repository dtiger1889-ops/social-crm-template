# SETUP.md — agent runbook: build this CRM from scratch

This file is written **for the AI agent** doing the setup. If you are a human: connect your agent to your Airtable account (MCP server or a personal access token), point it at this repo, and say *"set up this CRM"* — the agent follows the steps below. Expected agent working time: a few minutes; your involvement: one or two confirmations.

The source of truth for the base structure is [schema.json](schema.json). Do not improvise fields; build exactly what it specifies, then personalize the choice lists marked `"personalize": true` with the user.

---

## Prerequisites (confirm before building)

1. **Airtable access.** One of:
   - The **Airtable MCP server** is connected (tools like `list_bases`, `create_table`, `create_field` are available — load them via ToolSearch if they're deferred), or
   - A **personal access token** the user created at `airtable.com/create/tokens` with scopes `schema.bases:read`, `schema.bases:write`, `data.records:read`, `data.records:write`. With a raw token you use the REST endpoints listed at the bottom instead of MCP tools. **Never ask the user to paste the token into chat if a connected credential path exists.**
2. **A workspace to create the base in.** `list_workspaces` (MCP) tells you what exists; ask the user which one if there's more than one.

## Step 1 — Create the base

- Preferred: `create_base` with name `Social CRM` (or the user's preferred name) in the chosen workspace.
- If base creation isn't available on your plan/tooling: ask the user to create an **empty base** in the Airtable UI and tell you its ID (it starts with `app`). Delete the default "Table 1" at the end of step 2, or rename/reshape it into `People`.

## Step 2 — Pass 1: create all six tables with their non-link fields

Work through `schema.json` in `creationOrder.pass1_tables` order: **People, Groups, Events, Interactions, Rosters, Changelog**.

For each table, call `create_table` with:
- the table `name` and `description` from schema.json (the Interactions and Rosters descriptions are **behavioral rules for future agent sessions** — do not trim them),
- every field that is **not** `multipleRecordLinks`, in the listed order, with the listed `type`, `options`, and `description`. The field marked `"primary": true` must be first.

Gotchas:
- singleSelect / multipleSelects: pass the choice names from `options.choices`; colors are optional.
- date fields: ISO format (`YYYY-MM-DD`).
- number fields: precision 0 (integers).
- If `create_table` rejects field descriptions, create the fields bare and add descriptions afterward with `update_field`.

## Step 3 — Pass 2: create the link fields

Link fields (`multipleRecordLinks`) need the target table to exist, which is why they're a second pass. Work through `creationOrder.pass2_linkFields` in order; for each, call `create_field` on the **owning** table with the linked table's ID (`linkedTableId`).

Airtable auto-creates a reverse field on the target table. After each link field:
- rename the auto-created reverse field to the `reverseFieldName` given in schema.json (via `update_field`). This matters most on People, which receives **three** links from Rosters and one from itself (Partner) — unnamed, they collide into "Rosters", "Rosters 2", etc., and future sessions can't tell the anchor-reverse from the promoted-reverse.

## Step 4 — Verify against the spec

Call `list_tables_for_base` and diff the result against schema.json:
- all 6 tables present, every field present with the right type,
- select fields have the right choice lists,
- link fields point at the right tables.

Report any discrepancy to the user and fix it before continuing. Do not declare setup done without this check.

## Step 5 — Personalize with the user

Ask the user about the fields marked `"personalize": true`:
- **People.Tags** — replace the starter choices with the actual contexts of their life.
- **People.Member Status** — keep, rename, or delete depending on whether they have a primary club/scene.
- **Rosters.Source** — adjust to where their name lists actually come from.
- **Groups.Group Subtype** — no action needed; it's free text.

Also ask whether the **Relationship Tier** and **Follow Up Frequency** ladders fit; both are opinionated defaults documented in [SKILL.md](SKILL.md).

## Step 6 — Wire up the skill

1. Copy [SKILL.md](SKILL.md) to wherever the user's agent loads skills from (a claude.ai account skill, a Claude Code skill folder, or your platform's equivalent).
2. Fill in the placeholders in the copy: `<YOUR_BASE_ID>`, the table IDs (`<TBL_PEOPLE_ID>`, `<TBL_ROSTERS_ID>`), and the field/choice IDs referenced in the bump-scan section — all of them come from the `list_tables_for_base` / `get_table_schema` output you already have.
3. **The filled-in copy is a credential-bearing private file** (if it embeds a token for the HTTP fallback). It must never be committed to any repo. Only this template, with placeholders, is publishable.
4. Telegram placeholders are optional — skip them unless the user wants push summaries.

## Step 7 — Smoke test

1. Create one Group (e.g. ask the user for a real club or friend group of theirs).
2. Create one People record linked to it (ask the user for a real person — do not invent one).
3. Create a Changelog entry recording the setup (Tables Affected: `Base Structure`; summarize what was built).
4. Read the person back with `search_records` to confirm lookup works.

Done. Point the user at the README's use-case list for what to do with it.

---

## REST fallback (no MCP)

Same build, raw endpoints, `Authorization: Bearer <token>`:

| Action | Endpoint |
|---|---|
| Create base | `POST https://api.airtable.com/v0/meta/bases` (`{name, workspaceId, tables: [...]}` — you can create all pass-1 tables in this one call) |
| Create table | `POST https://api.airtable.com/v0/meta/bases/{baseId}/tables` |
| Create field | `POST https://api.airtable.com/v0/meta/bases/{baseId}/tables/{tableId}/fields` |
| Update field (rename reverse links, add descriptions) | `PATCH .../tables/{tableId}/fields/{fieldId}` |
| Read schema | `GET https://api.airtable.com/v0/meta/bases/{baseId}/tables` |

Link field payload shape: `{"name": "...", "type": "multipleRecordLinks", "options": {"linkedTableId": "tblXXXXXXXXXXXXXX"}}`.
