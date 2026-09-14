# 02 — Answers to the four decision questions

**When:** about 10:18, in reply to the agent's findings brief.

**Context:** The agent had presented the research findings and asked four questions: deployment scope, repository layout, tenant-isolation depth and the authentication model (it had proposed a dev-login JWT).

**Prompt** (verbatim; the first three are the options the candidate selected, the fourth is free text):

```text
Deploy scope: Stretch goal, after core (Recommended)
Repos: Two private GitHub repos (Recommended)
Isolation: App layer first, RLS if time allows (Recommended)
Auth: Let's approach something more production grade, let's create a endpoint called POST /auth/api/key where traders sent a secret and based on its validity (hashed stored in the database) it will return an API_KEY and this will be used to authenticate against our services
```

**Outcome:** Deployment was deferred to a stretch goal; two private repos were created; isolation was built in the application layer first with row-level security as a timeboxed add-on (it shipped); the JWT proposal was dropped for the candidate's model: `POST /auth/api/key` verifies a scrypt-hashed secret and issues an opaque `afk_` key whose sha256 is stored, used as Bearer on REST and in the WebSocket handshake.
