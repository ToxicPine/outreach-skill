---
name: outreach-lead-generation
description: Build lists of people to reach out to — sales prospects, potential hires, journalists, investors, collaborators, anyone. Use this skill whenever the user wants to find people to contact, build a prospect list, generate leads, research potential contacts, or enrich existing contacts with emails/LinkedIn/phone numbers. Also trigger when the user mentions outreach, prospecting, lead gen, contact lists, cold outreach, recruiting lists, or sourcing candidates — even if they don't explicitly say "lead generation."
---

# Outreach Lead Generation & Enrichment

Build a list of people to reach out to. The end goal is always a clean, complete list of contacts ready for outreach — with the right contact details filled in.

This is not a rigid pipeline. The user will steer: reviewing batches, asking for changes, filtering people out, pivoting criteria, going back to find more after enriching some. Follow the user's lead. The sections below describe capabilities and patterns, not a fixed sequence.

The core mechanic: research people, commit them to a file in small batches (each batch is a tool call the user can approve or deny), and progressively fill in their contact details. Approvals mean stay the course. Denials mean adjust. Keep going unless the user says stop.

## Agreeing on a Schema

Before finding anyone, establish what a fully-ready contact looks like. Ask:

- Who are we looking for? (role, industry, company type, geography, rough target count)
- What contact details do we need? (email, LinkedIn, phone, Twitter, etc.)
- Anything to exclude?

From the answers, propose a **single master schema** — every field that will ever matter, with types. Most fields start optional. Present it simply, get a yes, then start.

The schema is the north star. Without it, you end up with inconsistent fields across dozens of records. Once agreed, the field set doesn't change — only which fields are required can tighten as contacts get enriched.

## JSONL Files

Contacts go in JSONL files. Line 1 is always a schema object. Every subsequent line is one contact.

Use separate files for separate stages of work (e.g. `leads_discovered.jsonl`, `leads_enriched.jsonl`). Enrichment creates a new file — the discovery file is never modified. This means nothing is ever overwritten, and if something goes wrong, nothing is lost.

### Schema Line

```json
{"_schema":true,"stage":"discovered","fields":{"full_name":{"type":"string","required":true},"company":{"type":"string","required":false},"title":{"type":"string","required":false},"email":{"type":"email","required":false},"linkedin_url":{"type":"url","required":false},"phone":{"type":"e164","required":false},"notes":{"type":"string","required":false}}}
```

The enriched file has identical fields — only `required` flags change (more become `true`). Valid types: `string`, `url`, `email`, `e164`, `integer`, `boolean`.

## Finding People

Research candidates and commit them in batches of 5–8:

```bash
cat >> leads_discovered.jsonl << 'ENDBATCH'
{"full_name":"Jane Smith","company":"Acme","title":"VP Engineering","email":null,"linkedin_url":"https://linkedin.com/in/janesmith","phone":null,"notes":""}
{"full_name":"Tom Okoro","company":"Finstack","title":"CTO","email":null,"linkedin_url":null,"phone":null,"notes":"via TechCrunch"}
ENDBATCH
```

Every record includes every schema field. Use `null` for unknowns — never omit a key.

**Before each batch**, write one or two sentences identifying who's being added and why they fit. This is what the user reads to decide approve or deny — make it specific enough to be useful, short enough to read in a second. Example: *"Six CTOs at European fintech Series B/C companies — one ex-Stripe, a couple of repeat conference speakers."*

After a batch, immediately research the next one. If denied, infer what was wrong, say in one line what you're adjusting, and keep going. Only ask a direct question if you've been denied several times and genuinely can't tell why.

## Enriching Contacts

Enrichment fills in the fields that were null. Create a new file (e.g. `leads_enriched.jsonl`) with the same schema but tighter `required` flags. Work through discovered contacts, research each one, and append the enriched version.

Same narration discipline: one sentence before each batch saying what you found (or couldn't find), then append.

Before writing a record, verify required fields are non-null. If a required field can't be found after genuine effort, append anyway and note what's missing in `notes`. Never silently drop a contact.

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

Enforce these on every write.

**URLs** — full `https://` URL, never a bare handle or domain:

| Valid | Invalid |
|-------|---------|
| `https://linkedin.com/in/jane-doe` | `@jane` or `linkedin.com/in/jane` |
| `https://x.com/janedoe` | `@janedoe` |
| `https://github.com/janedoe` | `github.com/janedoe` |
| `https://acme.com` | `acme.com` |

If you only have a handle, construct the full URL. If you can't verify it resolves, write it and add `"unverified"` to notes.

**Phone** — E.164: `+` then country code then number, no spaces or punctuation (`+14155550100`, not `415-555-0100`).

**Email** — `local@domain.tld`. Only write what you actually found or can confidently infer from a confirmed company pattern. If uncertain, write `null` and note the suspected pattern.

**Any field you cannot find: write `null`. Never omit the key. Never guess.**

## Querying the Files

As the list grows, avoid reading entire files into context:

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

When the list is ready, export to CSV:

```bash
tail -n +2 leads_enriched.jsonl | python3 -c "
import sys, json, csv
rows = [json.loads(l) for l in sys.stdin]
w = csv.DictWriter(sys.stdout, fieldnames=rows[0].keys())
w.writeheader(); w.writerows(rows)
" > leads_export.csv
```

Spot-check formats before handing off:

```bash
grep -P '"linkedin_url":"(?!https://)' leads_enriched.jsonl
grep -P '"phone":"(?!\+)' leads_enriched.jsonl
```
