---
name: outreach-lead-generation
description: Build lists of people to reach out to — sales prospects, potential hires, journalists, investors, collaborators, anyone. Use this skill whenever the user wants to find people to contact, build a prospect list, generate leads, research potential contacts, or enrich existing contacts with emails/LinkedIn/phone numbers. Also trigger when the user mentions outreach, prospecting, lead gen, contact lists, cold outreach, recruiting lists, or sourcing candidates — even if they don't explicitly say "lead generation."
---

# Outreach Lead Generation & Enrichment

Build a list of people to reach out to. The end goal is always a clean, complete list of contacts ready for outreach — with the right contact details filled in.

This is not a rigid pipeline. In practice it involves searching, filtering, revisiting, merging sources, enriching some contacts before others, backtracking when a search direction isn't working. The user will steer: reviewing batches, asking for changes, filtering people out, pivoting criteria, going back to find more after enriching some. Follow the user's lead. The sections below describe capabilities and patterns you can draw on in whatever order makes sense.

The core mechanic: research people, commit them to a file in small batches (each batch is a tool call the user can approve or deny), and progressively fill in their contact details. The approve/deny on each batch is the primary steering mechanism — it's how the user shapes the list without having to micromanage. Approvals mean stay the course. Denials mean adjust. Keep going unless the user says stop.

## Agreeing on a Schema

When you're reading across dozens of web pages and sources, you need somewhere structured to put what you find. Without a schema agreed upfront, you end up with inconsistent fields, missing data, and reformatting work at the end. The schema describes what a fully-ready contact looks like — the north star. The work is just progressively getting each contact to that state.

Before finding anyone, have a brief conversation to establish:

- Who are we looking for? (role, industry, company type, geography, rough target count)
- What contact details do we need by the end? (email, LinkedIn, phone, Twitter, etc.)
- Anything to exclude?

From the answers, propose a **single master schema** — every field that will ever matter, with types. Include everything now because adding fields later means going back through every existing record. Most fields start optional and become required as contacts get enriched. Present the schema simply, get a yes, then start immediately.

## JSONL Files

Contacts go in JSONL files. Line 1 is always a schema object. Every subsequent line is one contact record. This format is chosen for specific reasons:

- **Append-only safety.** Writing new records never touches existing ones. There's no risk of corrupting or accidentally overwriting the list as it grows.
- **Separate files preserve work.** Use different files for different stages (e.g. `leads_discovered.jsonl`, `leads_enriched.jsonl`). Enrichment happens in a new file — the discovery file is never modified. If something goes wrong during enrichment, the discovery work is untouched.
- **Queryable without loading everything.** You can grep for a name, count records, or spot-check a field without pulling the whole file into context. This matters as the list grows past what fits comfortably in a conversation.

### Schema Line

Line 1 of each file is a schema object declaring the stage and every field with its type and whether it's required at that stage:

```json
{"_schema":true,"stage":"discovered","fields":{"full_name":{"type":"string","required":true},"company":{"type":"string","required":false},"title":{"type":"string","required":false},"email":{"type":"email","required":false},"linkedin_url":{"type":"url","required":false},"phone":{"type":"e164","required":false},"notes":{"type":"string","required":false}}}
```

The enriched file has identical fields — only `required` flags change (more become `true`). Valid types: `string`, `url`, `email`, `e164`, `integer`, `boolean`.

## Finding People

Research candidates and commit them in batches of 5–8. Batches this size are small enough for the user to meaningfully review (they can scan the names and companies at a glance) but large enough to make real progress.

```bash
cat >> leads_discovered.jsonl << 'ENDBATCH'
{"full_name":"Jane Smith","company":"Acme","title":"VP Engineering","email":null,"linkedin_url":"https://linkedin.com/in/janesmith","phone":null,"notes":""}
{"full_name":"Tom Okoro","company":"Finstack","title":"CTO","email":null,"linkedin_url":null,"phone":null,"notes":"via TechCrunch"}
ENDBATCH
```

Every record includes every schema field. Use `null` for unknowns — never omit a key. This consistency matters because downstream tools and exports will break on missing keys, and it makes it obvious at a glance what still needs to be found.

**Before each batch**, write one or two sentences identifying who's being added and why they fit. This is what the user reads to decide approve or deny — it needs to be specific enough to be useful, short enough to read in a second. Example: *"Six CTOs at European fintech Series B/C companies — one ex-Stripe, a couple of repeat conference speakers."*

After a batch, immediately research the next one. The user's approvals and denials are your feedback loop — if they keep approving without comment, trust your judgment and keep narration tight. If a batch is denied, infer what was wrong from context, say in one line what you're adjusting, and keep going. Only ask a direct question if you've been denied several times and genuinely can't tell why.

## Enriching Contacts

Enrichment fills in the fields that were null — researching each person to find their email, LinkedIn, phone, etc. This happens in a new file so the raw discovery list is preserved as-is. Create `leads_enriched.jsonl` with the same schema but tighter `required` flags reflecting what the user actually needs for outreach.

Work through discovered contacts, research each one, and append the enriched version. Same narration discipline: one sentence before each batch saying what you found (or couldn't find), then append.

Before writing a record, verify required fields are non-null. If a required field can't be found after genuine effort, append anyway and note what's missing in `notes`. The goal is a complete list — never silently drop a contact just because one field is hard to find. The user can decide what to do with incomplete records.

Track progress without loading full files:

```bash
# Who has been enriched
grep -o '"full_name":"[^"]*"' leads_enriched.jsonl

# Pull a specific record
grep '"Jane Smith"' leads_discovered.jsonl

# Find records still missing a required field
grep '"email":null' leads_enriched.jsonl
```

## Field Formats

Consistent formatting matters because the output list will be fed into outreach tools, CRMs, or mail merge — anything that expects a standard format will choke on inconsistencies. Enforce these on every write.

**URLs** — full `https://` URL, never a bare handle or domain. Outreach tools need clickable links, not handles they can't resolve:

| Valid | Invalid |
|-------|---------|
| `https://linkedin.com/in/jane-doe` | `@jane` or `linkedin.com/in/jane` |
| `https://x.com/janedoe` | `@janedoe` |
| `https://github.com/janedoe` | `github.com/janedoe` |
| `https://acme.com` | `acme.com` |

If you only have a handle, construct the full URL. If you can't verify it resolves, write it and add `"unverified"` to notes.

**Phone** — E.164 format (`+14155550100`, not `415-555-0100`). This is the international standard that every telephony API and CRM expects. No spaces, no dashes, no parentheses — just `+`, country code, number.

**Email** — `local@domain.tld`. Only write what you actually found or can confidently infer from a confirmed company pattern (e.g. you've verified that company X uses `first.last@company.com` for multiple employees). If uncertain, write `null` and note the suspected pattern — a wrong email is worse than no email because it wastes the user's outreach attempts and can hurt sender reputation.

**Any field you cannot find: write `null`. Never omit the key. Never guess.**

## Querying the Files

As the list grows, avoid reading entire files into context. These files can get large, and pulling hundreds of records into the conversation wastes context and slows everything down. Use targeted queries:

```bash
# Count records (subtract 1 for schema line)
wc -l leads_discovered.jsonl leads_enriched.jsonl

# Sample a few records
sed -n '2,5p' leads_discovered.jsonl

# Find a specific person
grep '"Jane Smith"' leads_discovered.jsonl

# Check for missing required fields
grep '"email":null' leads_enriched.jsonl
```

## Exporting

When the list is ready, export to CSV for use in outreach tools:

```bash
tail -n +2 leads_enriched.jsonl | python3 -c "
import sys, json, csv
rows = [json.loads(l) for l in sys.stdin]
w = csv.DictWriter(sys.stdout, fieldnames=rows[0].keys())
w.writeheader(); w.writerows(rows)
" > leads_export.csv
```

Spot-check formats before handing off — catching a bad URL or phone format now saves the user from discovering it mid-campaign:

```bash
grep -P '"linkedin_url":"(?!https://)' leads_enriched.jsonl
grep -P '"phone":"(?!\+)' leads_enriched.jsonl
```
