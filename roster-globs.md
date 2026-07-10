# Rosters ("globs") — name-dump records for the Airtable Social CRM
Status: applied then archived (2026-06-12)
Created: 2026-06-12

## Problem
After an event the user has a pile of names (an event sign-up list, six friends-of-a-friend at a bar) that do NOT each deserve a People record — most are one-offs. Today the only options are "create a full contact" (pollutes People) or "lose the names" (loses the recurrence signal when the same name shows up months later).

## Goal
A single record can hold a dumped list of names + event context, anchored to its social touchstone (a club Group or a friend's People record). When a name from old rosters reappears, it is findable, and a recurring name can be promoted into a real People record carrying its history.

## Non-goals
- NOT one record per name. The list is one record; names live as text inside it. No junction/"mentions" table in v1.
- NOT automatic promotion. Recurrence detection surfaces candidates; the user decides who becomes a contact.
- NOT a replacement for the Events table — a Roster can link to an Event but sign-up lists with 80 names the user never met are not "events he logged."
- NOT retroactive backfill of old data (can be done later as a separate task).

## Decision / design

### Naming
Table name: **Rosters**. ("Glob" works in conversation; "Roster" reads correctly in Airtable — a list of names tied to a context.) Rejected alternates: Sightings (sounds per-person), Name Dumps (ugly in UI), Cohorts (implies analysis).

### New table: Rosters
| Field | Type | Purpose |
|---|---|---|
| Roster Name | singleLineText (primary) | Short label naming the source or host, the event, and the date |
| Date | date | When the event/list happened |
| Source | singleSelect: Event sign-up, Met in person, Online list, Friend's circle, Other | How the names were captured |
| Names | multilineText | The dump. One name per line; optional context note after " — ". Nicknames welcome. |
| Anchor Group | link → Groups | The club/group touchstone (the run-club case) |
| Anchor Person | link → People | The friend touchstone (the "went out with a friend, met their six friends" case) |
| Event | link → Events | Optional, when the roster corresponds to a logged Event |
| Notes | multilineText | Event context, vibes, anything not per-name |
| Promoted People | link → People | Names from this roster that later became real contacts — the back-reference |

At least one anchor (Group, Person, or Event) must be set; any combination allowed. An Event is a first-class anchor on its own — a party roster can hang directly off an Events record with no Group or Person involved (confirmed by the user 2026-06-12).

### Workflows
1. **Intake.** "Glob this:" + names + context → one Rosters record. Look up/create the anchor (Group, Person, and/or Event) and link it. One Changelog entry. NO People records, NO Interactions.
   - **Primary intake channel is Cowork dispatch:** the user will typically send the roster from his phone on the way to a party or event. Intake must work unattended — the Airtable MCP path already does (no Chrome, no permission prompts). The skill doc must make the intake rule self-sufficient for a cold dispatch session.
2. **Cross-reference on every People add/lookup.** When adding or looking up a person, also `search_records` against Rosters.Names (and match People.Community Name where relevant). Hits = prior co-presence: "this name is on 4 run-club sign-up lists since January." Surface it; this is the "I'm seeing this person a lot" realization.
3. **Recurrence scan (on demand, optionally scheduled).** Fetch all Rosters, normalize names client-side (lowercase, strip "Just", match against People.Name/Community Name/Nicknames), count appearances per name across rosters. Report: names appearing ≥3 times not in People (promotion candidates), and names already in People (link suggestion). Spans months/years by design — rosters never expire.
4. **Promotion.** the user says yes → create People record: How We Met = earliest roster context, Known Since = earliest roster Date, Groups = anchor Group(s), Member Status if club-sourced. Link the new person into Promoted People on EVERY roster where the name appears (keep the name line in Names text — it's the historical record). Changelog entry.

### Why text-blob, not per-name records
- the user's explicit requirement: "the list exists as a single record."
- 80-name sign-up lists × weekly events = thousands of junk records under a junction model; Airtable free-tier record limits and UI noise both punish that.
- `search_records` full-text indexes multilineText, so name lookup across rosters works without per-name rows.
- Cost: no native Airtable rollup of "appearance count" — the recurrence scan is a Claude-run query, which fits how the CRM is operated anyway (everything goes through the skill).

### Implementation steps (when approved)
1. `create_table` Rosters in base <YOUR_BASE_ID> with fields above (link fields via `create_field` after, since multipleRecordLinks needs linkedTableId).
2. Set table description encoding the intake rule (no People creation from rosters).
3. Update the social-crm skill AT THE ACCOUNT LEVEL: it is a claude.ai account skill, managed at `https://claude.ai/customize/skills` ("..." menu: Edit with Claude / Replace / Download). Base the edit on the current account copy (Download it, or Edit with Claude directly); the synced local copy under `%APPDATA%\...\skills-plugin\` is a cache, and `Cowork/social-crm.skill` is a stale one-time upload artifact — neither is an edit target. Account sync delivers the update to interactive, dispatch, and scheduled sessions automatically. Additions: Rosters table doc, the "glob this" intake trigger, the cross-reference step in person lookup/add, the recurrence-scan and promotion procedures. Full model: `cowork_mgmt/cowork-sandbox-model.md` > "Account-level (claude.ai) custom skills".
4. Changelog entry for the schema change.
5. Smoke test: create one real roster from the user's next event, run a lookup against it.

## Resolved questions (2026-06-12)
- Recurrence scan: **on-demand only**, no scheduled report. Typical trigger: the user asks via Cowork dispatch around an event.
- Event as anchor: **yes**, first-class, can be the sole anchor.
- Skill source (corrected 2026-06-12 after docs review + claude.ai UI verification): social-crm is a claude.ai ACCOUNT skill — source of truth and edit surface is `https://claude.ai/customize/skills`, which syncs to all session types automatically. The `Cowork/social-crm.skill` zip is a one-time upload artifact (nothing installs from it; documented as legacy in Cowork/CLAUDE.md v5). No zip re-pack is part of this work. Dispatch was never running an outdated skill — the earlier "drift" worry was based on the wrong mechanism model.

## Open questions
- Promotion threshold for the scan report: default ≥3 appearances — the user may want ≥2.

## Closeout - DO NOT DELETE, fires when this spec's work is executed
This spec does not just sit in specs/. When the work is done, ONE of:
- [ ] ARCHIVE: move this file to social_event_planning/decisions/ (or archive/) - it is now a
  record of a made decision. Flip Status: archived.
- [x] PROMOTE: shipped 2026-06-12 — Rosters table created in Airtable (<TBL_ROSTERS_ID>, Changelog <REC_ID>) + social-crm skill updated in the live local store (canonical content: specs/skill_build/social-crm/SKILL.md). Residual: claude.ai account copy still stale (open thread in CHECKPOINT). Status flipped: applied then archived.
Until one box is checked, this spec is UNFINISHED.
