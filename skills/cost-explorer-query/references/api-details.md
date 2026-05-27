# `query_cost_data` — Full API Details

Reference material for constructing the request body, interpreting the
response, doing the billing math, and handling edge cases. Load this when
you're actually building a request or interpreting results — not just
deciding the question shape.

## Full request body

```jsonc
{
  "timeFrame": {
    "startTime": "2026-01-01T00:00:00Z",   // ISO-8601 UTC; "Z" suffix required
    "endTime":   "2026-04-01T00:00:00Z"    // EXCLUSIVE upper bound (see gotchas)
  },
  "granularity": "MONTHLY",        // HOURLY | DAILY | MONTHLY | TOTAL ; default MONTHLY
  "groupBy":     "RESOURCE",       // SOURCE_ACCOUNT | CLOUD_PROVIDER | RESOURCE_TYPE | RESOURCE | RESOURCE_AND_VAULT ; default SOURCE_ACCOUNT
  "usageUnit":   "BYTE_MONTHS",    // BYTE_MONTHS | BYTES ; default BYTE_MONTHS
  "costUnit":    "USD",            // CREDITS | USD ; default CREDITS — set USD explicitly for dollar answers
  "topN":        10,               // optional; minimum 1; omit entirely for no limit (do NOT send 0)
  "filters": {
    "cloudProvider":           { "in": ["AWS"],            "notIn": [] },
    "resourceType":            { "in": ["AWS_EBS_VOLUME"], "notIn": [] },
    "sourceAccountProviderId": { "in": [],                 "notIn": [] },
    "resourceId":              { "in": [],                 "notIn": [] },
    "tagKeys":                 { "containsAllOf": [], "containsAnyOf": [], "containsNoneOf": [] },
    "tagKeyValues":            { "containsAllOf": [], "containsAnyOf": ["cost-center=platform"], "containsNoneOf": [] }
  }
}
```

**Wire-format note.** The enum values are short strings like `"MONTHLY"`,
`"RESOURCE"`, `"BYTE_MONTHS"`, `"USD"`. The longer `COST_GRANULARITY_*` /
`COST_GROUP_BY_*` / `USAGE_UNIT_*` / `COST_UNIT_*` names are Go variable
names from the proto, **not** the JSON wire format — sending those will
fail schema validation.

**Tag filter format**: each entry is `"<key>=<value>"` joined with an
equals sign, e.g. `"cost-center=platform"`. Case-sensitive, exact match.

Every filter is optional. Omit a filter entirely if you're not using it —
don't send empty arrays just to be polite.

### Pagination

Pagination uses **query parameters**, not body fields:

- `?pageSize=100` — max records per response page
- `?pageToken=<opaque>` — cursor from the previous response's `nextToken`

If you need to walk all pages, pass the previous response's `nextToken` as
`pageToken` on the next call. An empty `nextToken` means there are no more
pages. To keep results consistent across pages, hold `timeFrame`,
`granularity`, `groupBy`, and `filters` constant between requests.

## Response shape

```jsonc
{
  "records": [
    {
      "recordTimeFrame": { "startTime": "...", "endTime": "..." },
      "dimensions": {
        // Which fields are populated depends on groupBy.
        "resourceId":         "uuid",
        "resourceName":       "prod-db-01",
        "resourceType":       "AWS_EBS_VOLUME",
        "cloudProvider":      "AWS",
        "sourceAccountId":    "uuid",
        "sourceAccountName":  "prod-platform",
        "sourceAccountProviderId": "123456789012",
        "providerResourceId": "vol-0abc...",
        "resourceSourceRegion": "us-east-1",
        "sourceStorageSize":  53687091200,   // current size in BYTES — see "Three sizes" below
        "vaultId":            "uuid",
        "resourceTags":       { "department": "A", "env": "prod" },
        "sourceAccountTags":  { "owner": "platform" }
      },
      "costs": [
        {
          "meteringDimension": "AWS_EC2_METERING_DIMENSION",
          "cost":  { "amount": 4.00,  "unit": "USD" },          // or "CREDITS"
          "usage": { "amount": 1.2e11, "unit": "BYTE_MONTHS" }, // or "BYTES"
          "metadata": { ... }   // present only for data-transfer rows
        }
      ],
      "resourceCount": 1
    }
  ],
  "totalCount":           47,    // total records in the full result set (for pagination)
  "totalUniqueResources": 250,   // total unique resources across the time range, BEFORE filters
  "resourceCount":        47,    // unique resources matching the filters
  "nextToken":            "..."  // empty when no more pages
}
```

### `BYTE_MONTHS` vs `BYTES` — they answer different questions

This is the trickiest part of the API. The two `usageUnit` modes return
fundamentally different things from the same hourly raw data:

| `usageUnit` | What `usage.amount` means | Use it for |
|---|---|---|
| `"BYTE_MONTHS"` (default) | **Time-weighted consumption.** Raw hourly bytes are divided by hours-in-month (~730) to produce a pro-rated GB-month value; those are summed across the queried period. | "How much storage did Department A consume in Q1?" — the kind of question billing answers. |
| `"BYTES"` | **Average size over the period.** Raw hourly byte values are summed and then divided by the hour-count to produce a true average footprint. | "What's the average size of my backups over the last month?" — typical-footprint questions, **not** consumption questions. |

**Worked example** — same resource (100 GB EC2 backup, stored for all of
March):

- With `"BYTE_MONTHS"`: `usage.amount = 107_374_182_400` → ÷ `2^30` = 100
  GiB-months. Cost = 400 credits = $4.00.
- With `"BYTES"`: `usage.amount = 107_374_182_400` → ÷ `2^30` = 100 GiB
  average size.

If the resource only existed for half the month, `BYTE_MONTHS` would
return ~50 GiB-months (half the consumption) while `BYTES` would still
return ~100 GiB (the average size while it was there). When in doubt,
use `BYTE_MONTHS` — it matches what the customer is billed on.

### Three sizes — don't confuse them

| Field | What it is |
|-------|-----------|
| `costs[].usage.amount` with `BYTE_MONTHS` | Time-weighted consumption. The number you sum to answer GB-month questions. |
| `costs[].usage.amount` with `BYTES` | Average size over the queried period. Not consumption; not current. |
| `dimensions.sourceStorageSize` | The resource's **current** size in bytes, attached to the row for context. Don't use this to answer time-range consumption or average-size questions. |

### Converting byte-month to GB-month

Cloud providers conventionally use **decimal GB** in cost docs:

- **GB-month (decimal, AWS-style)** = `usage.amount / 1_000_000_000`
- **GiB-month (binary)** = `usage.amount / 2^30` (≈ `/ 1_073_741_824`)

Use decimal unless the user explicitly asks for GiB / binary.

## Eon credits and billing math

The Eon billing system uses **credits**, not raw cloud-provider dollars.

- **Unit conversion**: `1 credit = $0.01 USD` (so 100 credits = $1).
- **Cost rows**: when `costUnit = "CREDITS"`, `cost.amount` is in credits;
  when `costUnit = "USD"`, it's already dollars. The conversion is done
  server-side — you don't multiply yourself.
- **Per-hour formula** (per resource, computed hourly):
  ```
  credits/hour = rate × backupStorageGB / hoursInMonth
  ```
  where:
  - `rate` = credits per GB-month, varies by resource type and vault
    cloud/region (e.g. EC2 backed up to an AWS vault ≈ 4 credits/GB-mo).
  - `backupStorageGB` = total GB currently stored in the vault for this
    resource at the sample time.
  - `hoursInMonth` ≈ 730 (the exact value is per-month — the server uses
    the actual hours in the calendar month being billed).

So a 100 GiB EC2 backup in an AWS vault at the 4 credits/GB-mo rate
accrues `4 × 100 / 730 = 0.548 credits/hour`, which over a 730-hour month
sums to 400 credits = $4.00.

If the customer asks "why is my bill X?", you can:
1. Run a per-resource query (`groupBy = "RESOURCE"`) over the period.
2. Show `usage.amount` (consumption in GB-month) and `cost.amount` per
   resource.
3. Explain the rate × consumption math using the formula above.

### Billing period vs calendar month

Eon billing cycles run from the **26th of one month to the 26th of the
next** (e.g. March's billing period = `2026-02-26T00:00:00Z` to
`2026-03-26T00:00:00Z`). They do **not** align with calendar months.

When a customer says "this month's bill" or "last billing period":
- If they're asking about the **calendar month**, use Jan 1 – Feb 1, etc.
- If they're asking about the **invoiced amount**, use the 26th-to-26th
  window.

These will produce different numbers — sometimes meaningfully different,
since the back-half of one calendar month and the front-half of the next
are in the same billing period. Always confirm which one they mean
before running the query, then state in the answer which window you used.

## Complete gotchas list

1. **`endTime` is exclusive and silently truncated to the current period
   boundary.** The handler snaps the upper bound to the start of the
   current incomplete period (month/day/hour) so you never see a
   half-complete row. For "all of March 2026" pass
   `endTime: "2026-04-01T00:00:00Z"`, not `"2026-03-31T23:59:59Z"`.
2. **Window minima and maxima.** Daily/monthly granularity requires
   ≥ 24 h between `startTime` and `endTime`. Hourly requires ≥ 1 h and is
   capped at 14 days. Requests outside these bounds return HTTP 400.
3. **`costUnit` defaults to `"CREDITS"`.** Always set `"USD"` explicitly
   when the customer asks for dollar amounts — don't assume the default
   matches their mental model.
4. **`usageUnit` defaults to `"BYTE_MONTHS"`.** That's the consumption
   metric used by billing. Use `"BYTES"` only for average-footprint
   questions (see the BYTE_MONTHS vs BYTES section above) — never for
   "how much was consumed."
5. **`sourceStorageSize` is the resource's current size, not its
   consumption or average size.** It's there for context. Use
   `costs[].usage.amount` for time-range questions.
6. **No `meteringDimension` filter on the request.** If the user wants a
   metering sub-type (e.g. `AWS_EC2_NATIVE_METERING_DIMENSION` for
   native snapshots vs. backup-vault EC2), filter `costs[]` rows in the
   response by `meteringDimension` after the call.
7. **Tag filters are case-sensitive, exact-match, joined with `=`.**
   `"department=A"` and `"Department=a"` are different filters. Format
   is `"<key>=<value>"`, not `"<key>:<value>"`.
8. **No `projectIds` needed in the request body.** The gateway resolves
   the caller's projects from the auth context — both for REST and MCP
   calls. Don't add `projectIds` to the body even if you see it in the
   proto definition; the public REST schema doesn't expose it (it's
   `x-internal`).
9. **Pagination lives in query params, not the body.** Use `pageSize` and
   `pageToken` on the URL. The previous response's `nextToken` is the
   value to send.
10. **`topN` minimum is 1.** The OpenAPI schema enforces
    `topN >= 1`. To return all resources unlimited, **omit `topN`
    entirely** — do not send `topN: 0`.
11. **No `"NONE"` value for `groupBy` at REST.** The proto has a
    `COST_GROUP_BY_NONE` value, but the public OpenAPI enum doesn't
    expose it. For a single grand total, use `granularity = "TOTAL"`
    with `groupBy` omitted (defaults to `"SOURCE_ACCOUNT"`) and sum the
    rows yourself, or use any other groupBy and sum.

## Resource type reference

The canonical OpenAPI `resourceType` enum values, with display names you
can show to the customer:

| OpenAPI value | Display name | Cloud |
|---------------|--------------|-------|
| `AWS_EC2` | EC2 | AWS |
| `AWS_EBS_VOLUME` | EBS Volume | AWS |
| `AWS_RDS` | RDS | AWS |
| `AWS_S3` | S3 | AWS |
| `AWS_EFS` | EFS | AWS |
| `AWS_FSX` | FSx | AWS |
| `AWS_DYNAMO_DB` | DynamoDB | AWS |
| `AWS_EKS_NAMESPACE` | EKS Namespace | AWS |
| `AWS_KEYSPACES_TABLE` | Keyspaces Table | AWS |
| `AWS_REDSHIFT` | Redshift | AWS |
| `AWS_DOCUMENTDB` | DocumentDB | AWS |
| `AWS_NEPTUNE` | Neptune | AWS |
| `AZURE_VIRTUAL_MACHINE` | Virtual Machine | Azure |
| `AZURE_DISK` | Disk | Azure |
| `AZURE_FILE_SHARE` | File Share | Azure |
| `AZURE_STORAGE_ACCOUNT` | Blob Storage | Azure |
| `AZURE_SQL_DATABASE` | SQL Database | Azure |
| `AZURE_SQL_MANAGED_INSTANCE` | SQL Managed Instance | Azure |
| `AZURE_SQL_VIRTUAL_MACHINE` | SQL Virtual Machine | Azure |
| `AZURE_SAP_HANA_VM` | SAP Hana Instance | Azure |
| `AZURE_MYSQL` | MySQL Flexible Server | Azure |
| `AZURE_POSTGRESQL` | PostgreSQL Flexible Server | Azure |
| `AZURE_COSMOSDB_MONGODB` | Cosmos DB for MongoDB | Azure |
| `AZURE_COSMOSDB_NOSQL` | Cosmos DB for NoSql | Azure |
| `AZURE_AKS_NAMESPACE` | AKS Namespace | Azure |
| `GCP_COMPUTE_ENGINE_INSTANCE` | GCE Instance | GCP |
| `GCP_DISK` | GCE Disk | GCP |
| `GCP_CLOUD_SQL_INSTANCE` | Cloud SQL | GCP |
| `GCP_CLOUD_STORAGE_BUCKET` | GCS Bucket | GCP |
| `GCP_GKE_NAMESPACE` | GKE Namespace | GCP |
| `GCP_BIG_QUERY` | BigQuery | GCP |
| `GCP_CLOUD_FIRESTORE` | FirestoreDB | GCP |
| `GCP_SAP_HANA_VM` | SAP Hana Instance | GCP |
| `ATLAS_MONGODB_CLUSTER` | Atlas Cluster | MongoDB Atlas |
| `GOOGLE_WORKSPACE_RESOURCE` | Google Workspace | Google Workspace |

If a customer asks for a resource type not in this table, fall back to
running an exploratory query with `groupBy = "RESOURCE_TYPE"` and read
the spelling off the response — the canonical mapping lives in
`shared/contracts/resourcetype/resourcetype.go` (`displayMetadata` map)
and may grow over time.

## Lifecycle delta — "what changed during the period?"

There is no direct API for "resources added or removed during the period
and their consumption impact." The cost-explorer reports usage attributed
to resources that existed; it doesn't track create/delete events.
Approximate the answer this way:

1. Call `query_cost_data` twice with `groupBy = "RESOURCE"` — once over
   the **earlier** sub-period, once over the **later** sub-period. Use
   identical filters in both calls.
2. Collect the `dimensions.resourceId` values from each response into two
   sets, `earlier_ids` and `later_ids`.
3. Compute the diff:
   - **Added** (appeared in the later period) = `later_ids - earlier_ids`
   - **Removed** (disappeared after the earlier period) = `earlier_ids - later_ids`
   - **Persistent** = `earlier_ids ∩ later_ids`
4. For each added/removed resource, attribute its `costs[].usage.amount`
   (from the period where it appeared) as the rough impact.

**State the limitation clearly when surfacing the result:** "I derived
this by comparing which resources had any reported usage in each period.
It's an approximation, not an audit of create/delete events — a resource
that existed but had zero usage in one period would look the same as one
that didn't exist at all."

If the customer needs ground-truth lifecycle data, point them to the
inventory or audit-log surfaces (separate APIs), not cost-explorer.
