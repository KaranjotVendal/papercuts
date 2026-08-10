# papercuts

A complaint box for coding agents.

Agents hit friction constantly — a tool call dead-ends, a doc is wrong, a flag has to be discovered
by experiment — and then work around it silently. The signal evaporates and the next agent hits the
same wall. `papercuts` gives an agent one cheap line to file the complaint at the moment it
happens, and gives you a way to review what keeps recurring and fix it.

```bash
papercuts add "ssh 'moshi-hook servers' fails with 'not found' because ~/.local/bin is absent from the non-interactive PATH; bash -lc fixes it" --tag ssh --tag path
```

The log is a single append-only JSONL file at `~/.papercuts/log.jsonl`. No server, no sync, no
telemetry, nothing committed into your repositories. The file is the product.

## Status

Design, specification, and technical spec are complete. Implementation has not started.

- [docs/design.md](docs/design.md) — why the tool exists and what counts as a papercut
- [docs/spec.md](docs/spec.md) — the contract
- [docs/tech-spec.md](docs/tech-spec.md) — module layout, invariants, and test slices
- [docs/build-brief.md](docs/build-brief.md) — start here to build it

## Install

```bash
uv tool install --editable .
```

## Commands

```
papercuts add "<text>" [--tag x] [--severity minor|major|blocker] [--cmd ...] [--exit N]
papercuts list [--since 7d] [--tag x] [--all] [--format md]
papercuts resolve <id> [--note "..."]
papercuts tags [--since 30d]
papercuts schema
```

`list` shows what is recent. `tags` shows what recurs, which is what tells you what to fix.

## What counts as a papercut

**File one when you worked around something rather than solved it, and the next agent would hit the
same wall.**

In scope: a tool call that dead-ended, a doc or help text that was wrong, an error message that
misled, a flag discovered by experiment, an environment assumption that did not hold.

Out of scope: your own reasoning mistakes, a test that failed because the code was wrong, anything
the user corrected you on, and anything already written down.

## Giving agents the pen

Add to `CLAUDE.md`, `AGENTS.md`, or a skill:

```markdown
When you work around something rather than solve it, and the next agent would hit the same wall,
file it before moving on:

    papercuts add "<what you hit, and what would have prevented it>" --tag <area>

Do not stop working. File it and push through. Run `papercuts schema` if you need the full
contract.
```

## Reviewing

A log nobody reads is worse than no log. Read it on a schedule:

```bash
papercuts tags --since 30d          # what keeps biting
papercuts list --since 7d --format md
papercuts resolve pc_9f2c --note "fixed in the skill"
```

Promote what recurs into durable notes or tickets; resolve the rest.

## Design notes

- **One local file per machine, never in a repository.** Papercuts are notes to ourselves, not
  content for a work repo's code review and history.
- **Append-only journal.** Resolving appends a record rather than editing one, which keeps
  concurrent writers safe and makes merging two machines' logs a concatenate-and-deduplicate.
- **Content-addressed ids.** The same complaint filed twice yields the same id, so duplicates are
  harmless and `add` is idempotent.
- **Every record carries its host**, so logs from two machines can be merged later without having
  planned the merge in advance.
- **No captured stderr.** Attaching stderr risks writing credentials into the log; `--cmd` and
  `--exit` carry the diagnostic value without it.

## Credit

The idea is [Steve Ruiz's](https://x.com/steveruizok/status/2075303919664734295).
[treygoff24/papercuts](https://github.com/treygoff24/papercuts) is a well-built Rust implementation
whose design informed several decisions here, in particular the `schema` command and treating the
log as an append-only journal.

## License

MIT. See [LICENSE](LICENSE).
