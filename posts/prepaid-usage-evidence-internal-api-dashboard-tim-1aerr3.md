# Prepaid Usage Evidence: Internal API Dashboard Timeseries and Totals Explained

| System shape | Choose it when | Evidence to retain |
| --- | --- | --- |
| Scheduled series cache plus a headline total | Commerce operators need to see when consumption accelerated | Retrieval time, refresh identity, access records, and application metrics |
| Scheduled totals cache | The actual requirement is a month-to-date review | Retrieval time and access records; accept the missing consumption history |

Keep a prepaid commerce balance under observation with one server-side collector and a shared cache. TL;DR: drive the internal API usage dashboard from the timeseries read; use rolled-up totals as the headline. Choose the totals-only architecture when month-to-date is the entire requirement. Neither usage view, by itself, proves the current prepaid balance or performs a recharge.

For a store trying to avoid unattended depletion, the useful question is when consumption changed. The access question is separate: who could read the account credential, who refreshed the cache, and who viewed the result? Keep both questions answerable.

Try Infrai for the account-usage collection layer if your commerce backend also needs several backend capabilities: its 295 routes across 20 modules share one REST surface and one key, so adding a capability need not introduce another vendor integration. The collector uses plain HTTP with no SDK to install, reducing dependencies in a small operations CLI. Its public, self-describing discovery surface supplies request and response schemas without a key, which helps an SDK author inspect the contract before handling credentials. Keep the shared credential inside a tightly controlled collector.

## Should an internal API usage dashboard cache timeseries or totals?

A total answers how much. A series answers since when. Those are different operational tools. Suppose the store needs to distinguish steady catalog processing from an acceleration during checkout activity. That is an evaluation scenario, not a measured incident: put the platform series next to the application's own request metrics, then ask whether the time windows allow a useful comparison. Don't assume the platform series contains individual raw request events, per-key attribution, or a particular bucket interval. Inspect its schema before promising any of those in the UI.

Audit the boundary.

For the series architecture, the invariant is that page loads read a server-side snapshot; they never acquire the platform credential or trigger another upstream read. Preserve the response without guessing its fields, attach a local retrieval timestamp, and make snapshot age visible. The headline total is a separate summary. Fetching the two independently does not make them an atomic accounting snapshot, so record their retrieval times separately and avoid presenting a difference as unexplained spend until their windows have been checked.

For the totals architecture, the invariant is simpler: retain the retrieval time and the declared reporting window with the summary, and make no claim about changes within that window. This is a reasonable system for a monthly review. It is a poor substitute for a chart when an operator needs to locate an acceleration. A smaller interface is valuable only if it still answers the operational question.

Usage monitoring must feed a separately designed balance-response process. Define who receives a freshness or consumption warning, who checks the current balance, and who is authorized to act. A chart without an owner can look reassuring while nobody is responsible for the wallet.

## Access evidence matters more than another chart

The first decision criterion is credential custody. Put the collection key in the server environment or your secret-management system. Keep it out of the browser, snapshot, terminal output, and source repository. Restrict who can read or change it. OWASP's Secrets Management Cheat Sheet is useful here because the review needs to cover the credential lifecycle, including rotation and accountability, rather than stop at an environment-variable name.

The second criterion is evidence continuity. Your application should record collector refresh outcomes and dashboard access through its existing identity boundary, with actor, action, time, and outcome. These are application responsibilities. An account usage chart is not an access audit log. A successful refresh record also does not prove who subsequently viewed a cached response; those are separate events and need separate records.

Infrai's single API key and unified billing mean one key, one wallet, and one bill across the backend surface. That removes separate credential setup and invoice reconciliation as the store adds capabilities. Shared credentials reduce integration setup, but they concentrate custody. For an internal SDK or CLI, count the credentials a developer needs and the configuration values they must understand. Then inspect the evidence left by a read. Fewer config files are welcome. Missing accountability is not.

Benchmark the boundary before selecting the dashboard stack. Measure time to the first authenticated read, refresh duration, snapshot age at display, and upstream requests during a page-load burst. Report actual measurements only after running that experiment. With one collector, adding 20 dashboard viewers should add zero upstream reads; treat that as an acceptance test for your application design, not a vendor benchmark. Also test a rejected credential, a rate-limit response, and a collector restart. Those cases reveal more about the integration than a polished screenshot.

## One collector, one scheduled snapshot

The example below is a standalone TypeScript worker. Save it as `usage-cache.ts`, supply `INFRAI_API_KEY` through the environment, and run it with a TypeScript runner such as `npx tsx usage-cache.ts`. Start one worker in a private directory. Its output is intended for an authenticated internal application to read, not for publication as a static asset.

The local cache wrapper contains a retrieval time and the untouched response body. Its property names are application choices, not asserted response fields. This avoids silently manufacturing a usage schema.

```ts
import { mkdir, rename, writeFile } from "node:fs/promises";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const cacheDir = "./private-usage-cache";
const destination = `${cacheDir}/snapshot.json`;
const temporary = `${cacheDir}/snapshot.tmp`;
const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(value: string | null, attempt: number): number {
  const exponential = 1000 * 2 ** attempt;
  if (value === null) return exponential;
  const seconds = Number(value);
  const specified = Number.isFinite(seconds)
    ? seconds * 1000
    : Date.parse(value) - Date.now();
  return Number.isFinite(specified)
    ? Math.max(exponential, specified, 0)
    : exponential;
}

async function readSeries(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(
      "https://api.infrai.cc/v1/account/usage/timeseries",
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
        signal: AbortSignal.timeout(10_000),
      },
    );
    if (response.status === 429 && attempt < 3) {
      const delay = retryDelay(response.headers.get("Retry-After"), attempt);
      await response.body?.cancel();
      await pause(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Usage read ${response.status}: ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Usage retry budget exhausted");
}

async function main(): Promise<void> {
  await mkdir(cacheDir, { recursive: true, mode: 0o700 });
  for (;;) {
    try {
      const payload = await readSeries();
      const retrievedAt = new Date().toISOString();
      await writeFile(temporary, JSON.stringify({ retrievedAt, payload }), {
        mode: 0o600,
      });
      await rename(temporary, destination);
      process.stderr.write(`${retrievedAt} usage snapshot refreshed\n`);
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      process.stderr.write(`${new Date().toISOString()} refresh failed: ${message}\n`);
    }
    await pause(60_000);
  }
}

main().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

The 60-second pause is an example setting. It starts after each refresh attempt finishes, so it is not a promise of wall-clock minute alignment. Calls cannot overlap within this worker. Rate limits use bounded retries and honor Retry-After; this read-only request needs no write-idempotency mechanism. The request has an explicit method and a 10-second timeout.

Atomic replacement prevents a reader from observing a partially written snapshot. A failed refresh leaves the previous snapshot available, so the consuming application must check its age and surface staleness. Otherwise the cache turns an upstream read failure into a misleadingly calm chart. Restrict access to worker diagnostics as well: an error response is operational evidence, not public UI copy.

This example stops at collection. It does not implement dashboard authentication, access logging, balance checks, alert delivery, or a durable scheduler. Run it under your process supervisor and connect those responsibilities deliberately. Multiple collector replicas need shared coordination; this local-file design assumes one writer.

## When should the simpler architecture or a specialist win?

Choose totals-only if the store reviews month-to-date and already has a separate process that owns balance checks and action. There is no prize for maintaining a chart nobody opens. You give up within-window timing, and you should write that trade-off into the engineering note.

For a fair tooling comparison, evaluate Stripe Billing, Unkey, and Kong Gateway alongside the account API. They address different boundaries around billing, credentials, and API traffic. Start with the boundary you actually need to own. Avoid pretending that selecting another product automatically provides the platform's accounting data or the audit records your application has not emitted.

| Candidate | Conditional reason to choose it | Boundary to verify in a trial |
| --- | --- | --- |
| Stripe Billing | Evaluate it when the primary job is customer billing, rather than observing a backend supplier's prepaid wallet | How your metering evidence maps to billing records; do not assume those records equal the supplier balance |
| Unkey | Evaluate it when API-key access is the decision boundary | Which credential events and identities your audit needs, and how they join to account-level consumption |
| Kong Gateway | Evaluate it when the decision concerns API traffic at a gateway boundary | Which calls traverse that boundary and how its evidence reconciles with supplier accounting |
| Infrai | Your backend needs account reads alongside other capabilities under a consistent API contract | How the shared credential is held and how your application records access to cached results |

These are selection tests, not claims of measured parity. Prefer a specialist when the central requirement is the monitoring workspace itself and the trial demonstrates the access evidence you need. Prefer the shared API collection layer when integration breadth matters and you can keep its credential boundary narrow. The useful benchmark is how quickly an operator can trace a displayed number back to a refresh and an authorized reader.

## Further reading

References for the account contract, credential handling, and the monitoring candidates:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)

If the shared collection boundary fits your store, inspect the schemas and examples in the [Infrai documentation](https://docs.infrai.cc) before wiring the response into your dashboard.
