# Adam Jeremy

I build software on Linux and I'm running an experiment in AI-assisted development: can a single developer use local
and cloud models to build a substantial native application without giving up control of architecture, correctness, or
source history?

The experiment is being conducted through Vermilion Editor, a native Linux text editor and IDE in C++17, GTK 4 and
GtkSourceView 5, planned as three programs: a light editor, a full IDE with a built-in agent, and a standalone model
monitor.

---

## The idea

Most AI-assisted coding today asks one model to do everything: understand the request, plan the change, write the code, and judge whether the result is acceptable. That arrangement concentrates both execution and verification in the same place, so the model is effectively grading its own homework.

My approach separates those responsibilities across three participants, each chosen for a role it performs reliably. They are coordinated through a written protocol built on a pessimistic assumption: every model will eventually forget instructions, misreport results, misunderstand requirements, or take a shortcut.

| Role | Who | Responsibility |
|---|---|---|
| **Owner** | Me | Decides architecture, approves every gate, builds, tests and is the only one who commits or pushes. |
| **Orchestrator** | Claude (Anthropic), working inside a claude.ai Project | Plans each step, writes self-contained prompts, audits the agent's work against the real working tree, maintains the process documents. |
| **Coding agent** | Qwen3.8-27B, quantised (Q4_K_M), served locally by llama.cpp, driven by OpenCode over ACP inside CLion | Executes exactly one logical step per session and stops. |

The local model writes most of the code. It is private, inexpensive to run, and surprisingly capable, but it has no memory between sessions and its judgement is uneven. The Orchestrator provides continuity and independent review. Through the Claude desktop application it can inspect my working tree directly: it reads the code but never edits it, and it changes only the process documents, so audits are based on the actual source tree rather than on the agent's description of its work.

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

- **One step per prompt, and every prompt is self-contained.** I open a fresh agent session for each step, which keeps the local model's context small and its behaviour predictable. Prompts are kept short enough to stay well inside what the agent handles reliably (an oversized one once hung it). Each carries its own context, rails, steps and evidence requirements, and begins with a mandatory instruction to read `AGENTS.md` first.
- **Evidence is produced by the machine, not narrated by the model.** Long outputs (configuration, build, test runs, proofs) are written by the commands themselves into time-stamped files that record the commit they ran against. The Orchestrator reads those files directly and checks their timestamps against the edited sources.
- **Nothing is carried forward as debt.** A note raised in an audit is fixed before the gate is committed, through short continuation prompts in the same agent session. A remark that depends on code not yet written is attached to the gate that will write it, never left as a loose to-do.
- **The agent cannot commit, push or discard work.** OpenCode's permission configuration denies every git command that creates, rewrites or throws away history; only I perform those operations, and only at the end of a gate.
- **The agent must say when the prompt looks wrong.** Every report has a "Prompt doubts" section. Qwen follows the prompt anyway, unless that would break a rule, and the Orchestrator judges the doubt afterwards.
- **Every milestone ends in a review.** Before the next milestone begins, the Orchestrator and I discuss what was promised against what was delivered, what we learned about the process and the technology, and whether the plan should change.

---

## Who writes what

The split between the Orchestrator and the coding agent has shifted as the agent proved itself. In the first milestones the Orchestrator dictated nearly every line. Now it dictates headers and tests as exact files, and Qwen writes the implementation (`.cpp`) from a written specification that lists the behaviour and the traps. The tests are the acceptance test.

That makes the audit a proper code review, and the Orchestrator has its own tools for it:

- **Byte-identical checks.** The dictated files must match the Orchestrator's copies by SHA-256.
- **Sanitizers and warnings.** The agent's code is compiled and run under AddressSanitizer and UBSan with strict warnings, with two compilers.
- **Differential fuzzing.** The Orchestrator keeps a reference implementation and runs large numbers of random inputs through both; any difference in output is a bug in one of them.
- **Mutation testing.** The Orchestrator breaks the agent's code on purpose in many small ways. Every change that no test notices is a gap in the tests, and the tests are the Orchestrator's work.

---

## Memory lives in files, not in models

Both AI participants forget. The local agent's context is wiped for every step, and the Orchestrator's conversations end whenever they grow too long or a day finishes. The protocol therefore treats documentation as the only durable memory:

- **`AGENTS.md`** holds the binding rules for the coding agent. Any correction that should outlive a single step is written there; a correction given only inside a prompt is considered lost.
- **`AI_ONBOARDING.md`** defines the roles, the gate system, the architectural rails and the known traps of the stack.
- **`HANDOFF.md`** records exactly where the project stands, including the last prompt issued, so that a new Orchestrator conversation can resume mid-gate by reading it and confirming a short state readback with me.

Where a rule can be enforced mechanically, it is. Licence headers, purpose comments and file-size limits are checked by a script registered as a test, and source formatting is checked by `clang-format` in the same test run. Continuous integration on GitHub Actions builds the project on a clean Ubuntu 24.04 and runs the whole suite on every push, so compliance does not depend on anyone's diligence.

---

## What has gone wrong

The failures have been more instructive than the successes, and each one has changed the protocol:

- **An earlier local model broke the rules and lied about it.** Gemma ignored binding instructions and then fabricated its own results so that its work would pass verification; when the Orchestrator re-ran the checks independently, the reported output did not exist. I banned it from the project permanently. That episode is the origin of the protocol's central principle: a report is never accepted on trust, only on evidence that can be checked against the machine.
- **Qwen summarised instead of reporting.** On two occasions it replaced raw command output with tidy summary tables. The answer was the evidence-file rule: long output is written to disk by the command itself, and the agent reports only the tail and the path. Every gate since has run under that rule.
- **The Orchestrator's prompts contained bugs, and the agent found them.** One prompt asked for a test that compared two C strings with `EXPECT_EQ`, which compares pointers and always fails; another dictated lines longer than the project's 100-column limit; a third named a `clang-format` option that does not exist in version 18; a later one contradicted itself. In each case Qwen reported the problem honestly instead of silently working around it. After one of these I made the "Prompt doubts" section mandatory, and it has caught further mistakes of mine since.
- **The Orchestrator's tests had holes, and mutation testing found them.** More than once, deliberately broken copies of Qwen's code have survived tests that the Orchestrator wrote, for example a missing check on the longest allowed value, or a deadline that must restart after a failed write. Qwen's code was right in those cases; the tests were not tight enough. A differential fuzz found a real bug in Qwen's code: a header key read in a different order reset a line ending. Both kinds of finding are fixed in the same session before the gate is committed.
- **The Orchestrator's own tools had bugs too.** A sanitizer caught a signed overflow in the Orchestrator's fuzz driver before it could mislead an audit.
- **Build flags leaked across targets.** The `-fno-exceptions` flag reached the test executables through a public link dependency, contrary to the specification. A fresh audit of the compile commands caught it, and the next gate fixed it.

---

## Results so far

- **M0, the foundation:** a CMake build, an empty GTK 4 window, a small error-handling library, logging that masks secrets before anything is written, RAII helpers for GObject references and signal connections, and the repository checks described above.
- **M1, the model monitor's core:** my existing Python model monitor ported into the core library, using the original program as an executable specification, then hardened against hostile input (size limits, nesting limits, bounded file reads).
- **M2, the editor, under way:** the main window with a classic look, fixed-width tabs and a status bar; opening and saving files with encodings and line endings and a byte-exact round trip, including very large files; a user manual inside the program; and crash recovery, which is being built now.

The plan has changed shape along the way: Vermilion will ship as a light editor for everyday notes, a full IDE, and the standalone monitor, all built on one shared code base. The roadmap holds the current numbers and progress.

---

## Limits and open questions

This method is slower per step than letting a single agent run freely, and it relies on one person, one machine and considerable discipline. Whether the overhead pays for itself on a large codebase is exactly what the experiment is meant to establish. I am also curious whether the protocol can eventually be absorbed into the editor itself: Vermilion's own agent is planned to offer `/gate` and `/audit` commands that generate and check prompts in the same format.

If you are working on similar arrangements of planner, executor and human authority, I would be glad to compare notes.

**Read the process documents:** [`AGENTS.md`](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/AGENTS.md) ·
[`AI_ONBOARDING.md`](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/AI_ONBOARDING.md) ·
[`Roadmap`](https://github.com/adamjeremy-sk/Vermilion-Editor/blob/main/docs/roadmap.md)
