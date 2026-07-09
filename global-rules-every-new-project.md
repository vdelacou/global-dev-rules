# The Global Rules Every New Project Should Have

The first question on a new software project is almost always the wrong one. "Which framework? Which database? Which cloud?" People reach for the stack before they agree on how they intend to work, and then spend the next two years paying for it.

The specific tools always date. The frameworks and databases worth naming a few years ago are already half-replaced today. That is exactly the point: underneath the specifics sits a much smaller set of principles that do not depend on any technology at all.

These are the global rules. They hold whether you build in any language, deploy to any cloud, and start this year or in ten. Agree on them before you write a line of code, because they are far harder to add later than to design in from the start.

I have grouped them into eighteen pillars. The through-line is simple: decide how the project works before you decide what it is built with.

## 1. Consistency: the codebase should read as if one person wrote it

A project's real cost is not writing code, it is reading and changing code that already exists. Every inconsistency taxes that work. Different styles, naming, and structure force every reader to re-learn the rules on every file, and they let real bugs hide in the noise of cosmetic difference.

So make consistency automatic, not a matter of discipline:

- Define formatting, linting, naming, and logging conventions once, in a single committed configuration (the same rule set in two places will quietly drift), and enforce it with tooling. Reviewers should never spend attention on style, because a machine already did.
- Put a ceiling on complexity and duplication. Keep units small enough to read at a glance, favor early returns over deep nesting, and treat copy-pasted logic as the exception you notice, not the norm you tolerate.

What good looks like: a new contributor's first commit is indistinguishable from a veteran's, and no one argues about spacing in review.

## 2. Simplicity by default: the best code is the code you never wrote

The steady pull on every project is toward more: more abstraction, more configuration, more cleverness, more code kept "just in case." Resist it. Every line you add is a line someone else has to read, test, secure, and eventually change. The cheapest, safest, most maintainable code is the code that does not exist.

- Do the least that works. Before writing something new, walk down a ladder and stop at the first rung that solves the problem: do you need it at all, can the language or platform already do it, does a dependency you already have cover it, does it collapse to a single clear line? Reach for a custom solution only when the rungs above genuinely fail.
- Delete before you add. A change that removes code is usually the better fix. Treat cleverness as a cost the next reader pays, and prefer the boring, obvious version.
- Earn your abstractions. Tolerate a little duplication rather than guess at the wrong shared abstraction, and extract the pattern only when the third real case shows you what it actually is. A wrong abstraction costs more to undo than the duplication it replaced.
- Defer the build, not the seam. When a future need is likely, put the boundary in now (a port, an adapter, a single place it will plug in) but do not build the heavy implementation until the need is real. YAGNI applies to code, not to the thin interfaces that keep a later swap cheap.
- But simplicity is not negligence. What you never cut for the sake of "simple" is the safety: input validation, error handling, security, and anything the user actually asked for. Laziness about ceremony is a virtue; laziness about safety is a bug.

What good looks like: reviewers can hold the whole change in their head, and the honest answer to "could this be smaller?" is usually no.

## 3. Keep clean boundaries: you should be able to swap any layer without touching the others

A system stays changeable only if its parts are actually separable. When business logic knows about the database, or the look-and-feel is tangled into the logic, every change ripples: a new framework becomes a rewrite, a rebrand becomes a hunt through the whole codebase. Draw the boundaries deliberately, so each part can change without disturbing the ones that should not care.

- Point dependencies inward. Your business logic should know nothing about the database, the web framework, or the cloud it runs on. It reaches the outside world only through interfaces it defines, so infrastructure depends on the logic and never the other way around. That is what lets you swap a vendor, or test the entire core against fakes, without rewriting it. Your domain model represents the business, not the database: the two serve different purposes, so give them different shapes, map between them at the repository, and let the business drive the schema rather than the schema drive the business.
- Put every external thing behind a port. A database, a payment gateway, a queue, an email sender: each sits behind a small interface with a real adapter and an in-memory fake, chosen once at a single wiring point. One contract, many implementations.
- Seal the presentation behind a design system. Make one design system the single source of visual truth: define every visual value once as a semantic token (color, spacing, type) and forbid raw literals everywhere else. Components take data in and return markup, with no business logic or data fetching inside them, and the styling engine stays invisible to the rest of the app.
- Keep the backend a client-agnostic API. Expose one resource-shaped API that every client (web, iOS, Android, a third-party integration) consumes the same way, rather than endpoints shaped to a single screen. A screen-shaped endpoint recouples the backend to one interface and has to change every time that screen does; a resource-shaped one lets a new client compose what it needs from what already exists.
- Let the frontend depend on a contract it controls. Agree the API contract first, then have the frontend reach data only through a gateway with two implementations, a real client and a fake returning canned data in the same shape, so the UI is built, demoed, and tested in parallel with the backend. Translate the API's shape into the frontend's own model at that gateway, so an API change is absorbed in one mapping function rather than rippling through every component. The same discipline holds on the server, where wire DTOs and domain types never cross.
- Make the boundary testable, not aspirational. The proof is mechanical: a rebrand touches only the design-system folder, swapping the styling engine leaves the app unchanged, and a change to a business rule never edits a component. If one change has to reach across a boundary, the boundary has already leaked.

What good looks like: you can replace the database, the web framework, or the entire visual theme one at a time, and the parts that should not care never notice.

## 4. Proof over hope: nothing is done until it is verified

"It works on my machine" is not evidence. Confidence in software should come from automated checks that anyone can run, not from the optimism of whoever wrote the last change. Every untested path is a liability that will surface at the worst possible time.

- Test in layers. Unit tests, integration tests, and end-to-end tests each catch what the others miss, and performance tests catch what all of them ignore.
- Keep the fast tests fast. Unit tests should run in milliseconds each, so the whole suite is something you hit constantly rather than a coffee break. A unit test that is slow is usually reaching for a database or a network it should be faking.
- Adopt a testing philosophy the whole team follows. Test-first is one option. A simpler rule that works: every bug you fix becomes a permanent test, so the same defect can never return unnoticed.
- Make coverage mean something. A high line-coverage number is easy to fake, so treat mutation testing, which checks that your suite actually fails when the code is deliberately broken, as the coverage metric that matters. Hold your core logic to a near-total bar and glue code to a looser one, and when a gate is hard to pass, restructure the code rather than lower the threshold.
- Test behavior, not internals. Point a test at what a feature should do, described in the language of the domain, so you can restructure the code freely without rewriting the tests. Prefer simple hand-written stand-ins for external systems over heavy mocking that only re-asserts your own call sequence, and never write a test whose real claim is that a third-party library works.
- Gate every merge. Static analysis and the full test suite run on every change, and nothing merges with a critical defect or a red pipeline.
- Hold generated code to the same bar. Code a tool — a scaffolder, a generator, or an AI assistant — produced is still code: it runs through the identical suite and review a human's would. Provenance is not proof; the checks are.

What good looks like: you can deploy on a Friday afternoon without fear, because the pipeline, not a nervous human, decides what is safe to ship.

## 5. Secure by default: assume someone hostile is paying attention

Security is a starting condition, not a phase you schedule near launch. The cheapest vulnerability to fix is the one you never introduce, and retrofitting security onto a finished system is expensive and almost always incomplete.

- Keep secrets out of the codebase entirely. Keys, passwords, and tokens belong in a secret manager, never in source, configuration files, wikis, tickets, logs, or shared drives, and no outside vendor or supplier should hold more of them than the job strictly requires. Manage keys centrally so no single person holds the one that unlocks everything, and treat the strongest secret as one generated by automation, rotated on a schedule, and never read by a human. When one leaks, rotate it and purge the history, because deleting today's copy does not un-leak it.
- Do not build authentication or cryptography yourself. Use a vetted library or an identity provider for login, sessions, tokens, and password handling, and put every cloud, admin, and provider console behind single sign-on and multi-factor authentication. The auth code you hand-roll is the auth code with the subtle, expensive bug.
- Control what you depend on. The third-party libraries and runtime images you build on carry known vulnerabilities of their own. Pin them to fixed, known versions so builds are reproducible, scan them continuously, refuse to ship anything flagged, and keep them current with automated updates so a pinned version never quietly rots into a known-vulnerable one.
- Secure the software supply chain, not just the code you write. Build immutable, versioned artifacts, record what went into each one (a bill of materials), and sign them, so only trusted, unmodified builds can reach production.
- Validate at the boundary and enforce on the server. Every untrusted value should pass through one validating checkpoint before it reaches anything dangerous (a database query, a command, a file path), so that "this was checked" is a fact you can point to rather than an assumption. Authorize on the server, not only in the interface.
- Expose only what has to be public. Databases, caches, message queues, internal services, and admin panels belong on a private network, never directly reachable from the internet; the only thing the public can address is the application itself, behind its own controls. A data store that answers a connection from any address on the internet is a breach waiting for someone to notice it.
- Apply one security baseline to every project rather than standards that drift from team to team, and enforce the fundamentals everywhere without exception: authenticated access, encrypted transport, and controls against abuse such as rate limits and allow or deny lists.

What good looks like: a lost laptop or a compromised dependency is an incident you contain calmly, not a catastrophe, because no single leaked secret unlocks the whole system.

## 6. Private by default: treat personal data as a liability, not an asset

Every piece of personal data you hold is a responsibility and a risk, not a trophy. The cheapest data to protect, and the only data that can never leak, is the data you chose not to collect. So gather the least you need, be honest about why, and design for the user's rights from the very first version of the schema.

- Know the law that applies to your users, not just your office. Privacy rules now follow the user: if you serve people in a region, its regime generally applies to you even when your company and servers sit elsewhere (China's PIPL, Europe's GDPR, and their siblings share the same spine). The penalties are sized to hurt, often a share of global revenue.
- Minimize and justify. Collect only what a real purpose requires, and be able to state that purpose in a sentence. Treat sensitive data such as health, biometrics, finances, and anything concerning minors with extra care and explicit, specific consent.
- Keep personal data out of the places it leaks by accident. No personal data in logs, and none in URLs or query strings, which get written to access logs, cached by proxies, and handed to analytics. Send it in a request body instead, so searching by someone's name is a POST, not a GET that ends up in three logs.
- Build for user rights from day one. Letting someone see, correct, export, or delete their data, and withdraw consent, should be a routine operation rather than an engineering emergency.
- Map your data. Know what personal information you hold, why, where it lives, how it flows, who you share it with, and whether any of it crosses a border. Classify it by sensitivity so the riskiest data gets the strongest controls.
- Never copy production data into a test or development environment. Those environments have weaker controls and wider access, which makes them the cheapest place to lose real data. Generate realistic synthetic fixtures instead.
- Assess before risky processing. Run and record a short impact assessment before anything high-risk (automated decisions, sensitive data, cross-border transfers), keep the record, and revisit it whenever the purpose, a contract, a regulation, or a breach changes.

What good looks like: you can produce a current map of every category of personal data on request, a deletion request is a routine job rather than a fire drill, and no one on the team is unsure what personal data the system holds or why.

## 7. Isolate by default: one user's data must never reach another

The worst failure in any system that holds data for more than one person is the quiet one: customer A sees customer B's records. It rarely announces itself, it breaks the deepest promise you make to users, and by the time you notice, the exposure is already history. So treat the boundary between one tenant, customer, or user and the next as a first-class part of the architecture, not a filter you remember to add.

- Derive who you are acting for from one trusted source. The identity of the tenant or user a request acts for should come from a verified token, never from a URL, header, or field the caller can change.
- Defend in depth, and assume each layer has a bug. Enforce the boundary in more than one place (in code that cannot run without the owner's identifier, and again in the data store that filters every row), so a mistake in one layer is caught by the next.
- Fail closed. Missing ownership context should return nothing, not everything and not an error. A bug should surface as absent data, never as someone else's data.
- Shrink the blast radius. Give the running system the narrowest access it can work with, so the worst case of a breach (one injection, one leaked credential) is one owner's data, not the whole estate.
- Prove it per feature. Every endpoint ships a test that one owner's credentials cannot reach another owner's resource. Isolation you have not tested is isolation you do not have.

What good looks like: you can state, and demonstrate on demand, that there is no path by which one customer reaches another's data, and the proof is a test that runs on every change.

## 8. Delivery should be boring: automate the path to production

Shipping should be a routine, not an event. Manual, heroic deployments are slow, error-prone, and impossible to repeat reliably. In delivery, boring is the highest compliment.

- Keep one source of truth for the code and adopt trunk-based development: a branching approach that is written down, with short-lived branches that live for days, not months. Long-lived, divergent branches are where painful merges and last-minute surprises come from. Integrating small changes continuously keeps the project close to shippable at all times, so the small, reviewable commit becomes the unit of work rather than an obstacle.
- Build, test, and deploy through an automated pipeline with a high success rate (expect well above nine builds in ten to be green, and treat a flaky pipeline as a bug in itself). Eliminate manual deployment entirely. Deploy in small increments and often: shipping many times a week, even many times a day, beats one large release a quarter, because each change is easy to understand and to roll back in one step (seconds, not a maintenance window). Roll releases out gradually, to a slice of traffic first (canary or blue-green), so a bad change reaches a few users rather than everyone.
- Describe your infrastructure as code. The entire environment lives in files and can be rebuilt from scratch with a single command in minutes (five, not a day of clicking through consoles). If a whole environment vanishes, you recreate it. Every change goes through version control with a visible history.
- Organize the system as vertical slices. A feature owns its code, its data access, and its slice of infrastructure end to end, so it can be built, deployed, and scaled on its own rather than through a shared layer everyone contends over.
- Change contracts additively. Never break a shipped API or database schema in place: add the new shape, migrate everything onto it, then remove the old one in a later step, and version anything outside consumers depend on. A client you cannot see should never wake up to a field that vanished.
- Keep environments separate and honest. Development, staging, and production are isolated, and production credentials never live in code. Any component should run on its own against fakes, so no one needs the entire system standing to work on a single piece of it. Spin up throwaway environments on demand for a branch, a test run, or a load test, and destroy them minutes later, so validation never waits on one shared, contended environment.

What good looks like: a new engineer goes from an empty laptop to the full system running with one command before lunch, and a production deploy is unremarkable enough that nobody gathers to watch it.

## 9. Run as little as possible yourself: rent the undifferentiated parts

Every server you operate, certificate you renew, and database you babysit is work that is not your product and a place your attention leaks away. The less you own at the operating-system level, the less there is to patch, misconfigure, or be woken up for. So lean on managed services for everything that is not your actual differentiator, and keep people out of the machines entirely.

- Prefer managed over self-run. Use managed databases, queues, and storage rather than services you install on a server you maintain. Never hand-run something a platform will operate for you more reliably, and never spin up a virtual machine you then have to own.
- No servers you log into. If the answer to an incident is "SSH in and poke around," the design is already wrong. No SSH, no long-lived machines to tend, no pets: deploy immutable units and replace them rather than logging in to fix them.
- Insist on automatic certificates. Only build on platforms that issue and renew TLS certificates for you. An outage caused by an expired certificate is a self-inflicted wound nobody should still be suffering.
- Only the pipeline touches infrastructure. People do not hold write access to production infrastructure; the infrastructure-as-code pipeline does. A change reaches the cloud by merging reviewed code, never by clicking in a console, which makes every infrastructure change versioned, reviewed, and reversible by construction.

What good looks like: there is nothing to SSH into, no certificate on anyone's calendar, and the only path to changing production infrastructure is a merged, reviewed commit.

## 10. Design for failure: everything breaks, so plan the recovery

Assume outages, data loss, and traffic spikes will happen, because they will. Reliability is not the absence of failure. It is fast, well-practiced recovery from it.

- Set explicit reliability targets. An uptime goal and an error-rate ceiling you hold yourselves to (for example, keeping failed requests to a small fraction of a percent) turn "reliable enough" into a number instead of a feeling.
- Make failure explicit in the code. Have operations that can fail return an outcome that says so, as an ordinary value the caller is forced to handle, and reserve exceptions for genuine bugs. A business failure is data, not an exception; when every error path is part of the contract, no failure gets silently swallowed.
- Keep read paths explicit. Write read queries by hand so you can see and tune exactly what runs, because an ORM will happily hide the query that is the difference between a 20-millisecond page and a 2-second one. Let the ORM handle writes, where its mapping, validation, and safety are what you actually want.
- Keep reads fast as the table grows. Page with a stable cursor rather than OFFSET, which re-reads from row zero on every page and slows linearly the deeper you go, and stream or bulk-load large result sets instead of holding them in memory.
- Do not fire and forget. When a committed change must trigger a side effect (an email, an event, a downstream update), record that intent in the same transaction as the change and let a separate step deliver it with retries. A follow-up that survives a crash beats a best-effort call that vanishes with the process.
- Keep backups you have actually restored. An untested backup is a rumor. Practice a full recovery on a schedule, at least quarterly, and time how long it takes, so a real incident is a rehearsal rather than a first attempt.
- Scale with demand. The system should grow and shrink with load on its own, and ideally cost close to nothing when idle. Favor stateless components so that scaling is simply a matter of adding copies, and cache deliberately where it counts.
- Meet performance targets under real load. Commit to response times and throughput, then prove them with load tests in a production-like environment (for instance, keeping the 99th percentile response under a second at your expected peak).
- Treat data as sacred. Prefer soft deletes and retained history over destruction, manage every schema change through versioned migrations, and choose storage deliberately so data stays durable and easy to query.
- Learn from every failure. After an incident, run a blameless review that finds the root cause and turns it into concrete backlog items, so the same outage cannot happen twice. Recovery without learning just schedules the next incident.

What good looks like: your disaster-recovery drill is boring, and a sudden spike in traffic is a line on a graph rather than a phone call at two in the morning.

## 11. Make it observable: you cannot fix what you cannot see

A running system should explain itself. In production the question is never simply "is something wrong," it is "what is wrong, where, and since when." Without instrumentation you are debugging blind, and the outage always lasts longer than it should.

- Instrument everything. Logs, metrics, and traces across the application, the infrastructure, and the data layer, so that a single request can be followed from end to end. Prefer an open, vendor-neutral telemetry standard so you are not locked to one tool.
- Watch behavior, not just health. Track how the product is actually used, including call volumes, latencies, and where users drop off, because that same data serves both reliability and product decisions.
- Alert on what matters. Monitor error rates and latency percentiles (P95 and P99, since averages hide the worst experiences), and alert on anomalies rather than only fixed thresholds, so the system tells you about a problem before your users do. Keep only alerts that are actionable: anything that pages twice without prompting a fix gets automated away or deleted. And after an incident slips through, fix what actually broke, not just the reason the alert did not fire.

What good looks like: when something breaks, you find the cause in minutes from a dashboard, not in hours of guessing.

## 12. No black boxes: transparency and documentation are not optional

Anyone with a stake in the project should be able to see its real state at any moment. Opacity is where projects quietly rot. Whether you are a founder trusting an outside supplier or a CTO running an internal team, you cannot manage what is hidden from you. When you evaluate the people building your product, the specific answers they give matter less than whether they are willing to be transparent at all.

- Document the essentials. How to install, test, run locally, and deploy each environment, plus the architecture and the code itself. Pick one working language for the whole project and keep everything in it. Treat documentation drift as a defect: if the README no longer matches how the project installs, runs, or deploys, the change is not finished, even if the tests pass.
- If you expose an API, treat its documentation as a product. Generate it from the contract so it cannot drift, give every endpoint a real example, and hold it to the standard of the references people actually enjoy using. An API without documentation someone could onboard against is a private API you happen to have left exposed.
- Record decisions where they cannot drift from the code. Change the document and the code in the same commit, and capture each significant choice with what it rejected and how it could be reversed, so the answer to "why is it like this" survives the people who were in the room.
- Build institutional memory. Keep an append-only log of the lessons that cost you (the mistake, the decision, the non-obvious gotcha) so the same wall is never hit twice, and keep the current plan and its open threads in a durable file rather than in someone's head or a chat thread. Work should survive the loss of all context, so a returning contributor, or a brand-new one, resumes from what is written down instead of from memory.
- Give stakeholders real-time access from the first day, to the repository, the pipeline, the task board, and the metrics. Not a monthly summary. Live access to the actual thing.
- Run a defined process with a visible backlog and honest bug tracking, so that priorities and problems sit in the open where they can be dealt with, and commit to measurable thresholds rather than adjectives. "Fast," "secure," and "well tested" mean nothing until they are numbers someone agreed to and anyone can check.

What good looks like: you could hand the project to a new team tomorrow and they would understand exactly where it stands without a single meeting.

## 13. Clear ownership: everything has a name next to it

Every part of the system, and every rule on this list, needs someone accountable for it. Shared ownership with no name attached is how things quietly rot, because everyone assumes someone else is handling it. Standards do not enforce themselves and systems do not maintain themselves, so naming an owner is one of the cheapest reliability and security measures there is.

- Make ownership explicit. For each area (a service, an environment, a pipeline, a security gate) name who is accountable for the outcome and who is responsible for the work, and write it where anyone can find it. A simple responsibility map (who is Responsible, Accountable, Consulted, and Informed) removes the "I thought you had it" failure.
- Separate duties where it matters. Whoever requests a sensitive change should not be the only one who approves it, and access to production should follow least privilege and pass through review.
- Keep an audit trail. Approvals, exceptions, and emergency access should leave a record, so accountability is real rather than nominal.
- Make finding problems safe, not costly. If whoever spots an issue is saddled with owning it, people quietly stop looking, so reward detection and route the fix deliberately. And whoever is accountable for something must have the access to verify it themselves, because "done" reported by someone who cannot check it is not done.

What good looks like: for anything that can break, you can name its owner in seconds, and no task, environment, or risk sits forgotten in the gap between two teams.

## 14. Pave the road: make the right way the easy way

You do not get all these pillars' worth of good behavior by asking every team to reinvent it under deadline. You get it by building a paved road: the reusable templates, pipelines, and self-service tools that make doing the right thing the path of least resistance. If the secure, tested, observable way is also the hard way, people will route around it, and every project will solve the same problems slightly differently.

- Provide golden paths. A new service should start from a template that already has the formatting, tests, security gates, pipeline, observability, and infrastructure wired in, so a team inherits the standards instead of assembling them by hand. Ship these as real, copyable artifacts (configs, hooks, scripts) rather than documentation, so a new project is born already passing every gate on a thin end-to-end skeleton, instead of migrating toward compliance later.
- Make it self-service. Teams should provision what they need (an environment, a pipeline, an access grant) on demand through the platform, not by filing a ticket and waiting on someone.
- Treat the platform as a product. Its users are your own engineers, so it needs documentation, maintenance, and a feedback loop, not a one-time setup that quietly rots.

What good looks like: the fastest way to start a project is also the compliant, secure, well-instrumented way, and teams follow the standards because it is easier than not, rather than because they were told to.

## 15. Enforce and verify: a rule you don't check is already broken

Two failures look different but share a root: a rule nobody enforces, and a control nobody tested. "Please remember to" decays within a week, and four security layers that everyone assumed were working can each turn out to be absent in fact. So make every standard a gate a machine enforces, and then treat every control as a hypothesis until someone has proven, independently and end to end, that it holds.

- Make the standard executable. Wire formatting, linting, tests, coverage, security scans, and commit hygiene into commit hooks and the pipeline, so a violation blocks the change instead of relying on someone to catch it in review.
- Prefer failing loud, and test the bypass. A check that stays green for the wrong reason, or a control nobody has tried to get around, protects nothing. Make sure each gate can actually fail on the thing it exists to catch, and that each control (a firewall, an auth path) cannot simply be gone around.
- Do not let people opt out silently. Bypasses and suppressions should be rare, visible, and justified in writing. If a rule is genuinely wrong, change it in the open at the project level rather than muting it line by line.
- Compliance is not proof. A passed audit and a green checklist describe paperwork, not reality. The standard is "show me how you check it, and let me run it myself," with evidence an independent person can reproduce.
- Audit the whole path, and fix the class. The dangerous gap usually lives in the seam between systems, where each one passes its own review, so test the path a real attacker would walk. When you find a flaw, assume it repeats wherever the pattern does, and close every instance, not just the copy you found.
- Spend human judgment where it counts. When machines own the gates, review is freed for what only people can weigh: the design, the naming, and whether this is even the right change to make.

What good looks like: doing the right thing is the automatic default and doing the wrong thing trips a visible wire, and nobody can tell you a control is fine because it is supposed to be, because they can show you the check and you can run it yourself.

## 16. Measure whether you are improving: keep score with a shared yardstick

Opinions about whether a team is getting better are cheap; a small set of agreed metrics settles the argument. Measure the performance of your delivery system as a whole, not the output of individuals, and use a recognized framework so you are comparing against the industry rather than grading your own homework.

- Track the four DORA metrics as your baseline: deployment frequency (how often you ship), lead time for changes (from commit to production), change failure rate (how often a release causes a problem), and time to restore service (how fast you recover). Together they balance speed against stability, so you cannot game one by sacrificing the other.
- Pair those delivery metrics with flow metrics such as cycle time, throughput, and work in progress, which show where work stalls between an idea and its release.
- Treat them as system metrics, not a stick to beat people with. They describe the pipeline and the process, and they improve when you fix the system rather than when you push individuals harder.
- Watch the trend, not the snapshot. A single reading means little; the direction over weeks tells you whether the rest of these pillars are actually paying off.
- Treat cost as a first-class metric. Cloud spend is a latency-shaped metric: measure it per service, alert on unexplained growth, and design components to cost close to nothing when idle, so scale brings revenue in faster than it brings a bill.

What good looks like: you can say, with numbers, whether this quarter's delivery was faster and more stable than last quarter's, and every change to how you work is judged by whether it moved those metrics.

## 17. Obsess over the whole experience: the product is more than the interface

You can satisfy every pillar above and still build something people quietly abandon, because none of them is about the person on the other side. The experience your user actually has is the whole thing: not just the screen, but how fast it responds, how little it asks of them, how it treats a payment and a mistake, and whether support treats them like a person. Care about that as much as you care about the code.

- Treat the entire journey as the product. Speed, payments, delivery, error messages, and customer service are all part of the experience, not adjuncts to it. A fast checkout and a human answer to a problem do more for trust than another feature.
- Earn trust rather than extract a sale. Being honest with users, even when it costs a conversion, is what turns them into people who come back. A short-term win that erodes trust is the most expensive kind.
- Design for your users, not the demo. Ground decisions in how real people actually behave, which can differ sharply by market and culture, rather than in what looks good in a pitch. Watch what they do, not what they say they will do.
- Let technology serve the person, not replace them. Automation should remove friction and give people back their time, not paper over a worse experience. The human touch is the part hardest to copy, so protect it.
- Speak the user's language. Hold every user-facing string in a catalog keyed by meaning, never hardcoded in the interface, so the product can be localized for the markets you serve — which may differ sharply from your home one.

What good looks like: users call the product a relief to use, they trust you with the hard moments (their data, their money, their mistakes), and the reasons they stay are things a competitor cannot clone in a sprint.

## 18. Validate before you build: make sure it is worth building at all

I have listed this rule last, but in time it comes first, before a single line of any of the others above. The most expensive software is not the badly built kind. It is the beautifully built kind that nobody needed. Every other pillar makes you good at building the thing right; none of them checks that it is the right thing. That check is your job before the project starts, and again at every fork along the way.

- Talk to real users before you write code. A handful of honest conversations with the people who would actually use and pay for this teaches you more than a quarter of building on assumptions. Ask about their current problem, not your idea.
- Test demand with the cheapest thing that can carry it. A landing page, a mockup, a manual concierge version done by hand: put something in front of real people and watch what they do, not what they say they would do.
- Set a dated, honest go/no-go. Before committing to build, write down what would make this worth pursuing (a few committed design partners, a pricing hypothesis that survives contact with a buyer) and then decide, on the record, to proceed or stop. A "no" is a cheap win here and an expensive one later.
- Keep validating after launch. The same discipline applies to every feature: the cost of building the wrong one does not disappear once the project exists.

What good looks like: you can point to the evidence that someone actually wants what you are building, and you have killed at least one plausible idea before it turned into code.

## The stack is a detail

None of these eighteen pillars name a language, a database, or a cloud, and that is deliberate. The tools date; these rules have not, because they describe how a team works rather than what it works with.

So a new project's first decision is not what to build with. It is whether the thing is worth building at all, and then how to work: consistently and simply, built on clean boundaries, with proof instead of hope, secure, private, and isolated by default, boring to deliver, run on as little as you operate yourself, ready to fail and recover, observable, transparent to everyone who depends on it, clearly owned, paved so the right way is the easy way, held to standards a machine enforces and independently verifies, measured against a shared yardstick, and obsessed over from the user's side. Agree these eighteen before you argue about the stack, and the stack becomes a detail you can change later without drama.

The encouraging part is that none of this requires a large team or a big budget. It requires deciding, on day one, that quality, security, reliability, and transparency are defaults rather than features you will get to when there is time. Because there is never time later.

## See also

- [Core Values: what my writing says I stand for](core-values-one-pager.md) — the five values these eighteen pillars serve.
- [The Global Rules: Do and Don't, with Examples](global-rules-dos-and-donts.md) — a Do/Don't with code for every sub-concept above.
