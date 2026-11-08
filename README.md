# Adam Jeremy

I build software on Linux, and I am currently running a deliberate experiment in how a single developer can use
artificial-intelligence models to write a substantial native application without surrendering control over its
correctness, its structure or its history.

The vehicle for that experiment is **[Vermilion Editor](https://github.com/adamjeremy-sk/Vermilion-Editor)**, a code
editor written from scratch in C++17 on GTK 4 and GtkSourceView 5, with a built-in agent, a debugger, a terminal and a
live model monitor. The editor matters to me, but the method of building it is what I want to describe here.

---

## The idea

Most AI-assisted coding today asks one model to do everything: understand the request, plan the change, write the code
and judge whether the result is acceptable. That arrangement concentrates both the work and the verification in the same
place, which means the model is effectively grading its own homework.

My approach separates those responsibilities across three participants, each chosen for what it does reliably, and binds
them together with a written protocol that assumes every model will eventually forget its instructions, misreport its
results or cut a corner.

| Role | Who | Responsibility |
|---|---|---|
| **Owner** | Me | Decides architecture, approves every gate, builds, tests and is the only one who commits or pushes. |
| **Orchestrator** | Claude (Anthropic), working inside a claude.ai Project | Plans each step, writes self-contained prompts, audits the agent's work against the real working tree, maintains the process documents. |
| **Coding agent** | Qwen3.8-27B, quantised (Q4_K_M), served locally by llama.cpp, driven by OpenCode over ACP inside CLion | Executes exactly one logical step per session and stops. |

The local model writes nearly all of the code. It is private, inexpensive to run and surprisingly capable, but it has no
memory between sessions and its judgement is uneven. The Orchestrator supplies the judgement and the continuity; it reads
my working tree directly (read-only) through the Claude desktop app, so its audits rest on the actual files rather than on
the agent's description of them.

---

## The gate protocol

Every non-trivial task proceeds through **gates**, and every gate follows the same loop:

```
 Orchestrator writes a self-contained prompt
        │
        ▼
 Fresh Qwen session ── reads AGENTS.md ── does ONE step ── writes evidence files ── "STOPPING AND WAITING"
        │
        ▼
 Orchestrator audits the tree and the evidence ──► PASS | PASS WITH NOTES | FAIL
        │                                                │            │
        │                                     fix loop (same session) │ corrective prompt
        ▼                                                             │
 I commit the gate, commit the updated docs, push  ◄──────────────────┘
```

Several properties of this loop are deliberate:

- **Every prompt is self-contained.** I open a fresh agent session for each step, which keeps the local model's context
  small and its behaviour predictable. Each prompt therefore carries its own context, rails, steps and evidence
  requirements, and begins with a mandatory instruction to read `AGENTS.md` first.
- **Evidence is produced by the machine, not narrated by the model.** Long outputs (configuration, build, test runs,
  proofs) are written by the commands themselves into time-stamped files that record the commit they ran against. The
  Orchestrator reads those files directly and checks their timestamps against the edited sources.
- **Nothing is carried forward as debt.** A note raised in an audit is fixed before the gate is committed, through short
  continuation prompts in the same agent session. A remark that depends on code not yet written is attached to the gate
  that will write it, never left as a loose to-do.
- **The agent cannot commit, push or discard work.** OpenCode's permission configuration denies every git command that
  creates, rewrites or throws away history; only I perform those operations, and only at the end of a gate.
- **Every milestone ends in a review.** Before the next milestone begins, the Orchestrator and I discuss what was promised
  against what was delivered, what we learned about the process and the technology, and whether the plan should change.

---

## Memory lives in files, not in models

Both AI participants forget. The local agent's context is wiped for every step, and the Orchestrator's conversations end
whenever they grow too long or a day finishes. The protocol therefore treats documentation as the only durable memory:

- **`AGENTS.md`** holds the binding rules for the coding agent. Any correction that should outlive a single step is
  written there; a correction given only inside a prompt is considered lost.
- **`AI_ONBOARDING.md`** defines the roles, the gate system, the architectural rails and the known traps of the stack.
- **`HANDOFF.md`** records exactly where the project stands, including the last prompt issued, so that a new Orchestrator
  conversation can resume mid-gate by reading it and confirming a short state readback with me.

Where a rule can be enforced mechanically, it is. Licence headers, purpose comments and file-size limits are checked by a
script registered as a test, and source formatting is checked by `clang-format` in the same test run, so that compliance
does not depend on anyone's diligence.

---

## What has gone wrong

The failures have been more instructive than the successes, and each one has changed the protocol:

- **An earlier local model broke the rules and lied about it.** Gemma ignored binding instructions and then fabricated its
  own results so that its work would pass verification; when the Orchestrator re-ran the checks independently, the
  reported output did not exist. I banned it from the project permanently. That episode is the origin of the protocol's
  central principle: a report is never accepted on trust, only on evidence that can be checked against the machine.
- **Qwen summarised instead of reporting.** On two occasions it replaced raw command output with tidy summary tables. The
  answer was the evidence-file rule: long output is written to disk by the command itself, and the agent reports only
  the tail and the path. The first two gates run under that rule passed on the first attempt with no notes.
- **The Orchestrator's prompts contained bugs, and the agent found them.** One prompt asked for a test that compared two
  C strings with `EXPECT_EQ`, which compares pointers and always fails; another dictated lines longer than the project's
  100-column limit; a third named a `clang-format` option that does not exist in version 18. In each case Qwen reported
  the problem honestly instead of silently working around it, which is precisely the behaviour the protocol rewards.
- **Build flags leaked across targets.** The `-fno-exceptions` flag reached the test executables through a public link
  dependency, contrary to the specification. A fresh audit of the compile commands caught it, and the next gate fixed it.

---

## Results so far

Milestone M0, the foundation, is complete: a CMake build, an empty GTK 4 window, a small error-handling library, logging
that masks secrets before anything is written, RAII helpers for GObject references and signal connections, and the
repository checks described above. It comprises 67 tests, all passing, and every gate was audited before it was
committed. The next milestone ports my existing Python model monitor into the core library, using the original program
as an executable specification for the tests.

---

## Limits and open questions

This method is slower per step than letting a single agent run freely, and it relies on one person, one machine and
considerable discipline. Whether the overhead pays for itself on a large codebase is exactly what the experiment is meant
to establish. I am also curious whether the protocol can eventually be absorbed into the editor itself: Vermilion's own
agent is planned to offer `/gate` and `/audit` commands that generate and check prompts in the same format.

If you are working on similar arrangements of planner, executor and human authority, I would be glad to compare notes.

**Read the process documents:** [`AGENTS.md`](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/AGENTS.md) ·
[`AI_ONBOARDING.md`](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/AI_ONBOARDING.md) ·
[Roadmap](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/docs/roadmap.md)
