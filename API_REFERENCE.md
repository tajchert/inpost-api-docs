# InPost Mobile API — Unofficial Reference

> Audience: senior engineer who has **no other documentation**. This is an
> unofficial reference for private InPost parcel flows, built on publicly
> available information and data. There is no public contract, no SLA, and
> InPost can change or block it at any time.
>
> Scope priority (per request): **inbound packages + their shipping status, the parcel QR code, and opening a Paczkomat compartment**. Secondary areas (loyalty/InCoins, shopping/"izi", sending parcels, returns, payments) are inventoried in the appendix but not detailed.

---

## 1. Architecture at a glance

- **Stack:** JSON-over-HTTPS with Bearer authentication.
- **Models:** request and response bodies use conventional JSON field names,
  with endpoint-specific optionality noted below.
- **Two backends** are used:
  1. **InPost Mobile API** (the "easypack24" host) — parcels, lockers, points, SMS login, etc. This is the one you care about.
  2. **InPost Group account/OAuth** (`account.inpost-group.com`) — the modern identity provider (login + token issuance).

---

## 2. Base URLs & environments

The mobile API commonly uses a `/global` base path, but endpoint paths are
mixed: some are relative to that base path and some are root-relative paths that
start with `/`.

| Environment | Base URL |
|---|---|
| **Production** | `https://api-inmobile-pl.easypack24.net/global` |
| Production (Cloudflare FQDN variant, feature-flagged) | `https://app-inmobile-pl.easypack24.net/global` |
| Integration/debug | `https://int-01-pl-inmobile-api-gateway.easypack24.net/global` |
| Test/mock | `https://test-api-inmobile-pl.easypack24.net/global` |

For standard HTTP clients, a leading-slash path drops the `/global`
base path. Do not blindly concatenate `/global` with every documented path.
Live custom-client testing found the SMS auth endpoints and tracked-parcel
endpoints working at host-root paths, for example:
`POST https://api-inmobile-pl.easypack24.net/v1/account` and
`GET https://api-inmobile-pl.easypack24.net/v4/parcels/tracked`.

When implementing a client, preserve the documented path style:
`v2/collect/validate` is relative to the configured base path, while
`/v1/account` is host-root relative.

**OAuth / identity host**:
- Issuer / authority: `https://account.inpost-group.com`
- Self-service account portal: `https://myaccount.inpost-group.com`
- `client_id`: `inpost-mobile`
- `redirect_uri`: `https://account.inpost-group.com/callback`

---

## 3. Headers sent on every request

Common clients may send these:

| Header | Value / format | Notes |
|---|---|---|
| `Authorization` | `Bearer <accessToken>` | Required on authenticated calls |
| `User-Agent` | Any stable client identifier | Optional |
| `X-App-Id` | App/client identifier | Optional |
| `Device-ID` | Device identifier | Optional |
| `device-uid` | Persisted random UUID | Optional |
| `Session-ID` | Fresh UUID per launch/session | Optional |
| `Accept-Language` | Current locale language tag (e.g. `pl`, `en`) | Optional |

Endpoints that take an `Authorization` token cover essentially everything except the login bootstrap calls (`v1/account`, `v1/account/verification`, OAuth `authorize`/`token`).

**No request body signing / HMAC** is used.

TLS certificate pinning in native clients does **not** affect your own
client calling the API; it only affects traffic inspection of those clients.

---

## 4. Authentication

There are **two** login paths: OAuth against InPost Group, and phone-number +
SMS directly against the mobile API. Both ultimately produce a **Bearer access
token** + **refresh token** that you attach as `Authorization`.

### 4.1 Token model

Token responses normalize to `accessToken`, `refreshToken`, optional `idToken`,
`expiresInSeconds`, and a token provider marker. Token type is **Bearer**. The
access token is a JWT; useful claims include:

- OAuth/GIP token claims: `jti`=deviceId, `sub`=userId, `phone`, `phone_prefix`, `exp`, `iat`.
- Legacy ("IMAG") token claims: `did`=deviceId, `client_id`=userId, `sub`=phone, `exp`, `iat`.

Refresh before expiry if you decode `exp`, and always refresh reactively on HTTP
`401`.

### 4.2 OAuth 2.0 / OIDC flow (primary, "Konto InPost")

Authorization-code flow with **PKCE (S256)**, commonly driven through a web
login where the `code` is delivered to the configured callback.

**Step 1 — Authorize (open in WebView):**
```
GET https://account.inpost-group.com/oauth2/authorize
  ?client_id=inpost-mobile
  &redirect_uri=https://account.inpost-group.com/callback
  &response_type=code
  &response_mode=form_post
  &scope=openid
  &code_challenge=<BASE64URL(SHA256(code_verifier))>
  &code_challenge_method=S256
  &state=<random>
  &nonce=<random>
  &prompt=login
  &lang=pl
  (also sends: fb_installation_id, app_ga_session_id, app_ga_instance_id, theme, supported_markets)
```
The user authenticates on InPost's pages; the callback delivers `code` (via `form_post` to `…/callback`).

**Step 2 — Exchange code for tokens:**
```
POST https://account.inpost-group.com/global/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=inpost-mobile
grant_type=authorization_code
redirect_uri=https://account.inpost-group.com/callback
code_verifier=<the PKCE verifier from step 1>
code=<code from callback>
```
Response `LogInPostTokensResponse`: `{ access_token, refresh_token, id_token, expires_in }`.

**Refresh:**
```
POST https://account.inpost-group.com/global/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=inpost-mobile
grant_type=refresh_token
refresh_token=<refresh_token>
```
→ same token response shape.

There is also a **token-exchange** grant (`grant_type` with `subject_token` + `audience` → `ExchangeCodeResponse`) used to trade the Group token for a mobile-API token; and **logout** via `GET …/global/oauth2/logout?id_token_hint=<id_token>` and `…/global/oauth2/logout-all`.

### 4.3 Legacy phone + SMS flow (mobile API)

Phone numbers are passed as a **structured object**:
`InternationalPhoneNumberNetwork = { "prefix": "+48", "value": "600123456" }`,
never a flat string.

**Step 1 — Request SMS code:**
```
POST /v1/account
Content-Type: application/json

{ "phoneNumber": { "prefix": "+48", "value": "600123456" } }
```
Response `SendSMSCodeResponse`: `{ "expirationTime": "<ISO-8601 ZonedDateTime>" }`
or an empty 2xx body, depending on backend version/client path. Treat any 2xx as
SMS dispatched.

**Step 2 — Confirm SMS code → get tokens:**
```
POST /v1/account/verification
Content-Type: application/json

{
  "phoneNumber": { "prefix": "+48", "value": "600123456" },
  "smsCode": "1234",
  "devicePlatform": "Android"      // NOTE: Kotlin field is phoneOS, JSON key is "devicePlatform"
}
```
Response `ConfirmSMSResponse`: `{ "authToken": "...", "refreshToken": "..." }`.
(`authToken` is the access token; attach as `Authorization: Bearer <authToken>`.)

**Step 3 (refresh) — re-authenticate with refresh token:**
```
POST /v1/authenticate
Content-Type: application/json

{ "refreshToken": "...", "phoneOS": "Android" }
```
Response `AuthenticationResponse`: `{ "authToken", "reauthenticationRequired"?, "refreshTokenExpiryDate"?, "pushIdStatus"? }`.

**Resend / reconfirm helpers**:
- `GET /v1/sendSMSCode` → `SendSMSCodeResponse`
- `POST /v1/reconfirmSMSCode` body `{ "refreshToken", "smsCode" }` → `ConfirmSMSResponse`

### 4.4 Logout & account deletion

- `POST /v1/logout` and `POST /v1/logout-all` — server-side session revoke.
- `DELETE /v1/account` — delete the user account.

---

## 5. Inbound packages (tracked parcels) — the core feature

All endpoints in this section require auth.

### 5.1 Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/v4/parcels/tracked` | **List all inbound/tracked parcels** for the user |
| `GET` | `/v4/parcels/tracked/{shipmentNumber}` | Single parcel detail (refreshes one parcel) |
| `GET` | `/v4/parcels/multi/{multiCompartmentUuid}` | All parcels in a multi-compartment delivery |
| `POST` | `/v4/parcels/observed` | Start tracking a parcel by number (add to "tracked") |
| `DELETE` | `/v4/parcels/observed/{shipmentNumber}` | Stop tracking / remove a parcel |
| `POST` | `/v4/parcels/properties` | Set a parcel property (e.g. mark/flag) |
| `POST` | `/v4/parcels/shared` | Share parcel(s) with friends (by friend UUIDs) |

Sent parcels (outbound) use separate endpoints:
`GET /v4/parcels/sent`, `GET /v4/parcels/sent/{shipmentNumber}`.

### 5.2 `GET /v4/parcels/tracked` — list response

`TrackedParcelsResponse`:
```jsonc
{
  "parcels": [ TrackedParcelNetwork, ... ],
  "removedParcelList": ["<shipmentNumber>", ...],   // nullable; numbers no longer tracked
  "updatedUntil": "<ZonedDateTime>",                // nullable
  "more": false                                      // boolean: more pages/data available
}
```

> The detail endpoint `GET /v4/parcels/tracked/{shipmentNumber}` accepts an optional header `notification-id` (`@Header("notification-id")`) — used when opening from a push; can be omitted.

### 5.3 `TrackedParcelNetwork` — the parcel object (full field list)

JSON keys shown (where the Kotlin field name differs, the JSON key is authoritative). Types: `S`=String, `ZDT`=ISO-8601 ZonedDateTime, `B`=Boolean. "req" = always present.

| JSON key | Type | Notes |
|---|---|---|
| `shipmentNumber` | S, req | Parcel tracking number |
| `shipmentType` | S, req | Free-form string, e.g. parcel vs courier type |
| `status` | S, req | **Detailed status** — see §5.4 |
| `statusGroup` | S, req | Coarse status bucket. **UPPERCASE on the wire** — observed `DELIVERED`, `CLAIMED`; also `READY`, `IN_DELIVERY`, `CREATED`, `OTHER` (match case-insensitively) |
| `events` | `EventNetwork[]`, req | Human-facing timeline entries |
| `eventLog` | `EventLogNetwork[]`, req | Structured event log |
| `openCode` | S, nullable | **Numeric pickup/open code** for the locker compartment |
| `qrCode` | S, nullable | **Opaque QR payload** to display at the locker scanner (≠ openCode) |
| `expiryDate` | ZDT, nullable | Pickup deadline |
| `storedDate` | ZDT, nullable | When placed in locker |
| `pickUpDate` | ZDT, nullable | When collected — present on finished parcels; the reliable "completed on" timestamp (see §5.3d) |
| `returnedToSenderDate` | ZDT, nullable | |
| `parcelSize` | S, nullable | Size code — usually a letter `A`–`J`, but the literal `OTHER` also occurs (see §5.3b) |
| `receiver` | `InternationalCustomerNetwork`, nullable | `.name` = receiver display name |
| `sender` | `InternationalCustomerNetwork`, nullable | `.name` = merchant/sender display name (populated on the list too — usable as a row title, see §5.3c) |
| `pickUpPoint` | `BoxMachineNetwork`, nullable | The Paczkomat/point — see §7.2 |
| `cod` | `CashOnDeliveryNetwork`, nullable | (Kotlin field `cashOnDelivery`) |
| `multiCompartment` | `MultiCompartmentNetwork`, nullable | Set when part of a multi delivery |
| `mobileCollectPossible` | B, nullable | **Legacy v1 field — NOT returned by v4 `/parcels/tracked`.** Do not gate on it; use `operations.collect` |
| `endOfWeekCollection` | boolean | |
| `internationalParcel` | B, nullable | |
| `operations` | `TrackedParcelOperationsNetwork`, req | **Capability flags** — see below |
| `avizoTransactionStatus` | S, nullable | |
| `courierPhoneNumber` | `InternationalPhoneNumberNetwork`, nullable | |
| `inEasyAccessZone` | B, nullable | |
| `sharedTo` | `FriendNetwork[]`, req | Who *I* shared this parcel **out** to (usually empty). Distinct from `ownershipStatus` |
| `ownershipStatus` | S, req | **`OWN`** = you own it, **`FRIEND`** = shared *with* you. Verified live; **not** "OWNER"/"OWNED" (see §5.3d) |
| `economyParcel` | boolean | (Kotlin field `isEconomy`) |
| `carbonFootprint` | `CarbonFootprintNetwork`, nullable | |

**`TrackedParcelOperationsNetwork`** (what the user is allowed to do with this parcel — drives the UI):
`manualArchive`(bool), `autoArchivableSince`(ZDT?), `delete`(bool), **`collect`(bool — can be opened/collected)**, `highlight`(bool), `refreshUntil`(ZDT?), `redirectionUrl`(S?), `expandAvizo`(bool), `requestEasyAccessZone`(S, req), `voicebot`(bool), `canShareToObserve`(bool), `canShareOpenCode`(bool), `canShareParcel`(bool), `withDonations`(B?).

**`EventNetwork`** (timeline): `date`(ZDT,req), `eventTitle`(S,req), `eventDescription`(S?), `location`(`EventLocationNetwork{countryCode, city?}`?), `eventCode`(S?).
**`EventLogNetwork`**: `type`(S,req), `name`(S,req), `date`(ZDT,req), `details`(`EventLogDetailsNetwork{invoiceRequested:bool}`?).
**`CashOnDeliveryNetwork`**: `price`(S?), `paid`(B?), `url`(S?), `payCode`(S?), `transactionStatus`(S?).
**`InternationalCustomerNetwork`** (sender/receiver party): `name`(S? — display name, company or person, e.g. *"Amazon Polska"*). On `receiver` this object also carries `phoneNumber`(`InternationalPhoneNumberNetwork{prefix,value}`) and `email`(S); on `sender` only `name` was observed. Other customer/address fields may be present but are not relied upon.
**`FriendNetwork`** (`sharedTo[]` entries): `uuid`(S), `name`(S), `phoneNumber`(`{prefix, value}`).
**`MultiCompartmentNetwork`**: `uuid`(S,req), `shipmentNumbers`(S[]?), `presentation`(bool), `collected`(bool), `carbonFootprint`(?).

### 5.3a Parcel tracking history (the "state history" timeline) — no dedicated endpoint

The in-app **"Parcel tracking" view** (chronological list of states with dates/times,
e.g. *"Delivered" / "Ready for collection" / "Accepted at the logistic center"*) is
**not** served by a separate tracking endpoint. There is no `/track`, `/history`,
`/events`, or `/timeline` route. The timeline is delivered **inline**, as the
`events` array inside each `TrackedParcelNetwork`.

Open a parcel detail view by calling
`GET /global/v4/parcels/tracked/{shipmentNumber}`. The same `events` array is
also already present on every parcel in the `GET /global/v4/parcels/tracked`
list response, so a list screen can render the latest state without a second
call. Nothing extra is required beyond the normal `Authorization: Bearer`.

Two parallel arrays carry the history:

**`events` — `EventNetwork[]` (the human-facing timeline shown in the UI):**

| Field | Type | Notes |
|-------|------|-------|
| `date` | ZonedDateTime, req | Timestamp of the state (this is the date/time shown per row) |
| `eventTitle` | S, req | Localized headline, e.g. *"Ready for collection"* |
| `eventDescription` | S, nullable | Longer localized description |
| `location` | `EventLocationNetwork`, nullable | `{ countryCode (S,req), city (S?) }` — e.g. "logistic center" city |
| `eventCode` | S, nullable | Stable machine code for the event type |

Rows are returned newest/oldest per backend ordering; sort by `date` for the
timeline. `eventTitle`/`eventDescription` are already localized server-side
(locale is negotiated at auth), so a client can display them verbatim.

**`eventLog` — `EventLogNetwork[]` (structured/audit log, less UI-facing):**

| Field | Type | Notes |
|-------|------|-------|
| `type` | S, req | Event category |
| `name` | S, req | Event name/code |
| `date` | ZonedDateTime, req | Timestamp |
| `details` | `EventLogDetailsNetwork`, nullable | `{ invoiceRequested (bool) }` |

Both arrays are `@NotNull` in the wire model (may be empty `[]`, never absent). For
rendering the tracking history, use `events`; `eventLog` is a secondary structured feed.
The coarse current state is `status` / `statusGroup` (§5.4); `events[]` is the full
per-state history behind it.

**Localization is fixed at the account/auth locale, not per-request.** Empirically, an
`Accept-Language` request header does **not** change `eventTitle`/`eventDescription` — the
server returns them in the account's language (e.g. Polish) regardless. A client that
needs a specific UI language should map the stable `eventLog[].name` codes (§5.3b-style
client mapping) rather than display `events[].eventTitle` verbatim.

### 5.3b Parcel size (`parcelSize`) → visible size label

`parcelSize` (§5.3) is a nullable size code — usually a single letter `A`–`J`, though
the literal string `OTHER` can also be returned on the wire. Either way it
collapses to UNKNOWN when it isn't a mapped letter (so `OTHER` → no size badge).
Clients can normalize it through two steps.

**Step 1 — wire code → `DeliveryDimension`.** Each dimension carries physical
limits (max W×H×D cm, max weight kg):

| Wire code | `DeliveryDimension` | Max dimensions |
|---|---|---|
| `D` | XS | 4 × 23 × 3 cm, 40 kg |
| `A`, `E`, `H` | SMALL | 8 × 38 × 25 cm, 64 kg |
| `B`, `F`, `I` | MEDIUM | 19 × 38 × 25 cm, 64 kg |
| `C`, `G`, `J` | LARGE | 41 × 38 × 25 cm, 64 kg |
| anything else / `null` | UNKNOWN | — |

(Note the many-to-one collapse: `A`/`E`/`H` all normalize to SMALL, etc.)

**Step 2 — `DeliveryDimension` → visible label.** There are two label alphabets, depending
on the screen:

| `DeliveryDimension` | Locker "gabaryt" badge | Size badge |
|---|---|---|
| XS | `D` | `XS` |
| SMALL | `A` | `S` |
| MEDIUM | `B` | `M` |
| LARGE | `C` | `L` |
| UNKNOWN | — | `OTHER` |

So a visible **`L`** means wire `parcelSize ∈ {C, G, J}` → LARGE → `L`. The `XS/S/M/L`
badge is used for multi-compartment inner shipments and the send/attributes
flow; the `A/B/C/D` "gabaryt" badge is the locker-compartment size. A client
that only wants the coarse size can map the wire code directly:
`D→XS`, `A,E,H→S`, `B,F,I→M`, `C,G,J→L`, else → none.

### 5.3c Sender / merchant name (`sender.name`) → list & history row title

`sender` (§5.3) is an `InternationalCustomerNetwork` whose `name` field carries the
sending merchant or person's display name — e.g. *"Amazon Polska"*, *"SuperSound Sp. z
o.o."*, *"nafta69 Marcin Błogosławski"*. Verified against live `/v4/parcels/tracked`:
`sender.name` was present on **every** parcel in the list response, not just on the
single-parcel detail fetch.

Practical consequence: a list/history row can be titled with `sender.name` (falling back
to the formatted `shipmentNumber` when absent) **without** a per-parcel
`GET /v4/parcels/tracked/{shipmentNumber}` call — the name, `parcelSize` and `eventLog`
are all already in the list payload. `receiver.name` is the same shape for the receiving
party.

### 5.3d Ownership vs sharing, and the "completed on" date

Two independent concepts, easy to conflate:

- **`ownershipStatus`** — your relationship to the parcel. Live wire values are
  **`OWN`** (you own it) and **`FRIEND`** (a friend shared it *with* you). It is **not**
  `OWNER`/`OWNED` — a classifier that only accepts those will flag every `OWN` parcel as
  "shared", which is wrong. Match `FRIEND` (case-insensitively) for "shared with me";
  treat `OWN` as owned.
- **`sharedTo`** — the outbound direction: friends *you* shared this parcel with
  (`FriendNetwork[]`, usually empty). Independent of `ownershipStatus`; an `OWN` parcel can
  still have a non-empty `sharedTo`.

**Completion date.** For a finished parcel prefer **`pickUpDate`** (collection time), then
**`returnedToSenderDate`** for returns. `expiryDate` is often absent on completed parcels
and `storedDate` is missing on courier deliveries, so ordering a history list by
stored/expiry alone leaves many rows undated — `pickUpDate` is present across finished
parcels and is the correct sort/label key.

### 5.4 Parcel `status` values

`status` is a **free-form string** (there is no enum in the wire model). The
full set includes these lowercase values
(e.g. `ready_to_pickup`):

```
created, confirmed,
adopted_at_source_branch, sent_from_source_branch, adopted_at_sorting_center,
sent_from_sorting_center, adopted_at_target_branch,
out_for_delivery, out_for_delivery_to_address,
ready_to_pickup, ready_for_collection,
ready_to_pickup_from_branch, ready_to_pickup_from_pok, ready_to_pickup_from_pok_registered,
pickup_reminder_sent, pickup_reminder_sent_address,
stack_in_box_machine, stack_in_customer_service_point,
stack_parcel_in_box_machine_pickup_time_expired, stack_parcel_pickup_time_expired,
unstack_from_box_machine, unstack_from_customer_service_point,
delivered, collected_by_customer, collected_from_sender, taken_by_courier, taken_by_courier_from_pok,
dispatched_by_sender, dispatched_by_sender_to_pok,
avizo, avizo_completed, avizo_rejected,
cod_completed, cod_rejected,
delay_in_delivery, delivery_attempt_failed,
redirect_to_box, canceled_redirect_to_box, permanently_redirected_to_box_machine,
permanently_redirected_to_customer_service_point, readdressed, readdressed,
return_pickup_confirmation_to_sender, returned_to_sender,
rejected_by_receiver, not_collected, missing, oversized, claimed, cancelled,
pickup_time_expired, undelivered (+ undelivered_* variants:
  undelivered_cod_cash_receiver, undelivered_incomplete_address,
  undelivered_lack_of_access_letterbox, undelivered_no_mailbox,
  undelivered_not_live_address, undelivered_unknown_receiver, undelivered_wrong_address)
```
For "is it waiting for me in the locker", check `status` ∈ {`ready_to_pickup`, `stack_in_box_machine`, …} **and** `operations.collect == true`. `statusGroup` gives the coarse bucket if you just need ready / in-transit / delivered.

**`claimed`** is a terminal *picked-up* state — a Paczkomat parcel becomes `claimed` after it is collected (remotely or in person), so it sorts **at or after** `delivered` / `collected_by_customer` in a history/pipeline, not mid-transit. Treat `claimed`, `collected_by_customer`, and `collected_from_sender` as "picked up".

### 5.5 Add / remove / share

**Add (start tracking):**
```
POST /v4/parcels/observed
{ "shipmentNumber": "...", "parcelType": "TRACKED" }   // parcelType enum: TRACKED | SENT (default TRACKED)
```
→ `TrackedObservedParcelResponse { "trackedParcel": TrackedParcelNetwork }`.

**Remove:** `DELETE /v4/parcels/observed/{shipmentNumber}` → empty.

**Share:**
```
POST /v4/parcels/shared
{ "parcels": [ { "shipmentNumber": "...", "friendUuids": ["...","..."] } ] }
```

---

## 6. The QR code

**Key fact:** the parcel carries **two distinct codes**:
- `openCode` — the **numeric** pickup code (the digits you'd type on the locker keypad), and the value the remote-open flow validates against (§7).
- `qrCode` — an **opaque server-supplied string** that is rendered as a QR bitmap for the **locker's built-in scanner**.

QR display is a dumb render step: take the `qrCode` string verbatim and encode
it as a QR (`QR_CODE`, error-correction **M**, UTF-8) — **no transformation, no
parsing**.

So to "see the QR code" as a consumer: read `TrackedParcelNetwork.qrCode` and render it as a standard QR. It is the credential the Paczkomat scanner reads to open the compartment — an alternative to the remote-open REST sequence below. The two are independent paths to the same outcome (compartment opens):
- **Scan path:** show `qrCode` at the machine → machine opens. No app↔backend calls.
- **Remote-open path:** the REST sequence in §7, keyed on `shipmentNumber` + `openCode`.

---

## 7. Opening a Paczkomat compartment (remote collect)

All endpoints in this flow are authenticated JSON `POST` requests.

**This is a pure server-side "remote open" — there is NO Bluetooth/BLE/NFC
pairing with the locker.** The phone never talks to the locker directly; it
tells InPost's backend to open it. `validate` sends a `geoPoint` (GPS position),
and clients should handle location-related validation failures.

> **Proximity caveat (cross-checked with
> [`alufers/inpost-cli`](https://github.com/alufers/inpost-cli)):** the
> `geoPoint` is **client-supplied data in the request body**, not a
> hardware-attested location. In practice proximity is **not** a strong
> server-side gate. Don't abuse this — opening a locker you're not at is
> pointless and likely against ToS.

### 7.0 Two backends (legacy vs "global")

There are **two implementations** of the identical open→close flow, selected by
a remote-config flag. Both expose the same 7 endpoints, the same five `status`
values, and the same `closed` boolean — only the host prefix, headers, and model
names differ.

| | **Legacy** (default) | **Global backend** |
|---|---|---|
| Path prefix | **bare** `v1/collect/…`, `v2/collect/validate` | **`/remote-opening/v1/collect/…`** |
| Extra headers | none | **`xCountryCode`** (required), `xAuthUserId`, `xCorrelationID` |
| Models | `CompartmentValidationRequest`, `CompartmentStatusResponse`, … (§7.3) | country-aware models; every call carries `deliveryPointCountryCode` |
| Scope | single-market (PL) | cross-market / international |

The selection flag is `isParcelLockerGlobalBackendEnabled`; unfetched config or
any error falls back to legacy. The legacy `/v1|v2/collect/…` backend is the
practical default target; the global backend is a server-gated migration path
enabled per user/market.

> **Third-party clients:** target the **legacy `/v1|v2/collect/…`** endpoints
> first. The two are wire-compatible on the status/close semantics documented
> below.

### 7.1 Endpoints

| Method | Path | Body → Response |
|---|---|---|
| `POST` | `/v2/collect/validate` | `CompartmentValidationRequest` → `CompartmentValidateResponse` |
| `POST` | `/v1/collect/compartment/open` | `CompartmentSessionIdRequest` → `CompartmentOpenResponse` |
| `POST` | `/v1/collect/compartment/reopen` | `CompartmentSessionIdRequest` → `CompartmentOpenResponse` |
| `POST` | `/v1/collect/compartment/status` | `CompartmentStatusRequest` → `CompartmentStatusResponse` |
| `POST` | `/v1/collect/compartment/closed` | `CompartmentSessionIdRequest` → `CompartmentCloseResponse` |
| `POST` | `/v1/collect/compartment/claim` | `CompartmentClaimRequest` → `CompartmentClaimResponse` |
| `POST` | `/v1/collect/terminate` | `CompartmentSessionIdRequest` → (empty) |

### 7.2 Flow sequence

```
[GPS fix] ── lat/lon/accuracy
   │
   ▼
1. POST /v2/collect/validate
     { "parcel": { "shipmentNumber": "...", "openCode": "...",
                   "receiverPhoneNumber": {prefix,value}? },
       "geoPoint": { "latitude": 52.2, "longitude": 21.0, "accuracy": 12.0 } }
   → { "sessionUuid": "...", "sessionExpirationTime": 1719_... }   // ← sessionUuid drives everything after
   (server may reject: InvalidLocationException if not at the locker)
   │
   ▼
2. POST /v1/collect/compartment/open      { "sessionUuid": "..." }
   → { "compartment": { "name": "...", "location": { "side","column","row" } },
       "openCompartmentWaitingTime": ms, "actionTime": ms, "confirmActionTime": ms }
   // 'compartment' tells the user WHICH door opened. (reopen = same shapes.)
   │
   ▼
3. POST /v1/collect/compartment/status    { "sessionUuid": "...", "expectedStatus": "OPENED" }
   → { "compartment": {...}, "status": "..." }
   // poll until OPENED. (expectedStatus enum: OPENED | CLOSED)
   │
   ▼  (user takes parcel, shuts door)
4. POST /v1/collect/compartment/status    { "sessionUuid": "...", "expectedStatus": "CLOSED" }
   → { ..., "status": "..." }
   │
   ▼
5. POST /v1/collect/compartment/closed    { "sessionUuid": "..." }
   → { "closed": true }                    // confirm door physically shut
   │
   ▼
6. POST /v1/collect/compartment/claim      { "sessionUuid": "...", "shipmentNumbers": ["...","..."] }
   → { "openCompartmentWaitingTime", "actionTime", "confirmActionTime" }
   // confirms which parcels were taken (matters for multi-compartment)

  (any time) POST /v1/collect/terminate    { "sessionUuid": "..." }   // cancel / on session expiry
```

### 7.3 Compartment models

- **`CompartmentValidationRequest`**: `parcel`(`ParcelCompartmentNetwork`, req) + `geoPoint`(`GeoPointNetwork`, req).
  - `ParcelCompartmentNetwork`: `shipmentNumber`(S,req), `openCode`(S,req), `receiverPhoneNumber`(`InternationalPhoneNumberNetwork`?).
  - `GeoPointNetwork`: `latitude`(double), `longitude`(double), `accuracy`(float).
- **`CompartmentValidateResponse`**: `sessionUuid`(S,req), `sessionExpirationTime`(long).
- **`CompartmentSessionIdRequest`**: `sessionUuid`(S,req).
- **`CompartmentOpenResponse`**: `compartment`(`CompartmentNetwork`?), `openCompartmentWaitingTime`(long), `actionTime`(long), `confirmActionTime`(long).
  - `CompartmentNetwork`: `name`(S,req), `location`(`CompartmentLocationNetwork{ side?, column?, row? }`?).
- **`CompartmentStatusRequest`**: `sessionUuid`(S,req), `expectedStatus`(`OPENED`|`CLOSED`, nullable).
- **`CompartmentStatusResponse`**: `compartment`(`CompartmentNetwork`?), `status`(S,req).
- **`CompartmentCloseResponse`**: `closed`(boolean).
- **`CompartmentClaimRequest`**: `sessionUuid`(S,req), `shipmentNumbers`(S[]?).
- **`CompartmentClaimResponse`**: `openCompartmentWaitingTime`(long), `actionTime`(long), `confirmActionTime`(long).

> The timing fields (`actionTime`, `confirmActionTime`,
> `openCompartmentWaitingTime`) are millisecond windows for countdown timers;
> they are not required for a minimal client.

There are parallel `…/return/…` and `…/send/…` compartment flows (returns and sending) with the same shape — see appendix.

### 7.4 Door opened / closed statuses & error states

**There is no numeric "door result" code and no hardware channel.** The whole open→close lifecycle is inferred by **polling `POST /v1/collect/compartment/status`** and reading one free-form string field, `status`, plus a final boolean from `/closed`. The phone never talks to the locker; it asks the backend "is the door open / closed yet?" on a loop.

**The `status` string is one of exactly five values** (`CompartmentStatusResponse.status`, mapped 1:1 into the domain enum `gk0.h` via `valueOf`, so the wire strings *must* be these — an unknown value throws):

| `status` | Meaning |
|---|---|
| `SCHEDULED` | Backend accepted the open command, door **not open yet** (command in flight to the locker). |
| `OPENED` | Door is physically **open**. |
| `FAILED` | Locker **failed to open** the compartment. |
| `CLOSED` | Door is physically **shut**. |
| `TIMEOUT` | Locker did not act within its window (server-side timeout). |

The same five values are used by both the legacy REST backend and the newer "global backend" gateway (`BoxMachineCompartmentStatusResponse`), so a client can treat them identically.

**Two polling phases, each with its own terminal condition** (`CompartmentOpeningStatusUseCase` / `CompartmentClosingStatusUseCase`):

1. **Wait-for-open.** Re-`POST …/status {expectedStatus:"OPENED"}` while
   `status == SCHEDULED` and the local `openCompartmentWaitingTime` deadline
   hasn't passed. Stop when `status` leaves `SCHEDULED`, then branch:
   - `OPENED` **or** `CLOSED` → **success**: UI shows the compartment (side/column/row), phone **vibrates**, and it rolls straight into wait-for-close. (`CLOSED` this early just means the user was very fast.)
   - `FAILED` → give up this compartment (terminate session → error screen).
   - `SCHEDULED` still set at the deadline, or `TIMEOUT` → **"opening timeout / try again"** error state.
2. **Wait-for-close.** Re-`POST …/status {expectedStatus:"CLOSED"}` and
   resolve when `status` is `CLOSED`, `FAILED`, or `TIMEOUT` (or the local
   `actionTime` deadline passes). On resolution, send the **final
   confirmation**:

   `POST /v1/collect/compartment/closed {sessionUuid}` → `{ "closed": true | false }`
   - `closed: true` → door confirmed shut → proceed to `claim` / finish (this is the definitive "success").
   - `closed: false` → **"compartment not closed"** error state (`c.b.d`) — the user is told to shut the door.

So: **"succeeded"** = wait-for-open reached `OPENED`/`CLOSED` **and** `/closed` returned `closed:true`. Everything else is a failure/timeout branch.

**Error / terminal states** a client can surface — these are client-side states,
not HTTP codes:

| State | Trigger |
|---|---|
| opening-timeout / "try again" (`c.b.g`) | `status` stuck at `SCHEDULED`, or `TIMEOUT`, or any local countdown (`openCompartmentWaitingTime` / `confirmActionTime`) elapsed. |
| compartment-not-closed (`c.b.d`) | `/closed` returned `closed:false`. |
| failed-to-open | `status == FAILED` during wait-for-open. |
| location-not-confirmed (`c.b.LocationNotConfirmed`) | `validate` raised `InvalidLocationException` (client-side proximity guard, see §7 caveat). |
| session/endpoint timeout (`c.b.SessionEndpointTimeout`) | `SocketTimeoutException` on a status/close call. |
| general / final error (`c.b.General` / `c.b.Final`) | Any other throwable; `Final` = no further compartment to retry. |

On any terminal error, call `POST /v1/collect/terminate {sessionUuid}` (or
`/compartment/claim`) to release the session. Enforce `sessionExpirationTime`
client-side and tear the session down when it lapses.

> **Minimal-client takeaway:** after `open`, poll `…/status` until `status ∈ {OPENED, FAILED, TIMEOUT}`; if `OPENED`, poll again until `status == CLOSED`, then call `…/closed` and trust its `closed` boolean as the success flag. There are no other signals.

---

## 8. Paczkomaty / pickup points (map)

Authenticated.

| Method | Path | Params |
|---|---|---|
| `GET` | `/v4/points/{boxMachineName}` | Path: the point name (e.g. `KRA01M`) → `PointV4Network` |
| `GET` | `/v4/points` | `relativePoint=<lat,lng>`, `maxDistance=<meters>?`, `filter=<csv>`, `country=<code>?` → `PointsV4Network` |
| `GET` | `/v4/points` | `query=<text search>`, `relativePostCode=?`, `filter=<csv>`, `country=?`, `perPage=<int, default 100>` → `PointsV4Network` |

`filter` is a **comma-separated list of capability tokens**. Known tokens
include `parcel_send`; others correspond to collect/return/cod/air-quality
capabilities. Pass an empty/broad filter to get all types.

**`PointsV4Network`**: `{ "points": [ PointV4Network, ... ] }`.

**`PointV4Network`** (key fields): `name`(S), `status`(S), `location`(`{latitude,longitude}`), `locationDescription`(S), `openingHours`(S), `openingHoursDetails`(`OpeningHoursDetailsNetwork[]`), `addressDetails`(`AddressDetailsNetwork`), `country`(S), `paymentType`(Map<int,String>), **`pointType`(S, req)** (e.g. parcel locker / POP / POK — free-form string), `virtual`(int), `location247`(bool), `doubled`(bool), `imageUrl`(S), `airSensor`(bool)/`airSensorData`, `remoteSend`(bool), `remoteReturn`(bool), `lockerAvailability`(`{status, details:{sizeA,sizeB,sizeC,sizeD}}`), `distance`(int), `printInStore`(B).

`AddressDetailsNetwork`: `city`, `country`(JSON key `country`, Kotlin `countryCode`), `province`, `postCode`, `street`, `buildingNumber`, `flatNumber`, `description` — all String?.
`OpeningHoursDetailsNetwork`: `day`(S,req), `opening`(`[{start:int,end:int}]`).

`BoxMachineNetwork` (the same point shape embedded as `pickUpPoint` on a parcel): `name`(req), `location`, `locationDescription`, `openingHours`(S, e.g. `"24/7"`), `addressDetails`, `paymentType`, `pointType`(S), `type`(S[], e.g. `["parcel_locker"]`), `virtual`(int), `location247`(bool), `easyAccessZone`(bool), `country`(S, e.g. `"PL"`), `doubled`, `imageUrl`, `airSensor`/`airSensorData`, `remoteSend`, `remoteReturn`, `distance`.

> **Verified on the wire (2026-07):** the embedded `pickUpPoint` on `/v4/parcels/tracked` really does carry the richer point extras — `openingHours` (flat string, e.g. `"24/7"`), `type` (array), `virtual`, `easyAccessZone`, and a top-level `country` — not just name/address. So a client can show locker opening hours and a locker photo (`imageUrl`) straight from the parcel payload, without a separate points/map lookup. Note the flat `openingHours` string here is distinct from the structured `openingHoursDetails` (`OpeningHoursDetailsNetwork[]`) returned by the points endpoint.

---

## 9. Pricing (parcel send prices — optional but parcels-adjacent)

`GET /v2/prices/parcels?sourceCountryCode=PL` → `PriceResponse`.
`GET /v3/prices/parcels?sourceCountryCode=PL` → `List<CountryPriceNetwork>`
(`{ sourceCountryCode, countries[], expandAvizo }`, with per-country
`boxMachineService/courierService/insurance` and
`DimensionsNetwork{height,width,length,dimensionsUnit}`,
`AdditionalServicesNetwork{oneDayDelivery,weekendDelivery}`).

---

## 10. Notification centre (drives the "new event" badges — optional)

- `GET /v3/notifications?type=<types>` → `NotificationV3Response`
- `POST /v1/notifications/read` body `MarkMultipleAsReadRequest`
- `POST /v1/notifications/readAll` body `MarkAllAsReadRequest`

(Worker also references `v1/notifications` for download.)

---

## 11. Practical recipe — "list my inbound parcels, show status + QR, open the locker"

1. **Authenticate** (§4). Easiest to reason about: legacy SMS flow → `authToken`. Or OAuth PKCE → `access_token`. Keep the refresh token.
2. **Attach headers** on every call (§3), especially `Authorization: Bearer <token>`, `User-Agent`, `Device-ID`, `device-uid`, `Session-ID`.
3. **List inbound parcels:** `GET /global/v4/parcels/tracked`. Iterate `parcels[]`.
4. **Per parcel:** read `status` / `statusGroup` for shipping state; `expiryDate`, `storedDate`; `pickUpPoint` for which Paczkomat; `openCode` (numeric) and `qrCode` (render as QR).
5. **Is it collectable now?** `operations.collect == true` (typically when `status` indicates it's stored/ready in a locker). Do **not** also require `mobileCollectPossible` — that is a legacy v1 field and is not present in v4 `/parcels/tracked` responses.
6. **To open remotely:** get a GPS fix at the locker → `POST /v2/collect/validate` with `{shipmentNumber, openCode}` + `geoPoint` → take `sessionUuid` → `POST /v1/collect/compartment/open` → poll `…/status` (`OPENED`) → user shuts door → `…/status` (`CLOSED`) → `…/closed` → `…/claim`. Or simply **display the `qrCode`** and scan it at the machine.

---

## 11b. Implementation notes for client builders

Everything below is what you actually need to get a read-only "my parcels +
status + QR + remote open" client working.

### Recommended auth path for a mobile/macOS client
Use the **legacy SMS flow** (§4.3), not OAuth. The OAuth flow (§4.2) is
web-login based and less convenient for native/headless clients. The SMS flow is
three plain JSON calls (`/v1/account` → `/v1/account/verification` → tokens) and
is trivial to implement on macOS/iOS. Persist the `refreshToken` and re-mint
access tokens via `POST /v1/authenticate`.

- `phoneOS` / `devicePlatform` field value: send **`"Android"`** for the legacy
  SMS flow. The backend rejects the lowercase value on the current production
  path tested by custom clients.
- **Phone number prefix includes the `+` for the legacy SMS flow**:
  `{ "prefix": "+48", "value": "600123456" }`. Phone is still a structured
  object, never a flat string.

### Minimum required headers
**Confirmed empirically via `alufers/inpost-cli`:** the server needs only:
- `Authorization: Bearer <token>` (on authed calls)
- `Content-Type: application/json` (on requests with a body) and `Accept: application/json`

[`alufers/inpost-cli`](https://github.com/alufers/inpost-cli) sends **none** of
the common device headers (`X-App-Id`, `Device-ID`, `device-uid`, `Session-ID`,
`Accept-Language`) and uses a generic `User-Agent`; the documented flows still
work. So those six headers are **not server-enforced**; treat them as optional
telemetry / anti-abuse signals, not requirements.

Caveat: `inpost-cli` exercises the **older v1** endpoints. The current
**v4/v5** endpoints could in principle be stricter. Pragmatic stance: send
`Authorization` + `Content-Type`/`Accept` to start; if a v4 call misbehaves,
add a plausible `User-Agent` and a persisted `device-uid`/`Device-ID` before
sending all seven.

### Wire formats
- **Dates**: ISO-8601 **offset** date-time, `ISO_OFFSET_DATE_TIME` /
  `yyyy-MM-dd'T'HH:mm:ss.SSSXXX`, e.g.
  `2024-06-30T14:30:00.000+02:00` (millis optional on parse). No bare
  dates/instants in the parcel/compartment models.
- **JSON**: standard; watch the field-name vs JSON-key mismatches called out in §5.3 (`cashOnDelivery`→`cod`, `isEconomy`→`economyParcel`, etc.).

### HTTP client config
- Timeouts around **14 s and 40 s** for connect/read/write are reasonable
  starting values. Keep connection retry enabled.
- Refresh reactively on HTTP `401`. Simplest correct client: call normally; on
  `401`, refresh and retry once. You can also pre-refresh using the JWT `exp`
  claim if you decode the token.

### Listing parcels & keeping in sync
- `GET /v4/parcels/tracked` takes **no query params** and returns the **full** tracked set. There is no `updatedAfter`/delta param.
- The response is **server-stateful paginated**: if `more == true`, call the same endpoint again (no cursor) and the server advances; loop until `more == false`, accumulating `parcels[]`.
- `removedParcelList[]` = shipment numbers to delete from your local store.
- `updatedUntil` is returned but is not enough for delta sync; do not rely on it
  alone.
- For a simple client, just re-fetch the whole list on a timer / pull-to-refresh.

### Remote-open timing & polling
- Get a GPS fix first. Accuracy around **≤ 30 m** and a fix **no older than
  10 s**, with a **20 s** fix timeout, are practical defaults. Coarse fallback
  can relax accuracy to 10 km. Remember (§7.2) `geoPoint` is client-supplied,
  so these are *your client's* checks, not a hard server gate.
- After `open`, the millisecond fields in the responses are **durations**, not timestamps — add them to "now":
  - `openCompartmentWaitingTime` → how long to wait for the door to physically open.
  - `actionTime` → window for the user to act once open.
  - `confirmActionTime` → final window to confirm the door was closed.
- `compartment/status` polling: a 1–2 s poll interval until you see `status`
  flip (toward `OPENED`, then `CLOSED`) or the window elapses is reasonable.

### Error handling
Error responses deserialize to **`ErrorResponse`**:
```jsonc
{
  "error": "sessionExpired",   // REQUIRED — the machine-readable code (see below)
  "description": "…",          // nullable human text
  "status": 409,               // nullable, echoes HTTP status
  "message": "…",              // nullable
  "details": {                 // nullable
    "appKeys": [...],          // BLIK app keys (payments)
    "deliveryPointName": "…",  // e.g. which locker
    "redirectUrl": "…"
  },
  "diagnosticId": "…"          // nullable — quote this if contacting InPost
}
```
The `error` value is a **camelCase** code (the enum names are SNAKE_CASE internally, but the wire value is camelCase). Codes relevant to you:

- **Locker / remote-open** (`RemoteCompartmentOpenApiErrorType`): `invalidSession`, `sessionExpired`, `invalidSessionState`, `sessionAlreadyExists`, `cannotFindCompartment`, `invalidCompartmentState`, `invalidParcelState`, `boxMachineNotFound`, `invalidBoxMachineConfiguration`, `sendNotSupported`, `returnNotSupported`, `unauthorizedToSend`, `unauthorizedToReturn`, `invalidReturnTicketState`. (A geofence rejection on `validate` surfaces here as an invalid-state/location error.)
- **Common** (`CommonApiErrorType`): `serverError`, `serviceCutOff`, `internalCommunicationError`.
- **Payments** (`PaymentApiErrorType`, only if you touch COD/payments): `parcelNotFound`, `parcelAlreadyPaid`, `paymentMethodNotAvailable`, `blikAliasNotFound`, `discountCodeExpired`, … (14 total).
- **Returns** (`FastReturnsApiErrorType`): `returnOrganizationOrReasonNotFound`, `returnTicketNotFound`.

Treat `401` specially (refresh + retry). For locker flows, `sessionExpired`/`invalidSessionState` mean start over from `validate`.

---

## 12. Caveats & unknowns

- Exact HTTP statuses and optional edge-case fields may vary by backend release.
- Error envelope: errors deserialize to `ErrorResponse` / `ApiErrorDetails`
  with typed error enums (`CommonApiErrorType`,
  `RemoteCompartmentOpenApiErrorType`, `PaymentApiErrorType`,
  `FastReturnsApiErrorType`). HTTP status semantics: `401` triggers token
  refresh; geofence failures surface as a remote-compartment error during
  `validate`.
- Field nullability reflects observed behavior; the server may be
  stricter/looser. **Validate against live responses before relying on any
  field.**
- Legal/ToS: this is a private API. Automated/headless use may violate InPost's terms and the geofenced open flow is deliberately gated to physical presence.

---

## Appendix A — Full endpoint inventory

Base `…/global`. Auth required unless the endpoint is part of the login
bootstrap.

| Area | Endpoints |
|---|---|
| **Parcels (inbound)** | `GET v4/parcels/tracked`, `GET v4/parcels/tracked/{shipmentNumber}`, `GET v4/parcels/multi/{multiCompartmentUuid}`, `POST v4/parcels/observed`, `DELETE v4/parcels/observed/{shipmentNumber}`, `POST v4/parcels/properties`, `POST v4/parcels/shared` |
| Parcels (sent) | `GET v4/parcels/sent`, `GET v4/parcels/sent/{shipmentNumber}`, `DELETE v4/parcels/observed/{shipmentNumber}` |
| Parcels (sent, new) | `POST v5/parcels`, `POST v4/parcels`, `GET v4/parcels/sent/{shipmentNumber}` |
| **Compartment collect** | `POST v2/collect/validate`, `POST v1/collect/compartment/open`, `/reopen`, `/closed`, `/claim`, `POST v1/collect/terminate` |
| Compartment status | `POST v1/collect/compartment/status` |
| **Points / map** | `GET v4/points`, `GET v4/points/{boxMachineName}` |
| **Auth — SMS** | `POST /v1/account`, `POST /v1/account/verification`, `POST v1/authenticate` |
| Auth — SMS refresh | `GET v1/sendSMSCode`, `POST v1/reconfirmSMSCode` |
| Auth — logout | `POST v1/logout`, `POST v1/logout-all` |
| Auth — OAuth | `POST global/oauth2/token` (auth_code / refresh_token / token-exchange grants), `GET global/oauth2/logout`, `global/oauth2/logout-all` |
| Account | `DELETE v1/account` |
| Prices | `GET v2/prices/parcels`, `GET v3/prices/parcels` |
| Notifications | `GET v3/notifications`, `POST v1/notifications/read`, `/readAll` |
| Preferences | `GET v1/preferences`, `/profile`, `GET/POST v1/preferences/notifications` |
| Push registration | `POST v2/setPushId` |
| Friends/sharing | `GET/POST v2/friends`, `GET v2/friends/{shipmentNumber}`, `DELETE v2/friends/{uuid}`, `POST v2/invitations/validate`, `/accept` |
| Send (remote locker) | `POST v1/send/validate`, `/confirm`, `/cancel`, `v1/send/compartment/open`, `/reopen`, `/closed`, `/status` |
| Return (remote locker) | `POST v1/return/validate`, `/confirm`, `/cancel`, `v1/return/compartment/open`, `/reopen`, `/closed`, `/status` |
| Returns/tickets | `GET v1/returns/parcels`, `v1/returns/organizations`, `v2/returns/organizations`, `v1/returns/forms/{uuid}`, `v1/returns/reasons/{uuid}`, `POST v1/returns/tickets`, `v1/returns/tickets/paid`, `v1/returns/parcels/tickets`, `GET v1/returns/tickets`, `/{uuid}` |
| Avizo | `POST v1/orders/expand-avizo`, `v2/orders/expand-avizo` |
| Payments | `POST v1/payments/transactions/create`, `/cancel`, `/create/blik`, `GET v1/payments/blik/alias/status` |
| COD / discounts / donations | `POST v1/orders/cash-on-delivery`, `v1/orders/discounts/validate`, `v1/donations` |
| Voicebot / chatbot | `POST v1/voicebot/authenticate`, `GET global/api/v1/chatbot/token`, `POST global/api/v1/chat-token` |
| Company / docs | `GET /v1/companies/{taxNumber}`, `v1/documents/policy`, `izi/app/shopping/documents/v1/policy` |
| Logging | `POST v1/logging` |
| **Loyalty / InCoins** | `loyalty/v1/wallet/balance`, `loyalty/v1/customer/quest/value` (InCoins), `loyalty/v3/customer-rewards`, `loyalty/v1/csr/*`, `loyalty/v1/customer/enrollment`, `loyalty/v1/virtualbox/*` … |
| **Shopping ("izi" / VonHalsky)** | `izi/app/shopping/v*/baskets…`, `…/checkout`, `…/orders`, `…/profile…`, `…/payments…`, `…/merchants…`, `…/products/hot-products`, `…/subscriptions…`, `…/coupons/recommended` … |
| **Nimbus (AI shopping)** | `api/v1/recommendations/*`, `api/v1/chats…`, `api/v1/product-catalog/*`, `api/v1/carts/current…`, `api/v1/personalization/favourites/*`, `api/v1/onboarding…`, `api/v2/conversations…` |
| Dotpay | `GET payment_api/channels/?...` |

(The shopping/loyalty/nimbus areas are large and out of scope; listed for completeness. Their base host is the same `…/global` mobile API, with `izi/app/shopping/...` path prefixes.)

---

## Appendix B — Cross-check against `alufers/inpost-cli`

The open-source Go client
[`alufers/inpost-cli`](https://github.com/alufers/inpost-cli) (and its
`inpost.swagger.yml`) is a useful public comparison point. It corroborates the
fundamentals below and helps explain older-vs-newer endpoint shapes.

**Confirmed independently** (high confidence these are correct):
- Host `api-inmobile-pl.easypack24.net`; auth `Authorization: Bearer <token>`.
- SMS login issues `authToken` + `refreshToken`; refresh via `POST /v1/authenticate` with `AuthenticateRequest { phoneOS, refreshToken }` → `authToken`. (Exact match.)
- The parcel object carries **both** `qrCode` and `openCode` as separate fields, plus `status`, `storedDate`, pickup date. (Confirms §6's two-code model.)
- Collect = a session state machine: `collect/validate` (body includes `geoPoint` + `parcel`) → `sessionUuid` → `compartment/open` → `compartment/claim` → `compartment/status`. (Confirms §7.)
- `POST /v2/setPushId` for push registration. (Exact match.)

**Divergences = API evolution:**

| Concern | `inpost-cli` (older) | This document |
|---|---|---|
| Base path | host root | host + **`/global`** |
| Send SMS | `POST /v1/sendSMSCode/{phoneNumber}` | `POST /v1/account` (phone = `{prefix,value}` body) |
| Confirm SMS | `POST /v1/confirmSMSCode/{phoneNumber}/{smsCode}` | `POST /v1/account/verification` (body) |
| List parcels | `GET /v1/parcel` | `GET /v4/parcels/tracked` |
| Parcel detail | `GET /v1/parcel/{shipmentNumber}` | `GET /v4/parcels/tracked/{shipmentNumber}` |
| Event history | `statusHistory` field | `events[]` + `eventLog[]` |
| Collect session id | **URL path param**, status via `GET` | **request body** (`CompartmentSessionIdRequest`), status via `POST` |
| Validate version | `POST /v1/collect/validate` | `POST /v2/collect/validate` |
| Points | `GET /v1/points` | `GET /v4/points` |

**Takeaway:** when targeting the **current** backend, use the v4/v5 paths and
the `/global` base from this document; the `inpost-cli` swagger is a useful
conceptual reference for older v1 shapes, but several of its exact paths are
stale.
