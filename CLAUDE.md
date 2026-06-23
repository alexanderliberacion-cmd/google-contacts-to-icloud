# CLAUDE.md — guide for AI agents working on this repo

This file orients an AI coding agent (Claude Code or similar) working on
`google-contacts-to-icloud`. Read it before making changes.

## ⛔ The one rule that matters most: never commit real contact data

This tool processes real people's names, phone numbers, and emails. **None of
that may ever enter git.**

- Real exports go in `input/`, output in `output/` — both are **gitignored**,
  along with `*.vcf`, `*.abbu`, and `contacts-*.csv`. Do not weaken these rules.
- The **only** contact-shaped files that may be committed are the anonymized
  fixtures in `tests/fixtures/`. They use reserved-fake `555-xxxx` numbers and
  obvious placeholder names (Jane Doe, Bob Smith…). If you add a fixture, it
  **must** be fake — never a real contact.
- When running the tool or tests during development, never paste real contact
  values into the conversation, commit messages, or reports. Aggregate counts
  only.

## What this tool is

A single-file Python CLI (`merge_contacts.py`, **standard library only — no
dependencies**) that merges Google Contacts CSV exports (and optional iCloud
vCard exports) into one clean, de-duplicated vCard for import into Apple
Contacts / iCloud.

Run: `python3 merge_contacts.py` (reads `./input`, writes `./output/icloud-ready.vcf`).
Tests: `python3 -m pytest tests/` or, with no pip, `python3 tests/test_merge.py`.

## Behavior contract (don't break these without a reason)

1. **Phone-only filter.** Contacts with no phone number are dropped. The whole
   point is a clean phone book, not a CRM.
2. **Dedup by normalized phone number only — never by name.** Two contacts
   merge iff they share a normalized number (digits only; leading NANP `1`
   stripped). Same name + no shared number → kept separate (avoids fusing two
   different people who happen to share a name).
3. **Clean Apple-native mapping is the default.** Output keeps only fields with
   a native Apple Contacts target (name, nickname, company, job title, phones,
   emails, addresses, birthday, notes, embedded iCloud photos). Google-specific
   data (websites, social, group labels, custom fields, photo URLs, relations)
   is dropped. **To change which fields are kept/dropped, edit `render_vcard()`**
   — that's the single place the field policy lives. The README's
   "What's kept vs dropped" table must stay in sync with it.
4. **Phone-label fidelity.** Standard labels → `TEL;TYPE=…`; iPhone/custom
   labels → `itemN.TEL` + `itemN.X-ABLabel:<label>` (Apple's item-group form).
5. **Company-only contacts** (organization, no person name) get
   `X-ABShowAs:COMPANY` so they file under the company name.
6. **Prefer the `+1` (E.164) form** when the same number appears both with and
   without it (no number is rewritten — the existing `+`-prefixed form is kept).
7. **iCloud passthrough.** A vCard contact not merged with a Google contact is
   emitted from its original block (preserving embedded photos, item-groups)
   with the field policy applied via `_filter_raw_vcard()`.

## Code map (`merge_contacts.py`)

| Area | Functions |
|---|---|
| Phone normalization / +1 preference | `normalize_phone`, `_dedupe_phones_prefer_plus` |
| Google CSV parsing | `parse_csv_row`, `read_csv_file` |
| vCard input parsing | `parse_vcf_file`, `_parse_vcard_property`, `_unfold_vcard_lines` |
| Dedup / merge | `merge_contacts`, `_merge_two_contacts`, `merge_by_name_pass` |
| Output rendering (field policy) | `render_vcard`, `_render_phone_lines`, `_render_address_lines`, `_filter_raw_vcard`, `_is_company_only` |
| Report | `FieldManifest` |

The field policy is hardcoded in `render_vcard()` (and `_filter_raw_vcard()`
for iCloud passthrough). To keep or drop a different set of fields, fork and
edit those functions directly — there are no field flags. The CLI exposes only
`--input`, `--output`, and `--merge-by-name`.

## When you change something

- **Add a test for it.** Tests live in `tests/test_merge.py` and run off the
  fake fixtures in `tests/fixtures/`. RED→GREEN. Use fake data only.
- **Keep the README's kept/dropped table in sync** with `render_vcard()`.
- **Don't add dependencies** — stdlib only is a feature (zero-install).
- **Don't loosen `.gitignore`** or commit anything from `input/`/`output/`.
