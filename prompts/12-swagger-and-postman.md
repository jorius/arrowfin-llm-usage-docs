# 12 — Swagger documentation and Postman collection

**When:** about 11:26, while the UI iteration tracks were running.

**Context:** The backend track was mid-way through the WebSocket configuration change.

**Prompt** (verbatim):

```text
Let's also create Swagger documentation within the NestJS backend along with a Postman collection that can be imported and showcase endpoints, documentation and at least a single smoke test for authentication, that's it
```

**Outcome:** Queued behind the WebSocket work in the same backend track so only one agent writes to that repository: OpenAPI via `@nestjs/swagger` served at `/docs` and switched off under `NODE_ENV=production` like the dev fill simulator, explicit decorators on controllers and DTOs, a Postman v2.1 collection with staging and development environment files, and an authentication smoke flow (issue key asserts 201 and an `afk_` key, wrong secret asserts 401, guarded calls, foreign account asserts 404, revoke asserts 204).
