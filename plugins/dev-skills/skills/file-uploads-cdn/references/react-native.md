# React Native / Expo specifics for file handling

The server rules of the skill apply unchanged — the app is one more untrusted client. Confirm
module APIs against the project's Expo SDK (file-system and picker APIs changed between SDKs).

## Picking
- Photos and videos: `expo-image-picker` (camera or library); documents: `expo-document-picker`.
  Both return a local URI (`file://`, or `content://` on Android) plus a name, size and MIME type
  that are hints only — the server still decides the type from the bytes.
- The name may be missing or generic (`image.jpg`, a UUID): the server builds stored names itself
  and shows a sensible display name.
- iPhone photos are often HEIC. Either the server accepts HEIC (and converts for display), or the
  app converts to JPEG before upload — decide once, for every picker in the app.
- Recent Android versions offer a system photo picker that needs no storage permission; iOS may
  grant limited photo access. Handle both, and the refusal (mobile-ui-quality §6).

## Size and privacy before upload
- Phone photos are several MB and videos far more. Resize / compress on the device to what the
  feature needs (e.g. `expo-image-manipulator`), then check the size against the **same** limits as
  the server before sending.
- Photos carry EXIF with GPS location. Re-encoding on the device usually drops it — verify — and
  the server strips it anyway (§6 of the skill).
- Normalize orientation (EXIF rotation) so the server and other clients show the image upright.

## Uploading
- Multipart with `FormData` and a file part `{ uri, name, type }` through the project's HTTP
  client, or the file-system module's upload API (which can keep going in a background session on
  iOS). Show progress for large files; disable submit in flight.
- The app may be backgrounded or killed mid-upload: the upload carries an idempotency key created
  once per user intent and persisted with the pending upload, and is retried or resumed on return
  (mobile-offline-lifecycle §2). For large files prefer direct-to-storage signed uploads that can
  resume.
- Map server error codes to translated messages.

## Temporary files and downloads
- Pickers and manipulators write copies into the cache directory: delete them after a successful
  upload; never leave sensitive documents there.
- Downloads: open the signed URL in the system browser / viewer, or download into the app sandbox
  with the server's `Content-Disposition` name and hand it to the share sheet (`expo-sharing`) when
  the user wants to keep it. Do not store signed URLs in persisted state (they expire).

## Tests
- Unit tests for size / type rejection, the request payload (part name, file name, type), error
  mapping and in-flight disabling; a manual run with a real camera photo (HEIC, large, with GPS) on
  iOS and Android, checking what the server stored.
