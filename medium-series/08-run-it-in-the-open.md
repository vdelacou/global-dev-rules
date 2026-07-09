================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
A Rule You Don't Check Is Already Broken

SUBTITLE
Five governance rules that keep a project honest: transparency, ownership, a paved road, enforcement you can prove, and metrics that settle the argument.

TAGS   (5 max, each 25 characters or fewer)
DevOps, Engineering Leadership, Best Practices, Metrics, Governance

COVER IMAGE   upload this file from the covers folder
covers/08-run-it-in-the-open.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 8 of 9 of The Global Rules Every New Project Should Have.*

Most of the pillars so far are about the code. This part is about the system around the code: who can see it, who owns it, how the right way becomes the easy way, how a standard actually gets enforced, and how you know whether any of it is working. Governance sounds like the boring part. It's the part that decides whether the other rules survive contact with a deadline.

One root sits under all five: a rule nobody enforces and a control nobody tested are the same thing, which is nothing. "Please remember to" decays within a week. A green checklist describes paperwork, not reality. So this is the longest part in the series, because I'm covering five pillars, and I'm going to keep each one tight and let the artifacts do the talking. No black boxes, clear ownership, pave the road, enforce and verify, measure.

## No black boxes: transparency is not optional

Anyone with a stake in the project should be able to see its real state at any moment. Opacity is where projects quietly rot. Whether you're a founder trusting an outside supplier or a CTO running an internal team, you cannot manage what is hidden from you. And when you evaluate the people building your product, the specific answers matter less than whether they're willing to be transparent at all.

The rule here that pays off most: treat documentation drift as a defect. If the README no longer matches how the project installs, runs, or deploys, the change is not finished, even if the tests pass. The way you make that real is to fail CI when the README goes stale. Extract the commands from it and actually run them.

```yaml
# .github/workflows/docs-check.yml
# DO: extract fenced bash blocks from the README and actually run them
jobs:
  readme-runs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun run docs:extract        # pulls ```bash blocks out of README.md
      - run: bash .ci/readme-steps.sh     # fails the PR if a documented command breaks
```

A README that CI proves is current is worth ten wikis that "someone should update." The rest of this pillar follows the same spirit: generate API docs from the contract so they cannot drift, give stakeholders live access to the repo, the pipeline, the board, and the metrics from day one rather than a monthly summary, and commit to numbers instead of adjectives. "Fast" and "secure" mean nothing until they're budgets a dashboard can pass or fail.

## Clear ownership: everything has a name next to it

Every part of the system, and every rule on this list, needs someone accountable for it. Shared ownership with no name attached is how things quietly rot, because everyone assumes someone else is handling it. Naming an owner is one of the cheapest reliability and security measures there is. Years of watching things fall through the cracks taught me that "the team owns it" almost always means nobody does.

Make it explicit and make it code. A CODEOWNERS file maps every path to an accountable team, so nothing is orphaned and no PR waits on whoever happens to notice.

```bash
# .github/CODEOWNERS
# DO: every path maps to an accountable team; last match wins in this file
*                       @org/platform          # default owner, nothing is orphaned
/packages/core/         @org/domain-team        # Accountable: domain-team lead
/packages/infra/        @org/platform           # Responsible: platform on-call
/docs/adr/              @org/architecture       # Consulted on any decision record

# RACI note (docs/OWNERSHIP.md): exactly one Accountable per area.
# If two teams claim A, split the area.
```

Ownership on its own is not enough, though. You also have to separate duties, so the person who requests a sensitive change is never its sole approver. That belongs in branch protection, as code, where nobody can quietly exempt themselves.

```yaml
# branch protection for `main` (as code)
# DON'T: no protection, the author clicks merge on their own PR
# DO: independent approval required, author excluded, checks must pass
required_pull_request_reviews:
  required_approving_review_count: 1
  require_code_owner_reviews: true    # a CODEOWNER must sign off
required_status_checks:
  strict: true
  contexts: ["build", "test", "typecheck", "lint"]
enforce_admins: true                  # admins are not exempt
allow_self_approval: false            # requester != sole approver
```

Two more habits complete the picture. Keep an audit trail, so approvals, exceptions, and emergency access leave a durable record and accountability is real rather than nominal. And make finding problems safe: if whoever spots an issue is saddled with owning it, people quietly stop looking. Reward detection, route the fix deliberately, and make sure whoever is accountable has the access to verify "done" for themselves, because done reported by someone who cannot check it is not done.

## Pave the road: make the right way the easy way

You don't get all these pillars' worth of good behavior by asking every team to reinvent it under deadline. You get it by building a paved road: the reusable templates, pipelines, and self-service tools that make doing the right thing the path of least resistance. If the secure, tested, observable way is also the hard way, people route around it, and every project solves the same problems slightly differently.

The mistake is shipping a wiki page titled "how to set up a service." Ship a scaffold instead, a real command that generates a service already passing every gate on a thin end-to-end skeleton.

```bash
# DON'T: "copy another repo and delete the bits you don't need" (drifts instantly)
# DO: one command generates a service that already passes every gate
$ bunx @org/create-service billing
# generates:
#   billing/
#   ├── src/index.ts              # Hono app + /health, already wired
#   ├── src/index.test.ts         # a passing smoke test, so CI is green on commit 1
#   ├── .github/workflows/ci.yml  # build + test + typecheck + lint, pre-configured
#   ├── .githooks/pre-commit      # the same gates locally
#   ├── Dockerfile                # golden base image, non-root, pinned
#   └── README.md                 # runnable, and CI proves it stays current
$ cd billing && bun install && bun test   # green out of the box
```

A team that starts from this inherits the standards instead of assembling them by hand. Two things keep the road paved: make it self-service, so a team provisions an environment or a pipeline through a declarative request they own rather than a ticket queue, and treat the platform as a product, with real docs, a maintenance owner, and a feedback loop, because its users are your own engineers and a paved road left to rot gets forked around within a quarter.

## Enforce and verify: prove the gate can actually fail

This is the pillar the other four lean on. A standard that is only prose is a suggestion. So make every standard a gate a machine enforces, then treat every control as a hypothesis until someone has proven, end to end, that it holds.

Step one: make the standard executable. Wire lint, tests, coverage, and secret scanning into a commit hook and the identical checks into CI, so a violation blocks the change instead of relying on someone to catch it in review.

```bash
#!/usr/bin/env bash
# .githooks/pre-commit
# DON'T: an empty file, or one that only echoes "remember to run the tests"
# DO: run the real gates and exit non-zero on the first failure so the commit is refused
set -euo pipefail
bun run lint:strict           # style and correctness rules, no warnings tolerated
bun test                      # the suite must be green
bun run coverage              # coverage thresholds enforced by the runner config
gitleaks protect --staged -v  # secret scan on staged content, before it ever lands
```

The hook is a fast first line. CI runs the identical gates so a hook bypassed with `--no-verify` is still caught. Two lines of defense, one set of rules.

Here is the part almost everyone skips, and it's the whole point of the pillar: prefer failing loud, and test the bypass. A check that stays green for the wrong reason protects nothing. A control nobody has tried to get around is a hope, not a defense. So don't only test that the authorized call succeeds. Test that the unauthorized one is refused.

```ts
// DON'T: proves nothing about isolation; the owner was always going to succeed
test("owner reads their expense", async () => {
  const res = await app.request(`/v1/expenses/${expenseA}`, authAs(orgA));
  expect(res.status).toBe(200);
});
// DO: org B's token against org A's resource must be indistinguishable from missing
test("cross-tenant read is refused as not_found", async () => {
  const res = await app.request(`/v1/expenses/${expenseA}`, authAs(orgB));
  expect(res.status).toBe(404);  // not 403: existence itself must not leak
});
```

The same logic scales up. Audit the whole path a real attacker would walk, especially the seam between two systems that each pass their own review, because that seam is where a forged header sneaks through. When you find a flaw, assume it repeats wherever the pattern does, grep for every instance, fix them all, and add a guard that fails the build if it returns. And don't let people opt out silently: a bypass should be rare, visible, and justified in writing, changed once at the project level rather than muted line by line. Compliance is not proof. The standard is "show me how you check it, and let me run it myself."

## Measure whether you are improving: keep score with a shared yardstick

Opinions about whether a team is getting better are cheap. A small set of agreed metrics settles the argument. Measure the delivery system as a whole, not the output of individuals, and use a recognized framework so you're comparing against the industry rather than grading your own homework.

The baseline is the four DORA metrics: deployment frequency, lead time for changes, change failure rate, and time to restore service. Together they balance speed against stability, so you cannot game one by wrecking the other. Don't hand-maintain a status slide someone updates from memory. Derive them from real CI/CD events.

```yaml
# .github/workflows/deploy.yml
# DON'T: rely on humans typing "we deployed ~5 times" into a spreadsheet later.
# DO: emit a machine-readable deployment event; the pipeline is the source of truth
- name: record deployment event
  if: success()
  run: |
    curl -sf -X POST "$FOURKEYS_EVENT_URL" \
      -H 'content-type: application/json' \
      -d "{\"event_type\":\"deployment\",
           \"service\":\"kuitto-api\",
           \"sha\":\"${GITHUB_SHA}\",
           \"deployed_at\":\"$(date -u +%FT%TZ)\"}"
# A matching "incident" event feeds change failure rate and time to restore.
```

Three guardrails keep the numbers useful. Pair the delivery metrics with flow metrics (cycle time, throughput, work in progress) so you see where work stalls, and measure the clock rather than story points, which inflate and hide queue time. Treat them as system metrics, never a stick: group by service, never by person, because per-developer leaderboards reward padded estimates and punish honesty. And watch the trend across weeks, not a single reading, where normal variation looks like a crisis or a win.

## The through-line

Notice what connects all five. Transparency, ownership, the paved road, enforcement, and measurement are each a way of moving something out of someone's head and into a place anyone can see and check. A README CI proves is current. An owner named in a file. A scaffold that ships compliance instead of a wiki that describes it. A gate you can watch fail. A metric computed from raw events. Run the project in the open, and you can hand it to a new team tomorrow and they'd understand exactly where it stands without a single meeting.

You have now built the thing right: consistent, simple, cleanly bounded, tested, secure, private, isolated, boring to deliver, resilient, observable, and governed in the open. There is one question none of that answers. Is it the right thing at all? In the finale, Part 9, **Build the Right Thing**, I cover the two rules that decide whether any of this was worth doing: obsess over the whole experience, and validate before you build.
