# CityLockers Partner API (Direction A, inbound)

Sell CityLockers locker capacity from your own product: discover locations, price a window, create and manage bookings, and hand your customer an access code.

> **This file is GENERATED from `docs/api/partner-api.openapi.yaml` and must not be edited by
> hand.** It reproduces that file's `info.description` in full, so this bundle, the rendered
> reference page and the Postman collection cannot describe different APIs. A hand edit is reverted
> by the next build and fails `npm run spec:postman`.

## What is in this bundle

| file | what it is |
|---|---|
| `README.md` | This file. |
| `partner-api.html` | The full API reference, rendered. Open in a browser. |
| `partner-api.openapi.yaml` | The OpenAPI 3.1 source every other file here is generated from. Point a client generator at this. |
| `partner-api.postman_collection.json` | Postman collection — all 11 operations, auth wired once at collection level. |
| `partner-api.postman_environment.json` | Postman environment — production host; `bearerToken` is empty, paste your key in. |

**Start with `partner-api.html`** — open it in a browser for the full endpoint-by-endpoint
reference. It renders offline; only its interactive chrome needs the network.

**Then import `partner-api.postman_collection.json` and
`partner-api.postman_environment.json` into Postman.** Set `bearerToken` in the environment to
the key you were issued. It ships empty, and no file in this bundle contains a credential.

**`Idempotency-Key` is wired to Postman's `{{$guid}}`** on create and extend, so every send
generates a fresh key. Leave it that way. A key is spent by its first outcome, **including a
rejection**, and a replay returns the stored result without re-reading your request body — so
pinning it to a literal means your first call burns it and every later call replays that outcome.

---

The Direction A partner API lets a partner sell CityLockers locker capacity
inside its own product. Every call is authenticated with a partner API key,
every result is scoped to the partner that key belongs to, and every response
— success or failure — uses the same envelope.

## The envelope

Every response body is a JSON object carrying `ok`.

* `ok: true` — the operation succeeded. The rest of the object is the
  operation's payload.
* `ok: false` — the operation was refused. `reason` is always present and is
  a machine-readable string from the vocabulary under *Reason codes*.
  `error` is sometimes
  present and is a human-readable sentence; **branch on `reason`, never on
  `error`**, and never on the HTTP status alone.

There is no separate error object and no `errors` array. A refusal is never
delivered as a bare HTTP status with an empty body.

## Getting started

1. **Ask your CityLockers contact for a partner API key.** Keys are issued by
   us; there is no self-service signup. The key is what defines which
   locations you may sell, which operations you may call, and which booking
   windows you are allowed to offer.
2. **Call `GET /bootstrap` once, at startup, and configure your UI from what
   it returns** — your locations, the locker sizes offered at each, your
   period policy, your granted operations, and the mode of the key you hold.
   **Do not hard-code any of it.** Every one of those is a property of the
   key and can change without a release on your side.
3. **Price a window with `GET /quote` and check stock with `GET
   /availability`** before you show a price or a "book" button. Neither
   creates anything.
4. **Create the booking with `POST /bookings`,** carrying an
   `Idempotency-Key`. You get back a booking `reference` and, where one
   exists, the customer's `access_code`.
5. **Manage it afterwards** with `GET /booking`, `POST /extend`,
   `POST /cancel`, `POST /end`, `POST /resend-code` and
   `POST /reissue-code`.

**You hold two keys, and the difference is the key, not the host.** A `live`
key transacts against your real scoped locations: real lockers, real money,
real customer notifications, and a commission entry on our ledger for every
create and every extend. A `test` key (0.268.0) uses the **same endpoints on
the same host** and is scoped to a shared **fixture location** — inventory
that is not sold to anyone. A booking made with a test key is stamped
`booking_type = 'test'` (it never appears in our staff booking views) and
**accrues no commission**: neither its create nor any later extend writes a
ledger row. Everything else — availability, quote, the access code, cancel,
end, the idempotency contract — behaves exactly as live. Integrate against
the test key; go live by swapping the credential. Read which one you hold
from `GET /bootstrap`'s `mode` field.

**Keep the key on your server.** Never put it in a browser, a mobile app, a
URL or a repository. It is a bearer credential: whoever holds it is you.

## The flow, in order

```
bootstrap ──▶ availability ──▶ quote ──▶ create ──▶ (extend | cancel | end)
   once          per search     per      per            per booking
 at startup                    window   basket
```

`bootstrap` is a startup call, not a per-request one. `availability` and
`quote` answer different questions and neither is authoritative: **create is
the only gate**. A window `availability` happily counts and `quote` happily
prices can still be refused by create — `period_not_allowed` if it falls
outside your key's period policy, `unavailable` if a concurrent booking took
the compartment first, `invalid` if it is under the engine's real floor.
Treat a create refusal as normal control flow, not as an exception.

## Routing: dispatch on the trailing path segment

The API is one Edge Function invoked at `/partner-bridge`. The sub-route is
the **trailing path segment** of the request URL, and the method then selects
the operation (`GET /bookings` lists, `POST /bookings` creates). This is
described here faithfully rather than reshaped.

Two consequences a generated client must respect:

1. **A bare `POST` with no path segment resolves to `bookings` (create).**
   `POST /partner-bridge` and `POST /partner-bridge/bookings` are the same
   operation. This is documented below as the path `/` and was proven live —
   a real second booking was created through a request with no trailing
   segment at all. It is an alias, not a twelfth endpoint.
2. **An unrecognised segment, or a recognised segment with the wrong method,
   returns `404 not_found`** — the same reason string a missing booking
   reference produces. A 404 therefore does not by itself mean "no such
   booking".

## Authentication

`Authorization: Bearer <key>` on every call, including the ones that can fail
before the key is checked (see the create endpoint's security note).

A raw key is `kpk_<mode>_<8hex>_<64hex>`, e.g.
`kpk_live_1a2b3c4d_<64 hex characters>`. Only the fourth segment is secret.
The first three segments together are the key's non-secret **prefix**
(`kpk_live_1a2b3c4d`); it is what the server stores alongside the SHA-256
hash of the full key, what the rate limiter buckets on, and the only part
safe to quote in a support ticket or a log line.

`mode` is a property of the key itself, `live` or `test`, and is the source
of truth for what the credential does. It is surfaced to you on
`GET /bootstrap` as the top-level `mode` field. Both modes are minted
(0.268.0). The mode is threaded server-side from the verified key into the
create and extend procedures: a `test` key's bookings are stamped
`booking_type = 'test'` and accrue no commission; a `live` key's are
`standard` and do. Nothing in a request body can change a key's mode.

**Legacy two-segment keys (`kpk_<8hex>_<64hex>`) no longer authenticate.**
Verification derives the prefix from the first three segments, so a
two-segment key yields a prefix that matches no stored row and is rejected
as `unauthorized`.

Verification hashes the full raw key with SHA-256 and constant-time compares
it against the stored hash, scoped to an **active key belonging to an active
partner**. Anything else — no key, malformed key, revoked key, suspended
partner — is `401 unauthorized`.

## Rate limiting

60 requests per minute per key prefix, over-limit → `429 rate_limited`. The
limiter is an in-process token bucket: it resets when the function instance
cold-starts and is not shared across instances, so the effective limit is a
floor, not a guarantee. Treat 429 as retryable with backoff. This behaviour
is expected to be replaced by a durable shared counter; do not build a client
that depends on the current leniency.

## Idempotency

`Idempotency-Key` is **required on the two operations that move money**:
`POST /bookings` (create) and `POST /extend`. It is not read on any other
endpoint. A missing or blank header is `400 invalid` before any work is done.

Replaying a key that has already been used returns the earlier request's
outcome instead of acting twice.

**A replay returns LIVE CURRENT STATE, not a frozen echo of the original
response.** This is the single most important thing to understand about
replay on this API, and it was proven live: a create replayed after the
booking had been extended and then cancelled came back with
`status: "cancelled"` and `total: 18`, where the original create had returned
`status: "active"` and `total: 9`. The create replay branch re-reads the
booking and customer rows at replay time. So on a create replay,
`status`, `total` and `access_code` describe the booking **now**;
only `reference` and `replayed: true` are stable facts about the original
request. (`POST /extend` differs: its replay returns the stored result
object of the original extension, stamped `replayed: true`.)

A replay of a request that was *rejected* returns the original rejection
reason with the original status. A replay that arrives while the first
request is still in flight returns `409 in_progress`.

## Booking windows: two shapes, and a floor that is not where you expect

**Send either `duration_minutes` or `end_time`, not both.** They are
alternative spellings of the same window and create accepts both:

* `duration_minutes` — an integer count of minutes from `start_time`. If you
  send both fields, **`duration_minutes` wins** and `end_time` is ignored.
* `end_time` — an ISO-8601 instant. The window is then derived from
  `start_time` (or from now, if you omit `start_time`).

Omit `start_time` to mean "now". `GET /availability` and `GET /quote` take
only `duration_minutes`, so if your UI works in end times you convert before
you price.

**The floor is 180 minutes, not the 60 the edge function accepts.** This is
the single most common way a first integration fails, and it fails *late*:
the edge function rejects anything under 60 minutes outright, but the booking
engine's smallest chip is **180 minutes**, so a 60–179 minute request passes
edge validation, reaches the engine, and comes back `400 invalid` with no
hint that duration was the problem. **If you offer sub-3-hour windows in your
UI, they will all fail.** Above a day, the window must be a whole number of
1440-minute blocks or it is `invalid_duration`.

## Reason codes, and why they are documented per endpoint

The full reason vocabulary the code can emit is:

| reason | usual status | meaning |
|---|---|---|
| `unauthorized` | 401, sometimes 403 | no usable key, or the partner is not active |
| `capability` | 403 | the partner lacks the *inbound* direction capability |
| `out_of_scope` | 403 | the location is not in this key's scope |
| `not_permitted` | 403 | the partner is not granted this operation |
| `not_found` | 404 | no such booking for this partner, or no such route |
| `unavailable` | 409 | nothing free / nothing priceable for the request |
| `in_progress` | 409 | a concurrent replay of the same Idempotency-Key |
| `invalid_state` | 409 | the booking is not in a state this operation accepts |
| `window_not_expired` | 409 | see the note below — **unreachable today** |
| `invalid` | 400 | malformed or missing input |
| `invalid_duration` | 400 | the duration is structurally wrong |
| `period_not_allowed` | 400 | the partner's period policy forbids this window |
| `rate_limited` | 429 | over the per-key request budget |
| `server_error` | 500 | an internal failure; retry is reasonable |

**`reason` alone does not determine the HTTP status, which is why this spec
models reason→status per endpoint per branch instead of publishing one global
table.** The concrete case: `unauthorized` is **401** when it comes from a
route's own key-verification branch (the common case — no key, bad key), and
**403** when it is returned by a wrapper RPC's *partner-not-active* branch
and mapped by the generic reason→status mapper. The two have never been
observed colliding, because key verification already scopes to an active
partner, so the 403 flavour is only reachable if a partner is deactivated in
the window between verification and the operation's own read. It is
nonetheless a real branch in shipped code, and a client keyed on a single
global `reason`→status table will mis-handle it. Each operation below lists
only the reasons that operation can actually produce, and each response says
which branch produces it.

Two further notes on the enum:

* **`window_not_expired` is unreachable.** It is documented here for
  completeness of the vocabulary only. The window-expiry guard on ending a
  booking was **deleted** in migration `20260728140000_end_booking_early.sql`,
  whose own header records that the reason string was left in the status maps
  deliberately, to avoid needless API churn. `POST /end` completes a live
  booking immediately, before its window closes — proven live. It is **not**
  listed as a response of `POST /end` anywhere in this document, and it will
  only become reachable if that deleted guard is reinstated.
* **`period_not_allowed` is real and was missing from the earlier written
  accounts of this API.** `POST /bookings` returns it (as a `400`) when the
  requested window falls outside the partner's own intraday/multiday policy.

## Grant vocabulary

A key's partner holds a set of granted operations. `GET /bootstrap` returns
them as `granted_operations`, an array of objects — `{ key, kind, label }`,
**not** an array of bare strings. The `key` values are the grant keys, and
each callable endpoint below names the one it requires in `x-grant-key`.

| grant key | kind | endpoint |
|---|---|---|
| `locations` | read | `GET /bootstrap` |
| `availability` | read | `GET /availability` |
| `quote` | read | `GET /quote` |
| `bookings.list` | read | `GET /bookings` |
| `booking.read` | read | `GET /booking` |
| `booking.create` | write | `POST /bookings` (and bare `POST /`) |
| `booking.cancel` | write | `POST /cancel` |
| `booking.end` | write | `POST /end` |
| `booking.extend` | write | `POST /extend` |
| `booking.resend_code` | write | `POST /resend-code` |
| `booking.reissue_code` | write | `POST /reissue-code` |

Calling an endpoint whose grant the partner does not hold is
`403 not_permitted`. A grant is checked inside the operation, so a missing
grant is reported after authentication, never instead of it.

Three further operation keys exist in the server-side registry but are **not
built** and have no endpoint: `booking.open`, `booking.add_lockers`,
`booking.overstay`. They are listed under `x-planned-operations` at the root
of this document so a client can recognise the strings, and they deliberately
appear nowhere as a path. If one of them shows up in your
`granted_operations`, there is still nothing to call.

## Behaviours that surprise integrators

* **Customer identity is first-write-wins by email.** Creating a booking
  resolves the customer by email (or phone) and reuses the existing row if
  one matches. A later create under the same email with a *different* `name`
  does **not** update the stored name, and the response echoes the stored
  one. Proven live: a create sending `"E2E Rerun B"` came back with the
  earlier `"E2E Rerun A"`. Do not use the returned `customer.name` to confirm
  what you sent.
* **`access_code` is not always present on a create.** See the create
  response schema — it is `string` or `null`, and the two conditions that
  produce `null` are named there.
* **`resend-code` and `reissue-code` do not email the customer.** They return
  the code in the response body and nothing else leaves the platform.
  Delivering it to the end customer is the partner's job. (`reissue-code`
  additionally enqueues a device event so the physical keypad learns the new
  code; until that is consumed, the OLD code still opens the door.)
* **`GET /bookings` is capped at 100 rows, newest first, with no
  pagination.** There is no cursor, no `offset` and no total count.
* **Times are UTC ISO-8601 instants.** Multi-day quoting anchors on the
  Asia/Dubai business date, which can make a multi-day quote indicative
  rather than exact; a create charges from the real booking start.

## Data handling

Creating a booking sends us your customer's contact details — an email
address, or a phone number, and optionally a name — because a booking
resolves to a customer record on our side and a confirmation notification is
sent to that address. That makes both parties handlers of personal data under
the **UAE Personal Data Protection Law (Federal Decree-Law No. 45 of 2021)**,
and a short data-handling understanding forms part of the partner agreement.
Agree it before you send live traffic, not after.

Practically, three rules:

* **Customer data goes in the request body, over HTTPS, never in a URL.** No
  endpoint here takes a customer identifier as a query parameter, and a URL
  is logged in places a body is not.
* **Send the minimum that identifies the customer.** One of `email` or
  `phone` is required; `name` is optional and is only used for the booking
  record.
* **Customer identity is first-write-wins by email** — see *Behaviours that
  surprise integrators*. The record we keep may not carry the name you sent,
  so do not treat the echoed `customer` object as confirmation of what you
  submitted.

## About the examples

**Every example in this document is illustrative.** The response examples are
reproduced from a real end-to-end run against the dev branch project on
2026-08-15, with every identifier, address, code and reference replaced by an
obviously synthetic placeholder. No example contains a real customer email, a
real access code, a real partner key or a real booking reference. Do not
treat any identifier here as callable, and do not copy an example UUID into a
request.
