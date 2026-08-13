# agent-patterns

Patterns for multi-agent coding sessions, written as `.agent` files: plain-text
role definitions a coding agent (Claude Code or similar) reads at startup.

The first pattern here is the **actor pattern**: one orchestrator, many
implementor sub-agents, coordinating over Redis Streams instead of shared files
or a chat transcript.

## The problem

Fanning a plan out to parallel coding sub-agents fails in ways single-agent
sessions never see, because of how these agents actually execute:

- **Sub-agents run to completion.** There is no push channel and no inbox: an
  agent can only learn something new at the moments it chooses to look. If
  another agent renames an interface mid-flight, nothing delivers that fact.
- **The orchestrator is turn-based.** While a sub-agent runs, the orchestrator
  is suspended and cannot reach it. Routing messages "through the lead" only
  works when an agent surfaces mid-run, peers are unreachable except through
  their own polling.
- **Agent context is lossy.** Contexts get compacted, sessions get resumed,
  agents exit before a late message lands. Anything that lives only inside an
  agent's context, could be, a claim on a file, a changed signature, a "don't touch
  this module", all of these, can silently vanish.
- **Naive fixes make it worse.** Busy-waiting on a dependency burns a whole
  agent doing nothing. Appending to a shared log file breaks under git
  worktrees. Re-reading a transcript from the top floods every agent's context
  with messages it has already handled.

The result, without structure: duplicated work, conflicting edits to the same
file, and decisions made on stale information with no way to even audit what
was communicated when.

The actor pattern's answer is to move coordination out of agent memory entirely,
onto a durable message stream with per-agent cursors, and to make "blocked"
a first-class state the orchestrator sequences around rather than something
agents wait out.

## The actor pattern

```
actor.agent               dispatcher + communication substrate (read first, both roles)
actor-orchestrator.agent  team lead — plans, brokers messages, ties up loose ends
actor-implementor.agent   teammate — implements a slice of the plan via TDD
```

Each agent gets an identity (`orchestrator`, `impl-backend`, …) and a shared
stream (`agentic:<plan-name>:channel`). Agents send typed messages
(`info | claim | ask | done`) with `XADD`, and poll their own consumer group at
checkpoints, all before claiming a file, making a design choice, or committing.

## Why Redis Streams

The substrate has to satisfy an unusual set of constraints: consumers that can
only poll, producers that may exit at any time, readers whose local state
(their context window) can be truncated behind their back, and a filesystem
that can't be trusted as a rendezvous point. Redis Streams fits each one:

- **Worktree-immune address.** A relative-path log file breaks under git
  worktrees because each agent's project dir differs per worktree, so appends never
  converge on one file. A network service at a stable address doesn't care
  where the checkout lives.
- **Server-side cursors survive lossy clients.** Consumer groups
  (`XREADGROUP`) give each agent a cursor held by the server, not the agent.
  Every poll returns only messages since the last poll; an idle stream returns
  nil. No duplicate handling, no re-reading the transcript from the top, and
  context compaction can't eat the cursor, which boils to, the one piece of state that must
  not be lost never enters the agent's context.
- **Durability decouples correctness from delivery.** The stream is the source
  of truth; the orchestrator is a lossy broker. A message must never live only
  in an agent's context. With a durable stream plus explicit acks (`XACK`),
  correctness reduces to "was every message eventually consumed", answerable
  mechanically, which is what makes the tie-up sweep possible.
- **Dead letters are queryable, not remembered.** At tie-up, `XPENDING`
  (delivered but un-acked) and a fresh `XREADGROUP` (never delivered) identify
  every message an exited agent missed, completely driven by stream state, not by anyone
  recalling who was told what. Missed messages become resume work, not silent
  loss.
- **The channel becomes an artifact.** `XRANGE` serializes the full
  conversation to `channel.md`, committed alongside the code, which gives an audit trail
  of what was claimed, asked, and decided during the run.

Nothing here is Redis-specific in spirit. Any log with consumer groups
(Kafka, NATS JetStream) would do. Redis is simply the cheapest thing to have
running on a dev machine (`redis-cli ping` is the entire health check).

## Coordination rules that matter

- **Never busy-wait.** A hard dependency becomes a `type=ask` message and a
  return to the orchestrator, which sequences the next invocation. Advisory
  info (claims, signatures, "module green") flows peer-to-peer and never blocks.
- **Memory files are serialized context.** Each implementor keeps
  `memories/<name>-memory.md` current so it can be resumed if a message lands
  after it exits.
- **Tie-up is stream-driven, not memory-driven.** After sub-agents finish, the
  orchestrator sweeps `XREADGROUP`/`XPENDING` for messages an agent never
  consumed, resumes agents to close loose ends, then serializes the whole
  stream to `channel.md` as the committable archive of record.

## Usage

Point your coding agent at `actor.agent` with the variables filled in
(`<PROJECT-DIR>`, `<plan-name>`, `<AGENT>`); it routes itself to the right role
file. Requires a reachable `redis-server` (`redis-cli ping` is checked before
any fan-out).

## Status

Early. One pattern so far; more as they prove themselves in real sessions.

## License

MIT
