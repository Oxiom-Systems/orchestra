# Codex runbook

Reference material for dispatching Codex (GPT-5.6 Terra) as an outside engine.
`SKILL.md` carries the four rules that must survive without this file; read this one
before a dispatch that matters — a hard second implementation, or a review you intend
to act on.

## Models

| Model | Slug | What it is | Default effort |
|---|---|---|---|
| **Terra** | `gpt-5.6-terra` | Balanced agentic coding model for everyday work | `medium` |
| **Sol** | `gpt-5.6-sol` | Reliable agentic workhorse for everyday tasks | `low` |
| **Luna** | `gpt-5.6-luna` | Fast and affordable agentic coding model | `medium` |

All three carry a 272k context window. Effort runs `low → medium → high → xhigh → max →
ultra`, and Luna stops at `max`. `ultra` means maximum reasoning *with automatic task
delegation* — Codex fanning out underneath you, a second orchestrator nested inside your
own, running a split you did not choose and cannot verify.

Nothing in the local model catalog supports calling Sol the frontier choice or Terra
merely "everyday" by comparison — the catalog describes Terra as the balanced choice and
Sol as the reliable workhorse. Check a model's own catalog entry before asserting which
is more capable; it may not be what the name suggests.

Terra is the everyday choice; keep it unless you have a reason to switch. Reach for Sol
as a second, same-lineage read when Terra's answer feels shaky — not because the catalog
calls it more capable, since it does not — and reach for Luna when volume matters more
than depth. Prefer raising Terra's effort before switching model: the axis that usually
helps is reasoning depth, not a different engine.

## Pinning the model is a pattern, not a given

Nothing ships a model pin for you. The author pins Terra locally with a PreToolUse hook,
`~/.claude/hooks/codex-force-model.py`, injecting `--model gpt-5.6-terra` into any
`codex-companion.mjs task|review|adversarial-review` call that does not already carry a
`--model` — that hook lives outside every plugin; write your own if you want the
guarantee, or pass `--model` explicitly on every call instead. Whichever you choose, do
not trust `~/.codex/config.toml` to hold the model you expect — the Codex desktop app
rewrites that file whenever a model is picked in its UI, and it has been found reading
`model = "gpt-5.6-sol"` while nothing in the session set it there.

**Effort is not pinned by the hook. It comes from `~/.codex/config.toml`, and `review` /
`adversarial-review` have no `--effort` flag to override it with.** Check
`grep -E '^(model|model_reasoning_effort)' ~/.codex/config.toml` before relying on
either — it answers in milliseconds. `codex --strict-config doctor` is not a reliable
substitute: it can take minutes scanning rollout history and has been observed to print
no active-model line at all. If config reads `ultra` or `max`, a `review`/
`adversarial-review` dispatch runs there regardless of what you intended, and Codex's
own `AGENTS.md` gives it standing authorization to fan out review sub-agents on top of
that. `task` does take `--effort`; when effort matters, run the review as a `task` with
an explicit prompt and `--effort high` rather than the purpose-built subcommand — that
is the only path on which effort is actually controllable. Keep `review --base <ref>`
for a small interactive run you will watch yourself.

A third rule for the pinning hook is designed but not yet built: read
`model_reasoning_effort` from config at dispatch time and raise a permission prompt
when it is `ultra`/`max` for a `review`/`adversarial-review`, or for a `task` carrying
no `--effort`. It fails open on any error, and it stops treating a `-m` that merely
appears inside prompt text — rather than as an actual flag — as an existing `--model`.
Until it ships, the config check above is manual; do it yourself first.

## `--write` is a writer

`executeTaskRun` sets `sandbox: request.write ? "workspace-write" : "read-only"`, with
`approvalPolicy: "never"`, resolved against whatever directory the Bash call's cwd
`git rev-parse --show-toplevel`s to. `isolation` is an `Agent`-tool feature; a raw Bash
call gets none of it, and a hand-written `codex` agent that merely wraps the Bash call
does not create a worktree on its own either — you have to ask for one. A `task --write`
dispatched from the orchestrator's own checkout is therefore an unattended,
approval-free writer sitting in the same checkout your other agents are using — exactly
the collision *Parallelism and isolation* says happens reliably within an hour.

If you have defined a `codex` agent (see *Arriving as an `Agent`-tool subagent* below),
dispatch it with `isolation: "worktree"`; otherwise `git worktree add` yourself first and
pass `--cwd <worktree>` (the companion accepts `--cwd`/`-C`). Job state then lives under
that worktree only (see below) — the run is visible from there and nowhere else.

## Never pass `--background`

That flag calls `spawnDetachedTaskWorker` — `spawn(…, {detached: true, stdio:
"ignore"})` followed by `child.unref()`. The run is orphaned from the session on
purpose: no stdout, no exit signal, no entry in the Background tasks panel, nothing you
can orchestrate. It is why dispatching the `codex-rescue` subagent with this flag
returns **empty** while the job runs on — you spend a Claude subagent's tokens on a
shell call and find the real output later, if you remember to look.

The plugin's own SessionEnd hook (`cleanupSessionJobs`) terminates every queued/running
job tied to this session and tears the broker down on exit — so `--background` does not
even buy the one thing its name promises, a run that outlives the session. It is the
one flag on this surface that does the opposite of what it says.

Drive the companion in its **foreground** mode instead, inside a harness-backgrounded
Bash call: `run_in_background: true` on the Bash tool call, no `--background` flag on
the companion itself.

```
Bash(run_in_background: true, command:
  node "$CODEX/scripts/codex-companion.mjs" task --write \
       --model gpt-5.6-terra --effort high "<prompt>")
```

`$CODEX` is the plugin root (`~/.claude/plugins/cache/openai-codex/codex/<version>`).
That one change buys the whole orchestration surface, because the process is now the
harness's:

- it appears in the **Background tasks panel** (`/tasks`) with a task id
- progress streams live into the task's output file — `Read` it at any point mid-run
- you are **re-invoked by a completion notification** when it exits; no polling loop
- `TaskStop` (the tool `KillShell` now aliases to) cancels the client's connection
- the plugin's own registry still works — foreground runs go through `runTrackedJob`,
  so `status --all`, `result <job-id> [--json]` and `cancel <job-id>` see the job as
  before

The companion has no internal foreground timeout; it blocks until Codex finishes.
Because the call is backgrounded, the 10-minute ceiling on a foreground Bash call does
not apply.

Cancelling only stops half of it: by default the companion connects through a detached
shared broker (`CodexAppServerClient.connect` → `ensureBrokerSession` →
`spawn(…, {detached: true})` + `unref()`), so killing the Bash task kills the client —
the Codex turn keeps running, and billing, on the broker until you call
`cancel <job-id>` (which does `turn/interrupt`) or the session ends.

## Two further traps

- **Job state is scoped to the repository/cwd.** `resolveStateDir` keys on
  `git rev-parse --show-toplevel`, which for a linked worktree is the worktree itself,
  not the main checkout. Poll from the same directory you dispatched from, or
  `status --all` reports "No jobs recorded yet" while the job is perfectly alive
  somewhere else. `status`/`result`/`cancel` also filter by
  `CODEX_COMPANION_SESSION_ID`; `--all` only lifts the eight-job cap, it does not cross
  sessions.
- **Also check the file the prompt asked for.** A detached or crashed job frequently
  writes its findings correctly even when nothing came back through the wrapper — treat
  a missing return as "look on disk", not as "it failed".

## Flags

`--effort` accepts only `none|minimal|low|medium|high|xhigh`; `max` and `ultra` come
from `~/.codex/config.toml` and are rejected as flags — there is no
`--model_reasoning_effort` flag, that is the config key only. An unrecognized `--flag`
is not rejected either: the parser turns it into a positional, so it lands silently
inside the prompt text Codex receives.

**`task` has no `--wait`.** Foreground is already `task`'s default, so there is nothing
to wait for. `--wait` is a real boolean on `review`, `adversarial-review` and `status`
— not on `task`. `--background` is a real boolean on `review`, `adversarial-review` and
`task` — not on `status`. Passing `["--wait", "<prompt>"]` to `task` returns
`{options:{}, positionals:["--wait","<prompt>"]}`: the flag is silently prepended to the
text Codex is asked to work on.

`/codex:rescue` and `/codex:review` remain fine for a small interactive run you will
watch:

```
/codex:rescue --model gpt-5.6-terra --effort high <task>
```

`--model gpt-5.6-sol` overrides a single one.

## Arriving as an `Agent`-tool subagent

The `openai-codex` plugin ships exactly one agent, `codex-rescue` — dispatch it
(`subagent_type: 'codex:codex-rescue'`, or `/codex:rescue`) when a Claude should hold
and relay a Codex run: visible in the agent panel, its output kept out of your own
context.

A `codex` agent that wraps the foreground-Bash pattern above verbatim — so a `Workflow`
can fan several runs out in parallel with `agent(prompt, {agentType: 'codex'})`, or so
the type name reads `codex` rather than `codex-rescue` — is not something the plugin
ships. It is a short agent definition you write yourself, under
`~/.claude/agents/codex.md` or wherever your own agents live, if you want that shape.
Use the raw Bash call directly instead when you want the output yourself, at zero
subagent cost — that path needs no agent definition at all.

## The rule that still governs it

Agreement is not verification here either, and doubly so: a different engine
disagreeing with your fleet is a signal worth reading; a different engine *agreeing*
with it is not. Codex output is a claim until you check it, same as any other agent's.
