# Design

## The problem

Coding agents hit friction constantly and say nothing about it. A tool call dead-ends, a doc is
wrong, a flag has to be discovered by experiment, an environment assumption fails. The agent works
around it, finishes the task, and the signal evaporates. The next agent hits the same wall.

In one working session on a two-machine setup, the following came up and were all worked around
silently or mentioned once in passing:

- Tools installed under `~/.local/bin` are absent from the non-interactive `PATH`, so every
  scripted `ssh host 'tool ...'` fails until wrapped in `bash -lc`. Hit twice, with two different
  tools.
- A terminal multiplexer refuses to nest, so its remote client has to be launched from a plain
  terminal rather than inside a pane.
- `who` does not see PTY-less SSH, so an established connection with empty `who` output looks like
  a stuck prompt. A remote agent misdiagnosed exactly this.
- A connection timing out means a packet filter is dropping SYN; connection refused means nothing
  is listening. Confusing the two sent a debugging session down the wrong path.
- A review tool pins one fixed port per remote host, so a second concurrent review always collides.
- `nc -z -G 1` reported a live port as closed.

Four of those were judged durable enough to write down by hand. The rest were lost. The judgment
call is the problem: it happens at the moment of friction, when the agent is mid-task and least
inclined to stop.

## The idea

Give the agent a one-line way to file the complaint, cheap enough that filing beats deciding
whether to file. Review the accumulated log periodically and fix what recurs.

Credit where due: the idea is [Steve Ruiz's](https://x.com/steveruizok/status/2075303919664734295),
and [treygoff24/papercuts](https://github.com/treygoff24/papercuts) is a well-built Rust
implementation of it whose design informed several decisions here.

## What counts as a papercut

**File one when you worked around something rather than solved it, and the next agent would hit
the same wall.**

In scope:

- a tool call that dead-ended
- a doc, README, or help text that was wrong
- an error message that misled
- a flag or behaviour that had to be discovered by experiment
- an environment assumption that did not hold

Out of scope:

- your own reasoning mistakes
- a test that failed because the code was wrong
- anything the user corrected you on, which belongs in durable memory instead
- anything already written down

The clause "the next agent would hit the same wall" is load-bearing. It separates friction that is
a property of the tooling from friction that was a property of that particular attempt.

## Decisions

### One local file per machine, outside any repository

The log is a single append-only JSONL file, `~/.papercuts/log.jsonl` by default, overridable with
`PAPERCUTS_FILE`.

Alternatives considered and rejected:

**A file committed at each repository root.** The strongest argument for it is free
synchronisation: the log travels with the repo, so every machine that clones gets it. Rejected
because papercuts are notes to ourselves, and in a shared work repository they would land in code
review and in permanent history.

**A shared folder synchronised between machines.** Rejected because the synchronisation daemon was
not installed on both machines, and installing an always-on service with a listening port on a
managed corporate laptop is a larger decision than this tool warrants.

**A single log on one machine, appended to over SSH.** Rejected because that machine's uptime is
unpredictable, which would make both writing and reading fail unpredictably.

The chosen design has no network dependency at all. Nothing blocks, nothing needs another machine
awake, and nothing enters a work repository.

### Merging is deferred, but designed for

Two machines produce two logs. Merging them later must not require coordination now, so:

- every record carries `host`
- ids are content-addressed, so the same complaint filed twice produces the same id

Merging is then concatenate and deduplicate. There is no conflict to resolve because nothing is
ever edited.

### A journal, not a database

Resolving a papercut appends a resolution record rather than editing the original. The file is
append-only in the strict sense, which is what makes concurrent writers safe and merging trivial.
Whether a cut is open is derived at read time from the presence of a resolution with its id.

### Concurrency is real here

Several agent sessions run in parallel on one machine, in separate worktrees. Two of them filing at
once must not corrupt the log. Writes take an advisory lock and append a single line; reads
tolerate a torn final line rather than failing.

### Agents self-orient with `schema`

A `schema` command returns the full machine contract: commands, flags, record shapes, exit codes.
An agent that has not seen the skill, or has seen a stale copy of it, can discover the current
contract from the tool itself. This is taken directly from the Rust implementation and is the best
idea in it.

### Evidence capture is deliberately limited

The Rust implementation can attach captured stderr to an entry, and warns that redaction is
best-effort. On a machine holding cloud credentials and API keys, storing captured stderr in a file
is a poor trade for the value it adds. `--cmd` and `--exit` carry most of the diagnostic weight
with none of the leak risk, so stderr capture is omitted.

### Format compatibility

Record shapes follow the Rust implementation's field names where they overlap. Either tool can then
read either file, so the data outlives whichever tool is in use. The file is the product.

## What makes it fail

**A log nobody reads.** This is the likely failure, and it is not solved by code. The mitigation is
a review ritual: read the week's entries during weekly planning, promote what recurs into durable
memory or a ticket, delete the noise. `tags` exists to make that review fast by showing what
recurs rather than what is merely recent.

**Under-reporting.** Agents push through problems by default, which is the entire premise. The
mitigation is a filing bar sharp enough to fire in the moment, and a command cheap enough that
filing is faster than deliberating.

**Over-reporting.** A log full of trivia is a log nobody reads. The out-of-scope list is as
important as the in-scope list.
