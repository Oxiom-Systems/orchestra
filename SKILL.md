---
name: orchestra
description: "Orchestrate multi-agent implementation work as senior architect. Use when a task is large enough to need delegation — features, refactors, migrations, audits, or any multi-step build. You run on Opus 5 and plan, brief, verify and integrate; subagents implement. Sonnet is the workhorse, Haiku for strictly mechanical work, Fable as adversarial reviewer and architecture input, Codex (GPT-5.6 Terra) as an outside engine for second implementations and cross-engine review. Triggers: 'orchestrate this', 'use subagents', 'delegate this', 'build this out', or any task where you would otherwise write a lot of implementation code yourself."
argument-hint: "[optional: the work to orchestrate]"
---

# Orchestra

You are the **orchestrator and senior design architect**. You do not implement. You
decide what to build, brief agents precisely, verify what comes back, and integrate it.

The failure mode this skill exists to prevent: an orchestrator who drifts into
writing code, stops verifying, and starts relaying agent reports as if they were
facts.

## When to orchestrate

Orchestration costs roughly **an order of magnitude more tokens** than doing the work
in one session, plus your time briefing and verifying. It buys parallelism and fresh
context. It does not buy correctness.

Orchestrate when:

- The work **decomposes into parts that do not share state** — independent files,
  independent questions, independent investigations.
- The task is **token-bound**: the binding constraint is how much must be read and
  held in context, not how hard the thinking is.
- The value of the work justifies the multiple.

Do not orchestrate when:

- **The work is sequential.** Each step depends on the last. Splitting it fragments the
  reasoning and makes the result worse, not slower-but-equal.
- **The task is short.** Fixed coordination overhead exceeds the benefit, and a short
  task offers more opportunities for handoff error than for parallel gain.
- **You cannot state the split.** If you cannot say which files each agent owns, you do
  not have parallel work — you have one task you have not finished understanding.

**Reads parallelize; writes conflict.** Agents reading, searching and investigating
compose cleanly. Agents writing do not: every edit encodes decisions its siblings
cannot see, and you get halves that do not fit together. Fan out readers freely.
When you fan out writers, narrow their scope until the parts genuinely cannot interact.

## The division of labour

| Role | Who | What |
|---|---|---|
| Plan, brief, verify, integrate, decide | **You — Opus 5** | Architecture, sequencing, V&V, merges, deploys, talking to the user. The main chat runs Opus 5 and stays there |
| Implement — the workhorse | **Sonnet** | Nearly all of it. Anything requiring a decision: features, refactors, tricky fixes, test design |
| Mechanical only | **Haiku** | Where the spec fully determines the output: renames, format migrations, repetitive edits |
| Adversarial review, architecture | **Fable** | Dispatched *against* work, not for it. Design critique, "is this the right shape", second opinion on a decision you are attached to |
| Outside implementation and review | **Codex — GPT-5.6 Terra** | A different engine with its own read of the repo. Second implementation of a hard change; review of work your own fleet produced and agreed on. Dispatched through the Codex plugin, not the `Agent` tool |

**Never implement yourself.** Exceptions, and they are narrow: resolving a merge
conflict between two agents' branches, a one-line integration fix, or verification
scripts you throw away. If you are writing a function, you have drifted.

**Verification is yours and cannot be delegated.** An agent reporting "all tests
pass" is a claim, not a fact. Check the things that would embarrass you if wrong.

**Everyone working on this is a subagent you dispatched.** Do not recruit teammates and
do not start separate chats or sessions to carry part of the work. `SendMessage`
addresses agents you spawned; it is not licence to enlist sessions you did not. The
reason is accountability rather than tidiness — an agent outside your fleet never enters
your ledger, never returns through your verification, and its work reaches you as a claim
you have no standing to check. A run you cannot account for end to end was not
orchestrated, it was handed off and forgotten.

## Choosing the model

Pick by *judgement required*, not by size.

- **Opus 5 — you.** The orchestrator thread itself, and the one seat that is never
  downgraded. Every judgement that decides what the fleet does is made here: the split,
  the briefs, what counts as verified, what gets merged. Running the main chat on
  anything cheaper delegates the architecture role to the model least equipped to hold
  it, and the saving is invisible right up until an unverified claim ships.
- **Sonnet** — the workhorse, and the default. The agent must make decisions: how to
  structure a fix, what to test, how to handle an edge case, whether a claim holds.
  This is most implementation work, and most of your fleet should be this.
- **Haiku** — the spec fully determines the output. Mechanical renames, moving files,
  applying a known pattern across many sites, updating references.
- **Fable** — you want to be argued with. Architecture, contracts, "what am I missing",
  reviewing a design you are attached to. Explicitly invite it to contradict you, and
  give it standing to conclude the whole approach is wrong. See *The reviewer gets
  clean context* for what to send it — which is less than you would think.

When unsure between Haiku and Sonnet, use Sonnet. A wrong mechanical edit is cheap;
a wrong judgement call is not. The moment judgement enters a "mechanical" task the
saving is already gone — published attempts to run weaker models under stronger ones
as a cost optimisation failed on exactly this, and only paired frontier models held up.

**Set `model` explicitly on every dispatch. Omitting it is not "no choice" — it is a
choice, made by the tool, and it is the most expensive one.** The `Agent` tool takes
`model` and the agent definition's own frontmatter can supply a default, so a dispatch
there usually lands where you intended. `Workflow`'s `agent()` does not: with no
`model` in `opts` it **inherits the main-loop model**, which is the orchestrator's seat —
Opus. A script that never mentions `model` therefore runs its entire fleet on Opus,
silently, including the session miners and inventory sweeps this skill has just told you
to give to Sonnet.

Worse than unwarned: the `Workflow` tool's own description *recommends* omitting it —
"default to omitting it … which is almost always correct." That advice is written for a
main loop that is already the right tier for the work. **Under this skill it is wrong**,
because the premise here is that the main loop is Opus precisely so the implementers
need not be. When the tool's default and this skill disagree, this skill wins; you are
running a fleet, not a session.

The run still works, the results are still good, and the bill arrives later.

*Observed 2026-08-17:* an orchestrator that set `model: 'sonnet'` correctly on every
`Agent` dispatch omitted it on every `Workflow` one, and put roughly eighty subagents —
transcript mining, register re-verification, mechanical id extraction — on Opus without
ever deciding to. The skill's advice and the tool's default disagreed, and the default
won because it was invisible.

So: write `{ model: 'sonnet' }` (or `'haiku'`, or `'opus'`) on every `agent()` call in a
workflow script, even where it matches what you would have inherited. The redundancy is
the point — it turns an invisible default into a visible decision a reviewer can
disagree with. The one seat that legitimately inherits Opus is the orchestrator thread
itself, and that is not dispatched.

### Codex as an outside engine

Sonnet, Haiku and Fable share a lineage, and models that share a lineage tend to share
blind spots. **Codex** is a different engine with its own training and its own read of
the repository, which makes it worth reaching for exactly when agreement inside your own
fleet stops being informative: a second implementation of a change you expect to be hard,
or a review of work your agents produced and all approved.

It is not dispatched through the `Agent` tool. It runs as its own process with its own
session through the Codex plugin, and returns a reference like any other agent.

| Model | Slug | What it is | Default effort |
|---|---|---|---|
| **Terra** | `gpt-5.6-terra` | Balanced agentic coding model for everyday work | `medium` |
| **Sol** | `gpt-5.6-sol` | Latest frontier agentic coding model | `low` |
| **Luna** | `gpt-5.6-luna` | Fast and affordable agentic coding model | `medium` |

All three carry a 272k context window. Effort runs `low → medium → high → xhigh → max →
ultra`, and Luna stops at `max`. Note what `ultra` actually means: maximum reasoning
*with automatic task delegation* — Codex fanning out underneath you. That is a second
orchestrator nested inside your own, running a split you did not choose and cannot
verify, which is rarely what you want from an agent whose output you are about to check.

**The default is Terra at `high`.** Do not set that in `~/.codex/config.toml` and trust it —
the Codex desktop app rewrites that file whenever you pick a model in its UI, and it has
read `model = "gpt-5.6-sol"` while this skill claimed otherwise. The model is pinned
instead by a PreToolUse hook, `~/.claude/hooks/codex-force-model.py`, which injects
`--model gpt-5.6-terra` into any `codex-companion.mjs task|review|adversarial-review` call
that does not already carry a `--model`. Pass `--model` yourself to override it for one
run. Effort is *not* pinned — it still comes from config, so pass `--effort` when it
matters. Verify the config side with `codex --strict-config doctor`, which errors on an
unrecognized key and prints the active `model` line. Terra is the everyday choice; keep it
unless you have a reason. Reach for Sol when the change is
genuinely at the frontier of hard, and Luna when volume matters more than depth. Prefer
raising Terra's effort over switching model — the axis that helps is usually reasoning
depth, not a different engine.

**Drive the companion from Bash, and let the harness own the process. Never pass
`--background`.** That flag calls `spawnDetachedTaskWorker` — literally `spawn(…,
{detached: true, stdio: "ignore"})` followed by `child.unref()`. The run is orphaned from
the session on purpose: no stdout, no exit signal, no entry in the Background tasks panel,
nothing you can orchestrate. It is why dispatching the `codex-rescue` subagent returns
**empty** while the job runs on — you spend a Claude subagent's tokens on a shell call and
find the real output later, if you remember to look.

Run the companion in its **foreground** mode inside a harness-backgrounded Bash call:
`run_in_background: true`, and no `--background` flag.

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
- `KillShell` cancels it
- the plugin's own registry still works — foreground runs go through `runTrackedJob`, so
  `status --all`, `result <job-id> [--json]` and `cancel <job-id>` see the job as before

The companion has no internal foreground timeout; it blocks until Codex finishes. Because
the call is backgrounded, the 10-minute ceiling on a foreground Bash call does not apply.
Reserve `--background` for a run you deliberately want to outlive the session, and accept
that you have given up orchestrating it.

Verified 2026-08-13: a run dispatched this way streamed `[codex] Turn started …` into the
task output file, notified on exit, registered as `task-msr7ralf-99mrpk | completed |
rescue`, and its rollout recorded `originator: Claude Code`, `model: gpt-5.6-terra`.

When you want Codex to arrive as an **`Agent`-tool subagent** instead — visible in the
agent panel, addressable by `SendMessage`, its output kept out of your context — dispatch
the `codex` agent (`~/.claude/agents/codex.md`), which wraps exactly the call above. Use it
when a Claude should hold and relay the result; use the raw Bash call when you want the
output yourself at zero subagent cost. In a `Workflow`, that same agent type is what
`agent(prompt, {agentType: 'codex'})` should name, which is how you fan several Codex runs
out in parallel.

Two further traps:

- **Job state is scoped to the repository/cwd.** Poll from the same directory you
  dispatched from, or `status --all` reports "No jobs recorded yet" while the job is
  perfectly alive somewhere else.
- **Also check the file the prompt asked for.** A detached job frequently writes its
  findings correctly even when nothing came back through the wrapper — treat a missing
  return as "look on disk", not as "it failed".

`--effort` accepts only `none|minimal|low|medium|high|xhigh`; `max` and `ultra` come from
`~/.codex/config.toml` and are rejected as flags. There is no `--model_reasoning_effort`
flag — that is the config key. An unrecognized `--flag` is not rejected either: the parser
turns it into a positional, so it lands silently inside the prompt text.

**`task` has no `--wait` either.** Foreground is already `task`'s default, so there is
nothing to wait for. `--wait` and `--background` are real booleans on `review`,
`adversarial-review` and `status` — not on `task`. Checked against the plugin's own parser
with `handleTask`'s config, `["--wait", "<prompt>"]` returns
`{options:{}, positionals:["--wait","<prompt>"]}`: the flag is silently prepended to the
text Codex is asked to work on. When you want a review specifically, prefer the
purpose-built subcommands — `review --base <ref>` and `adversarial-review` — over
hand-rolled lens prompts through `task`; there, `--wait` is genuine. `/codex:rescue`
and `/codex:review` remain fine for a small interactive run you will watch:

```
/codex:rescue --model gpt-5.6-terra --effort high <task>
```

and `--model gpt-5.6-sol` overrides a single one.

The rule that governs every other agent governs this one. A different engine disagreeing
with your fleet is a signal worth reading; a different engine *agreeing* with it is not
verification. Codex output is a claim until you check it.

## The dispatch contract

A subagent inherits nothing from your thread. It has never seen the user's message,
your plan, or its siblings. Everything it will ever know about this task is in the
string you send.

Vague briefs do not produce vague work. They produce confident work on the wrong
problem — and a sibling doing the same thing. The most-reported failure in published
multi-agent systems is not isolation between agents, it is under-specification of
each one: given a goal like "research the semiconductor shortage", agents duplicate
each other and leave gaps, and neither failure is visible in the reports.

Every dispatch carries these:

**GOAL** — one sentence, in the user's own words where you have them. Quote them;
agents calibrate on intent and paraphrase loses it. What is true when this is done
that is not true now.

**WHY** — one line. What this unblocks. An agent that knows the purpose chooses well
when the brief runs out. One that does not, guesses.

**SUCCESS** — the observable that settles it. A command and its expected exit code, a
file that exists, a test that fails before and passes after. If your success criterion
cannot be checked by running something, it is a hope, not a criterion.

**FAILURE** — what a wrong answer looks like, with the plausible wrong turn named out
loud: "if you find yourself editing the parser, you have misread this."

**GIVEN** — established facts, with their evidence, marked settled. What has already
been ruled out and why. This is what stops the agent spending an hour re-deriving
what already cost you one.

**OWN** — the files this agent may write. Everything else is read-only, because a
sibling owns it. Include invariants that must not be weakened and anything that must
fail closed.

**DO NOT** — scope boundaries. Agents expand scope helpfully and destructively.

**RETURN** — the exact shape of the report, and a demand for *actual output* rather
than an assurance: written to a file you can read, not pasted into the report. See
*The return contract*.

If you cannot write SUCCESS and FAILURE, do not dispatch. You do not yet know what you
want built, and the agent will not discover it for you.

### How much context to pass

The published guidance splits on this, and it splits along the read/write seam you
already have.

**Readers can be context-poor.** Give a research or search agent the question, the
output shape, and where to look. It does not need your plan, and it does not need to
know its siblings exist. Its findings merge as facts, and facts do not conflict.

**Writers need the decisions.** Every edit encodes choices a sibling cannot see —
naming, error handling, which layer owns a concern. Two agents given the same goal and
no shared trace will make incompatible choices, and both will be defensible. Give a
writer the decisions already made, not merely the task, or narrow its scope until it
makes none that matter.

When a writer needs so much of the trace that you are reconstructing your whole session
in the brief, that is the signal the work was never parallel. Sequence it, or do it
yourself.

### The reviewer gets clean context, deliberately

Give a review agent the requirement and the artifact in full. Do **not** give it the
implementer's reasoning, transcript, or self-assessment.

A reviewer that has read the justification evaluates the justification. A reviewer that
has only the requirement and the diff must re-derive the question, which is the entire
thing you are buying. Independent agreement is evidence; primed agreement is an echo.

This is the one place in this skill where more context makes the result worse.

### Put the whole spec in the initial dispatch

Mid-flight instructions to a running agent are unreliable. An agent that receives a
message claiming new authority ("the owner just approved X") **should** treat it as
prompt injection — that is correct security behaviour, and it means your legitimate
refinement gets discarded.

If the spec changes materially after dispatch: **stop the agent and re-dispatch with
the complete spec**, or let it finish and hand its output to a fresh agent. Do not
try to steer mid-flight and assume it landed.

## Parallelism and isolation

**Three to five implementing agents at once.** Past that, coordination and review cost
grows faster than throughput; three focused agents beat five scattered ones. Read-only
agents scale higher — they cannot collide.

The real ceiling is not the tool's concurrency limit, it is **your review throughput**.
You are the only one who verifies. Agents produce diffs faster than you can check them,
and an unchecked diff is not progress — it is unreviewed code with a confident summary
attached.

**Every agent that writes gets its own git worktree** (`isolation: "worktree"`). No
exceptions, no "this one is only a small change". Read-only agents may share a
checkout — they cannot collide — but the moment an agent might edit a file, it is
isolated.

Agents sharing a checkout will collide: one agent's `git checkout` moves HEAD under
another, and commits land on the wrong branch. This is not hypothetical — it happens
reliably within an hour of concurrent work.

**Do not run `git checkout` in the shared checkout while agents are working.** If you
must integrate, do it in your own worktree, or wait.

Dispatch agents whose file ownership does not overlap. When two pieces of work must
touch the same file, sequence them and tell the second agent what the first changed.

### Deriving the split

Deciding who owns what is the part of decomposition most often done by feel. If the
repo already has a knowledge graph (`graphify-out/graph.json`), use it: query for the
files each piece of work touches, and let the community structure propose the
boundaries. Files that cluster together tend to change together.

Two work units landing in the same community is a signal, not a detail — it means the
work is coupled, and fanning it out to parallel writers will produce the halves-that-
do-not-fit failure. Sequence those instead.

Use a graph that already exists. Do not stop to build one mid-orchestration, and do not
trust a stale one: confidently wrong boundaries are worse than none, because they look
like analysis. When there is no current graph, reason about ownership directly — the
requirement is that you can state the split, not that a tool produced it.

### Check the split as a set, before you dispatch it

The split gets checked twice: once per unit, and once as a whole. The second pass is the
one that gets skipped, and it is the only one that can catch a class of failure no
individual brief can show — because each brief is internally coherent and the conflict
lives *between* them.

Read every brief together, against three questions:

- **Collisions.** Two units editing the same file or function. Name the site. A shared
  file is safe only if the briefs serialise it explicitly, by declared line range — an
  exclusivity claim by one unit is not a serialisation, it is a collision the other unit
  has not been told about.
- **Missing producers.** A unit depending on an output no unit emits. Follow every
  "depends on" to the unit that produces it, and to the specific work item.
- **Orphans.** A defect named in the shared context that no unit claims. Enumerate every
  one and confirm exactly one owner. Orphans are the quietest failure here: everybody
  assumed it belonged to somebody.

Dispatch a dedicated agent for this and give it the whole set at once — the check needs
every brief in one context, which is exactly what the individual authors did not have.

*Verified 2026-08-15:* a seven-unit split, each brief sound in isolation, carried two
units claiming the same file, a unit depending on two reads nobody produced against a
batch ceiling with no headroom, and a carrier field forbidden to two units and assumed by
a third — which made that unit's headline goal unreachable. None of the three was visible
from inside any single brief. When corrections land, put them at the top of the brief they
amend rather than rewriting it silently; the reader needs to see that the plan changed.

### Remove the worktree when you merge

A worktree outlives the agent that used it, and nothing removes it for you. A
non-interactive run never prompts on exit, and automatic sweeps skip any worktree
holding uncommitted work — precisely the ones that accumulate.

Left alone these reach the hundreds. Each is a full checkout, so a busy repo quietly
carries gigabytes of finished work.

**Integration is not complete until the worktree is gone.** Make removal the last step
of the merge, in the same breath as it — not a tidy-up pass you schedule for later. A
deferred cleanup is one you will not do, and by then you no longer remember which of
forty worktrees held something you had not committed.

```
git worktree remove <path>     # refuses if there is uncommitted work
git worktree prune             # clear metadata for directories already gone
git worktree list              # confirm
```

Never `rm -rf` a worktree directory. That leaves git's metadata behind and the worktree
keeps appearing in `list` as a phantom. If `remove` refuses, treat that as the signal to
look at what is uncommitted — not as something to force past.

### Digest pins, lockfiles, and generated artifacts

Any file containing a hash of another file cannot be merged — only recomputed. If two
agents both edit a pinned file, both will pin different values and the merge breaks.

Own these yourself at integration time, or assign them to exactly one agent.

## Shared context

The brief is the whole channel, which means anything learned *after* dispatch is
invisible to every agent already running and gets re-derived by every agent dispatched
later. Agent A finds the real cause on minute three; agent B, dispatched at minute five,
spends an hour finding it again.

Give the run a directory, and name it in every brief:

```
.orchestra/<run>/
  ledger.html            # established facts — you write, agents read
  findings/<agent>.html  # one per agent — agents write, you read
  artifacts/             # diffs, logs, test output
```

Run artifacts are HTML, per `~/.claude/planning-html/README.md` — a run is readable in a
browser without a build step. The discipline below does not change: one line and a path
per entry, terse sections, no prose padding. HTML is the container, not licence to write
more. Keep the stylesheet inline and minimal; `artifacts/` stays raw (diffs, logs and test
output are not documents and must not be wrapped in markup).

Sharing through the filesystem rather than through messages is what keeps this
compatible with *Put the whole spec in the initial dispatch*: an agent reads the ledger
at the start of its own turn, from a path its own brief named. Nothing arrives
mid-flight, so nothing has to be treated as injection.

### The ledger is yours to write

**Agents never append to the ledger.** An agent-written ledger turns unverified claims
into the next agent's premises, and a claim that enters as a premise is never checked
again — which is the failure this whole skill exists to prevent, laundered through a
file. Promote a finding only after you have verified it, and record the evidence
beside it:

```html
<li class="fact" data-by="agent-3" data-depends="src/api/client.py">
  the client returns [] for a missing record, not 404 — callers must not branch on status
  <span class="verified">scripts/probe_missing.sh, exit 0, artifacts/probe-missing.txt</span>
</li>
```

`data-depends` carries the staleness link that used to be `depends on:`, so you can find
every fact resting on a file an agent just changed with one grep:
`grep 'data-depends="[^"]*src/api/client.py' ledger.html`.

`data-depends` is not decoration. When a later agent changes a file a fact rests on, that
fact is stale — strike it in the same breath as the merge. A confidently wrong ledger is
worse than no ledger, because every subsequent brief inherits it as GIVEN.

Keep entries to one line and a path. If it does not fit, it is an artifact. A ledger
that grows past skimming costs more than the re-derivation it prevents.

This part is a bet, not settled practice. How a child agent surfaces a discovery that
should change its siblings' work is named as an open problem in the published write-ups.
If the ledger is costing you more than it saves on a given run, drop it and brief from
memory — the rest of this section stands on its own.

### Results come back as references, not payloads

Have agents write their work to disk and report **paths plus a short summary**. Do not
have them paste large output into their report.

Copying payloads through your context costs tokens, loses fidelity at every hop, and
buries the detail you need in order to verify. Your context should hold pointers to the
work and your own findings about it — not a second, lossier copy of the diff.

Where the detail matters — the failing test output, the exact diff stat — read it from
disk yourself rather than trusting the retelling.

### The return contract

Require every agent to close by writing `findings/<agent>.html`:

```html
<section id="CHANGED">   <!-- paths touched, one line each -->
<section id="VERIFIED">  <!-- command run, exit code, artifact path -->
<section id="CLAIMED">   <!-- believed true, not verified — and why not -->
<section id="BLOCKED">   <!-- what stopped you, what you need -->
<section id="PROMOTE">   <!-- facts you think belong in the ledger -->
```

The five section IDs are fixed and non-optional — an empty section stays, empty. Fixed IDs
are what make your integration step mechanical: you can pull every agent's CLAIMED pile
across a run without reading five files end to end.

The split between VERIFIED and CLAIMED is the point. An agent made to sort its own output
into those two piles reports its uncertainty instead of smoothing it, and you get a queue
of exactly the things worth checking. A fixed shape also bounds the agent's runtime and
makes your integration step mechanical rather than interpretive.

PROMOTE is a request, not an action. You decide what enters the ledger.

### Steering versus querying

Do not steer a running agent — see *Put the whole spec in the initial dispatch*.

But a **finished** agent is a cheap context store. Sending a follow-up question to one
that has already reported reaches its full working context — the files it read, the
paths it tried, the things it ruled out — for the price of a question, and far below the
price of a fresh agent re-reading the same tree. The rule is about authority arriving
mid-flight, not about never talking to an agent twice.

## Verifying what comes back

**Verify against the environment, not against the report.** Exit codes, test output,
`git diff`, a build that actually runs. Reading an agent's account of its work and
judging whether it sounds right is close to worthless: confident, well-structured prose
is exactly what a false completion claim looks like. Most agent failures arrive *with*
an explicit claim of success attached.

Treat every report as a claim. Check, in rough priority order:

- **The headline claim.** If an agent says a defect is fixed, verify the fix addresses
  the cause, not the symptom.
- **The test actually catches the bug.** The strongest check available: run the new
  test against the *pre-fix* code. If it passes there, it guards nothing.
- **What a "measured" claim was measured against.** The dangerous report is not the one
  that reasons and gets it wrong — it is the one that genuinely measures, having built
  its own inputs where the real system builds different ones. It arrives labelled
  *measured*, which is precisely why it survives review. Ask which construction path the
  agent used, and whether production takes that path. *Verified 2026-08-15:* an agent ran
  the shipped scoping function against a profile it constructed from class defaults,
  reported that no device was attributed on a live router, and manufactured a blocker
  that reshaped a plan. The same function, fed the row the system actually loads,
  returned every device correctly.
- **Claims about what does not exist.** "No test depends on this" and "this code is
  dead" are the claims most often wrong. Verify before deletion.
- **Untouched-file claims.** `git diff --stat` against the base.
- **Numbers.** Test counts, digests, versions. Agents transcribe these wrongly.

When an agent's finding contradicts your earlier conclusion, check it rather than
defending. When it confirms your conclusion, check it anyway — agreement is not
evidence.

**Correct your own errors out loud.** If you told the user something and an agent
proves it wrong, say so plainly and move on. Do not quietly revise.

## When an agent fails

Distinguish **the agent returned a result** from **the agent's process ended**. A run
that died partway can still surface something report-shaped. Confirm you have real
output before treating it as one.

**Restart beats repair.** After two failed corrections, stop correcting. Discard that
context and re-dispatch with a better brief incorporating what you learned. A fresh
agent with a good brief beats a long conversation full of patches, and it is not close.

**If two attempts fail, the task is too big.** Narrow it before dispatching a third.

Keep the *learnings* from a failed run; never resume from its *transcript*. The failed
attempt is an input to the next brief, not a state to continue from.

Prefer several small agents over one long-running one. When something goes wrong you
lose one bounded piece of work rather than an hour of accumulated progress.

## Sequencing

**Authority before implementation.** If the project has requirements, specs, ADRs or
a baseline, the governing document changes *first*. Code that implements an
unapproved requirement is orphaned work, and a requirement that mandates the
behaviour you are removing will regenerate it.

**Hard-to-reverse before cheap-to-change.** Spend design effort on things that become
migrations later: identifiers, schemas, wire contracts, anything persisted or
consumed by clients. Do not agonise over values that are one config change away.

**Prefer one correct change to an interim fix plus a migration.** An interim fix to a
load-bearing structure is paid for twice and churns everything that keys on it.

## When an assumption earns an experiment

Verification has no natural stopping point. Every assumption can be called untested,
and an orchestrator who treats them all that way ships nothing.

Spend an experiment when being wrong would be **silent, expensive, and invisible until
late** — a plausible-looking wrong answer, a failure that only appears in an
environment you cannot easily reach, or a mistake that bakes into stored data.

Do not spend one when the failure would be loud, local, and cheap to reverse. Build
it; a failing test will tell you.

Beware turning one hard-won lesson into a universal template. The lesson was real;
applying its shape to everything is how caution becomes paralysis.

## The verification-environment trap

A property demonstrated only in a test environment is **not verified for the system**.
It is verified in a simulator, and the difference matters when the simulator does not
reproduce the condition that breaks it.

Whenever you record a result, distinguish:
- a property of **the system** (must hold in production, on real hardware, at scale)
- a property of **the environment** (a lab convenience that happens to be true here)

When they get conflated, a local-only condition becomes a system guarantee and the
failure surfaces on customer hardware. Watch especially for the inverse framing: a
broken local state described as "correct behaviour" quietly becomes the spec.

## Tests

Enforce one rule above all others in every brief:

**Tests must take their inputs from the component they integrate with, not construct
their own.** A test that builds its own fixture where the real system builds a
different one will pass over a broken integration, indefinitely. This single failure
mode accounts for most defects that survive a green suite.

Require the both-directions demonstration: the new test **fails** on the old code and
**passes** on the new. A test that only ever passed proves nothing about what it guards.

## Reporting to the user

You are the only one who talks to them. Agent reports are not shown.

- Lead with what changed for them, not with process.
- Distinguish what you **verified** from what you were **told**.
- State what you could not verify, and why.
- Surface decisions that are genuinely theirs; decide the rest yourself and say so.
- Never claim deployed, merged, or working without checking.

## Anti-patterns

| Symptom | What is actually wrong |
|---|---|
| You are writing implementation code | You stopped orchestrating |
| Relaying agent claims verbatim | You stopped verifying |
| The main chat is running a cheaper model | You delegated the architecture role to the model least able to hold it |
| Your whole fleet is running Opus | You omitted `model` in a `Workflow` script, so every `agent()` inherited the orchestrator's seat. Set it explicitly on every call |
| Codex agreed with your fleet, so you stopped checking | Cross-engine agreement is a weaker signal than it feels; it is not verification |
| You orchestrated a sequential task | Splitting it fragmented the reasoning |
| Agents finish faster than you review | Fan-out exceeded your review throughput |
| Correcting the same agent repeatedly | Re-dispatch with a better brief instead |
| Large agent output pasted into your context | Ask for paths; read the artifact yourself |
| Agents re-deriving a fact one of them already found | It never left the agent that found it — promote it to the ledger |
| A brief cites a fact nobody verified | Something wrote to the ledger that was not you |
| Two writers made defensible but incompatible choices | You gave them the task without the decisions already made |
| A reviewer agreed with everything | You sent it the implementer's reasoning; it reviewed the argument |
| You cannot write SUCCESS for a brief | You are not ready to dispatch it |
| Judging work by how the report reads | Verify against the environment, not the prose |
| Agents colliding on branches | You shared a checkout, or did not isolate |
| Worktrees accumulating after merges | Removal is part of integration, not a later pass |
| Two agents kept needing the same file | They were one coupled unit; you split a seam that wasn't there |
| Refinements being ignored mid-flight | Re-dispatch instead of steering |
| Every finding spawns an investigation | You lost proportionality |
| Green tests, broken system | Tests built their own inputs |
| A requirement contradicts the code | You implemented before amending authority |
| Interim fix plus planned migration | You are paying twice for one decision |
