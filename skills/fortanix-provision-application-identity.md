---
name: Provision a Fortanix DSM application identity
description: Create an application in Fortanix DSM, grant it group membership so it can use specific keys, retrieve its credential and rotate that credential.
api: openapi/fortanix-dsm-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-dsm-openapi-original.json
operations:
  - CreateGroup
  - ListGroups
  - CreateApp
  - GetApp
  - ListApps
  - UpdateApp
  - AddGroupMembership
  - GetAllGroupMemberships
  - UpdateGroupMembership
  - DeleteGroupMembership
  - GetAppCredential
  - ResetAppSecret
  - GetClientConfigs
  - DeleteApp
---

# Provision a Fortanix DSM application identity

Prerequisite: an authorized session with account-admin rights — see
`fortanix-authenticate-session.md`.

An **App** is Fortanix's machine identity. It holds a credential (API key or client
certificate) and a set of **group memberships**. Group membership is the whole
authorization story: an app can operate on a security object if and only if it is a
member of that object's group.

## Steps

1. **Decide the blast radius first.** `GET /sys/v1/groups` (`ListGroups`). Give the app
   the narrowest group set that satisfies its job. Create a dedicated group with
   `POST /sys/v1/groups` (`CreateGroup`) rather than adding an app to a broad shared one.
2. **Create the app.** `POST /sys/v1/apps` (`CreateApp`) with its name, description,
   credential type and initial group memberships.
3. **Grant additional groups.** `POST /sys/v1/apps/{app_id}/groups` (`AddGroupMembership`)
   adds a membership; `PATCH /sys/v1/apps/{app_id}/groups/{group_id}`
   (`UpdateGroupMembership`) narrows the permitted operations within one group;
   `DELETE /sys/v1/apps/{app_id}/groups/{group_id}` (`DeleteGroupMembership`) removes it.
4. **Audit what it can reach.** `GET /sys/v1/apps/{app_id}/groups`
   (`GetAllGroupMemberships`) is the authoritative answer to "what can this identity
   touch".
5. **Retrieve the credential.** `GET /sys/v1/apps/{app_id}/credential`
   (`GetAppCredential`). Hand the API key to the workload through a secret store — never
   log it, never commit it.
6. **Fetch client configuration.** `GET /sys/v1/apps/client_configs` (`GetClientConfigs`)
   returns configuration for the PKCS#11, CNG and JCE clients. This operation can only
   be called *by an app*, using the app's own credential.
7. **Rotate the credential.** `POST /sys/v1/apps/{app_id}/reset_secret` (`ResetAppSecret`)
   issues a new API key. The old key stops working, so deploy the new one first and cut
   over deliberately.
8. **Decommission.** `DELETE /sys/v1/apps/{app_id}` (`DeleteApp`).

## Rules

- **`PATCH` is a partial update.** All top-level fields are optional and omitted fields
  keep their existing value — but a nested object must be sent **whole**, not partially.
  Sending a partial nested object silently replaces the rest.
- **Listing is limit/offset** with `with_metadata=true` for the
  `{ items, metadata }` envelope.
- **No idempotency key.** A retried `CreateApp` creates a second identity with a second
  live credential. Check `GET /sys/v1/apps` (`ListApps`) before retrying.
- **The spec declares no error responses.** Branch on HTTP status.
- Credential retrieval and reset are high-consequence. Gate them behind human approval
  in any autonomous agent.
