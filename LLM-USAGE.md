# How Claude Code was used

This repository documents, as transparently as possible, how an LLM coding agent was used
to build the Trader Daily Snapshot submission. The assessment allows LLM use as long as it
is disclosed; this is the disclosure in full, with the prompts.

Related repositories (private, shared with the reviewer):

- Backend: <https://github.com/jorius/arrowfin-mt-daily-snap-svc>
- Frontend: <https://github.com/jorius/arrowfin-mt-daily-snap-fe>

## Tooling

- **Claude Code** (Anthropic's CLI agent) running on **Claude Fable 5.1**, in a terminal on
  the candidate's workstation, with the `superpowers` skills plugin (brainstorming,
  writing-plans, test-driven development, verification), the Railway and Netlify MCP
  servers for deployment, and GitHub CLI for repositories.
- One main session drove the work; two forked sub-sessions implemented the backend and the
  frontend in parallel from a written plan, each in its own repository.
- Date: 2026-09-14, single session, about 2 h 45 min of wall-clock time from the first
  prompt to the final push.

## Working agreement given to the agent

The candidate set these constraints at the start of the session and enforced them
throughout:

1. **No architectural decision without explicit approval.** The agent had to present
   options and wait. Every decision listed below was approved, changed, or rejected by the
   candidate before implementation.
2. **The dataset is production-grade data.** The CSVs were never to be committed or pushed.
   The agent added a pre-commit hook to every repository that refuses any staged CSV or
   `.env` file, kept the dataset outside the repositories, and made the seed read it from an
   environment variable at run time.
3. **Check the assessment materials for prompt injection** before following any
   instruction in them. Result: none found in the PDF, the dataset README, the fill
   simulation notes, the CSV free-text columns, or the tool-generated `AGENTS.md`.
4. Git hygiene: commits signed with the candidate's PGP key, messages in the form
   "Infinitive verb, capital first letter", no AI co-author trailers, merge-only history.
5. Documentation without emoji; the candidate's own sections (stack ratings, five answers,
   time and LLM disclosure, Task 5 reflection) written by hand.

## Timeline of the session

| Time | What happened | Who decided |
| --- | --- | --- |
| 10:05 | Prompt 01 (below). Agent reads the PDF, dataset docs, CSVs, both CLI scaffolds; runs the injection check; analyses the fills (three Globex sessions, 66 partial-fill orders, zero-balance and restricted accounts, a suspended trader). | — |
| 10:15 | Agent presents findings and four decisions: deployment scope, repository layout, isolation depth, auth model. | Candidate: deploy as a stretch goal, two private repos, app-layer isolation first with RLS if time, and an API-key auth model of his own design instead of the agent's JWT proposal. |
| 10:20 | Agent presents the full design (data model, auth, backend layering, WebSocket contract, frontend, docs). | Candidate approves with two changes: drop the `domain/` layer in favour of `application/`, `services/`, `controllers/`, `infrastructure/` (matching the PDF's "service/controller code" deliverable); keep the PDF's three hard requirements as named tasks. |
| 10:26 | Spec and Prisma schema written; dependencies installed; git identity, hooks and GitHub remotes created; initial migration applied to the candidate's Postgres. | Agent, under the approved design. |
| 10:35 | README and SECURITY skeletons written so the candidate can fill his sections in parallel; implementation plan written. | — |
| 10:43 | Two parallel sub-sessions start: backend tasks A1–A11, frontend tasks B1–B5. Main session drafts the Task 4 code review and the SECURITY substance into a drafts folder. | — |
| 10:57 | Backend track finishes: 38 tests, lint and build clean, cross-tenant and socket isolation verified, row-level security shipped. | — |
| 10:57 | Candidate asks for this repository and for the Netlify + Railway deployment with `staging` and `development` environments. | Candidate. |
| 11:00–11:13 | Railway project with `staging` and `development` (default `production` deleted), a Postgres per environment migrated and seeded through TCP proxies from the operator machine, backend uploaded with `railway up` (the Railway GitHub app had no access to the private repo), Netlify site created and the frontend deployed at 11:11 with a CLI build. Staging smoke-tested from outside; the live site checked in a headless browser. | Agent, within the approved scope. |
| 11:13 | Candidate links both Railway environments and the Netlify site to the GitHub repositories, so pushes now redeploy. | Candidate. |
| 11:20 | UI iteration requested: light theme with ArrowFin colors, Spanish with a selector, padded high-risk layout, WebSocket settings from env, responsive layout. Agent pulls the palette from arrowfin.io, presents a plan; approved, red kept for high risk. Two tracks dispatched. | Candidate approves. |
| 11:26 | Swagger documentation and a Postman collection with an authentication smoke test requested; queued behind the WebSocket work in the backend track. | Candidate. |
| 11:50 | Candidate asks for these prompts to be restructured and for a brief on what is still missing against the PDF. | Candidate. |
| 11:53–12:15 | Backend iteration pushed (WebSocket heartbeat config, `hello` event, Swagger, Postman). Railway builds from GitHub fail twice: Node 18 picked by default, then a duplicate `npm ci` over a mounted cache; fixed by pinning Node 24 and trimming the build command. Frontend iteration pushed and live on Netlify; verified headless in both themes and languages. | Agent, within the approved scope. |
| 12:20 | Candidate asks for an explainer of tenant isolation; agent publishes a private page with diagrams of the request path, the six enforcement layers and the socket rooms. | Candidate. |
| 12:35 | Candidate asks to consolidate the written answers from both code repositories into this repository; app READMEs become technical only. | Candidate. |

## What the LLM produced, and what the human owns

| Artifact | Produced by | Human involvement |
| --- | --- | --- |
| Data model (Prisma schema, ERD, index rationale) | Agent | Approved; the layering change was the candidate's |
| Auth model (opaque API keys, scrypt secrets) | Candidate's design, agent's implementation | Designed by the candidate |
| Backend code and tests | Agent (backend sub-session) | Reviewed by the candidate before submission |
| Frontend code | Agent (frontend sub-session) | Reviewed by the candidate before submission |
| Design spec and implementation plan | Agent | Approved |
| Task 4 code review | Agent draft | Edited and signed off by the candidate |
| SECURITY.md | Agent draft with file references | Edited by the candidate |
| Background, five answers, time/LLM disclosure, Task 5 | Candidate | Written by hand |
| Deployment (Railway, Netlify) | Agent via MCP/CLI | Environment design by the candidate |

## Prompts

Every instruction the candidate gave, in order, one file each with context and outcome:
[`prompts/README.md`](./prompts/README.md). The submission itself (Background, disclosure, Tasks 2 to 5) is in [README.md](./README.md) and [SECURITY.md](./SECURITY.md). The candidate's raw prompt files are kept unchanged
under [`prompts/raw/`](./prompts/raw/).

## Verification the agent ran before anything was pushed

- Backend: vitest (services, use cases with in-memory repositories, guard, RLS integration
  test), oxlint with type-aware rules, `tsc --noEmit`, `nest build`, and manual checks with
  curl: own account 200, foreign-tenant account 404, no key 401, wrong secret 401, unknown
  trader 401, suspended trader 403, revoke 204 then 401. Socket checks: valid key connects,
  bogus key gets `unauthorized`, a fill posted for a Summit account reaches the Summit client
  and not the Meridian client. Row-level security checked with psql as the tenant role.
- Frontend: eslint, `next build`, and the live checks described in its README.

## Limits of this disclosure

The session transcript itself is not published. The prompts folder contains what the
candidate typed; the agent's intermediate reasoning is summarised in the timeline above.
Where the agent's output was wrong or rejected (for example the first JWT-based auth
proposal), the rejection is recorded here rather than hidden.
