---
name: Read Fortanix Key Insight discovery scans and PQC reports
description: Authenticate to the Fortanix Armor / Key Insight API with OAuth 2.0 client credentials, then read discovery connections, scans, scan inventory and the AWS/Azure/on-prem/DSM/post-quantum report families.
api: openapi/fortanix-armor-key-insight-openapi-original.json
generated: '2026-08-01'
method: generated
source: openapi/fortanix-armor-key-insight-openapi-original.json
operations:
  - OauthToken
  - GetAllConnections
  - GetConnection
  - GetAllScans
  - GetScan
  - GetScanInventoryObjects
  - GetScanInventoryObject
  - GetInventoryObjects
  - GetInventoryObject
  - GetAllPolicies
  - GetPolicy
  - GetCumulativePqcReport
  - GetPqcReport
  - GetPqcReportConnections
  - GetPqcReportServices
  - GetAwsScanSummaryReport
  - GetAwsScanAssessmentReport
  - GetAwsScanKeyUsageReport
  - GetAzureScanSummaryReport
  - GetOnPremFsScanSummaryReport
  - GetDsmScanSummaryReport
  - GetServicesReportGroupedByServiceType
  - GetServicesReportGroupedByViolationType
  - Terminate
---

# Read Fortanix Key Insight discovery scans and PQC reports

Key Insight answers "what cryptography is actually deployed across my estate, and how
much of it is post-quantum ready". This is the Armor / Key Insight API — a different
surface from DSM, with a different host and a different auth model.

## Scope of this API — read this first

**The public Armor / Key Insight API is read-only.** All 34 discovery operations are
`GET`. There is no operation to create a connection, define a policy, or start a scan.
Connections and scans are configured in the Key Insight UI; the API consumes what the
UI produced. An agent must not promise to "run a scan" — it can only read scans that
already exist.

## Base URL

`https://api.armor.fortanix.com` — or `https://api.na.armor.fortanix.com` for the
North America endpoint.

## Authentication

1. `POST /api/v1/iam/session/oauth2/token` (`OauthToken`) — OAuth 2.0 **client
   credentials** (RFC 6749 §4.4). Use the returned access token as a bearer token.
2. The scheme declares an **empty scopes map**. There are no published OAuth scopes to
   request; authorization rides on the credential's identity. Federated
   client-credentials setups against Okta and Auth0 are documented separately.
3. `POST /api/v1/iam/session/terminate` (`Terminate`) ends the session.

## Steps

1. **List the environments under discovery.** `GET /api/v1/discovery/connections`
   (`GetAllConnections`); one connection by id with `GetConnection`.
2. **Find the scan.** `GET /api/v1/discovery/scans` (`GetAllScans`), then
   `GET /api/v1/discovery/scans/{id}` (`GetScan`) for status and metadata. Scans are
   long-running; poll `GetScan` — there is no webhook or event stream.
3. **Read the raw findings.**
   `GET /api/v1/discovery/scans/{id}/scan_inventory_objects` (`GetScanInventoryObjects`)
   lists every cryptographic asset one scan found;
   `GetScanInventoryObject` addresses a single asset by
   `scan_id` + `scan_inventory_object_id`.
   For the aggregated estate view across all scans use
   `GET /api/v1/discovery/inventory_objects` (`GetInventoryObjects`).
4. **Check what was being enforced.** `GET /api/v1/discovery/policies`
   (`GetAllPolicies`) and `GetPolicy`.
5. **Pull the reports.** They are split by environment and by shape:
   - **Post-quantum** — `GetCumulativePqcReport` (estate-wide),
     `GetPqcReport` (per scan), `GetPqcReportConnections`, `GetPqcReportServices`.
     This is the headline output of the product.
   - **AWS** — `GetAwsScanSummaryReport`, `GetAwsScanAssessmentReport`,
     `GetAwsScanKeyUsageReport`, `GetScannedAwsAccounts`.
   - **Azure** — `GetAzureScanSummaryReport`, `GetAzureScanAssessmentReport`,
     `GetAzureScanKeyUsageReport`, `GetScannedAzureSubscriptions`.
   - **On-premises** — filesystem, database and source-code variants
     (`GetOnPremFsScanSummaryReport`, `GetOnpremDatabaseScanSummaryReport`,
     `GetOnpremSourceCodeScanSummaryReport`, plus their assessment-report siblings and
     `GetOnpremDiscoveryScanSummaryReport`).
   - **Fortanix DSM** — `GetDsmScanSummaryReport`, for keys already under DSM management.
   - **Cross-service rollups** — `GetServicesReportGroupedByServiceType`,
     `GetServicesReportGroupedByViolationType`,
     `GetAwsServicesReportGroupedByAccounts`,
     `GetAzureServicesReportGroupedBySubscriptions`.

## Rules

- **You must follow redirects, including 308.** Published verbatim on the API reference:
  most HTTP libraries (Python `requests`, JavaScript `fetch`) do this by default; with
  cURL you must pass `--location`. A client that does not follow 308s will silently fail.
- The **only** error response declared anywhere in this spec is a `401` on the
  authentication surface. Everything else must be handled by HTTP status code with an
  opaque body.
- Every discovery operation is addressed by a path `id` — carry the `scan_id` through
  the whole report family rather than re-listing.
