================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Secure, Private, Isolated: The Three Defaults That Keep You Off the Front Page

SUBTITLE
Security, privacy, and tenant isolation are starting conditions, not phases you schedule near launch, and the code shows the difference.

TAGS   (5 max, each 25 characters or fewer)
Security, Data Privacy, Cybersecurity, Software Development, Best Practices

COVER IMAGE   upload this file from the covers folder
covers/05-protect-the-people.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 5 of 9 of The Global Rules Every New Project Should Have.*

Every other rule in this series protects the code. These three protect the people the code is for. Get them wrong and the failure is not a bug ticket, it is a breach notification, a regulator, and a headline with your company's name in it.

The through-line for all three is one word: default. Security, privacy, and isolation are not phases you schedule near launch, they are starting conditions. The cheapest vulnerability to fix is the one you never introduce. The only personal data that can never leak is the data you chose not to collect. The cross-tenant path that cannot exist is the one the architecture never allowed. Retrofitting any of these onto a finished system is expensive and almost always incomplete.

So this part covers three of the eighteen rules together: **secure by default**, **private by default**, and **isolate by default**. Make the safe choice the only easy choice, and most of the danger never gets written.

## Secure by default: assume someone hostile is paying attention

Security is not a feature you bolt on before launch. It is a property of the default path, and the goal is to make the safe choice the only one a developer can make without extra effort.

Start with secrets, because this is where the cheapest catastrophes live. Keys, passwords, and tokens belong in a secret manager, never in source, config, wikis, tickets, logs, or shared drives. A secret committed to git is a secret in the history forever, and deleting today's copy does not un-leak it.

```ts
// DON'T: literal secret in source, travels through git history forever
const stripe = new Stripe('sk_live_51H8xY2eZ...');
console.log('using key', process.env.STRIPE_KEY);
// DO: injected at runtime from the secret manager, never logged
const key = await secrets.get('stripe/api-key'); // resolved from the secret manager
if (!key.ok) return { ok: false, error: 'missing_secret' } as const;
const stripe = new Stripe(key.value);
logger.info('stripe client ready'); // value never leaves the store
```

Two sins in the first block, and the second is the sneaky one. The literal key is obvious. The `console.log` of the key is how a secret ends up in your logging platform, your error tracker, and three third-party services you forgot ingest logs. The value should never leave the store, and it certainly should never be logged.

The second rule under this pillar is the one people argue with, and they are wrong. Do not build authentication or cryptography yourself. Use a vetted library or an identity provider for login, sessions, tokens, and password handling. The auth code you hand-roll is the auth code with the subtle, expensive bug, and you will not be the one to find it. An attacker will.

This matters most on the frontend, where a bad choice is one line and permanently exploitable.

```tsx
// DON'T: a hand-rolled login that stores a JWT in localStorage, readable by any XSS on the page
const token = await api.post('/login', creds).then((r) => r.text());
localStorage.setItem('jwt', token);
// DO: use the identity provider's SDK; the session rides in an httpOnly, secure cookie the JS cannot read
const { login } = useAuth();   // OIDC/OAuth redirect handled by the provider
await login();                 // server responds Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
```

A token in `localStorage` is readable by any JavaScript on the page, which means the first cross-site scripting bug anywhere in your app hands an attacker a valid session. An `httpOnly` cookie is invisible to JavaScript by construction. Same session, but the entire class of "XSS steals the token" attack does not apply. You did not defend against it, you removed it.

The last piece of secure-by-default is validation and authorization, and the rule has two halves people love to get half-right. Validate every untrusted value at one checkpoint before it reaches anything dangerous. And authorize on the server, never in the interface.

```ts
// DON'T: raw body straight into a query; "admin" trusted from the client
app.post('/users', (c) => db.insert(users).values(c.req.json()));
// DO: validate at the edge, authorize server-side from the verified token
app.post('/users', zValidator('json', NewUser), async (c) => {
  if (c.get('claims').role !== 'admin') return c.json({ error: 'forbidden' }, 403);
  return c.json(await createUser(c.req.valid('json')));
});
```

Hiding a button in the UI is a courtesy to the user, not a control. The server must authorize every call as if the interface does not exist, because to an attacker with a terminal, it does not. A crafted request from a non-admin should get a 403 whether or not a button was ever rendered.

## Private by default: personal data is a liability, not an asset

Every piece of personal data you hold is a responsibility and a risk, not a trophy. And the law now follows the user, not your office. If you serve people in a region, its privacy regime generally applies to you even when your company and servers sit elsewhere. The penalties are sized to hurt, often a share of global revenue. So the discipline is not optional and it is not regional.

The cheapest data to protect is the data you never collected. Gather only what a stated purpose requires, and be able to say that purpose in a sentence.

```ts
// DON'T: hoover up sensitive extras with no purpose and no consent
const Signup = z.object({
  email: z.string().email(), ssn: z.string(), religion: z.string(), dob: z.string(),
});
// DO: only what the purpose needs; sensitive fields gated behind explicit consent
const Signup = z.object({
  email: z.string().email(),                 // needed to create the account
  displayName: z.string().min(1),            // needed to render the profile
}); // no SSN, no religion; a health field would require a separate consent flow
```

If you cannot name the purpose for a field, delete the field. Every column of personal data you do not hold is a column that cannot leak, cannot be subpoenaed, and cannot show up in a breach.

Then keep the data you do collect out of the places it leaks by accident. No personal data in logs, and none in URLs or query strings, which get written to access logs, cached by proxies, and handed to analytics. On the frontend this shows up in the address bar, where an email in a query string lands in browser history, referrer headers, and every analytics tool on the page.

```tsx
// DON'T: an email in the query string lands in browser history, referrer headers, and analytics
navigate(`/search?email=${encodeURIComponent(email)}`);
// DO: keep personal data out of the URL; pass it in state or a POST body, and show only an opaque id in the path
navigate('/search', { state: { email } }); // or POST the search; the address bar shows /search only
```

Searching by someone's name should be a POST, not a GET that ends up in three logs. Same for backend logging: log a hashed or pseudonymous reference, never the raw email or token, and let a redactor drop known-sensitive keys before anything is written.

One more rule under this pillar, and teams break it constantly because it is convenient. Never copy production data into a test or development environment. Those environments have weaker controls and wider access, which makes them the cheapest place to lose real data.

```ts
// DON'T: real customers restored into the dev database
const rows = await pgRestore('prod_backup.sql'); // live PII now in dev + backups
await db.insert(customers).values(rows);
// DO: deterministic synthetic data, correct shape, zero real subjects
const rows = Array.from({ length: 500 }, (_, i) => ({
  email: `user${i}@example.test`, taxId: fakeTaxId(i), country: pickCountry(i),
})); // faker-style, seeded for reproducibility
await db.insert(customers).values(rows);
```

"We just needed realistic data to debug" is how a production dump ends up on a laptop, in a dev database with no access controls, and in a nightly backup nobody is watching. I have watched exactly that convenience turn into a very bad week. Generate synthetic fixtures with the right shape and volume. They debug just as well and there is no real person to lose.

## Isolate by default: one user's data must never reach another

The worst failure in any system that holds data for more than one person is the quiet one: customer A sees customer B's records. It rarely announces itself, it breaks the deepest promise you make to users, and by the time you notice, the exposure is already history. Treat the boundary between one tenant and the next as a first-class part of the architecture, not a filter you remember to add.

It starts with where you get the tenant from. Derive who a request acts for from a verified token, never from a URL, header, or field the caller can change.

```ts
// DON'T: tenant comes from the path, so any caller can name another tenant
app.get('/orgs/:orgId/invoices', (c) =>
  c.json(listInvoices(c.req.param('orgId')))); // forge orgId, read anyone
// DO: tenant comes from the verified JWT, the URL cannot override it
app.get('/invoices', requireAuth, (c) => {
  const orgId = c.get('claims').org_id; // signed by the IdP, not caller-supplied
  return c.json(listInvoices(orgId));
});
```

The first route lets any authenticated user type a different org id into the URL and read someone else's invoices. The org must come from the signed token, which the caller cannot forge, not from a path segment, which they can edit in one second.

Then fail closed. Missing ownership context should return nothing, not everything and not an error. A bug should surface as absent data, never as someone else's data.

```ts
// DON'T: missing tenant falls through and selects across all tenants
function invoicesFor(orgId?: string) {
  const q = db.select().from(invoices);
  return orgId ? q.where(eq(invoices.orgId, orgId)) : q; // undefined => everything
}
// DO: no tenant means no data, never a wildcard read
function invoicesFor(orgId: string | undefined): Promise<Invoice[]> {
  if (!orgId) return Promise.resolve([]); // fail closed, empty result
  return db.select().from(invoices).where(eq(invoices.orgId, orgId));
}
```

That `undefined => everything` in the first version is a full cross-tenant dump waiting for one upstream bug to leave `orgId` unset. Fail-closed turns the same bug into an empty screen. One is an incident, the other is a confused user filing a ticket.

None of this is real until you prove it, per feature, on every change. Every endpoint ships a test where one owner's credentials cannot reach another owner's resource. Isolation you have not tested is isolation you do not have.

```ts
// DON'T: only the happy path is tested, cross-tenant access never exercised
test('owner reads own invoice', async () => {
  const res = await app.request(`/invoices/${own}`, authAs(orgA));
  expect(res.status).toBe(200);
});
// DO: assert org A cannot reach org B's resource, and it looks absent (404)
test('cross-tenant read is not_found', async () => {
  const res = await app.request(`/invoices/${orgBInvoice}`, authAs(orgA));
  expect(res.status).toBe(404); // not 403, so existence is not disclosed
});
```

Note the 404, not a 403. A 403 says "this exists but you cannot have it," which itself leaks that the resource exists. A 404 says nothing at all. Org A learns only that there is no such invoice for them, which is the truth.

Three rules, one idea: put the protection in the default path so the dangerous thing has to be typed on purpose. A lost laptop is an incident you contain calmly. A hostile request gets a 403. A missing tenant returns an empty list. None of them becomes the story about your company.

You have protected the code and the people. Now the work has to actually ship. In Part 6, **Shipping Should Be Boring**, I cover the two rules that turn deployment from an event people gather to watch into a non-event nobody notices: automate the path to production, and run as little as possible yourself.
