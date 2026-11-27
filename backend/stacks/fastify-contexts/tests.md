---
paths:
  - "**/*.test.ts"
  - "**/*.fakes.ts"
---

# Fastify Contexts: Tests

Vitest runs every `src/**/*.test.ts`. A test file sits next to the file it tests. Tests never need a database, Redis, or network.

| Under test | How |
|---|---|
| Domain rule or pure function | Plain unit test |
| Application Service | Construct it with fakes for its `deps` |
| Controller | A bare Fastify app with the real error filter and a fake session, called with `app.inject` |
| Repository query | A recording Kysely driver that returns canned rows and captures the SQL |

## Service tests

When a context has several Service tests, put the fakes in `application/<name>.fakes.ts`: a harness that builds the Service over an in-memory "world".

```ts
// application/order.fakes.ts
export const ORDER_ACCOUNT_ID = randomUUID();
export const ORDER_NOW = new Date('2026-01-01T12:00:00.000Z');

export function createOrderHarness() {
  const world = { orders: [] as Order[], emails: [] as EmailMessage[] };

  const orders: IOrderRepository = {
    findByIdForAccount: async (accountId, id) =>
      world.orders.find((o) => o.id.value === id && o.accountId.value === accountId) ?? null,
    findByReference: async (accountId, reference) =>
      world.orders.find((o) => o.accountId.value === accountId && o.reference === reference) ?? null,
    insert: async (order) => void world.orders.push(order),
    update: async () => {},
    deleteForAccount: async (_accountId, id) => {
      world.orders = world.orders.filter((o) => o.id.value !== id);
    },
  };

  const app: OrderServiceApp = {
    uow: { withTransaction: (fn) => fn(undefined as never) },
    email: { send: async (msg) => (world.emails.push(msg), { id: null, delivered: true }) },
    log: { warn: vi.fn(), error: vi.fn(), info: vi.fn() } as unknown as FastifyBaseLogger,
  };

  return { world, service: new OrderService({ orders, app, now: () => ORDER_NOW }) };
}
```

```ts
// application/order.service.test.ts
describe('OrderService.create', () => {
  it('rejects a duplicate reference with ORDER_ALREADY_EXISTS', async () => {
    const h = createOrderHarness();
    await h.service.create(ORDER_ACCOUNT_ID, { reference: 'A-1', quantity: 1 });

    await expect(
      h.service.create(ORDER_ACCOUNT_ID, { reference: 'A-1', quantity: 2 }),
    ).rejects.toMatchObject({ code: 'ORDER_ALREADY_EXISTS' });
    expect(h.world.orders).toHaveLength(1);
  });
});
```

- Assert on the error `code` and on the world, not on internal calls.
- Fake only the methods the Service uses. A narrow `<Name>App` type keeps the fake small.
- Control time with the `now` dependency, never with real timers.

## Controller tests

```ts
async function buildApp(h = createOrderHarness()) {
  const app = Fastify();
  app.setValidatorCompiler(validatorCompiler);
  app.setSerializerCompiler(serializerCompiler);
  app.setErrorHandler((error, request, reply) => handleHttpException(request, reply, error));
  app.addHook('onRequest', async (request) => {
    if (request.headers['x-test-session'] === 'none') return;
    request.auth = {
      userId: ORDER_ACCOUNT_ID,
      accountId: ORDER_ACCOUNT_ID,
      email: 'user@example.com',
      role: 'user',
      impersonatedBy: null,
    };
  });
  await app.register(orderController(h.service), { prefix: '/orders' });
  return app;
}
```

- Cover the guard answers (401, 403), validation (422), and the status of each route.
- Close every app in `afterEach`.

## Rules

- A new business rule or bug fix in a Service gets a test.
- Tests that call a real LLM or external API are evals, with their own config and command. They never run in `pnpm test`.
