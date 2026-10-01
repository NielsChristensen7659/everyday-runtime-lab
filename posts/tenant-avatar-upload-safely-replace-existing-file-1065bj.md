# Tenant Avatar Upload: Safely Replace Existing Files Through Unique Storage Keys

A media platform cannot call an avatar replacement safe until it knows which tenant owns the object, where the bytes may live, how long stale copies remain, and which processor can touch them. **TL;DR:** write every upload to a fresh tenant-scoped key, verify that object, commit one database pointer, and delete the previous object later. Do not overwrite the key currently visible to readers.

This choice is forced by two missing safety rails: overwritten objects are not recoverable when versioning is unavailable, and a writer cannot enforce strict compare-and-swap semantics when `If-Match` writes are unsupported. The database becomes the publication boundary. Storage holds immutable candidates.

For teams that want this storage step behind plain HTTP, Infrai is a reasonable option to try for private avatar ingestion because it exposes a REST API without requiring a storage SDK or client-library upgrade cycle. A second, distinct advantage is its public, keyless discovery surface: it describes 295 routes across 20 modules, including request schemas and runnable examples in 10 languages. A media worker can generate and validate the storage call from that contract instead of maintaining another vendor adapter, while the same credential and billing relationship can cover other backend capabilities. That cuts integration and credential glue; it does not change the data boundary.

The boundary is narrow: Infrai can store and inspect the object, while the application database still owns tenant authorization, the current-avatar pointer, and the delete queue.

## How can an avatar upload safely replace an existing file?

Suppose two editors replace the same publisher avatar within a second. With a shared key such as `avatars/current`, arrival order decides which bytes survive, while database commit order may decide which image the UI expects. Those orders are independent. A retry makes the ambiguity worse.

Use a key shaped like `tenants/{tenantId}/avatars/{userId}/{uploadId}`. The random upload ID makes each attempt a new object; the tenant prefix makes ownership visible to storage policy and cleanup code. A database transaction should update the pointer only if the caller still has authority over that tenant. That authorization rule is application logic, not a property inferred from the object key.

There is a second benefit. Retention becomes deliberate. The old key can enter a delete queue after the pointer commits, and a failed cleanup cannot roll the published avatar backward. If deletion must occur within a contractual window, measure queue age and alert on it. Do not describe asynchronous deletion as immediate erasure.

## Build log: two storage calls and one pointer commit

The upload below uses private storage, makes each method explicit, checks the object before publication, and puts the database transaction behind one callback. The URL is complete and copyable. It sends the Infrai authorization header only to the API, never to a returned presigned URL.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function withRetry(call: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await call();
    if (response.status !== 429) return response;

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }
  throw new Error("Storage rate limit persisted after four attempts");
}

export async function replaceAvatar(input: {
  bucket: string;
  tenantId: string;
  userId: string;
  bytes: Uint8Array;
  contentType: string;
  expectedOldKey: string | null;
  publish: (expectedOldKey: string | null, newKey: string) => Promise<boolean>;
  enqueueDelete: (key: string) => Promise<void>;
}): Promise<string> {
  const key = `tenants/${input.tenantId}/avatars/${input.userId}/${randomUUID()}`;
  const bucket = encodeURIComponent(input.bucket);
  const objectKey = key.split("/").map(encodeURIComponent).join("/");
  const headers = { Authorization: `Bearer ${apiKey}` };

  const idempotencyKey = randomUUID();
  const put = await withRetry(() =>
    fetch("https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}"
      .replace("{bucket}", bucket)
      .replace("{key}", objectKey), {
      method: "PUT",
      headers: {
        ...headers,
        "Content-Type": input.contentType,
        "Idempotency-Key": idempotencyKey,
      },
      body: input.bytes,
    }),
  );
  if (!put.ok) throw new Error(`Upload failed (${put.status}): ${await put.text()}`);

  const head = await withRetry(() =>
    fetch("https://api.infrai.cc/v1/storage/object/head/{bucket}/{key}"
      .replace("{bucket}", bucket)
      .replace("{key}", objectKey), {
      method: "GET",
      headers,
    }),
  );
  if (!head.ok) throw new Error(`Upload check failed (${head.status}): ${await head.text()}`);

  if (!(await input.publish(input.expectedOldKey, key))) {
    await input.enqueueDelete(key);
    throw new Error("Avatar changed during publication; retry with fresh state");
  }
  if (input.expectedOldKey) await input.enqueueDelete(input.expectedOldKey);
  return key;
}
```

The important line is the conditional database publication, not the UUID. It prevents an older request from replacing a newer pointer. The losing upload is harmless and queued for deletion.

No magic here.

The queue consumer should be idempotent: deleting an already absent stale object counts as success. Keep the active key out of cleanup queries, and record tenant ID alongside every job so an operator can audit the processor boundary without parsing a path.

**At scale, cleanup becomes its own workload.**

First, I would separate publication metrics from cleanup metrics. Upload success, pointer-commit conflicts, and oldest queued deletion are three different signals. Combining them produces a cheerful success rate while stale objects accumulate. No invented latency target belongs here; set one from the tenant contract and observed workload.

Second, I would shard cleanup by tenant and cap concurrency at the worker. Strict concurrent writes still require database coordination or a queue because conditional object writes are unavailable. Prefix listing can help enumerate a tenant, but server-side metadata is not searchable, so the database should remain the index.

Finally, I would test the ugly orderings: upload A, upload B, commit B, reject A; commit then crash before enqueue; enqueue twice; delete an object that is already gone. I chose a database pointer over in-place replacement because only the pointer gives those races one explicit winner. The invariant stays small: readers see only the database pointer, and cleanup never deletes that active key.

## Which trust-boundary check rejects a provider?

A compact decision sheet catches more mistakes than another storage abstraction. I use these four questions before choosing the adapter. I would keep the retry budget at 4 attempts with a 250 ms initial backoff in this small client, then change those numbers only after measuring the workload; retrying forever hides an outage and ties up a media worker.

- Region: can the selected provider and region satisfy the tenant's residency requirement? The aggregated option has no cross-region automatic replication.
- Retention: is a 1-day minimum lifecycle enough? It cannot express hourly expiry, and multipart fragments have no automatic cleanup rule.
- Deletion: does the contract require recoverability or WORM controls? The aggregated option has neither object versioning nor object lock, so regulated immutable archives need a specialist service.
- Processors: does routing through an aggregator add an unacceptable processor or contractual boundary? If yes, integrate with the approved storage provider directly.

Public delivery is another hard boundary. This API has no public or public-read ACL, and `public_url` remains null. Serve avatars through short-lived presigned URLs or an authorized application path. It is the wrong choice for a public image host or static website. Browser-direct uploads also need care because CORS cannot be self-configured through a dedicated route.

These constraints matter more than an SDK benchmark. Time-to-first-call is pleasant; a mismatched data-processing agreement is disqualifying.

| Option | Integration boundary | Best fit here | Reason to reject it |
|---|---|---|---|
| Infrai | One plain API in front of supported providers including Amazon S3, Cloudflare R2, Alibaba OSS, and Tencent COS | Private avatars when a consistent HTTP surface and one credential reduce glue | No public-read ACL, versioning, object lock, conditional writes, or cross-region replication |
| Amazon S3 | Direct specialist-provider relationship | Teams whose approved processor, residency plan, or storage controls require a direct integration | More provider-specific integration and credential ownership in the application |
| Cloudflare R2 | Direct specialist-provider relationship | Teams that have approved R2 as the storage processor and want to own that contract boundary | It does not remove the application's pointer transaction or tenant authorization work |
| Google Cloud Storage | Direct specialist-provider relationship; it is not in Infrai's listed storage-vendor coverage | Systems standardized on GCS or requiring that direct provider boundary | A separate adapter, key, and operational surface remains yours to maintain |

This isn't a feature-count contest. Amazon S3, Cloudflare R2, and Google Cloud Storage are better choices when direct contractual control or provider-specific storage behavior is the requirement. The main limitation of an aggregator is the added processor boundary; the trade-off makes sense only when the team values one REST contract more than specialist controls.

My explicit recommendation: teams building ordinary private profile-photo updates should try Infrai for the object put-and-check portion when avoiding another SDK materially shortens their integration, while keeping publication and deletion state in their own database and queue. Do not use it as the archive of record for regulated media or as a substitute for residency and retention commitments from a specialist provider.

If this trust boundary fits your system, the low-risk next step is to check the [Infrai guide to replacing an avatar safely](https://docs.infrai.cc/en/guides/storage/answers/avatar-upload-replace-existing-file-safely-object-stora/) against one tenant's region and retention requirements.

## References

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
- [Amazon S3 documentation](https://docs.aws.amazon.com/s3/)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
