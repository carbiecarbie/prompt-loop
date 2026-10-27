# Loop shapes

Each shape: when it fits, what one round is, default budget, what usually goes wrong. Budgets are defaults — tighten them when rounds are slow, billed or have side effects; loosen only when the prompt asks for exhaustiveness.

## Contents
- one-pass
- verify-fix
- coverage
- evidence
- critique-revise
- prompt-tuning
- Composites
- Choosing between close candidates

---

## one-pass

**Fits:** small, closed tasks with an obvious check — translate a sentence, rename a variable, answer a factual question, reformat a table.

**Round:** do it → check against the prompt's requirements once → fix once if needed.

**Budget:** 1 round + 1 fix.

**Goes wrong when:** a loop is forced on it anyway. Saying "this doesn't need a loop; one pass and a check" is a correct outcome of this skill, not a failure.

---

## verify-fix

**Fits:** an executable check decides success — tests, build, type checker, linter, a validation script, a diff against expected output, a benchmark threshold. If the prompt implies a check that doesn't exist yet and writing it is cheap and allowed, write it first.

**Round:** run the full check → pick the first *real* failure (root cause, not the noisiest symptom) → fix it → rerun the full check (not just the one test, so regressions show up).

**Budget:** 5 rounds per distinct failure; 15 total. Stop sooner on no progress.

**Goes wrong when:**
- the fix weakens the check (deleting or loosening a test, broad try/except, skipping) — treat any edit to the check itself as needing a reason the prompt supports;
- flaky checks: rerun once before treating a failure as real;
- the check passes but the prompt's other requirements don't (API must not change, no new dependencies) — those are invariants, check them at the end.

---

## coverage

**Fits:** the work is a set of units — files, endpoints, drivers, chapters, rows, components — listed in the prompt or implied by it ("every", "all", "each").

**Round:** take the next unit → do the per-unit work (often an inner loop of another shape) → record its status in the ledger: `done`, `Unknown (reason)`, `blocked (reason)`, `out of scope (prompt line)`.

**Ordering:** start with the reference or calibration unit if the prompt names one (the known-good case others are compared against), then the units that share the most with it, then the odd ones. Doing the reference first makes the rest cheaper and consistent.

**Budget:** bounded by the number of units; per-unit budget comes from the inner shape. A unit that exhausts its budget gets a status and the loop moves on — one hard unit must not starve the rest, unless the prompt says that unit blocks everything.

**Goes wrong when:**
- the unit list is taken from memory instead of the prompt — copy it verbatim into the ledger at the start;
- units the prompt excluded get analyzed anyway;
- "done" is marked without the per-unit checks actually passing;
- the end summary loses units — the final audit counts statuses against the original list.

---

## evidence

**Fits:** facts must be established under uncertainty — reverse engineering, debugging an unknown cause, research, auditing, analysis where the prompt asks for confidence or evidence labels, or allows "Unknown".

**Round:** pick the most decision-relevant open question → form a hypothesis → gather the cheapest evidence that could refute it → calibrate (does the method reproduce something already known? if the prompt names calibration targets, use them) → record the claim with its evidence, location (file, offset, line, URL) and confidence label → update open questions.

**Labels:** use the prompt's labels if it defines them. Otherwise: `confirmed` (direct evidence, calibrated method), `likely` (indirect or uncalibrated evidence), `Unknown` (not settled, with what would settle it).

**Budget:** 3 attempts per open question with *different* methods; then `Unknown` with the reason and the next step that would settle it.

**Goes wrong when:**
- a claim is labeled from a name, comment or assumption instead of evidence (the prompt may forbid this explicitly);
- evidence contradicts earlier evidence and the loop papers over it — contradictions are usually a prompt stop rule, otherwise they must be surfaced, not averaged;
- work already settled elsewhere is redone — check the sources the prompt points to first;
- the loop keeps digging past the point the prompt needs; stop when the question is settled to the level the deliverable requires.

---

## critique-revise

**Fits:** quality is a judgment — writing, explanations, design, naming, plans, UX copy.

**Round:** critique the current draft against a rubric built from the prompt (audience, tone, length, required points, forbidden things) plus at most a few `(added)` quality criteria → list problems ranked by impact → revise only the top problems → check the revision didn't break anything that was fine.

**Critic:** a fresh-context subagent if available, given only the prompt, the rubric and the draft. Otherwise critique after re-reading the prompt cold. The author critiquing from memory finds what it meant, not what it wrote.

**Budget:** 3 rounds. Stop earlier when a round's critique finds only cosmetic issues.

**Goes wrong when:**
- each round rewrites everything and the draft oscillates;
- the rubric drifts toward the critic's taste rather than the prompt;
- length creeps past the prompt's limit.

---

## prompt-tuning

**Fits:** the user wants the prompt itself improved — "make this prompt more reliable", "why does this prompt keep failing".

**Round:** run the prompt (or simulate it faithfully) on 2–4 representative inputs → compare outputs to what the prompt intends → locate each failure in the prompt (missing instruction, ambiguity, conflicting rules, buried requirement) → make the smallest edit that fixes the cause → rerun the same inputs.

**Budget:** 4 rounds. Keep a diff of each edit and why.

**Goes wrong when:**
- edits overfit to the sample inputs;
- the prompt grows each round — prefer clarifying or removing over adding;
- the user's intent changes silently. Edits that change *what* the prompt asks for go to the user.

---

## Composites

Common combinations:

- **coverage × evidence → audit** — analyze every item of a list where facts must be established (driver analysis, codebase audit, literature review).
- **coverage × verify-fix → audit** — fix or migrate many files/modules, each with tests.
- **evidence → verify-fix** — find a bug's root cause, then fix it under tests.
- **verify-fix → critique-revise** — make it work, then make it readable (only if the prompt asks for quality beyond passing).
- **coverage × critique-revise** — a set of documents or sections each judged on quality.

Every composite ends with the final audit against the original prompt.

## Choosing between close candidates

- Evidence vs verify-fix: if a command can say yes/no, it's verify-fix; if you have to argue for an answer, it's evidence.
- Coverage vs one big unit: if the prompt says "each"/"every" or lists items, use coverage even when the items seem small — it's what guarantees none is skipped.
- Critique-revise vs one-pass: if the output is under a few paragraphs and the prompt's requirements are concrete, one-pass with a check is enough.
- When two shapes fit equally, take the one with the cheaper round.
