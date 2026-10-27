---
name: prompt-loop
description: Takes a prompt the user already wrote (pasted, attached, or a file path) and wraps it in the iterative loop that fits it best — coverage over a work list, test-and-fix, evidence gathering, draft-critique-revise, or prompt tuning — with checks and stop conditions derived from the prompt itself, then runs it (or, on request, returns the prompt with a portable loop protocol attached). Use whenever the user wants to "loop", "iterate", "run until done", "keep going until it's right", "make this prompt run in a loop", or hands over a ready prompt and asks for it to be executed thoroughly or self-checked — even if they don't say "loop". Works in Claude Code and Codex.
---

# Prompt loop

The user brings a prompt they already wrote. Your job is to decide **how to iterate on it** — not to rewrite what it asks for. A good loop for a prompt is one where every check traces back to something the prompt says, every round leaves visible progress, and the loop stops for a reason you can name.

There is no single best loop. A 20-item audit, a failing test suite, a reverse-engineering question and a blog post each need a different one, and forcing one shape onto all of them is the main way this goes wrong. So most of the work is the analysis in Step 2.

## Step 1 — Get the prompt and the mode

- The prompt is whatever the user pasted, attached or pointed to. Read all of it before deciding anything, including the parts that look like boilerplate: stop rules and "do not" lines usually live there.
- **Mode.** Default is **run**: design the loop, show the loop card, execute. Switch to **plan-only** when the user asks for the plan / the protocol / "don't run it", says it will run somewhere else (another session, the other tool), or when the files, tools or machine the prompt depends on aren't reachable from here — in that last case say so in one line.

## Step 2 — Analyze the prompt

Answer these from the prompt's own text. Keep notes short; they feed the loop card.

1. **Deliverable.** What exactly must exist at the end, and where (file paths, a report, a code change, an answer)?
2. **Scope fence.** What it says to do *and stop at* ("do this stage and stop", "stage 0c is later"). The loop never continues past it.
3. **Hard rules.** Every "never", "only", "do not", "read-only", "ask before". These become invariants, checked at the end where checkable (for example: the tree is still clean, no file written outside the allowed folder).
4. **Done criteria.** Every "each", "every", "cover all", "at most N lines", "with X for each claim", required sections. These become the checklist.
5. **Its own stop rules.** "If X, stop and report", "record as a stop item and continue", "halt only when". These override any stop rule you'd add.
6. **Work structure.** Does it enumerate units (a list of files, drivers, endpoints, chapters) or imply a set to cover? That's an outer coverage loop.
7. **How correctness is established per unit.**
   - Executable check exists or can be written (tests, build, lint, a script, a diff) → verify-fix.
   - Facts have to be established under uncertainty (research, reverse engineering, debugging, analysis with confidence labels) → evidence.
   - Quality is a judgment (writing, design, tone) → draft-critique-revise.
   - The thing being improved is the prompt itself → prompt-tuning.
   - Small, closed, and obviously checkable → one pass and a check; no loop.
8. **Cost and risk of a round.** Cheap and reversible → loop freely. Expensive, slow, billed, or with side effects (sends, deploys, deletes, commits) → fewer rounds, and anything irreversible the prompt didn't already authorize goes to the user first.

If the prompt is contradictory in a way that changes what you'd build (two rules that can't both hold), stop and ask before running. Ambiguity you can resolve with a reasonable reading is not a reason to stop — pick the reading and state it on the card.

## Step 3 — Choose the loop shape

Read `references/loop-shapes.md` for the full description of each shape, its round structure, budget and failure modes. In short:

| Shape | Fits when | One round is |
|---|---|---|
| **one-pass** | small, closed, checkable | do it → check once → fix once |
| **verify-fix** | an executable check decides | run check → fix the first real failure → rerun |
| **coverage** | the work is a list of units | take next unit → do it → mark its status in the ledger |
| **evidence** | facts must be established, some may stay unknown | pick open question → gather evidence → calibrate → label confidence |
| **critique-revise** | quality is judged | critique against a rubric from the prompt → revise the worst problems |
| **prompt-tuning** | the deliverable is a better prompt | run prompt on samples → find failures → edit prompt |

Real prompts are often **composites**: an outer coverage loop over units with an inner verify-fix or evidence loop per unit, followed by one final audit. Name the composite explicitly ("coverage over 18 drivers × evidence per driver → final audit"). Don't invent a composite when one shape covers it.

Pick the simplest shape that covers the prompt. A loop that does more rounds than the work needs wastes the user's usage and buries the result.

**If the shape is one-pass, stop designing here:** do the task, check it once, and give the result with at most one line saying no loop was needed. No loop card, no ledger, no audit — ceremony around a one-sentence task is noise.

## Step 3b — Decide where the loop lives

The shape says *how* to iterate; this says *who keeps the loop going*. Choose one:

- **In-session (default).** The agent runs every round in one conversation. Simplest for the user, and the agent remembers everything it saw. Use it whenever the whole job comfortably fits in one session.
- **External loop (only when really needed).** A small driver script calls the agent repeatedly (`codex exec`, `claude -p`), one fresh session per unit, with progress kept in a state file. Each unit starts with a clean context, so nothing is lost to compaction and a crash resumes where it stopped. The cost is more moving parts and re-reading the basic context every session. Use it only when at least one holds:
  - the work clearly won't fit in one session: many units that each need heavy reading (disassembly, large files, long docs), or a run of hours;
  - the prompt's rules are long and losing one mid-run would be expensive (a compacted summary drops details);
  - the user wants to inspect results after the first units before letting the rest run.
  A long prompt or a long list alone is not enough — 20 small units fit in one session.
- **Gate (add to either).** When the known failure mode is the agent declaring done too early and an executable check exists, put the check *outside* the agent's control: a verify script the loop can't skip. Portable form: the driver script reruns the agent until the verify script passes or the budget ends. In Claude Code a Stop hook can do the same inside one session. Use a gate when the user mentions early victory, or when the prompt's done criteria are mechanical and the cost of a false "done" is high.

When you choose external or gate, read `references/external-loop.md` and generate its files. Say on the card why the in-session default wasn't enough, in one line.

## Step 4 — Build the checklist from the prompt

Go through the prompt line by line and turn every requirement from Step 2 into a check. For each check note:

- the line or phrase it comes from (so the user can see it isn't invented),
- its kind: `done` (must be true at the end), `invariant` (must never be violated), `stop` (a condition that ends the loop or a branch), `format` (shape of the output),
- how it is verified: a command, opening the file, counting, or a judgment.

Checks you add that the prompt doesn't state (for example "no claim without a source") are allowed when they serve the prompt's evident goal — mark them `(added)` so the user can veto them. Don't add checks that change what the prompt asks for.

## Step 5 — Set budgets and stop conditions

The loop ends on exactly one of these, and the final report says which:

1. **Done** — every `done` check passes and no invariant was broken.
2. **Prompt stop rule fired** — do what the prompt says for that case (usually report and stop).
3. **Budget reached** — default budgets are in `references/loop-shapes.md`; tighten them when rounds are expensive.
4. **No progress** — the same check failed twice in a row with the same approach: change the approach. Failed a third time: mark that item blocked or Unknown with the reason and move on (or stop, if the prompt says a failure there ends the run). Repeating an approach that already failed is the most common way loops burn time.

## Step 6 — Show the loop card, then run

Show this card before the first round (short; this is not a plan waiting for approval):

```
Loop: <shape or composite>
Runs: <in-session | external (why) | + gate (what it checks)>
Why: <one line tying the shape to the prompt>
Unit: <what one round handles>
Checks: <n done · n invariant · n stop · n format> — top 3–5 listed, (added) marked
Stops when: <done / prompt stop rules / budget>
Ledger: <where progress is kept>
Reading taken: <only if you resolved an ambiguity>
```

Then start immediately. Ask first only if the prompt itself requires confirmation, or the loop would take an irreversible step the prompt didn't authorize. For an external loop, "start" means generating its files; launching the driver script is a long, billed run, so ask before you launch it.

**Ledger.** Keep a small progress record: units and their status, the checklist with pass/fail, one line per round (what changed, what's next), open questions. Put it on disk when the prompt allows writing somewhere (inside the folder it designates for notes or work), otherwise keep it in the harness's plan/todo tool or in the conversation. A ledger on disk survives context compaction and session restarts, so for long runs it's what lets the loop resume without redoing work. Never put the ledger where the prompt forbids writes.

**Each round:** do the work for one unit or one failing check → verify it the way Step 4 says (run the command, open the file — don't verify from memory of what you intended) → update the ledger → decide: next unit, retry with a different approach, or stop.

Keep narration between rounds to a minimum; the ledger and the plan tool show progress.

## Step 7 — Final audit

Before reporting, re-read the original prompt from the top as if seeing it for the first time and check every item of the checklist against the actual outputs. When the harness can spawn a subagent (Claude Code's Agent/Task tool, a Codex subagent), give one the original prompt, the checklist and the output paths and have it audit independently — it hasn't seen your reasoning, so it catches what you'd rationalize. Otherwise do the cold re-read yourself. Fix what fails if budget allows; otherwise report it as open.

Verify invariants concretely where possible (e.g. `git status` still clean, no new files outside the allowed folder).

## Step 8 — Report

- If the prompt defines a report (length, language, sections), that format wins. Loop information goes inside it only where it fits — never break the prompt's limits to describe the loop.
- Otherwise report: the outcome, which stop condition ended the loop, rounds used, checks passed/failed, what's open, files written.
- Talk to the user in their language; write deliverables in the language the prompt asks for.

## Plan-only mode

For an **in-session** loop: output the loop card, then a block titled **Loop protocol** written as imperative instructions an agent can follow *without this skill* — paste-ready, appended after the original prompt (don't edit the original text). It contains: the shape and round structure, the checklist, budgets and stop conditions, the ledger location and format, the no-progress rule, and the final audit.

For an **external loop or gate**: output the loop card and the files from `references/external-loop.md` (iteration prompt, state file, driver script, verify script when there's a gate), plus two or three lines on how to start it and when to look at results.

Keep it lean. The protocol is read *together with* the original prompt, so refer to the prompt's sections instead of restating them ("the per-driver questions in item 1", not a copy of item 1). Restate a rule only when the loop changes how it's applied. For a long prompt, aim for well under half its length; a protocol close to the prompt's size is usually duplicating it. A short prompt (a few lines) leaves the loop to supply the structure, so a protocol longer than the prompt is fine there — just keep it to what the loop adds.

## Platform notes

- **Claude Code:** plan/todo tool for the ledger mirror; Agent/Task tool for the independent audit and, for coverage loops over independent units, for parallel units when the prompt allows it.
- **Codex:** `update_plan` (or its current plan tool) mirrors the ledger; use subagents for the audit when available, otherwise the cold re-read.
- **Chat without a shell:** loops still work; the ledger lives in the conversation and checks are done by reading.
- Use the tool names the harness actually exposes; don't assume either set exists.
