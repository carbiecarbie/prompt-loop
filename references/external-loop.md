# External loop and gate

Read this only when Step 3b chose an external loop or a gate. The idea: the agent does **one unit per fresh session**, and a small script keeps calling it until a state file says it's finished. The state file is the loop's only memory.

## Contents
- Folder layout
- iteration.md (the prompt each session gets)
- state.md
- Driver script (PowerShell and bash)
- Gate: verify script
- Gate inside one Claude Code session (Stop hook)
- Rules that keep it safe

---

## Folder layout

Put the loop folder where the original prompt allows writes (its notes/work folder). Never inside a place the prompt forbids.

```
<allowed work dir>/loop/
  brief.md      the original prompt, copied unchanged
  iteration.md  the per-session prompt (below)
  state.md      units, statuses, log — the loop's memory
  run.ps1 / run.sh
  verify.*      only with a gate
```

## iteration.md

Keep it short: it is re-read every session.

```markdown
You are one iteration of a loop. Your memory is the files, not this conversation.

1. Read `loop/brief.md` (all its rules apply, including what not to touch) and `loop/state.md`.
2. If state says `status: DONE` or `status: HALT`, print it and stop.
3. Check the guards: <the brief's baseline / invariant checks, e.g. git clean, HEAD = X>. If one fails, set `status: HALT` with the reason and stop.
4. Take the first unit marked `partial` (resume from its note), else the first `todo`.
5. Do only that unit, using the checks listed for it in state.md. Verify by running/opening things, not from memory.
6. Write its results where the brief says deliverables go.
7. Update state.md: the unit's status (`done`, `partial: <where you stopped>`, `Unknown: <reason>`, `blocked: <reason>`, `stop item: <rule>`), one log line, any new open question.
   - A unit that comes back `partial` for the 3rd time becomes `blocked` — move on.
   - If the brief says a condition halts everything, set `status: HALT` with the reason.
8. When no `todo`/`partial` units remain, do the final-audit unit; after it, set `status: DONE`.
9. Print at most 10 lines: unit, status, next unit. Stop.
```

Anything the brief says to "ask before" can't be asked in an unattended run: record it as `blocked: needs maintainer OK for <X>` and continue with other units.

## state.md

```markdown
status: RUNNING        # RUNNING | DONE | HALT: <reason>
iteration: 0

## Units
- [todo] U0 setup — read required sources; record what's already settled
- [todo] U1 <reference/calibration unit, if the brief names one>
- [todo] U2 <unit> — checks: <pointer to the brief section, not a copy>
...
- [todo] U-AUDIT — re-read brief.md cold; check every requirement against the outputs; check invariants concretely
- [todo] U-REPORT — the brief's report format

## Open questions
## Stop items
## Log
```

Order units so the reference/calibration unit comes first: everything after depends on it. Recommend the user check results after it before letting the rest run.

## Driver script

The agent's CLI flags change between versions. Write the command once at the top, and tell the user to confirm flags with `codex exec --help` / `claude --help` (approval and sandbox settings matter: an unattended run must be able to write in the work folder and nowhere else the brief forbids).

PowerShell:

```powershell
$Loop = "<allowed work dir>/loop"
$Max  = 40                                   # hard budget of sessions
$Agent = { param($p) codex exec $p }         # or: { param($p) claude -p $p }
$prev = ""; $same = 0
for ($i = 1; $i -le $Max; $i++) {
  $s = Get-Content "$Loop/state.md" -Raw
  if ($s -match 'status: (DONE|HALT.*)') { "Loop ended: $($Matches[1])"; break }
  & $Agent (Get-Content "$Loop/iteration.md" -Raw)
  $now = Get-Content "$Loop/state.md" -Raw
  if ($now -eq $prev) { $same++ } else { $same = 0 }
  if ($same -ge 2) { "No progress in 2 sessions; stopping."; break }
  $prev = $now
}
```

bash:

```bash
LOOP="<allowed work dir>/loop"; MAX=40; prev=""; same=0
agent() { codex exec "$1"; }               # or: claude -p "$1"
for i in $(seq 1 "$MAX"); do
  grep -qE 'status: (DONE|HALT)' "$LOOP/state.md" && { grep -E 'status:' "$LOOP/state.md"; break; }
  agent "$(cat "$LOOP/iteration.md")"
  now="$(cat "$LOOP/state.md")"
  [ "$now" = "$prev" ] && same=$((same+1)) || same=0
  [ "$same" -ge 2 ] && { echo "No progress in 2 sessions; stopping."; break; }
  prev="$now"
done
```

## Gate: verify script

A gate is a script that decides "done" so the agent can't. It exits 0 when every mechanical requirement holds and prints each failure otherwise. Typical checks, taken from the prompt: the full test suite (run by the script, not trusted from the agent), files that must not change (hash or `git diff --quiet -- <paths>` against a baseline snapshot), public signatures unchanged, no new dependencies, required sections present in the output, line/word limits.

Take the baseline snapshot **before** the first session (a `baseline` step in the script), or the gate compares against already-modified files.

Wire it into the driver: after each session, run verify; if it fails, write its output into `state.md` under `## Gate failures` and run the next session (the iteration prompt tells the agent to fix those first). Finish only when verify passes, or the budget ends — then report the remaining failures.

The gate must not be editable by the agent: tell the iteration prompt that the loop folder's scripts are off-limits, and check their hashes in verify itself.

## Gate inside one Claude Code session (Stop hook)

When the work fits one session but early "done" is the problem, a Claude Code `Stop` hook can run verify each time the agent tries to finish: on failure it returns `{"decision": "block", "reason": "<verify output>"}` and the agent keeps working. Keep an attempt counter in a file and let the agent stop after the budget (or when its final message starts with `BLOCKED:` and evidence), otherwise a broken check traps it forever. Codex has no equivalent session hook here — use the driver-script gate there.

## Rules that keep it safe

- `brief.md` is the original prompt unchanged; the loop adds structure, never scope.
- One unit per session. A session that tries to do several loses the benefit.
- Don't widen permissions to make the loop convenient; if a step needs more than the brief allows, it's `blocked`, not worked around.
- Every external run is real, billed usage. In run mode, generate the files and ask before starting the driver; don't launch it on your own.
