---
name: outreach-lead-generation
description: Build targeted lists of people to reach out to — sales prospects, potential hires, journalists, investors, collaborators — by autonomously researching, discovering contacts, and enriching them into outreach-ready JSONL files.
---

# Outreach Lead Generation & Enrichment

This skill builds a list of people to reach out to — sales prospects, potential hires, journalists, investors, collaborators, anyone where the end goal is sending them a message.

The process is not a rigid pipeline. In practice it involves searching, filtering, revisiting, merging sources, enriching some contacts before others, backtracking when a search direction isn't working. What matters is the destination: a complete, clean list of contacts ready for outreach. Everything the agent does is in service of getting there.

The agent works autonomously — researching, finding contacts, and committing them to a file in small batches. Each batch is a tool call the user can approve or deny. That approve/deny is the primary steering mechanism. The agent doesn't stop to ask "should I keep going?" — it just keeps going. Approvals mean stay the course. Denials mean adjust.

## When to Use

Use this skill when the user wants to:
- Build a list of people to contact for sales, recruiting, PR, fundraising, partnerships, or any outreach purpose
- Find and enrich contact details (email, LinkedIn, phone, etc.) for a set of target personas
- Generate a prospecting or lead list from scratch based on criteria like role, industry, company type, or geography

## Instructions

### Phase 0: Agree on a Schema

Before finding anyone, have a brief conversation to establish:

- **Who are we looking for?** (role, industry, company type, geography, rough target number)
- **What contact details will we need by the end?** (email, LinkedIn, phone, Twitter, etc.)
- **Anything to exclude?**

From the answers, propose a **single master schema** — the full shape of a contact as it will look when fully ready for outreach. Include every field that will ever matter. Most fields will start optional and become required as the work progresses. The field set itself doesn't change.

Present the schema simply. Get a yes. Then start immediately.

### JSONL File Format

Contacts are stored in JSONL files. Each file represents one stage of the pipeline (e.g. `leads_discovered.jsonl`, `leads_enriched.jsonl`). Line 1 of every file is a schema object. Every line after that is one contact record.

This format is intentional:
- **Nothing is ever overwritten.** Appending new records never touches existing ones.
- **Stages are preserved as separate files.** Enrichment happens in a new file — the discovery file is never modified. If something goes wrong, nothing is lost.
- **Easy to query without loading everything.** You can grep for a name, count records, or spot-check a field without pulling the whole file into context. This matters as the list grows.

### Schema Line Format

Line 1 of each stage file is a schema object identifying the stage and declaring every field with its type and whether it is required at that stage:

```json
{"_schema":true,"stage":"discovered","fields":{"full_name":{"type":"string","required":true},"company":{"type":"string","required":false},"title":{"type":"string","required":false},"email":{"type":"email","required":false},"linkedin_url":{"type":"url","required":false},"phone":{"type":"e164","required":false},"notes":{"type":"string","required":false}}}
```

The enriched file has the exact same fields. Only the `required` flags change — more become `true` as the stage advances. A field that is optional at discovery can become required at enrichment. A field cannot be added later that wasn't in the original schema.

Valid types: `string`, `url`, `email`, `e164`, `integer`, `boolean`

### Phase 1: Discovery — Finding People

Research candidates and commit them in batches of 5–8 per tool call:

```bash
cat >> leads_discovered.jsonl << 'ENDBATCH'
{"full_name":"Jane Smith","company":"Acme","title":"VP Engineering","email":null,"linkedin_url":"https://linkedin.com/in/janesmith","phone":null,"notes":""}
{"full_name":"Tom Okoro","company":"Finstack","title":"CTO","email":null,"linkedin_url":null,"phone":null,"notes":"via TechCrunch"}
ENDBATCH
```

Every record must include every field from the schema. Use `null` for anything not yet found — never omit a key.

**Before each batch tool call**, write one or two sentences identifying who is about to be added and why they fit the outreach target. For example: *"Six CTOs at European fintech Series B/C companies — one ex-Stripe, a couple of repeat conference speakers. Adding them now."* This is what the user reads to decide whether to approve or deny. It should be specific enough to be useful, short enough to read in a second.

After the tool call, immediately start researching the next batch. Don't wait.

**Reading approvals:** if the user keeps approving without comment, that's a signal to trust your own judgment and keep narration tight — just enough to identify the batch. If a batch is denied, infer what was wrong from context, say in one line what you're adjusting, and keep going. Only ask a direct question if you've been denied several times and genuinely cannot tell why.

### Phase 2: Enrichment — Filling In Contact Details

Once discovery is underway or complete, enrichment fills in the fields that were null or optional. This happens in a new file — `leads_discovered.jsonl` is never touched again.

Create `leads_enriched.jsonl` with a schema line identical in fields to the discovery file, but with the agreed fields now marked `required: true`. Then work through the discovered contacts, researching each one and appending the enriched version.

The same narration discipline applies: one sentence before the tool call saying what you found (or couldn't find), then append, then move to the next.

Before appending a record to the enriched file, verify all required fields are non-null. If a required field cannot be found after genuine effort, append the record anyway and note what's missing in `notes`. Never silently drop a contact.

To track progress without loading full files into context:

```bash
# Who has already been enriched
grep -o '"full_name":"[^"]*"' leads_enriched.jsonl

# Pull a specific discovered record to enrich
grep '"Jane Smith"' leads_discovered.jsonl

# Find enriched records still missing a required field
grep '"email":null' leads_enriched.jsonl
```

### Field Formats — Enforce on Every Write

**URLs** — always the full `https://` URL, never a bare handle or domain:

| Valid | Invalid |
|-------|---------|
| `https://linkedin.com/in/jane-doe` | `@jane` or `linkedin.com/in/jane` |
| `https://x.com/janedoe` | `@janedoe` |
| `https://github.com/janedoe` | `github.com/janedoe` |
| `https://acme.com` | `acme.com` |

If you only have a handle, construct the full URL and verify it resolves before writing. If you can't verify, write it and add `"unverified"` to notes.

**Phone** — E.164 format: `+` then country code then number, no spaces or punctuation:

| Valid | Invalid |
|-------|---------|
| `+14155550100` | `415-555-0100` |
| `+447700900100` | `07700 900100` |

**Email** — `local@domain.tld`. Only write what you actually found or can confidently infer from a confirmed company pattern. If uncertain, write `null` and note the suspected pattern.

**Any field you cannot find: write `null`. Never omit the key. Never guess.**

### Querying the Files

As the list grows, avoid reading entire files into context. Use targeted queries:

```bash
# Count records at each stage (subtract 1 for the schema line)
wc -l leads_discovered.jsonl leads_enriched.jsonl

# Sample a few records
sed -n '2,5p' leads_discovered.jsonl

# Find a specific person
grep '"Jane Smith"' leads_discovered.jsonl

# Check for missing required fields
grep '"email":null' leads_enriched.jsonl
```

### Phase 3: Export

When the enriched list is ready, export to CSV:

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
