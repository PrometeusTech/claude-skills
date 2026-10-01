---
name: notifications-integrations
description: Rules and review checklist for messages and third-party integrations — e-mails, push, SMS and WhatsApp notifications (when they are sent, to whom, how often, in which language, with what content), bulk sends to a whole tenant, user preferences and opt-out, inbound webhooks (signature verification, replay, ordering, idempotent processing, fast acknowledgement) and outbound calls to providers (e-mail/SMS/messaging services, payments, maps). Use it whenever you add or change a mailer, a notification, a message template, a recipient list, a webhook endpoint or handler, or a client for an external provider, and whenever you review such code — even if the task only says "notify the admin" or "handle the delivery status".
---

# Notifications and integrations

Messages leave the system and cannot be taken back: an e-mail sent for a transaction that rolled
back, a notification to a member of another tenant, the same WhatsApp message three times
because the job retried, a link to the staging site, a webhook that anyone can forge. The rules
below apply while building; the checklist is what a reviewer tries.

## 1. Send after the fact, once

- Notifications are triggered **after commit** of the change they describe (enqueue in
  `after_commit` or after the transaction in the service), never from inside the transaction.
- A retry must not send twice: record that a notification was sent (per recipient, per event) and
  check it before sending, or use the provider's idempotency key.
- Pass ids to jobs, re-load and re-check state when the job runs (the record may have changed or
  been deleted; the recipient may have left the tenant).
- With a non-durable queue, a lost job means a lost message: anything that must arrive is
  re-derivable from DB state (see deploy-safety §4).

## 2. The right recipients

- Recipient lists are computed from the tenant of the record and the roles the product named —
  not "all users with role X" across tenants, and not inactive or removed members.
- Personal data of one user is not sent to another unless the product requires it (no member's
  phone number in a message to other members).
- User preferences and opt-outs are honored per channel; legally required messages (password,
  security) are the only exceptions, and are labeled as such.

## 3. Volume and pacing

- Bulk sends (whole tenant, all tenants) are batched, throttled to the provider's limits, and
  recorded per recipient so a crash can resume without resending.
- A user action cannot trigger unbounded messages (comment storms, repeated status flips):
  debounce or digest.

## 4. Content

- Templates are translated (recipient's language, with a fallback), escape user content, and do
  not include secrets or tokens except purpose-made, expiring links.
- Links use the configured public URL of the environment (never a hard-coded host or
  `localhost`), and land on a page that works for the recipient's role after login.
- Channels with rules (WhatsApp 24-hour window and approved templates, SMS length, push payload
  size) are handled: outside the window, use an approved template or fall back to another channel.

## 5. Inbound webhooks

- Verify the signature (HMAC over the raw body, constant-time compare, provider's secret) and
  the timestamp (reject old ones to prevent replay) before parsing anything.
- Store the raw event (with the provider's event id, unique) before processing; a duplicate event
  id is acknowledged and ignored.
- Acknowledge fast (2xx) and process in a job; the handler is idempotent and tolerates events out
  of order (a "delivered" before "sent", a status older than the stored one is ignored).
- Unknown event types are stored and logged, not 500.
- The endpoint is public for authorization purposes, rate limited, and its body size capped.

## 6. Outbound provider calls

- Explicit timeouts, bounded retries on idempotent calls, typed errors, reporting
  (see idempotency-retries §6).
- Credentials per environment; a missing credential fails at boot or disables the channel
  explicitly — it does not silently skip sending while reporting success.
- Sandbox/test mode in non-production environments so tests and staging never message real users.

## Implementation checklist

- [ ] Enqueued after commit; sent-once record or provider idempotency; job re-checks state.
- [ ] Recipients from the record's tenant and named roles; active members only; preferences honored.
- [ ] Bulk sends batched, throttled, resumable; user actions cannot spam.
- [ ] Templates translated, escaped, environment URL, no secrets; channel rules handled.
- [ ] Webhooks: signature + timestamp verified on the raw body, raw event stored with unique id,
      fast ack, idempotent and order-tolerant processing, unknown types tolerated.
- [ ] Provider clients: timeouts, retries, boot-time credentials, sandbox outside production.
- [ ] Tests assert who receives what, and that a rollback / retry / duplicate sends nothing extra.

## Review / judge checklist (try to break it)

1. Make the triggering transaction fail after the notify call: was anything sent or enqueued?
2. Run the job twice (or retry it after a provider error): one message or two?
3. List recipients for a record as the code computes them; add a member of another tenant, an
   inactive member, a user who opted out — any of them included?
4. Read the rendered template in each language with hostile user content (HTML, long text); check
   links point to the environment's public URL and work for that role.
5. Bulk send to a large tenant with the provider stubbed to throttle/fail midway: resumes without
   duplicates?
6. Webhook: wrong signature, missing signature, valid signature with an old timestamp, the same
   event id twice, events in reverse order, an unknown event type, a huge body.
7. Remove a provider credential: boot error or explicit "channel disabled", not silent success.
