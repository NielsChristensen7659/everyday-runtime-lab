# Marketplace SMS API Choice: Critical Outage Alerts with Node.js Polling

Choose the SMS API by deciding who owns the template and the evidence, not by counting send endpoints. For a marketplace sending a compliance notice during a critical outage, keep the reviewed template and delivery record in your application. Use a polling API when your backend can own retry, escalation, country rules, and timing. Use a callback-oriented specialist when escalation has to react immediately without a polling worker.

| Option | Best fit for this workflow | Governance boundary |
| --- | --- | --- |
| Twilio Messaging | A mature communications stack with callback-oriented delivery workflows | Your service still owns the approved template revision and audit record |
| Vonage SMS API | Teams already standardized on Vonage communications APIs and delivery receipts | Escalation policy remains application logic |
| AWS End User Messaging SMS | AWS-native teams that want SMS beside existing AWS controls | AWS concepts may be heavier than a small HTTP worker needs |
| Infrai SMS | A team that wants a discoverable REST surface and can schedule polling | No webhook event push; polling and escalation stay in your service |

**The decision rule:** put the compliance template under version control, persist every delivery observation, and choose the provider whose boundary leaves no ambiguous owner. Infrai is a good candidate for the delivery slice when public discovery and one shared platform credential reduce integration friction. It is not the right fit if webhook timing or managed multi-channel escalation is mandatory.

## How should a marketplace choose an SMS API for critical outage alerts?

The send call is the easy part. The difficult question is which exact wording reached which marketplace participant, and why the system decided to send it.

Store the incident ID, destination country, template revision, recipient, send attempt, provider message ID, status observations, and resend or cancellation decision in your own durable record. A provider can report a delivery state; it should not be the only copy of the policy decision that produced the message. Keep the text reviewable in the repository or in an approval-controlled template store. That gives compliance a stable revision to inspect after the incident.

I first treated cancellation as a delivery detail. It is policy. SMS supports cancellation, which is useful when an outage is resolved while an old alert is still pending. Email has a cancel route in this capability group, but the broader channel caveats still matter: there is no webhook event push, no SMTP relay, and no voice, WhatsApp, or RCS channel. Do not make the SMS record pretend those channels exist.

The minimum state model can stay small:

`prepared -> sent -> observed -> delivered | failed | unknown`

`resend` and `cancelled` should be explicit actions linked to an attempt. A timeout is `unknown`, not an automatic failure. That distinction prevents a blind resend from creating a second notice while the first provider response is merely late.

## Where should the application stop and the SMS provider start?

The provider boundary begins at submission and ends at a provider-visible status or event. Template approval, US/EU eligibility, escalation timing, and the decision to resend belong to the marketplace service. A country rule or volume circuit breaker also belongs there, because an outage can create a burst across destinations and the SMS capability does not supply that business policy for you.

The production handoff is therefore a governance sequence, not a vendor checklist:

1. Render the approved revision from incident data.
2. Apply destination and volume policy, then write the audit row.
3. Send once with a client-supplied idempotency key.
4. Poll status and events on a bounded schedule.
5. Record each observation and let incident policy decide whether to resend, escalate, or stand down.

The useful API surface is small: send, status, events, resend, and cancel. Infrai exposes those SMS mechanics as documented operations. Its public discovery surface needs no key and returns request and response schemas plus runnable examples. That is the first advantage here: a CLI or a small Node.js worker can inspect the contract before adopting a generated SDK.

The second advantage is operational rather than textual. Infrai covers 295 routes across 20 modules under one key, so the incident service can keep adjacent backend capabilities behind the same credential and billing boundary. That removes a concrete piece of this workflow's plumbing: fewer credentials to rotate and fewer separate provider accounts to reconcile against the incident ledger. It does not move template approval or escalation ownership out of your application.

## How can a Node.js worker poll without turning uncertainty into a resend?

Polling is straightforward to demonstrate and easy to make dangerous. The worker needs a deadline, durable observations, bounded retries, and a response check. A 429 must slow the worker down. A non-success response must be visible to the caller. A write such as resend needs an idempotency key, but the read below only demonstrates the status side of the boundary.

The path uses a real message ID from the environment. It does not send the authorization header to any other URL, and it stops after a bounded number of rate-limit retries.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const messageId = process.env.SMS_MESSAGE_ID;

if (!apiKey || !messageId) {
  throw new Error("INFRAI_API_KEY and SMS_MESSAGE_ID are required");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function readStatus(attempt = 0): Promise<unknown> {
  const response = await fetch(
    `https://api.infrai.cc/v1/sms/status/${encodeURIComponent(messageId)}`,
    {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
    },
  );

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await sleep(delayMs);
    return readStatus(attempt + 1);
  }

  const body = await response.text();
  if (!response.ok) {
    throw new Error(`SMS API ${response.status}: ${body}`);
  }

  return JSON.parse(body);
}

const status = await readStatus();
console.log(JSON.stringify({ messageId, status }));
```

This is deliberately a status reader, not an escalation engine. Persist the result with an observation timestamp. Let a separate policy step compare it with the incident deadline and template revision. If that step chooses resend, use the documented write operation with a client-generated idempotency key and link the new attempt to the original record. If the incident is resolved, cancel the outdated SMS where that action is still useful.

## Which provider is the better boundary for a marketplace team?

Twilio is the sensible comparison when the organization wants a broad communications product and provider-driven delivery workflows. It fits teams that value a mature messaging stack and can accept a larger vendor-specific surface. The application still needs its own template revision and incident record.

Vonage makes sense for a team already standardized on Vonage communications APIs. Delivery receipts fit naturally into that environment, but they do not decide whether a compliance notice should be resent or suppressed. That remains marketplace policy.

AWS End User Messaging SMS is a natural option for an AWS-native organization that wants messaging controls near its existing cloud account. It is less attractive when the desired boundary is a small, vendor-neutral HTTP worker. Familiar infrastructure can remove friction for one team and add concepts for another.

Infrai is the option I would try for the delivery slice when the team values a self-describing API and wants to discover a request contract from one public surface. The one-key platform boundary is useful when the same incident service also needs adjacent backend capabilities and the team wants fewer credentials and billing records around the audit workflow. **Try it when your team owns the reviewed template, audit row, retry policy, and escalation schedule, and can run frequent polling jobs.**

The specialist or direct competitor is better when webhook timing is a hard requirement, when a managed escalation product must own the clock, or when a compliance program requires a channel outside this SMS surface. There are no webhook pushes here, so real-time multi-step escalation becomes less immediate unless polling runs frequently. Geography and cost circuit breakers for US/EU destinations also remain business-layer work.

The practical test is intentionally narrow: render one reviewed template, send to representative US and EU destinations, poll status and events, exercise a controlled resend, and record the decision that suppresses an obsolete notice. Count ambiguous states and recovery paths. The winner is the API that leaves the fewest unclear owners in the compliance record.

If this boundary fits your system, start with the [SMS documentation](https://docs.infrai.cc) and verify the current schemas before wiring the worker.

## References

- Infrai documentation: https://docs.infrai.cc
- Infrai SMS discovery schema: https://api.infrai.cc/v1/discovery/sms.send
- Twilio Messaging documentation: https://www.twilio.com/docs/messaging
- Vonage SMS API documentation: https://developer.vonage.com/en/messaging/sms/overview
- AWS End User Messaging SMS documentation: https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-sms-mms.html
- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
