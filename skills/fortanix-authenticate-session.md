---
name: Authenticate a Fortanix DSM session
description: Obtain, scope, refresh and terminate a Fortanix Data Security Manager bearer session so subsequent key and crypto calls are authorized.
api: openapi/fortanix-dsm-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-dsm-openapi-original.json
operations:
  - Authenticate
  - SelectAccount
  - Refresh
  - Reauthenticate
  - Terminate
  - AuthDiscover
  - Version
---

# Authenticate a Fortanix DSM session

Every other Fortanix DSM skill assumes this one has run. DSM is session-oriented: you
exchange a long-lived credential for a short-lived bearer token, scope it to an account,
and hand that token to subsequent calls.

## Base URL

DSM SaaS is regional. The OpenAPI models the host as a `{dsmEndpoint}` server variable
whose default is `https://amer.smartkey.io`. Use the endpoint for the customer's region;
never hard-code one.

## Steps

1. **(Optional) Discover the auth method.** `POST /sys/v1/session/auth/discover`
   (`AuthDiscover`) tells you which authentication methods the account accepts before
   you attempt one.
2. **Authenticate.** `POST /sys/v1/session/auth` (`Authenticate`) with HTTP Basic.
   - As an **application**: the Basic credential is `<app_uuid>:<api_key>`.
   - As a **user**: the Basic credential is `<email>:<password>`.
   The response carries the bearer token to use as `Authorization: Bearer <token>`.
3. **Select an account.** A user token is not yet scoped. `POST /sys/v1/session/select_account`
   (`SelectAccount`) binds the session to one `acct_id`. Application tokens are already
   account-scoped and skip this step.
4. **Keep it alive.** `POST /sys/v1/session/refresh` (`Refresh`) extends the session.
   Refresh on a timer; do not wait for a rejection.
5. **Re-elevate when required.** Sensitive operations may require a fresh proof of
   identity — `POST /sys/v1/session/reauth` (`Reauthenticate`).
6. **Terminate.** `POST /sys/v1/session/terminate` (`Terminate`) when the work is done.
   Do not leak sessions.

## Alternative: no session at all

DSM also accepts the app API key directly on each request via the `Authorization`
header (`apiKeyAuth`, documented as a token prefixed with `Basic `). This avoids
session management for simple, low-volume automation but gives up the ability to
revoke a single session.

## Multi-factor

If the account enforces 2FA, `Authenticate` returns a challenge and you complete it
with `POST /sys/v1/session/auth/2fa/u2f` (`U2fAuth`) or
`POST /sys/v1/session/auth/2fa/recovery_code` (`RecoveryCodeAuth`).

## Health check

`GET /sys/v1/version` (`Version`) is unauthenticated and confirms the endpoint is
reachable and which DSM build is serving it. Use it to validate the endpoint before
spending a credential.

## Rules

- Binary values in any DSM request body are **base64-encoded**.
- **Ignore unknown response fields.** Fortanix reserves the right to add fields at
  any time; a strict deserializer will break.
- The spec declares **no 4xx responses**. Do not assume a machine-readable error
  body — branch on the HTTP status code and treat the body as opaque text.
- There is **no idempotency key**. Do not blind-retry mutating calls.
