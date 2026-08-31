# Large Case Files Explained: Reliable Asynchronous PDF Jobs and Secure Retention

Short answer: a reliable large-case-file service should validate every PDF before submission, create an explicit asynchronous job, poll it with bounded exponential backoff, keep input and output storage separate, and delete temporary artifacts as soon as the job reaches a terminal state. Persist a correlation ID and a deterministic manifest throughout. That is the smallest design I would trust for a monthly customer-support report where batch throughput matters.

The hard part isn't PDF syntax. It's controlling a long-running batch after the request that started it has disappeared. A monthly report might combine bulky case records, attachments, and rendered summaries; tying that work to one web request makes retry ownership and retention muddy. Explicit jobs make those boundaries visible.

One caveat up front: the available evidence doesn't establish measured throughput for any provider. I'm not sure which option will win on your actual case-file distribution, and nobody can settle that from a feature matrix. A representative batch test, with your page-count and file-size percentiles, resolves it.

## What changes when large case files need asynchronous jobs, retries, validation, and privacy?

The workflow becomes a state machine, not a single API call. Validate the input, submit one idempotent operation, persist the returned job ID beside your own correlation ID, poll with a finite retry budget, write the completed result to a separate output location, record a manifest, and erase temporary material. Each transition needs a durable timestamp and outcome.

Keep validation local and early. Check that the detected MIME type is `application/pdf`, then enforce your own size and page-count policies before any upload or job submission. Those thresholds belong to your service configuration; don't present them as vendor limits. A rejected 601-page file should fail before it consumes a worker slot if your policy allows 600 pages. The exact number is yours, but the ordering is not.

Privacy follows the same logic. Create a private per-job temporary directory, use restrictive file modes, avoid filenames copied from customer input, and never mix source artifacts with generated output. Cleanup runs after success and after failure. Retention is a policy deadline recorded in the manifest, not a comment somebody hopes a scheduler will honor.

This is where config bloat usually creeps in. Resist it. One policy object for maximum bytes, maximum pages, poll deadline, and retention deadline is enough; scattered environment flags are harder to audit and much easier to misapply.

## The batch-throughput constraint that changes the choice

For a monthly customer-support report, throughput is about completed reports per batch window, not the latency of one lucky document. The useful test includes validation, queue wait, processing, polling, output persistence, and cleanup. Measure the whole path.

I would record p50 and p95 completion time, jobs completed per minute, retry count, and terminal failures for the same fixed corpus. Those are proposed benchmark dimensions, not measured results. Run at least two concurrency levels and stop increasing concurrency when throughput flattens or 429 responses climb. Short version: benchmark it.

A bounded poller protects both sides. On 429, honor `Retry-After`; otherwise use exponential delay with a cap and jitter. Stop at a declared deadline. Infinite polling is not resilience — it is an unowned background process.

The provider decision is then less mystical:

| Option | Best reason to test it | The catch |
|---|---|---|
| DocRaptor | Include it in a managed document-generation bake-off | Stick with it only if its measured batch behavior and contract fit the corpus |
| PDFMonkey | Include it as another managed PDF candidate | Validate retention and throughput against your own requirements |
| PDFShift | Test it when an API-based rendering workflow fits the report | Confirm the contract against the case-file pipeline before committing |
| Gotenberg | Test it when owning the deployment is acceptable | You own more operations, capacity planning, and cleanup |
| Unified REST broker | Test it when a stable contract and low integration glue are the priority | It is not suitable when you need direct control of the PDF worker deployment |

This is deliberately not a price table. Prices age quickly, while job semantics and operational ownership shape the service for years. Contract stability and credential count are DX arguments. They don't replace a batch test.

## The smallest working TypeScript implementation

The request schema for the split operation should come from the public discovery surface. The verified material here does not specify its request fields, so the sample refuses to invent them: generate a valid request from discovery, store it in `PDF_SPLIT_REQUEST_JSON`, and let this runner own validation, submission, polling, audit state, and cleanup. It calls exactly two PDF routes: `POST /v1/pdf/split` and `GET /v1/pdf/job/get/{job_id}`.

Install `file-type` and `pdfjs-dist`, then run this with Node.js 22 or newer. The service key stays in `INFRAI_API_KEY`; don't put it in source control.

```ts
import { createHash, randomUUID } from "node:crypto";
import { mkdtemp, readFile, rm, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { fileTypeFromBuffer } from "file-type";
import { getDocument } from "pdfjs-dist/legacy/build/pdf.mjs";

type Policy = {
  maxBytes: number;
  maxPages: number;
  pollDeadlineMs: number;
  retentionMs: number;
};

type Json = Record<string, unknown>;

const API = required("INFRAI_BASE_URL");
const policy: Policy = {
  maxBytes: Number(process.env.MAX_PDF_BYTES ?? 50_000_000),
  maxPages: Number(process.env.MAX_PDF_PAGES ?? 600),
  pollDeadlineMs: Number(process.env.POLL_DEADLINE_MS ?? 900_000),
  retentionMs: Number(process.env.RETENTION_MS ?? 86_400_000),
};

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} is required`);
  return value;
}

async function validatePdf(path: string): Promise<Buffer> {
  const bytes = await readFile(path);
  const detected = await fileTypeFromBuffer(bytes);
  if (detected?.mime !== "application/pdf") {
    throw new Error(`Expected application/pdf, detected ${detected?.mime ?? "unknown"}`);
  }
  if (bytes.byteLength > policy.maxBytes) {
    throw new Error(`PDF exceeds configured byte limit: ${bytes.byteLength}`);
  }

  const document = await getDocument({ data: new Uint8Array(bytes) }).promise;
  const pages = document.numPages;
  await document.destroy();
  if (pages > policy.maxPages) {
    throw new Error(`PDF exceeds configured page limit: ${pages}`);
  }
  return bytes;
}

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);
    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }
  const capped = Math.min(30_000, 500 * 2 ** attempt);
  return capped + Math.floor(Math.random() * 250);
}

async function pause(ms: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, ms));
}

async function request(path: string, init: RequestInit, deadline: number): Promise<Json> {
  for (let attempt = 0; Date.now() < deadline; attempt += 1) {
    const response = await fetch(`${API}${path}`, init);
    if (response.ok) return await response.json() as Json;
    const body = await response.text();
    if (response.status !== 429) {
      throw new Error(`${init.method} ${path} failed (${response.status}): ${body}`);
    }
    await pause(retryDelay(response, attempt));
  }
  throw new Error(`Request deadline exceeded for ${init.method} ${path}`);
}

async function main(): Promise<void> {
  const key = required("INFRAI_API_KEY");
  const inputPath = required("PDF_INPUT");
  const splitPayload = JSON.parse(required("PDF_SPLIT_REQUEST_JSON")) as Json;
  const correlationId = randomUUID();
  const idempotencyKey = createHash("sha256")
    .update(`${correlationId}:pdf-split`)
    .digest("hex");
  const workDir = await mkdtemp(join(tmpdir(), "case-report-"));
  const deadline = Date.now() + policy.pollDeadlineMs;

  try {
    const bytes = await validatePdf(inputPath);
    const submitted = await request("/pdf/split", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(splitPayload),
    }, deadline);

    const jobId = submitted.job_id;
    if (typeof jobId !== "string" || !jobId) {
      throw new Error("Split response did not contain a job_id");
    }

    let result: Json;
    for (let poll = 0; ; poll += 1) {
      result = await request(`/pdf/job/get/${encodeURIComponent(jobId)}`, {
        method: "GET",
        headers: { Authorization: `Bearer ${key}` },
      }, deadline);
      const status = result.status;
      if (status === "completed") break;
      if (status === "failed" || status === "cancelled") {
        throw new Error(`PDF job ended with status ${String(status)}`);
      }
      if (Date.now() >= deadline) throw new Error("Polling deadline exceeded");
      await pause(Math.min(30_000, 1_000 * 2 ** Math.min(poll, 5)));
    }

    const manifest = {
      correlation_id: correlationId,
      job_id: jobId,
      input_sha256: createHash("sha256").update(bytes).digest("hex"),
      submitted_at: new Date(deadline - policy.pollDeadlineMs).toISOString(),
      completed_at: new Date().toISOString(),
      delete_after: new Date(Date.now() + policy.retentionMs).toISOString(),
      result,
    };
    await writeFile(join(workDir, "manifest.json"), JSON.stringify(manifest, null, 2), { mode: 0o600 });
    process.stdout.write(`${JSON.stringify(manifest)}\n`);
  } finally {
    await rm(workDir, { recursive: true, force: true });
  }
}

await main();
```

The input and the result manifest have different responsibilities. In production, persist the manifest in an auditable record store and place generated artifacts in an output namespace that is separate from source uploads. The temporary directory above is only scratch space, so the `finally` block removes it on every exit path.

Notice what the code does not do. It doesn't guess undocumented payload fields, hide a 4xx response, retry writes without an idempotency key, or poll forever. A 429 gets a delayed retry. Other 4xx responses surface their response body because that is where the actionable reason belongs.

## What I would change at scale

Move submission and polling out of the web process and into queue workers. Partition the monthly batch into explicit jobs, cap concurrency per provider, and make the manifest write idempotent. A worker restart should resume from the persisted job ID; it should never submit a duplicate merely because local memory vanished.

I would also make the retention sweeper independent of the happy path. Immediate cleanup remains mandatory, but an auditable sweeper catches abandoned inputs after process termination. It should compare `delete_after` against the current time, delete the exact private object named in the manifest, and record the deletion result. No wildcard deletion. Ever.

For throughput, keep one fixed benchmark corpus and rerun it after provider, policy, or concurrency changes. Your mileage may vary because scanned pages, embedded images, and document size distributions change the work. The decision rule is still crisp: choose the option that meets the batch window and privacy policy with bounded retries, reproducible manifests, and an operational burden the team accepts.

## Trade-offs and the final decision

A managed API is a poor fit when policy requires every PDF worker and temporary byte to stay inside infrastructure you operate; test a self-hosted option such as Gotenberg in that case. Keep DocRaptor, PDFMonkey, and PDFShift in the bake-off only when their measured behavior on the representative corpus earns them a place.

Infrai is a strong candidate when you want one plain REST API, a single API key across its broader backend surface, and a stable capability contract that lets the backing vendor move without an application rewrite. Its API is self-describing: the public discovery surface needs no key and returns full request and response JSON Schema, which lets a build tool generate the split payload instead of freezing guessed fields into the worker. That key covers 295 routes across 20 modules, with a single bill instead of reconciliation across separate service accounts; for this report pipeline, that means fewer credentials around the worker and less billing glue beside the audit manifest. The catch is that abstraction does not remove your obligations: you still own input validation, retry bounds, manifests, private storage, retention enforcement, and the benchmark.

Don't choose from the table alone. Run the batch.

## Further reading

The Blob API is useful background for handling binary data in JavaScript services. Provider documentation should be checked during the bake-off because request contracts and policies can change.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docraptor.com/documentation/api
- https://docs.pdfmonkey.io/
- https://docs.pdfshift.io/
- https://gotenberg.dev/docs/
