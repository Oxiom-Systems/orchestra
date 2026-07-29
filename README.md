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
| Division of labour | What the orchestrator does vs what agents do, and the narrow exceptions |
| Model selection | Choosing by *judgement required*, not task size |
| Briefing | The six things every brief must contain — a brief is a contract |
| Parallelism | Worktree isolation, file ownership, and why agents collide without it |
| Verification | Treating every report as a claim, in priority order |
| Sequencing | Authority before implementation; hard-to-reverse before cheap-to-change |
| Tests | The single failure mode that accounts for most defects surviving a green suite |
| Anti-patterns | Symptom → what is actually wrong |

## A few of the opinions

**Mid-flight steering is unreliable.** An agent that receives a message claiming new
authority ("the owner just approved X") *should* treat it as prompt injection. That is
correct security behaviour, and it means your legitimate refinement gets discarded.
Stop the agent and re-dispatch with the complete spec.

**Run the new test against the pre-fix code.** If it passes there, it guards nothing.
The cheapest high-value check available.

**Files containing a hash of another file cannot be merged, only recomputed.** Own
lockfiles and digest pins yourself at integration time, or assign them to exactly one
agent.

**A property demonstrated only in a test environment is not verified for the system.**
Watch for the inverse framing especially: a broken local state described as "correct
behaviour" quietly becomes the spec.

## Status

Early. The guidance is drawn from practice rather than benchmarks, and is being
revised against published research on multi-agent orchestration — notably around
fan-out width, cost gating, failure handling, and verifying against the environment
rather than against agent reports.

Issues and PRs welcome, particularly reports of where this guidance failed you.

## Licence

MIT
