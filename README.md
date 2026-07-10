# social-crm-template

A template for running a personal social CRM in Airtable, operated by an AI agent (Claude) through the Airtable MCP server. Track friends and acquaintances, log real-world interactions, keep "reach out soon" cadences per person, and capture raw name lists from events without turning every name into a contact.

This is an anonymized template extracted from a real, daily-driven system. All IDs, tokens, and examples are placeholders; you supply your own base.

## What's in it

| File | What it is |
|---|---|
| [SKILL.md](SKILL.md) | The agent skill: schema documentation, MCP tool call patterns and gotchas, and the standing workflows (add a person, log an interaction, "glob" a roster, recurrence scan, bump scan) |
| [roster-globs.md](roster-globs.md) | Design doc for the Rosters pattern: storing name dumps as single list records with promotion-to-contact on recurrence, instead of polluting the contacts table |

## The data model

Four core tables:

- **People**: one record per person; relationship tier, per-person follow-up cadence, last contacted, context fields.
- **Groups**: clubs, teams, workplaces; linked from People.
- **Interactions**: dated log of real-world contact, linked to People. Powers the "who am I letting go cold" scan.
- **Rosters**: raw name lists from events ("glob this"), kept as lists. A name that keeps appearing across rosters is a candidate to promote to People.

## Setup

1. Create an Airtable base with the tables in SKILL.md (the field tables there are the spec).
2. Create a personal access token at airtable.com/create/tokens with data read/write scopes on that base.
3. Replace every `<YOUR_BASE_ID>`, `<YOUR_AIRTABLE_TOKEN>`, table/field ID placeholder in SKILL.md with your real values. Field and table IDs come from `GET /v0/meta/bases/{baseId}/tables` or the `list_tables_for_base` MCP tool.
4. Install SKILL.md as an agent skill (for Claude: a claude.ai account skill or a Claude Code skill folder) and connect the Airtable MCP server.
5. Optional: fill the Telegram placeholders to get batch-operation summaries pushed to your phone.

**Never commit your filled-in copy.** The placeholders exist because a real token in this file is a live credential.

## License

MIT. See [LICENSE](LICENSE).
