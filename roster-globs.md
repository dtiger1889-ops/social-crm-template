# Rosters ("globs") — name-dump records for the Airtable Social CRM
Design note, 2026-06-12: why a pile of names from an event is stored as one list record instead of a contact per name, and how intake, lookup, recurrence and promotion use it. Table setup is in `SETUP.md` and `schema.json`; the step-by-step agent procedures are in `SKILL.md`.

> **Later change:** this design predates auto-matching. The shipped skill now AUTO-LINKS high-confidence *existing* contacts on roster intake and on the recurrence scan (see the auto-match procedure and confidence bar in `SKILL.md`). Manual confirmation is still required only to PROMOTE a brand-new name to a People record. The rest of this design (one record per list, text-blob names, anchors, promotion) stands.

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
3. **Recurrence scan (on demand only; no scheduled report — the usual trigger is asking around an event).** Fetch all Rosters, normalize names client-side (lowercase, strip "Just", match against People.Name/Community Name/Nicknames), count appearances per name across rosters. Report: names appearing ≥3 times not in People (promotion candidates), and names already in People (link suggestion). Spans months/years by design — rosters never expire.
4. **Promotion.** the user says yes → create People record: How We Met = earliest roster context, Known Since = earliest roster Date, Groups = anchor Group(s), Member Status if club-sourced. Link the new person into Promoted People on EVERY roster where the name appears (keep the name line in Names text — it's the historical record). Changelog entry.

### Why text-blob, not per-name records
- the user's explicit requirement: "the list exists as a single record."
- 80-name sign-up lists × weekly events = thousands of junk records under a junction model; Airtable free-tier record limits and UI noise both punish that.
- `search_records` full-text indexes multilineText, so name lookup across rosters works without per-name rows.
- Cost: no native Airtable rollup of "appearance count" — the recurrence scan is a Claude-run query, which fits how the CRM is operated anyway (everything goes through the skill).

## Tuning
- **Promotion threshold.** The recurrence report lists names that appear on 3 or more rosters and are not yet in People. Drop it to 2 if you want candidates sooner.
