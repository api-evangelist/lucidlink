---
name: lucidlink-provision-project-filespace
description: >-
  Provision a new LucidLink filespace for a project on the public Service API, verify it,
  and know exactly what cannot be undone before you start.
api: LucidLink Service API
base_url: https://api.lucidlink.com/api/v1
operations:
  - createFilespace
  - getFilespaces
  - getFilespace
  - deleteFilespace
generated: '2026-08-25'
method: generated
source: openapi/lucidlink-service-api.json + conventions/lucidlink-conventions.yml
---

# Provision a project filespace

Every operationId below was read from `openapi/lucidlink-service-api.json`, the Swagger 2.0
document LucidLink serves at https://api.lucidlink.com/docs/api/v1/.

## Before you start

- **Credentials are not self-service.** OAuth2 client credentials are issued on request via
  support+ticket@lucidlink.com or https://support.lucidlink.com/hc/en-us. There is no
  console that mints them.
- **There is no test mode.** No sandbox, no `sa_test:` prefix. Anything you create here is
  real and billable — see `plans/lucidlink-plans-pricing.yml`.
- **There is no idempotency key.** If `createFilespace` times out, do **not** blind-retry;
  call `getFilespaces` first and check whether the filespace already exists.

## 1. Get an access token

```
POST https://auth.lucidlink.com/oauth2/token
Authorization: Basic base64(CLIENT_ID:CLIENT_SECRET)
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
```

Read `access_token` from the response.

## 2. Send the token on every call

LucidLink's own published example sends the raw token with **no `Bearer` prefix** on this
v1 API:

```
Authorization: <access_token>
Content-Type: application/json
```

(The self-hosted Management API and the v2 web service *do* use `Bearer`. They are different
APIs. See `conventions/lucidlink-conventions.yml`.)

## 3. Check what already exists — `getFilespaces`

```
GET /api/v1/filespaces
```

Returns an unbounded array. There is no `limit`, cursor or page parameter on this API, so
handle a large response in one read.

## 4. Create it — `createFilespace`

```
POST /api/v1/filespaces
```

Declared responses: `201` created, `400` bad request, `409` conflict, `422` unprocessable.

- `409` is your duplicate-name guard, and it is the only retry protection this API gives you.
- No `401` or `403` is declared even though every operation is OAuth2-secured and the live
  API returns `401` on an unauthenticated call. Handle 401 anyway.
- No `5xx` is declared on any operation. Handle 5xx anyway.

## 5. Confirm — `getFilespace`

```
GET /api/v1/filespaces/{id}
```

`400` on a malformed id, `404` if it is not there.

## Errors

The envelope on this API is `{"status": <int>, "message": <string>}`. There are no error
codes — the only discriminator is the HTTP status plus free text. Do not pattern-match the
message string. See `errors/lucidlink-problem-types.yml`.

## Undo

`deleteFilespace` (`DELETE /api/v1/filespaces/{id}`) is **irreversible through the API**.
There is no restore, undelete or trash operation. The only recovery path is a filespace
snapshot taken *beforehand* and restored from the LucidLink client — and snapshots do not
exist on the Starter plan, are 30 days on Business, and unlimited on Enterprise. `409` will
stop you deleting a filespace that is still in use, but nothing will bring one back.
