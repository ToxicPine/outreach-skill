---
name: outreach-lead-generation
description: Build lists of people to reach out to — sales prospects, potential hires, journalists, investors, collaborators, anyone. Use this skill whenever the user wants to find people to contact, build a prospect list, generate leads, research potential contacts, or enrich existing contacts with emails/LinkedIn/phone numbers. Also trigger when the user mentions outreach, prospecting, lead gen, contact lists, cold outreach, recruiting lists, or sourcing candidates — even if they don't explicitly say "lead generation."
---

# Outreach Lead Generation & Enrichment

Build a list of people to reach out to. The destination is always the same: a clean, complete list of contacts ready for outreach with the right contact details filled in.

Getting there involves searching, filtering, revisiting, merging sources, enriching some contacts before others, backtracking when a search direction isn't working. The user will steer — reviewing batches, asking for changes, filtering people out, pivoting criteria. Follow their lead. The patterns below are tools to draw on, not steps to follow in order.

## The Steering Mechanic

Work happens in small batches of 5–8 contacts. Each batch is a tool call the user can approve or deny. This is the primary feedback loop — it's how the user shapes the list without micromanaging every search.

Before each batch, write one or two sentences identifying who's being added and why they fit. This is what the user reads to decide. Make it specific enough to be useful, short enough to read in a second: *"Six CTOs at European fintech Series B/C companies — one ex-Stripe, a couple of repeat conference speakers."*

If they keep approving without comment, trust your judgment and keep narration tight. If denied, infer what was wrong, say in one line what you're adjusting, and keep going. Only ask a direct question if you've been denied several times and genuinely can't tell why.

## Schema First

When you're reading across dozens of web pages and sources, you need somewhere structured to put what you find. Without a schema agreed upfront, you end up with inconsistent fields, missing data, and reformatting work at the end. The schema describes what a fully-ready contact looks like. The work is just progressively getting each contact to that state.

Before finding anyone, establish:

- Who are we looking for? (role, industry, company type, geography, rough target count)
- What contact details do we need by the end? (email, LinkedIn, phone, Twitter, etc.)
- Anything to exclude?

Propose a **single master schema** with every field that will ever matter. Include everything now — adding fields later means going back through every existing record. Most fields start optional and become required as contacts get enriched. Get a yes, then start.

## JSONL Format

Contacts go in JSONL files — one schema object on line 1, then one contact per line after that.

This format exists for three reasons:
- **Append-only is safe.** Writing new records never touches existing ones. No risk of corrupting the list as it grows.
- **Separate files preserve work.** Discovery and enrichment live in different files (`leads_discovered.jsonl`, `leads_enriched.jsonl`). If enrichment goes sideways, discovery is untouched.
- **Queryable without loading everything.** Grep for a name, count records, or spot-check a field without pulling the whole file into context. This matters once the list outgrows what fits comfortably in conversation.

### Schema line (line 1 of each file)

```json
{"_schema":true,"stage":"discovered","fields":{"full_name":{"type":"string","required":true},"company":{"type":"string","required":false},"title":{"type":"string","required":false},"email":{"type":"email","required":false},"linkedin_url":{"type":"url","required":false},"phone":{"type":"e164","required":false},"notes":{"type":"string","required":false}}}
```

The enriched file has the same fields — only `required` flags tighten. Valid types: `string`, `url`, `email`, `e164`, `integer`, `boolean`.

### Writing records

```bash
cat >> leads_discovered.jsonl << 'ENDBATCH'
{"full_name":"Jane Smith","company":"Acme","title":"VP Engineering","email":null,"linkedin_url":"https://linkedin.com/in/janesmith","phone":null,"notes":""}
{"full_name":"Tom Okoro","company":"Finstack","title":"CTO","email":null,"linkedin_url":null,"phone":null,"notes":"via TechCrunch"}
ENDBATCH
```

Every record includes every schema field. Use `null` for unknowns — never omit a key. Downstream tools and exports break on missing keys, and it makes it obvious at a glance what still needs to be found.

## Enrichment

Enrichment fills in the nulls — researching each person to find their email, LinkedIn, phone, etc. It happens in a new file so the raw discovery list is preserved. The enriched file uses the same schema with tighter `required` flags.

If a required field can't be found after genuine effort, write the record anyway and note what's missing. Never silently drop a contact — the user can decide what to do with incomplete records.

## Field Formats

Consistent formatting matters because the output feeds into outreach tools, CRMs, and mail merge — anything that expects a standard format will choke on inconsistencies.

- **URLs** — always full `https://` URLs, never bare handles or domains. Outreach tools need clickable links. If you only have a handle, construct the full URL; if unverified, note it.
- **Phone** — E.164 (`+14155550100`). The international standard every telephony API and CRM expects. No spaces, dashes, or parentheses.
- **Email** — only write what you actually found or can confidently infer from a confirmed company pattern. A wrong email is worse than no email — it wastes outreach attempts and hurts sender reputation. If uncertain, write `null` and note the suspected pattern.
- **Unknown fields** — write `null`. Never omit the key. Never guess.

## Exporting

When ready, convert to CSV. Spot-check URL and phone formats before handing off — catching a bad format now saves the user from discovering it mid-campaign.
