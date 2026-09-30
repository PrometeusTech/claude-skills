# React specifics for file handling

## Picking and validating
- `<input type="file" accept=".pdf,.doc,.docx,application/pdf,...">` narrows the picker but is not
  validation (users can choose "All files").
- Validate size and extension before upload for fast feedback, using the same limits as the
  server. Keep the limits in one module (or read them from the API) so they do not drift.
- Never `FileReader.readAsDataURL` / base64 a large file just to send it if the API accepts
  multipart; base64 costs memory and 33% more bytes. If the API requires base64, check the size
  first.

## Uploading
- Use `FormData` + the project's HTTP client; show progress for large files; disable the submit
  button while the request is in flight (double submit = two uploads).
- If the endpoint supports an idempotency key, generate it once per form opening and reuse it on
  retry.
- Map server error codes to translated messages; do not show raw server strings.

## Downloading
- Ask the API for the download (it returns a signed URL or redirects); open it with a plain
  anchor / `window.location`. Do not fetch the file into a Blob unless you need to, and never keep
  signed URLs in long-lived state (they expire).
- If you must download via Blob (auth header required), revoke the object URL after use and use
  the server's `Content-Disposition` name.

## UI states
- Empty, loading, error and "file removed" states; replacing a file shows the current file name
  and date; deleting asks for confirmation.
- Content Security Policy: signed CDN URLs must be allowed by the policy's `connect-src` /
  `img-src` / navigation as relevant — test downloads with CSP enforced.

## Tests
- Manager/hook tests for: rejection by size and type, the request payload, error-code mapping,
  and in-flight disabling.
- An end-to-end (mocked API) flow: admin uploads, member sees and downloads.
