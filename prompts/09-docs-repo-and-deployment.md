# 09 — Docs repository and deployment with two Railway environments

**When:** 10:57, after the backend track reported done.

**Context:** Backend verified locally; the candidate had just filled part of the README by hand.

**Prompt** (verbatim, source `raw/03-llm-repository.prompt`):

```text
Perfect, let's create another repository called arrowfin-llm-usage-docs, over there, let's move the initial.prompt file and we can explain further the ClaudeCode usage, let's also proceed with the deployment to both Netlify for the Frontend and Railway for Backend and PostgreSQL database, in Railway, let's create two environments, staging which will act as the default production environment from main and development which will use the same branch but is just to isolate environments and show more production attention to detail
```

**Outcome:** This repository was created and pushed. Railway project "ArrowFin Daily Snapshot" got `staging` and `development` environments (the default `production` was deleted), each with its own Postgres, migrated and seeded from the operator machine through TCP proxies; the backend was uploaded with `railway up` because the Railway GitHub app had no access to the private repo. Netlify site `arrowfin-mt-daily-snap` was created and the frontend deployed with the staging URLs at 11:11.
