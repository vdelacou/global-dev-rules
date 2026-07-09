================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Agree on How You Work Before You Argue About the Stack

SUBTITLE
The eighteen rules that outlast every framework, database, and cloud you will pick this decade.

TAGS   (5 max, each 25 characters or fewer)
Software Architecture, Engineering Leadership, Best Practices, Technical Debt, Startups

COVER IMAGE   upload this file from the covers folder
covers/01-the-premise.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 1 of 9 of The Global Rules Every New Project Should Have.*

The first question on a new project is almost always the wrong one. Which framework, which database, which cloud. People reach for the stack before they agree on how they intend to work, and then spend the next two years paying for it.

I have watched this play out on enough projects to stop finding it interesting. The tools you argue about today are already half-replaced. The frameworks I would have defended a few years ago are legacy now. That is the point: underneath the specifics sits a much smaller set of principles that do not depend on any technology at all.

I call them the global rules. They hold whether you build in any language, deploy to any cloud, and start this year or in ten. Agree on them before you write a line of code. Every one of them is far harder to add later than to design in from the start.

There are eighteen. The through-line is one sentence: decide how the project works before you decide what it is built with. The whole map is below, one crisp line each, so you can see the shape before we go deep in the parts that follow.

**1. Consistency.** The codebase should read as if one careful person wrote it, because reading and changing code is the real cost, and every inconsistency taxes it.

**2. Simplicity by default.** The cheapest, safest, most maintainable code is the code that does not exist, so do the least that works and delete before you add.

**3. Keep clean boundaries.** You should be able to swap the database, the web framework, or the entire visual theme one at a time, and the parts that should not care never notice.

**4. Proof over hope.** "It works on my machine" is not evidence. Confidence comes from automated checks anyone can run, not from the optimism of whoever wrote the last change.

**5. Secure by default.** Assume someone hostile is paying attention. The cheapest vulnerability to fix is the one you never introduce, so make the safe choice the only easy choice.

**6. Private by default.** Every piece of personal data you hold is a liability, not a trophy. The only data that can never leak is the data you chose not to collect.

**7. Isolate by default.** One user's data must never reach another. Treat the boundary between tenants as architecture, not a filter you remember to add.

**8. Delivery should be boring.** Shipping is a routine, not an event. Manual, heroic deployments are slow, unrepeatable, and where the surprises come from. Boring is the highest compliment.

**9. Run as little as possible yourself.** Every server you babysit is work that is not your product. Rent the undifferentiated parts and keep people out of the machines entirely.

**10. Design for failure.** Everything breaks, so plan the recovery. Reliability is not the absence of failure, it is fast, well-practiced recovery from it.

**11. Make it observable.** You cannot fix what you cannot see. In production the question is never "is something wrong," it is "what, where, and since when."

**12. No black boxes.** Anyone with a stake should be able to see the real state at any moment. Opacity is where projects quietly rot.

**13. Clear ownership.** Everything, and every rule on this list, needs a name next to it. Shared ownership with no name attached is how things fall through the gap between two teams.

**14. Pave the road.** Make the right way the easy way. If the secure, tested, observable path is also the hard path, people route around it.

**15. Enforce and verify.** A rule you do not check is already broken. "Please remember to" decays within a week, so make every standard a gate a machine enforces, then prove the gate can actually fail.

**16. Measure whether you are improving.** Opinions about whether a team is getting better are cheap. A small set of agreed metrics settles the argument.

**17. Obsess over the whole experience.** You can satisfy every rule above and still build something people quietly abandon, because none of them is about the person on the other side.

**18. Validate before you build.** The most expensive software is not the badly built kind, it is the beautifully built kind that nobody needed. Check that it is the right thing before the project starts, and again at every fork.

Read that list twice and notice what is missing. Not one of the eighteen names a language, a database, or a cloud. That is deliberate. The tools date; these rules have not, because they describe how a team works rather than what it works with.

Now the part that surprises people. None of this requires a large team or a big budget. It requires one decision on day one: that quality, security, reliability, and transparency are defaults, not features you will get to when there is time. There is never time later. The team that "will add tests once things settle down" is the same team explaining an outage six months in, and the settling never came.

So treat the stack as a detail. It is the thing you can change later without drama, precisely because you got the rules right first. A project that agrees on these eighteen can migrate its database, replace its framework, and swap its cloud provider as ordinary maintenance. A project that skipped them treats each of those as a rewrite, because everything is tangled into everything else.

Over the next eight parts I will take these eighteen apart and show you the code, the Do next to the Don't, in TypeScript, Java, and React. Not theory. The exact shapes that make a rule real in a codebase instead of a slogan in a wiki.

We start where the money is. In the next part, **Write Code That Lasts**, I cover the two rules you pay for on every single file you ever open again: consistency and simplicity.
