# How to delete old records nightly in Node.js without stalling the cleanup queue

Pick the delivery guarantee before you pick the scheduler. For a nightly data cleanup in Node.js — the sweep that deletes old records and their attachments in a property management platform — at-least-once delivery with idempotent deletes is almost always the right answer, because running a delete twice costs one wasted HTTP call, while dropping one silently leaves a tenant application alive months past the retention window you promised in a contract. Cron, an HTTP endpoint and a queue worker are three answers to the question of *when* the work starts. None of them answers whether the work finishes.

That second question is where cleanup jobs die.

Here's the system I'll use throughout, because the shape matters more than the vendor. A property management back office holds work orders, inspection reports and applicant records. Photos live in object storage behind an API that allows roughly 20 requests per second and answers 429 when you exceed it. Every night, a sweep finds records past their retention date, deletes the photos, then marks the rows purged. On a quiet night that's 3,000 records. After a busy leasing month it's 90,000, and the whole thing has to drain through the same narrow rate-limited pipe.

## Start from the guarantee, then choose the trigger

Every option below can start a job at 02:15. They differ in what happens when the process is killed at 02:16, which is the only property worth choosing on.

| Trigger | Delivery guarantee | Pick this when | Main failure mode |
| --- | --- | --- | --- |
| OS cron / systemd timer | At most once, no retry, no visibility | The task is idempotent, cheap, and skipping a night is survivable | Silent skips; overlapping runs when a job outlives its interval |
| HTTP endpoint called by an external scheduler | At least once at the trigger, nothing downstream | Your app already runs behind HTTP and you want retries and logs from the caller | A long-running request hits a proxy timeout and gets retried while still running |
| Durable queue + worker pool | At least once per record, with redelivery and a dead-letter path | Work is large, rate-limited, or partially retryable — deleting 90,000 records | Poison messages looping forever if you skip the dead-letter queue |

Read that table as one sentence: the trigger buys you a start time, the queue buys you a finish. A nightly cleanup that touches thousands of rate-limited resources needs both, and mixing them is normal rather than over-engineering.

## Should a nightly cleanup job run on cron, an HTTP endpoint, or a queue worker?

Use plain cron when the whole job is a single statement your database can finish inside its own transaction — `DELETE FROM audit_events WHERE created_at < now() - interval '90 days'` on a table with an index that supports it. One process, one connection, no fan-out. The catch is that cron has no memory: if the box was rebooting at 02:15, nothing happened and nothing told you. Add a heartbeat you check in the morning, or don't use cron for anything you'd be embarrassed to explain in a compliance review.

An HTTP endpoint is the right trigger once the work belongs to your application rather than to a machine. The scheduler — an external cron service, a CI schedule, a platform trigger — sends an authenticated POST and your app decides what to do. You get retries, request logs and the ability to run the same job manually with `curl` during an incident. The trap is treating the request as the job. Anything that takes minutes will eventually meet a 60-second proxy timeout, the caller will retry, and now two sweeps are running against the same rows. Answer 202 immediately, do the work behind the response, and make the run idempotent with a claim row.

A queue with a worker pool is what you graduate to when the unit of work stops being "the night" and starts being "the record". Each record becomes a message. A crashed worker doesn't lose 90,000 deletions, it loses the handful of messages that were in flight, and those come back after their visibility timeout expires. This is also the only one of the three that gives you a natural place to park permanently broken work: a dead-letter queue, which AWS documents as the standard way to isolate messages a consumer can't process rather than letting them block the pipe.

Not every cleanup deserves this. A 20-row-a-night purge inside a single transaction is not a good fit for a queue — stick with the statement your database already runs well.

## Draining a rate-limited pool without dropping records

The end-to-end path is worth drawing before any code:

```text
cron tick (02:15)
  -> POST /internal/jobs/cleanup/run        returns 202 in ~5 ms
       -> sweep: select ids where purge_after < now() limit 5000
            -> enqueue one message per record        (at least once)
                 -> pool of 8 workers, one shared 20 req/s budget
                      -> 429 ? pause the budget, put the message back
                      -> 5 failed attempts ? dead-letter queue
```

Everything above the pool is boring, and it should be. The trigger's whole job is to start exactly one sweep per night and get out of the way.

```ts
// POST /internal/jobs/cleanup/run — the only thing the scheduler knows how to call.
app.post("/internal/jobs/cleanup/run", async (req, res) => {
  if (req.get("x-cleanup-token") !== process.env.CLEANUP_TOKEN) return res.sendStatus(401);

  const runId = req.get("x-run-id") ?? crypto.randomUUID();
  // Unique index on (job_name, run_date) makes a retried trigger a no-op instead of a second sweep.
  const claimed = await db.claimRun("nightly-cleanup", runId);
  if (!claimed) return res.status(200).json({ status: "already_running", runId });

  res.status(202).json({ status: "accepted", runId });
  void enqueueExpiredRecords(runId);
});
```

The interesting part is the pool. Eight workers pulling from one queue will happily send 8 concurrent deletes, so concurrency and rate need to be separate knobs: a shared budget decides how fast, the pool size decides how many long-running jobs overlap.

```ts
import { setTimeout as sleep } from "node:timers/promises";
import { db } from "./db.ts";
import type { Queue, CleanupJob } from "./queue.ts";

const PERMITS_PER_SECOND = 20;   // what the photo API allows this tenant
const MAX_ATTEMPTS = 5;
const POOL_SIZE = 8;

class Budget {
  private tokens = PERMITS_PER_SECOND;
  private pausedUntil = 0;

  constructor() {
    setInterval(() => { this.tokens = PERMITS_PER_SECOND; }, 1000).unref();
  }

  async take(): Promise<void> {
    for (;;) {
      const wait = this.pausedUntil - Date.now();
      if (wait > 0) { await sleep(wait); continue; }
      if (this.tokens > 0) { this.tokens -= 1; return; }
      await sleep(50);
    }
  }

  pause(seconds: number): void {
    this.pausedUntil = Math.max(this.pausedUntil, Date.now() + seconds * 1000);
  }
}

const budget = new Budget();

async function handle(job: CleanupJob, queue: Queue): Promise<void> {
  await budget.take();
  const res = await fetch(`https://photos.internal/objects/${job.photoKey}`, { method: "DELETE" });

  if (res.status === 429) {
    // Retry-After is the server telling you the drain rate. Believe it.
    const retryAfter = Number(res.headers.get("retry-after") ?? 5);
    budget.pause(retryAfter);
    await queue.release(job, retryAfter * 1000);
    return;
  }

  // 404 means an earlier attempt already removed it — expected under at-least-once delivery.
  if (res.ok || res.status === 404) {
    await db.markPurged(job.recordId);
    await queue.ack(job);
    return;
  }

  if (job.attempts + 1 >= MAX_ATTEMPTS) {
    await queue.deadLetter(job, `photo delete ${res.status}`);
    return;
  }
  await queue.release(job, 2 ** job.attempts * 1000);
}

export async function drain(queue: Queue, deadline: number): Promise<void> {
  const inFlight = new Set<Promise<unknown>>();

  while (Date.now() < deadline) {
    if (inFlight.size >= POOL_SIZE) { await Promise.race(inFlight); continue; }
    const job = await queue.reserve(30_000);   // 30s visibility timeout
    if (!job) break;                           // nothing left to drain
    const p = handle(job, queue).finally(() => inFlight.delete(p));
    inFlight.add(p);
  }

  await Promise.allSettled(inFlight);
}
```

Three details in there carry the whole design. The 429 branch pauses the shared budget instead of sleeping one worker, so a rate limit slows the pool rather than turning eight workers into eight independent retry storms — and it honours `Retry-After`, which MDN describes as either a delay in seconds or an HTTP date, so parse it defensively rather than assuming an integer. Treating 404 as success is what makes the delete idempotent, and idempotence is what earns you the right to choose at-least-once delivery in the first place. And the attempt counter with a dead-letter path is the difference between a queue that drains and a queue that spins: one record with a corrupted key can otherwise consume your entire budget every night, forever, while the other 89,999 wait.

## What actually breaks at 02:15

Overlapping runs are the first thing you'll hit. The sweep that took 40 minutes in March takes 5 hours in September, the next tick fires while it's still going, and two pools now share one rate budget — so each gets half, and the job that was already late gets later. The claim row in the endpoint above prevents that at the trigger, but you also want the drain loop to stop at a deadline it was given rather than run until it's done, so a slow night degrades into "finished 80% of the work" instead of colliding with the morning traffic peak.

Deploys are the second. A worker holding 8 in-flight deletes that gets `SIGTERM` should stop reserving, finish what it holds, and exit — usually 10 to 30 seconds. Anything it can't finish stays invisible only until the visibility timeout expires, then another worker picks it up. That is the redelivery you paid for.

Observability for this kind of job is not "did it log". Track four numbers: queue depth at the start and end of the run, drain rate in records per minute, count of 429 responses, and dead-letter depth. The first two tell you whether tonight's run will fit in the window before it doesn't; the third tells you whether your budget matches reality; the fourth is the only one that should page anyone. A dead-letter count that goes from 0 to 40 is a schema change or a permissions change, and it's worth catching at 08:00 rather than at the next audit.

Testing is duller than it sounds and worth doing anyway. Point the worker at a fake API that returns 429 for the first 50 calls, run the drain with a 10-second deadline, and assert that nothing was acked without a delete and that the dead-letter queue is empty.

## Limits of this design

At-least-once plus idempotent deletes buys reliability and gives up ordering: messages come back out of order after a redelivery, so anything that needs a strict sequence needs a different tool. There's no exactly-once here either, and I'd be suspicious of anyone selling it to you over an HTTP API you don't control — what you get is "effectively once" through idempotence, which is enough for deletes and not enough for, say, charging a card. Queues also cost you a moving part, a dashboard and an on-call story. If your nightly cleanup is one indexed statement that finishes in 200 ms, none of this applies and adding it would be the mistake.

## Sources

- MDN: HTTP 429 Too Many Requests — https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429
- AWS SQS dead-letter queues documentation — https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
