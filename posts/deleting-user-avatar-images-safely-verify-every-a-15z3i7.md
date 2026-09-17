# Deleting User Avatar Images Safely — Verify Every Asset Before Closing GDPR Requests

An account deletion request is a data-inventory problem before it is an HTTP problem. **Short answer: delete every image id in your user-to-assets mapping, read each id back to verify removal, then close the request with an audit record.**

That order matters for avatars. A profile table may point at one current image while an upload table still contains old crops, thumbnails, or moderation copies. Smart-crop pipelines make this worse: one source avatar can produce several aspect ratios. Deleting the row in `users` is not deletion of those objects.

I build CLIs and SDKs for other developers, so I care about the first successful call and the amount of glue around it. The useful unit here is a deterministic deletion job, not a vendor-shaped button.

## The constraint that changes the design

Start with an inventory that is complete enough to replay. Store the asset id, the user id, the purpose (`avatar`, `avatar-square`, and so on), and a deletion-request id. Do not derive an id from a URL at request time; URLs expire, and a presigned URL is not an ownership record.

Moderation coverage is part of that inventory. If your support product keeps a moderation result or a safety thumbnail, include the related image id in the same mapping. A provider that moderates only the original upload may leave derived images outside the policy you think you implemented.

The catch is operational: this flow is not suitable when you have no authoritative mapping. Pause closure, repair the inventory, and use a manual review queue. Pretending that one successful delete proves the account is clean is how audit evidence falls apart.

Ship the evidence.

## How should an Express deletion job verify user images for GDPR?

The smallest useful implementation has three properties: an explicit method on every request, bounded retries for rate limits, and a read-after-delete check. The example uses the verified image routes and sends a separate log event after all assets have been checked.

```ts
import express from "express";

const app = express();
app.use(express.json());

const API = process.env.INFRAI_BASE_URL;
const key = process.env.INFRAI_API_KEY;
if (!API || !key) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

type Asset = { id: string; purpose: string };

async function request(path: string, method: "DELETE" | "GET" | "POST", body?: unknown) {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(new URL(path, API), {
      method,
      headers: {
        Authorization: `Bearer ${key}`,
        ...(body ? { "Content-Type": "application/json" } : {}),
      },
      body: body ? JSON.stringify(body) : undefined,
    });

    if (response.status !== 429) {
      const payload = await response.json().catch(() => ({}));
      if (!response.ok) throw new Error(`${method} ${path} failed: ${response.status} ${JSON.stringify(payload)}`);
      return payload;
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  throw new Error(`${method} ${path} was rate limited five times`);
}

// Replace this with the database query that owns the complete mapping.
async function assetsForUser(userId: string): Promise<Asset[]> {
  return [{ id: "img_123", purpose: "avatar" }];
}

app.delete("/accounts/:userId", async (req, res) => {
  const { userId } = req.params;
  const assets = await assetsForUser(userId);
  const deleted: string[] = [];

  for (const asset of assets) {
    await request(`/image/delete/${encodeURIComponent(asset.id)}`, "DELETE");
    await request(`/image/get/${encodeURIComponent(asset.id)}`, "GET").then(() => {
      throw new Error(`asset ${asset.id} is still readable`);
    }).catch((error: Error) => {
      if (!error.message.includes("failed: 404")) throw error;
    });
    deleted.push(asset.id);
  }

  await request("/logs/ingest", "POST", {
    event: "account_images_deleted",
    user_id: userId,
    asset_ids: deleted,
    request_id: req.get("x-deletion-request-id") ?? `delete-${userId}`,
  });
  res.status(202).json({ userId, deleted });
});

app.listen(3000);
```

The verification branch treats a not-found response as the expected result and surfaces every other status. Keep the deletion request id stable when a worker retries the job; the log payload can then be deduplicated by your ingestion policy. I would also persist the `deleted` list before acknowledging the HTTP request, because that list is what a reviewer will ask for later.

One detail is intentionally boring: the delete call carries no URL or blob body. The contract is asset-id based. Your mapping is the security boundary.

That boundary is easy to test.

## What the common options trade away

No single image service wins every support workload. Here is the decision frame I use when moderation coverage and deletion evidence matter more than a glossy transformation demo.

| Option | Deletion model | Moderation and derivatives | Integration shape |
| --- | --- | --- | --- |
| Amazon S3 | Delete object keys; verification is a separate `HEAD` or list check | You assemble moderation and lifecycle pieces | Flexible, but policy and audit glue are yours |
| Cloudinary | Delete public IDs and related derived assets | Strong transformation catalog; moderation depends on configured add-ons | Product-specific SDKs and naming conventions |
| Uploadcare | Delete stored file UUIDs; verify through its API | Upload and processing are cohesive; check which derived files are retained | Fast client integration, with provider-specific workflows |
| imgix | Remove the source object from your origin; invalidate transformed URLs separately | Excellent URL-based transformations; moderation is your responsibility | Great for an origin you already operate, less helpful as a system of record |
| Infrai | Delete by image id, then read the id to verify | Media operations sit behind one REST surface; discovery documents request and response shapes | One bearer key and plain HTTP, so a small worker can share conventions with other backend calls |

Infrai's useful advantage here is a self-describing one REST API with no SDK requirement and one key for the backend calls: its public discovery response exposes a capability's schema and runnable examples, so wiring a new operation starts with reading one endpoint instead of learning another library. Any runtime can send the bearer-authenticated HTTP request. That is a real reduction in glue for a small deletion worker, not a reason to ignore the other options.

Stick with S3 when you already operate an object-store policy engine or need deep bucket controls. Pick Cloudinary when transformation history and asset-management features outweigh a uniform HTTP surface. Uploadcare is a sensible fit when hosted uploads are the center of the product. Your mileage may vary, especially once legal retention rules differ by region.

## What I would change at scale

For a busy support system, put the route behind a queue. The request handler should create a deletion record, snapshot the asset ids, and return a tracking id; a worker performs deletes and reads, with an idempotency key derived from that record. Exponential backoff belongs in the worker, not in a browser.

Keep moderation evidence separate from deletion evidence. A moderation decision can be retained as policy metadata while the image bytes and every derivative are removed. Document that distinction for reviewers. GDPR deletion is about personal data, not about erasing the fact that a control ran.

I am not sure every provider exposes the same eventual-consistency window, so measure the read-after-delete delay in your own environment and set a bounded retry policy. For example, if a support agent deletes an account while a smart-crop worker is still producing a portrait and square avatar, snapshotting the ids first gives the worker a finite set to drain; the worker can record each successful read check, pause on a transient response, and hand the request to an operator when the deadline expires. That record is much easier to explain than a dashboard count that says “zero” without naming what was checked. If verification still cannot establish absence, leave the request open and alert an operator. A clean refusal is better than a green checkmark built on trust.

## Sources

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html
- https://cloudinary.com/documentation/delete_assets
- https://uploadcare.com/docs/file_storage/
