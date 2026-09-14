# Prompts, in the order they were given

One file per instruction. Each file has the moment it was sent, the situation at that point, the
candidate's text quoted verbatim (typos included), and what happened as a result. The candidate's
own raw files are kept unchanged under `raw/`; `raw/0n-follow-up.prompt` bundles instructions 03,
04 and 05 in his consolidated form, and the closing paragraph of `raw/01-initial.prompt` was a
separate message he folded into the initial brief.

| No. | Title | When | Outcome in one line |
| --- | --- | --- | --- |
| [01](./01-initial-brief.md) | Initial brief | 10:05 | Research, injection check, database port typo found, four decision questions |
| [02](./02-decision-answers.md) | Answers to the four decision questions | ~10:18 | Stretch deploy, two private repos, app-layer isolation then RLS, API-key auth model |
| [03](./03-git-persona.md) | Git persona | ~10:19 | Signed commits as Jose Ríos <josed.riosc@gmail.com> |
| [04](./04-no-co-author.md) | No co-author trailers | ~10:19 | No AI trailers on any commit |
| [05](./05-design-approval.md) | Design approval with two changes | ~10:24 | Layering without `domain/`; PDF requirements as named tasks; GPG default key |
| [06](./06-commit-message-style.md) | Commit message style | ~10:31 | "Infinitive verb, capital first letter" |
| [07](./07-readme-and-security-first.md) | README and SECURITY first | ~10:33 | Skeletons first; agent drafts go to `docs/drafts/` |
| [08](./08-frontend-background-and-no-emojis.md) | Frontend Background and no emojis | ~10:48 | Independent Background per repo; all emojis removed |
| [09](./09-docs-repo-and-deployment.md) | Docs repo and deployment | 10:57 | This repo; Railway staging + development; Netlify live 11:11 |
| [10](./10-deployment-linked-status-check.md) | GitHub linked, status check | ~11:13 | Status brief and remaining items on each side |
| [11](./11-ui-iteration.md) | UI iteration | ~11:20 | Brand themes, Spanish, layout, WebSocket settings, responsive; red kept for high risk |
| [12](./12-swagger-and-postman.md) | Swagger and Postman | ~11:26 | OpenAPI at `/docs`, Postman collection with auth smoke flow |
| [13](./13-restructure-prompts-and-gap-brief.md) | Restructure prompts, gap brief | ~11:50 | This folder; PDF gap brief by the main session |

Not recorded as prompts: the shell commands the candidate ran in the session prompt to cache the
GPG passphrase, and the two moments where he confirmed the frontend was working.
