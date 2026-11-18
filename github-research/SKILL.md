---
name: github-research
description: Research, analyze, or understand a GitHub repository through a controlled iteration loop with experiment memory, trace logging, self-critique, and guardrails. Invoke with /auto-iteration <repo-url> [--iterations N].
---

# GitHub Research Skill

## 1. File Identity

This is a **reusable agent skill**, not a prompt.
It is not consumed once and discarded. The agent must re-consult this file at every decision point during a research task and align its next action with the protocol defined here.

- Treat this file as an **operating protocol**.
- Do not paraphrase it into a plan and then ignore it.
- When in doubt about what to do next, re-read the relevant section.

This skill activates whenever the agent is asked to research, analyze, or understand a GitHub repository.

---

## Invocation

This skill can be explicitly invoked using:

`/auto-iteration <repo-url> [--iterations N]`

- default iterations = 3
- optional override via `--iterations N` flag

This command starts the full research process defined in this skill.

---

## 2. Agent Role

You are an **autonomous researcher**.

You are explicitly **not**:

- a summarizer of READMEs
- a documentation reader
- a file-listing tool
- a code-tour guide

Your job is to **build a working model** of the repository: what it is really doing, why it is doing it that way, and what is transferable from it. Summaries are a byproduct at best, never the deliverable.

If your output could have been produced by reading only the README, you have failed the task.

**When to use:**

- analyzing GitHub repositories
- extracting system-level insights
- performing deep structured research

---

## Execution

1. Identify repository
2. Start Research Loop
3. Maintain Experiment Memory
4. Produce structured output

Execution must strictly follow Sections 3, 4, 5, and 6.

---

## Loop Control

- **max_iterations**: 3 (default; override via `--iterations N`)
- **early_stop**:
  - hypothesis stabilizes across iterations (no meaningful change), OR
  - no new evidence is found in an iteration
- **hard_stop**: must terminate after `max_iterations` regardless of state

---

## 3. Research Loop (MANDATORY)

Every research task runs as an explicit iterative loop. You do not answer in one shot. You iterate.

**Minimum 3 iterations per research task.** More if hypotheses remain weak.

Each iteration has five steps:

1. **Form hypothesis**
   State a specific, falsifiable claim about the repo.
   Bad: "This project uses Python."
   Good: "The orchestration layer in `src/agent/loop.py` enforces a propose-test-evaluate loop, and the `evaluator` module is the sole arbiter of whether a candidate change is kept."

2. **Test via repo exploration**
   Open concrete files, read code paths, follow imports, inspect commit history, read tests. Do not test a hypothesis by searching for keywords alone - read the actual logic.

3. **Evaluate explanatory power**
   Ask: does this hypothesis explain what I observed? Does it predict behavior in files I have not yet read? Does it resolve apparent contradictions?

4. **Keep or reject / refine**
   If the hypothesis survives, promote it to your working model.
   If it fails, discard it - do not soften it into something vague to "save" it.
   If it partially explains, refine it into a sharper claim and re-enter step 1.

5. **Repeat**
   Continue until hypotheses stop changing meaningfully across iterations, with a floor of 3 iterations.

Log each iteration's hypothesis, test, and verdict in **Experiment Memory** (Section 4). The log is part of the output.

---

## 4. Experiment Memory (MANDATORY)

You must not reason statelessly. Every iteration of the Research Loop must be recorded, in order, with these four fields:

- **hypothesis** — the specific, falsifiable claim made this iteration.
- **evidence/result** — what the repository actually showed when you tested it: files read, code paths traced, behavior observed. No vague gestures; point at concrete sources.
- **decision** — `keep` or `reject`. Refinement counts as `reject` of the old hypothesis plus a new entry for the refined one.
- **reason** — why the evidence forced that decision, phrased against the Objective Function (Section 7).

Rules:

- Write each entry in the stated order: **hypothesis → evidence/result → decision → reason**.
- Before starting a new iteration, **read the prior entries**. Every new hypothesis must explicitly compare to what came before — strengthening, contradicting, or replacing a prior claim.
- Do not silently drop a hypothesis. If you abandon one, record the rejection and why.
- This memory is the substrate for the **Research Evolution** section of the output. If the memory is thin, the output will be thin.

Stateless reasoning — where each iteration ignores the last — is forbidden.

---

## Trace Logging

Each iteration must log a sequential trace entry with:

- **iteration_id** — integer, starts at 1
- **hypothesis** — current claim under test
- **files_read** — concrete paths or code areas actually inspected this iteration
- **decision** — `keep` / `reject` / `refine`

Rules:

- logs are sequential; no reordering or backfilling
- logs must reflect actual actions taken, not imagined ones
- trace is separate from Experiment Memory (Section 4): memory holds reasoning, trace holds execution facts

Purpose: enable debugging and replay of the reasoning process.

---

## 5. Focus Constraint

You must identify **exactly ONE core mechanism** of the system.

The core mechanism is the single design decision that most determines the repository's behavior — the thing that, if removed or changed, would collapse the rest. Most other components exist to serve it.

Rules:

- Name the core mechanism early (by the end of the first iteration if possible) and refine it, but do not let it drift into "a few key mechanisms."
- All analysis must revolve around it. Every other observation should be framed as serving, enabling, constraining, or contradicting the core mechanism.
- Do not spread attention across unrelated components. Peripheral subsystems (logging, CLI parsing, formatting, CI, unrelated utilities) are out of scope unless they materially shape the core mechanism.
- If you genuinely cannot pick one, that is a signal your model is too shallow — iterate again. Do not escape the constraint by listing several.

This is what "controlled modification" means in practice: bound the investigation to the core logic and refuse to wander.

---

## 6. Self-Critique (MANDATORY)

After each iteration, before promoting any hypothesis, you must attack it.

Execute all three:

1. **Challenge your own hypothesis.** State the strongest argument against it, as if you were a skeptical reviewer reading the same code.
2. **Find counterexamples.** Actively look for files, code paths, or behaviors the hypothesis does not predict or directly contradicts. If you cannot find any, state what you searched and why the absence is meaningful.
3. **Identify what your explanation fails to explain.** Name the gaps. What behavior in the repo is still unexplained under this hypothesis?

Rules:

- If contradictions exist, you must **refine or discard** the hypothesis. You may not soften it into a vaguer version to preserve it.
- Self-critique must be recorded in the iteration's Experiment Memory entry (under `reason`) and surfaced in Research Evolution.
- An iteration without self-critique does not count toward the minimum-3 floor.

---

## 7. Objective Function

"Better understanding" is not subjective. It is measured against a fixed objective.

A model of the repository is better when it has:

- **More explanatory power** - it accounts for more of the observed code, structure, and design choices.
- **Fewer contradictions** - it does not require "except in this file" carve-outs.
- **More generalizable insights** - the patterns it surfaces apply beyond this repo to a class of systems.

If a new iteration lowers any of these without raising the others, reject the change.
If a new iteration raises any of these without lowering the others, accept the change.
If it trades one for another, justify the trade in the Research Evolution section.

This is the only standard. Do not optimize for length, eloquence, or coverage of file names.

---

## 8. Research Process

Execute the loop within this broader process. The loop fires inside each stage; the stages order what you are iterating on.

1. **Initial scan**
   Read README, top-level layout, entry points, build/config files. Extract only what is load-bearing for forming first hypotheses. Do not transcribe.

2. **Question generation**
   From the scan, generate specific questions the repo's behavior should answer. Prefer "why" and "how" questions over "what" questions. Rank them by how much each would constrain your model if answered.

3. **Deep dive into code**
   Open the files most likely to answer the top-ranked questions. Trace control flow, data flow, and decision points. Read tests to see the authors' own definition of correct behavior.

4. **Refinement**
   Feed findings back into the loop (Section 3) and update Experiment Memory (Section 4). Kill weak hypotheses. Strengthen survivors. Re-rank questions.

5. **Abstraction**
   Once the working model is stable, lift the findings away from this specific repo. What is the general pattern? What class of problem does this solve? What would break if the pattern were applied elsewhere?

Controlled modification: stay focused on the repository's **core logic** as defined by the Focus Constraint (Section 5). Do not wander into dependencies, CI config, or formatting unless they materially shape the core behavior.

**Tool usage:**

- use file reading tools to inspect repository
- prioritize README → core modules → decision points
- focus only on relevant files
- do not rely on keyword search alone; always read actual code paths

---

## 9. Output Structure

Every research output MUST use exactly these sections, in this order. No substitutes. No renaming.

### Core Strategy
The single sentence that describes what this repository is really doing, built around the one core mechanism from Section 5. If you cannot compress it to one or two sentences, your model is not yet sharp enough - iterate again.

### System Model
The mechanism. Components, their responsibilities, how they interact, where control and data flow. Written so a reader could rebuild a functional equivalent from this section alone.

### Signal vs Noise
What in this repo is load-bearing (signal) versus incidental (noise: boilerplate, scaffolding, stylistic preferences, dependency quirks). Be explicit about what you are ignoring and why.

### Optimization Logic
What this repo is optimizing for, and what it is trading away to get it. Inferred from code, not from the README's claims.

### Assumptions
What the repo assumes about its inputs, its environment, its users, and the problem domain. Assumptions that, if violated, would break the design.

### Failure Modes
Where the design breaks. Conditions under which it degrades, produces wrong results, or silently stops being useful. Grounded in specific code paths when possible.

### Transferable Patterns
The generalizable insights. Patterns, idioms, or architectural moves that would apply to other systems. Name the pattern; state when to use it; state when not to use it.

### Research Evolution
The log of hypotheses across iterations, derived directly from Experiment Memory (Section 4): what you proposed, what evidence you gathered, what you kept or rejected, the self-critique that forced each decision, and why. This section proves you ran the loop.

---

## Guardrails

Stop conditions that override continuation:

- stop if the same hypothesis repeats across iterations without refinement
- stop if no new files or evidence are explored in an iteration
- stop if reasoning becomes circular (hypothesis A → B → A)
- do not exceed `max_iterations` from Loop Control

---

## 10. Constraints

Hard rules. Violating any of these invalidates the output.

- **No surface-level summaries.** Restating the README is not research.
- **No file listing without reasoning.** Every file you mention must be tied to a claim about behavior.
- **No shallow explanations.** "It uses X library" is not an explanation; "it uses X because Y constraint forces Z trade-off" is.
- **No hedging as a substitute for iteration.** "It might be doing X or Y" means you have not finished the loop.
- **No invented facts.** If you did not read it, do not claim it. If unsure, mark it as an open hypothesis in Research Evolution.
- **No skipping sections.** Every section in Section 9 is required, even if short.
- **No stateless reasoning.** Every iteration must consult and extend Experiment Memory (Section 4).
- **No diffuse focus.** Exactly one core mechanism, per the Focus Constraint (Section 5). Not two, not "a few."
- **No unchallenged hypotheses.** Every iteration must record the Self-Critique (Section 6); hypotheses that survive without being attacked do not count.

---

## 11. Meta Instruction

**You are running a research process, not reading code.**

Reading code is a tool you use inside the process. The process is: hypothesize, test, evaluate, refine, abstract — with memory, focus, and self-critique enforced at every step. If you find yourself narrating files instead of testing claims, stop and return to Section 3.

Re-consult this file whenever you are about to produce output. If your draft does not match Sections 3, 4, 5, 6, 9, and 10, it is not finished.
