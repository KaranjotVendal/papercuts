# Build brief

Everything needed to implement papercuts from a clean machine. The design is settled; this is an
implementation job, not a design one.

## Read in this order

1. `docs/design.md` — why the tool exists, what counts as a papercut, and which alternatives were
   rejected and why. Read it to understand intent before touching the contract.
2. `docs/spec.md` — the contract. Record shapes, entry states, fold rules, commands, exit codes.
3. `docs/tech-spec.md` — the implementation plan: module layout, invariants, boundary rules,
   test slices.

If something in the spec looks wrong, raise it rather than diverging silently. The spec has already
been through one correction round and is deliberately specific.

## Setup

```bash
git clone https://github.com/KaranjotVendal/papercuts.git
cd papercuts
uv sync
```

Requires `uv` and Python 3.14. If `uv` is missing: `curl -LsSf https://astral.sh/uv/install.sh | sh`.

Install the CLI with `uv tool install --editable .`. Use the installed binary to exercise the tool.
`uv run` is for development tasks only — tests, lint, type check.

## Gates

All four must pass before any work is staged:

```bash
uv run ruff check .
uv run ruff format --check .
uv run ty check
uv run pytest
```

Never hand-edit `uv.lock`. Use `uv add`, `uv remove`, or `uv lock`.

## Conventions

These normally come from a skills directory that is not on this machine, so they are stated here.

- **No module-level docstrings.** The filename and contents convey purpose. Function and class
  docstrings are welcome.
- **`X | None`, never `typing.Optional`.** Built-in generics: `list[str]`, not `typing.List`.
- **`pathlib`, not `os.path`.**
- **f-strings in logging calls**: `logger.warning(f"missing {tag}")`, not `%`-style lazy args.
- **Descriptive names.** No cryptic abbreviations for domain objects. `idx` and `e` in an `except`
  clause are accepted idioms.
- **pydantic by default** for data models. Reach for a dataclass only when it genuinely earns it.
- **`pytest.mark.parametrize` by default** when several tests would share a body and differ only in
  inputs. Do not write round-trip tests for plain pydantic models with no custom validators — that
  tests the framework, not our code.
- **No emojis** anywhere in docs or output.
- The logging module follows the house pattern: file handler at DEBUG, console handler on
  **stderr** defaulting to WARNING via `PAPERCUTS_LOG_LEVEL`, `propagate = False`, idempotent.

## Working agreement

- **One task at a time.** Describe what the task covers, wait for an explicit go-ahead, then
  implement it. Do not batch tasks.
- **Stage, then stop.** When a task's gates are green, `git add` exactly that task's files and stop.
  Do not commit without approval — review happens on the staged diff.
- Commits use the personal email `karanjotgharu60@gmail.com`. Set it repo-locally on this machine
  before the first commit.

## Task plan

Pure core first, so every fold rule is under test before a file or a Typer app exists.

| # | Task | Covers |
|---|---|---|
| 1 | Skeleton and gates — `logging`, `errors`, `config`, `__main__`, `cli` with the `_run` boundary, empty `commands/`, `tests/conftest.py` | Ends green on a CLI that only prints help |
| 2 | Models — `Cut`, `Resolution`, `Severity`, `compute_id`, `normalise_tags`, `normalise_id_prefix`, RFC 3339 serialisation | |
| 3 | Journal — fold with occurrence deduplication, state derivation, query, find, tag counts | Entirely pure; touches no file |
| 4 | Store — `read_records`, `append_record`, locked append, `StoreError`, `LockTimeoutError` | Includes a multi-process flock test |
| 5 | Context — `capture_context`, repo detection in both `.git` forms, home-relative cwd | Needs a real `git worktree add` in the test |
| 6 | Render and `add` — envelope, error envelope, duration parsing, `add_cmd` | Includes the recurrence path end to end |
| 7 | `list` — filters, both formats, empty-is-success | |
| 8 | `resolve` — prefix resolution, batch atomicity | |
| 9 | `tags` | |
| 10 | `schema` and the boundary — contract derived from the app and models, drift test, exit-code mapping, `--json` purity | The drift test is what makes `schema` trustworthy |
| 11 | ast-grep boundary rules and README | Nine rules; drop the "in progress" status |

## Non-negotiables

- **No captured stderr.** `--cmd` and `--exit` only. Storing captured stderr risks writing
  credentials into the log. This is a deliberate divergence from the Rust implementation.
- **`schema` must not drift.** It is derived from the app and the models, not hand-maintained, and
  a test proves it.
- **The clock is injectable.** `PAPERCUTS_NOW` must be honoured everywhere; no module outside
  `context.py` may call `datetime.now()`. An unparseable value is exit 65, never a silent fallback.
- **The store owns file access and locking.** No other module opens the log.
- **Append-only.** Nothing is ever edited or deleted. Resolution and recurrence are records.

## One decision still open

`docs/spec.md` currently says re-filing an **open** cut appends nothing and reports
`changed: false`. Only a recurrence after resolution advances the count.

The argument for changing it: `occurrences` is meant to rank what hurts most, and hitting the same
wall five times in a week while it is still open reads as a single occurrence. The original
objection was double-counting when two machines' logs are merged, and fold rule 3 already solves
that by counting distinct `(id, ts, host)` triples — so appending on every filing is now safe.

**Recommendation:** change the `open`/`recurred` row of the `add` semantics table to append, and let
`occurrences` mean true filing frequency. Confirm before implementing task 6, since it changes
`add_cmd`, the fold, and the schema text.

## Reference

The idea is Steve Ruiz's. `treygoff24/papercuts` is a Rust implementation whose design informed the
`schema` command and the append-only journal model. Reading it is useful; copying from it is not
necessary, and the two tools deliberately differ on evidence capture.
