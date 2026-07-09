================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
The Most Expensive Software Is the Kind Nobody Needed

SUBTITLE
You can satisfy every engineering rule and still build the wrong thing beautifully. Two rules make sure you don't.

TAGS   (5 max, each 25 characters or fewer)
Product Management, Startups, User Experience, Product Strategy, Validation

COVER IMAGE   upload this file from the covers folder
covers/09-build-the-right-thing.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 9 of 9 of The Global Rules Every New Project Should Have.*

Eight parts of this series were about building the thing right. Every one of them assumed the thing was worth building. This part checks that assumption, because it's the one that costs the most when it's wrong.

The most expensive software is not the badly built kind. It's the beautifully built kind that nobody needed. I've watched more effort die that way than to any bug. Almost every rule in this series makes you good at building the thing right. None of them checks that it's the right thing, or that the person on the other side has a good time using it. Those two gaps are the last two rules: obsess over the whole experience, and validate before you build.

## Obsess over the whole experience: the product is more than the interface

You can pass every gate in this series and still ship something people quietly abandon, because none of those gates is about the person on the other side. The experience your user has is the whole thing. Not just the screen, but how fast it responds, how little it asks of them, how it treats a payment and a mistake, and whether support treats them like a person. Care about that as much as you care about the code.

Start with the moment things go wrong, because that's where trust is won or lost. A raw status code dumped at a user is a dead end. "Error 413" means nothing to an employee filing an expense from the back of a taxi. Name the cause and the next step, in the product's voice, and keep the machine-readable shape stable behind it so the client can branch on a code rather than parse prose.

```tsx
// DON'T: dumps the transport failure at the person, who can do nothing with it
function Error({ status }: { status: number }) {
  return <p>Error {status}</p>;  // "Error 413" means nothing to an employee in a taxi
}
// DO: name the cause and the next step, in the app's voice; string comes from the catalog
function ReceiptTooLarge({ t }: { t: Translate }) {
  return (
    <Callout tone="pending">
      {/* t("receipt.tooLarge") => "This photo is over 10 MB. Retake it or shrink it." */}
      {t("receipt.tooLarge")}
    </Callout>
  );
}
// The API shape behind it stays stable and typed, so the client branches on `code`, not prose:
//   { "error": { "code": "receipt_too_large", "maxBytes": 10485760 } }
```

That small difference is the difference between a user who fixes the problem and one who gives up and files a support ticket about your app being broken.

The same principle runs through the rest of this pillar. Earn trust rather than extract a sale: make cancelling as easy as subscribing, because a self-serve exit that mirrors the signup keeps the relationship clean, while a dark pattern that forces a phone call to cancel extracts a month of resentment, not a renewal.

```java
// DON'T: no cancel endpoint at all, so the only exit is a support ticket queue
// DO: a first-class, self-serve cancel that mirrors how they signed up
@POST @Path("/subscription/cancel")
public Response cancel(@Context OrgContext ctx) {
    var ends = billing.cancelAtPeriodEnd(ctx.orgId());  // honest: no clawback of paid time
    return Response.ok(new CancelResult(ends)).build();
}
```

Being honest with users, even when it costs a conversion, is what turns them into people who come back. And let technology serve the person, not replace them: automate to remove friction, but always leave a visible path to a human. A bot that loops with no exit is not support, it's a trap. The human touch is the part hardest for a competitor to copy, so protect it.

## Validate before you build: make sure it is worth building at all

I listed this rule last, but in time it comes first, before a single line of any of the others above. Every other pillar makes you good at building the thing right. This one checks that it's the right thing, and that check is your job before the project starts and again at every fork along the way.

The discipline has three moves. First, talk to real users before you write code. Not about your idea, which harvests polite yeses, but about their actual problem, the last time it happened, in their own words.

```md
<!-- research/problem-interview.md -- validate the problem, not your solution -->
DON'T: open with "would you use an app that auto-fills expense reports?"
  a leading question about your idea harvests polite yeses, not evidence.

DO: ask about the last real occurrence of the problem, in their words.
  1. "Walk me through the last expense you filed. What did you actually do?"
  2. "Where did that get annoying or slow?"        (find the pain, unprompted)
  3. "What did you do to work around it?"           (proof it hurts enough to act)
  4. "How often does this happen to you?"           (frequency = size of the wound)
```

Ten of those before a line of feature code. Listen for pain you did not invent.

Second, test demand with the cheapest thing that can carry it. A landing page, a mockup, a concierge version you run by hand. Put something in front of real people and watch what they do, not what they say they would do. A signup with an email attached is a vote; an opinion in a meeting is not.

Third, and this is the move most teams skip, set a dated, honest go/no-go. Before committing to build, write down what would make this worth pursuing, then decide, on the record, to proceed or stop. A "no" is a cheap win here and an expensive one later.

```md
<!-- docs/go-no-go-capture.md -- a decision with a date, not a vibe -->
# Go / No-Go: auto-prefill capture -- decided 2026-07-06
- [x] >= 10 problem interviews surfaced this pain unprompted        (met: 12)
- [x] landing page conversion >= 8 percent                          (met: 11 percent)
- [ ] >= 3 SMEs committed to a paid pilot                           (NOT met: 1)
Decision: NO-GO. Re-decide by 2026-08-01 after 2 more pilot conversations.
Owner: product. The unmet criterion is the reason, written down, not smoothed over.
```

Look at that verdict. Two of three criteria met, and the answer is still no, because the one that was not met is the one that mattered. That's what an honest gate looks like. It's allowed to say no even when the momentum wants a yes.

Keep validating after launch, too. Shipping is an output; adoption is the outcome, and only one pays rent. Gate a new feature behind a flag, measure real usage against a threshold, and keep or kill it on the evidence rather than assuming that a feature which shipped is a feature that gets used.

```ts
// DON'T: launch, delete the flag, move on, and never check if anyone adopted it.
// DO: keep the flag as a kill switch and gate keep/kill on a measured threshold.
if (flags.isEnabled("capture-prefill-v2", { orgId })) {
  analytics.track("prefill_used", { orgId });  // adoption signal, per real org
}
// A scheduled check reads the metric and forces the decision:
//   adoption(prefill_used, last=28d) < 0.05  =>  disable the flag, write the reason,
//   remove the code. no silent zombie features left half-live in the product.
```

The bravest thing a team can do is kill a plausible idea before it turns into code. If you've never done it, you're not validating, you're rationalizing.

## The stack was always a detail

That is the eighteen. Look back at the whole map and notice what is missing: not one of these rules names a language, a database, or a cloud. That was deliberate from the first line of Part 1. The tools date. These rules have not, because they describe how a team works rather than what it works with.

So a new project's first decision was never what to build with. It was whether the thing is worth building at all, and then how to work. Consistently and simply, on clean boundaries, with proof instead of hope. Secure, private, and isolated by default. Boring to deliver, running on as little as you operate yourself. Ready to fail and recover, observable, transparent to everyone who depends on it, clearly owned, paved so the right way is the easy way, held to standards a machine enforces and independently verifies, measured against a shared yardstick, and obsessed over from the user's side. Agree these eighteen before you argue about the stack, and the stack becomes a detail you can change later without drama.

The encouraging part, and I'll end on it because it's the part people get wrong, is that none of this requires a large team or a big budget. It requires one decision on day one: that quality, security, reliability, and transparency are defaults, not features you'll get to when there's time. Because there's never time later. The team that will add tests once things settle down is the same team explaining an outage six months in, and the settling never came.

So here is the call to action, and it's simple. Don't adopt eighteen rules this afternoon. Pick the three from this series that your current project most obviously lacks, and make them defaults this week. Write the CODEOWNERS file. Wire the pre-commit gate. Book the ten interviews before the next build. Then pick three more. Start now, on day one of whatever you're building, because every one of these is far harder to add later than to design in from the start.

That is the whole series. Now go build the right thing, right.
