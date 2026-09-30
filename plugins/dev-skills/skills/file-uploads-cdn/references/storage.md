# Storage backends

## BunnyNet (Storage Zone + Pull Zone)

- **Storage Zone** holds the objects; its API key is a server secret (env var), used only for
  PUT / DELETE / LIST from the backend. It never reaches the browser.
- **Pull Zone** serves objects over the CDN. For private files enable **Token Authentication** on
  the pull zone and issue signed URLs from the backend (token + expiry, optionally bound to a path
  prefix). Keep expiries short (minutes) for private documents.
- Follow BunnyNet's current documentation for the exact token format (hash input order, base64url,
  optional path/IP parameters) and reuse the project's existing signer if there is one — do not
  hand-roll a second implementation.
- Keys: `<env>/<tenant-id>/<model>/<uuid>.<ext>`. Deleting the row deletes the object via the
  Storage API after commit.
- Uploads from the backend: stream the tempfile to the Storage API (PUT with the key); check the
  HTTP status; on failure raise a typed error the service turns into a clean response.
- Caching: private files should not be cached publicly by intermediaries beyond the token TTL;
  replaced files get a new key rather than relying on purge.
- Local development usually has no BunnyNet credentials: the project should fall back to local disk
  (or a stub) behind the same interface, and tests stub the client.

## Local disk on a VM

- Store outside the web root (e.g. `/var/lib/<app>/uploads`), owned by the app user, not
  world-readable.
- Serve through the app after the authorization check. With nginx, use `X-Accel-Redirect` to an
  `internal` location so the file is sent by nginx without being public.
- Watch disk space (alerts), include the directory in backups, and keep the same key scheme as for
  the CDN so a later migration is a copy.
- Never let a user-controlled value become part of the filesystem path.
