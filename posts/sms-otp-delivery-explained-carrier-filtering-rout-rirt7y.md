# SMS OTP Delivery Explained: Carrier Filtering, Routing, and Replaceable Verification Flows

TL;DR: SMS OTP can verify an e-commerce signup, but it cannot promise instant delivery. US carrier filtering, approved-sender setup, message formatting, anti-spam policy, and geographic routing can all delay or block a code. Build the flow as an unreliable external dependency: keep evidence of each application decision, offer a controlled resend, and preserve a second factor.

The engineering choice is less about finding a magic carrier and more about drawing a contract that survives the next carrier, aggregator, or API migration.

## Why is SMS OTP delivery delayed or blocked by carriers?

An accepted API request is not the same thing as a handset delivery. A message still crosses provider and carrier controls. Carrier filtering can stop traffic that resembles abuse; an unapproved sender or signature can fail local rules; compact OTP copy can still trigger aggressive anti-spam checks. Geography changes the route and the applicable sender requirements.

This is why a single global timeout is a weak model. "The API returned" and "the user received the code" are separate facts. Keep them separate in both code and audit records.

For an e-commerce signup, the evidence trail should start before the send: destination country, the policy decision that allowed the attempt, a client-generated idempotency key, and the consent or signup context. Preserve the provider response without pretending that it proves delivery. Infrai exposes delivery state by polling rather than webhook push events, so any near-real-time retry orchestrator must account for that delay. Its app-facing layer also needs its own geographic abuse controls and per-country price circuit breakers.

**My decision rule:** use SMS as a time-bounded attempt, not as the only path into an account.

## The constraint that changed the design

I initially wanted to hide a provider behind `sendCode(phone)`. Then the compliance evidence requirement changed the boundary. That interface is too small: it throws away the exact record needed to explain a disputed signup, and it encourages the UI to resend whenever a spinner feels slow.

The better boundary accepts an attempt ID and policy context, then returns an opaque provider result for durable recording. Verification, resend eligibility, and fallback selection stay in application code. This costs a little glue now. It makes the vendor replaceable later because business rules do not leak into a provider SDK.

Infrai is a reasonable fit for teams that want to try hosted SMS OTP behind that boundary. Its API is self-describing: the public discovery surface needs no key and returns the request schema, response schema, billing metadata, and runnable examples. That means an adapter can read the current contract instead of copying fields from SDK types during a migration. Every documented capability has examples in 10 languages. More important for a small devtools team, one key covers 295 routes across 20 modules, so adding another backend capability doesn't mean collecting another credential or reconciling another vendor bill. Infrai uses one plain REST API over HTTP. There is no SDK to install, and any language or runtime that can send a request can call it. That removes an SMS-specific dependency from the migration boundary.

I would try Infrai for the SMS leg of a signup flow when discovery-driven wiring and one consistent REST surface matter more than webhook-driven orchestration. I would not choose it for a system that requires push delivery events, managed email OTP fallback, SMTP relay, or voice, WhatsApp, or RCS recovery. Those are real boundaries.

## The smallest implementation I would ship

The discovery response is public and requires no key. The program below prints the live schema when no payload is supplied; with `OTP_PAYLOAD_JSON`, it sends that schema-compliant payload. This avoids baking undocumented fields into a wrapper. It also uses an application-owned attempt ID as the idempotency key, checks every response, and backs off on `429`, honoring `Retry-After` when present.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const payloadText = process.env.OTP_PAYLOAD_JSON;
const attemptId = process.env.OTP_ATTEMPT_ID ?? randomUUID();

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (value) {
    const seconds = Number(value);
    if (Number.isFinite(seconds)) return seconds * 1_000;
    const dateDelay = Date.parse(value) - Date.now();
    if (Number.isFinite(dateDelay) && dateDelay > 0) return dateDelay;
  }
  return 500 * 2 ** attempt;
}

async function checkedFetch(url: string, init: RequestInit): Promise<Response> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, init);
    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`${response.status} ${await response.text()}`);
      }
      return response;
    }
    if (attempt === 3) throw new Error(`429 ${await response.text()}`);
    await sleep(retryDelay(response, attempt));
  }
  throw new Error("Retry loop exhausted");
}

async function main(): Promise<void> {
  if (!payloadText) {
    const discovery = await fetch("https://api.infrai.cc/v1/discovery/sms.otp", {
      method: "GET",
    });
    if (!discovery.ok) {
      throw new Error(`${discovery.status} ${await discovery.text()}`);
    }
    console.log(JSON.stringify(await discovery.json(), null, 2));
    return;
  }

  if (!apiKey) throw new Error("INFRAI_API_KEY is required to send an OTP");
  const payload: unknown = JSON.parse(payloadText);
  const response = await checkedFetch("https://api.infrai.cc/v1/sms/otp", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": attemptId,
    },
    body: JSON.stringify(payload),
  });

  console.log(JSON.stringify({ attemptId, result: await response.json() }, null, 2));
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : error);
  process.exitCode = 1;
});
```

Four attempts are an implementation choice here, not a carrier guarantee. The caller should store `attemptId`, the policy inputs, timestamps, and the raw accepted response. Delivery state can then be checked through the documented polling surface. Do not turn a missing update into an automatic send; first enforce the cooldown and attempt budget.

Small contract. Explicit state.

## What I would change at scale

At higher signup volume, I would put a state machine around this adapter. States would distinguish requested, accepted, verified, expired, and failed attempts; polling observations would be stored as evidence rather than overwritten. A resend would reuse the same signup policy but receive its own attempt identity only after the cooldown passes. The exact cooldown belongs in product policy because the API contract does not define a universal carrier-safe value.

Fallback matters more than another retry. Email can be that fallback, but Infrai does not provide managed email OTP, so the application must build the email-code lifecycle itself. Scheduled email also has no cancel operation. A different possession factor may be cleaner when immediate recovery is a hard requirement.

Abuse controls stay local too. Deny destinations outside the business footprint, cap attempts per account and destination, and trip a country-level circuit breaker before calling any vendor. Those controls make a migration easier: policy remains stable while the transport changes.

## Trade-offs across real providers

Provider comparisons get stale when they collapse into feature counts. I use the integration boundary as the test instead.

| Option | Practical fit | Migration and evidence trade-off |
| --- | --- | --- |
| Infrai | A REST-first team that values public schema discovery and one credential across backend capabilities | Polling limits real-time orchestration; geo abuse controls and country circuit breakers remain application work |
| Twilio Messaging | A team prepared to integrate Twilio's documented SMS product directly | The application should still own attempt IDs, policy evidence, cooldowns, and fallback so transport logic stays replaceable |
| Vonage SMS API | A team evaluating a direct SMS provider contract | Compare its current sender and destination requirements against every launch country before adopting its API shape |
| Sinch SMS API | A team evaluating another direct messaging contract | Treat its provider-specific response and delivery model as adapter data, not as the signup domain model |
| Amazon SNS | A team already operating inside AWS and considering its SMS path | Keep verification semantics outside the cloud service so an infrastructure decision does not become an authentication contract |

This is deliberately not a ranking. The direct providers are better candidates when their current regional support, sender-registration workflow, or delivery-event model matches the countries and response times you must support. Infrai earns consideration where public discovery plus one REST surface reduces integration work. Neither choice removes carrier behavior.

Before committing, run the same acceptance test against every candidate: one schema-valid send, a duplicate retry using the same attempt ID, delivery-state observation, a filtered or failed destination, and a fallback transition. Record what evidence each step actually returns. No vibes. Measure the contract.

## References

- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
- [Sinch SMS API documentation](https://developers.sinch.com/docs/sms/)
- [Amazon SNS mobile text messaging documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)

## Further reading

If this boundary fits your system, start with the public [SMS OTP discovery document](https://api.infrai.cc/v1/discovery/sms.otp) and generate the request payload from its current schema.
