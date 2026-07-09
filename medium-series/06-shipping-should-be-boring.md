================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Boring Is the Highest Compliment You Can Pay a Deployment

SUBTITLE
Automate the path to production and rent the parts that are not your product, so shipping is a non-event nobody gathers to watch.

TAGS   (5 max, each 25 characters or fewer)
DevOps, CI CD, Continuous Delivery, Cloud Computing, Automation

COVER IMAGE   upload this file from the covers folder
covers/06-shipping-should-be-boring.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 6 of 9 of The Global Rules Every New Project Should Have.*

You can tell how healthy a team is by watching a deploy. If people gather, hold their breath, and keep a rollback plan open in another tab, the process is broken no matter how the release turns out. If nobody notices it happened, the process is right.

Shipping should be a routine, not an event. Manual, heroic deployments are slow, error-prone, and impossible to repeat reliably. In delivery, boring is the highest compliment. And you get boring the same way every time: by taking the humans out of the parts that do not need judgment, and handing the undifferentiated machinery to someone whose actual job it is.

This part covers two of the eighteen rules that produce that calm: **delivery should be boring**, and **run as little as possible yourself**. One automates the path to production. The other shrinks how much you own along that path. Together they turn a deploy into a non-event.

## Delivery should be boring: automate the path to production

Start with how work reaches the trunk, because everything downstream inherits the mess or the discipline of this one habit. Keep one source of truth and adopt trunk-based development: short-lived branches that live for days, not months, integrated continuously. Long-lived divergent branches are where painful merges and last-minute surprises come from.

```bash
# DON'T: a long-lived branch that drifts for weeks, then a single massive PR
git checkout -b feature/big-rewrite
# ... 3 weeks, 87 files, 6000 lines later ...
git merge main            # days of conflict resolution, nobody can review this
# DO: short-lived branch, one focused change, rebased and merged the same day
git checkout -b add-receipt-total-field
git commit -m "Add total_cents column to receipts (expand step)"
git rebase main           # stay current with trunk continuously
# open a small PR, review in minutes, merge, delete the branch same day
```

An 87-file pull request is not reviewable, and everyone in the review knows it, so they approve it on trust and hope. The small, focused change is the unit of work, not an obstacle to it. It reviews in minutes, merges the same day, and rolls back in one step if it goes wrong. Integrating small changes continuously keeps the project close to shippable at all times.

Once changes land, the path to production is a pipeline, not a person. Build, test, and deploy through automation with a high success rate. Eliminate manual deployment entirely. Deploy in small increments, often, and roll out gradually so a bad change reaches a few users rather than everyone.

```yaml
# DON'T: a "deploy" that is a human running a script against prod with no gate or rollback
# (there is no artifact for this: it is someone SSHing in and hoping)

# DO: automated build, canary weight, promote on healthy metrics, one-command rollback
- run: bun test && bun run typecheck && bun run lint:strict
- name: Deploy canary (10% traffic)
  run: scw container deploy --image "$IMAGE" --traffic canary=10
- name: Watch canary SLO for 5 minutes
  run: ./ci/watch-error-rate.sh --max 0.01 --window 5m   # aborts if breached
- name: Promote to 100%
  run: scw container deploy --image "$IMAGE" --traffic stable=100
# rollback is one step: scw container deploy --image "$LAST_GOOD_IMAGE" --traffic stable=100
```

The canary is the whole trick. The new build takes 10 percent of traffic while the pipeline watches the error rate. If it breaches, the job aborts before the change reaches everyone. If it stays healthy, it promotes to full. A bad release becomes a blip for a tenth of your users instead of an outage for all of them, and the rollback is one command, seconds not a maintenance window.

The environment itself lives in files. Describe your infrastructure as code, so the whole thing can be rebuilt from scratch with a single command. If an environment vanishes, you recreate it. Every change goes through version control with a visible history.

```hcl
# DON'T: "created by hand" in the console, no file, no history, no review
# Bucket kuitto-receipts was clicked into existence by an engineer in the web UI.
# Nobody can recreate it, diff it, or know why versioning is off. This is the anti-pattern.

# DO: the resource is a file, so it is reviewable, reproducible, and destroyable
resource "scaleway_object_bucket" "receipts" {
  name = "kuitto-receipts"
  versioning { enabled = true }
  lifecycle_rule {
    id = "expire-tmp", prefix = "tmp/", enabled = true
    expiration { days = 7 }
  }
}
# tofu plan && tofu apply -> rebuilds the whole thing from files, every time
```

A bucket clicked into existence on a Tuesday is a resource nobody can recreate, diff, or explain. Six months later someone asks why versioning is off and the only honest answer is a shrug. The same bucket as a file is reviewable, reproducible, and destroyable, and the "why" lives right next to the "what."

The last habit under this pillar is the one that saves you from waking a client at 2 a.m.: change contracts additively. Never break a shipped API or database schema in place. Add the new shape, migrate everything onto it, then remove the old one in a later step. This is the expand-contract pattern, and it stops being optional the moment real clients depend on you.

```ts
// DON'T: destructive migration that renames a live column in one shot
// old readers and in-flight requests break the instant this runs
await db.execute(sql`ALTER TABLE receipts RENAME COLUMN amount TO total_cents`);

// DO: expand (add nullable new column), migrate, then contract in a separate release
// migration 0007_expand.ts
await db.execute(sql`ALTER TABLE receipts ADD COLUMN total_cents integer`);
await db.execute(sql`UPDATE receipts SET total_cents = amount WHERE total_cents IS NULL`);
// deploy code that dual-writes both columns, then a later 0009_contract drops "amount"
```

The rename looks clean and is a landmine. The instant it runs, every old reader and every in-flight request breaks, because the column they name no longer exists. Expand-contract splits it into safe steps: add the new column, backfill it, run code that writes both, verify, and only then, in a separate release, drop the old one. A client you cannot see should never wake up to a field that vanished.

## Run as little as possible yourself: rent the undifferentiated parts

Every server you operate, certificate you renew, and database you babysit is work that is not your product and a place your attention leaks away. The less you own at the operating-system level, the less there is to patch, misconfigure, or be woken up for. So lean on managed services for everything that is not your actual differentiator, and keep people out of the machines entirely.

Prefer managed over self-run. A managed database means patching, backups, and failover are the provider's job, not a pager duty of yours.

```hcl
# DON'T: a raw VM you now have to patch, back up, monitor, and fail over yourself
resource "scaleway_instance_server" "db_vm" {
  type  = "DEV1-M"
  image = "ubuntu_jammy"
  # ... then someone hand-installs postgres, and it is now a pet you feed forever
}

# DO: a managed database; the provider handles patching, backups, and HA
resource "scaleway_rdb_instance" "main" {
  name          = "kuitto-prod"
  engine        = "PostgreSQL-15"
  is_ha_cluster = true
  # automated backups, minor-version patching, and failover are the provider's problem
}
```

The VM is a pet. The moment someone hand-installs Postgres on it, you own its kernel updates, its disk growth, and its failover at 3 a.m. The managed instance is a line of config, and the boring operational work belongs to someone whose entire business is doing it well.

The sharper version of this rule: no servers you log into. If the answer to an incident is "SSH in and poke around," the design is already wrong. Deploy immutable units and replace them rather than logging in to fix them.

```dockerfile
# DON'T: a box you SSH into to "just fix it live"
#   ssh -i prod.pem ubuntu@bastion -> edit files in place -> undocumented drift
#   the fix survives until the next reboot, then vanishes. No SSH keys should exist.

# DO: an immutable image. To change the app you build a new image and redeploy, never log in.
FROM oven/bun:1-slim
WORKDIR /app
COPY package.json bun.lockb ./
RUN bun install --frozen-lockfile --production
COPY . .
USER bun                       # non-root, nothing to log into
CMD ["bun", "run", "start"]
# the running container has no shell access granted and no persistent state; kill and replace freely
```

The live fix is the worst kind of fix. It works until the next reboot, then vanishes, and in the meantime that box no longer matches any code you can see. Now your running system and your repository disagree and nobody knows by how much. I have lost real hours to a "why is prod behaving differently" hunt that ended at exactly one hand-edited box. An immutable image has no shell to log into and no state to drift. You change it by building a new one and replacing the old, which means every change is in the code, reviewed, and repeatable.

Two more pieces complete the picture. Insist on automatic certificates: only build on platforms that issue and renew TLS for you, because an outage caused by an expired cert is a self-inflicted wound nobody should still be suffering. And let only the pipeline touch infrastructure. People do not hold write access to production infrastructure; the infrastructure-as-code pipeline does. A change reaches the cloud by merging reviewed code, never by clicking in a console.

```hcl
# DON'T: developers hold AdministratorAccess and change prod from their laptops
# (broad standing write credentials on humans: the thing to remove)

# DO: humans are read-only; only the CI identity can apply infra, and only via a reviewed merge
resource "github_branch_protection" "main" {
  pattern = "main"
  required_pull_request_reviews { required_approving_review_count = 1 }
  required_status_checks { strict = true, contexts = ["ci/plan", "ci/typecheck"] }
  enforce_admins = true             # even admins cannot push straight to main
}
# the deploy role (write access to infra) is assumed only by the pipeline, never by a person
```

Standing admin keys on human laptops are the credential that turns one phished engineer into a company-wide incident. Take them away. Humans get read-only, the pipeline gets write, and every infrastructure change becomes versioned, reviewed, and reversible by construction. `enforce_admins` matters here: it means even the person who set up the rule cannot route around it, which is the difference between a control and a suggestion.

What good looks like across both rules is a specific, checkable calm. A new engineer goes from an empty laptop to the full system running with one command before lunch. There is nothing to SSH into. No certificate sits on anyone's calendar. The only path to changing production is a merged, reviewed commit. And a production deploy is unremarkable enough that nobody gathers to watch it, because there is nothing to see.

That is the payoff. When shipping is boring, you ship more, you ship smaller, and you sleep. The teams that treat every release as an event ship less often precisely because it is so painful, which makes each release bigger and more dangerous, which makes it more of an event. Boring breaks that loop.

Delivery is smooth and you own little. But everything still breaks eventually, and you need to see it when it does. In Part 7, **Design for Failure and See Everything**, I cover the two rules that decide whether an outage is a two-minute glance at a dashboard or a two-hour hunt in the dark: design for failure, and make it observable.
