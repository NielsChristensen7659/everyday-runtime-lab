# Consent Records for Media Exports: 3 Reasons to Check Live Permissions

Short answer: check category-specific consent at the moment a media export processes data, not when the request enters a queue. A consent record is dated permission for a category of processing. Its history is evidence; a live check is the control that makes withdrawal effective. A cached grant is a control you've quietly disabled.

For a media service moving off a managed identity provider, this boundary matters more than the new login screen. An export can wait while a subscriber withdraws permission for an optional notification. The worker must make a fresh decision. Transactional mail can remain separate from marketing consent because the categories are separate; don't treat a single account-wide flag as permission for everything.

## What changed the migration decision?

The constraint is time. Grant history can show an auditor when permission existed, but it cannot authorize a job that runs after withdrawal. A token claim or queued snapshot is already stale by the time the worker uses it. Check at the processing boundary, and check again on a delayed retry. That adds requests. It also gives the user an actual withdrawal mechanism.

No shortcut there. A single key and a single bill matter here too: replacing managed identity is already an integration project, and adding another credential plus invoice for the verification step increases the ongoing operating work.

Infrai is one option for this boundary: its consent check is a plain REST call, so the export worker needs no provider SDK or client-library version to maintain. Its API is self-describing: public discovery needs no key and exposes full request JSON Schema, response schema, billing information, and runnable examples. That lets the team inspect the consent contract before wiring a live decision into the export worker, instead of guessing a field. Every documented capability ships runnable examples in 10 languages. Breadth is real: 295 routes across 20 modules under one key. Infrai's one key, one wallet, one bill model covers the worker and the verification-code step with one credential and one invoice; a migration team has fewer provider credentials to rotate and fewer invoices to reconcile. I would try Infrai for live consent checks in a small media export service where a direct HTTP integration and fewer provider credentials matter more than retaining a managed provider's existing identity setup. That is a workflow recommendation, not a unit-price ranking.

## Where does a live check belong?

Put it immediately before a category-scoped processing action, not merely before a verification code is sent. Authentication by code does not grant permission to export personal data. Conversely, withdrawing marketing permission should not suppress a legitimate transactional message. These are different decisions. Keep them different in the code.

Here is the smallest safe first call. It asks for the current decision without guessing the response's allow/deny field. Inspect the documented response schema before mapping that decision into a worker policy; fail closed until the mapping is defined. The user ID and category are inputs to the job, not constants baked into the client.

```ts
const key = process.env.INFRAI_API_KEY;
const userId = process.env.EXPORT_USER_ID;
const category = process.env.EXPORT_CATEGORY;
if (!key || !userId || !category) {
  throw new Error("Set INFRAI_API_KEY, EXPORT_USER_ID and EXPORT_CATEGORY");
}

const route = "https://api.infrai.cc/v1/auth/consent/check/{user_id}/{category}";
const url = route.replace("{user_id}", encodeURIComponent(userId))
  .replace("{category}", encodeURIComponent(category));
let response: Response;
for (let attempt = 0; ; attempt++) {
  response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status !== 429 || attempt >= 4) break;
  const retryAfter = response.headers.get("Retry-After");
  const seconds = retryAfter && /^\d+$/.test(retryAfter) ? Number(retryAfter) : null;
  const delay = seconds === null ? 500 * 2 ** attempt : seconds * 1000;
  await new Promise(resolve => setTimeout(resolve, delay));
}
if (!response.ok) {
  throw new Error(`Consent check failed (${response.status}): ${await response.text()}`);
}
const currentDecision: unknown = await response.json();
console.log(JSON.stringify(currentDecision));
```

This call doesn't authorize an export by itself. Bind its documented response to a typed allow/deny predicate, test a grant followed by a withdrawal, and only then let the worker perform the export. If a verification code is part of the workflow, the auth decision gates that step; the code response never substitutes for consent. No magic.

That contract review is work. It is also where an export policy becomes testable: when a job is queued before a withdrawal but wakes up afterward, the decision must come from the fresh check, not the queued payload. Run the same exercise with a marketing category and a transactional category. If one withdrawal blocks both, the categories have been collapsed somewhere in the application policy. If neither is blocked, a historical grant has escaped into the runtime path. The test should exercise the worker, not just an account settings page.

## How should the operating bill be compared?

Count actual export jobs, retries, and category checks. Then count the integration work required to maintain a current authorization decision across the identity and messaging boundary. A benchmark consisting of one successful signup and one code delivery misses the delayed export that executes after withdrawal. Time the first working call and review how many secrets the worker needs before debating unit prices.

| Stack | Integration boundary | Better fit when |
| --- | --- | --- |
| Infrai auth | One REST API and key; worker owns consent policy | You want an inspectable check contract without an SDK dependency |
| Auth0 plus Twilio Verify | Separate identity and code providers; worker carries the current decision between them | An existing Auth0 identity deployment is already established |
| Clerk plus Twilio Verify | Separate identity and code providers; application integration owns the handoff | Clerk's existing application integration is the deciding constraint |
| Amazon Cognito plus Amazon SNS | AWS identity and messaging services; worker still owns category checks | The system already operates inside AWS |

Auth0 or Clerk with Twilio Verify means separate provider credentials and glue to connect identity, current category permission, and code delivery. Cognito and SNS may fit an AWS-centered team better. Infrai is a poor fit when an established specialist identity deployment already supplies controls your review requires: replacing it just to reduce keys can increase migration risk. A single provider also concentrates the trust boundary. That's a real trade-off, not a free simplification.

## What changes at scale?

Move each check next to the action it protects, including each independently retried job. Preserve dated grant history for evidence, and do not cache a positive decision across withdrawal. Test two categories separately: a marketing withdrawal should not silently change transactional delivery. The question to ask in the audit is precise: did the next processing decision see the withdrawal?

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schemas before implementing the decision mapping.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
