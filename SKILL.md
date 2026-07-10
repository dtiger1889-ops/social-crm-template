---
name: social-crm
description: >
  A personal Social CRM template in Airtable. Use this skill for ANY task involving the Social CRM —
  adding people, logging interactions, updating records, querying contacts, linking groups/clubs,
  globbing rosters (name dumps from events/sign-up lists), running data quality checks, or managing
  schema. Triggers on mentions of CRM, Social CRM, Airtable contacts, adding or updating a person,
  logging an interaction, community nicknames, clubs, "glob this", roster, sign-up list, or any request to
  look up, add, or update a contact.
---

# Social CRM — Personal Airtable CRM (template)

## What this is
A personal Social CRM built in Airtable for tracking relationships: friends, club contacts, acquaintances. The CRM records people, their contact info and context, their connections to groups (clubs, teams, etc.), interactions the user has with them, and Rosters — raw name-dumps from events that are kept as lists, not contacts.

---

## Credentials & API Access

- **Airtable Base ID:** `<YOUR_BASE_ID>`
- **Airtable Token:** `<YOUR_AIRTABLE_TOKEN>`
- **Telegram Bot Token:** `<YOUR_TELEGRAM_BOT_TOKEN>`
- **Telegram Chat ID (the user):** `<YOUR_TELEGRAM_CHAT_ID>`

### How to make API calls

**Preferred: Airtable MCP server** (tools named `<airtable-mcp>__*`). It works in any session — interactive AND unattended scheduled runs — with no Chrome dependency and no permission prompts. Use this for ALL routine CRM operations.

If the MCP tools aren't in your active tool list, load them in bulk:
```
ToolSearch { query: "<your-airtable-mcp-server-id>", max_results: 30 }
```
That returns every Airtable MCP tool by server-name substring match. Don't `select:` one tool at a time.

**Key MCP tools and their gotchas:**

- `list_bases` → confirm the My People base ID is `<YOUR_BASE_ID>`.
- `list_tables_for_base` → get table IDs and field IDs. Call this once at task start instead of the old metadata HTTP call.
- `get_table_schema` → required input is `tables: [{tableId, fieldIds: [...]}]` (array of objects with explicit field IDs). Use this when you need singleSelect choice IDs.
- `list_records_for_table` → **does NOT support `filterByFormula`.** Use the structured `filters` parameter:
  - Shape: `{operator: "and"|"or", operands: [...]}` at the top level. **Operands are flat — you cannot nest another `or` inside `operands`**, only individual conditions. If you need a multi-clause OR (e.g., a multi-value "Location contains X OR Y OR ..."), filter server-side on what you can and finish the filter client-side after the fetch.
  - Each condition: `{operator: "contains" | "=" | "isAnyOf" | "isEmpty" | ..., operands: [fieldId, value]}`.
  - For singleSelect/multipleSelects, pass **choice IDs not names** — get them from `get_table_schema` first.
  - `sort` is `[{fieldId, direction}]` (not `field`).
  - Use field IDs (`<FIELD_ID>`), not field names, for reliability.
- `search_records` → free-text fuzzy search across fields. Good for "find person X by partial name" but does NOT cover date / rating / checkbox / button fields.
- `create_records_for_table`, `update_records_for_table`, `delete_records_for_table` → for writes. Pass singleSelect values as plain string names on write (not as objects), even though they come back as objects on read.

**Pattern — typical lookup:**
```
1. (once per session) list_tables_for_base → cache table/field IDs
2. (if needed) get_table_schema → cache singleSelect choice IDs
3. list_records_for_table with structured filters → fetch records
4. Filter / sort client-side for anything the structured filter can't express
```

**Fallback: Chrome `javascript_tool` fetch().** Use only when the Airtable MCP errors or returns nothing. Same token, same endpoints:
1. Call `tabs_context_mcp(createIfEmpty: true)` then navigate to `https://airtable.com` — exactly once per task.
2. Do ALL work in a single `javascript_tool` IIFE (every extra call opens a new tab).
3. If Chrome is unreachable, call `mcp__computer-use__request_access` for "Google Chrome" then `mcp__computer-use__open_application` to launch it. Do NOT ask the user to open Chrome.

```js
// Chrome fallback pattern
(async () => {
  const TOKEN = '<YOUR_AIRTABLE_TOKEN>';
  const BASE = '<YOUR_BASE_ID>';
  const H = { 'Authorization': `Bearer ${TOKEN}`, 'Content-Type': 'application/json' };
  const api = (path, opts = {}) => fetch(`https://api.airtable.com/v0/${path}`, { headers: H, ...opts }).then(r => r.json());
  // ...
})()
```

For Metadata API (schema changes via Chrome fallback): `https://api.airtable.com/v0/meta/bases/{baseId}/...`
For Data API (records via Chrome fallback): `https://api.airtable.com/v0/{baseId}/{tableIdOrName}/...`

---

## Automated runs that depend on this CRM (read before any schema change)

If you build scheduled automations on top of this CRM (nudge reports, data-quality scans, digest jobs), they will hard-code field IDs, choice IDs, and tier/cadence names. Before you rename a field, change a singleSelect choice, retire a tier, or restructure a table, find every automation that references the base ID and update it in the same session; otherwise the automation silently breaks or writes to the wrong place. A simple way to keep an inventory current: grep your automations folder for the base ID and maintain a dependency table (run, what it touches, which IDs it depends on) in this file. Example automations that work well on this schema: a weekly "who is due for a reach-out" nudge, a monthly data-quality scan, a periodic changelog consolidation.

---

## Tables

Always fetch current schema at the start of any task to get field IDs:
```
GET https://api.airtable.com/v0/meta/bases/<YOUR_BASE_ID>/tables
```

### People (<TBL_PEOPLE_ID>)
The main contact table. One record per person.

| Field Name | Type | Purpose / Notes |
|---|---|---|
| Name | singleLineText | Full name or primary identifier |
| Community Name | singleLineText | Nickname used in a social scene or club, if any |
| Nickname / Aliases | singleLineText | Common nicknames or alternate names |
| Phone | phoneNumber | Primary phone |
| Email | email | Primary email |
| Location | singleLineText | Street address or a rough area description |
| Birthday | date | Format: YYYY-MM-DD |
| Known Since | date | Approximate date first met — use YYYY-01-01 if only year known |
| Relationship Tier | singleSelect | Close Friend, Friend, Acquaintance, **Building**, etc. — see "Relationship Tier options" below. `Building` = someone the user is actively turning into a friend (not a friend yet). |
| Follow Up Frequency | singleSelect | Per-person contact cadence — how often the user wants to reach out. Drives the bump scan. See "Follow Up Frequency (cadence)" below. |
| Next Follow-Up | date | Optional MANUAL override date for the next planned reach-out. Not auto-computed; the bump scan derives due-ness from Last Contacted + cadence, but the user can set this by hand to pin a specific date. |
| Likes & Interests | multilineText | Hobbies, preferences, things they enjoy |
| Family and Pets | multilineText | Spouses, kids, pets — people close to them who don't have their own record. One relation per line in the form "Relation: Name"; use "(name unknown)" where a name is not known |
| Job/Employer | singleLineText | Job title and/or employer (field ID: <FIELD_ID>) |
| Notes | multilineText | **Catch-all / last resort.** Data here should be minimal. If it belongs in a structured field, move it there. |
| Last Contacted | date | Date of most recent real-world contact |
| How We Met | singleLineText | Brief context for where/how the user met this person |
| Romantic Interest | singleSelect | See options below |
| Groups | linkedRecord → Groups | **Clubs and groups this person belongs to.** This is a linked record — never store club/group names as text in Notes or any other field. Look up or create the Group record and link it. |

**Romantic Interest options** (with the user's definitions):
- `Interested` — the user is interested, doesn't know if mutual
- `Flirting` — something going on, not confirmed how strong
- `Mutual` — physical contact has happened (made out, hooked up, etc.)
- `Situationship` — ongoing something
- `Former Hookup` — was physical, no longer
- `Former Flame` — above a hookup level, no longer happening

**Relationship Tier options:** Close Friend, Friend, Acquaintance, Work Contact, Scene Friend, Scene Foe, Old Friend, **Building**.
- `Building` (added 2026-06-13) is the home for people the user has met and wants to befriend but who **aren't friends yet**. It keeps them out of the "Friend" bucket while marking them as active investment. A Building person should get a `Follow Up Frequency` so the bump scan can keep the energy going. When they graduate, move them to Friend.

**Follow Up Frequency (cadence)** — the per-person "how often do I want to reach out" dial. Each choice maps to a number of days; the **bump scan** computes "due for a bump" as `daysSince(Last Contacted) >= cadenceDays` (never-contacted = always due). Mapping:

| Follow Up Frequency | Days |
|---|---|
| Weekly | 7 |
| Every 2 weeks, Frequent | 14 |
| Monthly check-in, Occasional text | 30 |
| Every 6 weeks | 42 |
| Occasional drinks, Brunch | 45 |
| Quarterly | 90 |
| Twice a year | 182 |
| Annual / Life events | 365 |
| As needed | never auto-surfaces |
| (unset) | default 30 for Building people; otherwise not scanned |

The clean interval buckets (Weekly / Every 2 weeks / Every 6 weeks / Twice a year) were added 2026-06-13. The older vibe labels (Occasional drinks, Brunch, etc.) are mapped to best-guess day counts — adjust the mapping if the user corrects them.

### Groups (look up table ID via metadata)
Represents clubs, running groups, social groups, etc.

| Field Name | Type | Purpose |
|---|---|---|
| Name | singleLineText | Group name (primary field) |
| Type | singleSelect | e.g., Running Club, Social Group, Work, etc. |
| Members | linkedRecord → People | Reverse link — populated automatically when People.Groups is set |

**CRITICAL: Clubs are Groups.** When someone's record mentions any recurring named club or group, that information belongs in the Groups linked record field — NOT in Notes, NOT in any text field. Look up the Group record by name, or create it if it doesn't exist, then link it.

### Interactions (look up table ID via metadata — has description set)
Logs real-world contact events between the user and a person already in People.

**LANGUAGE TRIGGER:** When the user casually says "interaction" — e.g. "log an interaction", "that was an interaction", "update the interaction" — treat it as a direct instruction to create a record in this table. An "interaction" in the user's vocabulary = a real-world contact event (in-person meetup, call, text exchange, etc.) that he is reporting having had with an existing CRM contact.

| Field Name | Type | Purpose |
|---|---|---|
| Person | linkedRecord → People | Who the interaction was with |
| Date | date | When it happened |
| Summary | multilineText | What happened (free text) |
| Type | singleSelect | Contact method — valid options: check current schema; "In Person" is confirmed valid |

**CRITICAL: Only create an Interaction record when the user reports having real-world contact with an existing person** (a text, call, meeting, etc.). Do NOT create Interaction records when:
- Adding a new person to the CRM (even if the user says when they last talked)
- Adding a Roster (a name dump is not contact with anyone)
- Running data quality tasks
- Making schema changes

The Changelog covers record additions. Interactions cover contact events.

### Rosters (<TBL_ROSTERS_ID>)
Name-dump records — the user calls them "globs". One record = one list of names from an event, sign-up list, or social circle, kept as TEXT, not contacts. Most names on a roster will never become People records; the roster preserves them so recurrence is detectable months or years later.

**LANGUAGE TRIGGER:** When the user says "glob this", "add a roster", "here's the sign-up list", or dumps a list of names with event context — create ONE Rosters record. Do NOT create People records or Interactions for the names on it.

| Field Name | Type | Purpose |
|---|---|---|
| Roster Name | singleLineText | Short label naming the source or host, the event, and the date |
| Date | date | When the event/list happened (YYYY-MM-DD) |
| Source | singleSelect | Event sign-up, Met in person, Online list, Friend's circle, Other |
| Names | multilineText | The dump. One name per line; optional context note after " — ". Nicknames welcome. |
| Anchor Group | linkedRecord → Groups | The club/group touchstone |
| Anchor Person | linkedRecord → People | The friend whose circle this is |
| Event | linkedRecord → Events | The Events record this corresponds to, when one exists |
| Notes | multilineText | Event context, vibes, anything not per-name |
| Promoted People | linkedRecord → People | Names from this roster that later became real contacts — the back-reference |

**Anchoring rule:** every roster links AT LEAST ONE of Anchor Group, Anchor Person, or Event; any combination is fine. An Event alone is a valid anchor (a party roster with no club or specific friend). Look up or create the anchor record before saving the roster.

### Changelog (look up table ID via metadata)
Audit log of changes made to the CRM.

| Field Name | Type | Purpose |
|---|---|---|
| Summary | singleLineText | One-line description of what changed |
| Changes Made | multilineText | Detail of what was added/updated |
| Tables Affected | multipleSelects | Which tables were touched (pass names as strings with typecast: true if a new table name is needed) |
| Records Added | number | Count of new records created |
| Records Updated | number | Count of existing records patched |
| Date | date | When the change was made (use today's date) |

Create a Changelog record for: adding new people, adding rosters, bulk updates, schema changes. Do NOT create Changelog records for routine data quality corrections (moving data between fields on the same record).

---

## Standard Operations

### Adding a new person
Create 2 records in one javascript_tool call:
1. **POST to People** — populate all known structured fields. Never leave data in Notes if a proper field exists.
2. **POST to Changelog** — summarize the addition.

Do NOT create an Interaction record.

If the person has a club/group: look up the Group record by name first, then link it in the People POST.

**Roster cross-reference (do this on every add):** after creating the person, search Rosters.Names for their name(s) — real name, community name, nicknames. If they appear on rosters, link the new People record into each roster's Promoted People field, and tell the user the history (e.g., "they've been on 4 sign-up lists since January — set Known Since accordingly?"). Use the EARLIEST roster Date as the Known Since candidate and the roster's anchor as How We Met context.

### Adding a roster ("glob this")
Trigger: the user dumps a list of names with event context, says "glob this", "add a roster", "here's the sign-up list", etc. Often arrives via dispatch from his phone on the way to or from an event — the procedure must work unattended.
1. Look up (or create) the anchor: Group (club/group), Person (the friend), and/or Event. At least one.
2. **POST to Rosters** — Names as given, one per line; Date; Source; Notes for event context.
3. Quick scan: check whether any roster name is already a People contact (search People by name/community name). If yes, mention it to the user — but do NOT auto-link or create anything beyond the roster.
4. **POST to Changelog** (1 record added, Tables Affected: Rosters).
Do NOT create People records or Interactions for roster names.

### Recurrence scan (on demand only)
Trigger: the user asks "who keeps showing up", "run the roster scan", "any repeat names", etc. Never scheduled.
1. Fetch all Rosters (Names, Date, anchors).
2. Normalize names client-side: lowercase, strip any "Just " name prefix, collapse whitespace.
3. Count appearances per normalized name across rosters; also match each against People (Name, Community Name, Nickname / Aliases).
4. Report: (a) names appearing on 3+ rosters that are NOT in People — promotion candidates, with their roster dates and anchors; (b) names already in People but not linked in those rosters' Promoted People — offer to link.
5. Only act (promote/link) on the user's confirmation.

### Bump scan — who's due for a reach-out (on demand)
Trigger: the user asks "who's due for a bump", "who's overdue", "who should I reach out to", "who am I letting go cold", "run the bump scan", etc.A scheduled weekly automation can do this for the Building tier; this command is the on-demand, any-tier version.

1. Decide scope from the user's phrasing:
   - Default / "warm prospects" / "people I'm building with" → filter Relationship Tier = **Building** (`<CHOICE_ID>`).
   - "everyone" / "all my friends" → no tier filter (scan all People with a Follow Up Frequency set).
   - A named tier → filter that tier.
2. `list_records_for_table` (People, `<TBL_PEOPLE_ID>`), fields: Name, Last Contacted (`<FLD_LAST_CONTACTED_ID>`), Follow Up Frequency (`<FIELD_ID>`), Location, Phone, Relationship Tier. `pageSize` 200.
3. Compute due-ness client-side using the **Follow Up Frequency → days** mapping in the People-table section above:
   - `cadenceDays` from the choice name; `As needed` → skip; unset → 30 for Building, else skip.
   - `daysSinceContact = today − Last Contacted`; blank Last Contacted → never contacted (always due, top priority).
   - **Due** if never contacted OR `daysSinceContact >= cadenceDays`.
4. Report due people sorted never-contacted-first, then most-overdue first (`daysSinceContact − cadenceDays`). Show name, cadence, last-contact date + days overdue, and Phone so the user can act. This is a READ-ONLY report — do not create or modify records. When the user later says he reached out, that's a normal **interaction** log (which updates Last Contacted and clears the overdue state).

### Promoting a roster name to a contact
When the user confirms a recurring name deserves a record:
1. Create the People record. Known Since = earliest roster Date they appear on; How We Met = that roster's context; Groups = the roster's Anchor Group(s) if club-sourced.
2. Link the new person into Promoted People on EVERY roster where the name appears. Keep their name line in each roster's Names text — it is the historical record, not a duplicate.
3. Changelog entry (1 added to People, N rosters updated).

### Logging an interaction
Trigger: the user says "I talked to X", "texted Y", "had lunch with Z", **or uses the word "interaction" in any casual form** ("log an interaction", "that was an interaction", etc.). Create 1 Interaction record. Optionally update Last Contacted on the Person record. No Changelog needed.

### Looking up a person

**Preferred: `search_records` MCP tool** — free-text fuzzy search with token-order independence. Handles typos and partial names automatically.
```
<airtable-mcp>__search_records({
  baseId: "<YOUR_BASE_ID>",
  table: "People",
  query: "<name>",
  fields: "ALL_SEARCHABLE_FIELDS"
})
```

If `search_records` returns nothing and the user asserts the person exists, do NOT conclude they're missing. Fall back: `list_records_for_table` with no filter (paginate via `cursor`), then JS-side regex match across all field values. JS-side scanning skips empty fields naturally (they're `undefined`) and has no hidden failure modes.

**Also search Rosters.** A person not in People may still be on rosters — run the same query against the Rosters table (Names field). Roster hits are prior co-presence history; report them ("on the May 3 and May 17 sign-up lists") even when no contact record exists.

**Chrome-fallback path — formula gotcha.** If you're using the Chrome `javascript_tool` fetch() path for any reason, Airtable's REST `filterByFormula` has a silent failure mode: `SEARCH()` and `LOWER()` can error on `BLANK()` haystacks, and a single errored row inside an `OR()` branch drops that row from results with no diagnostic. Coerce every field ref with `&""` and prefer `FIND` over `SEARCH`:

```
OR(
  FIND(LOWER("<name>"), LOWER({Name}&"")),
  FIND(LOWER("<name>"), LOWER({Community Name}&"")),
  FIND(LOWER("<name>"), LOWER({Nickname / Aliases}&""))
)
```

This is how a known-existing record (a record with a Community Name that shares no tokens with the legal name) returned zero hits in May 2026 before this rule was documented. The MCP `search_records` path does not have this problem — it uses Airtable's indexed full-text search, not formula evaluation.

### Updating a field
PATCH the People record. If moving data out of Notes into a proper field: set the target field AND clear Notes (or remove just that portion from Notes). Never overwrite a non-empty field — flag it for human review instead.

### Linking a group
1. Search Groups: `filterByFormula={Name}="<group name>"`
2. If not found: POST a new Group record
3. PATCH People record: `{ fields: { "Groups": [{ id: groupRecordId }] } }`

---

## Data Quality Rules

- **Notes is a catch-all and should be minimized.** Any time you read a Notes field, check if the content belongs in a structured field and move it.
- **Never overwrite a field that already has a value** without explicit instruction from the user. Flag it instead.
- **Clubs belong in Groups** (linked record) — never store club names as text anywhere.
- **Name dumps belong in Rosters** — never bulk-create People records from a list of names, and never paste a name list into a person's or group's Notes.
- **Job info** belongs in Job/Employer field — if that field doesn't exist yet, check schema first.
- **Addresses** → Location field
- **Birthdays** → Birthday field (YYYY-MM-DD)
- **When they met** → Known Since field and/or How We Met field
- **Family / partner / kids / pets** → Family and Pets field (multiline, one per line)

---

## Telegram Notifications
Send summaries to the user via Telegram when useful (batch operations, scheduled tasks, dispatch runs).

**Preferred: the Telegram MCP tool** `mcp__Telegram_Notifier__send_message` (load via `ToolSearch { query: "select:mcp__Telegram_Notifier__send_message" }` if absent). It is the proven path from sandboxed/scheduled runs.

**Fallback (direct HTTP — has returned 401 from some sandboxes; verify before relying on it):**
```js
const tgSend = (text) => fetch(`https://api.telegram.org/bot<YOUR_TELEGRAM_BOT_TOKEN>/sendMessage`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ chat_id: '<YOUR_TELEGRAM_CHAT_ID>', text })
}).then(r => r.json());
```
