---
name: cost-explorer-query
description: >
  Answer Eon customer questions about cloud-resource consumption — how many
  GB/month or USD/credits an account, project, or department spent on EC2,
  RDS, S3, Azure, or GCP storage, across any time range, with per-month or
  per-resource breakdowns. Use whenever the user asks how much was consumed,
  who is spending the most, top-N resources by usage, or month-over-month
  cost trends.
---

# Cloud Cost & Storage Consumption

You help Eon customers understand their cloud-resource consumption — usage
in GB/month or cost in USD/credits, broken down by cloud, resource type,
account, or individual resource, optionally scoped to a department tag.
The single tool you call is `query_cost_data`. Most of your work is asking
the right clarifying questions, then packing the answer into the request
correctly.

The full request body shape, the response field guide, the Eon credits
formula, the complete list of gotchas, and the "resources added/removed
during the period" approximation recipe all live in
[references/api-details.md](references/api-details.md). Load that file
before you construct the call or interpret the response — it has the
billing math and the wire-format details you need to get right.

## Step 1 — Clarify the question

Make sure four things are clear before calling the tool. Ask the user only
for what you can't infer from their message; compute relative dates ("last
month", "year to date") yourself.

| Need | If user said it | If not, ask |
|------|------------------|--------------|
| **Time range** | "January", "last quarter", "Mar 1 – Apr 30" | "What time range? (e.g. last month, Q1 2026, Jan–Mar)" |
| **Scope** | "EC2", "S3", "all AWS", "everything" | "Which cloud / resource type? (EC2, RDS, S3, all AWS, all clouds…)" |
| **Unit** | "GB/month", "GiB", "dollars", "credits" | Default to **GB-month** for usage questions, **USD** for cost questions. If the customer asked about credits, use credits. Confirm if ambiguous. |
| **Aggregation** | "total", "per month", "per resource", "top 10" | "Do you want a single total, a per-month breakdown, or a per-resource list?" |

**Calendar month vs. billing period.** When the user says "this month",
"last month", "this billing cycle", or "this period," they could mean two
different things and they're not interchangeable in Eon:

- **Calendar month** — Jan 1 – Feb 1 (the typical default).
- **Eon billing period** — runs from the 26th of one month to the 26th of
  the next. The "last billing period" relative to today depends on the
  current date; see [references/api-details.md](references/api-details.md#billing-period-vs-calendar-month).

If the user mentions "billing", "invoice", "bill", or "billing cycle,"
ask which they mean before constructing the time range.

## Step 2 — Resolve the "department"

Eon has no first-class "department" entity. A department is a resource tag
the customer applied — commonly `department`, `cost-center`, `team`,
`business-unit`, `owner`, or `environment`.

1. If the user named the tag (e.g. "Department A is `cost-center=platform`"),
   use it.
2. If the user named only the value, ask: **"Which tag identifies
   departments in your environment? Common ones are `department`,
   `cost-center`, `team`, or `business-unit`."**
3. Construct the filter:
   `filters.tagKeyValues.containsAnyOf: ["<key>=<value>"]` — **joined with
   `=`** (not `:`), case-sensitive, exact match. Example:
   `["cost-center=platform"]`.

If the customer doesn't know their tag convention, run an exploratory
query (no tag filter, `groupBy = "RESOURCE"`, small `pageSize`) and
surface the `dimensions.resourceTags` values you see in the response so
they can pick the right key.

## Step 3 — Pick the request shape

Choose `granularity`, `groupBy`, and `topN` from the customer's question:

| Question shape | `granularity` | `groupBy` | `topN` | Notes |
|----------------|---------------|-----------|--------|-------|
| Single total for a period | `"TOTAL"` | omit (defaults to `"SOURCE_ACCOUNT"`) | omit | Sum `costs[].usage.amount` (or `cost.amount`) across the returned rows yourself. |
| Per-month breakdown | `"MONTHLY"` | `"RESOURCE_TYPE"` (or `"SOURCE_ACCOUNT"`) | omit | One record per (month × group). |
| Split by cloud or account | `"TOTAL"` | `"CLOUD_PROVIDER"` or `"SOURCE_ACCOUNT"` | omit | One record per group. |
| List individual resources | `"TOTAL"` | `"RESOURCE"` | omit | `resourceId`, `resourceName`, `sourceStorageSize` populated on each record. |
| Top N resources by usage / cost | `"TOTAL"` | `"RESOURCE"` | `<N≥1>` | Tail collapses into a synthetic "Others" record with null `dimensions`. Treat it as the residual bucket, not as a real resource. **Never send `topN: 0`** — the schema requires `topN ≥ 1`; omit the field entirely for "no limit". |

The wire enum values are exactly as shown above — `"TOTAL"`, not
`"COST_GRANULARITY_TOTAL"`. The `COST_*` and `USAGE_*` prefixed names are
Go variable names from the proto, not the JSON wire format.

## Step 4 — Filter EC2 / RDS / S3

There is **no** `meteringDimension` filter on the request. To scope to a
specific service, combine `cloudProvider` and `resourceType`. The
canonical OpenAPI enum values:

| Customer asked | `filters.cloudProvider.in` | `filters.resourceType.in` |
|----------------|---------------------------|---------------------------|
| EC2 instances (whole VM) | `["AWS"]` | `["AWS_EC2"]` |
| EBS volumes | `["AWS"]` | `["AWS_EBS_VOLUME"]` |
| RDS | `["AWS"]` | `["AWS_RDS"]` |
| S3 | `["AWS"]` | `["AWS_S3"]` |
| DynamoDB | `["AWS"]` | `["AWS_DYNAMO_DB"]` |
| Azure VMs | `["AZURE"]` | `["AZURE_VIRTUAL_MACHINE"]` |
| Azure Blob storage | `["AZURE"]` | `["AZURE_STORAGE_ACCOUNT"]` |
| Azure SQL DB | `["AZURE"]` | `["AZURE_SQL_DATABASE"]` |
| GCE instances | `["GCP"]` | `["GCP_COMPUTE_ENGINE_INSTANCE"]` |
| GCS buckets | `["GCP"]` | `["GCP_CLOUD_STORAGE_BUCKET"]` |
| Cloud SQL | `["GCP"]` | `["GCP_CLOUD_SQL_INSTANCE"]` |
| BigQuery | `["GCP"]` | `["GCP_BIG_QUERY"]` |

For exotic types not in this table (FSx, EKS Namespaces, MongoDB Atlas,
SAP HANA, etc.) see the full table in
[references/api-details.md](references/api-details.md#resource-type-reference).
If still unsure, run one exploratory query with `groupBy = "RESOURCE_TYPE"`
and no resource-type filter, then read the canonical spellings off the
response.

## Step 5 — Build and call

Read [references/api-details.md](references/api-details.md) for the full
request body shape, pagination details, and the response field guide.
Then call `query_cost_data` with the body you constructed.

For **storage** questions: send `"usageUnit": "BYTE_MONTHS"` (the default)
and sum `records[].costs[].usage.amount` across the relevant rows. Divide
by `1_000_000_000` for **GB-month** (decimal, AWS-style); divide by
`2^30` for **GiB-month** only if the customer explicitly asks for binary.

For **cost** questions: send `"costUnit": "USD"` and sum
`records[].costs[].cost.amount`.

For **credits**: send `"costUnit": "CREDITS"` (the default). Conversion is
`1 credit = $0.01 USD` — i.e. 100 credits = $1. See
[references/api-details.md](references/api-details.md#eon-credits-and-billing-math)
for the per-hour pricing formula if the customer wants to understand how
their bill is computed.

## Critical gotchas (the rest are in references/api-details.md)

1. **`endTime` is exclusive and silently truncated** to the start of the
   current incomplete period. For "all of March 2026" send
   `endTime: "2026-04-01T00:00:00Z"`, not `"2026-03-31T..."`.
2. **`costUnit` defaults to `"CREDITS"`, not USD.** Set `"USD"` explicitly
   whenever the customer asks for dollars.
3. **`"BYTES"` is the average size over the queried period, not the
   current size.** Use `"BYTE_MONTHS"` (the default) for time-weighted
   storage consumption ("how much did Department A consume this month"),
   and `"BYTES"` for average-footprint questions ("what's the typical
   size of my backups"). `sourceStorageSize` on response records is the
   current size — different from both.
4. **Tag filters are case-sensitive, exact-match, joined with `=`.**
   `department=A` and `Department=a` are different filters. Confirm
   casing with the customer if you're not sure.

## When you can't directly answer

**"Which resources were added or removed during the period, and how did
that affect the total?"** — there is no single API for resource lifecycle
deltas. The full approximation recipe (two grouped queries + client-side
diff of `resourceId` sets) is documented in
[references/api-details.md](references/api-details.md#lifecycle-delta--what-changed-during-the-period).
When you surface the result, **state clearly it's an approximation based
on whether a resource had any reported usage in each period, not a
ground-truth audit log of create/delete events.**

**"What does an individual resource cost right now?"** —
`query_cost_data` returns cost history, not real-time pricing. For
point-in-time pricing questions, defer to the cloud provider's pricing
pages, or explain the Eon credit rate for the relevant resource type
(see [references/api-details.md](references/api-details.md#eon-credits-and-billing-math)).
