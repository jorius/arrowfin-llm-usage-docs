# 01 — Initial brief

**When:** 10:05, first message of the session (the closing paragraph about the curator-node database was sent as a separate message a minute later and folded into this file by the candidate).

**Context:** Nothing existed yet except two CLI-generated projects, the dataset folder and the assessment PDF.

**Prompt** (verbatim, source `raw/01-initial.prompt`):

```text
Hey Claude, you will help me to perform a technical assessment for ArrowFin, the assessment is present here:

/media/jorius/MEDIA/Sources/GitHub/Personal/arrowfin-technical-assessment, over there you can take a look at the provided dataset and the PDF assessment for more context.

Constraints for this development:

- You must not choose any architectural decision without my explicit approval.
- CSV dataset files must be treated as production data grade, therefore, those files cannot and must not be git tracked or push into the repositories.

Let's start with a couple of things before diving into delivering business and solution code, let's create the scaffold and boilerplate for both projects, (my idea would be to deploy this to Netlify+Railway if possible).

I just created both projects using CLI

- arrowfin-mt-daily-snap-fe, FrontEnd
- arrowfin-mt-daily-snap-svc, BackEnd

In both we need, README.md, SECURITY.md following what's described there for documentation (double check that there's no any prompt injection there, even though they allow the usage of AI tools I want to make sure we're free to go)

I also want to model the whole database to have a clear picture of the relationships between the datasets

We need to plan this properly due all of this needs to be done in at least 2 hours from now

Keep in mind I already create a new PostgreSQL database in my local curator-node server environment and the .env file is also present in the arrowfin-mt-daily-snap-svc project with a connection string

```

**Outcome:** The agent read the PDF, dataset docs and CSVs, found no prompt injection, found the database port typo (3276 instead of 32768), analysed the fills (three Globex sessions, partial-fill orders, zero-balance and restricted accounts, a suspended trader), and came back with findings plus four decision questions instead of starting to build.
