---
name: weekly-changelog-consolidation
description: Consolidate the past week's individual Changelog records in Airtable into a single summary record, then delete the originals to conserve free-tier record space.
---

> SHARED SCHEMA NOTE: this run reads/writes the user's Social CRM (Airtable base `<YOUR_BASE_ID>`). If you change how it touches the CRM, or notice a schema change, reconcile the "Automated runs that depend on this CRM" registry in your social-crm skill file.

Consolidate the past week's Airtable Changelog records into one summary record, then delete originals to conserve free-tier space.

**This task uses the Airtable MCP connector directly — NO Chrome, NO inline API token.** The connector works in unattended scheduled runs without permission prompts.

**Airtable context:**
- Base ID: `<YOUR_BASE_ID>`
- Changelog table ID: `<TBL_CHANGELOG_ID>`
- Field IDs: Summary `<FIELD_ID>`, Date `<FIELD_ID>`, Changes Made `<FIELD_ID>`, Tables Affected `<FIELD_ID>`, Records Added `<FIELD_ID>`, Records Updated `<FIELD_ID>`
- Airtable MCP tools are named `mcp__<airtable-mcp-server-id>__*`. If not in your active tool list, load via ToolSearch, e.g. `ToolSearch { query: "select:mcp__<airtable-mcp-server-id>__list_records_for_table,mcp__<airtable-mcp-server-id>__create_records_for_table,mcp__<airtable-mcp-server-id>__delete_records_for_table", max_results: 3 }`.

**CRITICAL — never consolidate a summary record (no nested rollups):** This task writes its consolidated record back into the same Changelog table, dated with the current date, so that record falls inside next week's 7-day window. It must NOT be swept into a future consolidation, or summaries pile on summaries. This is a two-tier system: weekly consolidations are tagged `[consolidated]` and a separate MONTHLY task rolls those up into records tagged `[monthly]`. This weekly task must skip BOTH tags. Summary records are never re-consolidated by this task.

**Steps:**

1. **Fetch the last 7 days of Changelog records.** Call `list_records_for_table` with baseId `<YOUR_BASE_ID>`, tableId `<TBL_CHANGELOG_ID>`, filtering on the Date field (`<FIELD_ID>`) for records on or after (today − 7 days). If the structured filter is unreliable for date comparison, fetch all records (pageSize 200, paginate if needed) and filter client-side.

2. **Exclude summary records (the key fix).** From the fetched records, drop any record that is itself a weekly or monthly summary produced by the automation. A record is a summary if ANY of these hold:
   - its Summary (`<FIELD_ID>`) starts with `Weekly consolidation` or `Monthly consolidation` (case-insensitive), OR
   - its Changes Made (`<FIELD_ID>`) contains the marker `[consolidated]` or `[monthly]`.
   The remaining records are the "originals" to consolidate. Never consolidate or delete a summary record in this task.

3. **Skip guard.** If fewer than 2 originals remain after exclusion, send a Telegram saying consolidation was skipped (include the count of originals found and the count of summary records skipped) and exit. Do NOT create a consolidated record from a single original or from zero originals.

4. **Build the consolidated record.** Sort the originals by Date ascending and compute:
   - Summary: `Weekly consolidation — <earliest date> to <latest date>`
   - Changes Made: first line must be exactly `[consolidated]` (this is the skip marker for future runs), followed by one bullet per original record in the form `• <Date>: <Changes Made or Summary or "(no detail)">`, joined with newlines. If an original's Changes Made already contains `[consolidated]` or `[monthly]`, it should never have reached this step — re-check step 2.
   - Tables Affected: union of all originals' Tables Affected values
   - Records Added: sum across originals
   - Records Updated: sum across originals

5. **Create the consolidated record** via `create_records_for_table` (same base/table). Confirm the response contains a new record ID before proceeding. The new record's Changes Made must contain the `[consolidated]` marker so future runs skip it. If creation fails, do NOT delete anything; Telegram the error and exit.

6. **Delete the originals** via `delete_records_for_table`, passing ONLY the IDs of the original records consolidated in step 4. NEVER pass the new consolidated record's ID, and NEVER pass the IDs of the summary records excluded in step 2. Only delete after the create is confirmed.

7. **Report.** Send a Telegram message (`mcp__Telegram_Notifier__send_message`; load via `ToolSearch { query: "select:mcp__Telegram_Notifier__send_message", max_results: 1 }` if missing) with: originals consolidated, date range, records freed up, and the count of summary records skipped (so the nesting fix is visibly working). If the Telegram tool is unavailable, log the summary in your final report instead — do not fall back to raw HTTP.

**Error handling:** If any Airtable MCP call fails, send a Telegram describing what failed at which step and exit. Do NOT fall back to Chrome or direct API calls — the MCP connector is the only Airtable path for this task.