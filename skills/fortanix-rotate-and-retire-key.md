---
name: Rotate and retire a Fortanix DSM key
description: Rotate a security object, verify the new version, then deactivate or destroy the old one — the highest-consequence flow in Fortanix DSM.
api: openapi/fortanix-dsm-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-dsm-openapi-original.json
operations:
  - GetSobject
  - RotateSobject
  - ReplaceSobject
  - RevertPrevKeyOp
  - RevokeSobject
  - DestroySobject
  - RemovePrivate
  - GetKcv
  - VerifyKcv
  - GetAllLogs
---

# Rotate and retire a Fortanix DSM key

Prerequisite: an authorized session — see `fortanix-authenticate-session.md`.

**This is a destructive flow.** `DestroySobject` renders a key permanently unusable for
cryptography. An agent must not run steps 4–5 without explicit human confirmation.

## The lifecycle

A Fortanix security object moves through **Pre-Active → Active → Deactivated →
Compromised → Destroyed**. Destroyed objects keep their metadata (so audit trails stay
intact) but can never perform another cryptographic operation.

## Steps

1. **Read the current state.** `POST /crypto/v1/keys/info` (`GetSobject`). Record the
   `kid`, the state and the group. Refuse to proceed if the object is already Deactivated
   or Destroyed.
2. **Rotate.** `POST /crypto/v1/keys/rekey` (`RotateSobject`) creates a new version and
   links it to the previous one. Use `POST /crypto/v1/keys/replace` (`ReplaceSobject`)
   instead when you need to rotate *onto* an existing security object.
3. **Verify the new material before retiring the old.** For AES/DES/DES3 keys, compute
   a key check value with `POST /crypto/v1/keys/kcv` (`GetKcv`) and confirm it with
   `POST /crypto/v1/keys/kcv/verify` (`VerifyKcv`). Do not skip this — retiring the old
   key before the new one is proven is how data becomes unrecoverable.
4. **Deactivate the old version.** `POST /crypto/v1/keys/{key_id}/revoke`
   (`RevokeSobject`) transitions it to Deactivated or Compromised. This is reversible in
   effect (the material still exists) and is the correct stopping point for most rotations.
5. **Destroy only when policy requires it.** `POST /crypto/v1/keys/{key_id}/destroy`
   (`DestroySobject`). **Irreversible.** For asymmetric keys where you only need to
   remove the ability to sign or decrypt, prefer
   `DELETE /crypto/v1/keys/{key_id}/private` (`RemovePrivate`), which drops the private
   half and leaves the public half usable for verification.
6. **Record it.** `GET /sys/v1/logs` (`GetAllLogs`) filtered by `object_id` gives the
   audit trail for the rotation. Paginate with `size`, `from` and the `previous_id`
   cursor, bounded by `range_from` / `range_to` epoch seconds.

## Undo

`PUT /crypto/v1/keys/{key_id}/revert` (`RevertPrevKeyOp`) reverts a security object to
a previous state. It is the only rollback available and it does not undo a destroy.

## Quorum approval

If the account applies a quorum policy to key destruction, the destructive call will not
execute inline. It becomes an approval request — see
`fortanix-quorum-approval-workflow.md`. Treat "approval pending" as a normal branch,
not a failure.

## Rules

- **No idempotency key.** A retried `RotateSobject` rotates twice. Always re-read with
  `GetSobject` before retrying.
- **The spec declares no error responses.** Branch on HTTP status only.
- Destroy and revoke are classified `safety-critical` in
  `agentic-access/fortanix-agentic-access.yml`; they require human-in-the-loop.
