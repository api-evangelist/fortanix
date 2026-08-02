---
name: Create a key and encrypt data with Fortanix DSM
description: Create a group-scoped security object in Fortanix DSM, then encrypt and decrypt data with it, including the multi-part path for large payloads.
api: openapi/fortanix-dsm-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-dsm-openapi-original.json
operations:
  - CreateGroup
  - ListGroups
  - CreateSobject
  - GetSobject
  - ListSobjects
  - Encrypt
  - Decrypt
  - EncryptInit
  - EncryptUpdate
  - EncryptFinal
  - DecryptInit
  - DecryptUpdate
  - DecryptFinal
  - BatchEncrypt
  - BatchDecrypt
---

# Create a key and encrypt data with Fortanix DSM

Prerequisite: an authorized session — see `fortanix-authenticate-session.md`.

The Fortanix data model is `Account → Group → Sobject`. A **security object**
(`Sobject`) is the key; the **group** is the authorization boundary. You cannot create
a key without choosing a group, and an app can only use keys in groups it is a member of.

## Steps

1. **Pick or create the group.**
   - `GET /sys/v1/groups` (`ListGroups`) to find an existing one.
   - `POST /sys/v1/groups` (`CreateGroup`) to make a new one. A group may be backed by
     an external HSM/KMS (Azure Key Vault, GCP Key Ring, AWS KMS, OCI Vault) rather
     than DSM itself.
2. **Create the key.** `POST /crypto/v1/keys` (`CreateSobject`) with the object type,
   key size, group id and the operations the key is permitted to perform. To bring
   your own key material instead, use `PUT /crypto/v1/keys` (`ImportSobject`).
3. **Confirm it.** `POST /crypto/v1/keys/info` (`GetSobject`) returns the object,
   including its `kid` and lifecycle state. A newly created key must be **Active**
   before it will perform crypto; if it is Pre-Active, call
   `POST /crypto/v1/keys/{key_id}/activate` (`ActivateSobject`).
4. **Encrypt.** `POST /crypto/v1/encrypt` (`Encrypt`) with the `kid`, the algorithm/mode,
   and the **base64-encoded** plaintext. Keep the returned IV/nonce and any auth tag —
   you need them to decrypt.
5. **Decrypt.** `POST /crypto/v1/decrypt` (`Decrypt`) with the same `kid`, the ciphertext
   and the IV.

## Large payloads

Do not buffer a large object into one request. Use the streaming triple:

`POST /crypto/v1/encrypt/init` (`EncryptInit`) →
`POST /crypto/v1/encrypt/update` (`EncryptUpdate`, repeated) →
`POST /crypto/v1/encrypt/final` (`EncryptFinal`)

and the mirror-image `DecryptInit` / `DecryptUpdate` / `DecryptFinal`. The init call
returns a state handle you must pass to every subsequent call in the sequence.

## Many payloads

To encrypt or decrypt many items — possibly across different keys — in one round trip,
use `POST /crypto/v1/keys/batch/encrypt` (`BatchEncrypt`) and
`POST /crypto/v1/keys/batch/decrypt` (`BatchDecrypt`).

## Rules

- **All binary input is base64-encoded.** Plaintext, ciphertext, IVs and auth tags all
  travel as base64 strings (`format: byte` in the spec).
- **Listing is limit/offset.** `GET /crypto/v1/keys` (`ListSobjects`) takes `limit`
  (default 1000) and `offset`. Pass `with_metadata=true` to get
  `{ items, metadata: { total_count, filtered_count } }` instead of a bare array, which
  is the only way to know whether more pages exist.
- **Filtering is a JSON expression** with `$`-prefixed operators (`$and`, `$or`, `$not`,
  `$eq`, `$lt`, `$gte`, `$text`, `$range`, `$exists`) over attributes such as `name`,
  `kid`, `group_name`, `created_at`, `key_size` and `custom_attributes`.
- **No idempotency key exists.** A retried `CreateSobject` creates a second key. Check
  with `ListSobjects` filtered by name before retrying a creation you are unsure about.
- **The spec declares no error responses.** Branch on HTTP status; do not parse an
  error envelope that is not contractually defined.
