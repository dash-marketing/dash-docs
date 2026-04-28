# Customer Import API Documentation

## Overview

The Customer Import API ingests three core resources from your systems into Dash Marketing:

- **Transactions** — individual parking sessions, sales, or reservations
- **Accounts** — monthly-billing companies (commercial parkers, fleets)
- **Subscriptions** — recurring permits attached to an account

It is designed for backend-to-backend integrations: nightly batches from a PMS, near-real-time forwarders from a POS, or one-time historical backfills. All endpoints accept JSON, are idempotent on stable customer-supplied reference IDs, and return per-record outcomes so you can reconcile partial successes deterministically.

**Try it in Postman:** [dash-customer-import.postman_collection.json](./dash-customer-import.postman_collection.json) — pre-wired collection covering every endpoint, with shared variables for `baseUrl` and `apiKey` and small sample bodies you can send immediately.

**Base URL:** `https://api.dashmarketing.io`

**Endpoints:**

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/integrations/transactions` | Submit a batch of transactions. Resolves or creates related account/contact/vehicle/subscription inline. |
| `POST` | `/integrations/accounts` | Submit a batch of accounts (monthly-billing companies). |
| `POST` | `/integrations/subscriptions` | Submit a batch of subscriptions/permits. Each must reference an account. |
| `GET`  | `/integrations/locations` | List your locations with their `uuid` and `timeZone`, for resolving your internal location IDs. |
| `GET`  | `/integrations/imports` | Paginated history of recent import runs for your organization. |
| `GET`  | `/integrations/imports/{referenceUuid}` | Full per-record results for a prior import run. |

---

## Authentication

All requests require an API key in the `X-API-KEY` header.

```
X-API-KEY: your_api_key_here
```

Keys are scoped to a single organization. Every record you submit is automatically attributed to that organization — there is no `orgUuid` field in any request body.

### Obtaining a Key

See the [main API documentation](./API_DOCUMENTATION.md#authentication) for key creation. For backend integrations, create a dedicated key per integration (e.g., "PMS-Transactions-Sync", "Backfill-2026-Q1") so it can be rotated or revoked independently.

### Security Notes

- Store keys in your secrets manager (AWS Secrets Manager, Vault, etc.); never in source.
- The Customer Import endpoints are intended to be called from server-side code only. Do not embed keys in browser, mobile, or other client-side bundles.
- Submit through HTTPS only. Plain-HTTP requests are refused.

---

## Core Concepts

### Idempotency and Dedup Keys

Every resource has a customer-owned reference ID that doubles as the dedup key. Replays of the same record are detected and reported as `Skipped` — they are not double-inserted.

| Resource | Dedup field | Scope |
|----------|-------------|-------|
| Transaction | `transactionId` | Unique within your organization |
| Account | `tpReferenceId` | Unique within your organization |
| Subscription | `tpReferenceId` | Unique within your organization |

> `tpReferenceId` stands for "third-party reference ID" — it's whatever stable identifier your source system already uses (account number, permit ID, customer ID).

This means safe-by-default replay: if a webhook delivery fails and you retry the same batch, every previously-accepted record comes back as `Skipped`, and any new ones are `Created`.

### Resolving Related Entities

Transactions naturally have an account, contact, vehicle, and (for monthlies) subscription. When submitting transactions you have three resolution strategies per relation, in order of priority:

1. **By Dash UUID** — `accountUuid`, `contactUuid`, `vehicleUuid`, `subscriptionUuid`. Use when you've already imported the entity and stored the returned UUID.
2. **By third-party reference** — `accountTPReferenceId`, `subscriptionTPReferenceId`. Resolves an existing entity by its external reference. Does not create new entities.
3. **Nested object** — `account`, `contact`, `vehicle`, `subscription`. Resolves an existing match if one exists; otherwise creates it inline.

You can mix strategies freely within a batch and even within a single record. The resolver matches:

- **Contacts** — by email, then by phone number
- **Vehicles** — by license plate
- **Accounts / Subscriptions** — by `tpReferenceId`

### Synchronous vs. Asynchronous

POST endpoints are synchronous. The full per-record `results` array is returned in the response body, along with a `referenceUuid` you can persist and look up later via `GET /integrations/imports/{referenceUuid}`.

Even on a fatal server-side error, the response includes the `referenceUuid` so you can correlate logs. No work is committed silently — every accepted record is reflected in `results`.

---

## Discover Your Locations

Before you can submit transactions, you need the Dash `locationUuid` for each of your physical locations. Use the locations endpoint to fetch them.

### Endpoint

```
GET /integrations/locations
```

### Query Parameters

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `page` | integer | 1 | 1-based page number |
| `pageSize` | integer | 100 | Page size (max 500) |
| `query` | string | — | Free-text search across name/address/city |

### Response

```json
{
  "data": [
    {
      "uuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "name": "Downtown Garage",
      "address": "123 Main St",
      "city": "New York",
      "state": "NY",
      "zipcode": "10001",
      "timeZone": "America/New_York",
      "isActive": true,
      "tpReferenceId": "GARAGE-001"
    }
  ],
  "totalCount": 42,
  "pageSize": 100,
  "currentPage": 1,
  "totalPages": 1,
  "hasPreviousPage": false,
  "hasNextPage": false
}
```

`tpReferenceId` is the external ID we have on file for that location. If you set it during onboarding, you can use it to map your internal location IDs to Dash `uuid`s once at startup.

`timeZone` is an IANA zone (e.g., `America/New_York`). Inherited by every transaction submitted for that location — see [Time Zones](#time-zones-and-timestamps) below.

**Recommendation:** call this endpoint once at process startup, build an in-memory map (your-internal-id → Dash uuid), and refresh on a long interval (hourly or daily). The endpoint is rate-limited to **300 requests per minute per organization** — plenty for periodic refresh, but not for per-transaction lookups.

---

## Submit Transactions

### Endpoint

```
POST /integrations/transactions
```

### Limits

| Constraint | Value |
|------------|-------|
| Records per request | **500** |
| Request body size | **5 MB** |
| Rate limit | **60 requests per 60 seconds per organization** |

Exceeding records-per-batch returns `413 Payload Too Large`. Exceeding body size returns `413` from the framework with no body. Exceeding rate limit returns `429` with a `Retry-After` header.

### Request Envelope

```json
{
  "transactions": [ /* TransactionImportItem */ ],
  "metadata": { "batchSource": "pms-nightly", "runId": "2026-04-27T03:00:00Z" }
}
```

The top-level `metadata` object is stored verbatim on the import log for your own debugging. It is not propagated onto individual transaction records.

### TransactionImportItem

#### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `transactionId` | string (≤128) | Your external transaction ID. Idempotency key. |
| `locationUuid` | uuid | Dash location UUID. Must belong to your org. |
| `type` | enum | `1` Transient, `2` Monthly, `3` Flex, `4` Reservation |
| `transactionDate` | ISO 8601 | When the transaction occurred (UTC). |

#### Optional Financial Fields

All amounts are decimals. Send `0` (not `null`) when not applicable; only `promotionAmount` is genuinely nullable.

| Field | Type | Description |
|-------|------|-------------|
| `amountTotal` | decimal | Gross customer charge. |
| `netAmount` | decimal | Operator net (after fees). |
| `convenienceFee` | decimal | Convenience fee component. |
| `processingFee` | decimal | Processing fee component. |
| `promotionAmount` | decimal? | Promotion/discount amount. |
| `cardType` | string (≤64) | `Visa`, `Mastercard`, etc. |
| `isRefunded` | bool | Whether this record represents a refund. |
| `promoApplied` | bool | Whether a promo was applied. |
| `promoCode` | string (≤64) | Promo code text. |

#### Optional Session Fields

| Field | Type | Description |
|-------|------|-------------|
| `sessionStart` | ISO 8601 | Actual park-in time. |
| `sessionEnd` | ISO 8601 | Actual park-out time. |
| `scheduledArrival` | ISO 8601 | Reservation arrival time. |
| `scheduledExit` | ISO 8601 | Reservation exit time. |

#### Booking Source

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `bookingSource` | enum | `6` (CustomerApi) | `1` App, `2` Website, `3` Whitelabel, `4` Text, `5` Other, `6` CustomerApi |

For most integrations leave this unset and the default `CustomerApi` is applied.

#### Related Entities

Each relation accepts any of the three resolution strategies (UUID / TPRef / nested). Provide whichever you have.

| Field | Type | Description |
|-------|------|-------------|
| `accountUuid` | uuid? | Existing Dash account UUID. |
| `accountTPReferenceId` | string (≤128) | Resolve an existing account by external ref. Does not create. |
| `account` | object? | Nested account — created if no match by `tpReferenceId`. See [Account Object](#account-object). |
| `contactUuid` | uuid? | Existing Dash contact UUID. |
| `contact` | object? | Nested contact — matched by email/phone, created if no match. See [Contact Object](#contact-object). |
| `vehicleUuid` | uuid? | Existing Dash vehicle UUID. |
| `vehicle` | object? | Nested vehicle — matched by `licensePlate`, created if no match. See [Vehicle Object](#vehicle-object). |
| `subscriptionUuid` | uuid? | Existing Dash subscription UUID. |
| `subscriptionTPReferenceId` | string (≤128) | Resolve an existing subscription by external ref. Does not create. |
| `subscription` | object? | Nested subscription — only created if an account is also resolved on this record. See [Subscription Object](#subscription-object). |

#### Metadata

```json
"metadata": { "anyKey": "any-json-value" }
```

Per-record `metadata` is stored on the transaction and retrievable later. Use for fields you collect that don't map to a static field (lot section, attendant ID, integration-specific identifiers).

---

### Account Object

Used inline on a transaction or as a top-level item on `POST /integrations/accounts`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tpReferenceId` | string (≤128) | yes | External account ID — dedup key. |
| `name` | string (≤256) | yes | Company name. |
| `accountNumber` | string (≤128) | no | Display account number. |
| `type` | enum | no | `1` Monthly (default), `2` Flex, `3` Reservation. |
| `isActive` | bool | no | Defaults to `true`. |
| `contactName` | string (≤256) | no | Primary contact name. |
| `contactEmail` | string (≤320) | no | Primary contact email. |
| `contactPhoneNumber` | string (≤32) | no | Primary contact phone. |
| `locationUuid` | uuid | no | Optional primary location. Must belong to your org. |
| `metadata` | object | no | Arbitrary JSON. |

### Contact Object

Inline on a transaction.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string (≤320) | one of email/phone | Used for matching. |
| `phoneNumber` | string (≤32) | one of email/phone | Used for matching when no email match. |
| `firstName` | string (≤128) | no | |
| `lastName` | string (≤128) | no | |
| `tpReferenceId` | string (≤128) | no | Your customer ID, stored for cross-reference. |
| `metadata` | object | no | |

### Vehicle Object

Inline on a transaction.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `licensePlate` | string (≤32) | yes | Used for matching. |
| `state` | string (≤8) | no | US state / province code. |
| `year` | integer | no | |
| `make` | string (≤64) | no | |
| `model` | string (≤64) | no | |
| `color` | string (≤32) | no | |
| `type` | string (≤32) | no | Free-form vehicle type. |
| `metadata` | object | no | |

### Subscription Object

Used inline on a transaction or as a top-level item on `POST /integrations/subscriptions`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tpReferenceId` | string (≤128) | yes | External permit ID — dedup key. |
| `accountUuid` | uuid | one of (uuid / tpref / nested account) | Existing account UUID. |
| `accountTPReferenceId` | string (≤128) | one of | Resolve account by external ref. |
| `locationUuid` | uuid | no | Subscription's home location. |
| `permitNumber` | string (≤64) | no | |
| `status` | enum | no | `1` Active (default), `2` Suspended, `3` Cancelled, `4` PendingApproval, `5` Expired. |
| `permitType` | string (≤64) | no | |
| `permitClassification` | string (≤64) | no | |
| `numberOfSpaces` | integer (1–1000) | no | Defaults to `1`. |
| `startDate` | ISO 8601 | no | |
| `endDate` | ISO 8601 | no | |
| `currentBaseRate` | decimal | no | |
| `currentRate` | decimal | no | |
| `currentTax` | decimal | no | |
| `currentFeeTotal` | decimal | no | |
| `metadata` | object | no | |

When a `subscription` is nested on a transaction, it is created only if the transaction also resolves an account (otherwise it is silently dropped — there is nowhere to attach it).

---

## Submit Accounts

For onboarding accounts ahead of any transactions (a typical first step in a migration).

### Endpoint

```
POST /integrations/accounts
```

### Limits

| Constraint | Value |
|------------|-------|
| Records per request | **100** |
| Request body size | **5 MB** |
| Rate limit | **60 requests per 60 seconds per organization** |

### Request

```json
{
  "accounts": [ /* AccountImportItem */ ],
  "metadata": { "source": "initial-migration" }
}
```

Each item is the [Account Object](#account-object) defined above. Items with a `tpReferenceId` that already exists are returned as `Skipped` with the existing UUID — useful for re-running migrations.

---

## Submit Subscriptions

For onboarding subscriptions/permits independent of transactions.

### Endpoint

```
POST /integrations/subscriptions
```

### Limits

| Constraint | Value |
|------------|-------|
| Records per request | **100** |
| Request body size | **5 MB** |
| Rate limit | **60 requests per 60 seconds per organization** |

### Request

```json
{
  "subscriptions": [ /* SubscriptionImportItem */ ],
  "metadata": { "source": "initial-migration" }
}
```

Each item is the [Subscription Object](#subscription-object) defined above. **Each subscription must resolve to an existing account** — either via `accountUuid` or `accountTPReferenceId`. There is no inline account creation on this endpoint; submit accounts first.

A subscription that cannot resolve an account returns `Failed` with `reasonCode: AccountNotFound`.

---

## Response Shape

All four POST endpoints — and the GET-by-reference endpoint — return the same `CustomerImportResponse` shape.

### Success / Partial Success

**HTTP Status:** `200 OK`

```json
{
  "referenceUuid": "f9e8d7c6-b5a4-3210-9876-543210fedcba",
  "resource": 1,
  "status": 1,
  "startedAt": "2026-04-27T03:00:00.123Z",
  "completedAt": "2026-04-27T03:00:01.456Z",
  "durationMs": 1333,
  "totalSubmitted": 3,
  "successCount": 2,
  "failedCount": 0,
  "skippedCount": 1,
  "createdEntities": {
    "accounts": 0,
    "contacts": 1,
    "vehicles": 1,
    "subscriptions": 0
  },
  "results": [
    {
      "reference": "POS-2026-0001",
      "status": 1,
      "uuid": "11111111-2222-3333-4444-555555555555"
    },
    {
      "reference": "POS-2026-0002",
      "status": 1,
      "uuid": "66666666-7777-8888-9999-aaaaaaaaaaaa"
    },
    {
      "reference": "POS-2026-0001",
      "status": 3,
      "reasonCode": 1,
      "errorDetail": "Transaction with this ID already exists for the organization."
    }
  ]
}
```

### Top-level fields

| Field | Description |
|-------|-------------|
| `referenceUuid` | Stable UUID for this run. Persist this — you can re-fetch results later via `GET /integrations/imports/{referenceUuid}`. |
| `resource` | `1` Transactions, `2` Accounts, `3` Subscriptions. |
| `status` | `1` Completed, `2` Failed, `3` PartialSuccess. (Other values are reserved for vendor-pull flows and not returned by these endpoints.) |
| `totalSubmitted` | Echo of input length. |
| `successCount` | Records with status `Created` or `Updated`. |
| `failedCount` / `skippedCount` | Records with `Failed` / `Skipped` status. |
| `createdEntities` | Count of newly created related entities (only meaningful for `/integrations/transactions`). |
| `results` | Per-record outcomes. Always returned in input order. |

### Per-record `results` items

| Field | Description |
|-------|-------------|
| `reference` | The customer-supplied reference: `transactionId` for transactions, `tpReferenceId` for accounts/subscriptions. |
| `status` | `1` Created, `2` Updated, `3` Skipped, `4` Failed. |
| `uuid` | Dash-assigned UUID when `status` is `Created` (or `Skipped` against an existing record). |
| `reasonCode` | Structured failure code — see [Reason Codes](#reason-codes). |
| `errorDetail` | Human-readable message when `status = Failed`. |

### Validation Error

**HTTP Status:** `400 Bad Request`

```json
{
  "error": "At least one transaction is required."
}
```

Returned when the batch envelope itself is invalid (empty array, missing required fields). The framework also returns `400` with a `ValidationProblemDetails` shape for malformed JSON or missing top-level required fields.

### Authentication Error

**HTTP Status:** `401 Unauthorized` if the API key is missing/invalid; `403 Forbidden` if the key is valid but lacks permission for the resource.

### Batch Too Large

**HTTP Status:** `413 Payload Too Large`

```json
{
  "error": "Batch size exceeds the maximum of 500 transactions per request.",
  "submitted": 750,
  "max": 500
}
```

### Rate Limit Exceeded

**HTTP Status:** `429 Too Many Requests` with `Retry-After: 60` header.

```json
{
  "success": false,
  "message": "Rate limit exceeded. Please try again later.",
  "retryAfter": 60,
  "timestamp": "2026-04-27T03:00:00.000Z"
}
```

---

## Read Past Imports

### List recent imports

```
GET /integrations/imports?page=1&pageSize=20
```

Returns a paginated list of recent runs across all three resources, most recent first. Each item is a compact summary:

```json
{
  "data": [
    {
      "referenceUuid": "f9e8d7c6-b5a4-3210-9876-543210fedcba",
      "resource": 1,
      "status": 1,
      "startedAt": "2026-04-27T03:00:00.123Z",
      "completedAt": "2026-04-27T03:00:01.456Z",
      "durationMs": 1333,
      "totalSubmitted": 500,
      "successCount": 498,
      "failedCount": 0,
      "skippedCount": 2
    }
  ],
  "totalCount": 1240,
  "pageSize": 20,
  "currentPage": 1,
  "totalPages": 62,
  "hasPreviousPage": false,
  "hasNextPage": true
}
```

`pageSize` is clamped to 1–100. Rate limit: 300/min per organization.

### Get full results by reference

```
GET /integrations/imports/{referenceUuid}
```

Returns the same response shape you got at submit time, including the full `results` array. Useful for:

- Reconciling against your own database after the fact
- Debugging without re-submitting
- Async workflows where the submitting process and the auditing process are different

Returns `404 Not Found` if the `referenceUuid` doesn't belong to your organization.

---

## Reason Codes

`reasonCode` is an integer enum returned on `Failed` and `Skipped` records.

| Code | Name | Meaning |
|------|------|---------|
| 1 | `DuplicateTransactionId` | Returned for Skipped records whose dedup key already exists. Despite the name, applies to all three resources (transaction id, account/subscription `tpReferenceId`). |
| 2 | `LocationNotFound` | `locationUuid` does not belong to your organization. |
| 3 | `InvalidVendor` | Reserved for internal use. |
| 4 | `MissingRequiredField` | A required field on a nested entity was missing. |
| 5 | `ValidationFailed` | Domain validation failed (e.g., out-of-range numeric). |
| 6 | `AccountNotFound` | A subscription could not resolve its account by either `accountUuid` or `accountTPReferenceId`. |
| 7 | `SubscriptionNotFound` | A reference to an existing subscription could not be resolved. |
| 8 | `ContactNotFound` | A reference to an existing contact could not be resolved. |
| 9 | `VehicleNotFound` | A reference to an existing vehicle could not be resolved. |
| 99 | `InternalError` | Unhandled server-side error during record processing. The batch is committed for all other records. |

**Decision rule for clients:** treat `Skipped` + `DuplicateTransactionId` as success; treat `Failed` as something to retry only after the input has been corrected. `InternalError` may be retried — but only for the affected records, not the whole batch.

---

## Time Zones and Timestamps

- All timestamps in requests and responses are ISO 8601. Send UTC (`Z` suffix).
- Each transaction is stored with the `timeZone` of its `locationUuid` — pulled from the location's IANA zone at write time.
- Dash analytics endpoints bucket by location-local wall-clock time. Practical implication: a session at `2026-04-27T03:30:00Z` for a `locationUuid` in `America/New_York` is reported as occurring on April 26th in analytics.
- If a transaction's `transactionDate` is local-wall-clock in your source system, convert to UTC before sending. Do not send local times with a `Z` suffix; you will see analytics drift equal to the location offset.

---

## Examples

### Example 1: Bootstrap — one account, one subscription, one transaction

The most realistic first integration test: prove the full graph by submitting one transaction that creates everything inline.

```bash
curl -X POST https://api.dashmarketing.io/integrations/transactions \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: $DASH_API_KEY" \
  -d '{
    "transactions": [
      {
        "transactionId": "TEST-0001",
        "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "type": 2,
        "bookingSource": 6,
        "transactionDate": "2026-04-27T18:30:00Z",
        "amountTotal": 250.00,
        "netAmount": 235.00,
        "convenienceFee": 10.00,
        "processingFee": 5.00,
        "account": {
          "tpReferenceId": "ACME-CORP-001",
          "name": "Acme Corp",
          "type": 1,
          "contactEmail": "billing@acme.com"
        },
        "contact": {
          "email": "driver@acme.com",
          "firstName": "Jane",
          "lastName": "Doe"
        },
        "vehicle": {
          "licensePlate": "ABC1234",
          "state": "NY",
          "make": "Honda",
          "model": "Civic"
        },
        "subscription": {
          "tpReferenceId": "PERMIT-2026-001",
          "permitNumber": "P001",
          "status": 1,
          "startDate": "2026-04-01T00:00:00Z",
          "currentBaseRate": 250.00,
          "currentRate": 250.00,
          "numberOfSpaces": 1
        }
      }
    ]
  }'
```

A single 200 response will tell you the transaction was created and how many of `accounts`, `contacts`, `vehicles`, `subscriptions` were newly created (probably 1 of each on the first run, 0 of each on a replay).

### Example 2: Nightly transaction batch (Node.js / TypeScript)

```typescript
import axios, { AxiosError } from "axios";

interface TransactionItem {
  transactionId: string;
  locationUuid: string;
  type: 1 | 2 | 3 | 4;
  transactionDate: string; // ISO UTC
  amountTotal: number;
  netAmount: number;
  convenienceFee: number;
  processingFee: number;
  contact?: { email?: string; phoneNumber?: string; firstName?: string; lastName?: string };
  vehicle?: { licensePlate: string; state?: string };
  accountTPReferenceId?: string;
  metadata?: Record<string, unknown>;
}

const client = axios.create({
  baseURL: "https://api.dashmarketing.io",
  headers: {
    "Content-Type": "application/json",
    "X-API-KEY": process.env.DASH_API_KEY!,
  },
  timeout: 30_000,
});

async function submitBatch(transactions: TransactionItem[], runId: string) {
  const { data } = await client.post("/integrations/transactions", {
    transactions,
    metadata: { source: "pms-nightly", runId },
  });

  const failed = data.results.filter((r: { status: number }) => r.status === 4);
  if (failed.length > 0) {
    console.warn(`[${data.referenceUuid}] ${failed.length} records failed`, failed);
  }
  return data;
}

// Chunk into 500-record batches; the API enforces this hard cap.
async function submitAll(items: TransactionItem[]) {
  const BATCH_SIZE = 500;
  const runId = new Date().toISOString();

  for (let i = 0; i < items.length; i += BATCH_SIZE) {
    const chunk = items.slice(i, i + BATCH_SIZE);
    try {
      await submitBatch(chunk, `${runId}-${i / BATCH_SIZE}`);
    } catch (err) {
      const ax = err as AxiosError<{ error?: string }>;
      if (ax.response?.status === 429) {
        // Honor Retry-After
        const retryAfter = Number(ax.response.headers["retry-after"] ?? 60);
        await new Promise((r) => setTimeout(r, retryAfter * 1000));
        i -= BATCH_SIZE; // retry this chunk
        continue;
      }
      throw err; // Surface 4xx/5xx for the caller to alert on
    }
  }
}
```

### Example 3: Retry-safe forwarder with idempotent replay (Python)

```python
import os
import time
import requests
from typing import Iterable

BASE_URL = "https://api.dashmarketing.io"
HEADERS = {
    "Content-Type": "application/json",
    "X-API-KEY": os.environ["DASH_API_KEY"],
}

def submit_transactions(items: list[dict], *, run_id: str) -> dict:
    """Submit a batch with bounded retries on 5xx and 429."""
    payload = {"transactions": items, "metadata": {"runId": run_id}}

    backoff = 1.0
    for attempt in range(5):
        r = requests.post(f"{BASE_URL}/integrations/transactions", json=payload, headers=HEADERS, timeout=30)

        if r.status_code == 429:
            time.sleep(int(r.headers.get("Retry-After", "60")))
            continue
        if 500 <= r.status_code < 600:
            time.sleep(backoff)
            backoff *= 2
            continue

        r.raise_for_status()
        return r.json()

    raise RuntimeError("Exhausted retries submitting transactions")


def chunked(iterable: Iterable[dict], size: int):
    chunk: list[dict] = []
    for item in iterable:
        chunk.append(item)
        if len(chunk) >= size:
            yield chunk
            chunk = []
    if chunk:
        yield chunk


def forward_all(transactions: Iterable[dict], run_id: str) -> None:
    for chunk in chunked(transactions, 500):
        result = submit_transactions(chunk, run_id=run_id)
        # Persist (transactionId -> status, uuid) for reconciliation.
        for r in result["results"]:
            persist_outcome(run_id, r)
```

Because `transactionId` is the dedup key, replaying a chunk after a partial network failure is safe: previously-accepted records come back as `Skipped` (`reasonCode: 1`) and any new records are `Created`.

### Example 4: Bulk account onboarding before transaction backfill

```bash
curl -X POST https://api.dashmarketing.io/integrations/accounts \
  -H "Content-Type: application/json" \
  -H "X-API-KEY: $DASH_API_KEY" \
  -d '{
    "accounts": [
      {
        "tpReferenceId": "ACME-CORP-001",
        "name": "Acme Corp",
        "accountNumber": "ACME-001",
        "type": 1,
        "contactName": "Billing Dept",
        "contactEmail": "billing@acme.com",
        "locationUuid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
      },
      {
        "tpReferenceId": "WIDGETS-LLC-002",
        "name": "Widgets LLC",
        "type": 1,
        "contactEmail": "ap@widgets.example"
      }
    ]
  }'
```

Persist `referenceUuid` and the `results[].uuid` mapping. When you later submit subscriptions or transactions, you can reference accounts by either the original `tpReferenceId` or the returned Dash `uuid`.

---

## Operational Guidance

### Capacity Planning

The hard caps are 500 transactions and 100 accounts/subscriptions per request, with a 60 req/min/org rate limit on writes. This gives an effective ceiling of:

- **30,000 transactions/min** (500 × 60)
- **6,000 accounts/min** or **6,000 subscriptions/min**

If you need higher throughput for a one-time backfill, contact support before kicking off the run.

### Concurrency

The rate limit is per organization, not per API key. Two concurrent jobs sharing the same org-key budget will collectively trip the limit at 60 req/min. Coordinate concurrency with a queue or shared rate-limiting layer on your side, or split work across separate jobs sequentially.

### Observability

- Persist every `referenceUuid` you receive. Ours and yours are the only stable correlators between your system and ours.
- Log full `results` arrays for any batch with `failedCount > 0`. The list endpoint stores up to the **first 100 failures** per run for later diagnosis; anything beyond that is in the immediate response only.
- The `durationMs` field is server-side processing time. If your client-observed latency far exceeds this, the bottleneck is network, not Dash.

### Backfills

For historical backfills:

1. Submit accounts first via `POST /integrations/accounts`.
2. Submit subscriptions next via `POST /integrations/subscriptions`.
3. Submit transactions last via `POST /integrations/transactions`, using `accountTPReferenceId` and `subscriptionTPReferenceId` to wire them up.

Splitting along these lines is faster than mixing nested entities into transactions, because each pre-loaded entity is then resolved by reference (one indexed lookup) instead of re-validated on every transaction.

---

## Troubleshooting

### `LocationNotFound` on a transaction
Verify the `locationUuid` against `GET /integrations/locations`. The most common cause is using your internal location ID instead of the Dash UUID.

### `AccountNotFound` on a subscription
The standalone subscription endpoint does not auto-create accounts. Either submit the account first, or move the subscription into a transaction request where nested account creation is supported.

### A transaction came back `Created` but the account didn't
On `/integrations/transactions`, only the `createdEntities` counts indicate newly created related entities. A `Created` transaction status means *the transaction* was created — its account may have been resolved against an existing record by `tpReferenceId`. This is expected and idempotent.

### Replays return all `Skipped`
That's the design. The dedup key (`transactionId` or `tpReferenceId`) is unique per organization. If you intended to update existing records, see the changelog — update semantics are not yet exposed via this API.

### Sudden 429 errors
Verify both your job concurrency and any other systems sharing the same API key. The rate limit is per organization, not per key — overlapping deploys are a common surprise.

### Batch fails entirely with `"InternalError"` for a single record
Other records in the batch are still committed. Inspect `results[]` to identify which records failed and re-submit only those. The full batch should not be replayed if the rest succeeded.

---

## Support

For technical support, integration assistance, or to request rate limit increases for one-time backfills:

- **Email:** support@dashmarketing.io
- **Documentation:** https://docs.dashmarketing.io

When reporting issues, please include:
1. The `referenceUuid` of the affected import run
2. A representative `transactionId` / `tpReferenceId` from the failure
3. The HTTP status code and full response body
4. Approximate timestamp (UTC) of the request

---

## Changelog

### Version 1.0 (April 2026)
- Initial release
- `POST /integrations/transactions`, `/integrations/accounts`, `/integrations/subscriptions`
- `GET /integrations/locations`, `/integrations/imports`, `/integrations/imports/{referenceUuid}`
- Per-record outcomes with structured reason codes
- Per-organization rate limiting
- Inline upsert of related entities (account, contact, vehicle, subscription) on transaction submit
