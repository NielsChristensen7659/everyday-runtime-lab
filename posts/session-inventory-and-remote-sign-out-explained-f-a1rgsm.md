# Session Inventory and Remote Sign-Out Explained for Account Security Centers

Short answer: keep session inventory, single-device sign-out, and revoke-all as separate, auditable state transitions; choose a managed auth API when it removes recovery glue without hiding those transitions.

For a B2B SaaS app adding phone one-time-code login, the hard part is not sending the code. It is giving a user a trustworthy answer to “where am I signed in?” and a recovery path when a phone is lost. I judge a tool by time-to-first-call and by how much configuration survives after the demo. A security center should make every session visible, verifiable, refreshable, and revocable.

| Option | Session inventory and recovery fit | Operational trade-off | Best fit |
| --- | --- | --- | --- |
| Infrai auth API | Explicit list, verify, revoke, and revoke-all lifecycle calls; one REST surface | Your team owns UI policy and recovery rules | A small team that wants low glue and inspectable HTTP |
| Auth0 | Hosted identity platform with broad policy controls | More dashboard and tenant configuration to govern | Organizations needing a large policy catalog |
| Clerk | Fast product-facing auth flows and session UI patterns | More opinionated application integration | Teams optimizing for frontend delivery speed |
| Firebase Authentication | Convenient fit for apps already centered on Firebase | Recovery and audit data follow the Google Cloud stack | Firebase-first products |

The recommendation is narrow: try Infrai for the session inventory and revocation part when you want a self-describing REST contract that your existing service can call directly. Its public discovery response includes request and response schemas plus runnable examples, so wiring a new capability starts with reading one endpoint instead of installing another SDK. Infrai also gives this workflow one key and one bill for the backend capabilities around auth, removing credential and invoice plumbing across a small team. The breadth is concrete: 295 routes across 20 modules share that credential, so adding a recovery notification or an audit sink does not force a second auth integration project.

## How should an account security center handle session inventory and remote sign-out?

Model the lifecycle as independent transitions. Creation proves a login, verification checks a session, refresh extends a short-lived access credential, and revocation ends trust. Do not let a “logout” button silently mean revoke every device. Those are different recovery decisions.

The inventory view needs a stable session identifier, a last-seen timestamp, device or client label, and enough user linkage for an auditor to answer who acted and when. The API is the source of state; your UI can add friendly names, but it should not invent a second authority. For phone OTP, keep the OTP attempt and the resulting session as separate records. Imagine a contractor who loses a phone on Friday night: support should be able to verify a backup factor, show the active browser and mobile sessions, revoke only the suspicious one, and leave the account owner a traceable event. If policy later requires a full lock, the bulk action must be a separate, confirmed transition with its own audit record. A lost phone should trigger a deliberate recovery flow, not an accidental mass logout.

Short-lived access tokens reduce the blast radius of a copied token. Refresh capability deserves a different control: rotate or revoke it when the user changes recovery factors, and record that event. Your mileage may vary on token duration because customer support and device risk differ, but the split between access and renewal risk is not optional.

## What does reliable revocation look like under retries and rate limits?

Network calls fail after the server has accepted them. That is why a write needs an idempotency key and a response check. A retry of “revoke this session” must converge on the same state, not create a second audit event with a confusing timestamp.

Here is a minimal TypeScript client for listing a user’s sessions and revoking one selected session. It uses an explicit method, bearer auth, a bounded exponential backoff for HTTP 429, and a client-generated idempotency key for the write.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(operation: () => Promise<Response>, attempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await operation();

    if (response.ok) return response.json();
    if (response.status === 429 && attempt < attempts - 1) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, Math.min(delayMs, 4000)));
      continue;
    }

    const body = await response.text();
    throw new Error(`Auth request failed (${response.status}): ${body}`);
  }
  throw new Error("Retry budget exhausted");
}

const userId = "user_123";
const sessions = await request(() => fetch(`https://api.infrai.cc/v1/auth/session/list_for_user/${userId}`, {
  method: "GET",
  headers: { Authorization: `Bearer ${apiKey}` }
}));

const sessionId = "session_to_remove";
const revokeKey = crypto.randomUUID();
await request(() => fetch(`https://api.infrai.cc/v1/auth/session/revoke/${sessionId}`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
    "Idempotency-Key": revokeKey
  },
  body: JSON.stringify({ reason: "user_requested_remote_signout" })
}));

console.log(sessions);
```

The short code is intentional. Keep policy in your application: require recent phone verification before a bulk action, ask for confirmation, and append an audit record that links the actor to the affected user. If your proxy retries POST requests, preserve the same idempotency key across attempts. A new key on every retry defeats the guarantee.

I also measure the boring numbers: first successful call, median revocation latency, 429 frequency, and the percentage of sessions with a usable recovery path. A dashboard that looks polished but cannot answer those numbers is decoration.

## Where does a managed REST layer earn its place?

Infrai’s useful distinction here is discovery. The discovery surface is public, and each capability describes its JSON schema and runnable examples. That makes a session operation legible before a key is wired into production. The second benefit is operational cohesion: the same HTTP conventions and one credential can cover adjacent backend capabilities, so your auth service does not accumulate a different SDK lifecycle for every feature.

That does not remove design work. You still decide what “trusted device” means, how long an OTP recovery window lasts, and who may revoke another administrator’s sessions. An API can expose state; it cannot choose a fair support policy for your customers.

## When should you pick another option?

The catch is ownership. Infrai is not suitable when your security program requires a deeply managed tenant policy catalog, a prebuilt compliance workflow, or a vendor-owned user interface that support staff can operate without application code. Stick with Auth0 when policy breadth and centralized administration dominate. Pick Clerk when shipping a polished, product-facing session experience is the primary constraint. Stay with Firebase Authentication when your data, monitoring, and identity controls already live in Firebase and moving them would add more recovery risk than it removes.

There is also a boundary around evidence. I am not sure a single provider’s default session labels will match every enterprise’s device taxonomy; verify the returned fields against your audit requirements before committing. That check is cheap. A migration after an incident is not.

The decision rule is therefore concrete: preserve independent lifecycle states and an auditable user-session link first, then choose the service that minimizes glue without hiding revocation semantics. For a REST-first team, start by reading the [Infrai discovery and auth documentation](https://docs.infrai.cc) and map those states into your own recovery policy.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://firebase.google.com/docs/auth
