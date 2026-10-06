# Orchestra

Orchestra is a role-based **Codex** skill for substantial multi-agent work. It keeps
briefing, isolation, evidence, verification, and accountability with the orchestrator
while requiring explicit model routing for subagent roles. It is not a claim that a
Markdown skill can switch the active chat's model automatically.

## The model split

| Role | Model | Use |
|---|---|---|
| Expert | Astra — `gpt-6-astra` | Consequential architecture/design advice, difficult questions, and independent adversarial review |
| Complex orchestration | Sol — `gpt-6.1-sol` | Plan, brief, coordinate, integrate, accept evidence, and handle complex or high-risk work |
| Default builder | Terra — `gpt-5.6-terra` | Implement, test, fix, validate, and complete clear owned-scope tasks |
| Simple mechanical work | Luna — `gpt-6-luna` | Really simple, tightly specified renames, formatting, and repetitive edits with clear checks |

A Sol-led run normally routes experts to Astra, builders to Terra, and the simplest
edits to Luna. This prevents every child from silently inheriting the orchestrator's
model. Same-model dispatch is only for a deliberate role-required complex Sol task or
a disclosed supported-model fallback; record the justification in the run ledger.

Promote Luna to Terra when implementation decisions arise, and Terra to Sol when the
work becomes complex, ambiguous, or high-risk. See the focused
[Codex routing runbook](references/codex.md) for execution-surface checks and Terra's
managed CLI route.

## Install for Codex

```bash
git clone https://github.com/Oxiom-Systems/orchestra.git ~/.agents/skills/orchestra
```

Then invoke it in a Codex session:

```text
$orchestra
```

It also applies to requests such as “orchestrate this”, “use subagents”, “delegate
this”, or “build this out”.

## Optional Claude Code compatibility packaging

The public repository retains Claude Code marketplace/plugin packaging for compatibility;
it is not a Claude-only product. To install that optional package:

```bash
claude plugin marketplace add Oxiom-Systems/orchestra
claude plugin install orchestra@orchestra
```

Then use `/orchestra`. Alternatively, clone it directly into your Claude Code skills
directory:

```bash
git clone https://github.com/Oxiom-Systems/orchestra.git ~/.claude/skills/orchestra
```

Claude Code invokes these Codex model roles through the supervised Codex CLI route in
`references/codex.md`; GPT model IDs are not Claude-native Agent model selectors.

## What it covers

| Section | What it gives you |
|---|---|
| When to orchestrate | The cost multiple, and cases where delegation makes the result worse |
| Division of labour | Explicit roles, ownership, and final orchestrator accountability |
| Model selection | Explicit Codex model/effort routing and promotion rules |
| Dispatch contract | Eight fields for implementer briefs; concise clean-context expert review |
| Shared context | Run directory, orchestrator-owned ledger, and fixed findings contract |
| Parallelism | Fan-out width, worktree isolation, and file ownership |
| Verification | Evidence before acceptance, including raw test results and provenance limits |
| Failure and sequencing | Restart versus repair; authority before implementation |

## A few of the opinions

**Mid-flight steering is unreliable.** If the specification changes materially, stop
the agent and re-dispatch it with the complete specification.

**The reviewer gets clean context deliberately.** Give Astra the requirement, artifact,
and question—not the implementer's reasoning. Agreement is evidence to inspect, never
automatic acceptance.

**Readers can be context-poor; writers need decisions.** Findings merge as facts, but
each edit encodes choices. Give writers the decisions already made or narrow their
scope until they make none that matter.

**Run the new test against the pre-fix code.** If it passes there, it guards nothing.

**A property demonstrated only in a test environment is not verified for the system.**
Keep the revision, inputs, command, raw output, execution surface, and model provenance
with every result.

## Licence

MIT
