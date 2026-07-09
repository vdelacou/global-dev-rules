================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Write Code That Lasts

SUBTITLE
Consistency and simplicity are not style preferences. They are the two rules you pay for on every file you ever open again.

TAGS   (5 max, each 25 characters or fewer)
Clean Code, Software Engineering, Best Practices, Programming, Refactoring

COVER IMAGE   upload this file from the covers folder
covers/02-write-code-that-lasts.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 2 of 9 of The Global Rules Every New Project Should Have.*

A project's real cost is not writing code. It is reading and changing code that already exists. You write a line once and read it fifty times: in review, in the debugger at midnight, six months later when you have forgotten every assumption behind it. Optimize for the fifty, not the one.

Two of the eighteen rules attack that cost head-on. Consistency, so the code reads as if one person wrote it. Simplicity, so there is less of it to read at all. Get these two wrong and every other rule gets more expensive, because you are enforcing them on top of a codebase that already fights you.

## Consistency: the codebase should read as if one person wrote it

Every inconsistency taxes the reader. Different styles, naming, and structure force whoever opens the file to re-learn the local rules before they can even see the logic. And cosmetic difference is where real bugs hide: when half your files use one pattern and half use another, nobody can tell a meaningful deviation from a stylistic one at a glance.

The fix is not discipline. Discipline does not scale, and it does not survive a deadline. The fix is to make consistency automatic, so a machine defends the style and a reviewer never spends a second on it.

Two moves get you most of the way.

First, define formatting, linting, naming, and logging conventions once, in a single committed configuration, and enforce it with tooling. The word that matters is *once*. The same rule set in two places will quietly drift, and now you have two sources of truth that disagree. I once inherited a repo where one developer's laptop formatted every file the opposite of everyone else's, so every commit was a war of invisible whitespace. The cause was a local override nobody remembered adding.

```ts
// DON'T: a local override file drifts from the repo config, two files format differently
// .eslintrc.local.json (gitignored, only on one laptop)
{ "rules": { "semi": "off", "quotes": ["error", "double"] } }
// DO: one root config, committed, referenced by every script and by CI
// eslint.config.ts (the only lint source of truth)
export default [{ rules: { semi: ["error", "always"], quotes: ["error", "single"] } }];
```

The CI job runs the exact same command as your laptop, with no extra flags to disagree about. In Java it is the same principle: pin one formatter version and one ruleset in the parent build file, and let every module inherit it, rather than letting each module pin its own version so the output depends on who built last.

```java
// DON'T: each module pins its own formatter version, output depends on who built last
// module-a uses spotless 2.30, module-b uses 2.43, results differ
<plugin><groupId>com.diffplug.spotless</groupId><version>2.30.0</version></plugin>
// DO: one version and one ruleset in the parent pom, every module inherits it
<plugin>
  <groupId>com.diffplug.spotless</groupId><artifactId>spotless-maven-plugin</artifactId>
  <version>2.43.0</version>
  <configuration><java><googleJavaFormat/></java></configuration>
</plugin>
```

Second, put a ceiling on complexity and duplication. Keep units small enough to read at a glance. Favor early returns over deep nesting, so the happy path runs straight down the left margin instead of burrowing three braces deep. Treat copy-pasted logic as the exception you notice, not the norm you tolerate.

Nesting is the quiet killer. Deeply nested conditionals hide the one line that actually matters inside an arrow of braces, and every reader has to unwind the whole structure to find it.

```ts
// DON'T: arrow-of-arrows nesting hides the one line that matters
function price(o: Order): number {
  if (o) { if (o.items.length) { if (o.tier === "premium") { return o.subtotal * 0.8; }
      else { return o.subtotal; } } else { return 0; } } else { return 0; }
}
// DO: guard clauses return early, the rule reads top to bottom
function price(o: Order): number {
  if (!o || o.items.length === 0) return 0;
  return o.tier === "premium" ? o.subtotal * 0.8 : o.subtotal;
}
```

Same behavior. One reads like a maze, the other reads like a sentence.

What good looks like: a new contributor's first commit is indistinguishable from a veteran's, and nobody argues about spacing in review because a machine already settled it.

## Simplicity by default: the best code is the code you never wrote

The steady pull on every project is toward more. More abstraction, more configuration, more cleverness, more code kept "just in case." Resist it, deliberately, because that pull never stops on its own. Every line you add is a line someone else has to read, test, secure, and eventually change. The cheapest, safest, most maintainable code is the code that does not exist.

Five habits keep a codebase honest.

**Do the least that works.** Before you write something new, walk down a ladder and stop at the first rung that solves the problem. Do you need it at all. Can the language or platform already do it. Does a dependency you already have cover it. Does it collapse to one clear line. Reach for a custom solution only when the rungs above genuinely fail. Most of the time they do not, and you were about to add a dependency for something the runtime ships for free.

```ts
// DON'T: a dependency plus custom code to do what the runtime already ships
import { v4 as uuidv4 } from "uuid";
function newId(): string { return uuidv4(); }
// DO: the platform already has it, one less dependency to audit and update
function newId(): string {
  return crypto.randomUUID();
}
```

That deleted import is not nothing. It is one fewer package to audit, patch, and worry about the day it publishes a compromised version. Every dependency you do not take is a supply-chain risk you do not carry.

**Delete before you add.** A change that removes code is usually the better fix. Treat cleverness as a cost the next reader pays, and prefer the boring, obvious version. A config-driven strategy map for two cases that will never grow is not flexibility. It is indirection you now have to trace through.

```ts
// DON'T: a config-driven strategy map for two fixed cases nobody will extend
const handlers: Record<string, (n: number) => number> = {
  double: (n) => n * 2, triple: (n) => n * 3,
};
function apply(kind: string, n: number) { return handlers[kind](n); }
// DO: the two cases inline, less code and no lookup that can miss
function apply(kind: "double" | "triple", n: number): number {
  return kind === "double" ? n * 2 : n * 3;
}
```

The second version is shorter, has no lookup that can return undefined, and the type system now guarantees the two cases are the only two. You removed code and gained safety at the same time.

**Earn your abstractions.** Tolerate a little duplication rather than guess at the wrong shared abstraction. Extract the pattern only when the third real case shows you what it actually is. A wrong abstraction costs more to undo than the duplication it replaced, because everyone downstream has already bent their code to fit it. Two similar call sites are not a pattern. They are two call sites. Wait for the third. The third is what tells you which parts are shared and which only looked shared.

**Defer the build, not the seam.** This is the one people get backwards. YAGNI applies to code, not to the thin interfaces that keep a later swap cheap. When a future need is likely, put the boundary in now, a port, an adapter, a single place it will plug in, but do not build the heavy implementation until the need is real.

```ts
// DON'T: a full retrying, batching, metric-emitting mailer nobody needs at launch
class SmtpMailer {
  async send(m: Mail) { /* retries, circuit breaker, batching, metrics, 200 lines */ }
}
// DO: define the port now, ship the smallest real adapter, swap later without churn
interface Mailer { send(m: Mail): Promise<Result<void, MailError>>; }
class ConsoleMailer implements Mailer {
  async send(m: Mail) { return { ok: true, value: undefined } as const; }
}
```

The interface is the cheap part and the valuable part. It costs three lines today and it lets you replace `ConsoleMailer` with a real SMTP adapter later without touching a single caller. The two-hundred-line mailer is the expensive part, and you do not need it until you actually send mail in anger. Put in the seam, skip the build.

**But simplicity is not negligence.** This is the line that separates a clean codebase from a broken one. What you never cut for the sake of "simple" is the safety: input validation, error handling, security, and anything the user actually asked for. Laziness about ceremony is a virtue. Laziness about safety is a bug. The "simpler" version that trusts its input and drops its checks is not simpler. It is a vulnerability with fewer lines.

```ts
// DON'T: "simpler" code trusts input and concatenates it straight into a query
async function find(email: string) {
  return db.execute(`SELECT * FROM users WHERE email = '${email}'`);
}
// DO: validate at the edge, use a parameterized query, return a typed error
async function find(raw: unknown): Promise<Result<User, "invalid_email" | "not_found">> {
  const p = z.string().email().safeParse(raw);
  if (!p.success) return { ok: false, error: "invalid_email" };
  const u = await db.query.users.findFirst({ where: eq(users.email, p.data) });
  return u ? { ok: true, value: u } : { ok: false, error: "not_found" };
}
```

The first one is shorter and it is a SQL injection. The second is the correct amount of code: it validates at the boundary, it parameterizes the query, and it returns the failure as a value the caller must handle. That is what "do the least that works" means. The least that *works*, not the least that compiles.

What good looks like: reviewers can hold the whole change in their head, and the honest answer to "could this be smaller?" is usually no, because you already deleted everything that could go.

## The two together

Consistency and simplicity reinforce each other. Consistent code is easier to simplify, because you can see the patterns clearly instead of squinting past cosmetic noise. Simple code is easier to keep consistent, because there is less of it to drift. Both buy you the same thing: a codebase where the next change is cheap, whether the next person is a veteran or someone who joined last week.

Both of these live inside a single service, a single file, a single function. The next rule zooms out to the shape of the whole system. In the next part, **Keep Clean Boundaries**, I show you the architecture that lets you swap the database, the framework, or the entire visual theme one at a time, with the code to prove the boundary is real and not just a diagram.
