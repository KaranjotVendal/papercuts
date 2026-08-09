# Specification

Contract version `1`. The authoritative machine-readable copy is `papercuts schema`.

## Storage

A single append-only JSONL file. One JSON object per line, UTF-8, newline-terminated.

| | |
|---|---|
| Default path | `~/.papercuts/log.jsonl` |
| Override | `PAPERCUTS_FILE` |
| Created | On first write, with parent directories |

Writes take an advisory lock (`flock`) and append one line. Reads skip any line that fails to
parse rather than failing, so a torn final line from an interrupted write cannot break the tool.

Records are never edited or removed. Resolving appends a separate record.

## Record shapes

### `cut`

```json
{
  "kind": "cut",
  "id": "pc_9f2c41d0a8b3",
  "ts": "2026-08-09T21:14:03.412Z",
  "text": "While calling moshi-hook over ssh the command failed with 'not found', because ~/.local/bin is absent from the non-interactive PATH. Wrapping in bash -lc fixes it.",
  "tags": ["ssh", "path"],
  "severity": "minor",
  "agent": "claude-opus-5",
  "user": "alex",
  "host": "workstation",
  "repo": "moshi-browser",
  "cwd": "~/dev/moshi-browser",
  "cmd": "ssh personal 'moshi-hook servers'",
  "exit_code": 127
}
```

| Field | Type | Notes |
|---|---|---|
| `kind` | `"cut"` | Discriminator |
| `id` | string | `pc_` + first 12 hex of SHA-256 of the trimmed text. Content-addressed, so filing the same text twice yields the same id |
| `ts` | RFC 3339, UTC | Overridable with `PAPERCUTS_NOW` for deterministic tests |
| `text` | string | Required. Prose, not structured |
| `tags` | string[] | Lower-cased, deduplicated, may be empty |
| `severity` | `minor` \| `major` \| `blocker` | Defaults to `minor` |
| `agent` | string \| null | From `--agent` or `PAPERCUTS_AGENT` |
| `user` | string \| null | From `USER` |
| `host` | string \| null | Machine name. Present so two logs can be merged later |
| `repo` | string \| null | Name of the containing git repository, if any |
| `cwd` | string \| null | Home-relative where possible |
| `cmd` | string \| null | The command that failed, from `--cmd` |
| `exit_code` | int \| null | Its exit status, from `--exit` |

Captured stderr is deliberately not supported. See the design document.

### `resolution`

```json
{
  "kind": "resolution",
  "id": "pc_9f2c41d0a8b3",
  "ts": "2026-08-09T22:01:55.004Z",
  "agent": "alex",
  "host": "workstation",
  "note": "documented in the skill"
}
```

A cut is **open** unless the log contains a `resolution` with its id.

## Fold rules

Applied when reading:

1. Unparseable lines are skipped.
2. Duplicate `cut` records with the same id fold to the first occurrence. Later duplicates are
   discarded, so post-merge duplicates are harmless.
3. A `resolution` whose id matches no cut is ignored.
4. Default ordering is severity first (`blocker`, `major`, `minor`), then newest first.

## Commands

### `papercuts add`

```
papercuts add <text> [--tag TAG]... [--severity minor|major|blocker]
                     [--agent NAME] [--cmd CMD] [--exit CODE] [--json]
```

Appends one `cut`. Reading `-` as the text takes it from stdin. Duplicate-safe: filing text that
already has an open cut appends nothing and reports `changed: false`.

### `papercuts list`

```
papercuts list [--since DURATION] [--tag TAG] [--repo NAME] [--host NAME]
               [--severity LEVEL] [--all] [--format table|md] [--json]
```

Open cuts by default; `--all` includes resolved. `DURATION` is `7d`, `24h`, `30m`, or a
combination such as `1d12h`. `--format md` emits a review digest for pasting into notes.

An empty result is success, exit `0`, not an error.

### `papercuts resolve`

```
papercuts resolve <ID>... [--note TEXT] [--agent NAME] [--json]
```

Appends a `resolution` per id. Ids may be given as any unique prefix. An ambiguous prefix is an
error; an already-resolved id is reported as `changed: false` rather than failing.

### `papercuts tags`

```
papercuts tags [--since DURATION] [--all] [--json]
```

Tag counts, most frequent first. This is the review command: recency is what `list` shows,
recurrence is what tells you what to fix.

### `papercuts schema`

```
papercuts schema
```

Emits the full contract as JSON: contract version, commands and their flags, record shapes,
environment variables, and exit codes. Agents run this to self-orient without depending on a
skill file being current.

## Output

Human-readable by default. `--json` emits a single envelope on stdout:

```json
{
  "ok": true,
  "data": { },
  "meta": { "contract": 1, "file": "/Users/x/.papercuts/log.jsonl", "host": "workstation" }
}
```

Errors go to stderr as `{"ok": false, "error": {"code": "...", "message": "..."}}`.

With `--json`, stdout carries data only, so it stays safe to pipe into `jq`.

## Environment

| Variable | Purpose |
|---|---|
| `PAPERCUTS_FILE` | Log location |
| `PAPERCUTS_AGENT` | Default value for `agent` |
| `PAPERCUTS_NOW` | Fixed RFC 3339 timestamp, for deterministic tests |
| `PAPERCUTS_LOG_LEVEL` | Console log level on stderr, default `WARNING` |

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success, including empty results |
| 2 | Usage error, from the argument parser |
| 65 | Bad input, such as an unparseable duration or an ambiguous id prefix |
| 66 | Not found, such as an id prefix matching nothing |
| 74 | I/O error reading or writing the log |
| 75 | Lock timeout, retryable |
