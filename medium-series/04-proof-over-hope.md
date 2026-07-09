================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Nothing Is Done Until a Machine Says So

SUBTITLE
Why "it works on my machine" is not evidence, and what to build instead so a pipeline, not a nervous human, decides what ships.

TAGS   (5 max, each 25 characters or fewer)
Software Testing, Test Automation, Best Practices, Programming, Quality Assurance

COVER IMAGE   upload this file from the covers folder
covers/04-proof-over-hope.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 4 of 9 of The Global Rules Every New Project Should Have.*

"It works on my machine" is the most expensive sentence in software. It is not evidence, it is a feeling, and feelings do not survive contact with production. I have sat in enough post-incident reviews to know the shape of the story before anyone finishes telling it: a path nobody tested, a case nobody thought about, surfacing at the worst possible moment in front of the worst possible audience.

The fix is not more heroics. It is a rule you can say in one line. **Nothing is done until it is verified.** Confidence in software should come from automated checks that anyone can run, not from the optimism of whoever wrote the last change. Every untested path is a liability that will surface eventually, and it will pick the time, not you.

This is the fourth of the eighteen global rules, and it is the one that lets you sleep. Here is what it looks like in code.

## Test in layers, and keep the fast ones fast

Unit tests, integration tests, and end-to-end tests each catch what the others miss. Performance tests catch what all of them ignore. That is not a nice-to-have taxonomy, it is a division of labor. A layer that tries to do everything does nothing well.

The most common mistake I see is pushing every check up into slow end-to-end tests, because that is where the code "obviously" runs. So a pure arithmetic rule gets verified through a full HTTP round trip. It is slow, and when it breaks it tells you nothing about where.

```ts
// DON'T: a pure calculation verified only through a full HTTP round trip
test("discount via API", async () => {
  const res = await app.request("/checkout", { method: "POST", body: cartJson });
  expect((await res.json()).total).toBe(80); // slow, and hides where a break is
});
// DO: the rule at the unit layer, the wiring at the integration layer, each owns its risk
test("premium discount (unit)", () => expect(withDiscount(100, "premium")).toBe(80));
test("POST /checkout returns 200 (integration)", async () => {
  const res = await app.request("/checkout", { method: "POST", body: cartJson });
  expect(res.status).toBe(200);
});
```

The unit test owns the business rule. The integration test owns the wiring. When the rule breaks, exactly one test goes red and it names the problem.

That only works if the fast layer stays fast. A unit test should run in milliseconds, so the whole suite is something you hit constantly rather than a coffee break. The tell: a unit test that runs slow is almost always reaching for a database or a network it should be faking.

```ts
// DON'T: hits Postgres, so the "unit" test takes 300ms and needs a running DB
test("total", async () => { const o = await db.query.orders.findFirst(); expect(total(o)).toBe(80); });
// DO: pure input, runs in under a millisecond, no I/O
test("premium customer gets 20% off", () => {
  expect(total({ price: 100, tier: "premium" })).toBe(80);
});
```

The difference between 300 milliseconds and one millisecond does not sound like much until you have two thousand tests. Then it is the difference between a suite you run on every save and a suite you run once, reluctantly, before you push. The one you run constantly is the one that actually protects you.

## Make coverage mean something

This is where most teams fool themselves. Line coverage is easy to fake and reassuring to look at, which is a dangerous combination. You can hit 100 percent of your lines and assert nothing, and the number will smile back at you while your suite catches exactly zero real defects.

```ts
// DON'T: 100% line coverage, zero assertions, every mutant survives
test("covers total", () => {
  total({ price: 100, tier: "premium" }); // line is hit, nothing is checked
});
// DO: assert the outcome so Stryker's mutants (e.g. 0.8 -> 0.9) get killed
test("premium discount is exactly 20%", () => {
  expect(total({ price: 100, tier: "premium" })).toBe(80);
});
// stryker.config.json sets a thresholds.break so a low score fails CI
```

The metric that actually matters is mutation testing. A mutation tool deliberately breaks your code, one small change at a time (turn a `0.8` into a `0.9`, flip a `>` to a `>=`, delete a line) and then runs your suite. If your tests still pass, the mutant "survived," which means that break could ship and nothing would catch it. Stryker does this for TypeScript, PIT for Java.

Hold your core logic to a near-total bar and glue code to a looser one. When a gate is hard to pass, restructure the code rather than lower the threshold. A rule that is hard to test is usually telling you something true about the design.

## Test behavior, not internals

The last mistake is the subtlest, because it produces green tests that feel thorough and are actually a trap. You write a test that asserts *how* the code works: which method it called, in what order. Then someone refactors the code without changing what it does, and every one of those tests goes red for no reason. You have coupled your tests to the implementation, so the tests now fight the exact refactoring they were supposed to make safe.

Point a test at what a feature should do, described in the language of the domain. Then you can restructure freely.

```ts
// DON'T: asserts an internal call and effectively tests that Drizzle was invoked
test("register calls insert", async () => {
  const spy = jest.spyOn(db, "insert");
  await register(input); expect(spy).toHaveBeenCalled(); // couples to implementation
});
// DO: a hand-written fake, assert the observable result through the port
test("register persists the user", async () => {
  const store = new MemoryUserStore();
  const res = await register(store, { email: "a@b.co" });
  expect(res.ok).toBe(true);
  expect(store.byEmail("a@b.co")).toBeDefined();
});
```

Notice the second version uses a hand-written fake, a `MemoryUserStore`, not a mocking framework. This is deliberate. Prefer simple stand-ins for external systems over heavy mocking that only re-asserts your own call sequence. A fake behaves like the real thing (you can put a user in and read it back out), so the test checks a real outcome. A mock only records that a method was called, which proves nothing a user would ever notice. And never write a test whose real claim is that a third-party library works. That is their job, not yours.

The same principle is even sharper on the frontend, because there is a genuine user to point at. Do not reach inside a component to inspect its state. Interact with it the way a person would and assert on what they would see.

```tsx
// DON'T: assert on internal state and implementation; it breaks on any refactor and proves nothing a user sees
expect(wrapper.find(Counter).state('count')).toBe(1);
// DO: interact and assert on what the user sees (React Testing Library)
render(<Counter />);
await userEvent.click(screen.getByRole('button', { name: /increment/i }));
expect(screen.getByText('Count: 1')).toBeVisible();
```

The first test knows the component has a `count` state. Rename that field, split the component, move to a reducer, and it breaks, even though the button still works perfectly. The second test clicks the button and reads the screen, exactly like a user. It survives every refactor that keeps the behavior, and it fails only when a real person would see something wrong. That is the whole point of a test.

## Gate every merge, and turn every bug into a test

Two habits turn all of this from aspiration into fact.

First, a testing philosophy the whole team follows. Test-first is one option. A simpler rule that works on any team: every bug you fix becomes a permanent test that would have caught it. Fix the empty-cart crash, then write the test that goes red without your fix and green with it. Now that specific defect can never return unnoticed, because the suite remembers it long after everyone who lived through the incident has forgotten.

Second, gate every merge. Static analysis and the full suite run on every change, and nothing merges with a critical defect or a red pipeline. Not "we usually run it." The machine runs it, and the machine blocks the merge. A test suite that can be skipped is a suggestion, and suggestions decay.

```ts
// DON'T: the workflow ignores test failures, so red merges anyway
// - run: bun test || true          # swallows the exit code
// DO: the suite must pass, a non-zero exit fails the job and blocks the merge
// - run: bun run lint && bun run typecheck && bun test
// branch protection requires this check green before merge is allowed
```

The `|| true` is the enemy. It is how a pipeline stays green over a suite that is red, and it is always added by someone in a hurry who means to fix it later. There is no later. Delete it. Make the check required in branch protection so red cannot merge even if someone wants it to.

What good looks like is specific and worth wanting. You can deploy on a Friday afternoon without fear, because the pipeline, not a nervous human, decides what is safe to ship. Every path that matters has proof behind it. The suite runs in the time it takes to read a Slack message. And the last person to touch the code is not the person the whole team's confidence rests on.

Get this right and you earn a kind of calm that no amount of careful reviewing buys you. You stop hoping the code works and start knowing it, because a machine checked, and you can run the same check yourself in seconds.

Proof protects the code. The next three rules protect the people the code is for. In Part 5, **Protect the People You Build For**, I cover the three defaults that keep a lost laptop or a hostile request from becoming a headline: secure, private, and isolated.
