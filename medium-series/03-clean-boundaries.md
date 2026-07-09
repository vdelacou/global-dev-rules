================================================================================
PUBLISH METADATA   copy each field into the matching Medium field
================================================================================

TITLE
Keep Clean Boundaries

SUBTITLE
Swap the database, the framework, or the entire visual theme one at a time, and the parts that should not care never notice. This is the code that makes that real.

TAGS   (5 max, each 25 characters or fewer)
Software Architecture, Clean Code, Frontend Development, Programming, TypeScript

COVER IMAGE   upload this file from the covers folder
covers/03-clean-boundaries.png

================================================================================
ARTICLE BODY   select everything below this line and paste into Medium
================================================================================

*Part 3 of 9 of The Global Rules Every New Project Should Have.*

A system stays changeable only if its parts are actually separable. When business logic knows about the database, a new framework becomes a rewrite. When the look-and-feel is tangled into the logic, a rebrand becomes a hunt through the whole codebase. Every change ripples out to code that had no business caring about it.

The fix is to draw the boundaries deliberately, so each part can change without disturbing the ones that should not care. This is the pillar with the most moving parts, so this part is heavy on code. Six ideas, each with the Don't next to the Do. Read the contrast, not just the prose.

## Point dependencies inward

Your business logic should know nothing about the database, the web framework, or the cloud it runs on. It reaches the outside world only through interfaces it defines itself. Infrastructure depends on the logic, never the other way around. That single arrow, pointing inward, is what lets you swap a vendor or test the entire core against fakes without rewriting it.

The violation looks harmless. A use-case imports the ORM to do one update.

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

Once that import exists, your domain is welded to Postgres. Change the database and `activate` changes. Test the rule and you need a database running. The `AccountStore` version depends on an interface the domain owns, and infra implements it somewhere the domain never looks. Same in Java: have the service call a port instead of reaching for Panache, and the JPA adapter lives out in the infrastructure layer where it belongs.

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

## Put every external thing behind a port, with a fake

A database, a payment gateway, a queue, an object store, an email sender. Each sits behind a small interface with two implementations: a real adapter for production and an in-memory fake for tests. You choose which one at a single wiring point. One contract, many implementations.

The fake is the part people skip, and it is the part that pays off every day. Without it, every test that touches storage needs a network or a heavy mock.

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

`MemoryBlobs` is eight lines and it makes every test that stores a file instant and offline. The real S3 adapter implements the same `Blobs` interface, and the composition root is the only place that knows which is which. Everywhere else in the codebase just sees `Blobs`.

## Seal the presentation behind a design system

Make one design system the single source of visual truth. Define every visual value once as a semantic token: color, spacing, type. Forbid raw literals everywhere else. Components take data in and return markup, with no business logic and no data fetching inside them. The styling engine stays invisible to the rest of the app.

A presentational component that hardcodes a hex color and fetches its own data breaks two boundaries at once.

```tsx
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

The first `Badge` cannot be reused, cannot be themed, and cannot be tested without stubbing a network call. The second takes everything through props and reads its color from a token. Change `--color-success` in one place and every success badge in the product moves with it. That is what makes a rebrand a one-folder change instead of a codebase-wide hunt.

The same discipline keeps business rules out of components. A VAT rate stranded in a React component means a tax change edits your UI.

```tsx
// DON'T: the tax rule lives in the React component, so a rate change edits the UI
function Total({ items }: { items: Item[] }) {
  const net = items.reduce((s, i) => s + i.price, 0);
  const total = net * 1.2; // VAT rule stranded in presentation
  return <div className="total">{total}</div>;
}
// DO: the rule sits in the domain, the component only renders the computed value
export const withVat = (net: number): number => net * VAT_RATE;   // core/pricing.ts
function Total({ total }: { total: number }) { return <div className="total">{total}</div>; }
```

## Keep the backend a client-agnostic API

Expose one resource-shaped API that every client consumes the same way: web, iOS, Android, a third-party integration, an AI agent. Not endpoints shaped to a single screen.

This is the boundary most backends get wrong, and the cost shows up later. A screen-shaped endpoint recouples the backend to one interface. It has to change every time that screen does, and it is useless to the next client, which has a different screen. A resource-shaped API lets any client compose what it needs from what already exists.

```ts
// DON'T: one endpoint per screen; the backend now knows the mobile home layout
app.get('/mobile-home-screen', (c) =>
  c.json({ banner, greeting, recentOrders, promoWidget }));
// DO: stable resource endpoints; each client composes its own screen
app.get('/orders', (c) => c.json(listOrders(c.get('userId'))));
app.get('/promotions', (c) => c.json(listActivePromotions()));
```

When the mobile team redesigns the home screen, the first version forces a backend change and a deploy. The second does not: the app recomposes `/orders` and `/promotions` however it likes, and a brand-new AI agent client hits the exact same endpoints without anyone building it a bespoke one. Design the API around the nouns of your domain, not the layout of today's screen.

## Build the frontend against a contract, not a running backend

Agree the API contract first. Then have the frontend reach data only through a gateway with two implementations: a real client, and a fake returning canned data in the same shape. Now the UI is built, demoed, and tested in parallel with the backend instead of waiting on it.

The anti-pattern is calling `fetch` straight from a component, which blocks all frontend work until the API is deployed.

```tsx
// DON'T: the component is wired to a live endpoint; blocked until the API is deployed
function Orders() {
  const [orders, setOrders] = useState<Order[]>([]);
  useEffect(() => { fetch('/api/orders').then((r) => r.json()).then(setOrders); }, []);
  return <OrderList orders={orders} />;
}
// DO: components depend on an OrderGateway interface; swap real and fake by config
type OrderGateway = { list: () => Promise<Order[]> };
const httpOrders: OrderGateway = { list: () => api.get('/orders') };
const fakeOrders: OrderGateway = { list: async () => [{ id: '1', total: 80 }] }; // canned, contract-shaped
export const orderGateway: OrderGateway = env.USE_FAKE_API ? fakeOrders : httpOrders;
```

Publish the contract, OpenAPI or a shared schema, before either side writes an implementation. The frontend generates a typed client and a mock from it. Both teams code to the contract, not to each other's calendars. When the real endpoint lands, one env flag flips from fake to real and no UI code changes. Your designer can click through a fully working app days before the backend exists.

## Map the API into your own model, and keep the domain out of the database

Two boundaries, one principle: the shape you receive at an edge is never the shape you work with inside.

On the frontend, translate the API response into your own model at the gateway, and let the two shapes differ. Pass raw API DTOs through the app and every field the API renames breaks code in a dozen components.

```tsx
// DON'T: the raw API shape (snake_case, cents, nullable) leaks into every component
type ApiOrder = { order_id: string; total_cents: number; customer: { first_name: string | null } };
const OrderRow = ({ o }: { o: ApiOrder }) =>
  <div>{o.customer.first_name ?? 'Guest'} pays {o.total_cents / 100}</div>;
// DO: the gateway maps the DTO into an internal model the UI owns
type Order = { id: string; total: Money; customerName: string };   // your shape
const toOrder = (d: ApiOrder): Order => ({                         // the one mapping point
  id: d.order_id,
  total: Money.fromCents(d.total_cents),
  customerName: d.customer.first_name ?? 'Guest',
});
const httpOrders: OrderGateway = { list: async () => (await api.get<ApiOrder[]>('/orders')).map(toOrder) };
```

Every component uses `Order`. When the API renames `order_id`, exactly one function changes: `toOrder`. The blast radius of an API change collapses from "everywhere" to "one mapping point."

The same discipline holds on the server, and it is the one teams resist hardest: your domain model is not your database model. The two serve different purposes, so give them different shapes, map between them at the repository, and let the business drive the schema rather than the schema drive the business. Serialize a JPA entity straight to JSON and your database table shape becomes your public API shape, forever.

```java
// DON'T: the JPA entity is serialized straight to JSON, so the DB shape becomes the API shape
@GET public List<Order> all() { return orderRepo.listAll(); } // Order is the @Entity
// DO: map the domain to a response record at the boundary; the two can differ
@GET public List<OrderResponse> all() {
  return service.list().stream().map(OrderResponse::from).toList();
}
// OrderResponse is shaped for the API; the domain Order stays internal, DTOs never enter use-cases
```

And the domain object itself should carry behavior, not persistence concerns. A Drizzle row passed around as "the order" leaks nullable columns and table shape into every business rule that reads it.

```ts
// DON'T: the Drizzle row is "the order"; nullable columns and DB shape leak everywhere
const row = await db.query.orders.findFirst({ where: eq(orders.id, id) });
if (row.status === 'PAID') { /* business logic reading raw DB columns */ }
// DO: a domain type with behavior; the repository maps row -> domain, the DB shape stops there
type Order = { id: OrderId; lines: Line[]; status: OrderStatus };
const canRefund = (o: Order): boolean => o.status === 'paid' && o.lines.length > 0; // a business rule
const orderRepo = {
  find: async (id: OrderId): Promise<Order | null> => {
    const row = await db.query.orders.findFirst({ where: eq(orders.id, id) });
    return row ? toDomain(row) : null; // one mapping point
  },
};
```

The wire DTO, the domain type, and the database row are three different shapes, and they never cross. Each boundary has exactly one mapping function. That is the entire trick: an external change is absorbed in one place instead of rippling through everything that touched the leaked shape.

## Make the boundary testable, not aspirational

A boundary you cannot test is a diagram, not an architecture. The proof is mechanical, and you should be able to state it plainly:

- A rebrand touches only the design-system folder.
- Swapping the styling engine leaves the app unchanged.
- A change to a business rule never edits a component.
- Replacing the database changes only the repository mappings.

If one change has to reach across a boundary, the boundary has already leaked, and you found out cheaply instead of during a migration. That is the whole point of drawing them deliberately: not architectural purity for its own sake, but the ability to replace the database, the web framework, or the entire visual theme one at a time, while the parts that should not care never notice.

Clean boundaries give you separable parts. The next rule makes sure those parts actually work. In the next part, **Proof Over Hope**, I cover testing: layered tests, why mutation score is the coverage metric that matters, and how to test behavior through your ports so you can restructure the code freely without rewriting a single test.
