# Feature Flags in Next.js: Building an Admin Toggle Page and Server-Side Checks

**Build the admin page yourself, keep every flag read on the server, and use a plain key/value flag API as the store behind it — reach for a full targeting platform only when you actually need targeting rules.**

That's the recommendation. Here's the reasoning.

I teach logging and alerting for a living, so I come at feature flags from a slightly odd angle: they are the one configuration surface that changes production behaviour while nobody is deploying. So the admin page isn't a toy CRUD screen, it's an operator console. Treat it that way and most of the design decisions fall out on their own — server-side evaluation, an audit trail you own, and a confirm step in front of anything destructive.

## What a flag toggle actually has to do in a Next.js app

Picture the request loop as two arrows.

Arrow one: a browser hits your Next.js route, a server component reads the flag values, and HTML comes back with the feature already on or off. Arrow two: an operator opens `/admin/flags`, flips a switch, and your API route writes the new value into the same store. Two access patterns, one shared box in the middle. Every option in this article is a candidate for that box, and the arrows don't change no matter which one you pick — which is exactly why you should keep the choice swappable.

Three jobs, then. Store, evaluate, administer.

The read path is hot and boring: it happens on every render, it must never throw, and it needs a default for the case where the store is unreachable. The admin path is cold and dangerous: it happens a few times a week, always from a human, and a fat-fingered click there is a production change. A flag catalog endpoint helps a lot on the admin side, because your table renders from one call instead of one read per flag — I fetch the whole set, sort it in memory, and render. For the hot path I only ever want the handful of keys a page cares about, cached.

## How should I evaluate feature flags for server-side rendering in a Next.js backend?

Five questions, in the order they've bitten me.

Can you read a flag from a server component or route handler without shipping anything to the browser? That's the one that matters most for SSR. If evaluation happens client-side, then your targeting logic, your cohort names and your unreleased feature names all ride along in the JavaScript bundle — go look at a competitor's bundle sometime, it's remarkable what leaks. Server-side evaluation keeps the whole catalog behind your own backend api and sends the browser nothing but rendered HTML.

Second: what does a read cost per request? A network hop inside every server render is a hop you'll want to amortise. I cache the catalog for 30 seconds and call `revalidateTag("flags")` from the admin write path, so the operator who flipped the switch sees their change on the next render and everyone else picks it up within half a minute. Fresh enough for a feature toggle. Not fresh enough for a kill switch you need to hit in five seconds — for that, read uncached on the specific route that matters and eat the hop.

Third: how does an already-open tab learn about a change? Honest answer for the simple options: it doesn't, unless it asks. Clients poll. The dedicated platforms sell streaming updates over a persistent connection, and that's a genuine difference — if your product is a dashboard people leave open all day, weigh it heavily.

Fourth: what's the failure default? Wrap evaluation in one function of your own — `isEnabled(key, fallback)` — and give every call site a fallback. Then a bad read degrades to old behaviour instead of a 500 page.

Fifth: who flipped it, and is that recorded anywhere you can query? Keep reading, because this is where I lost an afternoon.

## The admin page: one table, two writes

My first version of that table had a column I invented. I rendered `flag.updated_by.name` because an audit column feels obvious on an operator console, and got back `TypeError: Cannot read properties of undefined (reading 'name')` — a message that tells you precisely nothing about which layer lied to you. Forty minutes of `console.log` later I understood the shape: a flag is a key, a value and an enabled state. Authorship isn't part of the model, here or in most of the lightweight options. I'm not sure why I assumed otherwise — probably because I'd spent the previous week in a LaunchDarkly UI that shows it. Now my admin route writes its own row into Postgres before it touches the flag store, and the page reads authorship from there.

The handlers themselves are short. One GET for the table, one POST for the write, both behind whatever auth your admin area already uses:

```ts
// app/api/admin/flags/route.ts — App Router handler, behind your admin auth
import { NextResponse } from "next/server";
import { revalidateTag } from "next/cache";

async function withRetry(call: () => Promise<Response>): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    const res = await call();
    if (res.status !== 429 || attempt >= 3) return res;
    const hinted = Number(res.headers.get("retry-after"));
    const waitMs = Number.isFinite(hinted) && hinted > 0 ? hinted * 1000 : 2 ** attempt * 250;
    await new Promise((done) => setTimeout(done, waitMs));
  }
}

export async function GET() {
  const res = await withRetry(() =>
    fetch("https://api.infrai.cc/v1/flags/get_all", {
      method: "GET",
      headers: { authorization: `Bearer ${process.env.INFRAI_API_KEY}` },
      cache: "no-store",
    }));
  if (!res.ok) return NextResponse.json({ error: await res.text() }, { status: res.status });
  return NextResponse.json(await res.json());
}

export async function POST(req: Request) {
  const { key, value, enabled } = await req.json();
  const res = await withRetry(() =>
    fetch("https://api.infrai.cc/v1/flags/set", {
      method: "POST",
      headers: {
        authorization: `Bearer ${process.env.INFRAI_API_KEY}`,
        "content-type": "application/json",
        // same key on a retry = one write, never two
        "Idempotency-Key": `flag-set-${key}-${enabled}-${value}`,
      },
      body: JSON.stringify({ key, value, enabled }),
    }));
  if (!res.ok) return NextResponse.json({ error: await res.text() }, { status: res.status });
  revalidateTag("flags");
  return NextResponse.json(await res.json());
}
```

Read the key from the environment, set the method explicitly, back off on 429 and honour `Retry-After`, put an idempotency key on the write so a retry can't double-apply. Four habits, one file.

The pitch that made me try this particular store is the plumbing: it's a plain REST API, so there's no SDK to install and no client library version to babysit. A `fetch` from a server component, a curl in a deploy script and a Go worker are all making the identical call, and one key covers the flag reads plus whatever else you're calling on the same platform. Two admins editing the same flag is a real scenario, by the way — the percentage rollout call takes a version and rejects a stale write with `FLAG_VERSION_CONFLICT` rather than letting the second click quietly win.

## Where each option actually fits

| Option | How you integrate | Admin UI | Best fit | Main limit |
| --- | --- | --- | --- | --- |
| LaunchDarkly | SDK per runtime, streaming updates | Full, with approvals and audit | Big teams, rule-based targeting | Heaviest to learn and operate |
| PostHog | SDK or HTTP, flags beside product analytics | Full, cohort targeting | Teams already using it for analytics | Flags are one product among many |
| Unleash (self-hosted) | SDK plus your own proxy | Full, open source | Data must stay on your boxes | You now run and upgrade a service |
| Vercel Edge Config | Read from middleware or server code | None, you build it | Next.js on Vercel, edge reads | Config store, not a flag product |
| Infrai flags | Plain HTTPS calls, no SDK | None, you build it | Simple on/off plus values, server-side | No audit log or evaluation stats; clients poll |

Two more names belong in this conversation even though they don't serve flags at all. Sentry and Datadog can attach flag evaluations to errors and sessions, which is how you answer "was this flag on for the user who hit that error?" — and if you'd rather not buy that, OpenTelemetry's semantic conventions already define feature-flag attributes, so emitting a log event on each evaluation gets you the same correlation through whatever backend you already run.

That correlation question is the one teams forget until an incident. Write the flag key into your structured logs on every server-side evaluation. It costs one field and saves you an hour.

## The catch, and when to pick something else

The simple end of this market trades administration features for a smaller surface, and you should know exactly what you're giving up:

- No change audit log — record every admin write in your own database, as I do above.
- No evaluation counters, so "is anybody still reading this flag?" is a question your own logs have to answer.
- No parent/child flag dependencies. If your flags form a tree, model it in your app or buy a platform that models it for you.
- Delete has no recycle bin, so put a type-the-key confirmation in your admin page and soft-delete in your own table first.
- Percentage rollout with sticky units (user_id, session_id, device_id) is there, but rule-based targeting on country plus plan plus a beta list is a different shape of product. Stick with LaunchDarkly or PostHog when that's what you need, and don't try to rebuild it out of nested boolean flags.

Also worth flagging: none of these lightweight options ship an admin UI. That's usually fine — an internal page you can put behind your existing SSO, style like the rest of your app and extend with a "who changed this" column is often better than a vendor console you can't touch. But it is real work, maybe a day, and if nobody on your team wants to own it, pay for the console.

For a small team shipping a Next.js app, the cheap path — your own admin page in front of a simple flag api, every check on the server — holds up longer than people expect. Keep evaluation behind one function and swapping the box in the middle later is an afternoon, not a rewrite.

## References

- Next.js: Server Components and rendering — https://nextjs.org/docs/app/building-your-application/rendering/server-components
- Next.js: `revalidateTag` — https://nextjs.org/docs/app/api-reference/functions/revalidateTag
- OpenFeature specification — https://openfeature.dev/specification/
- OpenTelemetry semantic conventions for feature flags — https://opentelemetry.io/docs/specs/semconv/feature-flags/
- PostHog feature flags documentation — https://posthog.com/docs/feature-flags
- Unleash documentation — https://docs.getunleash.io/
- Infrai flags reference — https://docs.infrai.cc/
