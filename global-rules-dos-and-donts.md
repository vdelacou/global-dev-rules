# The Global Rules: Do and Don't, with Examples

A companion to [The Global Rules Every New Project Should Have](global-rules-every-new-project.md). For each of the eighteen pillars, every sub-concept below gives a clear Do and Don't with short, contrasting code examples in TypeScript and Java. The examples favor the language and its standard library; a framework appears only where the concept needs one to be shown. Where a principle is not application code (infrastructure, delivery, ownership, metrics, product), the example is the real artifact instead: a config file, a CI step, an infrastructure block, or a command, with the tool named and explained in comments.

Errors are modeled as values throughout: in TypeScript as a Result union, in Java as a native sealed Result type. Every example below assumes these two shapes:

```ts
type Result<T, E = string> =
  | { ok: true; value: T }
  | { ok: false; error: E };
```

```java
sealed interface Result<T> permits Ok, Err {}
record Ok<T>(T value) implements Result<T> {}
record Err<T>(String error) implements Result<T> {}
```

---

## Contents

1. [Pillar 1: Consistency](#pillar-1-consistency)
2. [Pillar 2: Simplicity by default](#pillar-2-simplicity-by-default)
3. [Pillar 3: Keep clean boundaries](#pillar-3-keep-clean-boundaries)
4. [Pillar 4: Proof over hope](#pillar-4-proof-over-hope)
5. [Pillar 5: Secure by default](#pillar-5-secure-by-default)
6. [Pillar 6: Private by default](#pillar-6-private-by-default)
7. [Pillar 7: Isolate by default](#pillar-7-isolate-by-default)
8. [Pillar 8: Delivery should be boring](#pillar-8-delivery-should-be-boring)
9. [Pillar 9: Run as little as possible yourself](#pillar-9-run-as-little-as-possible-yourself)
10. [Pillar 10: Design for failure](#pillar-10-design-for-failure)
11. [Pillar 11: Make it observable](#pillar-11-make-it-observable)
12. [Pillar 12: No black boxes](#pillar-12-no-black-boxes)
13. [Pillar 13: Clear ownership](#pillar-13-clear-ownership)
14. [Pillar 14: Pave the road](#pillar-14-pave-the-road)
15. [Pillar 15: Enforce and verify](#pillar-15-enforce-and-verify)
16. [Pillar 16: Measure whether you are improving](#pillar-16-measure-whether-you-are-improving)
17. [Pillar 17: Obsess over the whole experience](#pillar-17-obsess-over-the-whole-experience)
18. [Pillar 18: Validate before you build](#pillar-18-validate-before-you-build)

---

## Pillar 1: Consistency

A codebase that reads as if one careful author wrote it lets a machine, not a reviewer, defend the style.

### 1.1 One committed config for style
**Do:** Commit a single formatter and linter config that every developer and CI job shares.
**Don't:** Keep a second local or per-folder config that quietly diverges from the shared one.

TypeScript:
```ts
// DON'T: a local override file drifts from the repo config, so two files format differently
// .eslintrc.local.json (gitignored, only on one laptop)
{ "rules": { "semi": "off", "quotes": ["error", "double"] } }
// DO: one root config, committed, referenced by every script and by CI
// eslint.config.ts (the only lint source of truth)
export default [{ rules: { semi: ["error", "always"], quotes: ["error", "single"] } }];
// package.json: "lint": "eslint .", CI runs the same command, no flags to disagree about
```

Java:
```java
// DON'T: each module pins its own formatter version, so output depends on who built last
// module-a/pom.xml uses spotless 2.30, module-b/pom.xml uses 2.43, results differ
<plugin><groupId>com.diffplug.spotless</groupId><version>2.30.0</version></plugin>
// DO: one version and one ruleset in the parent pom, every module inherits it
// parent pom.xml (single source), children declare nothing
<plugin>
  <groupId>com.diffplug.spotless</groupId><artifactId>spotless-maven-plugin</artifactId>
  <version>2.43.0</version>
  <configuration><java><googleJavaFormat/></java></configuration>
</plugin>
```

### 1.2 Cap complexity and duplication
**Do:** Keep functions small and use guard clauses so the happy path stays flat.
**Don't:** Nest conditionals deeply or copy-paste the same branch into several call sites.

TypeScript:
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

Java:
```java
// DON'T: same discount branch copy-pasted, one edit now means three edits later
double checkout(Order o) { if (o.tier == PREMIUM) return o.subtotal * 0.8; return o.subtotal; }
double invoice(Order o)  { if (o.tier == PREMIUM) return o.subtotal * 0.8; return o.subtotal; }
double quote(Order o)    { if (o.tier == PREMIUM) return o.subtotal * 0.8; return o.subtotal; }
// DO: one guarded method, called from the three sites, single point of change
double net(Order o) {
  if (o.tier != Tier.PREMIUM) return o.subtotal;
  return o.subtotal * 0.8;
}
```

## Pillar 2: Simplicity by default

Reach for the smallest thing that solves the real problem, and prefer deleting code to writing it.

### 2.1 Do the least that works
**Do:** Use the language and standard library before pulling in a dependency or hand-rolling code.
**Don't:** Add a package (or write your own utility) for something the platform already gives you.

TypeScript:
```ts
// DON'T: a dependency plus custom code to do what the runtime already ships
import { v4 as uuidv4 } from "uuid";
function newId(): string { return uuidv4(); }
// DO: the platform already has it, one less dependency to audit and update
function newId(): string {
  return crypto.randomUUID();
}
```

Java:
```java
// DON'T: hand-written null-safe getter chain reinvents a stdlib method
String city(User u) {
  if (u != null && u.address() != null && u.address().city() != null) return u.address().city();
  return "unknown";
}
// DO: Optional expresses the same intent with the standard library
String city(User u) {
  return Optional.ofNullable(u).map(User::address).map(Address::city).orElse("unknown");
}
```

### 2.2 Delete before you add
**Do:** Prefer the change that removes code and keeps the boring, obvious shape.
**Don't:** Add a clever indirection when deleting a branch would do the job.

TypeScript:
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

Java:
```java
// DON'T: an abstract base plus one subclass, ceremony around a single behavior
abstract class Greeter { abstract String greet(String n); }
class EnGreeter extends Greeter { String greet(String n) { return "Hi " + n; } }
// DO: delete the hierarchy, the method is enough until a second language is real
final class Greeter {
  String greet(String name) { return "Hi " + name; }
}
```

### 2.3 Earn abstractions with the Rule of Three
**Do:** Tolerate a little duplication and wait for the third occurrence before extracting.
**Don't:** Extract a shared abstraction the moment you see the second similar case.

TypeScript:
```ts
// DON'T: a premature generic helper forced to fit two callers that will soon diverge
function format<T>(x: T, kind: "date" | "money"): string {
  return kind === "date" ? new Date(x as number).toISOString() : `$${x}`;
}
// DO: keep the two simple call sites, extract only when a third proves the pattern
const asDate = (ms: number) => new Date(ms).toISOString();
const asMoney = (n: number) => `$${n.toFixed(2)}`;
// two usages today, no shared abstraction yet, the shape is still forming
```

Java:
```java
// DON'T: a config-object abstraction built on the second caller, over-parameterized
record ExportOpts(boolean header, char sep, boolean quote, String charset) {}
String export(List<Row> rows, ExportOpts o) { /* handles cases nobody asked for */ return ""; }
// DO: two direct methods, duplication is cheaper than the wrong abstraction
String toCsv(List<Row> rows)  { return join(rows, ','); }
String toTsv(List<Row> rows)  { return join(rows, '\t'); }
// extract a shared exporter when the third format arrives
```

### 2.4 Defer the build, not the seam
**Do:** Introduce the interface now and implement only the version you actually need today.
**Don't:** Build the heavy, feature-rich implementation before there is a real requirement for it.

TypeScript:
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

Java:
```java
// DON'T: a fully pooled, sharded cache built before any load justifies it
class RedisCache { /* connection pool, sharding, TTL sweeper, eviction policy */ }
// DO: an interface plus a trivial in-memory version, the seam is ready for the real one
interface Cache<K, V> { Optional<V> get(K key); void put(K key, V value); }
final class MapCache<K, V> implements Cache<K, V> {
  private final Map<K, V> m = new ConcurrentHashMap<>();
  public Optional<V> get(K k) { return Optional.ofNullable(m.get(k)); }
  public void put(K k, V v) { m.put(k, v); }
}
```

### 2.5 Simplicity is not negligence
**Do:** Keep input validation, error handling, and security even when trimming everything else.
**Don't:** Simplify a path by dropping the checks that keep it correct and safe.

TypeScript:
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

Java:
```java
// DON'T: skips validation, lets a null or bad amount reach the ledger
@POST public Response charge(ChargeRequest req) {
  ledger.debit(req.accountId(), req.amount()); return Response.ok().build();
}
// DO: constraints enforced, invalid input never reaches the domain
public record ChargeRequest(@NotNull UUID accountId, @Positive BigDecimal amount) {}
@POST public Response charge(@Valid ChargeRequest req) {
  ledger.debit(req.accountId(), req.amount());
  return Response.ok().build();
}
```

## Pillar 3: Keep clean boundaries

Point every dependency inward, hide each external thing behind a port, and let the domain stay ignorant of the world.

### 3.1 Point dependencies inward
**Do:** Have the domain depend on an interface it defines, never on the framework or driver.
**Don't:** Import the ORM, HTTP client, or cloud SDK from inside business logic.

TypeScript:
```ts
// DON'T: the use-case imports Drizzle, so the domain now knows about Postgres
import { db } from "../infra/drizzle";
async function activate(id: string) {
  await db.update(accounts).set({ active: true }).where(eq(accounts.id, id));
}
// DO: the use-case depends on a port it owns, infra implements it elsewhere
interface AccountStore { setActive(id: string, active: boolean): Promise<void>; }
async function activate(store: AccountStore, id: string): Promise<void> {
  await store.setActive(id, true);
}
```

Java:
```java
// DON'T: the service reaches for Panache, coupling the domain to Hibernate
@ApplicationScoped class AccountService {
  void activate(UUID id) { Account.update("active = true where id = ?1", id); }
}
// DO: the domain calls a port, the JPA adapter lives in the infrastructure layer
interface AccountStore { void setActive(UUID id, boolean active); }
@ApplicationScoped class AccountService {
  private final AccountStore store;
  AccountService(AccountStore store) { this.store = store; }
  void activate(UUID id) { store.setActive(id, true); }
}
```

### 3.2 Put every external thing behind a port
**Do:** Wrap each external system in a port with a real adapter and an in-memory fake, wired once.
**Don't:** Call a third-party client directly from many call sites with no seam to fake.

TypeScript:
```ts
// DON'T: the S3 SDK is called inline, so every test needs a network or a heavy mock
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
async function saveReceipt(bytes: Uint8Array) {
  await new S3Client({}).send(new PutObjectCommand({ Bucket: "r", Key: "k", Body: bytes }));
}
// DO: one port, a real adapter and a fake, composed at the root
interface Blobs { put(key: string, body: Uint8Array): Promise<void>; }
class MemoryBlobs implements Blobs {
  store = new Map<string, Uint8Array>();
  async put(k: string, b: Uint8Array) { this.store.set(k, b); }
}
// composition root picks S3Blobs in prod, MemoryBlobs in tests
```

Java:
```java
// DON'T: the SDK client is new'd in the service, untestable without real credentials
@ApplicationScoped class ReceiptService {
  void save(byte[] b) { S3Client.create().putObject(req -> req.bucket("r").key("k"), fromBytes(b)); }
}
// DO: a port with two implementations, the container injects whichever fits
interface Blobs { void put(String key, byte[] body); }
@ApplicationScoped class MemoryBlobs implements Blobs {
  final Map<String, byte[]> store = new ConcurrentHashMap<>();
  public void put(String key, byte[] body) { store.put(key, body); }
}
// @IfBuildProfile("prod") selects the S3 adapter, tests get the in-memory one
```

### 3.3 Seal the presentation behind a design system
**Do:** Make presentational components take props and read semantic tokens, with no fetching inside.
**Don't:** Hardcode hex colors or inline styles, or fetch and compute business data in a component.

TypeScript:
```ts
// DON'T: raw hex, inline style literal, and a fetch buried in a presentational component
function Badge({ userId }: { userId: string }) {
  const [u, setU] = useState<User>();
  useEffect(() => { fetch(`/api/u/${userId}`).then(r => r.json()).then(setU); }, [userId]);
  return <span style={{ background: "#16a34a", color: "#fff" }}>{u?.plan}</span>;
}
// DO: props in, semantic tokens for color, no data logic inside
function Badge({ label, tone }: { label: string; tone: "success" | "muted" }) {
  return <span className={`badge badge--${tone}`}>{label}</span>;
}
// .badge--success { background: var(--color-success); color: var(--color-on-success); }
```

Java:
```java
// Frontend-only concept: no Java counterpart.
```

### 3.4 The backend is a client-agnostic API
**Do:** Expose one resource-shaped API that every client (web, iOS, Android, third-party integrations) consumes the same way.
**Don't:** Shape endpoints to a single screen, so a new screen or a new client forces a new backend endpoint.

TypeScript:
```ts
// DON'T: one endpoint per screen; the backend now knows the mobile home layout
app.get('/mobile-home-screen', (c) =>
  c.json({ banner, greeting, recentOrders, promoWidget }));
// DO: stable resource endpoints; each client (web, iOS, Android) composes its own screen
app.get('/orders', (c) => c.json(listOrders(c.get('userId'))));
app.get('/promotions', (c) => c.json(listActivePromotions()));
```

Java:
```java
// DON'T: a screen-shaped response tied to today's mobile home screen
@GET @Path("/mobile-home-screen")
public HomeScreenDto home() { return buildHomeScreen(userId); }
// DO: resource endpoints any client can compose the same way
@GET @Path("/orders") public List<Order> orders() { return orders.forUser(userId); }
@GET @Path("/promotions") public List<Promotion> promotions() { return promotions.active(); }
```

### 3.5 Build the frontend against a contract, not a running backend
**Do:** Put the frontend's data access behind an interface with a real API client and an in-memory fake, so the UI is built and tested in parallel with the backend.
**Don't:** Call fetch or an HTTP client straight from components, so nothing works until the backend is deployed.

TypeScript (React):
```tsx
// DON'T: the component is wired to a live endpoint; blocked until the API is deployed
function Orders() {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => { fetch('/api/orders').then((r) => r.json()).then(setOrders); }, []);
  return <OrderList orders={orders} />;
}

// DO: components depend on an OrderGateway interface; swap real and fake by config
// the gateway returns a Result, so a failure is a value the UI can render, not a throw that blanks the screen (see 10.2)
type OrderGateway = { list: () => Promise<Result<Order[], LoadError>> };
const httpOrders: OrderGateway = { list: () => api.get('/orders') }; // api.get already returns Result<Order[], LoadError>
const fakeOrders: OrderGateway = { list: async () => ({ ok: true, value: [{ id: '1', total: 80 }] }) }; // canned data in the contract's shape
// one wiring point picks the implementation from an env flag; UI code never changes
export const orderGateway: OrderGateway = env.USE_FAKE_API ? fakeOrders : httpOrders;
```

Backend side (any stack):
```text
Publish the API contract first (OpenAPI or a shared schema), before the implementation.
The frontend generates a typed client and a mock from it, so both teams code to the
contract, not to each other's calendars. When the real endpoint lands, only the wiring
flips from fake to real, and no UI code changes.
```

### 3.6 The internal model is yours, not the API's shape
**Do:** Translate the API response into your own model at the boundary, and let the two shapes differ.
**Don't:** Pass raw API DTOs through the app, so every field the API renames breaks code everywhere.

TypeScript (React):
```tsx
// DON'T: the raw API shape (snake_case, cents, nullable) leaks into every component
type ApiOrder = { order_id: string; total_cents: number; customer: { first_name: string | null } };
const OrderRow = ({ o }: { o: ApiOrder }) =>
  <div>{o.customer.first_name ?? 'Guest'} pays {o.total_cents / 100}</div>;

// DO: the gateway maps the DTO into an internal model the UI owns
type Order = { id: string; total: Money; customerName: string };   // internal model, your shape
const toOrder = (d: ApiOrder): Order => ({                         // the one mapping point
  id: d.order_id,
  total: Money.fromCents(d.total_cents),
  customerName: d.customer.first_name ?? 'Guest',
});
// components use Order; if the API renames a field, only toOrder changes
const httpOrders: OrderGateway = {
  list: async () => {
    const dto = await api.get<ApiOrder[]>('/orders');           // Result<ApiOrder[], LoadError>
    if (!dto.ok) return dto;                                     // failure propagates with the same error
    return { ok: true, value: dto.value.map(toOrder) };          // success maps the DTO into the internal model
  },
};
```

Java (JPA):
```java
// DON'T: the JPA entity is serialized straight to JSON, so the DB shape becomes the API shape
@GET public List<Order> all() { return orderRepo.listAll(); } // Order is the @Entity

// DO: map the domain to a response record at the boundary; the two can differ
@GET public List<OrderResponse> all() {
  return service.list().stream().map(OrderResponse::from).toList();
}
// OrderResponse is shaped for the API; the domain Order stays internal, DTOs never enter use-cases
```

### 3.7 The domain model is not the database model
**Do:** Model the business in the domain and storage in the database, and map between them at the repository.
**Don't:** Let the ORM entity or table shape be your domain object, so persistence concerns dictate business behavior.

TypeScript (Drizzle):
```ts
// DON'T: the Drizzle row is passed around as "the order"; nullable columns and DB shape leak everywhere
const row = await db.query.orders.findFirst({ where: eq(orders.id, id) });
if (row.status === 'PAID') { /* business logic reading raw DB columns */ }

// DO: a domain type with behavior; the repository maps row -> domain, and the DB shape stops there
type Order = { id: OrderId; lines: Line[]; status: OrderStatus };
const canRefund = (o: Order): boolean => o.status === 'paid' && o.lines.length > 0; // a business rule
const orderRepo = {
  find: async (id: OrderId): Promise<Order | null> => {
    const row = await db.query.orders.findFirst({ where: eq(orders.id, id) });
    return row ? toDomain(row) : null; // one mapping point
  },
};
```

Java (JPA):
```java
// DON'T: the @Entity is the model, so JPA annotations, lazy relations, and DB shape leak into business code
@Entity public class Order extends PanacheEntity { public String status; public List<Line> lines; }
if (order.status.equals("PAID")) { /* business logic running on a persistence object */ }

// DO: a domain type (record) with behavior; the repository maps the entity to it
public record Order(OrderId id, List<Line> lines, OrderStatus status) {
  public boolean canRefund() { return status == OrderStatus.PAID && !lines.isEmpty(); }
}
// OrderRepository loads the @Entity and returns the domain Order; the entity stays inside infra
```

### 3.8 Make the boundary testable
**Do:** Arrange things so a business-rule change touches the domain and never a UI or adapter.
**Don't:** Spread one rule across a component and an adapter so any change edits both.

TypeScript:
```ts
// DON'T: the tax rule lives in the React component, so a rate change edits the UI
function Total({ items }: { items: Item[] }) {
  const net = items.reduce((s, i) => s + i.price, 0);
  const total = net * 1.2; // VAT rule stranded in presentation
  return <div className="total">{total}</div>;
}
// DO: the rule sits in the domain, the component only renders the computed value
// core/pricing.ts
export const withVat = (net: number): number => net * VAT_RATE;
function Total({ total }: { total: number }) { return <div className="total">{total}</div>; }
```

Java:
```java
// DON'T: the rule is entangled with the JPA adapter, changing it edits persistence code
@ApplicationScoped class InvoiceRepo {
  BigDecimal total(UUID id) {
    var net = find(id).net();
    return net.multiply(new BigDecimal("1.20")); // VAT rule buried in the adapter
  }
}
// DO: the rule is pure domain, the adapter only fetches, swapping storage never touches it
final class Pricing {
  static BigDecimal withVat(BigDecimal net) { return net.multiply(VAT_RATE); }
}
// InvoiceRepo returns the entity, the caller applies Pricing.withVat
```

## Pillar 4: Proof over hope

Trust the code because layered tests, mutation scores, and a red-blocking pipeline say so, not because it looked right.

### 4.1 Test in layers
**Do:** Split tests into unit, integration, end-to-end, and performance, each owning a distinct risk.
**Don't:** Push every check into slow end-to-end tests and skip the fast layers underneath.

TypeScript:
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

Java:
```java
// DON'T: booting the whole app to check one arithmetic rule
@QuarkusTest class CheckoutIT {
  @Test void discount() { given().body(cart).post("/checkout").then().body("total", is(80)); }
}
// DO: JUnit owns the rule, REST Assured owns the endpoint, distinct fast and slow layers
class PricingTest {
  @Test void premiumDiscount() { assertEquals(80, Pricing.withDiscount(100, PREMIUM)); }
}
@QuarkusTest class CheckoutIT {
  @Test void endpointReturns200() { given().body(cart).post("/checkout").then().statusCode(200); }
}
```

### 4.2 Keep unit tests in milliseconds
**Do:** Keep a unit test in memory so it runs in milliseconds.
**Don't:** Touch a real database or network in a unit test.

TypeScript:
```ts
// DON'T: hits Postgres, so the "unit" test takes 300ms and needs a running DB
test("total", async () => { const o = await db.query.orders.findFirst(); expect(total(o)).toBe(80); });
// DO: pure input, runs in under a millisecond, no I/O
test("premium customer gets 20% off", () => {
  expect(total({ price: 100, tier: "premium" })).toBe(80);
});
```

Java:
```java
// DON'T: @QuarkusTest boots the app and the datasource for a pure calculation
@QuarkusTest class OrderIT { @Inject OrderRepo repo; @Test void total() { /* slow, needs DB */ } }
// DO: plain JUnit, no container, sub-millisecond
class OrderTest {
  @Test void premiumCustomerGets20PercentOff() {
    assertEquals(80, Order.total(new Order(100, Tier.PREMIUM)));
  }
}
```

### 4.3 Have a testing philosophy
**Do:** Turn every fixed bug into a permanent regression test that would have caught it.
**Don't:** Patch the bug and move on without a test that pins the behavior.

TypeScript:
```ts
// DON'T: the empty-cart crash is fixed in code but nothing stops it coming back
export function total(items: Item[]): number {
  if (items.length === 0) return 0; // fix applied, no test guards it
  return items.reduce((s, i) => s + i.price, 0);
}
// DO: the exact bug becomes a named test, red before the fix, green after
test("regression: empty cart totals to zero, not NaN", () => {
  expect(total([])).toBe(0);
});
```

Java:
```java
// DON'T: null-name NPE quietly patched, no test records what went wrong
String greet(User u) {
  var name = u.name() == null ? "there" : u.name(); // fix, but undocumented and unguarded
  return "Hi " + name;
}
// DO: a regression test names the defect and fails without the guard
class GreetTest {
  @Test void regressionNullNameGreetsGenerically() {
    assertEquals("Hi there", greet(new User(null)));
  }
}
```

### 4.4 Treat mutation testing as the real coverage KPI
**Do:** Judge a suite by its mutation score, using Stryker for TS and PIT for Java.
**Don't:** Trust a green line-coverage number that no assertion actually defends.

TypeScript:
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

Java:
```java
// DON'T: the method runs but no assertion, PIT reports every mutation as survived
@Test void runsDiscount() { Pricing.withDiscount(100, PREMIUM); /* covered, undefended */ }
// DO: assert the boundary so PIT's arithmetic and conditional mutants die
@Test void premiumDiscountIsExactlyTwentyPercent() {
  assertEquals(80, Pricing.withDiscount(100, PREMIUM));
}
// pitest mutationThreshold in the pom fails the build below the bar
```

### 4.5 Test behavior, not internals
**Do:** Assert outcomes through the port using a hand-written fake.
**Don't:** Mock internals with a framework, or write a test that only proves a library works.

TypeScript:
```ts
// DON'T: asserts an internal call and effectively tests that the ORM was invoked
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

Java:
```java
// DON'T: Mockito verifies a call, asserting how the code works, not what it does
@Test void savesUser() {
  var repo = mock(UserRepo.class); new SignUp(repo).run(input);
  verify(repo).persist(any()); // breaks on any refactor, proves nothing observable
}
// DO: a hand-written fake, assert the effect through the port
@Test void signUpPersistsUser() {
  var store = new MemoryUserStore();
  new SignUp(store).run(new SignUpInput("a@b.co"));
  assertTrue(store.byEmail("a@b.co").isPresent());
}
```

TypeScript (React):
```tsx
// DON'T: assert on internal state and implementation; it breaks on any refactor and proves nothing a user sees
expect(wrapper.find(Counter).state('count')).toBe(1);
// DO: interact and assert on what the user sees (React Testing Library)
render(<Counter />);
await userEvent.click(screen.getByRole('button', { name: /increment/i }));
expect(screen.getByText('Count: 1')).toBeVisible();
```

### 4.6 Gate every merge
**Do:** Run the full suite in CI and block any merge on a red pipeline.
**Don't:** Let changes land when tests are failing, skipped, or never run on the branch.

TypeScript:
```ts
// DON'T: the workflow ignores test failures, so red merges anyway
// .github/workflows/ci.yml
// - run: bun test || true          # swallows the non-zero exit code
// - run: bun test
//   continue-on-error: true        # step warns but the job still passes
// DO: the suite must pass, a non-zero exit fails the job and blocks the merge
// .github/workflows/ci.yml
// - run: bun run lint && bun run typecheck && bun test
// branch protection requires this check green before merge is allowed
```

Java:
```java
// DON'T: Surefire configured to ignore failures, CI stays green over red tests
// pom.xml
// <configuration><testFailureIgnore>true</testFailureIgnore></configuration>
// mvn verify then "succeeds" while tests are failing
// DO: failures fail the build, the required check blocks the merge
// pom.xml keeps the default (testFailureIgnore = false)
// CI: mvn -B verify   // non-zero exit on any failing test, required for merge
```

### 4.7 Hold generated code to the same bar
**Do:** Run AI- or generator-produced code through the identical suite a human's would, and treat its output as a draft until the tests and a reviewer say otherwise.
**Don't:** Trust generated code on the strength of the tool that produced it, or merge it with the checks waived.

Stack-agnostic (provenance is not proof; the change proves itself through the gates):
```text
# DON'T: the change came from a generator, so it ships past the gates on trust
#   "the tool wrote it, it must be fine" -- provenance is not proof
#   git commit --no-verify; reviewer rubber-stamps "AI-generated, looks fine"
# DO: generated code is a draft until the suite and a human say otherwise
#   lint + tests + typecheck run on it like any other change;
#   the reviewer reads the diff, not the attribution
```

## Pillar 5: Secure by default

Security is a property of the default path, not a feature you bolt on later, so make the safe choice the only choice a developer can make without extra effort.

### 5.1 Keep secrets out of the codebase
**Do:** Read secrets at runtime from a managed secret store, keep keys under central control so no single person holds the master, and on any leak rotate the value and purge it from history.
**Don't:** Commit secrets to source, print them to logs, or paste them into a wiki.

TypeScript:
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

Java:
```java
// DON'T: hardcoded credential compiled into the artifact
String pwd = "P@ssw0rd-prod-2024";
Log.info("db password is " + pwd);
// DO: resolved from config backed by the secret manager, absent from logs
@ConfigProperty(name = "db.password") String pwd; // sourced via Secret Manager
void connect() {
    Log.info("opening db connection"); // no secret in the message
    dataSource.authenticate(pwd);
}
```

### 5.2 Do not build authentication or crypto yourself
**Do:** Delegate identity to an OIDC provider and put every admin console behind SSO plus MFA, using vetted libraries for any hashing or signing.
**Don't:** Roll a custom login, hand-roll token signing, or invent your own password hashing.

TypeScript:
```ts
// DON'T: homemade session token and a reversible "hash"
const token = btoa(user.id + ':' + Date.now());
const stored = user.password.split('').reverse().join('');
// DO: verify tokens minted by the identity provider, delegate the crypto
const claims = await verifyOidcToken(bearer, ISSUER); // Zitadel JWKS, standard JWT verify
if (!claims.ok) return c.json({ error: 'unauthorized' }, 401);
c.set('claims', claims.value); // MFA and SSO enforced upstream by the IdP
```

Java:
```java
// DON'T: custom "encryption" and self-checked passwords
String enc = new StringBuilder(secret).reverse().toString();
if (input.equals(user.plaintextPassword)) grantAccess();
// DO: provider-issued JWT verified by the framework, admin behind SSO + MFA
@Authenticated
@RolesAllowed("admin")
@GET @Path("/console")
public Dashboard console(@Context SecurityIdentity id) { // OIDC verifies the token
    return dashboard.forOperator(id.getPrincipal().getName());
}
```

TypeScript (React):
```tsx
// DON'T: a hand-rolled login that stores a JWT in localStorage, readable by any XSS on the page
const token = await api.post('/login', creds).then((r) => r.text());
localStorage.setItem('jwt', token);
// DO: use the identity provider's SDK; the session rides in an httpOnly, secure cookie the JS cannot read
const { login } = useAuth();   // OIDC/OAuth redirect handled by the provider
await login();                 // server responds Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax
```

### 5.3 Control your dependencies
**Do:** Pin every dependency to a fixed version, scan continuously for known vulnerabilities, and let automated updates keep you current.
**Don't:** Float on version ranges, or leave advisories unread until an incident.

TypeScript:
```ts
// DON'T: caret ranges resolve to unknown code on every install
// package.json
{ "dependencies": { "hono": "^4.0.0", "zod": "*" } }
// DO: exact pins, a committed lockfile, and a scanner in CI
// package.json  (bun install --frozen-lockfile in CI, Renovate opens update PRs)
{ "dependencies": { "hono": "4.6.14", "zod": "3.24.1" } }
// .github/workflows/ci.yml runs: bun audit  (fails the build on a known CVE)
```

Java:
```xml
<!-- DON'T: version ranges pull in whatever is newest and untested -->
<dependency><groupId>org.acme</groupId><artifactId>lib</artifactId>
  <version>[1.0,)</version></dependency>
<!-- DO: fixed version, the OWASP dependency-check plugin gates the build -->
<dependency><groupId>org.acme</groupId><artifactId>lib</artifactId>
  <version>1.4.2</version></dependency>
<!-- mvn org.owasp:dependency-check-maven:check fails on a CVSS >= 7; Renovate updates pins -->
```

### 5.4 Secure the supply chain
**Do:** Ship immutable, versioned, signed artifacts accompanied by a software bill of materials.
**Don't:** Deploy a mutable `latest` tag built from an unverifiable source.

TypeScript:
```ts
// DON'T: rebuildable floating tag, no provenance, no SBOM
// deploy.sh
docker build -t registry/app:latest . && docker push registry/app:latest
// DO: content-addressed digest, signed, with an SBOM attached
// release.sh
docker build -t registry/app:1.8.0 .
syft registry/app:1.8.0 -o spdx-json > sbom.json      // bill of materials
cosign sign --key env://COSIGN_KEY registry/app@sha256:9f2a...  // signature bound to digest
```

Java:
```xml
<!-- DON'T: SNAPSHOT artifact, mutable, unsigned, no inventory -->
<version>3.0-SNAPSHOT</version>
<!-- DO: released immutable version, GPG signed, SBOM generated at build -->
<version>3.0.1</version>
<!-- mvn deploy runs maven-gpg-plugin (signs the jar)
     and cyclonedx-maven-plugin (emits target/bom.json), verified on pull -->
```

### 5.5 Validate at the boundary, authorize on the server
**Do:** Pass every untrusted value through one validating checkpoint before it reaches a query, command, or file path, and check permissions on the server.
**Don't:** Trust input shape, or rely on the UI hiding a button as your access control.

TypeScript:
```ts
// DON'T: raw body straight into a query; "admin" trusted from the client
app.post('/users', (c) => db.insert(users).values(c.req.json()));
// DO: validate at the edge, authorize server-side from the verified token
app.post('/users', zValidator('json', NewUser), async (c) => {
  if (c.get('claims').role !== 'admin') return c.json({ error: 'forbidden' }, 403);
  return c.json(await createUser(c.req.valid('json')));
});
```

Java:
```java
// DON'T: unvalidated DTO and no server-side authz
@POST public User create(UserDto dto) { return repo.save(dto); }
// DO: Bean Validation at the edge, @RolesAllowed enforced on the server
@POST @RolesAllowed("admin")
public User create(@Valid NewUser in) { return service.create(in); }
```

TypeScript (React):
```tsx
// DON'T: the only thing stopping the action is a hidden button; the endpoint trusts the client, and a 403 is unhandled
{isAdmin && <button onClick={() => api.post('/admin/purge')}>Delete all</button>}
// DO: show it only for UX, and handle the server's 403 so a non-admin crafted request fails gracefully — the server is the real gate
{isAdmin && (
  <button onClick={async () => {
    const res = await api.post('/admin/purge');
    if (res.status === 403) setToast(t('errors.forbidden')); // authorized on the server; the UI just explains the refusal
  }}>Delete all</button>
)}
```

### 5.6 Expose only what has to be public
**Do:** Keep databases, caches, queues, and admin panels on a private network reachable only from your own services.
**Don't:** Give a datastore a public endpoint and rely on a password as the only barrier.

Terraform / OpenTofu:
```hcl
# DON'T: database open to the whole internet, one credential away from breach
resource "managed_database" "db" {
  name             = "app-db"
  is_ha_cluster    = true
  # private_network omitted => public endpoint reachable from 0.0.0.0/0
}

# DO: attach the datastore to a private network, no public endpoint
resource "managed_database" "db" {
  name          = "app-db"
  is_ha_cluster = true
  private_network {
    pn_id = private_network.main.id  # only in-VPC services can dial in
  }
  # no public_endpoint block => not routable from the internet
}
```

Security-group rule (the same idea at the firewall):
```hcl
# DON'T: Postgres port open to the world
ingress {
  from_port = 5432
  to_port   = 5432
  protocol  = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
# DO: only the app subnet may reach the database port
ingress {
  from_port   = 5432
  to_port     = 5432
  protocol    = "tcp"
  cidr_blocks = ["10.0.1.0/24"]
}
```

### 5.7 One security baseline everywhere
**Do:** Make authenticated access, TLS, rate limits, and allow/deny lists the default on every route and environment.
**Don't:** Leave endpoints open by default and add protection route by route from memory.

TypeScript:
```ts
// DON'T: protection opt-in per route, plaintext, no throttling
app.get('/reports', (c) => c.json(getReports())); // anyone, any rate, over http
// DO: baseline middleware applied to the whole app, TLS terminated upstream
app.use('*', requireAuth);                         // authenticated by default
app.use('*', arcjet.rateLimit({ max: 100, window: '1m' })); // throttle + deny lists
app.use('*', hsts());                              // enforce https
app.get('/reports', (c) => c.json(getReports()));  // inherits every guard
```

Java:
```java
// DON'T: endpoint permits all by default, per-route hardening forgotten
@GET @Path("/reports") @PermitAll
public List<Report> reports() { return service.all(); }
// DO: deny-by-default policy plus a global rate-limit and TLS filter
// application.properties: quarkus.http.auth.permission.default.policy=authenticated
//                         quarkus.http.ssl.protocols=TLSv1.3
@GET @Path("/reports")                 // inherits authenticated-by-default policy
@RateLimited(value = 100, window = 1, unit = MINUTES)
public List<Report> reports() { return service.all(); }
```

## Pillar 6: Private by default

Personal data is a liability you choose to hold, so collect the least, guard it by default, and make the user's rights ordinary operations rather than special projects.

### 6.1 Know the law that follows the user
**Do:** Identify every privacy regime your users fall under and design to the strictest one you serve.
**Don't:** Assume your home jurisdiction's rules apply to everyone.

TypeScript:
```ts
// DON'T: one hardcoded local assumption, ignores where the user actually is
function needsConsent() { return false; } // "we're fine, we're a small shop"
// DO: resolve the governing regime and apply the strictest requirements
function policyFor(user: User): PrivacyPolicy {
  const regimes = regimesFor(user.residency, user.dataLocations); // GDPR, PIPL, ...
  return strictest(regimes); // e.g. explicit opt-in consent, data-export rights
}
const policy = policyFor(user);
if (policy.requiresExplicitConsent && !user.consentGiven) blockProcessing();
```

Java:
```java
// DON'T: consent logic branches on a single assumed country
if (user.country.equals("US")) proceedWithoutConsent();
// DO: derive the applicable regimes and enforce the strictest baseline
PrivacyPolicy policy = privacyPolicies.strictestFor(
    user.residency(), user.dataLocations()); // union of GDPR, PIPL, and others
if (policy.requiresExplicitConsent() && !user.hasConsented()) {
    throw new ConsentRequiredException();
}
```

### 6.2 Minimize and justify collection
**Do:** Collect only the fields a stated purpose needs, and require explicit consent for sensitive data or minors.
**Don't:** Grab everything the form could offer in case it is useful later.

TypeScript:
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

Java:
```java
// DON'T: collect sensitive attributes speculatively, unjustified
public record Signup(@Email String email, String nationalId,
                     String ethnicity, LocalDate birthDate) {}
// DO: minimal fields; sensitive data requires an explicit, purpose-bound consent
public record Signup(@Email String email, @NotBlank String displayName) {}
// A health-related field would live behind @ValidConsent(purpose = MEDICAL, minors = false)
```

### 6.3 Keep personal data out of logs and URLs
**Do:** Send personal data in a POST body and redact it at the logger before anything is written.
**Don't:** Put identifiers or secrets in a GET query string, or log raw request bodies.

TypeScript:
```ts
// DON'T: PII in the query string (lands in access logs) and a raw-body log line
await fetch(`/search?email=${user.email}&token=${session}`);
logger.info('request', { body: req.body }); // writes personal data to disk
// DO: PII in the POST body, logger redacts known-sensitive keys
await fetch('/search', { method: 'POST', body: JSON.stringify({ email: user.email }) });
logger.info('request handled', { userId: hash(user.id) }); // redactor drops email, token, ssn
```

Java:
```java
// DON'T: email in the URL and the whole payload dumped to the log
@GET @Path("/search")
public Result search(@QueryParam("email") String email) {
    Log.info("searching payload: " + email); // PII in access log and app log
    return svc.search(email);
}
// DO: PII in the body, structured logging with the sensitive field masked
@POST @Path("/search")
public Result search(@Valid SearchRequest req) {
    Log.infov("search handled for user {0}", req.userRef()); // pseudonymous ref only
    return svc.search(req.email()); // MDC filter redacts email/token keys
}
```

TypeScript (React):
```tsx
// DON'T: an email in the query string lands in browser history, referrer headers, and analytics
navigate(`/search?email=${encodeURIComponent(email)}`);
// DO: keep personal data out of the URL; pass it in state or a POST body, and show only an opaque id in the path
navigate('/search', { state: { email } }); // or POST the search; the address bar shows /search only
```

### 6.4 Build for user rights from day one
**Do:** Ship see, correct, export, delete, and withdraw-consent as routine, first-class operations.
**Don't:** Treat a deletion or export request as a manual database ticket.

TypeScript:
```ts
// DON'T: rights are ad hoc, no export, delete means "we'll run some SQL"
// (no endpoint exists; requests handled by hand in a console)
// DO: rights are ordinary endpoints, audited like any other write
app.get('/me/data', requireAuth, (c) => c.json(exportSubject(c.get('userId'))));   // see + export
app.patch('/me', requireAuth, zValidator('json', Correction), correctSubject);      // correct
app.delete('/me', requireAuth, (c) => eraseSubject(c.get('userId')));               // delete
app.post('/me/consent/withdraw', requireAuth, withdrawConsent);                     // withdraw
```

Java:
```java
// DON'T: no self-service rights, everything routed through support tickets
// (only internal admin tooling exists)
// DO: each right is a first-class, authenticated operation
@GET  @Path("/me/data") @Authenticated
public SubjectExport export(@Context SecurityIdentity id) { return svc.export(id); }   // see + export
@DELETE @Path("/me") @Authenticated
public void erase(@Context SecurityIdentity id) { svc.erase(id); }                     // delete
@POST @Path("/me/consent/withdraw") @Authenticated
public void withdraw(@Context SecurityIdentity id) { svc.withdrawConsent(id); }        // withdraw
```

### 6.5 Map and classify your data
**Do:** Maintain an inventory of what personal data you hold, why, where it flows, whether it crosses a border, and tag every field by sensitivity.
**Don't:** Store fields with no owner, purpose, or sensitivity label.

TypeScript:
```ts
// DON'T: untyped bag of columns, no purpose, no classification
type Customer = { a: string; b: string; c: string };
// DO: each field carries a sensitivity class and purpose the inventory can read
const CustomerSchema = {
  email:   { class: 'pii',       purpose: 'account',  crossesBorder: false },
  taxId:   { class: 'sensitive', purpose: 'invoicing', crossesBorder: false },
  country: { class: 'pii',       purpose: 'tax-rules', crossesBorder: true  },
} as const; // generates the data map and drives retention + residency checks
```

Java:
```java
// DON'T: plain entity, nothing records what the data is or why it is held
@Entity public class Customer { public String email; public String taxId; }
// DO: annotate classification and purpose so tooling can build the data map
@Entity public class Customer {
    @DataClass(Sensitivity.PII)       @Purpose("account")   public String email;
    @DataClass(Sensitivity.SENSITIVE) @Purpose("invoicing") public String taxId;
    @DataClass(Sensitivity.PII) @Purpose("tax") @CrossBorder public String country;
}
```

### 6.6 Never copy production data into test or dev
**Do:** Generate synthetic fixtures that mimic shape and volume without any real person's data.
**Don't:** Clone a production dump into a lower environment.

TypeScript:
```ts
// DON'T: real customers restored into the dev database
// seed.ts
const rows = await pgRestore('prod_backup.sql'); // live PII now in dev + backups
await db.insert(customers).values(rows);
// DO: deterministic synthetic data, correct shape, zero real subjects
// seed.ts
const rows = Array.from({ length: 500 }, (_, i) => ({
  email: `user${i}@example.test`, taxId: fakeTaxId(i), country: pickCountry(i),
})); // faker-style, seeded for reproducibility
await db.insert(customers).values(rows);
```

Java:
```java
// DON'T: import a production export for local testing
@BeforeEach void seed() { db.execute(readFile("prod_dump.sql")); } // real PII in tests
// DO: build fixtures programmatically, no real subject ever touched
@BeforeEach void seed() {
    IntStream.range(0, 500).forEach(i -> customerRepo.persist(
        new Customer("user" + i + "@example.test", Fixtures.taxId(i), Fixtures.country(i))));
}
```

### 6.7 Assess before risky processing
**Do:** Record a short impact assessment before automated decisions, sensitive-data processing, or cross-border transfers.
**Don't:** Ship high-risk processing and reason about the risk afterward.

TypeScript:
```ts
// DON'T: automated decision goes live with no recorded assessment
function autoRejectApplicant(a: Application) { return score(a) < 0.4; } // no DPIA
// DO: gate the risky operation on a filed, approved assessment
async function autoDecide(a: Application): Promise<Result<Decision, 'dpia_missing'>> {
  const dpia = await assessments.find('auto-credit-decision');
  if (!dpia.ok || dpia.value.status !== 'approved') return { ok: false, error: 'dpia_missing' };
  return { ok: true, value: score(a) < 0.4 ? 'reject' : 'review' };
}
```

Java:
```java
// DON'T: cross-border transfer runs with no impact assessment on record
public void exportToRegion(Dataset d, Region r) { transfer.send(d, r); } // unassessed
// DO: require an approved assessment before the risky processing proceeds
public void exportToRegion(Dataset d, Region r) {
    ImpactAssessment dpia = assessments.forTransfer(d.classification(), r)
        .orElseThrow(() -> new AssessmentRequiredException("cross-border DPIA missing"));
    if (dpia.approved()) transfer.send(d, r); // else the transfer never happens
}
```

## Pillar 7: Isolate by default

In a multi-tenant system every query is a chance to leak, so derive the owner from something the caller cannot forge and enforce that boundary at every layer.

### 7.1 Derive the tenant/owner from one trusted source
**Do:** Take the tenant or owner from a verified token claim.
**Don't:** Read the tenant from a URL segment, header, or body field the caller controls.

TypeScript:
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

Java:
```java
// DON'T: trust a client-set header to choose the tenant
@GET public List<Invoice> list(@HeaderParam("X-Org-Id") String orgId) {
    return repo.byOrg(orgId); // attacker sets any org id
}
// DO: read the tenant from the verified security identity
@GET @Authenticated
public List<Invoice> list(@Context SecurityIdentity id) {
    String orgId = id.getAttribute("org_id"); // from the validated token claim
    return repo.byOrg(orgId);
}
```

### 7.2 Defend in depth
**Do:** Enforce the owner boundary in application code and again in the data store with row-level security.
**Don't:** Rely on a single WHERE clause as the only thing standing between tenants.

TypeScript:
```ts
// DON'T: one app-level filter; a forgotten clause anywhere leaks across tenants
const rows = await db.select().from(invoices).where(eq(invoices.orgId, orgId));
// DO: filter in code AND set the tenant in a session var that RLS enforces, inside one transaction
const rows = await db.transaction(async (tx) => {
  await tx.execute(sql`SELECT set_config('app.current_org', ${orgId}, true)`); // true => tx-local, read by the RLS policy
  return tx.select().from(invoices).where(eq(invoices.orgId, orgId));
});
// migration: CREATE POLICY tenant_isolation ON invoices
//   USING (org_id = current_setting('app.current_org')::uuid);
```

Java:
```java
// DON'T: application filter only, database would happily return every tenant
return em.createQuery("select i from Invoice i where i.orgId = :o", Invoice.class)
         .setParameter("o", orgId).getResultList();
// DO: app filter plus a database RLS policy scoped by a session setting
em.createNativeQuery("SELECT set_config('app.current_org', :o, true)").setParameter("o", orgId).getSingleResult(); // tx-scoped: same @Transactional block as the read below
return Invoice.list("orgId", orgId); // RLS policy on invoices re-checks org_id
// V5__rls.sql: CREATE POLICY tenant_isolation ON invoices
//   USING (org_id = current_setting('app.current_org')::uuid);
```

### 7.3 Fail closed
**Do:** Return zero rows when tenant context is missing.
**Don't:** Return everything, or throw a 500, when the tenant is absent.

TypeScript:
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

Java:
```java
// DON'T: null tenant returns the full table
public List<Invoice> forOrg(String orgId) {
    if (orgId == null) return Invoice.listAll(); // leaks every tenant
    return Invoice.list("orgId", orgId);
}
// DO: absent tenant yields an empty list, not the estate and not an error
public List<Invoice> forOrg(String orgId) {
    if (orgId == null || orgId.isBlank()) return List.of(); // fail closed
    return Invoice.list("orgId", orgId);
}
```

### 7.4 Shrink the blast radius
**Do:** Give the runtime role the narrowest privileges it needs, so one leaked credential exposes one tenant, not the whole estate.
**Don't:** Run the application as a superuser that can read and drop everything.

SQL / migration (grants the runtime role actually gets):
```sql
-- DON'T: the app connects as an all-powerful role
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_runtime; -- can read + drop
ALTER ROLE app_runtime WITH SUPERUSER BYPASSRLS;                    -- ignores tenant policy
-- DO: least privilege, and crucially the runtime role cannot bypass RLS
CREATE ROLE app_runtime NOSUPERUSER NOBYPASSRLS;                    -- RLS always applies
GRANT SELECT, INSERT, UPDATE ON invoices, receipts TO app_runtime; -- no DELETE, no DDL
-- migrations run as a separate migrator role, only in CI, never at runtime
```

TypeScript (the connection the app opens):
```ts
// DON'T: runtime uses the owner/superuser connection string
const db = drizzle(postgres(process.env.SUPERUSER_URL!)); // full rights on every table
// DO: runtime uses the constrained, RLS-bound role
const db = drizzle(postgres(process.env.APP_RUNTIME_URL!)); // NOBYPASSRLS, scoped grants
```

### 7.5 Prove isolation per endpoint
**Do:** Add a test per endpoint where org A's token against org B's resource returns not_found.
**Don't:** Assume the shared filter holds without an explicit cross-tenant test.

TypeScript:
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

Java:
```java
// DON'T: test only fetches the caller's own record
@Test void ownerReadsOwnInvoice() {
    given().auth().oauth2(tokenOrgA)
        .when().get("/invoices/" + ownInvoice).then().statusCode(200);
}
// DO: org A hitting org B's invoice must return not_found
@Test void crossTenantReadIsNotFound() {
    given().auth().oauth2(tokenOrgA)
        .when().get("/invoices/" + orgBInvoice)
        .then().statusCode(404); // hides existence, no cross-tenant leak
}
```


## Pillar 8: Delivery should be boring

Deployment is a non-event when the pipeline is automated, changes are small and additive, and every environment is reproducible from files.

### 8.1 Trunk-based development with small commits
**Do:** Merge small, reviewable commits into a short-lived branch off main and integrate daily.
**Don't:** Sit on a long-lived feature branch for weeks so integration becomes one giant, unreviewable merge.

Applies to any stack (git workflow):
```bash
# DON'T: a long-lived branch that drifts for weeks, then a single massive PR
git checkout -b feature/big-rewrite
# ... 3 weeks, 87 files, 6000 lines later ...
git merge main            # days of conflict resolution, nobody can review this
# DO: short-lived branch, one focused change, rebased and merged the same day
git checkout -b add-receipt-total-field
git commit -m "Add total_cents column to receipts (expand step)"
git rebase main           # stay current with trunk continuously
git push --set-upstream origin add-receipt-total-field
# open a small PR, review in minutes, merge, delete the branch same day
```

### 8.2 Automated pipeline, progressive delivery, one-step rollback
**Do:** Deploy only through the pipeline, roll out to a canary before full traffic, and keep rollback to a single action.
**Don't:** Deploy by hand and treat a bad release as an emergency you fix live.

Applies to any stack (GitHub Actions CI/CD):
```yaml
# DON'T: a "deploy" that is a human running a script against prod with no gate or rollback
# (there is no artifact for this: it is someone SSHing in and hoping)

# DO: automated build, canary weight, promote on healthy metrics, one-command rollback
name: deploy
on:
  push:
    branches: [main]
jobs:
  ship:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: bun install --frozen-lockfile
      - run: bun test && bun run typecheck && bun run lint:strict
      - name: Deploy canary (10% traffic)
        run: ./scripts/deploy --canary 10 --image "$IMAGE"
      - name: Watch canary SLO for 5 minutes
        run: ./ci/watch-error-rate.sh --max 0.01 --window 5m   # aborts job if breached
      - name: Promote to 100%
        run: ./scripts/deploy --promote 100 --image "$IMAGE"
  # rollback is one step: re-run the previous green deploy, or:
  #   ./scripts/deploy --promote 100 --image "$LAST_GOOD_IMAGE"
```

### 8.3 Infrastructure as code
**Do:** Define every resource in version-controlled files and apply it with one command.
**Don't:** Click a resource into existence in a cloud console where its config lives only in someone's memory.

Applies to any stack (OpenTofu/Terraform):
```hcl
# DON'T: "created by hand" in the console, no file, no history, no review
# Bucket "receipts" was clicked into existence by an engineer in the web UI on a Tuesday.
# Nobody can recreate it, diff it, or know why versioning is off. This is the anti-pattern.

# DO: the resource is a file, so it is reviewable, reproducible, and destroyable
resource "object_bucket" "receipts" {
  name = "receipts"
  versioning {
    enabled = true
  }
  lifecycle_rule {
    id      = "expire-tmp"
    prefix  = "tmp/"
    enabled = true
    expiration { days = 7 }
  }
}
# tofu plan && tofu apply  -> rebuilds the whole thing from files, every time
```

### 8.4 Vertical slices
**Do:** Let each feature own its handler, its data access, and its slice of infra so it deploys independently.
**Don't:** Funnel every feature through one shared "service" or "DAO" layer that everyone edits and blocks on.

TypeScript:
```ts
// DON'T: one god module every feature must edit, so all deploys contend on it
// packages/core/megaService.ts
export class MegaService {
  createReceipt() { /* ... */ }
  approveExpense() { /* ... */ }
  exportReport() { /* ... */ } // three teams fighting over one file and one deploy
}
// DO: a self-contained slice owns its route, use-case, and repository
// features/receipts/createReceipt.ts
export function createReceipt(deps: ReceiptDeps) {
  return async (input: NewReceipt): Promise<Result<Receipt, ReceiptError>> => {
    const saved = await deps.receipts.insert(input); // slice owns its data access
    return { ok: true, value: saved };
  };
}
```

Java:
```java
// DON'T: a shared EJB-style facade all features route through, one bottleneck deploy
@ApplicationScoped
public class MegaService {
  public Receipt createReceipt() { /* ... */ return null; }
  public void approveExpense() { /* ... */ }   // every team edits this class
  public byte[] exportReport() { /* ... */ return null; }
}
// DO: a vertical slice with its own resource, use-case, and repository
@Path("/v1/receipts")
public class CreateReceiptResource {
  @Inject ReceiptRepository receipts;           // slice owns its persistence
  @POST
  public Result<Receipt> create(NewReceipt input) {
    return receipts.persistReceipt(input);      // deployable on its own
  }
}
```

### 8.5 Change contracts additively / expand-contract
**Do:** Add the new column or field, backfill and migrate readers, then remove the old one in a later release.
**Don't:** Rename or drop a shipped column or response field in place and break live clients.

TypeScript:
```ts
// DON'T: destructive Drizzle migration that renames a live column in one shot
// old readers and in-flight requests break the instant this runs
export async function up(db: DB) {
  await db.execute(sql`ALTER TABLE receipts RENAME COLUMN amount TO total_cents`);
}
// DO: expand (add nullable new column), migrate, then contract in a separate release
// migration 0007_expand.ts
export async function up(db: DB) {
  await db.execute(sql`ALTER TABLE receipts ADD COLUMN total_cents integer`);
  await db.execute(sql`UPDATE receipts SET total_cents = amount WHERE total_cents IS NULL`);
} // deploy code that dual-writes both columns, then a later 0009_contract drops "amount"
```

Java (Flyway migration):
```sql
-- DON'T (Flyway V7__break.sql): drop the column an old app version still selects
ALTER TABLE receipts DROP COLUMN amount;

-- DO (Flyway V7__expand.sql): add the new column, backfill, keep old one for now
ALTER TABLE receipts ADD COLUMN total_cents integer;
UPDATE receipts SET total_cents = amount WHERE total_cents IS NULL;
-- ship an app version that writes both columns, verify, then V9__contract.sql drops "amount"
```

### 8.6 Separate and ephemeral environments
**Do:** Give each branch or load test its own isolated environment and destroy it when the work is done.
**Don't:** Share one long-lived "staging" box where everyone's half-finished work collides.

Applies to any stack (GitHub Actions + IaC, per-branch preview):
```yaml
# DON'T: everyone deploys to a single shared staging that is always half-broken
# (one mutable box, no isolation, no teardown: the anti-pattern)

# DO: spin up a throwaway env keyed to the branch, tear it down on PR close
name: preview-env
on:
  pull_request:
    types: [opened, synchronize, closed]
jobs:
  up:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tofu workspace select "pr-${{ github.event.number }}" || tofu workspace new "pr-${{ github.event.number }}"
      - run: tofu apply -auto-approve -var "env=pr-${{ github.event.number }}"   # isolated db + container
  down:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: tofu workspace select "pr-${{ github.event.number }}"
      - run: tofu destroy -auto-approve -var "env=pr-${{ github.event.number }}"  # nothing lingers, nothing to pay for
```

## Pillar 9: Run as little as possible yourself

Every server you patch, cert you renew by hand, and console you click in is undifferentiated toil; hand it to a managed platform and to the pipeline.

### 9.1 Prefer managed over self-run
**Do:** Use a managed database, queue, or object store so patching, backups, and failover are the provider's job.
**Don't:** Run Postgres on a VM you own, then own its kernel updates, disk growth, and 3 a.m. failover.

Applies to any stack (OpenTofu/Terraform):
```hcl
# DON'T: a raw VM you now have to patch, back up, monitor, and fail over yourself
resource "compute_instance" "db_vm" {
  type  = "DEV1-M"
  image = "ubuntu_jammy"
  # ... then someone hand-installs postgres, and it is now a pet you feed forever
}

# DO: a managed database; the provider handles patching, backups, and HA
resource "managed_database" "main" {
  name          = "app-prod"
  engine        = "PostgreSQL-15"
  node_type     = "DB-GP-S"
  is_ha_cluster = true
  volume_type   = "bssd"
  volume_size_in_gb = 20
  # automated backups, minor-version patching, and failover are the provider's problem
}
```

### 9.2 No servers you SSH into
**Do:** Ship immutable, replaceable container units and redeploy to change anything.
**Don't:** Keep an SSH key and a bastion so you can log in and mutate a running box.

Applies to any stack (Dockerfile + commentary):
```dockerfile
# DON'T: a box you SSH into to "just fix it live"
#   ssh -i prod.pem ubuntu@bastion   ->  edit files in place  ->  undocumented drift
#   the fix survives until the next reboot, then vanishes. No SSH keys should exist.

# DO: an immutable image. To change the app you build a new image and redeploy, never log in.
FROM oven/bun:1-slim
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile --production
COPY . .
USER bun                       # non-root, nothing to log into
CMD ["bun", "run", "start"]
# the running container has no shell access granted and no persistent state; kill and replace freely
```

### 9.3 Automatic TLS certificates
**Do:** Let the platform issue and renew certificates automatically for every hostname.
**Don't:** Buy a cert, install it by hand, and put its expiry on someone's calendar.

Applies to any stack (OpenTofu/Terraform):
```hcl
# DON'T: a manual cert that expires at 2 a.m. on a holiday because the calendar reminder was missed
# Engineer pastes a .pem into a load balancer once a year. Miss the date -> full outage. Anti-pattern.

# DO: managed certificate, issued and renewed by the platform, no human in the loop
resource "lb_certificate" "app" {
  lb_id = load_balancer.main.id
  name  = "app-tls"
  letsencrypt {
    common_name = "app.example.com"          # platform requests and auto-renews via ACME
  }
}
# renewal happens on its own; nobody owns an expiry date on a calendar
```

### 9.4 Only the pipeline touches infrastructure
**Do:** Give humans read-only prod access and make every infra change flow through a merge.
**Don't:** Hand engineers admin keys so they can mutate prod directly whenever they like.

Applies to any stack (branch protection as code + read-only human IAM):
```hcl
# DON'T: developers hold AdministratorAccess and change prod from their laptops
# (broad standing write credentials on humans: the thing to remove)

# DO: humans are read-only; only the CI identity can apply infra, and only via a reviewed merge
resource "github_branch_protection" "main" {
  repository_id = github_repository.app.node_id
  pattern       = "main"
  required_pull_request_reviews {
    required_approving_review_count = 1
  }
  required_status_checks {
    strict   = true
    contexts = ["ci/plan", "ci/typecheck"]
  }
  enforce_admins = true             # even admins cannot push straight to main
}
# the deploy role (write access to infra) is assumed only by the pipeline, never by a person
```

## Pillar 10: Design for failure

Assume disks die, dependencies time out, and requests crash mid-flight; make failures explicit, recoverable, and something you learn from.

### 10.1 Set explicit reliability targets
**Do:** Write down an SLO with a concrete uptime goal and error-rate ceiling, and alert against it.
**Don't:** Rely on "it feels reliable enough" with no number anyone can check.

Applies to any stack (SLO as code, e.g. an alerting rule):
```yaml
# DON'T: reliability is a vibe. "It's usually fine." Nothing measures it, nothing pages anyone.

# DO: an explicit objective with a burn-rate alert (99.9% availability => 0.1% error budget)
slo:
  service: app-api
  objective: 0.999                 # 99.9% of requests succeed over 30 days
  error_budget: 0.001
alerting:
  - name: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
    for: 5m                        # sustained breach of the 1% ceiling pages on-call
```

### 10.2 Errors as values, not exceptions
**Do:** Return a typed success-or-failure value from anything that can fail; reserve exceptions for genuine bugs.
**Don't:** Throw for an expected business outcome and hope a caller catches it.

TypeScript:
```ts
// DON'T: an expected "insufficient funds" thrown as an exception, easy to forget to catch
function withdraw(a: Account, n: number): Account { if (n > a.balance) throw new Error('nsf'); return a; }
// DO: the failure is in the return type, so the caller must handle it
function withdraw(a: Account, n: number): Result<Account, 'insufficient_funds'> {
  if (n > a.balance) return { ok: false, error: 'insufficient_funds' };
  return { ok: true, value: { ...a, balance: a.balance - n } };
}
```

Java:
```java
// DON'T: expected failure as an unchecked exception
Account withdraw(Account a, long n) { if (n > a.balance()) throw new IllegalStateException("nsf"); return a; }
// DO: sealed Result makes the failure part of the contract (Vavr Either is an alternative)
Result<Account> withdraw(Account a, long n) {
  if (n > a.balance()) return new Err<>("insufficient_funds");
  return new Ok<>(a.debit(n));
}
```

TypeScript (React):
```tsx
// DON'T: the call throws and blanks the screen, or the rejection is silently swallowed
const orders = await gateway.list(); // throws on failure, so render crashes
return <OrderList orders={orders} />;
// DO: the gateway returns a Result; the component renders an explicit error state
const res = await gateway.list();    // Result<Order[], LoadError>
if (!res.ok) return <ErrorState kind={res.error} onRetry={reload} />;
return <OrderList orders={res.value} />;
```

### 10.3 Keep read paths explicit
**Do:** Write reads as hand-authored SQL you can see and tune, and let the ORM own writes.
**Don't:** Let an ORM generate opaque queries on your hot read path — a lazy N+1 or a hidden relation cascade — that you cannot see or tune.

TypeScript:
```ts
// DON'T: an ORM relation walk fires a cascade of hidden queries across relation levels — you cannot EXPLAIN or tune them, and they balloon on a list path
const orgs = await db.query.orgs.findMany({ with: { receipts: { with: { lines: true } } } });
// DO: explicit SQL for the read (see it, EXPLAIN it, tune it); ORM stays for the write
const rows = await db.execute(sql`
  SELECT r.id, r.total_cents, count(l.id) AS line_count
  FROM receipts r LEFT JOIN receipt_lines l ON l.receipt_id = r.id
  WHERE r.org_id = ${orgId}
  GROUP BY r.id ORDER BY r.created_at DESC LIMIT 50`);
await db.insert(receipts).values(newReceipt);   // writes go through the ORM
```

Java:
```java
// DON'T: Panache lazy navigation triggering N+1 on a list read
List<Receipt> all = Receipt.listAll();
all.forEach(r -> r.lines.size());   // one extra SELECT per receipt, invisible until it is slow
// DO: an explicit projection query you can read and index; ORM persists the write
List<ReceiptSummary> summaries = getEntityManager().createNativeQuery("""
    SELECT r.id, r.total_cents, count(l.id) AS line_count
    FROM receipts r LEFT JOIN receipt_lines l ON l.receipt_id = r.id
    WHERE r.org_id = :org
    GROUP BY r.id ORDER BY r.created_at DESC LIMIT 50""", ReceiptSummary.class)
  .setParameter("org", orgId).getResultList();
newReceipt.persist();   // write path stays with the ORM
```

### 10.4 Keep reads fast as the table grows
**Do:** Page with a stable cursor (keyset) and stream or bulk-load large result sets, so a query that is fast at ten thousand rows stays fast at ten million.
**Don't:** Page with OFFSET, which scans and discards ever more rows the deeper you page.

TypeScript:
```ts
// DON'T: OFFSET paging re-reads from row zero on every page; page 1000 does 1000x the work
async function list(page: number) {
  return db.select().from(receipts).limit(50).offset(page * 50);
}
// DO: keyset page on an indexed cursor; each page costs the same, no matter how deep
//   page on a unique cursor (e.g. createdAt, id) so tied rows neither skip nor duplicate
async function listAfter(afterCreatedAt?: string) {
  return db.select().from(receipts)
    .where(afterCreatedAt ? gt(receipts.createdAt, new Date(afterCreatedAt)) : sql`true`)
    .orderBy(asc(receipts.createdAt))
    .limit(50);
  // the last row's createdAt becomes the next cursor; never scans from zero
}
// for large exports, stream or cursor through the driver instead of loading every row into memory
```

Java:
```java
// DON'T: OFFSET paging re-reads and discards the rows before it; page 1000 scans 50,000 rows
List<Receipt> list(int page) {
    return find("order by createdAt").page(Page.of(page, 50)).list();
}
// DO: keyset page on an indexed cursor; each page costs the same regardless of depth
//   use a unique cursor (createdAt, id) so ties neither skip nor duplicate
List<Receipt> listAfter(Instant cursor) {
    return find("createdAt > ?1 order by createdAt", cursor).page(Page.of(0, 50)).list();
    // the last row's createdAt becomes the next cursor; depth no longer costs rows
}
// stream large exports instead of holding the whole table in the heap
try (var stream = find("order by createdAt").stream()) {
    stream.forEach(this::process);   // one row at a time, constant memory
}
```

### 10.5 Do not fire and forget
**Do:** Record the side-effect intent in the same transaction as the state change and deliver it with a retrying worker (transactional outbox).
**Don't:** Make a best-effort external call inside the request that silently vanishes if the process crashes.

TypeScript:
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

Java:
```java
// DON'T: best-effort send inside the transaction, lost on any crash after commit
receipt.persist();
mailer.send(receipt.owner(), "receipt_ready");   // no record it was ever attempted
// DO: enqueue an outbox row in the same transaction; a scheduled worker delivers with retries
@Transactional
public void createReceipt(Receipt receipt) {
  receipt.persist();
  new OutboxEntry("receipt_ready", receipt.id).persist();   // committed atomically with the receipt
}
// @Scheduled worker polls unsent OutboxEntry rows, sends, marks sent, retries on failure
```

### 10.6 Keep backups you have actually restored
**Do:** Run a scheduled drill that restores the backup into a scratch database and times it.
**Don't:** Trust a backup you have never restored and assume it works.

Applies to any stack (scheduled restore drill in CI):
```yaml
# DON'T: nightly dumps pile up in a bucket, never once restored. On disaster day you learn they were empty.

# DO: a scheduled job restores the latest backup into a throwaway DB and asserts it is intact
name: restore-drill
on:
  schedule:
    - cron: "0 3 * * 1"          # every Monday at 03:00
jobs:
  drill:
    runs-on: ubuntu-latest
    steps:
      - name: Restore latest backup into a scratch database
        run: |
          START=$(date +%s)
          pg_restore --clean --dbname "$SCRATCH_DB_URL" latest.dump
          echo "restore_seconds=$(( $(date +%s) - START ))"   # track RTO over time
      - name: Assert the data is really there
        run: psql "$SCRATCH_DB_URL" -c "SELECT count(*) FROM receipts" | grep -qv ' 0$'
      - name: Drop the scratch database
        run: psql "$ADMIN_URL" -c "DROP DATABASE scratch_restore"
```

### 10.7 Scale with demand
**Do:** Keep components stateless, autoscale on load, and cache deliberately with a clear invalidation rule.
**Don't:** Pin user state to a single instance so you can never add a second one.

TypeScript:
```ts
// DON'T: in-process session state, so requests must stick to one instance; horizontal scaling breaks
const sessions = new Map<string, User>();      // lost on restart, unshared across replicas
app.get('/me', (c) => c.json(sessions.get(c.req.header('sid')!)));
// DO: stateless handler, shared store, explicit cache with a TTL
app.get('/me', async (c) => {
  const sid = c.req.header('sid')!;
  const cached = await cache.get(`sess:${sid}`);      // shared Redis, any replica can serve
  if (cached) return c.json(JSON.parse(cached));
  const user = await store.loadSession(sid);
  await cache.set(`sess:${sid}`, JSON.stringify(user), { ex: 300 }); // deliberate 5-min TTL
  return c.json(user);
});
```

Java:
```java
// DON'T: @Singleton holding per-user state, so you can only ever run one instance
@Singleton
public class SessionCache {
  private final Map<String, User> sessions = new HashMap<>();  // dies on restart, not shared
}
// DO: stateless resource backed by a shared cache with an explicit expiry
@Path("/me")
public class MeResource {
  @Inject @Remote("sessions") RemoteCache<String, User> cache;  // shared, any node serves
  @Inject SessionStore store;
  @GET
  public User me(@HeaderParam("sid") String sid) {
    return cache.computeIfAbsent(sid, store::load, 5, TimeUnit.MINUTES);    // deliberate TTL
  }
}
```

### 10.8 Meet performance targets under load
**Do:** Set a p99 latency budget and prove it with a load test in the pipeline before shipping.
**Don't:** Ship the endpoint and find out about the p99 from angry users in production.

Applies to any stack (load test gate in CI, e.g. k6):
```yaml
# DON'T: no load test. "It was fast on my laptop with one request." Production p99 is a surprise.

# DO: a load test asserts the p99 budget and fails the build if the endpoint is too slow
name: load-test
on: [pull_request]
jobs:
  k6:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run k6 with a p99 threshold
        uses: grafana/k6-action@v0.3.1
        with:
          filename: load/receipts.js
          # receipts.js sets:  thresholds: { http_req_duration: ['p(99)<300'] }
          # 100 virtual users for 2 minutes; build fails if p99 exceeds 300ms
```

### 10.9 Treat data as sacred
**Do:** Soft-delete by stamping a deleted_at and keeping the row, route every schema change through a versioned migration, and choose storage whose durability and query shape fit the data.
**Don't:** Hard-delete a live row, or apply a destructive schema change by hand outside version control.

TypeScript:
```ts
// DON'T: a hard delete in the handler; the row and its history vanish forever, recovery is impossible
async function remove(id: OrderId) {
  await db.delete(orders).where(eq(orders.id, id));
}
// DO: soft-delete keeps the row; reads exclude it, a retention sweep decides later, recovery is a flag flip
async function remove(id: OrderId): Promise<Result<void, 'not_found'>> {
  const updated = await db.update(orders)
    .set({ deletedAt: new Date() })
    .where(and(eq(orders.id, id), isNull(orders.deletedAt)))
    .returning({ id: orders.id });
  return updated.length > 0 ? { ok: true, value: undefined } : { ok: false, error: 'not_found' };
}
// reads stay honest: db.select().from(orders).where(isNull(orders.deletedAt))
// every schema change ships as a numbered Drizzle migration (0008_soft_delete.ts), never a hand-run ALTER
```

Java:
```java
// DON'T: a real DELETE drops the row and every link to it, unrecoverable
@DELETE public void remove(UUID id) { orderRepo.deleteById(id); }
// DO: soft-delete keeps the row; recovery is a flag flip, a retention sweep decides later
@DELETE public Result<Void> remove(UUID id) {
    int n = orderRepo.update("deletedAt = ?1 where id = ?2 and deletedAt is null", Instant.now(), id);
    return n > 0 ? new Ok<>(null) : new Err<>("not_found");
}
// @SQLRestriction("deleted_at IS NULL") on the entity keeps reads honest by default
// every schema change is a versioned Flyway migration (V8__soft_delete.sql), never a hand ALTER
```

### 10.10 Learn from every failure
**Do:** Run a blameless postmortem after every incident that ends in concrete backlog items with owners.
**Don't:** Close the incident ticket the moment service is restored and move on.

Applies to any stack (postmortem template as a committed file):
```markdown
<!-- DON'T: incident resolved, ticket closed, root cause never written down, same outage recurs next quarter -->

<!-- DO: a committed, blameless postmortem that ends in tracked, owned action items -->
# Postmortem: 2026-07-04 receipt export outage
- Impact: exports failed for 22 min, ~140 orgs affected.
- Timeline: 14:02 deploy, 14:05 error rate breached SLO, 14:24 rolled back.
- Root cause: unbounded query on a missing index (no process failed, a gap did).
- Action items (each becomes a backlog ticket with an owner and due date):
  - [ ] Add index on receipt_lines(receipt_id)  (owner: A, due: 07-08)
  - [ ] Add p99 load-test gate for the export path (owner: B, due: 07-11)
  - [ ] Alert on query-plan regressions (owner: C, due: 07-18)
```


## Pillar 11: Make it observable

If you cannot see what a system is doing in production, you are guessing, and observability is what turns guesses into answers under one correlated trace.

### 11.1 Instrument everything with correlated traces
**Do:** Emit logs, metrics, and traces under one correlated trace id, using an open standard.
**Don't:** Scatter print statements with no way to follow one request end to end.

TypeScript:
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

Java:
```java
// DON'T: System.out, no trace context, nothing to correlate on
System.out.println("charging user " + userId);
// DO: OpenTelemetry span, auto-correlated by the runtime, attributes queryable later
Span span = tracer.spanBuilder("charge").startSpan();
try (Scope s = span.makeCurrent()) {
    span.setAttribute("user.id", userId);
    charge(userId);
} finally {
    span.end();
}
```

TypeScript (React):
```tsx
// DON'T: a render error blanks the page and only console.log exists, so no one sees it
console.log('render', data);
// DO: an error boundary reports to the telemetry backend, and web-vitals go out tagged with the session id
<ErrorBoundary onError={(e, info) => telemetry.captureException(e, { info, sessionId })}>
  <App />
</ErrorBoundary>;
onCLS((m) => telemetry.metric('web-vitals', m, { sessionId })); // plus LCP and INP, correlated to backend traces
```

### 11.2 Watch behavior, not just health
**Do:** Instrument product behavior (usage counts, latency, drop-off) so dashboards answer "are users succeeding".
**Don't:** Ship only CPU and memory graphs that stay green while the funnel quietly collapses.

TypeScript:
```ts
// DON'T: infra-only, tells you the box is alive, not whether checkout works
process.memoryUsage().heapUsed;
// DO: a domain counter and a histogram, split by outcome, so drop-off is visible
import { metrics } from '@opentelemetry/api';
const meter = metrics.getMeter('checkout');
const started = meter.createCounter('checkout.started');
const completed = meter.createCounter('checkout.completed');
const latency = meter.createHistogram('checkout.latency.ms');
export const recordCheckout = (ok: boolean, ms: number): void => {
  started.add(1);
  if (ok) completed.add(1);
  latency.record(ms, { outcome: ok ? 'ok' : 'abandoned' });
};
```

Java:
```java
// DON'T: only JVM heap gauges, blind to the user journey
Runtime.getRuntime().freeMemory();
// DO: Micrometer counters and a timer tagged by outcome, funnel becomes measurable
Counter.builder("checkout.started").register(registry).increment();
if (ok) Counter.builder("checkout.completed").register(registry).increment();
Timer.builder("checkout.latency")
     .tag("outcome", ok ? "ok" : "abandoned")
     .register(registry)
     .record(Duration.ofMillis(ms));
```

### 11.3 Alert on what matters
**Do:** Alert on error rate and p95/p99 latency crossing a user-visible budget, page only when a human must act, and retire any alert that pages twice without prompting a fix; after an incident slips through, fix what broke, not just the alert that missed it.
**Don't:** Wire noisy static-threshold alerts that fire nightly and train everyone to ignore the pager, or leave a useless alert standing because no one owns removing it.

TypeScript (stack-agnostic alert rule, Prometheus style):
```yaml
# DON'T: raw CPU threshold, no user meaning, fires on every batch job and gets muted
# - alert: HighCpu
#   expr: cpu_usage > 0.8
# DO: symptom-based, tied to a budget, waits long enough to exclude a blip
groups:
  - name: checkout-slo
    rules:
      - alert: CheckoutErrorBudgetBurn
        expr: |
          sum(rate(http_requests_total{route="/checkout",status=~"5.."}[5m]))
            / sum(rate(http_requests_total{route="/checkout"}[5m])) > 0.02
        for: 10m               # sustained, not a single spike
        labels: { severity: page }
        annotations:
          summary: "Checkout 5xx rate above 2% for 10m"
      - alert: CheckoutLatencyP99
        expr: histogram_quantile(0.99, sum(rate(checkout_latency_ms_bucket[5m])) by (le)) > 2000
        for: 10m
        labels: { severity: page }
```

## Pillar 12: No black boxes

A system nobody can explain is a liability, so make the essentials legible and keep the explanation attached to the code that it describes.

### 12.1 Document the essentials and treat doc drift as a defect
**Do:** Keep a README that actually installs and runs the project, and fail CI when it goes stale.
**Don't:** Let setup instructions rot until "works on my machine" is the only real onboarding path.

Stack-agnostic (README skeleton):
````md
<!-- DON'T: a vibes README, no commands, no way to tell if it still works -->
<!-- # MyService
     Some notes about the service. Ask Alice how to run it. -->

<!-- DO: exact, runnable, and testable so CI can prove it is current -->
# billing-service
Billing service API (TypeScript).

## Prerequisites
- Bun >= 1.1
- Docker (local Postgres + MinIO)

## Run locally
```bash
bun install
bun run dev:up        # backing services
bun run db:migrate
bun run dev           # http://localhost:3000
```

## Verify
```bash
curl -fsS http://localhost:3000/health   # -> {"status":"ok"}
```
````

Stack-agnostic (docs-check in CI, so drift fails the build):
```yaml
# .github/workflows/docs-check.yml
# DO: extract fenced bash blocks from the README and actually run them
name: docs-check
on: [pull_request]
jobs:
  readme-runs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun run docs:extract   # pulls ```bash blocks out of README.md
      - run: bash .ci/readme-steps.sh   # fails the PR if a documented command breaks
```

### 12.2 Generate API docs from the contract
**Do:** Derive the OpenAPI spec from the same schema that validates requests, with examples, and publish it.
**Don't:** Hand-maintain a separate API document that silently disagrees with the running code.

TypeScript:
```ts
// DON'T: prose in a wiki that no request ever touches, guaranteed to drift
// "POST /invoices takes an amount (probably a number) and returns an id."
// DO: one Zod schema validates AND emits the OpenAPI, examples included
import { createRoute, z } from '@hono/zod-openapi';
const Invoice = z.object({
  amountCents: z.number().int().positive().openapi({ example: 4200 }),
  currency: z.literal('EUR').openapi({ example: 'EUR' }),
}).openapi('Invoice');
export const createInvoiceRoute = createRoute({
  method: 'post',
  path: '/v1/invoices',
  request: { body: { content: { 'application/json': { schema: Invoice } } } },
  responses: { 201: { description: 'Created' } },
});
```

Java:
```java
// DON'T: a Confluence page describing the payload, updated by hand, or not
// DO: MicroProfile OpenAPI annotations render the spec from the code itself
@POST
@Path("/v1/invoices")
@Operation(summary = "Create an invoice")
@APIResponse(responseCode = "201", description = "Created")
@RequestBody(content = @Content(examples =
    @ExampleObject(value = "{\"amountCents\":4200,\"currency\":\"EUR\"}")))
public Response create(@Valid Invoice invoice) {
    return Response.status(201).entity(service.create(invoice)).build();
}
```

### 12.3 Record decisions where they cannot drift
**Do:** Commit an ADR next to the code, capturing the options rejected and how to reverse the choice.
**Don't:** Bury the "why" in a closed Slack thread that no future maintainer will ever find.

Stack-agnostic (ADR template, docs/adr/NNNN-title.md):
```md
<!-- DON'T: decision explained in chat, lost the moment the thread scrolls away -->
<!-- DO: versioned with the code, states alternatives and the exit -->
# 0007: Encrypt Tofu state client-side

- Status: accepted
- Date: 2026-07-06

## Context
State holds secrets. The backend bucket is shared. We need at-rest protection
independent of the provider.

## Decision
Encrypt state client-side before upload; keys in Secret Manager.

## Options considered
- Provider-side SSE only. Rejected: does not protect against bucket-scope leaks.
- Remote state service. Rejected: adds a vendor and a new failure mode.

## Consequences
- Every dev needs decrypt access to plan.
- Reversal: drop the encryption block, re-init the backend, rotate keys.
```

### 12.4 Build institutional memory
**Do:** Keep an append-only lessons log and a durable plan file so hard-won knowledge outlives the people who earned it.
**Don't:** Let the only record of a two-day outage postmortem live in one engineer's memory.

Stack-agnostic (append-only lessons log):
```md
<!-- DON'T: "we learned this last quarter" said aloud, then forgotten -->
<!-- DO: dated, append-only, each entry a durable and non-obvious lesson -->
# LESSONS.md  (append only; never rewrite history)

## 2026-06-14  Postgres advisory locks leak on connection reset
Symptom: migrations hung after a pod restart.
Cause: session-level lock never released when the socket dropped.
Fix: use transaction-level advisory locks (pg_advisory_xact_lock).
Guard: added a lock-timeout test in migrate.spec.ts.

## 2026-07-02  The mail provider rate-limits per sender, not per key
Guard: batch sends behind a token bucket in the mailer adapter.
```

### 12.5 Give real-time access and commit to measurable thresholds
**Do:** Let stakeholders watch the repo, CI, board, and live metrics, and state targets as numbers with a window.
**Don't:** Report progress in adjectives ("fast", "stable") that no one can verify or falsify.

Stack-agnostic (thresholds config, numbers not adjectives):
```json
{
  "//": "DON'T: 'the API should feel snappy and be reliable', unmeasurable",
  "//do": "DO: explicit budgets a dashboard can pass or fail against",
  "slo": {
    "availability": { "target": 0.999, "window": "30d" },
    "latency_p95_ms": { "target": 300, "route": "/v1/invoices" },
    "latency_p99_ms": { "target": 800, "route": "/v1/invoices" },
    "error_rate": { "target": 0.005, "window": "30d" }
  },
  "visibility": {
    "status_page": "https://status.example.com",
    "dashboard": "https://grafana.example.com/d/billing",
    "board": "https://github.com/org/billing/projects/1"
  }
}
```

### 12.6 Run one visible, honest backlog
**Do:** Keep every task, bug, and decision in one shared, openly visible tracker where anyone with a stake can read the priorities and statuses, and never carry a second shadow list in a spreadsheet or a chat thread.
**Don't:** Let the "real" priorities live in someone's head or a private doc while the official board shows a sanitized version.

Stack-agnostic (the artifact is the board configuration and a single-source rule, not app code):
```md
<!-- docs/process.md -- DON'T: a public board of safe work plus a private spreadsheet of what is actually burning.
     the gap between them is where trust and priority rot. -->
<!-- DO: one board is the source of truth; anything not on it does not get worked on -->
# Process
- Source of truth: https://github.com/org/billing/issues  (one board, read access for every stakeholder)
- Rule: if it is not on the board, it is not work. No side lists, no "can you also..." in DMs that never land.
- Bugs are first-class issues, not a hand-maintained "known issues" page that drifts from reality.
- Priorities are visible, including the things deliberately deferred and why, not only the work in flight.
- Status is honest: "blocked, waiting on X" beats "in progress" that has not moved in a week.
```

## Pillar 13: Clear ownership

Every part of the system needs a name attached to it, because shared responsibility with no accountable owner is responsibility that evaporates.

### 13.1 Make ownership explicit
**Do:** Declare an accountable owner for every area in CODEOWNERS, backed by a short RACI note.
**Don't:** Leave whole directories orphaned so reviews stall and nobody answers when they break.

Stack-agnostic (CODEOWNERS + RACI note):
```bash
# DON'T: no CODEOWNERS at all, so every PR waits on whoever happens to notice
# DO: every path maps to an accountable team; last match wins in this file
# .github/CODEOWNERS
*                       @org/platform          # default owner, nothing is orphaned
/packages/core/         @org/domain-team        # Accountable: domain-team lead
/packages/infra/        @org/platform           # Responsible: platform on-call
/apps/api/              @org/api-team
/docs/adr/              @org/architecture       # Consulted on any decision record

# RACI note (docs/OWNERSHIP.md): for each area name exactly one Accountable (A),
# any number of Responsible (R), who is Consulted (C) and Informed (I).
# Rule: exactly one A per area. If two teams claim A, split the area.
```

### 13.2 Separate duties
**Do:** Require an independent reviewer so the person requesting a change is never its sole approver, and gate prod access through that review.
**Don't:** Let an author self-merge to main or grant themselves standing production credentials.

Stack-agnostic (branch protection / required reviews):
```yaml
# DON'T: no protection, the author clicks merge on their own PR
# DO: independent approval required, author excluded, checks must pass
# branch protection for `main` (as code, e.g. via the GitHub API/Terraform)
required_pull_request_reviews:
  required_approving_review_count: 1
  require_code_owner_reviews: true          # a CODEOWNER must sign off
  dismiss_stale_reviews: true               # new commits re-open review
required_status_checks:
  strict: true
  contexts: ["build", "test", "typecheck", "lint"]
enforce_admins: true                        # admins are not exempt
# required_approving_review_count: 1 already means the author cannot self-merge:
# GitHub blocks self-approval, so a different reviewer must always sign off
restrictions:
  push: []                                  # nobody pushes to main directly
```

### 13.3 Keep an audit trail
**Do:** Record who approved, who granted an exception, and when emergency access was used, in a durable log.
**Don't:** Change production state through channels that leave no trace of who did what.

TypeScript:
```ts
// DON'T: mutate a resource with no record of the actor or the reason
await db.update(orgs).set({ plan: 'enterprise' }).where(eq(orgs.id, orgId));
// DO: write an immutable audit row in the same transaction as the change
export const upgradePlan = async (
  ctx: { actorId: string; reason: string },
  orgId: string,
): Promise<void> =>
  db.transaction(async (tx) => {
    await tx.update(orgs).set({ plan: 'enterprise' }).where(eq(orgs.id, orgId));
    await tx.insert(auditLog).values({
      actorId: ctx.actorId, action: 'plan.upgrade',
      target: orgId, reason: ctx.reason, at: new Date(),
    });
  });
```

Java:
```java
// DON'T: silent update, no actor, no reason, unauditable
orgRepository.update(orgId, "enterprise");
// DO: change and audit entry commit together, so every mutation is attributable
@Transactional
public void upgradePlan(String actorId, String orgId, String reason) {
    orgRepository.update(orgId, "enterprise");
    auditLog.persist(new AuditEntry(actorId, "plan.upgrade", orgId, reason, Instant.now()));
}
```

### 13.4 Make finding problems safe and let the accountable verify
**Do:** Protect the person who reports a problem and give the accountable owner the access to confirm "done" for themselves.
**Don't:** Shoot the messenger, then let the owner mark work complete on someone else's word.

TypeScript:
```ts
// DON'T: "done" is a trust-me boolean the owner cannot check independently
const done = true; // reporter got blamed; owner never re-ran anything
// DO: completion is a re-runnable, owner-invokable verification, not an opinion
export type Verification = { passed: boolean; evidence: string; checkedBy: string };
export const verifyFix = async (owner: string): Promise<Verification> => {
  const result = await runRegressionSuite('leak-2026-06-14'); // owner runs it too
  return { passed: result.ok, evidence: result.reportUrl, checkedBy: owner };
};
```

Java:
```java
// DON'T: a flag set from a status meeting, unverifiable by the owner
boolean done = true;
// DO: the owner triggers the check and gets evidence, not a secondhand claim
public Verification verifyFix(String owner) {
    RegressionResult r = regression.run("leak-2026-06-14");
    return new Verification(r.ok(), r.reportUrl(), owner);
}
```

## Pillar 14: Pave the road

Make the correct path the easy one, because a golden path teams can generate beats a wiki page telling them to assemble it by hand.

### 14.1 Provide golden paths as real artifacts
**Do:** Ship a scaffold that starts green with the gates already wired in, so a new service is compliant on line one.
**Don't:** Publish a "how to set up a service" wiki and leave every team to bolt on lint, tests, and CI themselves.

Stack-agnostic (scaffold command + generated structure):
```bash
# DON'T: "copy another repo and delete the bits you don't need" (drifts instantly)
# DO: one command generates a service that already passes every gate
$ bunx @org/create-service billing
# generates:
#   billing/
#   ├── src/index.ts            # app + /health, already wired
#   ├── src/index.test.ts       # a passing smoke test, so CI is green on commit 1
#   ├── .github/workflows/ci.yml  # build + test + typecheck + lint, pre-configured
#   ├── .githooks/pre-commit    # the same gates locally
#   ├── Dockerfile              # golden base image, non-root, pinned
#   └── README.md               # runnable, matches 12.1
$ cd billing && bun install && bun test   # green out of the box
```

### 14.2 Make it self-service
**Do:** Let a team provision an environment or pipeline on demand through a declarative request they own.
**Don't:** Route every new database or preview environment through a ticket queue and a human bottleneck.

Stack-agnostic (self-service environment as code):
```yaml
# DON'T: file a ticket, wait three days for ops to click through a console
# DO: declare the environment; a controller reconciles it automatically
# environments/billing-staging.yaml  (committed; applied by the platform)
apiVersion: platform.org/v1
kind: Environment
metadata:
  name: billing-staging
spec:
  template: standard-api        # the golden path from 14.1
  database: { engine: postgres, size: small, ephemeral: true }
  objectStore: { bucket: billing-staging }
  autostop: 12h                 # cost guardrail, no ticket to tear it down
# `git push` -> controller provisions Postgres, bucket, and a preview URL
```

### 14.3 Treat the platform as a product
**Do:** Give the platform real docs, a maintenance owner, and a feedback loop so it improves with its users.
**Don't:** Stand up the tooling once and let it rot until every team quietly forks around it.

Stack-agnostic (platform product manifest):
```md
<!-- DON'T: a one-off setup with no owner, no changelog, no way to report pain -->
<!-- DO: run it like a product, versioned, supported, and listening -->
# Platform: create-service  (v3.2.0)

- Owner: @org/platform  (Accountable; see CODEOWNERS)
- Support: #platform-help  (SLA: first response < 4 business hours)
- Changelog: CHANGELOG.md  (semver; breaking changes = major + migration note)
- Feedback: open an issue with label `platform/feedback`; triaged weekly
- Deprecation policy: two-minor-version notice before a template is removed
- Adoption metric tracked: % of services on the current major (target > 90%)
```


## Pillar 15: Enforce and verify

A rule nobody enforces and a control nobody tested share a root: neither has been proven to work. Wire every standard into hooks and CI so a violation blocks the change, and treat every control as a hypothesis until a test has walked the attacker's path and shown the bypass is refused.

### 15.1 Make the standard executable
**Do:** Wire lint, tests, coverage, and secret scanning into a pre-commit hook and the same checks into CI so a violation blocks the commit and the merge.
**Don't:** Leave the standard as prose in a README and hope everyone reads and honors it.

Stack-agnostic (the artifact is a git hook plus a CI job, not app code):

```bash
#!/usr/bin/env bash
# .githooks/pre-commit -- DON'T: an empty file, or one that only echoes a reminder
# echo "remember to run the tests before you push"  # advisory, blocks nothing

# DO: run the real gates and exit non-zero on the first failure so the commit is refused
set -euo pipefail
bun run lint:strict           # style and correctness rules, no warnings tolerated
bun test                      # the suite must be green
bun run coverage              # coverage thresholds enforced by the runner config
gitleaks protect --staged -v  # secret scan on staged content, before it ever lands
```

```yaml
# .github/workflows/ci.yml -- DO: CI runs the identical gates so a bypassed hook is still caught
# a local hook is a fast first line; CI is the line that cannot be skipped with --no-verify
jobs:
  gates:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - run: bun install --frozen-lockfile
      - run: bun run lint:strict && bun test && bun run coverage
      - run: gitleaks detect --redact  # full history scan, redacted output in logs
```

### 15.2 Prefer failing loud to passing quietly
**Do:** Configure coverage so an untested file reports as 0 percent and drags the number down.
**Don't:** Let untested files be invisible to the report, where absence reads as a pass.

Stack-agnostic (this is a coverage tool configuration, not app code):

```yaml
# bunfig.toml (TS) / jacoco config (Java) -- the failure mode is the whole point
# DON'T: coverage measured only over files that happen to be imported by a test.
#   a brand-new untested module is simply not counted, so the percentage stays green
#   and the gap is silent. a gate that cannot fail is worse than no gate: it lies.
# DO: force full discovery so an untested file is present in the denominator at 0 percent.
[test.coverage]
coverageThreshold = { line = 0.80, function = 0.80 }
coverageSkipTestFiles = true
coveragePathIgnorePatterns = []   # ignore nothing by default; every source file counts
# Java equivalent: JaCoCo runs over all classes in the module, and the
# check goal fails the build when a class sits below the minimum, including 0 percent.
```

### 15.3 No silent opt-out
**Do:** Fix the code or change a rule's severity once, at the project level, with a written reason.
**Don't:** Sprinkle inline suppressions that mute a check line by line.

TypeScript:
```ts
// DON'T: the warning is real, and this hides it from everyone forever
// eslint-disable-next-line @typescript-eslint/no-floating-promises
sendEmail(user);
// DO: handle it; if the rule is genuinely wrong, disable it in eslint config with a comment
await sendEmail(user);
```

Java:
```java
// DON'T: @SuppressWarnings buries the deprecation at the call site, invisibly
@SuppressWarnings("deprecation")
var client = legacyClient();
// DO: stop calling the deprecated API; migrate to the supported one
var client = supportedClient();
```

```yaml
# DO (project level, auditable): one decision, in the config, with a reason
# eslint.config: rule relaxed because our logger intentionally returns void
'@typescript-eslint/no-floating-promises': ['error', { ignoreVoid: true }]
```

### 15.4 Test the bypass, not the happy path
**Do:** Assert the forbidden path is refused — the wrong token, the missing token, the lower-privilege role — not only that the authorized call succeeds. The cross-tenant case is shown in 7.5.
**Don't:** Test only the happy path and assume the guard covers everyone else.

TypeScript:
```ts
// DON'T: proves nothing about authorization; the admin was always going to succeed
test("admin can purge", async () => {
  const res = await app.request('/v1/admin/purge', authAs(adminUser));
  expect(res.status).toBe(204);
});
// DO: a non-admin hitting the same path is refused — the guard is what the test proves
test("non-admin purge is refused", async () => {
  const res = await app.request('/v1/admin/purge', authAs(staffUser));
  expect(res.status).toBe(403);
});
```

Java:
```java
// DON'T: only the admin call, so a missing role check would still pass
@Test void adminCanPurge() {
    given().auth().oauth2(adminToken).delete("/v1/admin/purge").then().statusCode(204);
}
// DO: the lower-privilege token is the assertion; the bypass is what fails the test
@Test void nonAdminPurgeIsForbidden() {
    given().auth().oauth2(staffToken).delete("/v1/admin/purge").then().statusCode(403);
}
```

### 15.5 Compliance is not security
**Do:** Produce proof anyone can independently re-check by running it, not a ticked box in an audit sheet.
**Don't:** Treat a green compliance checklist as evidence the control actually works.

Stack-agnostic (the artifact is a runnable verification script, not app code):

```bash
#!/usr/bin/env bash
# scripts/verify-tls.sh
# DON'T rely on: a spreadsheet row reading "TLS 1.2+ enforced [x]" signed last quarter.
#   a checkbox is a claim; it is not the state of the running system today.
# DO: assert the property against the live endpoint, so the proof re-runs on demand.
set -euo pipefail
host="api.example.com"
# fail if the server negotiates anything below TLS 1.2
if openssl s_client -connect "$host:443" -tls1_1 </dev/null 2>/dev/null \
     | grep -q "Protocol.*TLSv1.1"; then
  echo "FAIL: $host accepted TLS 1.1" >&2
  exit 1
fi
echo "OK: $host refuses TLS < 1.2"  # anyone can run this and get the same answer
```

### 15.6 Audit the gaps between systems
**Do:** Test the full path a request travels, edge to service to database, so a hole between two correct systems is caught.
**Don't:** Verify each service in isolation and assume the composition is safe.

TypeScript:
```ts
// DON'T: unit-test the handler with a pre-authenticated context and stop there.
//   the gap is upstream: does the CDN/proxy actually strip a spoofed header?
const ctx = { orgId: orgA };            // hand-built, so the real edge is never exercised
expect(handler(ctx).orgId).toBe(orgA);  // green, and blind to the seam
// DO: drive the request through the real edge and assert the injected header is ignored
const res = await fetch(`${edgeUrl}/v1/expenses`, {
  headers: { authorization: bearer(orgB), "x-org-id": orgA }, // attacker forges the trust header
});
expect(res.status).toBe(404);  // the identity comes from the verified token, not the header
```

Java:
```java
// DON'T: a slice test that trusts the security context the framework handed you
//   never sees the proxy -> app seam where a forged header could win
// DO: full-stack test that sends the spoofed header through the real filter chain
@Test void forgedOrgHeaderIsIgnored() {
    given().auth().oauth2(tokenOrgB).header("X-Org-Id", orgA)
        .get("/v1/expenses/{id}", expenseA)
        .then().statusCode(404);  // scope derives from the token, the header is inert
}
```

### 15.7 Fix the class, not the instance
**Do:** Grep for every occurrence of the pattern, fix them all, and add a guard that fails the build if it returns.
**Don't:** Patch the one reported spot and leave siblings live.

Stack-agnostic (the artifact is a search command plus a CI guard, not app code):

```bash
# DON'T: patch only the file in the bug report and close the ticket.
#   the same raw interpolation almost certainly lives elsewhere, still exploitable.
# DO: enumerate the whole class first, fix each hit, then lock the door behind you.
rg -n '(db\.query|db\.execute)\(`.*\$\{' -- 'packages/**/*.ts'   # every string-interpolated SQL sink

# Then a ripgrep-backed CI gate makes a regression fail loudly:
# .github/workflows/ci.yml
#   - name: forbid interpolated SQL
#     run: |
#       if rg -q '(db\.query|db\.execute)\(`.*\$\{' -- 'packages/**/*.ts'; then
#         echo "raw interpolation in a SQL sink; use parameterised queries" >&2
#         exit 1
#       fi
```

### 15.8 Make proof re-checkable
**Do:** Ship reproducible evidence anyone accountable can run themselves, and make sure they have the access to run it.
**Don't:** Paste a screenshot of a passing run into a slide and call it verified.

Stack-agnostic (the artifact is a committed, runnable check, not app code):

```md
<!-- SECURITY.md -- how the isolation claim is verified, not a claim that it is -->
DON'T: attach `passing-tests.png` to the quarterly deck as the record of proof.
  a screenshot is unfalsifiable, undated in practice, and no one can re-run a picture.

DO: commit the check and document how to run it, so proof is reproducible on demand.
  Cross-tenant isolation:  `bun test packages/api/tenancy.test.ts`
  Every endpoint carries an org-A-token-against-org-B-resource case (see 7.5).
  Access: the security owner has read on this repo and CI; the run is one command.
  Evidence = the exit code you get when you run it, not an image someone showed you.
```

### 15.9 Spend human judgment where it counts
**Do:** Let the machine own formatting and mechanics so review can focus on design and whether this is the right change.
**Don't:** Burn a reviewer's attention debating brace placement a formatter already settles.

TypeScript:
```ts
// DON'T (a review comment): "put a space after the comma and align these"
//   the formatter already has an opinion; the human just wasted a round trip
// DO (a review comment): "this use-case now knows about the HTTP layer.
//   should the presenter map the error instead, so core stays transport-free?"
//   mechanics are settled by `bun run lint` in CI; the human debates the design
```

Java:
```java
// DON'T (a review comment): "tabs vs spaces here, and this import is unused"
//   Spotless and the compiler catch both; the comment adds no judgment
// DO (a review comment): "ExpenseService reaches into the repository twice for the
//   same row. is a single load clearer, and does this belong in the domain at all?"
```

## Pillar 16: Measure whether you are improving

Numbers only help if they point at the system and move over time; instrument delivery and flow, watch the trend, and never turn them into a stick.

### 16.1 Track the four DORA metrics
**Do:** Derive deploy frequency, lead time, change failure rate, and time to restore from real CI/CD events using a Four Keys style pipeline.
**Don't:** Hand-maintain a status slide someone updates from memory once a month.

Use a Four Keys style pipeline: CI/CD emits a structured event on every deployment and every incident; those events land in a warehouse table, and a dashboard computes the four metrics from them. The source of truth is the pipeline, not a person. Stack-agnostic (the artifact is the event emission step in CI):

```yaml
# .github/workflows/deploy.yml -- DO: emit a machine-readable deployment event
# DON'T: rely on humans typing "we deployed ~5 times" into a spreadsheet later.
#   metrics you cannot recompute from raw events are anecdotes with decimals.
- name: record deployment event
  if: success()
  run: |
    # posted to the Four Keys ingestion endpoint; lead time = commit ts -> deploy ts
    curl -sf -X POST "$FOURKEYS_EVENT_URL" \
      -H 'content-type: application/json' \
      -d "{\"event_type\":\"deployment\",
           \"service\":\"app-api\",
           \"sha\":\"${GITHUB_SHA}\",
           \"deployed_at\":\"$(date -u +%FT%TZ)\"}"
# A matching "incident" event (opened + resolved timestamps) feeds
# change failure rate and time to restore. Four metrics, one event stream.
```

### 16.2 Pair delivery metrics with flow metrics
**Do:** Track cycle time, throughput, and work in progress alongside the delivery metrics so you see where work actually stalls.
**Don't:** Report velocity in story points as if a rising number meant faster value.

Stack-agnostic (the artifact is a flow query over issue-tracker timestamps, not app code):

```sql
-- DON'T: sum(story_points) per sprint. points are a local estimation currency;
--   they inflate, they do not compare across teams, and they hide queue time.
-- DO: measure the clock. cycle time is when work started until it shipped.
SELECT
  date_trunc('week', done_at)                     AS week,
  percentile_cont(0.5) WITHIN GROUP (
    ORDER BY done_at - started_at)                AS median_cycle_time,  -- flow speed
  count(*)                                        AS throughput,         -- items shipped
  avg(open_items_at_snapshot)                     AS avg_wip             -- work in progress
FROM issues
WHERE done_at IS NOT NULL
GROUP BY 1 ORDER BY 1;
```

### 16.3 Treat them as system metrics, not a stick for individuals
**Do:** Report every metric at the team or service level so it drives process improvement.
**Don't:** Build per-developer leaderboards that reward gaming and punish honesty.

Stack-agnostic (the artifact is the grouping in the metrics query, the choice that sets the culture):

```sql
-- DON'T: rank humans. this invites padded estimates, split PRs to juke the count,
--   and silence about real problems. it measures fear, not throughput.
--   SELECT author, count(*) AS prs FROM merged GROUP BY author ORDER BY prs DESC;
-- DO: group by the system under improvement, never by person.
SELECT service, percentile_cont(0.5) WITHIN GROUP (ORDER BY lead_time) AS median_lead
FROM deployments GROUP BY service;
-- the question is "is our delivery getting healthier", not "who is fastest this week".
```

### 16.4 Watch the trend, not the snapshot
**Do:** Read the direction across several weeks and alert on a sustained change.
**Don't:** React to a single reading, where normal variation looks like a crisis or a win.

Stack-agnostic (the artifact is the alert rule, evaluated over a window, not app code):

```yaml
# alerting rule -- DON'T: page on one bad day
#   expr: change_failure_rate > 0.15    # a single spiky afternoon fires this
# DO: require a sustained shift before anyone is alarmed
groups:
  - name: delivery-health
    rules:
      - alert: ChangeFailureRateTrendingUp
        # 7-day average, and it must hold above threshold for a day, not a moment
        expr: avg_over_time(change_failure_rate[7d]) > 0.15
        for: 24h
        annotations:
          summary: "CFR trending up over a week -- look at the process, not yesterday"
```

### 16.5 Treat cost as a first-class metric
**Do:** Measure spend per service and alert on unexplained growth, the way you alert on latency, and design components to cost near nothing when idle.
**Don't:** Discover cost in the monthly invoice, after the waste has run for weeks.

Stack-agnostic (the artifact is a cost-growth alert over the billing export, not app code):
```yaml
# DON'T: cost surfaces once a month in a PDF no engineer reads, weeks after the leak.
# DO: alert on spend growth the same way you alert on error rate, against a budget.
# cloud_cost_usd_total: daily spend per service, exported from the cloud billing feed
- alert: DailyCostAnomaly
  # spend up >50% week-over-week, sustained for a day, before anyone is paged
  expr: avg_over_time(cloud_cost_usd_total[7d]) > 1.5 * avg_over_time(cloud_cost_usd_total[7d] offset 7d)
  for: 24h
  annotations:
    summary: "Daily spend up >50% week-over-week -- a service is leaking money"
```

## Pillar 17: Obsess over the whole experience

The product is the entire journey the person lives through, so treat error copy, trust, real behavior, and the human fallback as part of what you ship.

### 17.1 Treat the whole journey as the product
**Do:** Write error copy that tells the person what happened and what to do next, and give the API a stable machine-readable shape behind it.
**Don't:** Surface a raw status code or stack trace and leave the user stranded.

TypeScript (React):
```tsx
// DON'T: dumps the transport failure at the person, who can do nothing with it
function Error({ status }: { status: number }) {
  return <p>Error {status}</p>;  // "Error 413" means nothing to an employee in a taxi
}
// DO: name the cause and the next step, in the app's voice; string comes from the catalog
function ReceiptTooLarge({ t }: { t: Translate }) {
  return (
    <Callout tone="warning">
      {/* t("receipt.tooLarge") => "This photo is over 10 MB. Retake it or shrink it." */}
      {t("receipt.tooLarge")}
    </Callout>
  );
}
// The API shape behind it is stable and typed, so the client can branch on `code`, not prose:
//   { "error": { "code": "receipt_too_large", "maxBytes": 10485760 } }
```

### 17.2 Earn trust rather than extract a sale
**Do:** Default to the honest option and make cancelling as easy as subscribing.
**Don't:** Bury the exit in a dark pattern that traps the user into staying.

TypeScript:
```ts
// DON'T: force a phone call to cancel while signup was one click. this is a dark pattern:
//   it extracts a month of resentment, not a renewal.
app.post("/v1/subscription/cancel", () => err({ code: "call_support_to_cancel" }));
// DO: cancel is self-serve and symmetric with subscribe; the user leaves clean.
app.post("/v1/subscription/cancel", async (c) => {
  const r = await cancelAtPeriodEnd(c.get("orgId"));  // keeps access already paid for
  return r.ok ? c.json({ endsAt: r.value.periodEnd }) : c.json(mapError(r), 400);
});
```

Java:
```java
// DON'T: no cancel endpoint at all, so the only exit is a support ticket queue
// DO: a first-class, self-serve cancel that mirrors how they signed up
@POST @Path("/subscription/cancel")
public Response cancel(@Context OrgContext ctx) {
    var ends = billing.cancelAtPeriodEnd(ctx.orgId());  // honest: no clawback of paid time
    return Response.ok(new CancelResult(ends)).build();
}
```

### 17.3 Design for real behavior, not the demo
**Do:** Ship the change behind a feature flag, instrument it, and run an experiment on real users.
**Don't:** Roll it to everyone on the strength of a demo and a strong opinion.

TypeScript (the artifact centers on a feature-flag call):

```ts
// DON'T: hardcode the new flow for all, betting the whole userbase on one opinion
const useNewCapture = true;  // no gradual rollout, no measurement, no way back but a deploy
// DO: gate on a flag, expose a fraction, and let the analytics decide
const useNewCapture = flags.isEnabled("capture-prefill-v2", {
  orgId: ctx.orgId,              // targeting + a stable bucket for the experiment
});
analytics.track("capture_started", { orgId: ctx.orgId, variant: useNewCapture ? "v2" : "v1" });
// keep or kill the flow on the measured completion rate, not on how the demo felt.
```

### 17.4 Let technology serve the person, not replace them
**Do:** Automate to remove friction, and always leave a visible path to a human.
**Don't:** Trap the user in a bot with no way to reach a person.

TypeScript (React):
```tsx
// DON'T: a bot that loops and offers no exit is a dead end, not support
function Help() {
  return <Bot fallback={null} />;  // when it cannot help, the user is simply stuck
}
// DO: the assistant handles the easy path and the human path is always one tap away
function Help({ t }: { t: Translate }) {
  return (
    <>
      <Bot fallback={null} />
      {/* always rendered, never hidden behind N failed bot turns */}
      <ContactHuman href="mailto:support@example.com">
        {t("help.talkToSomeone") /* "Talk to someone" */}
      </ContactHuman>
    </>
  );
}
```

### 17.5 Speak the user's language
**Do:** Hold every user-facing string in a catalog keyed by meaning, so localization is a data change, not a code change.
**Don't:** Hardcode prose in components, so every new market is a rewrite of every screen.

TypeScript (React):
```tsx
// DON'T: copy baked into the component; a new language means editing every screen
function Empty() {
  return <p>No expenses yet. Add your first receipt.</p>;
}
// DO: a key into a catalog; the translator works on the catalog, not the code
function Empty({ t }: { t: Translate }) {
  return <p>{t("expenses.empty")}</p>; // "Aucune note de frais. Ajoutez votre premier justificatif."
}
```

## Pillar 18: Validate before you build

The cheapest code is the code you did not write; talk to users, test demand with a page instead of a product, decide on a date, and keep validating after launch.

### 18.1 Talk to real users before you write code
**Do:** Run short problem interviews about how people handle the pain today before committing an engineer.
**Don't:** Build a feature on an internal assumption no customer has confirmed.

Problem interviews. Stack-agnostic (the artifact is an interview script, not app code):

```md
<!-- research/problem-interview.md -- validate the problem, not your solution -->
DON'T: open with "would you use an app that auto-fills expense reports?"
  a leading question about your idea harvests polite yeses, not evidence.

DO: ask about the last real occurrence of the problem, in their words.
  1. "Walk me through the last expense you filed. What did you actually do?"
  2. "Where did that get annoying or slow?"        (find the pain, unprompted)
  3. "What did you do to work around it?"           (proof it hurts enough to act)
  4. "How often does this happen to you?"           (frequency = size of the wound)
  Ten of these before a line of feature code. Listen for pain you did not invent.
```

### 18.2 Test demand with the cheapest thing
**Do:** Put up a landing page with an email capture, or run a concierge MVP by hand, before building the product.
**Don't:** Build the full system to discover whether anyone wants it.

A static landing page (any host) with a form wired to an email-capture and product-analytics tool, plus a manual concierge process for the first users. Stack-agnostic (the artifact is the capture snippet on the page):

```html
<!-- landing/index.html -- DON'T: skip straight to a 3-month build to test demand -->
<!-- DO: measure intent for the price of a page; a signup is a vote with an email -->
<form onsubmit="capture(event)">
  <input name="email" type="email" required placeholder="Your email" />
  <button>Notify me at launch</button>
</form>
<script>
  function capture(e) {
    e.preventDefault();
    const email = e.target.email.value;
    // product-analytics tool: records the conversion so we can read the demand signal
    analytics.track("waitlist_signup", { email, source: "landing" });
    // concierge MVP: for the first N signups we process expenses by hand and learn,
    // building nothing automated until the demand and the workflow are proven.
  }
</script>
```

### 18.3 Set a dated, honest go/no-go
**Do:** Write a checklist with explicit criteria, a decision, and a date, and hold to it.
**Don't:** Let momentum carry the team into a build no one formally chose.

Stack-agnostic (the artifact is a committed go/no-go checklist, not app code):

```md
<!-- docs/go-no-go-capture.md -- a decision with a date, not a vibe -->
DON'T: "everyone seems keen, let us just start" -- no bar, no date, no way to say no.

DO: name the criteria up front, then mark the honest verdict on the day.
# Go / No-Go: auto-prefill capture -- decided 2026-07-06
- [x] >= 10 problem interviews surfaced this pain unprompted        (met: 12)
- [x] landing page conversion >= 8 percent                          (met: 11 percent)
- [ ] >= 3 SMEs committed to a paid pilot                           (NOT met: 1)
Decision: NO-GO. Re-decide by 2026-08-01 after 2 more pilot conversations.
Owner: product. The unmet criterion is the reason, written down, not smoothed over.
```

### 18.4 Keep validating after launch
**Do:** Ship behind a flag, measure real usage against a threshold, and keep or kill on the evidence.
**Don't:** Assume a feature that shipped is a feature that gets used.

TypeScript (the artifact is a feature-flag call joined to a usage metric):

```ts
// DON'T: launch, delete the flag, move on, and never check if anyone adopted it.
//   "shipped" is an output; "used" is the outcome, and only one pays rent.
// DO: keep the flag as a kill switch and gate the keep/kill on a measured threshold.
if (flags.isEnabled("capture-prefill-v2", { orgId })) {
  analytics.track("prefill_used", { orgId });  // adoption signal, per real org
}
// A scheduled check reads the metric and forces the decision:
//   adoption(prefill_used, last=28d) < 0.05  =>  disable the flag, write the reason,
//   remove the code. no silent zombie features left half-live in the product.
```

---

*Eighteen pillars, each with its Do and Don't — the stack is a detail; these are not. See [The Global Rules Every New Project Should Have](global-rules-every-new-project.md) for the prose, and [Core Values: what my writing says I stand for](core-values-one-pager.md) for the five values they serve.*

