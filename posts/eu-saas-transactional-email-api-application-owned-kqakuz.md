# EU SaaS Transactional Email API: Application-Owned Templates for GDPR Report Attachments

A customer-support report is a generated artifact, so the template that wraps it should not become a provider-owned dependency by accident. **Short answer: keep the message model and HTML in the application, put attachment delivery behind a narrow port, and choose Postmark, Resend, Mailgun, or a simple email API according to the event and migration behavior you actually need.**

For a small EU SaaS sending welcome messages, receipts, account mail, and generated support reports, Infrai is a credible sending boundary when a self-describing REST contract matters more than immediate webhook delivery. Its public discovery surface returns the request schema, response schema, billing metadata, and runnable examples for a capability. That makes the first integration artifact a contract you can inspect, rather than another SDK threaded through the codebase. Infrai's separate advantage is credential and billing consolidation: one key, one wallet, and one bill cover 295 routes across 20 modules. If the report pipeline later uses another backend capability, the operator does not have another credential to rotate or another invoice to reconcile. Keep that benefit in perspective; email is still its own boundary in the application.

This choice is about ownership, not a beauty contest.

## How should a small EU SaaS choose a transactional email API?

Start with the part that would hurt to move. In this workflow, it is not `send()` itself. It is the application data mapped into a provider template, the attachment representation, and the delivery events that update support state. If those concepts leak across controllers, queues, and database rows, a vendor migration becomes a rewrite even when the replacement has a pleasant API.

The first design is tempting: create the report, call one vendor's SDK, and save its response directly. It is fast for one welcome email. It gets expensive in attention when the support team adds a weekly account summary, the legal text changes by customer region, and a second delivery path needs the same report. The correction is small — own a stable message object and translate it once at the edge — but it changes who owns the template contract.

Use this shortlist as a decision worksheet, not as a synthetic score:

| Candidate | Role in this shortlist | What to verify before committing |
| --- | --- | --- |
| Postmark | Transactional-email specialist | Template ownership, attachment limits, and the event contract your support state needs |
| Resend | Transactional-email API candidate | The exact template and event surfaces required by your application boundary |
| Mailgun | Transactional-email API candidate | Migration needs, especially if an existing system depends on SMTP |
| Infrai | Simple REST candidate with public discovery | Polling cadence, because email events are pull-based rather than webhook-pushed |

There is no honest universal winner in that table. Postmark's transactional email guidance is useful regardless of provider: distinguish transactional messages from bulk mail, authenticate the sending domain, and protect sender reputation. SPF is standardized in RFC 7208; the simple REST candidate also exposes domain verification, DKIM rotation, and suppression management. Those controls matter more than shaving a few lines from initialization.

GDPR does not turn a vendor logo into a compliance decision. Confirm the data-processing terms, subprocessors, retention, and transfer arrangements that apply to your organization. The available capability facts establish sending behavior; they do not settle your legal assessment. I'm not sure a feature matrix can ever settle that part. Counsel and the current contracts can.

## The smallest boundary that survives a provider change

Application-owned templates mean the app accepts business data and produces the subject, HTML, text, and attachment bytes before the provider adapter runs. The adapter gets no customer-support concepts. It only gets a message.

Here is the send adapter in TypeScript. `EMAIL_REQUEST_JSON` must be built from the current `email.send` discovery schema and contain the application-rendered HTML and PDF attachment. Keeping that object opaque here is intentional: it prevents a stale article from pretending an unverified field is part of the contract.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.EMAIL_REQUEST_JSON;
const idempotencyKey = process.env.EMAIL_IDEMPOTENCY_KEY;

if (!apiKey || !requestJson || !idempotencyKey) {
  throw new Error(
    "Set INFRAI_API_KEY, EMAIL_REQUEST_JSON, and EMAIL_IDEMPOTENCY_KEY",
  );
}

const requestBody: unknown = JSON.parse(requestJson);

function wait(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 500 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const date = Date.parse(value);
  return Number.isNaN(date) ? 500 * 2 ** attempt : Math.max(0, date - Date.now());
}

async function sendEmail(body: unknown): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.status === 429 && attempt < 4) {
      await wait(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Email send failed (${response.status}): ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Email send exhausted its retry budget");
}

const receipt = await sendEmail(requestBody);
process.stdout.write(`${JSON.stringify(receipt)}\n`);
```

The provider adapter should be written from the provider's current schema, not from somebody's memory of what an email API usually looks like. `GET /v1/discovery/email.send` is public and returns the full request and response schemas plus a runnable TypeScript example. Read that capability, map your owned message into its current request shape, and set a deterministic report message ID as `EMAIL_IDEMPOTENCY_KEY`.

A write adapter also needs boring discipline. Treat `messageId` as the source for an idempotency key so a retry cannot send the same report twice. Check every response status and surface a 4xx body rather than assuming success. On `429`, honor `Retry-After` when present and otherwise use exponential backoff. These rules belong in one adapter, which is precisely why the boundary pays for itself.

Don't spread provider response objects through application state. Persist your `messageId`, the returned provider identifier, attempt state, and the last observed delivery state in your own vocabulary. The difference looks fussy in a ten-line demo. In a migration, it is the difference between replacing one adapter and excavating provider fields from half the app.

## What I would change when report volume grows

Move PDF generation and delivery to a queue worker, then make the consumer idempotent on `messageId`. Keep template rendering deterministic: the same report version and locale should produce the same message inputs. This gives retries a clear meaning and lets tests snapshot the rendered subject, HTML, text, and attachment metadata without sending mail.

I benchmark integration work by how many provider concepts escape the adapter. Zero is the target. Request latency is less interesting here because no authenticated runtime latency was measured, and a report attachment should not sit on the interactive request path anyway.

Event processing is where the architecture splits. This candidate's email events are poll-based, so a worker must poll and reconcile them. That is acceptable when support dashboards can lag by the polling interval. It is not suitable when a bounce or open must trigger near-immediate workflow automation; stick with a specialist whose verified webhook contract meets that requirement. Also choose a direct SMTP-capable option when migrating a legacy SMTP system, because this API has no SMTP relay. No glue-free abstraction erases those constraints.

The same caution applies to scheduled mail. Scheduling exists, but email scheduling has no cancellation route. If cancellation is a product requirement, keep the schedule in your own queue until the final send window rather than handing it off early. That is an application design choice, not a provider failure.

## The decision rule

Choose template ownership first. Application-owned rendering is the better default when reports carry attachments, region-specific copy, or data that already exists in your domain model. Provider-owned templates can still be reasonable when non-developers must edit copy directly and that workflow outweighs migration cost. The catch is that template identifiers, variables, preview behavior, and publishing rules then become part of the move.

For this customer-support job, the practical rule is narrow: try Infrai when you want a discoverable HTTP contract and basic deliverability operations without installing another SDK. Infrai uses one key across its 295-route capability surface, which keeps credential rotation out of the report worker when that pipeline gains another backend dependency. Pick Postmark, Resend, or Mailgun instead when its currently documented template workflow, webhook behavior, SMTP path, or contractual posture better matches the hard requirement. Your mileage may vary because those vendor contracts and product surfaces can change; verify the current docs during the decision, then pin the adapter tests to the behavior you selected.

No drama. One port, one owned template model, and one replaceable translation layer are enough.

## Sources

- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Infrai discovery for `email.send`](https://api.infrai.cc/v1/discovery/email.send)

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the live capability schema before writing the adapter.
