---
name: idempotency-retries
description: Rules and review checklist for operations that may run twice — idempotency keys on create/upload endpoints, double submits, client and proxy retries, concurrent requests for the same thing, unique-constraint races, and calls to external services (storage/CDN, payment, email, messaging) with timeouts and bounded retries. Use it whenever you add or change an endpoint that creates something or has a side effect, an `Idempotency-Key` header, a shared idempotency store, a retry loop or an HTTP client for an external service, and whenever you review such code — even if the task only says "make create safe to retry" or "upload to the CDN".
---

# Idempotency, retries and races

Networks retry. Users double-click. Mobile clients resend after a timeout the server never saw.
A create endpoint that is not designed for this produces duplicates, orphaned side effects
(a file uploaded, a mail sent, a payment taken) and confusing errors on the second attempt. The
rules apply while building; the checklist is what a reviewer fires at the endpoint.

## 1. Validate the key before doing anything

- Check the idempotency key's presence/format/length **before** any side effect. A key the store
  cannot save (too long for its column, wrong charset) must be a clean 4xx — not an effect that
  happened with the key silently unsaved, followed by a generic error.
- Scope keys by caller and operation (user + tenant + endpoint), so one user's key cannot replay
  another's response.
- Store a fingerprint of the request body with the key; the same key with a different body is a
  4xx (`idempotency_key_reused`), not a replay of the old response.

## 2. Reserve the key, then act, then record the result

The common bug is "do the work, then save the key": two concurrent requests with the same key both
do the work, and the second fails on a uniqueness rule with an error the client did not expect.

1. **Reserve** the key atomically (insert a row with status `in_progress`; a unique index on the
   scoped key makes the second insert fail).
2. A request that finds the key `in_progress` answers 409 (or waits briefly), never re-runs the
   effect.
3. Do the work. On success, store the status code and response body with the key; on failure that
   left no effect, release the key so the client can retry; on failure after a partial effect,
   compensate first (see the file-uploads-cdn skill §5).
4. Keys expire (e.g. 24 h) and are cleaned up.

## 3. Decide what a replay means after the resource changed

Replaying the stored response is correct for "the same request arrived again". Decide explicitly,
and document, what happens when the resource was since **deleted or changed**: return the stored
response anyway (stale, may point at a deleted record), or 410/404. Whatever you choose, test it.

## 4. Own the store in one place

A shared idempotency store belongs to shared infrastructure, not to whichever domain built it
first. A new feature should not import another domain's store class (`Onboarding::…Store`) —
extract a generic one, or note the coupling as a follow-up.

## 5. Uniqueness races

- Every "must be unique" rule has a unique index; the index violation is caught and mapped to the
  same error code as the validation.
- Check-then-insert in code is not enough under concurrency; the index is the guarantee.

## 6. External calls: timeouts, bounded retries, no infinite waits

- Every HTTP client to an external service has explicit **open and read timeouts** (seconds, not
  the library default, which may be 60 s or unlimited) so one slow provider cannot pin web workers.
- Retry only idempotent operations (PUT with the same key, GET, DELETE), a small fixed number of
  times with backoff; never retry a non-idempotent POST blindly.
- Failures become a typed error the service turns into a clean response; they are reported to the
  error tracker.
- Credentials per purpose (storage write key ≠ URL-signing key); no silent fallback from one to the
  other.

## 7. Frontend

- Generate the idempotency key once per form opening (not per click) and reuse it on retry.
- Disable submit while the request is in flight; ignore a second submit.
- A 409 "in progress" or a replayed success is shown as success, not as an error.

## Implementation checklist

- [ ] Key validated (format, max length matching the store column) before any effect.
- [ ] Key scoped by user/tenant/operation; body fingerprint stored and compared.
- [ ] Reserve → act → record, atomically; concurrent same-key request never re-runs the effect.
- [ ] Replay-after-delete behavior decided, documented, tested.
- [ ] Store lives in shared infrastructure; no cross-domain import.
- [ ] Unique indexes behind uniqueness rules; violations mapped to the documented error code.
- [ ] External clients: explicit timeouts, bounded retries on idempotent calls only, typed errors.
- [ ] Frontend: one key per form opening, submit disabled in flight.

## Review / judge checklist (try to break it)

1. Same key, same body, sequentially → second response identical, no second effect (count rows,
   storage objects, mails).
2. Same key, **concurrently** (two or more parallel requests) → exactly one effect; the others get
   the replay or 409 — not a uniqueness error, not a duplicate.
3. Same key, different body → 4xx, no effect.
4. Key of 255, 256, 1000 characters, with spaces/unicode → clean 4xx before any effect (check
   storage for an uploaded object and the DB for a row).
5. Create, delete the resource, replay the key → behavior matches the documented decision.
6. Kill the external service (stub it to hang or 500) → response time bounded by the timeout,
   clean error, no row pointing at nothing, workers not exhausted.
7. Double-click the submit button with network throttled → one request, one record.
