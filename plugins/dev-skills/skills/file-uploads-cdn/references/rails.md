# Rails specifics for file handling

## Where the rules live
- A plain Ruby policy object per use case (types + max size, optionally per type) and one
  validator service that takes `(io_or_base64, policy)` and returns the detected type or a
  structured error (`code`, translated message). Controllers call it through a service, never
  inline.
- Keep uploaders (CarrierWave) or attachment models (Active Storage) thin: storage location and
  key only. Validation belongs to the validator so it is testable without storage.

## Content detection
- `Marcel::MimeType.for(io, name: ...)` (used by Active Storage) sniffs magic bytes but, for
  OOXML, often returns a generic zip or trusts the extension. For Word / Excel, add a small sniffer
  that opens the ZIP (`Zip::File` from rubyzip), parses `[Content_Types].xml` and requires the
  exact main-part content type (see the skill §2). Matching entry names
  (`entries.any? { _1.name == "word/document.xml" }`) is not enough.
- For legacy OLE files, parse the compound-file directory (the `ruby-ole` gem, or a minimal
  reader of the header + directory sectors) and look for a `WordDocument` stream **entry**.
  `bytes.include?("WordDocument".encode("UTF-16LE"))` accepts any XLS that contains that text.
- Rewind IOs after sniffing. Read only the bytes you need (headers, ZIP central directory), not
  the whole file.

## Size and memory
- Multipart uploads arrive as `ActionDispatch::Http::UploadedFile` backed by a tempfile: check
  `.size` before any `read`.
- Base64 JSON bodies are fully in memory by the time the controller runs: cap the request body
  (Rack middleware or web server) and check `encoded.bytesize <= max * 4 / 3 + padding` before
  `Base64.decode64`. Prefer `strict_decode64` and treat decode errors as 422, not 500.
- Remember `Rack::Utils.multipart_part_limit` and the web server's body limit.

## Text metadata
- Clean the original file name and the title before validation:
  `name.unicode_normalize(:nfc).gsub(/[\p{Cc}\p{Cf}]/, "").strip`, then cap the length. A NUL
  left in the name tends to raise deep in a lower layer (`ArgumentError: string contains null
  byte` from file APIs, or an error from the DB adapter or the storage SDK), i.e. a 500 that may
  come **after** the upload.
- Validate the row (`record.validate`) before calling the storage client.

## Callbacks and consistency
- Do storage deletes in `after_commit` (or explicitly after the transaction in the service), never
  in `before_destroy` / inside the transaction — a rollback would otherwise leave a row whose file
  is gone.
- Wrap external storage errors: report to the error tracker, return a clean error, keep the DB
  consistent (§5 of the skill).
- Replacement: capture the old key before assigning the new file; delete it after commit.
- Concurrency: `record.with_lock` around read-old-key / assign-new / save when two admins may edit
  the same record.

## Storage HTTP client
- Set timeouts explicitly: `Net::HTTP` `open_timeout` / `read_timeout` / `write_timeout`,
  HTTParty `timeout:` (or `open_timeout:` + `read_timeout:`), Faraday `request.timeout` /
  `open_timeout`. Library defaults are 60 s or more per phase.
- Retry PUT/DELETE a small number of times on connection errors and 5xx; never on 4xx.

## Serving
- Metadata endpoint and download endpoint share one policy method.
- `redirect_to signed_url, allow_other_host: true` (Rails 7+) for CDN files; `send_file` only for
  local-disk storage, with `disposition: "attachment"` and a sanitized `filename:`. Rails encodes
  `filename*` for you in `send_file`/`send_data`; when you build the header yourself use
  `ActionDispatch::Http::ContentDisposition.format(disposition:, filename:)`.

## Params and errors
- Never call `.read`, `.size` or `Base64` on `params[...]` without checking its shape first (a
  string where a file is expected, an array, a hash) — malformed input must be 400/422, never 500.
- Error codes in the API contract (e.g. `file_type_not_allowed`, `file_too_large`,
  `file_empty`) so the frontend can translate them.

## Tests
- Keep tiny real fixture files per accepted and rejected type (a real `.doc`, `.docx`, `.docm`,
  `.xlsx`, `.pdf`, a renamed zip). Generate large files in the test, don't commit them.
- Stub the storage client with an object that records calls and assert on them (put / delete with
  which key, and that delete happens only after commit).
- Add one regression test per existing consumer when the shared validator changes.
