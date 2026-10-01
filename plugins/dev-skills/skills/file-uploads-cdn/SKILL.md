---
name: file-uploads-cdn
description: Rules and review checklist for anything that accepts, stores, serves or deletes user files — uploads, attachments, documents, images, avatars, imports, downloads, signed URLs, CDN or object storage (BunnyNet, S3, local disk on a VM), the shared upload infrastructure (validators, content-type detection, uploaders), and generated files (PDF, CSV, exports). Use it whenever you implement, change or review such code, even if the task only says "add a PDF field", "let admins attach a file", "allow Word documents" or "fix the download link", and whenever a diff touches upload validators, uploaders or storage services, because those are shared by every feature that handles files.
---

# File uploads, storage and downloads

Files are where small shortcuts turn into security bugs (disguised executables, path traversal,
header injection), privacy leaks (public URLs, EXIF GPS) and silent data rot (orphaned objects,
rows pointing at deleted files). The rules below apply both when **building** a file feature and
when **reviewing** one; the checklist at the end is what a reviewer tries to break.

Stack specifics live in references — read the one that matches the code you touch:
- `references/rails.md` — Rails backends (CarrierWave / Active Storage / custom services, Rack limits, callbacks)
- `references/react.md` — React frontends (file inputs, client validation, progress, downloads)
- `references/storage.md` — BunnyNet storage + pull zone with signed URLs, and local disk on a VM

## 1. One upload pipeline, many consumers

Put the rules in one place: a policy object per use case (allowed types, max size per type) and
one validator that every feature calls. Features then differ only by policy.

Why: when each feature validates on its own, the weakest one defines your security. The flip
side: **a change to the shared pipeline is a change to every feature that uploads files.** When
you touch it, list all its consumers (grep for the validator / uploader / policy class) and run or
add a regression test for each type they accept. A reviewer should treat an edit there as
cross-cutting, not as part of the feature that motivated it.

## 2. Decide the type from the bytes, not from the name

- Allowlist types per policy. Never a denylist.
- Detect from content: magic bytes, and for container formats inspect the container.
  - DOCX / XLSX / PPTX are ZIP files. Decide from the **content types declared in
    `[Content_Types].xml`**, not from entry names: for Word the main part's override must be
    exactly `application/vnd.openxmlformats-officedocument.wordprocessingml.document.main+xml`.
    That one rule rejects `.docm` (`…macroEnabled.main+xml`), `.dotx` / `.dotm` (`…template…`)
    and any `vbaProject` part however it is named. Checking that `word/document.xml` exists or
    that no entry is literally called `vbaProject.bin` is bypassed by renaming.
  - Legacy DOC / XLS / PPT share the OLE compound-file magic (`D0 CF 11 E0`). Tell them apart by
    **parsing the compound-file directory** and finding the `WordDocument` stream as a directory
    entry — not by searching the bytes for the UTF-16 string `WordDocument`, which any XLS can
    contain (a sheet cell, a stream name) and so passes as a Word file.
  - PDF: `%PDF-` at offset 0. A PDF can also be valid HTML (polyglot); serving it as an
    attachment with the stored type and `nosniff` (§7) neutralizes it, and you may additionally
    reject PDFs whose first kilobytes contain `<html` / `<script`.
  - Images: real image header, then decode it (see §6).
- The client's `Content-Type` and the extension are hints for error messages only. A file whose
  extension disagrees with its detected type is rejected, not "fixed".
- Store the detected type and use it when serving.

## 3. Enforce size before you pay for it

- Reject on `Content-Length` / multipart part size before reading the body into memory when the
  framework allows it; stream to a temp file instead of building strings.
- Base64 payloads are ~4/3 of the file: decide whether the limit applies to the decoded file (it
  should, for users) and cap the encoded payload accordingly, **before** decoding.
- The web server has its own limit (e.g. nginx `client_max_body_size`); keep it slightly above the
  app limit so users get the app's clear error, not a bare 413 page.
- Empty files are invalid unless the product says otherwise.
- Count the in-memory copies of one upload (raw body, decoded string, `StringIO`, buffer for the
  storage client): three copies of a 10 MB file is 30 MB per request, times the worker count.
  Stream from the tempfile where you can and measure peak memory with a max-size file.

## 4. Store privately, reference by key

- Default to private storage. Serve through short-lived signed URLs, or through the app after an
  authorization check. "It's just a blank form" is still private by default — public is a
  deliberate, documented decision.
- Generate the storage key yourself (random/UUID, grouped by tenant and model). Never use the
  user's filename in the path. Keep the original filename only as display metadata, cleaned like
  any other user text (NUL, control and bidi characters, length — see the text-input-hardening
  skill).
- Never return storage keys, bucket paths or storage credentials in API responses.

## 5. Keep database and storage consistent

Storage calls are not part of your DB transaction, so decide the order explicitly:

- **Validate everything first.** Every check that can fail on the row (title, tags, original
  file name, lengths, uniqueness, idempotency key) runs **before** the storage call. A NUL byte
  in the file name that only fails at `INSERT` means the object is already uploaded and orphaned.
- **Create:** upload the object, then save the row. If the save still fails, delete the object
  you just uploaded (compensation), or leave it for an orphan-cleanup job — but never leave a row
  pointing at nothing.
- **Replace:** upload the new object, save the row, and delete the old object **after commit**.
  Deleting first loses the file if the save fails.
- **Delete:** delete the row, delete the object after commit. A failed object delete is reported
  (error tracker) and retried/cleaned later; it must not resurrect or 500 the request.
- Two concurrent replacements must not leave the row pointing at a deleted object (lock the row or
  compare the key you are replacing).
- Deletes are idempotent (object already gone = success).
- The storage client has explicit open/read timeouts and a small, bounded retry for idempotent
  calls (PUT with the same key, DELETE); see the idempotency-retries skill §6.

## 6. Images and other decoded formats

- Decode and re-encode images server-side: strips EXIF (GPS, device data), normalizes orientation
  and neutralizes polyglots. Cap pixel dimensions to avoid decompression bombs.
- Generate thumbnails/variants asynchronously when they are expensive.
- Never serve SVG, HTML or XML uploads inline from the app's origin (script execution). Serve them
  as attachments, from a separate domain, or not at all.

## 7. Serve downloads safely

- Prefer redirecting to a short-lived signed URL over streaming the bytes through the app process.
- `Content-Disposition: attachment` with a sanitized name: strip CR/LF, quotes, path separators,
  control and bidi characters; cap the length; send `filename*=UTF-8''<percent-encoded>` plus an
  ASCII fallback. The download name is usually "<document title>.<detected extension>", not the
  uploader's original name — and not "<title>.pdf.pdf" when the title already ends in the
  extension.
- **The name the user gets is what the browser saves**, not a `filename` field in a JSON
  response. When you redirect to a CDN, the CDN's response headers decide the saved name (the
  app's headers on the redirect are ignored), so set the disposition through the CDN/storage
  (object metadata or a signed response-header parameter, if the provider supports it) or accept
  and document the object name. Verify with a real browser download.
- `Content-Type` from the stored detected type; `X-Content-Type-Options: nosniff`.
- The download endpoint runs the same authorization as the metadata endpoint (see the
  authz-multitenancy skill). Signed URLs are issued only after that check.
- A download of a deleted or missing file says "not found" (and the frontend shows that), not
  "you are not allowed".

## 8. Generated files (PDF, CSV, exports)

- Generated files are documents of the tenant: same authorization, private storage, signed or
  checked download as uploads.
- User content inside them is escaped for the format (PDF markup, CSV formula injection: prefix
  cells starting with `=`, `+`, `-`, `@` with `'`).
- Fonts cover the product's language (e.g. Romanian `ș ț ă î â` need a Unicode TTF; built-in PDF
  fonts render them as `?` or crash) — test with real diacritics.
- Heavy generation (many pages, images, whole-tenant exports) runs in a job or is bounded by size
  and time; the request does not block a web worker for tens of seconds.

## 9. Frontend expectations

Mirror the server rules in the UI (accept attribute, size check, readable error) so users get fast
feedback — but the server stays the only enforcement. Map the server's error codes to translated
messages. See `references/react.md`.

## Implementation checklist

- [ ] Uses the shared policy/validator; policy lists types + max size; no ad-hoc checks.
- [ ] Type detected from content; macro-enabled / disguised formats rejected; detected type stored.
- [ ] Size enforced before full read; base64 overhead handled; web server limit consistent.
- [ ] Private storage, generated keys, no keys or credentials in responses.
- [ ] All row validation (incl. cleaned file name) before the storage call; create / replace /
      delete ordering as in §5; failures reported; orphan cleanup exists or is noted.
- [ ] Storage client: explicit timeouts, bounded retries, separate credentials per purpose.
- [ ] Memory: copies per upload counted; peak measured with a max-size file.
- [ ] Download: authorization, signed URL or checked stream, safe `Content-Disposition`, `nosniff`;
      saved file name checked in a browser; deleted file → "not found".
- [ ] If the shared pipeline changed: every consumer re-tested.
- [ ] Tests: allowed types, each rejection path, oversize, empty, replace/delete cleanup (stub the
      storage client, and assert the calls — a stub that accepts anything proves nothing).
- [ ] API contract documents the upload fields, limits and error codes.
- [ ] Generated files: authorized, escaped (CSV formulas), Unicode fonts, bounded or async.

## Review / judge checklist (try to break it)

Build the probe files in a scratch directory, never in the repo:

1. Disguised files: `.exe`/`.html` renamed to `.pdf`/`.docx`; a ZIP renamed to `.docx`; an XLSX
   renamed to `.docx`; a `.docm` and a `.dotx` renamed to `.docx`; a DOCX with `vbaProject.bin`
   renamed to an innocent name but still declared in `[Content_Types].xml`; an XLS that contains
   the text `WordDocument` (in a cell or as a fake stream name) renamed to `.doc`; a PDF/HTML
   polyglot.
2. Empty file, 1 byte over the limit, a payload far over the limit (watch memory and time — is it
   rejected before decode/read?).
3. Filenames: `../../etc/passwd`, `a"; b.pdf`, **NUL inside**, CR/LF inside, very long (300+),
   emoji / diacritics, RTL override (U+202E). For each: response status **and** whether an object
   was left in storage without a row.
4. Spoofed `Content-Type` header vs real bytes.
5. Download: headers for titles with quotes, diacritics, newlines; then the **name the browser
   actually saves** (Playwright `download.suggestedFilename()`), including through the CDN
   redirect; a title already ending in `.pdf`; a deleted document's download link.
6. Storage failure injected mid-create / mid-replace / mid-delete: what is left in DB and in storage?
7. Two concurrent replacements of the same record.
8. Access: foreign tenant, anonymous, lower role → metadata **and** download refused; signed URL
   expiry actually short; key not guessable.
9. Other features that use the shared pipeline still accept what they accepted before.
10. Frontend: client-side limits match the server; server errors shown translated; no whole-file
    reads into memory for large files.
11. Memory and time for a max-size and a far-over-size upload (peak RSS of the worker, not only
    the response); storage stubbed to hang → bounded by the client timeout.
12. Generated files: put `ș ț ă`, `=HYPERLINK(...)`, HTML and a 5,000-character text into the
    source data; generate for the largest realistic tenant; check rendering, escaping, time and
    who can download it.
