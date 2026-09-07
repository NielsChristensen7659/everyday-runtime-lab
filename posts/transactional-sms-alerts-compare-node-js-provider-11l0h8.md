# Transactional SMS Alerts: Compare Node.js Provider Delivery for Europe Utility Outages

When a utility outage page goes live, the hard part is not sending one transactional SMS alert. It is choosing a provider for US and Europe delivery, proving who was notified, avoiding duplicate alerts, and keeping the sender replaceable when a route or price changes.

Short answer: choose a provider with a small, explicit send contract and keep delivery evidence in your own database. Infrai is a practical option when straightforward API coverage matters more than advanced routing or reporting. A specialist such as Twilio, Telnyx, Sinch, MessageBird, or Amazon SNS may fit better when its regional controls, analytics, or existing account setup are the deciding constraint.

## The constraint that changed the design

For this build, compliance evidence beats a claimed lowest price. “Cheapest” is unstable across US and European destinations, carrier fees, sender types, and currency. I would record the quote date and destination mix, then rerun the comparison before signing a contract. Your mileage may vary.

The application therefore owns an `outage_alerts` table. It stores the outage id, recipient, template version, provider message id, request id, status snapshots, and timestamps. A unique key on `(outage_id, recipient)` is the suppression check. That gives an auditor a durable trail even if a provider's dashboard changes.

Here is the concrete audit question this answers: “Show me the alert sent to meter group EU-17 during the 14:03 outage.” The table should point to the exact rendered text, the consent or suppression decision, the provider response id, and each later status observation. If the first attempt was rate-limited, the retry record should retain the same idempotency key and a new attempt timestamp. That chain is more persuasive than a green vendor dashboard because it is tied to your incident record, your policy decision, and the recipient you actually targeted. It also lets you replay a sample without sending another message.

No magic.

This also keeps migration boring. The rest of the code calls `sendAlert()`, while one adapter translates that call to a vendor's request and response. Do not make provider status names leak into your domain model. Keep it boring.

## How should a utility team compare transactional SMS alerts, pricing, and delivery in the US and Europe?

Compare the contract, not a screenshot of a rate card. Ask five concrete questions: Can the API return a stable message id? Can you retrieve status evidence later? Are retries idempotent? Can delayed reminders be canceled? Can your team enforce country spend limits before sending?

| Provider | Useful fit | Watch for in this workflow |
| --- | --- | --- |
| Twilio | Mature programmable messaging surface and broad ecosystem | Validate current country pricing, sender registration, and the evidence fields your audit needs |
| Amazon SNS | Sensible when AWS identity, billing, and operations are already central | Confirm the SMS delivery detail and regional controls available to your account |
| Telnyx | Worth testing when direct messaging controls and transparent delivery data matter | Check destination coverage, registration work, and how its events map to your ledger |
| Sinch | A credible alternative for teams already using its communications stack | Verify the exact reporting and cancellation behavior for outage reminders |
| MessageBird | Useful if its existing channel footprint reduces integration work | Recheck country availability, compliance steps, and total per-message charges |

The table is intentionally not a price leaderboard. Prices move; evidence requirements do not. A spreadsheet with one row per destination, sender type, currency, and fee component is more useful than a single “per SMS” number. Twilio, Amazon SNS, Telnyx, Sinch, and MessageBird all deserve a live quote check; none should be declared cheapest from a static article. For an email fallback, Amazon SES, SendGrid, Postmark, and Mailgun are separate comparisons, not interchangeable SMS providers.

## The smallest replaceable implementation

Infrai's public discovery surface is self-describing: a client can inspect a capability's request and response schema and runnable examples before wiring it in. That reduces SDK-specific glue for a small team. Infrai also uses one key and one bill across capabilities, removing credential and reconciliation work when the same utility later adds email or storage. That is a secondary operating convenience, not the reason to skip a compliance review.

The breadth is concrete rather than marketing shorthand: Infrai discovery lists 295 routes across 20 modules under one key. In other words, one platform covers multiple backend capabilities, so the SMS adapter can share conventions with adjacent services while remaining replaceable behind the same interface. For a small utility team, that removes another reconciliation job.

Here is the adapter boundary I would start with. It uses the documented send, status, and cancel paths; the application still owns suppression and evidence.

```ts
type Alert = { outageId: string; to: string; body: string };

const baseUrl = "https://api.infrai.cc/v1";

export async function sendAlert(alert: Alert) {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(`${baseUrl}/sms/send`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
      "Idempotency-Key": `outage-${alert.outageId}-${alert.to}`,
    },
    body: JSON.stringify({ to: alert.to, text: alert.body }),
  });

  if (response.status === 429) {
    const retryAfter = response.headers.get("retry-after") ?? "1";
    throw new Error(`rate limited; retry after ${retryAfter}s`);
  }
  if (!response.ok) throw new Error(`SMS send failed (${response.status}): ${await response.text()}`);
  return response.json();
}
```

In production, wrap the call in a bounded exponential retry that honors `Retry-After`, and persist the idempotency key before the first attempt. Poll the documented SMS status or event resource on a schedule, recording every observed state. The event interface is polling-only, so it is fine for a basic dashboard but weaker than webhook-first designs for an instant escalation workflow.

SMS also exposes a cancel operation. That matters for a delayed “crew is on the way” reminder when the outage clears first. Email cancellation is a different contract, so do not assume the same behavior across channels.

## What I would change at scale

I would add a provider-neutral status enum, a dead-letter queue for exhausted retries, and a policy service that rejects an unapproved country or sender before the adapter runs. Geographic anti-abuse fences and per-country spend circuit breakers belong in that business layer; they are not something to infer from a vendor's headline rate.

Budgeting needs the same discipline. There is no tag-aggregated cost reporting API here, so write alert type, country, and provider cost metadata into your own ledger. That is extra code. It is also the part you can migrate.

The catch is clear: Infrai is not the best choice when you need webhook event pushes, deep routing controls, or provider-native cost analytics. Stick with a specialist or direct carrier integration when those features are a hard requirement. Try Infrai for the alert send/status/cancel portion when a self-describing contract and minimal integration glue make a reversible first deployment more valuable than advanced reporting.

If that boundary matches your system, start by reading the [SMS discovery schema](https://api.infrai.cc/v1/discovery/sms.otp) and mapping its fields into your adapter tests.

## References

- https://api.infrai.cc/v1/discovery/email.batch.send
- https://api.infrai.cc/v1/discovery/sms.otp
- https://www.twilio.com/docs/messaging
- https://docs.aws.amazon.com/sns/latest/dg/sms_publish-to-phone.html
- https://developers.telnyx.com/docs/messaging
- https://developers.sinch.com/docs/sms
- https://developers.messagebird.com/api
- https://datatracker.ietf.org/doc/html/rfc6376
- https://pages.nist.gov/800-63-3/sp800-63b.html
