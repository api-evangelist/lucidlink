---
name: lucidlink-onboard-team-member
description: >-
  Onboard or offboard a person in a LucidLink workspace through the self-hosted Management
  API — member, group membership, and folder permission — and know when to use SCIM instead.
api: LucidLink Management API
base_url: http://{api-container-host}:3003/api/v1
operations:
  - POST /api/v1/members
  - GET /api/v1/members
  - GET /api/v1/members/{memberId}
  - PATCH /api/v1/members/{memberId}
  - DELETE /api/v1/members/{memberId}
  - GET /api/v1/members/{memberId}/groups
  - POST /api/v1/groups
  - GET /api/v1/groups
  - PUT /api/v1/groups/members
  - PUT /api/v1/groups/{groupId}/members/{memberId}
  - DELETE /api/v1/groups/{groupId}/members/{memberId}
  - POST /api/v1/filespaces/{filespaceId}/permissions
  - GET /api/v1/filespaces/{filespaceId}/permissions
  - DELETE /api/v1/filespaces/{filespaceId}/permissions/{permissionId}
generated: '2026-08-25'
method: generated
source: >-
  https://support.lucidlink.com/hc/en-us/articles/40235274774797-API-Key-Functionalities-Common-Automation-Scenarios
  + conventions/lucidlink-conventions.yml
---

# Onboard and offboard a workspace member

This runs against the **self-hosted** Management API, not the public Service API. Paths are
quoted from LucidLink's published endpoint reference; there is no fetchable OpenAPI for this
API, because the container generates its own Swagger at `/api/v1/docs` on your host.

## Prerequisites

- The container is running: `docker run -p 3003:3003 lucidlink/lucidlink-api:latest`
  (pin a real tag rather than `:latest` — LucidLink's own best-practices article says so).
- A service-account secret key. Business or Enterprise plan, minted by a workspace admin.
- The container speaks plain HTTP on 3003. **Put it behind a TLS-terminating reverse proxy
  before anything outside the host talks to it** — the bearer token is your workspace's
  admin credential and this is LucidLink's own stated requirement.

```
Authorization: Bearer <service key>
Content-Type: application/json
```

## Check first: should this be SCIM instead?

If the workspace has SSO and SCIM enabled, the identity provider owns members and groups.
A SCIM-managed user **cannot** be deleted from LucidLink, and a SCIM-managed group cannot be
renamed, deleted, or have members added or removed through this API. Doing lifecycle work by
API on a SCIM workspace will fail or drift. See
https://support.lucidlink.com/hc/en-us/articles/38861860730637-Understanding-SCIM-Integration-in-LucidLink

## 1. Add the member

```
POST /api/v1/members
{"email": "person@example.com"}
```

The response carries the new `memberId` — which is a **principalId**. Members and groups
share one identifier space, which is why permissions take a single `principalId`.

## 2. Put them in a group

Single:

```
PUT /api/v1/groups/{groupId}/members/{memberId}
```

Bulk (the documented onboarding path for many people at once):

```
PUT /api/v1/groups/members
{"memberships": [{"groupId": "<GROUP_ID>", "memberId": "<MEMBER_ID>"}]}
```

Prefer the bulk form. There is no published rate limit, but LucidLink's guidance is
explicitly to batch rather than to fire sequential calls.

## 3. Grant filespace access

```
POST /api/v1/filespaces/{filespaceId}/permissions
{"path": "/", "permissions": ["read"], "principalId": "<GROUP_ID>"}
```

Grant to the **group**, not the person. That is what makes step 5 a one-liner.

## 4. Audit

```
GET /api/v1/filespaces/{filespaceId}/permissions
GET /api/v1/groups/{groupId}/members
GET /api/v1/members/{memberId}/groups
```

## 5. Offboard

Reversibility matters here and the two paths are genuinely different:

- `DELETE /api/v1/members/{memberId}` revokes access immediately. It is **not** reversible —
  re-adding the person is a fresh `POST /api/v1/members` and does **not** restore their prior
  group memberships or permissions.
- Under SCIM, **deactivate in the IdP instead**. A deactivated user keeps every permission and
  configuration, stops counting toward billing, and regains full access intact on
  reactivation. That is the only true undo in this flow, and it does not live in this API.

Removing a single permission (`DELETE .../permissions/{permissionId}`) *is* cleanly
reversible — re-POST the same grant. Permission changes are documented as taking effect in
real time in both directions.

## Retry safety

No idempotency key exists on this API. A retried `POST /api/v1/members` after a timeout may
create a duplicate. Call `GET /api/v1/members` and check before retrying.
