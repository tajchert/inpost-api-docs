# InPost Mobile API — Unofficial Documentation

Practical, build-ready documentation for the **private HTTP API** behind InPost
parcel flows (Paczkomaty / parcel lockers in Poland). It exists so you can build
your own client — a script, a widget, a home-automation hook, a nicer app.

> ⚠️ **Unofficial.** This project was built on publicly available information
> and data. It is **not** published, supported, or endorsed by InPost. There is
> no SLA; endpoints, fields, and auth can change or be blocked without notice.
> Use it for personal/interoperability purposes, be a good citizen (no hammering,
> no abuse), and check InPost's Terms of Service before doing anything at scale.

---

## What's in this folder

| File | What it is | Use it when… |
|------|-----------|--------------|
| **`README.md`** (this file) | Human orientation: concepts, the happy path, and what's easy to get wrong. | Start here. |
| **`API_REFERENCE.md`** | The deep reference. Every endpoint, every field, status enums, auth internals, error codes, and a full endpoint inventory (incl. loyalty/shopping). | You need the exact field name / status value / detail. |
| **`openapi.yaml`** | OpenAPI 3.0 spec for the **core** flows (auth, parcels, compartment, points, notifications). Valid; loads in Swagger UI / Redoc / codegen. | You want generated client code, a Postman collection, or an interactive explorer. |

Scope of the OpenAPI file is the **consumer core**. The long tail (loyalty/
"InCoins", shopping "izi", Nimbus AI, sending, returns, payments) is *inventoried*
in `API_REFERENCE.md` Appendix A but not modeled field-by-field.

---

## What you can do with it

- **List your inbound parcels** and their shipping status.
- **Show a parcel's pickup QR code** (the one a Paczkomat scanner reads).
- **Open a locker compartment remotely** to collect a parcel.
- **Find Paczkomaty / pickup points** near a location.
- Log in, refresh tokens, manage tracked parcels, read notifications.

The whole thing is a normal JSON-over-HTTPS API with Bearer tokens. If you've
consumed any REST API, you already know how this works.

---

## 60-second mental model

1. **One base URL** (note the `/global`):
   `https://api-inmobile-pl.easypack24.net/global`
2. **Log in** → get an **access token** (`authToken`) + a **refresh token**.
3. Put `Authorization: Bearer <accessToken>` on every call.
4. `GET /v4/parcels/tracked` → your parcels, each with a `status` and (when
   waiting in a locker) an `openCode` + `qrCode`.
5. To collect: either **show the `qrCode`** at the machine, **or** run the
   **remote-open** sequence (`validate → open → status → closed → claim`).

That's the entire happy path. Everything else is detail.

---

## Quick start — the happy path

### 1. Authenticate (SMS flow — recommended for non-app clients)

Don't use the OAuth flow unless you must (it's WebView-based and painful to
automate). The SMS flow is three JSON calls:

```http
POST /v1/account
Content-Type: application/json

{ "phoneNumber": { "prefix": "+48", "value": "600123456" } }
```
→ user receives an SMS code. Some backend versions return an empty 2xx body;
do not require a JSON response here.

```http
POST /v1/account/verification
Content-Type: application/json

{
  "phoneNumber": { "prefix": "+48", "value": "600123456" },
  "smsCode": "1234",
  "devicePlatform": "Android"
}
```
→ `{ "authToken": "...", "refreshToken": "..." }`

Store both. When `authToken` expires (you'll get a `401`), mint a new one:

```http
POST /v1/authenticate
Content-Type: application/json

{ "refreshToken": "...", "phoneOS": "Android" }
```

### 2. List inbound parcels

```http
GET /v4/parcels/tracked
Authorization: Bearer <authToken>
```
→ `{ "parcels": [ ... ], "removedParcelList": [...], "more": false }`

If `more` is `true`, call the **same** endpoint again and append — pagination is
server-stateful, there is no cursor.

### 3. Show the QR / read the codes

Each parcel waiting in a locker has:
- `openCode` — the **numeric** code (locker keypad / remote open),
- `qrCode` — an **opaque string** you render as a QR for the locker's scanner.

### 4. Open the compartment remotely

```
POST /v2/collect/validate          { parcel:{shipmentNumber, openCode}, geoPoint:{lat,lng,accuracy} }
   → { sessionUuid }
POST /v1/collect/compartment/open  { sessionUuid }   → which door, timers
POST /v1/collect/compartment/status{ sessionUuid, expectedStatus:"OPENED" }   (poll)
   … user takes parcel, shuts door …
POST /v1/collect/compartment/status{ sessionUuid, expectedStatus:"CLOSED" }   (poll)
POST /v1/collect/compartment/closed{ sessionUuid }
POST /v1/collect/compartment/claim { sessionUuid, shipmentNumbers:[...] }
```

`sessionUuid` from step 1 threads through every later call.

> **Two backends behind this flow.** There is both a **legacy** implementation
> (bare `/v1|v2/collect/…`, single-market PL) and a newer **global backend**
> (`/remote-opening/v1/collect/…`, country-aware, extra `xCountryCode` header). A
> remote-config flag (`isParcelLockerGlobalBackendEnabled`, **default `false`**)
> picks one per call. The **legacy** paths are the practical default target; the
> two share identical `status`/`closed` semantics. Details in
> `API_REFERENCE.md §7.0`.

---

## Things that are easy to get wrong (read this)

1. **Do not blindly concatenate `/global` with every path.** The default base is
   `https://api-inmobile-pl.easypack24.net/global`, but several documented
   endpoints are root-relative (`/v1/account`, `/v4/parcels/tracked`). In
   standard HTTP clients, a leading `/` drops the base path. Verify the final
   URL your client sends; SMS auth and tracked-parcel calls can work
   working at host-root paths such as
   `https://api-inmobile-pl.easypack24.net/v1/account`.

2. **A parcel has TWO different codes.** `openCode` ≠ `qrCode`. `openCode` is the
   short numeric code; `qrCode` is a long opaque token. The QR screen encodes
   `qrCode` **verbatim** — don't QR-encode the `openCode` and expect it to work,
   and don't try to derive one from the other.

3. **Legacy SMS phone numbers include the `+` in the structured prefix.** Send
   `"+48"` for the SMS login flow. Phone is a structured object
   `{prefix, value}`, never a flat string.

4. **`status` is a free-form string, not an enum** in the payload (e.g.
   `ready_to_pickup`, `stack_in_box_machine`, `delivered`). Match on the strings
   listed in `API_REFERENCE.md §5.4`. Use `statusGroup` for a coarse bucket. To
   decide "can I open it now?", check `operations.collect == true`. (A parcel
   field `mobileCollectPossible` used to exist in the v1 API and is easy to
   assume is still here — but the v4 `/parcels/tracked` response does **not**
   return it. Do not gate on it; `operations.collect` is the signal.)

5. **Remote open is GPS-gated, but the GPS is whatever you send.** `validate`
   takes a `geoPoint` from the client. The proximity check is effectively a
   client-side guardrail, not a hard server gate — which is why CLI clients can
   open lockers without "being close". Don't build on the assumption that the
   server strictly enforces your location; equally, don't abuse it.

6. **The locker timers are durations, not timestamps.** `actionTime`,
   `confirmActionTime`, `openCompartmentWaitingTime` are **milliseconds to add to
   "now"**, not absolute epoch times.

7. **`expectedStatus` only has two values:** `OPENED` and `CLOSED`. Poll for
   `OPENED` after opening, then `CLOSED` after the door shuts.

8. **`/v4/parcels/tracked` takes no query parameters** and returns the full set.
   There's no `updatedAfter`/delta sync. `updatedUntil` is returned, but a
   simple client should re-fetch and reconcile (honor `removedParcelList[]`).

9. **Dates are ISO-8601 with offset** (`2024-06-30T14:30:00.000+02:00`). Parse as
   offset date-times, not naive local times.

10. **Tokens: refresh reactively on `401`.** There's no documented proactive
    skew. Simplest correct behavior: call → on 401, refresh once → retry.

11. **TLS certificate pinning** in native clients does **not** affect your
    own client calling the API; it only affects traffic inspection of those
    clients.

---

## Headers: what you actually need

Minimal working set, consistent with
[`alufers/inpost-cli`](https://github.com/alufers/inpost-cli):

```
Authorization: Bearer <accessToken>     # on authenticated calls
Content-Type: application/json            # on requests with a body
Accept: application/json
```

Common clients may also send `User-Agent`, `Device-ID`, `device-uid`,
`Session-ID`, `X-App-Id`, `Accept-Language` — these are **not server-enforced**
(telemetry / anti-abuse). Add a plausible `User-Agent` and a persisted random
`device-uid` only if a v4/v5 endpoint ever pushes back. See
`API_REFERENCE.md §3, §11b`.

---

## Errors

All errors share one envelope:

```jsonc
{
  "error": "sessionExpired",   // machine-readable, camelCase — switch on this
  "description": "…",          // human text (nullable)
  "status": 409,               // echoes HTTP status (nullable)
  "diagnosticId": "…"          // quote this if you contact InPost
}
```

Codes you'll meet on the locker flow: `invalidSession`, `sessionExpired`,
`invalidSessionState`, `invalidCompartmentState`, `invalidParcelState`,
`cannotFindCompartment`, `boxMachineNotFound`. On `sessionExpired` /
`invalidSessionState`, start over from `validate`. Full list in
`API_REFERENCE.md §11b`.

---

## Using the OpenAPI file

```bash
# Interactive docs
npx @redocly/cli preview-docs openapi.yaml
# or
npx @redocly/cli lint openapi.yaml        # validates (warnings are cosmetic)

# Generate a client (examples)
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g typescript-fetch -o ./client-ts
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g swift5 -o ./client-swift
npx @openapitools/openapi-generator-cli generate -i openapi.yaml -g python -o ./client-py
```

Import `openapi.yaml` into Postman/Insomnia to get a ready request collection.
Set the base server, drop your Bearer token into the auth, and go.

Notes for codegen consumers:
- Some long-tail response bodies (sent parcels, notifications) are modeled as
  open objects (`additionalProperties: true`) — fields not fully extracted.
- A few nested objects are referenced via `allOf`; treat them as optional.

---

## Notes & Keeping This Current

- This project was built on publicly available information and data.
- Cross-check useful behavior against
  [`alufers/inpost-cli`](https://github.com/alufers/inpost-cli), especially the
  auth model, the `openCode`/`qrCode` split, and the collect state machine.
- Verify exact HTTP statuses, optional headers, and edge-case responses against
  live behavior before relying on them.
- When behavior changes, update `openapi.yaml` and note the change here.

---

## A note on responsible use

This documentation is for interoperability and personal use. Don't use it to
hammer InPost's servers, scrape other people's data, or anything you wouldn't be
comfortable explaining. The remote-open flow in particular is tied to real
physical lockers — only open compartments for parcels that are actually yours.
