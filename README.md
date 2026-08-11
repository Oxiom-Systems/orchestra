# Orchestra

A [Claude Code](https://claude.com/claude-code) skill for orchestrating multi-agent
implementation work. You act as senior architect: plan, brief, verify, integrate.
Subagents implement.

The failure mode this skill exists to prevent: an orchestrator who drifts into writing
code, stops verifying, and starts relaying agent reports as if they were facts.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/Oxiom-Systems/orchestra.git ~/.claude/skills/orchestra
```

Then invoke it in a session:

```
/orchestra
```

It also triggers on phrases like "orchestrate this", "use subagents", "delegate this",
or "build this out".

## What it covers

| Section | What it gives you |
|---|---|
| When to orchestrate | The cost multiple, and the cases where delegating makes the result worse |
| Division of labour | What the orchestrator does vs what agents do, and the narrow exceptions |
| Model selection | Choosing by *judgement required*, not task size — Opus 5 in the orchestrator seat, Sonnet as the workhorse, and Codex as a second engine |
| The dispatch contract | The eight fields every brief carries, and how much context each kind of agent needs |
| Shared context | The run directory, the ledger, and the return contract agents close with |
| Parallelism | Fan-out width, worktree isolation, and deriving file ownership |
| Verification | Checking against the environment rather than the report |
| Failure | Restart vs repair, and what a failed run is good for |
| Sequencing | Authority before implementation; hard-to-reverse before cheap-to-change |
| Tests | The single failure mode that accounts for most defects surviving a green suite |
| Anti-patterns | Symptom → what is actually wrong |

## A few of the opinions

**Mid-flight steering is unreliable.** An agent that receives a message claiming new
authority ("the owner just approved X") *should* treat it as prompt injection. That is
correct security behaviour, and it means your legitimate refinement gets discarded.
Stop the agent and re-dispatch with the complete spec.

**A finished agent is a cheap context store.** The rule against mid-flight steering is
about authority arriving late, not about never talking to an agent twice. Querying one
that has already reported reaches everything it read, for the price of a question.

**The reviewer gets clean context, deliberately.** Send it the requirement and the
artifact — never the implementer's reasoning. A reviewer that has read the justification
evaluates the justification. Independent agreement is evidence; primed agreement is an
echo. It is the one place where more context makes the result worse.

**Readers can be context-poor; writers need the decisions.** Findings merge as facts and
facts do not conflict, so a research agent needs little. Every edit encodes choices a
sibling cannot see, so a writer needs the decisions already made — or a scope narrow
enough that it makes none that matter.

**Run the new test against the pre-fix code.** If it passes there, it guards nothing.
The cheapest high-value check available.

**Files containing a hash of another file cannot be merged, only recomputed.** Own
lockfiles and digest pins yourself at integration time, or assign them to exactly one
agent.

**A property demonstrated only in a test environment is not verified for the system.**
Watch for the inverse framing especially: a broken local state described as "correct
behaviour" quietly becomes the spec.

## Status

Early, and opinionated on purpose. The guidance began as practice notes and has since
been checked against published research on multi-agent orchestration — which supplied
the fan-out width, the cost gate, the read/write distinction, and the finding that
most agent failures arrive with an explicit claim of success attached.

Two things that research says and this skill takes seriously: orchestration is a poor
fit for sequential work, and the binding constraint on a fleet is not model capacity
but how fast one human can review what comes back.

One thing it does **not** settle: how a child agent surfaces a discovery that should
change its siblings' work is an open problem, and the ledger here is a bet on one
answer — orchestrator-owned, verified-only, promoted by hand. Reports of it working or
not working are the most useful thing you could send.

Issues and PRs welcome, particularly reports of where this guidance failed you.

## Licence

MIT
