---
name: text-input-hardening
description: Rules and review checklist for user-supplied text that the app stores, compares or shows back — titles, names, descriptions, tags, labels, file names, search terms, slugs. Covers invisible and control characters (NUL, bidi overrides, zero-width), length limits vs column size, normalization that matches the database collation (case, diacritics) for uniqueness and de-duplication, limits on list sizes, and how hostile text renders in the UI. Use it whenever you add or change a form field, a free-text or tag input, a uniqueness rule, an upload's file name or display name, or review such code — even if the task only says "add a title field" or "let admins tag documents".
---

# Hostile text input

Free text looks harmless, so it gets the least attention — and then a NUL byte turns into a 500
after a file was already uploaded, an RTL override makes `exe.pdf` look like `fdp.exe`, two tags
that the database considers equal both get saved, and a 300-character tag breaks the mobile
layout. The rules below are cheap while building; the checklist is what a reviewer feeds the app.

## 1. Decide what a field accepts, then clean it once, at the boundary

- One normalizer per kind of field (title, tag, file name), called by the service before any
  validation, uniqueness check or side effect. Not in the controller, not repeated per feature.
- Remove or reject:
  - **control characters** (`\p{Cc}`: NUL, CR/LF where a single line is expected, ESC…) — NUL in
    particular breaks C-string-based layers (databases, storage SDKs, file systems);
  - **format characters** (`\p{Cf}`: bidi overrides U+202A–U+202E, U+2066–U+2069, zero-width
    U+200B–U+200D, U+FEFF) — they spoof what users read and defeat uniqueness;
  - leading/trailing whitespace; collapse internal runs of whitespace for single-line fields.
- Apply Unicode normalization (NFC) so the same visible text has one byte form.
- Decide reject vs strip per field and say it in the contract: a title with a NUL is probably an
  attack (reject with a clear code); a file name with one can be cleaned silently.
- Validate the **cleaned** value (empty after cleaning = invalid).

## 2. Lengths are product limits, not just column sizes

- Every text field has a maximum chosen by the product and enforced by validation with a clear
  error code — below the column size, and counted in characters (not bytes) for display limits.
- Long values must produce the field's own error, not an unrelated one (e.g. an over-long file
  name reported as "file required").
- Lists of values (tags, recipients, options) have a maximum count and a per-item maximum length.

## 3. Equality follows the database collation

When the app checks uniqueness or de-duplicates (tags, titles, emails, slugs), the comparison in
code must be the same one the database uses, or the two will disagree:

- With a case- and accent-insensitive collation (MySQL `utf8mb4_0900_ai_ci`, the default),
  `Café`, `cafe` and `CAFÉ` are equal for `WHERE` / unique indexes. Code that only
  downcases will save "duplicates" that the DB then treats as one (or that a unique index rejects
  with a 500).
- Options: let the database decide (`WHERE col = ?` already uses the column's collation; group
  suggestions with `GROUP BY col`), or build a normalization key in code that folds case **and**
  diacritics (transliterate + downcase) and use it everywhere — including the list of suggestions
  shown to users. Values stored inside JSON columns do **not** use the column collation; compare
  them in code with the folded key.
- Back uniqueness with a unique index; map the index violation to the same clean error as the
  validation (races bypass the validation).
- Display keeps the first-seen spelling; comparison uses the folded key.

## 4. Hostile text in the UI

- Long unbroken strings (`aaaa…`, URLs) wrap (`overflow-wrap: anywhere`) or are truncated with a
  tooltip; check at phone width (≈390 px) and in tables, chips, cards, menus and toasts.
- Many items (30 tags) are capped visually ("+12") rather than growing the card without limit.
- Text is rendered as text (no `dangerouslySetInnerHTML`), and bidi-sensitive values (file names)
  are wrapped in `<bdi>` or isolated with `unicode-bidi: isolate` if they may contain RTL text.
- Floating / sticky elements (FABs, cookie banners, bottom bars) must not cover the last row's
  actions — leave bottom padding equal to their height.

## Implementation checklist

- [ ] One normalizer per field kind; control and format characters handled; NFC; trimmed.
- [ ] Reject vs strip decided per field; error codes in the contract and translated.
- [ ] Max length per field (characters) and max count per list, each with its own error.
- [ ] Uniqueness / de-dupe uses the DB collation's notion of equality; unique index + clean
      error on violation; suggestions de-duplicated the same way.
- [ ] Cleaning and validation happen **before** any side effect (upload, email, external call).
- [ ] UI wraps/truncates long values, caps long lists, isolates bidi text, nothing covers actions.
- [ ] Tests with the probe strings below.

## Review / judge checklist (try to break it)

Send each probe through every new text field (create **and** update), then look at the response,
the stored row, any side effect (stored file, email), and the UI:

1. `"a\u0000b"` (NUL), `"a\r\nb"`, `"\u0007"`, `"\u001b[31m"` → clean 4xx or cleaned value, never
   500, never a side effect left behind.
2. `"‮fdp.exe"`, `"a​b"`, `"﻿title"` → stripped or rejected; uniqueness not fooled.
3. Exactly the max length, max+1, 10× max, and a 4-byte emoji string near the max → the field's
   own error, no truncation by the DB, no unrelated error message.
4. Case/diacritic twins in the product's language: `Report`, `report`, `REPORT`, `Répört`;
   `Café` / `cafe` → same
   decision in code and DB; suggestions show one entry.
5. Lists: 0 items, max items, max+1, 1000 items, duplicates inside one request.
6. Empty after cleaning: `"   "`, `"​"` → "required" error.
7. UI at 390 px and 1366 px with a 200-char unbroken title, 30 tags of 40 chars, an RTL file
   name; scroll to the last row and click its actions.
