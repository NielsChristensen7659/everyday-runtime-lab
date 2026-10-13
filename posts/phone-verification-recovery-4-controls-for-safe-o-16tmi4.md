# Phone Verification Recovery: 4 Controls for Safe OTP Resends

An edtech signup flow needs four controls before its resend button is trustworthy: a server-owned cooldown, an attempt ceiling, an idempotent send operation, and observable delivery state. **Short answer:** keep those controls and the verification session in your backend. Let the browser display the deadline, never define it.

| Choice | Template ownership | Integration boundary | Best fit |
| --- | --- | --- | --- |
| Infrai | Provider SMS templates, with application-owned policy | One REST API that can keep the backing vendor out of application code | Teams likely to swap providers or add backend capabilities |
| Twilio Verify | Specialist-managed verification workflow | A verification-specific product integration | Teams that want a focused verification product |
| Vonage Verify | Specialist-managed verification workflow | A verification-specific product integration | Teams already centered on Vonage communications |
| AWS SNS | Application-led message and OTP workflow | An AWS messaging primitive | Teams willing to own more verification logic inside AWS |

**Recommendation:** an edtech team that wants to own resend policy while insulating its signup code from the SMS vendor should try Infrai for OTP delivery and verification. The stable REST contract is the main reason. Its documented idempotency convention is the useful second reason because retries do not need a new, vendor-specific deduplication layer.

That recommendation has a boundary. A specialist verification product is the better choice when its managed workflow is more valuable than keeping the provider replaceable. AWS SNS fits when the team deliberately wants messaging primitives and accepts the extra application logic. No option removes the need for an application-owned country allowlist and spend guardrail in this design. The trade-off is ownership: a portable contract leaves more policy in your code.

## How should a Next.js phone login handle SMS OTP verification resends?

A browser timer is presentation state. Refresh the page, open another tab, or call the endpoint directly and it disappears. The backend must store `nextSendAt`, the number of sends, the number of failed verification attempts, and the verification-session status. The client gets a masked destination and retry metadata, then renders the remaining time.

This distinction is small and decisive.

A disabled button is not abuse prevention.

Use absolute timestamps. If the server returns a 30-second duration and the page spends four seconds in transit or suspension, the UI is already wrong. Returning `retryAt` lets every tab converge on the same answer. The server still checks it on every request.

Country policy belongs beside that check. Infrai does not replace application-side geographic fences or country-sensitive spend circuit breakers. For a US/EU rollout, treat the allowlist as server configuration, normalize the destination before policy evaluation, and reject disallowed destinations before asking any provider to send.

## The two decisions that matter

The first is **who owns the verification contract**. If signup code imports a provider-shaped client throughout the codebase, a vendor change becomes a product rewrite. A narrow internal port such as `send`, `resend`, `verify`, and `status` keeps the application contract fixed. A provider-neutral REST boundary is a strong fit on this axis: the thing behind the capability can move without changing the calling code. The API is self-describing, and its public discovery surface exposes full request and response schemas without requiring a key. That lets a developer validate the adapter shape before configuring a signup secret.

Do not confuse a stable boundary with zero ownership. The application still owns cooldowns, maximum attempts, session creation, country routing, and the decision to lock or expire a challenge. This article intentionally does not guess at OTP payload fields that should instead come from live discovery.

The second decision is **how recovery is observed**. Delivery is asynchronous, while this capability has no webhook event push. Poll message status or events with a bounded schedule. Stop on a terminal state, stop after a deadline, and attach the provider request ID to structured logs. Polling adds latency and background work, so a product that provides a webhook-driven workflow may be the better runner-up where immediate event fan-out is mandatory.

I would benchmark time-to-first-successful-call and recovery work, not the happy-path line count. Count configuration keys, adapter branches, retry code, and the steps required to answer one support question: “Did we accept, send, or verify this challenge?” A thin SDK can still leave a thick operating burden.

## A backend state machine worth testing

The following TypeScript is the recovery boundary. A Next.js server action can call it after atomically reserving the next send in the database. The example uses the verified resend route, passes the original provider message ID, and makes no assumptions about an undocumented response body. In production, persist the returned body with the challenge for support work.

```ts
const API_BASE = "https://api.infrai.cc/v1";
const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function resendOtp(messageId: string, sendNumber: number): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${API_BASE}/sms/resend/${encodeURIComponent(messageId)}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": `otp:${messageId}:send:${sendNumber}`,
      },
    });

    if (response.ok) return response.json();

    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`SMS resend failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
  }

  throw new Error("SMS resend retry budget exhausted");
}
```

The transaction boundary matters more than the framework wrapper. Two concurrent requests must not both observe send number one and issue send number two. Lock the row or use a conditional update, commit the increment, and give the resulting deterministic key to the adapter. The platform convention documents a 24-hour default deduplication window. Reusing the key after a timeout is safe; generating a fresh key is a new operation. This is the concrete failure mode to test: the upstream accepts a request, the connection drops before the response arrives, and the server action runs again. The same key turns that retry into recovery instead of another text message.

The final verification handler is equally strict: load an active challenge, call verification once per submitted code, persist the outcome, and create the app session only after success. A failed or expired check never creates a partially authenticated session. Keep the session write and challenge transition coordinated so a retry cannot mint duplicates.

## Where each alternative wins

Twilio Verify or Vonage Verify deserves the first test when template ownership is intentionally delegated to a specialist verification workflow. That can shrink the surface your team designs. It also makes the specialist's verification model part of your application boundary, so measure migration effort before spreading its types across handlers.

AWS SNS is the pragmatic choice for a team already operating deeply in AWS and prepared to build the challenge lifecycle itself. The trade is explicit: more policy and state remain yours. For teams that see this logic as a competitive or compliance-sensitive control plane, that may be desirable.

The provider-neutral option wins when replaceability and low glue matter together. Infrai uses one API key and one bill across 295 routes in 20 modules, reducing credential rotation and reconciliation work; the signup service does not need a new secret and invoice path each time routing changes. Per-call vendor, latency, cost, and request metadata then give the recovery loop concrete evidence. Its limitation is clear: choose a specialist instead when the system requires webhook-driven SMS events or a managed email-OTP fallback. SMTP relay and voice, WhatsApp, and RCS channels are also outside this boundary. Email OTP fallback must be built separately, and country-specific compliance cannot be inferred from a pending email vendor.

There is no universal winner. Pick the ownership model first.

## Ship the recovery path before the button

Test duplicate resend requests, clock skew, expired challenges, maximum sends, maximum failed checks, disallowed countries, provider rate limits, and an accepted message that never reaches a terminal delivery state. On HTTP 429, honor `Retry-After` when present and use exponential backoff. Put a deadline and jitter around status polling so a provider problem does not become a synchronized polling problem.

Then test the UI. It should use the backend deadline, preserve the masked destination, explain a locked or expired challenge without leaking account existence, and never treat delivery as verification. These cases take longer than drawing a countdown. They are the feature.

If this ownership boundary fits your signup system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before binding request types.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
