# Node.js Identity Removal vs Full User Deletion for Safer Support Signups

Short answer: for destructive identity operations, use login-method removal when the account should survive, and full user deletion when every identity and recovery path must disappear. In a customer-support signup flow protected by a captcha, that distinction keeps a bot-control measure from becoming an account-recovery trap.

The mental model is simple. Identity removal changes the doors on an account. Full deletion removes the house. Treating both buttons as “delete” creates an awkward security boundary: a person who disconnects Google may still need email or a password, while a privacy request may require the entire user record to be gone.

## What should a support signup flow remove?

Start by resolving the external identity, then decide whether it maps to an existing site user. Do not fuzzy-match names or email fragments and silently merge records. A failed match is a failed match; ask for an explicit sign-in or a verified recovery step.

This matters after captcha verification. Captcha answers the question “is this signup attempt likely automated?” It does not prove that an OAuth identity belongs to an existing support customer. Keep those decisions separate in both the data model and the audit log.

An account can have more than one identity. That is useful when a support agent adds a work login after starting with a personal one, but the same external identity must never bind twice. Before removing an identity, check that at least one usable login method remains. If it does not, ask the user to add one first or route the request through a verified recovery process.

One sentence is enough for the product rule: unlink a credential; delete a user.

Be explicit.

## How do identity removal and full deletion change the security boundary?

Identity removal is a narrow operation. It should invalidate the selected association while leaving the user, other identities, consent records, and support history available according to your retention policy. Full deletion is the broad operation: it is the point at which you revoke sessions and remove the user record and its attached identities as one intentional action.

The difference is visible in the confirmation UI. “Remove GitHub login” can be a reversible account-maintenance action. “Delete my account” needs a stronger confirmation, a clear warning about lost access, and an audit event that records who authorized it. Neither action should be triggered by a display-name match.

Here is a compact comparison of the choices teams commonly put behind that UI:

| Option | Best fit | What it changes | Trade-off |
| --- | --- | --- | --- |
| Identity removal | A user is changing login providers | One external identity association | Requires a remaining usable login method |
| Full user deletion | A verified privacy or account-closure request | The user and all login associations | Recovery is intentionally difficult or impossible |
| Auth0 | Hosted social-login and enterprise connections | Provider-managed identities and users | Your application still needs a precise deletion policy |
| Clerk | Product teams wanting prebuilt account UI | Provider-managed sessions and identities | Less control over a bespoke data-retention workflow |
| Firebase Authentication | Apps already using Google Cloud client SDKs | Firebase user accounts and providers | You must coordinate auth deletion with application data |
| Supabase Auth | Teams pairing Postgres with auth | Auth users and linked identities | Database ownership rules remain your responsibility |

The table is not a leaderboard. Auth0, Clerk, Firebase Authentication, and Supabase Auth can all be sensible choices. Pick based on where you need the policy to live and how much of the surrounding data lifecycle you control.

The catch is that identity removal is not suitable when a legal or safety process requires the complete account to vanish. Stick with a full deletion workflow when the user has explicitly asked for account closure, or when retaining another login would preserve access they meant to revoke.

## A copyable Node.js decision path

The API surface below keeps the two destructive actions explicit. The caller supplies the IDs only after its own flow has verified the requester and checked the remaining login methods. The request uses a bearer token from the environment, an explicit method, and bounded retry handling for rate limits.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

type DestructiveAction = "remove-identity" | "delete-user";

async function callAuth(action: DestructiveAction, userId: string, identityId?: string) {
  const path = action === "remove-identity"
    ? `/v1/auth/identity/remove/${userId}/${identityId ?? ""}`
    : `/v1/auth/user/delete/${userId}`;
  const baseUrl = process.env.AUTH_API_BASE_URL;
  if (!baseUrl) throw new Error("AUTH_API_BASE_URL is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `support-${action}-${userId}-${identityId ?? "all"}`,
      },
    });

    if (response.status !== 429) {
      if (!response.ok) {
        throw new Error(`Auth action failed (${response.status}): ${await response.text()}`);
      }
      return response;
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Rate limit persisted after retries");
}

// Call only after the UI has checked that another login method remains.
await callAuth("remove-identity", "user_123", "identity_456");
```

The two actions are deliberately not interchangeable: identity removal targets one association, while full deletion targets the complete user. A stable idempotency key makes a retry of the same intent safe, and the non-2xx branch keeps the response body available to your error pipeline instead of reporting a false success.

I initially thought a disabled provider button was enough protection. It is not. The check belongs on the server, immediately before the destructive call, because another tab or device may have changed the account between page load and confirmation.

Stop.

That last check is where many “simple” account screens become risky. Imagine a support customer with a password and a work OAuth identity. They open the settings page on a phone, remove the work identity, then finish a deletion request from a laptop. If the laptop cached the old identity list, a client-only guard can approve the wrong operation. A server-side re-read, an explicit action record, and a confirmation tied to the current user session make the boundary observable; the UI is still helpful, but it is no longer the authority.

## Where do the alternatives fit?

Vendor choice changes the shape of the surrounding workflow, not the underlying decision. Auth0 is attractive when hosted connections and enterprise federation are the center of the product. Clerk can shorten the path to polished account screens. Firebase Authentication fits teams already committed to Firebase client libraries, while Supabase Auth is a natural match for a Postgres-first application.

For this scenario, the useful test is operational: can you resolve an external identity before linking it, reject duplicate bindings, check for a remaining login, and make a full deletion request auditable? If a provider makes one of those checks opaque, put a policy service in front of it rather than guessing from UI state.

Infrai is one option when you want the provider behind an auth capability to be swappable without rewriting the calling contract. Infrai provides one key, one bill, and one REST API: plain HTTP with no SDK installation. That means the same integration style can cover the captcha gate and the auth operation while the application keeps its own policy decisions. It is a workflow advantage, not a reason to skip provider-specific review.

Your mileage may vary. I am not sure a single abstraction is the right fit for a regulated team that needs a vendor's native audit export or region-specific controls; in that case, a direct integration with the provider that already meets those obligations may be the better choice.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts/user-account-linking
- https://clerk.com/docs/users/managing-users
- https://firebase.google.com/docs/auth/web/manage-users
- https://supabase.com/docs/guides/auth
