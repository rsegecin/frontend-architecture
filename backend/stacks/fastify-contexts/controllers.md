---
paths:
  - "**/presentation/http/**"
  - "**/lib/errors/**"
  - "**/lib/guards/**"
---

# Fastify Contexts: Controllers and Contracts

## Controller shape

A controller is a factory that receives its Services and returns a Fastify plugin. It declares routes and nothing else.

```ts
// src/contexts/orders/presentation/http/order.controller.ts
import type { FastifyPluginAsync } from 'fastify';
import type { ZodTypeProvider } from 'fastify-type-provider-zod';
import { z } from 'zod';

import { Guardian } from '../../../../lib/guards/index.js';
import type { OrderService } from '../../application/order.service.js';
import { CreateOrderInput } from './dto/input/create-order.input.js';
import { OrderListOutput, OrderOutput } from './dto/output/order.output.js';

const OrderIdParams = z.object({ id: z.string().uuid() });

export const orderController =
  (service: OrderService): FastifyPluginAsync =>
  async (app) => {
    app.addHook('preHandler', Guardian.Authenticated);
    const r = app.withTypeProvider<ZodTypeProvider>();

    r.get(
      '/',
      {
        schema: {
          operationId: 'listOrders',
          tags: ['Orders'],
          security: [{ session: [] }],
          response: { 200: OrderListOutput },
        },
      },
      async (request) => service.list(request.auth!.accountId),
    );

    r.post(
      '/',
      {
        preHandler: Guardian.NotImpersonated,
        schema: {
          operationId: 'createOrder',
          tags: ['Orders'],
          security: [{ session: [] }],
          body: CreateOrderInput,
          response: { 201: OrderOutput },
        },
      },
      async (request, reply) => {
        const out = await service.create(request.auth!.accountId, request.body);
        return reply.status(201).send(out);
      },
    );

    r.delete(
      '/:id',
      {
        schema: {
          operationId: 'deleteOrder',
          tags: ['Orders'],
          security: [{ session: [] }],
          params: OrderIdParams,
          response: { 204: z.null() },
        },
      },
      async (request, reply) => {
        await service.remove(request.auth!.accountId, request.params.id);
        return reply.status(204).send(null);
      },
    );
  };
```

## Route rules

- Every route has an `operationId` in camelCase, unique across the API, plus `tags`. Generated clients name their methods after it. Add `summary` when the name alone is unclear.
- Authenticated routes declare `security: [{ session: [] }]`, so the OpenAPI document shows the requirement.
- A handler makes one Service call and returns its result. Business checks, loops over Repositories, and response mapping belong in the Service.
- Return the Service result as is. Add a cast only when TypeScript cannot match the Service type to the response schema, never to hide a real mismatch.
- Set the status with `reply.status(201).send(out)`. A route with no body answers 204 with the `z.null()` response.
- Never `try/catch` to build an error response. Throw, and the error filter answers.
- Small route-only schemas, such as `OrderIdParams`, may sit at the top of the controller. Anything reused goes in `dto/`.

## Guards

`Guardian` in `src/lib/guards/` holds the guards: `Authenticated`, `Admin`, `Owner`, `NotImpersonated`.

- A controller where every route needs a session adds `app.addHook('preHandler', Guardian.Authenticated)` once.
- A guard for one route goes in that route's `preHandler`.
- The session plugin fills `request.auth` before guards run. After `Guardian.Authenticated`, read it with `request.auth!`.
- Write routes with financial or account effects add `Guardian.NotImpersonated`, because an impersonated session is read-only.
- A guard answers "may this caller use this route". Whether the caller may touch a specific resource is checked in the Service.

## Contracts

One file per Contract. The zod schema and its inferred type share one name.

```ts
// presentation/http/dto/input/create-order.input.ts
import { z } from 'zod';

export const CreateOrderInput = z.object({
  reference: z.string().min(1).max(64),
  quantity: z.number().int().positive(),
  notes: z.string().max(1000).optional(),
});
export type CreateOrderInput = z.infer<typeof CreateOrderInput>;
```

- Inputs end in `Input` and live in `dto/input/`; responses end in `Output` and live in `dto/output/`.
- A paginated list takes `PaginationQuery` (extended with its filters) as `querystring` and answers `paginated(OrderOutput)`, both from `shared-schemas`. An unpaginated list is `{ data: z.array(OrderOutput) }`, never a bare array.
- Dates in responses are ISO strings: `z.string().datetime()`. The Service converts with `toISOString()`.
- Primitives, enums, ids, and error codes shared with the front end come from the workspace `shared-schemas` package. Never redefine one locally.
- A Service may import Contract types with `import type`. Domain and Repositories never import them.

## Errors

- Throw one of the exceptions in `src/lib/errors/index.ts`: `ItemNotFound`, `ConflictException`, `UnauthorizedException`, `ForbiddenException`, `ApplicationLevelError`, `TooManyRequestsException`, `ServiceUnavailableException`, and the others there. Domain code throws `DomainError`.
- Each exception carries an `ErrorCode` from `shared-schemas`. A new code goes into that union first.
- `handleHttpException` in `http-exception.filter.ts` maps every exception to its status and `OutputError` body. A new exception class gets a branch there, before the generic fallbacks.
- Unknown errors are logged once by the filter and answered as 500 `UNEXPECTED`. Do not log them again before rethrowing.
