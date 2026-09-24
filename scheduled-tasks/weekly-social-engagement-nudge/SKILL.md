---
name: weekly-social-engagement-nudge
description: Every Tuesday at 6 PM. (A) Always — scan "Building" tier contacts (people the user is turning into friends) and flag any overdue for a bump per their personal Follow Up Frequency cadence. (B) If this coming Saturday has no social event on Google Calendar, also pull 5 Local friends most overdue for contact. Send the user one Telegram combining whichever sections apply.
---

> SHARED SCHEMA NOTE: this run reads/writes the user's Social CRM (Airtable base `<YOUR_BASE_ID>`). If you change how it touches the CRM, or notice a schema change, reconcile the "Automated runs that depend on this CRM" registry in your social-crm skill file.

You are running the user's weekly social engagement check. Today is Tuesday. Follow these steps exactly.

This task uses **the Airtable MCP server directly** for CRM access — NO Chrome needed. The Airtable MCP works in unattended scheduled runs without permission prompts.

There are **two independent checks**: the **Building bump scan (STEP 1) runs every week no matter what**, and the **weekend-social nudge (STEPS 2–3) only runs if Saturday is open**. They are combined into one Telegram in STEP 4.

If the Airtable MCP tools aren't loaded:
```
ToolSearch { query: "select:mcp__<airtable-mcp-server-id>__list_records_for_table", max_results: 1 }
```

---

## STEP 1 — Building-tier bump scan (ALWAYS runs)

"Building" is the Relationship Tier for people the user is actively turning into friends — not friends yet, but he wants to keep the energy up. Each has a personal cadence in the **Follow Up Frequency** field. This step flags the ones who've gone quiet past *their own* cadence.

Call `mcp__<airtable-mcp-server-id>__list_records_for_table` with:
- `baseId`: `<YOUR_BASE_ID>`
- `tableId`: `<TBL_PEOPLE_ID>` (People)
- `filters`: `{operator: "and", operands: [{operator: "isAnyOf", operands: ["<FIELD_ID>", ["<CHOICE_ID>"]]}]}` (Relationship Tier = Building)
- `fieldIds`: `["<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>"]` (Name, Last Contacted, Follow Up Frequency, Location, Phone)
- `pageSize`: 200

For each returned person, compute whether they're **due for a bump**, client-side:

1. **Cadence → days** (map the Follow Up Frequency choice name):
   - `Weekly` → 7
   - `Every 2 weeks`, `Frequent` → 14
   - `Monthly check-in`, `Occasional text` → 30
   - `Every 6 weeks` → 42
   - `Occasional drinks`, `Brunch` → 45
   - `Quarterly` → 90
   - `Twice a year` → 182
   - `Annual / Life events` → 365
   - `As needed` → **skip this person entirely** (no auto-surfacing)
   - **unset / blank** → default 30 (a warm prospect with no cadence still surfaces; note "no cadence set")
2. **Days since contact**: `today − Last Contacted`. If Last Contacted is blank → treat as **never contacted** (always due, highest priority).
3. **Due** if never contacted, OR `daysSinceContact >= cadenceDays`.
4. Sort the due people: never-contacted first, then by **most overdue** (`daysSinceContact − cadenceDays`) descending. Cap at **7**.

Hold this list for STEP 4. If nobody is due, the bump section is simply omitted.

**Choice-ID drift:** if the Building filter returns nothing unexpectedly, re-fetch IDs via `get_table_schema` with `tables: [{tableId: "<TBL_PEOPLE_ID>", fieldIds: ["<FIELD_ID>", "<FIELD_ID>"]}]` and retry once. Building's ID as of 2026-06-13 is `<CHOICE_ID>`.

---

## STEP 2 — Check Google Calendar for Saturday (gates STEP 3 only)

Calculate this coming Saturday's date. Use the Google Calendar MCP (`mcp__<gcal-mcp-server-id>__list_events`) to fetch events on Saturday:
- calendarId: "primary"
- startTime: Saturday at 00:00:00 local time (America/New_York)
- endTime: Saturday at 23:59:59 local time
- timeZone: "America/New_York"

A "social event" is any event that involves other people — dinner, party, drinks, brunch, birthday, group run, social gathering, concert with friends, etc. Exclude: solo workouts, work events, appointments, errands, reminders, all-day informational events (reminder-style entries with no attendees).

- If Saturday has at least one social event → **skip STEP 3** (no weekend list); note the event name for STEP 4.
- If Saturday is free → continue to STEP 3.

(Note: this only gates the weekend list. The STEP 1 bump scan already ran regardless.)

## STEP 3 — Pull 5 Local friends from the CRM (only if Saturday is open)

**Filtering logic:** The CRM's `Tags` multipleSelects field has a **"Local"** option (`<CHOICE_ID>`) that marks local people the user can casually hang out with via public transit / Uber. This is the source of truth for "people I could realistically meet up with this weekend" — NOT a location-string match. The associated view in Airtable is "DC Social Targets". Only pull from the Local-tagged subset.

Call `mcp__<airtable-mcp-server-id>__list_records_for_table` with:
- `baseId`: `<YOUR_BASE_ID>`
- `tableId`: `<TBL_PEOPLE_ID>` (People)
- `filters`: structured, AND of two conditions:
  1. Tags `hasAnyOf` `["<CHOICE_ID>"]` (the Local choice ID, on field `<FIELD_ID>`)
  2. Relationship Tier `isAnyOf` `["<CHOICE_ID>", "<CHOICE_ID>"]` (Friend, Close Friend on field `<FIELD_ID>`)
- `fieldIds`: `["<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>"]` (Name, Last Contacted, Location, Follow Up Frequency)
- `pageSize`: 200

The MCP server's sort param has been unreliable; sort client-side instead. Sort the records so `Last Contacted` ascending — records with **no** Last Contacted (never contacted) come first, then oldest to newest. Take the first 5.

Concrete example sort: records with `<FIELD_ID> === undefined` first, then sorted by date ascending.

**Choice ID verification:** if the Local tag choice ID changes (the user renames it, or you suspect a stale ID), call `get_table_schema` with `tables: [{tableId: "<TBL_PEOPLE_ID>", fieldIds: ["<FIELD_ID>", "<FIELD_ID>"]}]` to re-fetch current choice IDs. Same if Relationship Tier choices shift.

**If the Airtable MCP errors or returns nothing on either pull:** retry the call once (transient connector hiccups happen), and re-fetch choice IDs via `get_table_schema` if a filter returned zero records unexpectedly. If it still fails, send a Telegram: "weekly-social-engagement-nudge: Airtable MCP failed — couldn't pull the CRM. Reach out to someone this week anyway!" and exit. Do NOT fall back to Chrome or direct API calls — the MCP connector is the only Airtable path for this task.

## STEP 3.5 — Attach conversation openers (Follow-up Hooks)

The point: the nudge should say *what* to open with, not just *who*. When the user logs an interaction, forward-looking topics go into the Interactions field **Follow-up Hooks**, one per line. This step reads them back for the people you are about to surface.

1. Collect the record IDs of every person who will be SHOWN in Section A and Section B. If that set is empty, skip this step.
2. One call to `mcp__<airtable-mcp-server-id>__list_records_for_table`:
   - `baseId`: `<YOUR_BASE_ID>`
   - `tableId`: `<TBL_INTERACTIONS_ID>` (Interactions)
   - `fieldIds`: `["<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>", "<FIELD_ID>"]` (Person, Date, Summary, Follow-up Hooks)
   - `sort`: `[{fieldId: "<FIELD_ID>", direction: "desc"}]` (the Date field; sort takes field IDs, a field name returns an error)
   - `pageSize`: 200
3. For each shown person, look at their **three most recent** interactions (Person contains their record ID). Take the hooks from the most recent one whose Follow-up Hooks is non-blank. Drop any line that starts with `(resolved`. Keep at most 2 hooks.
4. Hold them for STEP 4 as that person's opener line. A person with no hooks gets no opener line; never invent one from Summary, Notes, or anything else.

If this call fails, send the nudge without openers and say so in the STEP 5 report. Openers are an extra, never a reason to skip the Telegram.

## STEP 4 — Compose and send ONE Telegram via Telegram MCP

Send via the `telegram-notifier` MCP extension. Bot token + chat ID live in Windows Credential Manager — you never see them.

1. If `mcp__Telegram_Notifier__send_message` is in your active tool list, call it directly.
2. If not: `ToolSearch { query: "select:mcp__Telegram_Notifier__send_message", max_results: 1 }`, then call it.
3. Signature: `mcp__Telegram_Notifier__send_message({ text, parse_mode?: "Markdown" })`.
4. If ToolSearch doesn't return the tool, log the contents in your final report with a "couldn't deliver to Telegram" note and exit.

Assemble the message from whichever sections apply:

**Section A — Building bumps (include only if STEP 1 found due people):**
```
🌱 *Warm prospects due for a bump*

People you're building with who've gone quiet past their cadence:

1. *<Name>* — <Follow Up Frequency or "no cadence set">; last contact <YYYY-MM-DD> (<N>d ago, <M>d overdue)
2. ...
```
For never-contacted, write `never contacted yet — kick it off` instead of the date/overdue part.
**Opener line (Sections A and B):** if STEP 3.5 found hooks for a shown person, add one indented line directly under their entry: `   💬 Ask about: <hook>; <hook> (from <YYYY-MM-DD>)`.

**Section B — Weekend social (include only if Saturday was open, STEP 3 ran):**
```
🍹 *Weekend Social Check — Saturday is open!*

Your calendar is free Saturday. Here are 5 friends to reach out to for Friday drinks or Saturday brunch:

1. *<Name>* — last contact: <YYYY-MM-DD> · <Follow Up Frequency>
   📍 <Location>
2. ...

_Reach out by Wednesday to lock something in._
```
If Last Contacted is null, write `never contacted`. If Follow Up Frequency is null, omit the ` · <freq>` segment.

**Combining rules:**
- Both sections apply → send Section A, then a blank line, then Section B.
- Only one applies → send just that one.
- **Neither applies** (no bumps due AND Saturday is booked) → send a short confirmation: `✅ You're already social this Saturday — <event name>. And no warm-prospect bumps are due. Nothing to chase this week!`

## STEP 5 — Report

Brief summary of what was sent: the Building bump names (if any) and the 5 weekend names (if sent), plus how many got an opener line (STEP 3.5).

## Notes
- **Airtable MCP is the ONLY CRM path** — it works in unattended runs without Chrome or permission prompts. The former Chrome/fetch() fallback was removed 2026-06-04 along with the inline API token.
- Google Calendar MCP tools are named `mcp__<gcal-mcp-server-id>__*` (the server id is a UUID specific to your claude.ai connector; find it in your tool list).
- Airtable MCP tools are named `mcp__<airtable-mcp-server-id>__*`.
- The cadence→days mapping in STEP 1 mirrors the on-demand "bump scan" in the social-crm skill; keep the two in sync if either changes.
