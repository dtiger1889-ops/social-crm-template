---
name: monthly-crm-data-quality-scan
description: Monthly scan of CRM People records for misplaced data — moves content from Notes into structured fields and flags conflicts for review
---

> SHARED SCHEMA NOTE: this run reads/writes the user's Social CRM (Airtable base `<YOUR_BASE_ID>`). If you change how it touches the CRM, or notice a schema change, reconcile the "Automated runs that depend on this CRM" registry in your social-crm skill file.

Run the monthly CRM data quality scan against the user's Social CRM in Airtable. Use the social-crm skill for schema conventions and field semantics.

**This task uses the Airtable MCP connector directly — NO Chrome, NO inline API token.** The connector works in unattended scheduled runs without permission prompts.

**Airtable context:**
- Base ID: `<YOUR_BASE_ID>`
- People table ID: `<TBL_PEOPLE_ID>` — key fields: Name `<FIELD_ID>`, Notes `<FIELD_ID>`, Location `<FIELD_ID>`, Birthday `<FIELD_ID>`, Job/Employer `<FIELD_ID>`, How We Met `<FIELD_ID>`, Known Since `<FIELD_ID>`, Family and Pets `<FIELD_ID>`, Groups `<FIELD_ID>`, History `<FIELD_ID>`
- Interactions table ID: `<TBL_INTERACTIONS_ID>` — fields: Summary `<FIELD_ID>`, Date `<FIELD_ID>`, Type `<FIELD_ID>`, Notes `<FIELD_ID>`, Person `<FIELD_ID>`
- Groups table ID: `<TBL_GROUPS_ID>`
- Airtable MCP tools are named `mcp__<airtable-mcp-server-id>__*` (list_records_for_table, update_records_for_table, delete_records_for_table, get_table_schema). Load via ToolSearch if missing. If field IDs above seem stale, re-fetch with `get_table_schema` before patching.

**Part 1 — Data quality scan:**

1. Fetch ALL People records via `list_records_for_table` (pageSize 200, paginate with offset until done).
2. For each record:
   - Scan Notes for content that belongs in a structured field: Location, Birthday, Job/Employer, How We Met, Known Since, Family and Pets
   - Scan any text fields for club/group names that should be linked records in Groups
   - Use `update_records_for_table` to move misplaced data to the correct field and clear it from Notes (batch updates, up to 10 records per call)
   - If a target field is already non-empty, skip and add to the flags list — never overwrite
3. Track for the report: # records scanned, # records patched, what was moved, and any flags needing manual review

**Part 2 — History field consolidation (Interactions older than 6 months):**

The People table has a **History** field (multilineText) serving as a running archive of older interaction history.

4. Fetch ALL Interaction records via `list_records_for_table` on `<TBL_INTERACTIONS_ID>`; keep those with Date more than 6 months ago.
5. Group those interactions by their linked Person record.
6. For each Person with old interactions:
   - Build a summary line per interaction: `[YYYY-MM-DD] <Type>: <Summary or Notes snippet>`
   - Append those lines to the Person's History field (preserve existing content — always append, never overwrite)
   - PATCH the People record via `update_records_for_table` with the updated History value
7. Once History is successfully updated for a person, delete the consolidated old Interaction records via `delete_records_for_table` — only after confirming the update succeeded.

**Report:**

8. Send a Telegram summary via `mcp__Telegram_Notifier__send_message` (load via `ToolSearch { query: "select:mcp__Telegram_Notifier__send_message", max_results: 1 }` if missing): # records scanned, # patched, what was moved, flags for review, # interactions consolidated into History, # interaction records deleted, any deletion failures. If the Telegram tool is unavailable, log the summary in your final report — do not fall back to raw HTTP.

**Error handling:** If any Airtable MCP call fails, send a Telegram describing what failed at which step and exit gracefully. Do NOT fall back to Chrome or direct API calls — the MCP connector is the only Airtable path for this task. Never delete an Interaction whose History append was not confirmed.
