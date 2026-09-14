# 03 — Git persona

**When:** about 10:19, sent while the agent was checking GPG.

**Context:** The agent had asked nothing about git identity yet; the global git config had a signing key but no name or email.

**Prompt** (verbatim; also item 1 of `raw/0n-follow-up.prompt`):

```text
For the git persona, let's use my default josed.riosc@gmail.com with signed pgp commits
```

**Outcome:** Every repository got a local identity "Jose Ríos <josed.riosc@gmail.com>", `commit.gpgsign=true` and signing key `365602820FC1B86C`; every commit in the three repositories is signed with it.
