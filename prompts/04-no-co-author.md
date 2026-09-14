# 04 — No co-author trailers

**When:** about 10:19, right after the git persona message.

**Context:** Claude Code adds `Co-Authored-By` and session-link trailers to commits by default.

**Prompt** (verbatim; also item 2 of `raw/0n-follow-up.prompt`):

```text
Do not co author yourself
```

**Outcome:** No commit in any of the three repositories carries a co-author or session trailer; the instruction was passed to every sub-session.
