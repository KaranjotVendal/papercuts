# Technical Specification

Implementation handoff for the contract in [spec.md](spec.md), reasoned from [design.md](design.md).
Contract version `1`.

This document holds signatures and call stacks, not function bodies. A signature is architecture; a
body is implementation and appears at implementation time.

## Summary

A single-binary Typer CLI over one append-only JSONL file. The file is the only external surface, so
it gets the treatment `client.py` gets in jiracli and bbcli: one boundary module (`store.py`) that
owns the filesystem, the advisory lock, and line parsing, and raises its own domain errors.
Everything above it is pure — records fold into an immutable `Journal` value that answers every
query `list`, `resolve`, and `tags` need, with no I/O in sight. `cli.py` holds the app and the single
error boundary; `commands/` holds one module per verb.

The correctness lever throughout is content-addressing: `id` is derived from `text`, so idempotence,
deduplication, and cross-machine merging are properties of the type rather than rules a caller must
remember.

## Context / Current State

The repository holds the design and the contract, a complete `pyproject.toml` (uv_build, Python 3.14,
typer/rich/pydantic, dev group pytest/ruff/ty), `.python-version`, and an empty `src/papercuts/`.
No code, no tests, no lockfile.

Precedent read before writing this:

- `jiracli/src/jiracli/cli.py` — `app` plus `_run()` as the sole error boundary; Click's `SystemExit`
  passes through untouched; unclassified exceptions surface as tracebacks by design.
- `jiracli/src/jiracli/context.py` — a context helper that assembles what every command needs.
- `bbcli/src/bbcli/{client,config,resolve}.py` — each module defines the named exception it raises
  (`BitbucketError`, `ConfigMissingError`, `ResolutionError`); `cli.py` maps them to exit codes in
  one place.
- `{jiracli,bbcli}/src/*/render.py` — `render_json` uses bare `print` rather than the Rich console,
  so piped output stays valid for `jq`.
- `jiracli/src/jiracli/logging.py` — the house logger, CLI-adapted: stderr console at `WARNING` from
  an env var, file handler at `DEBUG`.

### Rulings folded in

Settled during design review, and reflected in the amended `spec.md`:

1. **Re-filing a resolved cut appends.** Reporting `changed: false` would discard the most valuable
   entry the log can hold: a papercut recurring after it was resolved means the fix did not work.
   The record is appended as a normal `cut`, and the fold surfaces the entry as **recurred** with an
   occurrence count. No `reopen` concept, no new record kind.
2. **Lock timeout is 5.0 seconds**, published in `schema` so it is contract rather than a hidden
   constant.
3. **Errors always emit the JSON error object to stderr**, regardless of `--json`. One clause in the
   boundary, no flag threaded into it, parseable either way, and stderr never pollutes a `jq` pipe.
4. **An invalid `PAPERCUTS_NOW` is `BadInputError`, exit 65.** Silent fallback to the real clock
   would make a broken test harness look green.
5. **Ids are accepted with or without the `pc_` prefix.**
6. **`tags` defaults to open-only**, consistent with `list`, widened by `--all`.
7. **Repo detection handles `.git` as a file as well as a directory**, because most agent work here
   runs in git worktrees, where `.git` is a file. An `is_dir()` check would leave `repo` null in the
   common case.

## Goals

- Implement `add`, `list`, `resolve`, `tags`, `schema` exactly to the amended `spec.md`.
- Concurrent `add` from parallel agent sessions on one machine never corrupts the log.
- A torn final line never fails a read.
- `--json` stdout is always a single valid envelope, safe to pipe into `jq`.
- A papercut that recurs after resolution is visible as a regression, not silently swallowed.
- `papercuts schema` cannot drift from the implemented CLI surface — enforced by a test, not by
  discipline.
- Deterministic tests via `PAPERCUTS_NOW` and `PAPERCUTS_FILE`; no test touches `~/.papercuts`.

## Non-Goals

- Captured stderr on entries. `design.md` rules it out on credential-leak grounds.
- Any network, sync, daemon, or cross-machine read path. Merging is concatenate-and-deduplicate,
  later, by hand.
- Editing or deleting records, or a `reopen` concept.
- A `merge` command. Fold rules already make `cat a b > c` correct; a command can come later if it
  earns its place.

## Invariants

1. `cut.id == "pc_" + sha256(cut.text.strip()).hexdigest()[:12]` for every cut this tool creates.
2. The log file is append-only. No code path opens it for writing without `O_APPEND` and an exclusive
   `flock`.
3. Every appended line is exactly one JSON object plus `\n`, written in a single write call under the
   lock.
4. A line that fails to parse is skipped and logged at `DEBUG`; it never propagates as an error.
5. Tags are lower-cased, stripped, deduplicated, order-preserving.
6. An entry's state is derived, never stored: it is `resolved` when the latest resolution is at or
   after the latest occurrence, `recurred` when an occurrence follows the latest resolution, and
   `open` when there is no resolution at all.
7. Occurrences are counted over distinct `(id, ts, host)` triples, so concatenating a log with itself
   changes nothing and merging two machines' logs counts genuine separate sightings.
8. Timestamps are RFC 3339, UTC, millisecond precision, `Z` suffix — on write and on the wire.
9. Under `--json`, stdout carries the envelope and nothing else. Every log line, warning, and error
   goes to stderr.
10. `fcntl` is imported in exactly one module. `os.environ` is read in exactly one module.
    `datetime.now` is called in exactly one module.

## Design Constraints

- Field names follow the Rust implementation where they overlap, so either tool can read either file.
- Exit codes are fixed: `0` success including empty results, `2` usage (Click's own), `65` bad input,
  `66` not found, `74` I/O, `75` lock timeout.
- Stack is fixed: Python 3.14, uv with the `uv_build` backend, Typer, pydantic, rich. No httpx and no
  keyring here — there is no network and no secret.
- Gates: `uv run ruff check`, `uv run ruff format --check`, `uv run ty check`, `uv run pytest`.

## Alternatives Considered

The axis that matters is where the file boundary sits, and therefore what a command holds and what a
test must fake. All three ship the same CLI surface.

### Option 1: Thin store, fold in the commands

`store.py` exposes `read_records()` and `append_record()`. Each command module folds, filters, and
sorts for itself.

Fewest modules and the shortest call stack. But the fold rules are consumed by `list`, `resolve`, and
`tags` alike, so they get restated three times and drift three ways; and every test of a rule
(severity ordering, prefix ambiguity, recurrence detection) has to go through a real temp file to
reach it. Rejected: it puts domain rules in the presentation layer, which is exactly the layering
jiracli avoids by keeping JQL construction out of `me_cmd`.

### Option 2: Journal value over a file adapter — recommended

`store.py` is a pure I/O adapter: `Path` to `tuple[Record, ...]`, plus locked append. `journal.py` is
a pure, immutable domain layer: `Journal.fold(records)` applies the fold rules once, and `Journal`
answers `query`, `find`, and `tag_counts`. Commands compose read, fold, query, render.

One impure seam, one place per rule, and every rule testable by constructing a `Journal` from
in-memory records with no `tmp_path` at all. The file tests then shrink to what genuinely needs a
file: locking, torn lines, append semantics, a missing file. Costs one extra module over Option 1.

### Option 3: `PapercutsLog` repository object

A class holding the path, with `read()`, `append()`, `open_cuts()`, `resolve_prefix()`, injected into
commands by a `context.py` helper in jiracli's `get_client_and_config` shape.

Familiar, and gives commands one dependency. But it fuses I/O with rules in a single type, so testing
prefix ambiguity means either a real file or a fake of our own class — the low-value patch target the
standards rule out. It also earns its keep only when there is per-instance state to carry: jiracli's
client object exists because there is a live `httpx.Client` to own. Here there is a path and nothing
else. Rejected as a seam without an invariant to hold.

## Recommendation

Option 2. The file is a boundary; the fold rules are a domain. Separating them buys a pure,
exhaustively testable core and confines `fcntl`, `open`, and torn-line tolerance to one auditable
module — which is also the module that owns `StoreError` and `LockTimeoutError`.

## Proposed Design

```
__main__.py   main() -> cli._run()
cli.py        Typer app; registers one command per verb; the single error boundary
commands/     add_cmd, list_cmd, resolve_cmd, tags_cmd, schema_cmd, _shared
context.py    ambient capture: clock, host, user, cwd, repo  -> RecordContext
store.py      BOUNDARY: filesystem + flock + line parsing; StoreError, LockTimeoutError
journal.py    pure domain: fold rules, state derivation, filtering, ordering, prefixes, tag counts
models.py     Cut, Resolution, Severity, content-addressed id, RFC 3339 serialization
duration.py   "1d12h" -> timedelta
config.py     Settings, all env reads, all tunable constants
render.py     the {ok, data, meta} envelope, tables, markdown digest
errors.py     PapercutsError taxonomy carrying its own wire code and exit code
logging.py    initialise_logger, house pattern, PAPERCUTS_LOG_LEVEL
```

Layering rule: `commands/` may import everything; `journal.py` and `models.py` import nothing above
`errors` and `config`; `store.py` is the only importer of `fcntl`; `config.py` is the only reader of
the environment; `context.py` is the only caller of the clock and the hostname.

## Domain Model and Types

```python
# models.py
from __future__ import annotations

from collections.abc import Sequence
from datetime import datetime
from enum import StrEnum
from typing import Annotated, Literal, Self

from pydantic import BaseModel, Field, TypeAdapter, field_serializer, field_validator


class Severity(StrEnum):
    MINOR = "minor"
    MAJOR = "major"
    BLOCKER = "blocker"

    @property
    def rank(self) -> int:
        """Sort key for default ordering: blocker(0) < major(1) < minor(2)."""
        ...


def compute_id(text: str) -> str:
    """Return 'pc_' plus the first ID_HEX_LENGTH hex characters of SHA-256 over the trimmed text."""
    ...


def normalise_id_prefix(raw: str) -> str:
    """Strip a leading 'pc_' so a prefix typed either way matches the same ids."""
    ...


def normalise_tags(tags: Sequence[str]) -> tuple[str, ...]:
    """Lower-case, strip, drop empties, deduplicate, preserve first-seen order."""
    ...


class Cut(BaseModel, frozen=True):
    kind: Literal["cut"] = "cut"
    id: str
    ts: datetime
    text: str
    tags: tuple[str, ...] = ()
    severity: Severity = Severity.MINOR
    agent: str | None = None
    user: str | None = None
    host: str | None = None
    repo: str | None = None
    cwd: str | None = None
    cmd: str | None = None
    exit_code: int | None = None

    @classmethod
    def create(
        cls,
        text: str,
        context: RecordContext,
        tags: Sequence[str] = (),
        severity: Severity = Severity.MINOR,
        cmd: str | None = None,
        exit_code: int | None = None,
    ) -> Self:
        """Build a cut with id derived from text and identity fields taken from the context.

        The only sanctioned construction path for a new cut: the id cannot disagree with the
        text, because the caller never supplies it.
        """
        ...

    @field_validator("text")
    @classmethod
    def _reject_blank_text(cls, value: str) -> str: ...

    @field_serializer("ts")
    def _serialise_ts(self, value: datetime) -> str: ...


class Resolution(BaseModel, frozen=True):
    kind: Literal["resolution"] = "resolution"
    id: str
    ts: datetime
    agent: str | None = None
    host: str | None = None
    note: str | None = None

    @field_serializer("ts")
    def _serialise_ts(self, value: datetime) -> str: ...


Record = Annotated[Cut | Resolution, Field(discriminator="kind")]
RECORD_ADAPTER: TypeAdapter[Record] = TypeAdapter(Record)
```

Parsing versus creating: `Cut.create` computes the id, so this tool can never write a cut whose id
disagrees with its text. Parsing a file trusts the stored id, because the file may have been written
by the Rust implementation, and the fold rule says skip what does not parse — not rewrite what
disagrees.

```python
# journal.py
from __future__ import annotations

from collections.abc import Iterable
from datetime import datetime, timedelta
from enum import StrEnum
from typing import Self

from pydantic import BaseModel


class CutState(StrEnum):
    OPEN = "open"
    RESOLVED = "resolved"
    RECURRED = "recurred"


class CutView(BaseModel, frozen=True):
    """One folded entry: the canonical cut plus everything derived from repeats and resolutions."""

    cut: Cut
    occurrences: int
    first_seen: datetime
    last_seen: datetime
    resolutions: tuple[Resolution, ...] = ()

    @property
    def state(self) -> CutState:
        """OPEN with no resolution; RECURRED when last_seen postdates the latest resolution;
        RESOLVED otherwise."""
        ...

    @property
    def is_open(self) -> bool:
        """True for OPEN and RECURRED — a recurred cut is unfinished business, not history."""
        ...

    @property
    def resolved_at(self) -> datetime | None: ...


class ListFilters(BaseModel, frozen=True):
    since: timedelta | None = None
    tag: str | None = None
    repo: str | None = None
    host: str | None = None
    severity: Severity | None = None
    include_resolved: bool = False


class TagCount(BaseModel, frozen=True):
    tag: str
    count: int


class Journal(BaseModel, frozen=True):
    views: tuple[CutView, ...]

    @classmethod
    def fold(cls, records: Iterable[Record]) -> Self:
        """Apply the fold rules: first cut per id supplies canonical fields, repeats accumulate
        into occurrences and last_seen, orphan resolutions are ignored."""
        ...

    def get(self, cut_id: str) -> CutView | None:
        """Exact-id lookup, used by add to decide append versus no-op."""
        ...

    def find(self, prefix: str) -> CutView:
        """Resolve a unique id prefix, given with or without 'pc_'.

        Raises AmbiguousIdError on more than one match, IdNotFoundError on none.
        """
        ...

    def query(self, filters: ListFilters, now: datetime) -> tuple[CutView, ...]:
        """Filter, then order by severity rank and then last_seen, newest first."""
        ...

    def tag_counts(
        self, since: timedelta | None, include_resolved: bool, now: datetime
    ) -> tuple[TagCount, ...]:
        """Counts descending, ties broken alphabetically so output is stable."""
        ...
```

Occurrence counting is deduplicated on `(id, ts, host)`. Concatenating a log with itself therefore
changes nothing, while two machines that each hit the same wall count as two sightings — which is the
signal the review ritual is looking for.

## Types, Interfaces, and APIs

```python
# errors.py
from __future__ import annotations

from typing import ClassVar


class PapercutsError(Exception):
    """Base for every expected failure, carrying its own wire code and exit code.

    The error boundary needs one clause, and a new failure mode cannot be added without
    declaring how it exits.
    """

    code: ClassVar[str]
    exit_code: ClassVar[int]


class BadInputError(PapercutsError):
    code: ClassVar[str] = "bad_input"
    exit_code: ClassVar[int] = 65


class AmbiguousIdError(BadInputError):
    code: ClassVar[str] = "ambiguous_id"


class IdNotFoundError(PapercutsError):
    code: ClassVar[str] = "not_found"
    exit_code: ClassVar[int] = 66
```

```python
# store.py — the external surface, owning its domain errors
from __future__ import annotations

from collections.abc import Iterator
from contextlib import contextmanager
from pathlib import Path
from typing import IO, ClassVar


class StoreError(PapercutsError):
    """Raised when the log cannot be read or written."""

    code: ClassVar[str] = "io_error"
    exit_code: ClassVar[int] = 74


class LockTimeoutError(StoreError):
    """Raised when the advisory lock is not acquired within LOCK_TIMEOUT_SECONDS."""

    code: ClassVar[str] = "lock_timeout"
    exit_code: ClassVar[int] = 75


def read_records(path: Path) -> tuple[Record, ...]:
    """Parse every line of the log, skipping any line that fails.

    A missing file yields an empty tuple — an unused log is not an error. Raises StoreError only
    on genuine I/O failure, such as an unreadable path or a directory where a file belongs.
    """
    ...


def append_record(path: Path, record: Record, timeout: float = LOCK_TIMEOUT_SECONDS) -> None:
    """Append one JSON line under an exclusive advisory lock.

    Creates parent directories on first write. Raises LockTimeoutError on timeout and StoreError
    on any other I/O failure.
    """
    ...


@contextmanager
def _locked_append(path: Path, timeout: float) -> Iterator[IO[str]]:
    """Open O_APPEND and hold LOCK_EX for the body, polling until the timeout elapses."""
    ...
```

```python
# config.py — the only module that reads the environment
from __future__ import annotations

from collections.abc import Mapping
from datetime import datetime
from pathlib import Path

from pydantic import BaseModel

CONTRACT_VERSION: int = 1
ID_PREFIX: str = "pc_"
ID_HEX_LENGTH: int = 12
DEFAULT_LOG_PATH: Path = Path.home() / ".papercuts" / "log.jsonl"
LOCK_TIMEOUT_SECONDS: float = 5.0
LOCK_POLL_INTERVAL_SECONDS: float = 0.05
STDIN_SENTINEL: str = "-"

ENV_FILE: str = "PAPERCUTS_FILE"
ENV_AGENT: str = "PAPERCUTS_AGENT"
ENV_NOW: str = "PAPERCUTS_NOW"
ENV_LOG_LEVEL: str = "PAPERCUTS_LOG_LEVEL"


class Settings(BaseModel, frozen=True):
    log_file: Path
    agent: str | None = None
    frozen_now: datetime | None = None


def load_settings(env: Mapping[str, str] | None = None) -> Settings:
    """Read the environment once, defaulting to os.environ.

    Raises BadInputError when PAPERCUTS_NOW is set but is not RFC 3339: a silently ignored
    frozen clock would make a broken test harness look green.
    """
    ...
```

```python
# context.py — the only caller of the clock, hostname, user, and cwd
from __future__ import annotations

from datetime import datetime
from pathlib import Path

from pydantic import BaseModel


class RecordContext(BaseModel, frozen=True):
    ts: datetime
    agent: str | None = None
    user: str | None = None
    host: str | None = None
    repo: str | None = None
    cwd: str | None = None


def capture_context(settings: Settings, agent: str | None = None) -> RecordContext:
    """Assemble the ambient fields every record carries.

    ts is settings.frozen_now when set, otherwise datetime.now(UTC). agent is the --agent flag,
    otherwise PAPERCUTS_AGENT. Never raises: an undiscoverable field is None, because failing to
    name the host must not block filing a papercut.
    """
    ...


def detect_repo(start: Path) -> str | None:
    """Name of the repository containing `start`, or None outside a repository.

    Walks parents with pathlib for a `.git` entry and handles both forms:

    - `.git` is a DIRECTORY (ordinary clone): the repository name is that directory's parent.
    - `.git` is a FILE (git worktree, submodule): it holds `gitdir: <path>`, pointing at
      `<repo>/.git/worktrees/<name>`. The repository name is the parent of the `.git` component
      of that path, so every worktree of a repository reports the same repo and `--repo` groups
      them. An unparseable or unexpected pointer falls back to the containing directory's name.

    Never shells out to git, so this works in a worktree, in a bare checkout, and on a machine
    without git installed.
    """
    ...


def home_relative(path: Path) -> str:
    """Render a path as ~/... when under home, otherwise absolute."""
    ...
```

The worktree case is the common one on this setup, not an edge case: agent streams run in worktrees,
where `.git` is a file. An `is_dir()` check would leave `repo` null for most entries and quietly
break `--repo` filtering.

```python
# render.py
from __future__ import annotations

from collections.abc import Sequence

from pydantic import BaseModel


class Meta(BaseModel, frozen=True):
    contract: int
    file: str
    host: str | None


def render_envelope(data: object, meta: Meta) -> None:
    """Write {"ok": true, "data": ..., "meta": ...} to stdout as one JSON object.

    Uses bare print rather than the Rich console: Rich can inject ANSI and soft wrapping that
    corrupts a jq pipe.
    """
    ...


def render_error(error: PapercutsError) -> None:
    """Write {"ok": false, "error": {"code", "message"}} to stderr.

    Emitted for every failure whether or not --json was passed, so an agent gets a parseable
    failure either way and the boundary needs no knowledge of output mode.
    """
    ...


def render_cut_table(views: Sequence[CutView]) -> None: ...
def render_cut_markdown(views: Sequence[CutView]) -> None: ...
def render_tag_table(counts: Sequence[TagCount]) -> None: ...
def render_add_result(result: AddResult) -> None: ...
def render_resolve_results(results: Sequence[ResolveOutcome]) -> None: ...
```

```python
# duration.py
def parse_duration(text: str) -> timedelta:
    """Parse '30m', '24h', '7d', '1d12h' into a timedelta.

    Raises BadInputError on an unparseable or zero-length duration.
    """
    ...
```

```python
# commands/_shared.py
class AddResult(BaseModel, frozen=True):
    id: str
    changed: bool
    state: CutState
    occurrences: int


class ResolveOutcome(BaseModel, frozen=True):
    id: str
    changed: bool
    state: CutState


def open_journal(settings: Settings) -> Journal:
    """read_records then Journal.fold — the one composition every read command shares."""
    ...


def build_meta(settings: Settings, context: RecordContext) -> Meta: ...


def read_text_argument(text: str) -> str:
    """Return text, or read stdin when text is the '-' sentinel."""
    ...
```

```python
# commands/* — Typer callbacks, one module per verb
def add_command(
    text: str = typer.Argument(...),
    tag: list[str] = typer.Option([], "--tag"),
    severity: Severity = typer.Option(Severity.MINOR, "--severity"),
    agent: str | None = typer.Option(None, "--agent"),
    cmd: str | None = typer.Option(None, "--cmd"),
    exit_code: int | None = typer.Option(None, "--exit"),
    as_json: bool = typer.Option(False, "--json"),
) -> None: ...


def list_command(
    since: str | None = typer.Option(None, "--since"),
    tag: str | None = typer.Option(None, "--tag"),
    repo: str | None = typer.Option(None, "--repo"),
    host: str | None = typer.Option(None, "--host"),
    severity: Severity | None = typer.Option(None, "--severity"),
    show_all: bool = typer.Option(False, "--all"),
    output_format: ListFormat = typer.Option(ListFormat.TABLE, "--format"),
    as_json: bool = typer.Option(False, "--json"),
) -> None: ...


def resolve_command(
    ids: list[str] = typer.Argument(...),
    note: str | None = typer.Option(None, "--note"),
    agent: str | None = typer.Option(None, "--agent"),
    as_json: bool = typer.Option(False, "--json"),
) -> None: ...


def tags_command(
    since: str | None = typer.Option(None, "--since"),
    show_all: bool = typer.Option(False, "--all"),
    as_json: bool = typer.Option(False, "--json"),
) -> None: ...


def schema_command() -> None:
    """Emit the full machine contract as JSON. Always JSON; --json is accepted and ignored."""
    ...
```

```python
# cli.py
app = typer.Typer(
    no_args_is_help=True,
    add_completion=False,
    help="A complaint box for coding agents.",
)
app.command(name="add")(add_cmd.add_command)
app.command(name="list")(list_cmd.list_command)
app.command(name="resolve")(resolve_cmd.resolve_command)
app.command(name="tags")(tags_cmd.tags_command)
app.command(name="schema")(schema_cmd.schema_command)


def _run() -> None:
    """Entry point holding the single error boundary.

    Click runs in standalone mode and raises SystemExit for its own control flow (0 on success,
    2 on usage), which passes straight through. Every expected failure is a PapercutsError
    carrying its own exit code, so this is one clause rather than a growing chain. Anything else
    is a defect and surfaces as a traceback, by design.
    """
    ...
```

## Seams, Boundaries, Adapters, and Implementations

| Seam | Module | What crosses it | What must not leak |
|---|---|---|---|
| Filesystem and advisory lock | `store.py` | `Path` in, `tuple[Record, ...]` out; one `Record` in, nothing out | `fcntl`, file handles, `JSONDecodeError`, `OSError` — all translated to `StoreError` or `LockTimeoutError` |
| Environment | `config.py` | `Mapping[str, str]` in, `Settings` out | Raw env strings, `os.environ` access |
| Clock, hostname, user, cwd, repository layout | `context.py` | `Settings` in, `RecordContext` out | `datetime.now`, `socket`, `getpass`, `Path.cwd` |
| Terminal | `render.py` | Domain values in, bytes on stdout and stderr | Rich objects, `print` |
| Argument parsing | `commands/` | Click primitives in, domain types out | `typer` and `click` types below `commands/` |

`store.py` is the analogue of jiracli's `client.py`: the one module that touches the outside world,
and the one that names the failure. Test doubles are permitted only at these five seams. Nothing
inside `journal.py`, `models.py`, or `duration.py` is ever patched, because all three are pure and
can simply be called.

## Call Stacks and Data Flow

### `papercuts add "..." --tag ssh --exit 127`

```txt
__main__.main
  -> cli._run
    -> typer app  (Click parses argv; a usage failure exits 2 here)
      -> commands.add_cmd.add_command
        -> config.load_settings(os.environ)              -> Settings
        -> commands._shared.read_text_argument           -> str        (stdin when "-")
        -> context.capture_context(settings, agent)      -> RecordContext
          -> context.detect_repo(Path.cwd())             -> str | None
        -> models.Cut.create(text, context, tags, ...)   -> Cut        (id derived here)
        -> commands._shared.open_journal(settings)
          -> store.read_records(settings.log_file)       -> tuple[Record, ...]
          -> journal.Journal.fold                        -> Journal
        -> journal.Journal.get(cut.id)                   -> CutView | None
        -- absent, or present with state RESOLVED:
        -> store.append_record(settings.log_file, cut)   -> None       [SIDE EFFECT]
        -- present and open or recurred: no append
        -> render.render_envelope | render.render_add_result
```

Type and data-flow trace:

```txt
argv (str)
  -> Click parameters (str, list[str], int | None)
  -> read_text_argument / normalise_tags / Severity      (parse at the boundary)
  -> Cut  (frozen, id == f(text))
  -> RECORD_ADAPTER.dump_json  -> one JSONL line
  -> flock(LOCK_EX) + O_APPEND write
  -> AddResult{id, changed, state, occurrences} -> envelope | human line
```

Three outcomes:

| Existing entry | Appends | `changed` | Resulting state |
|---|---|---|---|
| none | yes | true | `open` |
| open or recurred | no | false | unchanged |
| resolved | yes | true | `recurred` |

The middle row keeps the log quiet while a wall is already flagged. The bottom row is the regression
signal: a resolved papercut filed again means the fix did not hold, and that is worth more than the
original report.

### `papercuts list --since 7d --format md`

```txt
cli._run -> list_cmd.list_command
  -> config.load_settings                       -> Settings
  -> duration.parse_duration("7d")              -> timedelta      (BadInputError -> 65)
  -> context.capture_context                    -> RecordContext  (supplies now and meta.host)
  -> _shared.open_journal
    -> store.read_records                       -> tuple[Record, ...]   (bad lines skipped)
    -> Journal.fold                             -> Journal              (occurrences, resolutions)
  -> Journal.query(ListFilters, now)            -> tuple[CutView, ...]  (severity, then last_seen)
  -> render.render_cut_markdown | render_cut_table | render_envelope
```

The default view is open plus recurred. `--all` adds resolved. An empty tuple renders an empty table,
or `{"ok": true, "data": {"cuts": []}, ...}`, and exits `0`.

### `papercuts resolve 9f2c 8a1b --note "..."`

```txt
cli._run -> resolve_cmd.resolve_command
  -> load_settings; capture_context
  -> _shared.open_journal                       -> Journal
  -> for each id argument:
     -> Journal.find(normalise_id_prefix(arg))  -> CutView
        |  0 matches  -> IdNotFoundError   (66)
        |  2+ matches -> AmbiguousIdError  (65)
  -- every argument resolved before the first append
  -> for each resolved view where view.is_open:
     -> Resolution(id=view.cut.id, ts=context.ts, agent=..., host=..., note=...)
     -> store.append_record                     -> None   [SIDE EFFECT]
  -- already resolved: changed=False, no append
  -> render.render_resolve_results | render_envelope
```

Resolving all arguments before appending any makes a batch containing one bad prefix fail without
half-applying. A recurred entry resolves normally: the new resolution postdates the latest
occurrence, so the state returns to `resolved`.

### `papercuts tags --since 30d`

```txt
cli._run -> tags_cmd.tags_command
  -> load_settings; parse_duration; capture_context
  -> _shared.open_journal -> Journal.tag_counts(since, include_resolved, now)
  -> tuple[TagCount, ...] -> render_tag_table | render_envelope
```

### `papercuts schema`

```txt
cli._run -> schema_cmd.schema_command
  -> schema_cmd.build_contract(app)             -> ContractDocument
     -- commands and flags derived by walking the registered Typer app
     -- record shapes from Cut.model_json_schema / Resolution.model_json_schema
     -- states from CutState, env vars and LOCK_TIMEOUT_SECONDS from config constants
     -- exit codes from the PapercutsError subclasses
  -> render_envelope
```

Deriving rather than transcribing is what keeps `schema` honest; the drift test below closes the
remaining gap.

### Failure Flow

```txt
any command
  -> raises a PapercutsError subclass
    -> propagates through the Typer callback (Click does not catch it)
      -> cli._run
        -> render.render_error(error)      -> stderr: {"ok": false, "error": {...}}
        -> raise SystemExit(error.exit_code)
```

| Raised by | Exception | `code` | Exit |
|---|---|---|---|
| Click | `UsageError` to `SystemExit` | — | 2 |
| `parse_duration`, `load_settings` | `BadInputError` | `bad_input` | 65 |
| `Journal.find` | `AmbiguousIdError` | `ambiguous_id` | 65 |
| `Journal.find` | `IdNotFoundError` | `not_found` | 66 |
| `store.read_records`, `store.append_record` | `StoreError` | `io_error` | 74 |
| `store.append_record` | `LockTimeoutError` | `lock_timeout` | 75 |

### Retry / Cancellation / Idempotency Flow

```txt
two concurrent `add` of the same text, sessions A and B
  A: read -> fold -> get(id) -> None -> append_record
       flock(LOCK_EX) [held] -> write one line -> unlock
  B: read -> fold -> get(id) -> None -> append_record
       flock(LOCK_EX) [blocks until A releases] -> write one line -> unlock

result: two `cut` lines with one id, differing in ts
read:   fold treats them as two occurrences of one entry
effect: one entry, occurrences 2, state open
```

The check-then-append race is real and deliberately unguarded: content-addressing makes the losing
branch harmless, so holding the lock across read and write would buy nothing and cost contention.
Two sightings a second apart are a marginally noisier count, not a corrupt log.

Idempotence of merging: concatenating a log with itself produces identical `(id, ts, host)` triples,
which fold to one occurrence each, so the merged journal equals the original.

`LockTimeoutError` at exit `75` is documented as retryable, and the tool does not retry internally —
an agent filing a papercut must never block on another session.

### Observability Flow

```txt
initialise_logger(__name__)
  -> file handler   DEBUG   -> ~/.papercuts/logs/papercuts.log
  -> stderr handler WARNING (PAPERCUTS_LOG_LEVEL overrides)

store.read_records    DEBUG   f"skipping unparseable line {line_number} in {path}"
store._locked_append  DEBUG   f"waiting for lock on {path}"
store.append_record   WARNING f"lock on {path} not acquired within {timeout}s"
context.detect_repo   DEBUG   f"repo not detected from {start}"
add_cmd.add_command   INFO    f"cut {cut_id} recurred after resolution at {resolved_at}"
```

Log lines name paths, counts, ids, and line numbers, never cut text — which may quote a failing
command. Nothing is logged to stdout, so `--json` stays pipe-clean at any log level.

## Files to Add / Change / Delete

| File | Status | Owns |
|---|---|---|
| `src/papercuts/__main__.py` | add | `main()` delegating to `cli._run` |
| `src/papercuts/cli.py` | add | Typer app, command registration, the single error boundary |
| `src/papercuts/errors.py` | add | `PapercutsError` taxonomy with wire codes and exit codes |
| `src/papercuts/config.py` | add | `Settings`, all environment reads, all constants |
| `src/papercuts/logging.py` | add | `initialise_logger`, CLI-adapted house pattern |
| `src/papercuts/models.py` | add | `Cut`, `Resolution`, `Severity`, id derivation, RFC 3339 serialization |
| `src/papercuts/journal.py` | add | Fold rules, state derivation, filtering, ordering, prefixes, tag counts |
| `src/papercuts/store.py` | add | Filesystem boundary, `flock`, tolerant read, `StoreError`, `LockTimeoutError` |
| `src/papercuts/context.py` | add | Clock, host, user, cwd, repo detection including the worktree form |
| `src/papercuts/duration.py` | add | `parse_duration` |
| `src/papercuts/render.py` | add | Envelope, error envelope, tables, markdown digest |
| `src/papercuts/commands/{__init__,_shared,add_cmd,list_cmd,resolve_cmd,tags_cmd,schema_cmd}.py` | add | One module per verb, plus shared composition helpers |
| `tests/{__init__,conftest}.py` | add | `papercuts_env` fixture (`PAPERCUTS_FILE` under `tmp_path`, frozen `PAPERCUTS_NOW`) and a record factory |
| `tests/test_{models,journal,duration,store,context,render}.py` | add | Unit coverage per module |
| `tests/test_cmd_{add,list,resolve,tags,schema}.py` | add | End to end through `CliRunner` |
| `tests/test_cli_errors.py` | add | Exception-to-exit-code mapping |
| `docs/spec.md` | change | Amended fold rules, entry states, `add` semantics, lock timeout, error output |
| `pyproject.toml` | change | Per-file ignores if needed; confirm `ty` include covers `src` only |
| `uv.lock` | add | Generated by `uv sync` |
| `README.md` | change | Drop the "implementation in progress" status once shipped |

## RGR TDD Test Plan

Vertical slices, ordered. Each is one failing behavior, then the minimum to pass it. No test bodies
here.

1. **`compute_id` is stable and content-addressed.** The same trimmed text always yields the same
   `pc_`-prefixed twelve-hex id; different text does not. Seam: pure function. Oracle: an
   independently computed SHA-256 digest, not a recomputation the same way.
2. **`Cut.create` derives id from text and adopts context identity.** No id from the caller;
   `host`, `user`, `repo`, `ts` come from the `RecordContext`.
3. **Tag normalisation.** Parametrized over mixed case, surrounding whitespace, duplicates, empties.
4. **RFC 3339 round trip.** A serialized `Cut` carries `ts` as `...Z` at millisecond precision, and
   re-parsing yields an equal model. Earns its place because `_serialise_ts` is custom serialization,
   not plain pydantic field shape.
5. **Discriminated parsing.** `kind: "cut"` parses to `Cut`, `kind: "resolution"` to `Resolution`,
   an unknown `kind` raises. Exercises our `Field(discriminator=...)` wiring.
6. **`parse_duration`.** Parametrized accepted (`30m`, `24h`, `7d`, `1d12h`) and rejected (`""`, `7`,
   `d`, `0m`, `1x`) shapes.
7. **Fold: occurrences and orphans.** Repeated cut ids fold into one entry whose canonical fields come
   from the first record, with `occurrences` and `last_seen` advanced by later ones; a resolution
   matching no cut is ignored. Seam: `Journal.fold` on in-memory records, no file.
8. **Fold: occurrence deduplication.** Records identical in `(id, ts, host)` count once, so folding a
   record sequence concatenated with itself equals folding it once. This is the property that makes
   merging two machines' logs safe.
9. **State derivation.** Parametrized: no resolution gives `open`; a resolution after the last
   occurrence gives `resolved`; an occurrence after the latest resolution gives `recurred`; resolving
   a recurred entry returns it to `resolved`.
10. **Ordering.** Blocker before major before minor, and within a severity newest `last_seen` first.
    Oracle: a hand-ordered expected sequence.
11. **`ListFilters`.** Parametrized over `since`, `tag`, `repo`, `host`, `severity`, and
    `include_resolved`. Each narrows as specified, combinations intersect, and recurred entries
    appear by default while resolved ones need `--all`.
12. **Prefix resolution.** A unique prefix resolves with or without `pc_`; an ambiguous prefix raises
    `AmbiguousIdError`; an unmatched one raises `IdNotFoundError`.
13. **Tag counts.** Descending by count, alphabetical on ties, honouring `since` and `--all`.
14. **Store read tolerance.** A log whose final line is truncated mid-JSON yields every complete
    record and no error; a missing file yields empty. Seam: `tmp_path` — a real file, since torn-line
    tolerance is exactly what a fake would hide.
15. **Store append.** Appending to a nonexistent path creates parents and writes one
    newline-terminated line; appending twice preserves both in order. Asserts on file bytes.
16. **Concurrent append integrity.** Several processes appending at once produce that many complete,
    individually parseable lines with no interleaving. Seam: real subprocesses over one `tmp_path`
    file — the only honest test of `flock`.
17. **Repo detection, ordinary clone.** A `tmp_path` directory containing a `.git` directory, and a
    nested subdirectory within it, both report the repository directory's name; a directory outside
    any repository reports `None`.
18. **Repo detection, git worktree.** Build a real repository and a real `git worktree add` under
    `tmp_path`, then assert that a path inside the worktree — where `.git` is a file — reports the
    repository name rather than `None`. Uses real git in the test, never in `src`.
19. **`add` end to end.** A fresh file gains one cut with the expected id and `changed: true`; the
    `--json` envelope carries `ok`, `data`, `meta.contract`, `meta.file`. Seam: `CliRunner` with
    `PAPERCUTS_FILE` and `PAPERCUTS_NOW`.
20. **`add` is quiet while a cut is open.** Filing identical text twice appends nothing the second
    time and reports `changed: false`; the file still holds one line.
21. **`add` after resolution records a recurrence.** Add, resolve, then add the same text again: the
    file gains a second cut line, `changed` is true, the reported state is `recurred` with
    `occurrences` 2, and the entry is back in the default `list`.
22. **`add` from stdin.** Text `-` reads stdin.
23. **`list` empty is success.** An empty result prints an empty table and exits `0`; under `--json`,
    `data.cuts` is `[]`.
24. **`list --format md`.** A markdown digest containing each entry's id, severity, state, and text.
25. **`resolve` end to end.** A resolution line is appended, the cut leaves the default `list`, and
    re-resolving reports `changed: false` and appends nothing further.
26. **`resolve` batch atomicity.** A batch whose second prefix matches nothing exits `66` and appends
    nothing at all.
27. **Exit-code mapping.** Parametrized over each `PapercutsError` subclass raised through a command:
    the exit code and the stderr `code` field match the table, and the error object is emitted
    whether or not `--json` was passed. Plus: an unknown flag exits `2`.
28. **`--json` purity.** With `PAPERCUTS_LOG_LEVEL=DEBUG`, stdout still parses as exactly one JSON
    object. This is the invariant that protects every agent's `jq` pipe.
29. **`schema` is not stale.** Every command registered on the Typer app appears in the schema payload
    with the same flag names, and every command in the payload exists on the app; the declared exit
    codes match the `PapercutsError` subclasses, and the published lock timeout matches
    `LOCK_TIMEOUT_SECONDS`. This is the drift guard that makes `schema` trustworthy.

Not tested, deliberately: plain pydantic field shapes, which `ty` covers; Rich table styling; and the
literal hostname from `capture_context`, asserted as present and a string rather than equal to this
machine's name — a test that hardcodes the host fails on the other machine.

## Boundary Rules (ast-grep candidates)

| Rule | Forbids | Enforces |
|---|---|---|
| `store-owns-flock` | `import fcntl` outside `store.py` | Invariant 2: one module holds the locking discipline |
| `store-owns-the-log` | `open(...)`, `Path.write_text`, `Path.open` outside `store.py` and `logging.py` | The append-only guarantee cannot be bypassed |
| `config-owns-env` | `os.environ`, `os.getenv` outside `config.py` | A single composition root for configuration |
| `context-owns-the-clock` | `datetime.now(...)`, `time.time()` outside `context.py` | `PAPERCUTS_NOW` determinism; no unfrozen timestamp can sneak in |
| `no-print-in-modules` | `print(` outside `render.py`, with the house `ast-grep-ignore` on the two envelope writers | Invariant 9: stdout has exactly one writer |
| `no-bare-logging` | `import logging` outside `logging.py` | The house logging pattern |
| `no-subprocess` | `import subprocess`, `os.system` anywhere under `src/` | Repo detection stays pathlib-only, with no shell or git dependency |
| `no-typing-optional` | `Optional[` anywhere | `X \| None` house style |
| `no-typer-below-commands` | `import typer` outside `cli.py` and `commands/` | Domain layers stay framework-free and directly callable in tests |

The first four are the ones a signature cannot express: a helper in `journal.py` that calls
`datetime.now()` type-checks perfectly and silently breaks every deterministic test.

## Risks and Open Questions

1. **`add` stays quiet while a cut is open.** The recurrence ruling covers re-filing a *resolved*
   cut. Re-filing an *open* one keeps the original contract: nothing is appended, `changed: false`.
   The joint reading is that `occurrences` counts one initial sighting plus each post-resolution
   recurrence, which is the number the review ritual actually wants. Flagged because the alternative
   — appending every re-filing — would make `occurrences` a raw frequency instead, and that is a
   different, defensible number.
2. **No `--state` filter yet.** Recurred entries are visible by default and carry a state column, but
   there is no way to ask for only them. If the review ritual wants "show me what came back", that is
   a small later addition, not a change to the fold.
3. **`flock` is POSIX-only**, so there is no Windows support. Both machines are macOS or Linux;
   recorded rather than abstracted, since a portability layer for a platform nobody uses would be
   speculative.
4. **Fold cost is linear in the whole file** for every command. At tens to hundreds of lines a month
   this is irrelevant, and an index would be a second source of truth. A deliberate
   non-optimisation.
5. **Occurrence counts across merged logs are approximate** if a log is ever edited by hand, since
   deduplication keys on `(id, ts, host)`. Hand-editing an append-only journal is out of contract, so
   this is noted rather than defended against.
