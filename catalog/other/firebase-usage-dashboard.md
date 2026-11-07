# Firebase / GCP usage dashboard (Cloud Monitoring + Billing Budget)

- **category**: other
- **provider**: Google Cloud (Cloud Monitoring API v3, Cloud Billing Budget API v1, Cloud Logging log-based metrics), reached over plain REST with a service account
- **reusable**: yes. The REST client, free-tier maths, Pacific-day bucketing and cost estimate are app-agnostic. Only the tab's UI and the free-tier numbers are app-specific.
- **docs**: https://cloud.google.com/monitoring/api/v3, https://cloud.google.com/billing/docs/how-to/budget-api-overview, https://cloud.google.com/logging/docs/logs-based-metrics, https://firebase.google.com/docs/firestore/quotas

Use this to give a super-admin an in-app view of how close a Firebase project is to its free-tier limits: Firestore reads/writes/deletes per day, Storage bytes and egress, a 31-day history, a month-to-date cost estimate with a forecast, the real GCP budget amount, and "who is calling Firestore". Neither the Firebase Admin SDK nor the Firebase MCP tools expose any usage or quota data, so Cloud Monitoring is the only source.

## Playbook

### Prerequisites
- A Firebase/GCP project with Firestore and Storage in use.
- The service account already used for the Admin SDK, plus the **Cloud Monitoring API** enabled on the project and `roles/monitoring.viewer` granted to that account.
- For the budget card (optional): the **Cloud Billing Budget API** (`billingbudgets.googleapis.com`) enabled, and **Billing Account Viewer** (`roles/billing.viewer`) granted to the service account **on the billing account**, not the project.
- For the callers card (optional): Firestore **Data Access audit logs** turned on (IAM & Admin → Audit Logs → Cloud Firestore API → Data Read + Data Write) and one log-based metric (below).
- `google-auth-library` (already a transitive dependency of `firebase-admin`; declare it explicitly rather than rely on hoisting). Don't pull in the gRPC `@google-cloud/monitoring` package for a few read-only calls.

### Setup steps
1. Enable the APIs: `gcloud services enable monitoring.googleapis.com billingbudgets.googleapis.com --project=<PROJECT_ID>`.
2. Grant the roles: `roles/monitoring.viewer` on the project; `roles/billing.viewer` on the billing account (`gcloud billing accounts add-iam-policy-binding <BILLING_ACCOUNT_ID> --member=serviceAccount:<SA_EMAIL> --role=roles/billing.viewer`). Find the account with `gcloud billing projects describe <PROJECT_ID>`.
3. Create a budget in Console → Billing → Budgets & alerts, scoped to the project. The API returns only its amount and alert thresholds, never actual spend.
4. Add the thin REST client and the domain layer (below). Put each section behind its **own** admin-only route so one failing Google API blanks one card, not the whole tab.
5. Memoize each route in process for 1–10 minutes and accept `?refresh=1` to bypass it. There is no mutation to invalidate on, and freshness is the point, so use a plain in-memory memo rather than your framework's tagged cache.
6. (Callers card) create the log-based metric, then read it back through the same Monitoring client.

### Core pattern

One auth helper shared by every Google REST client:

```ts
// gcp-auth.ts
import { GoogleAuth } from "google-auth-library";

const SCOPES = [
  "https://www.googleapis.com/auth/monitoring.read",
  // The Budget API rejects cloud-billing.readonly. Access is still read-only
  // because the service account only holds Billing Account Viewer.
  "https://www.googleapis.com/auth/cloud-billing",
];

export class GcpApiError extends Error {
  constructor(message: string, public status: number, public googleReason?: string) { super(message); }
}

let auth: GoogleAuth | null = null;
export async function gcpGetJson<T>(url: string, label: string): Promise<T> {
  auth ??= new GoogleAuth({
    credentials: {
      client_email: process.env.FIREBASE_CLIENT_EMAIL,
      private_key: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, "\n"),
    },
    scopes: SCOPES,
  });
  const { token } = await (await auth.getClient()).getAccessToken();
  if (!token) throw new GcpApiError("Failed to mint an access token", 500);
  const res = await fetch(url, { headers: { Authorization: `Bearer ${token}` } });
  if (!res.ok) {
    const body = await res.json().catch(() => null);
    throw new GcpApiError(`${label} request failed (${res.status})`, res.status, body?.error?.message ?? body?.error?.status);
  }
  return res.json() as Promise<T>;
}
```

`timeSeries.list` with pagination, then two reducers (sum for counters, latest-point for gauges):

```ts
// monitoring-client.ts
export async function fetchTimeSeries(q: {
  metricType: string; resourceType?: string; extraFilters?: string[];
  startTime: Date; endTime: Date; alignmentPeriodSeconds: number;
  perSeriesAligner: string; crossSeriesReducer?: string; groupByFields?: string[];
}) {
  const filter = [`metric.type="${q.metricType}"`,
    ...(q.resourceType ? [`resource.type="${q.resourceType}"`] : []),
    ...(q.extraFilters ?? [])].join(" AND ");
  const out: { labels: Record<string, string>; points: { endTime: string; value: number }[] }[] = [];
  let pageToken: string | undefined;
  do {
    const p = new URLSearchParams({
      filter,
      "interval.startTime": q.startTime.toISOString(),
      "interval.endTime": q.endTime.toISOString(),
      "aggregation.alignmentPeriod": `${q.alignmentPeriodSeconds}s`,
      "aggregation.perSeriesAligner": q.perSeriesAligner,
    });
    if (q.crossSeriesReducer) p.set("aggregation.crossSeriesReducer", q.crossSeriesReducer);
    for (const f of q.groupByFields ?? []) p.append("aggregation.groupByFields", f);
    if (pageToken) p.set("pageToken", pageToken);
    const page: any = await gcpGetJson(
      `https://monitoring.googleapis.com/v3/projects/${PROJECT_ID}/timeSeries?${p}`, "Cloud Monitoring");
    for (const s of page.timeSeries ?? []) {
      out.push({
        labels: { ...s.resource?.labels, ...s.metric?.labels },
        points: (s.points ?? []).map((pt: any) => ({
          endTime: pt.interval?.endTime ?? "",
          value: pt.value?.doubleValue ?? Number(pt.value?.int64Value ?? 0),
        })),
      });
    }
    pageToken = page.nextPageToken || undefined;
  } while (pageToken);
  return out;
}
export const sumSeries = (s: typeof out) => s.reduce((t, x) => t + x.points.reduce((a, p) => a + p.value, 0), 0);
// latestGaugeValue: per series take the point with the greatest endTime, then sum across series.
```

Today's tiles: one query per metric, **one alignment period spanning the whole interval** (day start → now), `ALIGN_SUM`:

| Tile | `metric.type` | `resource.type` | Aligner |
|---|---|---|---|
| Firestore reads / writes / deletes | `firestore.googleapis.com/document/{read,write,delete}_ops_count` | `firestore.googleapis.com/Database` (+ `resource.label.database_id="(default)"`) | `ALIGN_SUM` |
| Storage stored bytes | `storage.googleapis.com/storage/total_bytes` | `gcs_bucket` (+ `bucket_name`) | `ALIGN_MEAN`, latest point of the last hour |
| Storage egress this month | `storage.googleapis.com/network/sent_bytes_count` | `gcs_bucket` | `ALIGN_SUM`, month start → now |

Free-tier constants live in a dependency-free module so client components can import them without pulling the server clients into the browser bundle:

```ts
export const FIRESTORE_FREE_TIER = { readsPerDay: 50_000, writesPerDay: 20_000, deletesPerDay: 20_000 };
export const STORAGE_FREE_TIER = { storedBytes: 5 * 1024 ** 3, egressBytesPerMonth: 100 * 1024 ** 3 };
const pct = (v: number, limit: number) => Math.min(100, Math.round((v / limit) * 100));
```

Cost estimate (pure function, testable): usage above the free tier times list price, with Firestore costed **per Pacific day** (its free tier is daily) and Storage/egress costed **per month** (project the usage first, then cost it, otherwise you miss the point where usage crosses the monthly free tier):

```ts
const billable = (used: number, free: number) => Math.max(0, used - free);
export function firestoreDayCost(d: { reads: number; writes: number; deletes: number }) {
  return (billable(d.reads, FREE.readsPerDay) / 1e5) * PRICE.readPer100k
       + (billable(d.writes, FREE.writesPerDay) / 1e5) * PRICE.writePer100k
       + (billable(d.deletes, FREE.deletesPerDay) / 1e5) * PRICE.deletePer100k;
}
// Month-to-date = sum of the month's days; forecast = daysInMonth / daysElapsed scaling.
```

Budget read (amount and thresholds only):

```ts
// GET https://billingbudgets.googleapis.com/v1/billingAccounts/<ACCOUNT>/budgets
// amount.specifiedAmount = { currencyCode, units, nanos }  → units + nanos / 1e9
// thresholdRules[].thresholdPercent → [0.5, 0.9, 1]
// Prefer a budget scoped to exactly this project (budgetFilter.projects contains "projects/<NUMBER>"),
// then one that includes it, then an account-wide one, unless an explicit budget id is configured.
```

"Who is calling Firestore": a user-defined **log-based counter** over the Data Access audit logs, so a single Monitoring query returns per-caller counts and needs no Logging IAM (`monitoring.viewer` is enough):

```yaml
# metric.yaml — gcloud logging metrics create firestore_calls_by_caller --project=<PROJECT_ID> --config-from-file=metric.yaml
filter: protoPayload.serviceName="firestore.googleapis.com" AND logName:"data_access"
metricDescriptor:
  metricKind: DELTA
  valueType: INT64
  labels: [{key: principal}, {key: ip}, {key: ua}, {key: method}]
labelExtractors:
  principal: EXTRACT(protoPayload.authenticationInfo.principalEmail)
  ip: EXTRACT(protoPayload.requestMetadata.callerIp)
  ua: REGEXP_EXTRACT(protoPayload.requestMetadata.callerSuppliedUserAgent, "^(\\S+)")
  method: EXTRACT(protoPayload.methodName)
```

Read it with `metricType = "logging.googleapis.com/user/firestore_calls_by_caller"`, `groupByFields = ["metric.label.principal","metric.label.ip","metric.label.ua","metric.label.method"]`, once for today and once for 7 days. Group rows by `ip` (top ~25) with the principal/UA/method breakdown expandable underneath, and let the admin name an IP inline (store `{ip: label}` in a config doc).

### Env vars
- `FIREBASE_PROJECT_ID`, `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` (the existing Admin SDK credentials are reused)
- Storage bucket name (to scope the Storage metrics)
- Optional: `GCP_BILLING_ACCOUNT_ID`, `GCP_BUDGET_ID` (pin a budget), `GCP_COST_FX_RATE` (budget currency per 1 USD when the billing account is not in USD)
- Optional fallback for naming callers: a `ip=label,ip=label` list

### Gotchas
- **Use the current metric names.** `document/{read,write,delete}_ops_count` on resource `firestore.googleapis.com/Database`. Older examples use `read_count`/`write_count`, which return nothing.
- **There is no Firestore stored-bytes metric.** Show that tile as "unavailable" rather than inventing a number.
- **Bucket the day yourself.** Monitoring's 86400s alignment buckets are anchored to the query end time, not midnight. Fetch hourly points and re-bucket them onto **Pacific** midnights in code, because Firestore's free-tier day resets at Pacific midnight. Monitoring keeps roughly 6 weeks, which caps any history range.
- **Storage's free tier is regional.** The 5 GB / 100 GB-egress allowance applies only to buckets in `us-central1`, `us-west1` or `us-east1`. Check the bucket's location before showing a percentage.
- **The estimate is not the invoice.** The Budget API never returns actual spend; exact spend needs a Cloud Billing export to BigQuery. Label the number "estimate" in the UI. Exclude what you can't measure (Firestore stored bytes, Cloud Logging ingestion, hosting) and say so.
- **Re-check prices.** Read list prices from the Cloud Billing Catalog API (`cloudbilling.googleapis.com/v1/services/<SERVICE_ID>/skus`) and stamp the date next to the constants. Firestore SKUs are per region.
- **The Budget API rejects `cloud-billing.readonly`.** Request the `cloud-billing` scope. Access stays read-only because the account only holds Billing Account Viewer.
- **The callers metric counts RPCs, not documents.** One `RunQuery` returning 2,000 docs counts as 1. It only counts from the metric's creation onward, and audit logging must be on first. Local dev, scripts, serverless functions and any MCP server usually share one service account, so the caller **IP** is the real discriminator. Name known IPs in the table.
- **Aggregate metrics carry no caller identity.** A read spike shows that something read heavily but not what. Turn on Data Access audit logs **before** you need them, since they can't attribute an incident after the fact.
- **One route per card.** Usage, history, cost, callers and IP labels each fail independently (a missing role on one API shouldn't blank the others). Surface Google's own error string (`googleReason`) so `SERVICE_DISABLED` / `PERMISSION_DENIED` is obvious.
- **Super-admin only.** The numbers hint at traffic and cost. Gate every route with the uncached admin check, and keep the free-tier constants in a client-safe module.

### Playbook confidence: high
Extracted from a real, locally available implementation; the metric names, scopes and bucketing quirks were confirmed against the live APIs. Prices and free-tier numbers are a dated snapshot: re-check them.

## Adoption
Used in **1** repo in this marketplace. It reads usage through plain REST with the Admin SDK's service account rather than a Firebase SDK, and splits the tab into independently loaded sections so one failing Google API only blanks its own card.
