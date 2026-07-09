================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Everything Breaks. The Only Question Is Whether You Can See It.

SUBTITLE
Design failure into the code and instrument it so an outage is a line on a graph, not a phone call at two in the morning.

TAGS   (5 max, each 25 characters or fewer)
SRE, Observability, Reliability, Software Architecture, DevOps

COVER IMAGE   upload this file from the covers folder
covers/07-design-for-failure.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 7 of 9 of The Global Rules Every New Project Should Have.*

Disks die. Dependencies time out. A request crashes mid-flight and takes a half-finished write with it. None of this is an edge case. It's Tuesday. The teams that get burned are the ones who wrote every code path as if the happy one were the only one, and then had no way to see which path actually ran when it all went wrong.

Reliability is not the absence of failure. It's fast, well-practiced recovery from it. And you cannot recover from what you cannot see. These two rules travel together, so I'm covering them together: design for failure, and make it observable.

## Design for failure: plan the recovery, in the code

Assume outages, data loss, and traffic spikes will happen, because they will. The work is not preventing every failure. It's making each one explicit, recoverable, and something you learn from. Four habits carry most of the weight.

### A business failure is data, not an exception

Start with the smallest one, because it sets the tone for everything else. An expected outcome (insufficient funds, receipt too large, a slot already taken) is not exceptional. It's a normal branch of the business. So put it in the return type, where the caller is forced to handle it, and reserve exceptions for genuine bugs. When every error path is part of the contract, no failure gets silently swallowed.

```ts
// DON'T: an expected "insufficient funds" thrown as an exception, easy to forget to catch
function withdraw(a: Account, n: number): Account {
  if (n > a.balance) throw new Error('nsf');
  return a;
}
// DO: the failure is in the return type, so the caller must handle it
function withdraw(a: Account, n: number): Result<Account, 'insufficient_funds'> {
  if (n > a.balance) return { ok: false, error: 'insufficient_funds' };
  return { ok: true, value: { ...a, balance: a.balance - n } };
}
```

The same discipline in Java, where a sealed type makes the failure part of the signature rather than a runtime surprise:

```java
// DON'T: expected failure as an unchecked exception
Account withdraw(Account a, long n) {
  if (n > a.balance()) throw new IllegalStateException("nsf");
  return a;
}
// DO: sealed Result makes the failure part of the contract
Result<Account> withdraw(Account a, long n) {
  if (n > a.balance()) return new Err<>("insufficient_funds");
  return new Ok<>(a.debit(n));
}
```

This is not ceremony. A thrown exception for a routine outcome is a bug waiting for the one caller who forgot the try block. A typed failure cannot be ignored, because the code won't compile until you deal with it.

### Keep read paths explicit

An ORM is a wonderful tool for writes, where its mapping, validation, and safety are exactly what you want. On the hot read path it's a liability, because it will happily hide the query that is the difference between a 20-millisecond page and a 2-second one. The classic trap is a lazy relation walk that fires one query per row, invisible until the day your list endpoint has a thousand rows on it.

```ts
// DON'T: an ORM relation walk that fires one query per receipt on the list endpoint (N+1)
const orgs = await db.query.orgs.findMany({ with: { receipts: { with: { lines: true } } } });
// DO: explicit SQL for the read (see it, EXPLAIN it, tune it); ORM stays for the write
const rows = await db.execute(sql`
  SELECT r.id, r.total_cents, count(l.id) AS line_count
  FROM receipts r LEFT JOIN receipt_lines l ON l.receipt_id = r.id
  WHERE r.org_id = ${orgId}
  GROUP BY r.id ORDER BY r.created_at DESC LIMIT 50`);
await db.insert(receipts).values(newReceipt);   // writes go through the ORM
```

Write your reads by hand so you can see exactly what runs, run EXPLAIN on it, and index for it. Let the ORM own the writes. That split gives you performance you can reason about and safety where it counts.

### Do not fire and forget

Here is the one that quietly loses money. A request commits a change, then makes a best-effort external call: send the email, publish the event, notify the downstream system. If the process dies after the commit but before the call, the change is real and the side effect is gone, with no record it was ever attempted. Nobody notices until a customer asks where their receipt went.

The fix is the transactional outbox. Record the intent in the same transaction as the change, and let a separate worker deliver it with retries.

```ts
// DON'T: fire the email in-request; a crash after commit loses it with no trace
await db.insert(receipts).values(r);
await email.send(r.owner, 'receipt_ready');   // if this throws or the pod dies, it is just gone
// DO: write the intent atomically with the state, a worker delivers it with retries
await db.transaction(async (tx) => {
  await tx.insert(receipts).values(r);
  await tx.insert(outbox).values({ topic: 'receipt_ready', payload: r.id }); // same commit
});
// a separate poller reads unsent outbox rows, sends, marks done, and retries on failure
```

A follow-up that survives a crash beats a best-effort call that vanishes with the process. This one pattern removes an entire category of "it usually works" bugs. I used to treat these background sends as too small to bother about. Every one that silently failed cost me a support thread and an apology, so now they go in the outbox.

There is more to this rule that I won't belabor here: set explicit reliability targets so "reliable enough" becomes a number, not a feeling. Keep backups you have actually restored, because an untested backup is a rumor, and a scheduled restore drill turns disaster day into a rehearsal. Scale with demand by keeping components stateless. And after every incident, run a blameless postmortem that ends in owned, tracked backlog items, because recovery without learning just schedules the next outage. Pick the ones that matter for your system and make them defaults.

## Make it observable: a running system should explain itself

Now the other half. In production the question is never "is something wrong." It's "what is wrong, where, and since when." Without instrumentation you are debugging blind, and the outage always lasts longer than it should.

### Instrument everything under one correlated trace

A bare log line tells you almost nothing across a distributed system. You need to follow a single request end to end, across services, and join its logs, metrics, and traces on one correlation id. Use an open, vendor-neutral standard so you're not married to one tool for the life of the product.

```ts
// DON'T: a bare console log, no correlation, no structure, invisible across services
console.log('charging user', userId);
// DO: a span carries the trace id, so logs and metrics join up in the backend
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('billing');
export const chargeUser = async (userId: string): Promise<void> =>
  tracer.startActiveSpan('charge', async (span) => {
    span.setAttribute('user.id', userId);
    try { await charge(userId); } finally { span.end(); }
  });
```

The difference is not cosmetic. With the span, one trace id ties the frontend click, the API handler, and the database call into a single story you can pull up in seconds. With the console log, you have a string in a file and a long night ahead.

### Watch behavior, not just health

CPU and memory graphs tell you the box is alive. They stay reassuringly green while your checkout funnel quietly collapses. Instrument the product itself: how often a flow starts, how often it completes, where users drop off. The same data serves reliability and product decisions at once.

```ts
// DON'T: infra-only, tells you the box is alive, not whether checkout works
process.memoryUsage().heapUsed;
// DO: a domain counter and a histogram, split by outcome, so drop-off is visible
const started = meter.createCounter('checkout.started');
const completed = meter.createCounter('checkout.completed');
const latency = meter.createHistogram('checkout.latency.ms');
export const recordCheckout = (ok: boolean, ms: number): void => {
  started.add(1);
  if (ok) completed.add(1);
  latency.record(ms, { outcome: ok ? 'ok' : 'abandoned' });
};
```

Now "are users succeeding" is a dashboard, not a guess.

### Alert on what matters, and only what matters

The fastest way to make a pager useless is to make it noisy. A raw CPU threshold fires on every batch job, and within a week the whole team has trained itself to swipe the alert away. So alert on symptoms tied to a user-visible budget: error rate and the P95 or P99 latency, because averages hide the worst experiences. And wait long enough to exclude a blip.

```yaml
# DON'T: raw CPU threshold, no user meaning, fires on every batch job and gets muted
# DO: symptom-based, tied to a budget, waits long enough to exclude a single spike
- alert: CheckoutErrorBudgetBurn
  expr: |
    sum(rate(http_requests_total{route="/checkout",status=~"5.."}[5m]))
      / sum(rate(http_requests_total{route="/checkout"}[5m])) > 0.02
  for: 10m               # sustained, not a single spike
  labels: { severity: page }
```

Keep only alerts that are actionable. Anything that pages twice without prompting a fix gets automated away or deleted. And after an incident slips through, fix what actually broke, not the reason the alert did not fire.

## The payoff

Design failure into the code and your disaster-recovery drill is boring. Instrument the system properly and a sudden spike in traffic is a line on a graph rather than a two-in-the-morning phone call. When something does break, you find the cause in minutes from a dashboard instead of hours of guessing. That's the whole trade: a little discipline up front, in exchange for calm on the day it matters.

You now have a system that fails gracefully and tells you when it does. But who gets to see that dashboard, own that alert, and enforce these rules when nobody is watching? That's governance, and it's next. In Part 8, **Run It in the Open**, I cover the five rules that keep a project honest: no black boxes, clear ownership, pave the road, enforce and verify, and measure whether you are improving.
