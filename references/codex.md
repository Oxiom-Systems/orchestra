# Codex routing runbook

Use this runbook when dispatching an Orchestra role. It documents observed host
capabilities on 2026-10-06, not universal availability, pricing, context, or execution
identity guarantees. Recheck the catalog and the actual dispatch surface immediately
before each launch.

## Role routing

| Role | Preferred model | Use it for |
|---|---|---|
| Expert | `gpt-6-astra` | Consequential architecture/design advice, difficult questions, and adversarial review |
| Complex orchestration | `gpt-6.1-sol` | Planning, briefing, coordination, integration, evidence acceptance, and complex or high-risk work |
| Builder | `gpt-5.6-terra` | Default implementation, tests, fixes, validation, and task completion within a clear owned scope |
| Simple | `gpt-6-luna` | Really simple, tightly specified mechanical tasks with clear checks |

Use a different explicit model for each role; do not let a subagent inherit the
orchestrator's model. A deliberate complex Sol task may use Sol again, and a supported
model fallback is permitted only when recorded with its reason in the ledger.

Promote Luna to Terra when implementation decisions arise. Promote Terra to Sol when
the task becomes complex, ambiguous, or high-risk. Ask Astra for important expert input
or an independent adversarial read. A role preference does not change the main chat
model: the active orchestrator remains accountable for final acceptance.

At this check, Astra, Sol, and Terra were observed with `low`, `medium`, `high`,
`xhigh`, `max`, and `ultra` effort choices. Start at `high` for Astra/Sol, `medium` or
`high` for Terra, and `low` or `medium` for Luna, then choose only an effort accepted by
the surface you are actually using.

## Native dispatch

The native collaboration schema exposed Astra, Sol, and Luna, but not Terra. Give
every agent a self-contained brief and use `fork_turns: "none"` for fresh review or
validation context. Set both model and effort explicitly; do not use a full-history fork
when the runtime would inherit the parent settings.

```json
{
  "task_name": "expert_review",
  "model": "gpt-6-astra",
  "reasoning_effort": "high",
  "fork_turns": "none",
  "message": "GOAL: ...\nWHY: ...\nGIVEN: ...\nRETURN: findings/expert.html"
}
```

Use the same shape for Sol and Luna with their role-appropriate IDs and supported
efforts. Record the native agent ID, requested model/effort, raw result location, and
any observable reported identity. An accepted request is not independent attestation
that a particular model executed.

## Managed Terra route

Do not invent a native Terra launch where native dispatch rejects it. First recheck
`codex exec --help` and the installed CLI's current catalog. The observed CLI supports
`--ephemeral`, `--model`, `-c model_reasoning_effort`, `--sandbox`, `--json`,
`--output-last-message`, `--cd`, and `--skip-git-repo-check`.

Run Terra as a tracked foreground `codex exec` child. Keep its execution session or
process identifier, stdout, stderr, exit status, final response, and artifacts. Never
detach it (`nohup`, `disown`, `&`, or a separate user-owned chat). Use `read-only` for
analysis; use `workspace-write` only in an assigned isolated workspace for authorized
implementation.

Replace the `/path/to` paths with the actual workspace and run paths, and create
`findings/` and `artifacts/` first. The brief tells the worker to write
`findings/terra.html`; capture its final paths-and-summary response separately.

```bash
codex exec --ephemeral --model gpt-5.6-terra \
  -c 'model_reasoning_effort="high"' \
  --sandbox workspace-write --json \
  --cd /path/to/assigned-workspace \
  --output-last-message /path/to/run/artifacts/terra-final.txt \
  - < /path/to/run/artifacts/terra-brief.txt \
  > /path/to/run/artifacts/terra-events.jsonl \
  2> /path/to/run/artifacts/terra-stderr.txt
```

Use `--skip-git-repo-check` only when the installed help supports it and the assigned
workspace is intentionally not a Git repository. Prefer these per-run flags over global
configuration, hooks, or authentication changes.

If Terra is unavailable through every supervised surface, dispatch a supported Sol
fallback explicitly and record the Terra-to-Sol fallback, reason, actual model, and
effort. Never silently substitute Astra for builder work.

## Luna availability

Native dispatch at this check exposed `gpt-6-luna`. The installed CLI cached catalog
exposed the older `gpt-5.6-luna` instead. Prefer the latest Luna model that the chosen
surface actually supports. If only the older Luna or Terra is supported, record that
fallback and its checks; never silently substitute it.

## Evidence and review

For a clean Astra review, provide only the requirement, final artifact, and the question
to answer—not the implementer's reasoning or self-assessment. For every delegated test
or validation, preserve the command, inputs, raw output, exit code, artifact/revision,
execution surface, requested model/effort, and any observed identity. That evidence
informs acceptance; it does not transfer acceptance from the orchestrator.
