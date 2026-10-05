# Password Reset Email Links With Gated SMS OTP Fallback for Marketplaces

TL;DR: use an email verification link for the normal marketplace signup and password-reset path. Offer a texted one-time code only after an explicit failure or user request, and only for a phone number already bound to that account. Do not switch channels because an email-open event is missing. Apple Mail Privacy Protection can prevent senders from seeing whether a recipient opened a message, so opens are a poor control signal.

| Choice | Integration surface | Recovery property | Best fit |
| --- | --- | --- | --- |
| Email link first | Token issue, message send, link consume | One credential on the common path | Default signup verification and reset |
| SMS code first | Code issue, phone delivery, code verify | Works without inbox access | A verified phone is already required |
| Email with gated SMS | Both flows and one shared state machine | A second delivery path without silent switching | A marketplace already holds verified phones |

**Recommendation:** choose email-link-first with a user-invoked SMS fallback. The extra channel earns its keep only when the account already has a verified phone and the team can operate both paths as one recovery system. Otherwise it is config bloat wearing a reliability badge.

## Should password reset use an email link or SMS OTP fallback?

A delivery event is evidence about a provider handoff. An open event is evidence about a mail client, and sometimes not even that. Neither proves that the marketplace user controls the inbox. The clean trigger is boring: the user says the link did not arrive or cannot be used, then requests another allowed method.

## Why privacy signals cannot govern recovery

This distinction matters on Apple devices because Mail Privacy Protection downloads remote content in the background and prevents senders from learning whether a message was opened. A rule such as "send SMS when no open appears after two minutes" therefore mixes an unreliable observation with an account-security decision. It can also send an unexpected text after the user has already completed the link flow.

Keep the transition explicit. A signup attempt can move from `email_pending` to `verified`; a recovery attempt can move from `email_pending` to `sms_pending`, then to `recovered`. Completion invalidates every outstanding credential for that attempt. No parallel winners.

That is the first criterion: **the fallback trigger must come from user intent and server state, not tracking pixels.**

Pixels lie.

## The race that defines account ownership

The main integration cost is not calling a second sender. It is making two credentials obey the same expiry, attempt, completion, audit, and privacy rules. If email and SMS live in separate handlers with separate databases, edge cases multiply quickly: a consumed link may leave a code valid, resends may create several live credentials, and support tooling may show conflicting outcomes.

Model the attempt before choosing adapters. The sender interfaces should be dull. That is good DX.

```ts
import { createHash, randomBytes, randomInt } from "node:crypto";

type Channel = "email_link" | "sms_code";
type Status = "email_pending" | "sms_pending" | "complete" | "expired";

interface VerificationAttempt {
  id: string;
  accountId: string;
  purpose: "signup" | "password_reset";
  status: Status;
  credentialHash: string;
  expiresAt: Date;
  failedChecks: number;
}

interface DeliveryAdapter {
  send(destination: string, credential: string): Promise<void>;
}

const digest = (value: string): string =>
  createHash("sha256").update(value).digest("hex");

function issueCredential(channel: Channel): { raw: string; hash: string } {
  const raw = channel === "email_link"
    ? randomBytes(32).toString("base64url")
    : randomInt(0, 1_000_000).toString().padStart(6, "0");
  return { raw, hash: digest(raw) };
}
```

The snippet deliberately stops at the trust boundary. Hashing prevents the stored value from being the credential itself, but a short numeric code still needs server-side attempt limits because its search space is small. The database transition that consumes a credential must be atomic: compare the current status and expiry, mark the attempt complete, and reject a second consumer.

Do not let adapters decide policy. They send. The application service decides whether the account has an eligible phone, whether another credential may be issued, and whether the attempt is already finished.

## Benchmark ownership after the first successful call

The second criterion is **how much shared policy the integration forces you to duplicate.** Count persisted states, retry queues, secrets, templates, callbacks, dashboards, and on-call checks. Benchmark time-to-first-successful-call, but also time a less flattering task: changing the expiry rule once and proving that both channels now follow it. If that takes edits in two workflows, the abstraction is leaking.

A small service can keep fallback policy visible without coupling account data to delivery code. Its inputs are authoritative account state and the current attempt, not inferred engagement.

```ts
interface RecoveryView {
  accountId: string;
  verifiedPhone?: string;
}

interface AttemptStore {
  replaceCredential(
    id: string,
    expected: Status,
    next: Status,
    hash: string,
    expiresAt: Date
  ): Promise<boolean>;
}

async function requestSmsFallback(
  account: RecoveryView,
  attempt: VerificationAttempt,
  store: AttemptStore,
  sms: DeliveryAdapter,
  now: Date
): Promise<{ accepted: boolean }> {
  if (!account.verifiedPhone || attempt.status !== "email_pending") {
    return { accepted: false };
  }

  const credential = issueCredential("sms_code");
  const expiresAt = new Date(now.getTime() + 10 * 60 * 1000);
  const changed = await store.replaceCredential(
    attempt.id, "email_pending", "sms_pending", credential.hash, expiresAt
  );

  if (!changed) return { accepted: false };
  await sms.send(account.verifiedPhone, credential.raw);
  return { accepted: true };
}
```

Ten minutes and six digits are example policy values, not universal recommendations. Put both in reviewed application configuration, record the policy version on the attempt, and test boundary values with a fixed clock. Do not scatter them across templates and callbacks.

The production flow also needs idempotency around the user action, a bounded verification-attempt counter, redacted logs, and reconciliation for accepted transitions whose delivery call failed. Return the same neutral response for unknown and known accounts. That keeps the public behavior from becoming an account lookup tool. A concrete race is easy to miss: the email link is consumed after the fallback transaction commits but before the text leaves the sender. The state transition, rather than either delivery adapter, must select the winner. The losing request gets a neutral result, and a queued delivery must check current state before another attempt. This costs more integration work than two happy-path API calls suggest.

Test this as a transition table, not as a pile of mocked sender calls. `email_pending + verified phone + explicit request` may enter `sms_pending`. `complete + any request` stays complete. Expired attempts stay expired. Two simultaneous fallback requests produce one active credential. A successful email-link consumption racing a fallback request produces one winner. Those tests catch integration failures that a provider sandbox cannot see.

## When is SMS-first the better runner-up?

The email-first recommendation has a clear limitation: it is a poor fit when inbox access is routinely unavailable. SMS-first is defensible when a verified phone number is already mandatory for the marketplace relationship and inbox access is not a reasonable assumption. It removes one branch from the common flow. The trade-off is that phone-number lifecycle handling becomes central rather than optional, so the team must own number changes, loss of access, and support escalation as first-class account events.

Email-only is the better runner-up when phone collection would exist solely for fallback. Fewer states, fewer templates, fewer sensitive fields. Nice. Add another channel after observed recovery failures justify its operational surface, not because a matrix cell looks empty.

No channel is free.

US versus EU should not be implemented as a hard-coded channel switch. Geography does not prove consent, account ownership, or user preference. In the EU, if consent is the legal basis for a particular processing activity, GDPR Article 7 requires that consent be demonstrable, distinguishable from other matters, clear, and as easy to withdraw as to give. Recovery messages and promotional messages should remain separate policy categories. Get the actual legal basis and retention rules reviewed for the service; do not smuggle marketing permission into an authentication checkbox.

The decision rule is compact: ship the smallest state machine that covers a real loss-of-access case. Benchmark integration work across the whole lifecycle, including the second credential, race tests, observability, and support operations. The fastest demo is not the fastest system to own.

## References

- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- https://gdpr-info.eu/art-7-gdpr/
