---
name: Run a Fortanix DSM quorum approval workflow
description: Create, track, approve or deny a quorum-gated Fortanix DSM operation and collect its deferred result — the correct way to handle any operation that does not execute inline.
api: openapi/fortanix-dsm-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-dsm-openapi-original.json
operations:
  - CreateApprovalRequest
  - ListApprovalRequests
  - GetApprovalRequest
  - ApproveRequest
  - DenyRequest
  - MfaChallenge
  - GetApprovalRequestResult
  - DeleteApprovalRequest
  - GetAllLogs
---

# Run a Fortanix DSM quorum approval workflow

Prerequisite: an authorized session — see `fortanix-authenticate-session.md`.

When a Fortanix group or account applies a quorum policy, a gated operation does **not**
run when you call it. It becomes an **approval request** carrying the deferred operation,
and the result is collected separately once enough approvers have signed off. An agent
that treats this as an error will corrupt its own state model — treat it as a normal
control-flow branch.

## Steps

1. **Create the request.** `POST /sys/v1/approval_requests` (`CreateApprovalRequest`)
   with the operation to defer. The response carries the `req_id` — persist it; it is
   the only handle you have on the pending work.
2. **Track it.** `GET /sys/v1/approval_requests/{req_id}` (`GetApprovalRequest`) returns
   the current state. `GET /sys/v1/approval_requests` (`ListApprovalRequests`) lists all
   requests visible to the caller, for a queue view.
3. **Approve or deny.** An approver calls
   `POST /sys/v1/approval_requests/{req_id}/approve` (`ApproveRequest`) or
   `POST /sys/v1/approval_requests/{req_id}/deny` (`DenyRequest`).
   Where the policy demands a second factor, the approver first obtains a challenge from
   `POST /sys/v1/approval_requests/{req_id}/challenge` (`MfaChallenge`).
4. **Collect the result.** Once the quorum is met,
   `POST /sys/v1/approval_requests/{req_id}/result` (`GetApprovalRequestResult`) returns
   the output of the original deferred operation. This is the **only** place that result
   appears — the approve call does not return it.
5. **Withdraw.** `DELETE /sys/v1/approval_requests/{req_id}` (`DeleteApprovalRequest`)
   cancels a request that is no longer wanted.
6. **Record it.** `GET /sys/v1/logs` (`GetAllLogs`) filtered by `object_id` gives the
   full approval trail.

## Agent rules

- **Never auto-approve.** An agent that both creates and approves a request has defeated
  the entire purpose of the quorum control. `ApproveRequest` must be driven by a human.
- **Poll, do not spin.** There is no webhook, no event stream and no AsyncAPI surface
  anywhere in the Fortanix platform. `GetApprovalRequest` polling is the only way to
  learn that state changed. Back off between polls.
- **Persist `req_id` durably.** Losing it strands the deferred operation; there is no
  way to correlate it back from the target object.
- **No idempotency key.** A retried `CreateApprovalRequest` opens a second pending
  request for the same work. List before you retry.
- **The spec declares no error responses.** Branch on HTTP status only.
