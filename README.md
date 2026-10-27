# prompt-loop

An agent skill that takes a prompt you already wrote and runs it inside the **iterative loop that fits it** — with checks and stop conditions taken from the prompt itself.

You don't pick the loop. The skill reads the prompt and decides two things.

**The shape** — how to iterate:

| Shape | For prompts like |
|---|---|
| one-pass | small, closed tasks — no loop needed |
| verify-fix | "make the tests pass", "fix the build" |
| coverage | "analyze every X in this list" |
| evidence | reverse engineering, debugging, research with confidence labels |
| critique-revise | writing and design judged on quality |
| prompt-tuning | "make this prompt more reliable" |

…or a composite such as *coverage over 18 items × evidence per item → final audit*.

**Where the loop lives** — who keeps it going:

| Where | When |
|---|---|
| in-session (default) | the whole job fits comfortably in one session |
| external loop | the work won't fit one session (many heavy units, hours of work, long rules that compaction could drop). A driver script calls `codex exec` / `claude -p` once per unit; progress lives in a `state.md` file |
| + gate | the agent tends to declare "done" too early and an executable check exists. A verify script the agent can't skip decides when it's done |

The prompt's own rules always win: if it says "do this stage and stop" or "never write to the repo", the loop respects that. Your prompt text is never rewritten — the loop is added after it.

Works in **Claude Code** and **Codex** (and any harness that supports the open Agent Skills format).

## Install

```bash
npx skills add carbiecarbie/prompt-loop
```

Or copy this folder into:

- Claude Code: `~/.claude/skills/prompt-loop/`
- Codex: `~/.codex/skills/prompt-loop/`

## Usage

### 1. Give it your prompt

Paste the prompt, attach it, or point to a file, and ask for a loop. Any of these work:

```
/prompt-loop  (then paste the prompt)
loop this prompt: <prompt>
run the prompt in prompts/stage-0b.md until it's right
faz um loop iterativo pra esse prompt: <prompt>
```

You don't have to name the skill — "loop", "iterate", "run until done", "keep going until it's right" trigger it.

### 2. Pick a mode (or let it pick)

- **Run (default).** It shows a short loop card and starts working in the current session.
- **Plan only.** Say "plan only", "don't run it", or that you'll run it somewhere else ("I'll run it in Codex"). You get the loop card plus everything needed to run it elsewhere. It also switches to plan only by itself when the files or tools your prompt needs aren't reachable from the current session.

### 3. Read the loop card

Before the first round it shows something like:

```
Loop: coverage over 18 drivers × evidence per driver → final audit
Runs: external (18 heavy units would be compacted in one session)
Why: the prompt lists every driver and needs a calibrated label per claim
Unit: one driver per session
Checks: 9 done · 10 invariant · 3 stop · 3 format — top: ...
Stops when: done · baseline differs · contradictory evidence
Ledger: m94/loop/state.md
Reading taken: a single uncalibratable claim becomes Unknown, not a halt
```

Checks marked `(added)` are ones the skill added that your prompt didn't state — veto any of them. "Reading taken" shows how it resolved an ambiguity in your prompt.

### 4. What you get back

**In-session loop, run mode** — the work itself, plus a report that says which stop condition ended the loop. If your prompt defines a report format, that format is used.

**In-session loop, plan only** — a **Loop protocol** block to paste *after* your original prompt. Your prompt stays unchanged.

**External loop** — a `loop/` folder, created where your prompt allows writes:

| File | What it is |
|---|---|
| `brief.md` | your original prompt, unchanged |
| `iteration.md` | the short prompt each fresh session gets: read the brief and state, do one unit, update state, stop |
| `state.md` | the unit list with statuses, open questions, stop items and a log — the loop's only memory |
| `run.ps1` / `run.sh` | the driver script |
| `verify.*` | only with a gate |

**Gate in Claude Code** — a verify script plus a Stop hook, so the agent can't finish until the checks pass.

### 5. Running an external loop

1. Confirm the CLI flags for your version: `codex exec --help` or `claude --help`. An unattended run must be able to write in the work folder, and nowhere your prompt forbids.
2. Run the driver script (`./run.ps1` or `./run.sh`). Each iteration is a fresh session doing one unit.
3. Check the first units before letting it run. If your prompt has reference or calibration units, the loop puts them first. If the script has a `$StopAfter` setting, use it to pause after them.
4. The script stops when `state.md` says `DONE` or `HALT`, after two sessions with no progress, or at the session budget.
5. If it stops midway, run it again. It resumes from `state.md` without redoing finished units.

Anything your prompt says to "ask before" can't be asked in an unattended run. Those items are marked `blocked: needs OK for …` in `state.md` so you can decide afterwards.

Every external session is real, billed usage. In run mode the skill creates the files and asks before launching the driver.

### 6. Using a gate

1. Commit or stash your changes, then take the baseline: run the verify script's `baseline` step **before** the agent starts. It snapshots the files and signatures that must not change.
2. In Claude Code, add the generated hook settings to the project's `.claude/settings.local.json`. In Codex, use the driver script, which reruns verify after each session.
3. Start the agent with your prompt plus the protocol. It can only finish when verify passes, when it hits the attempt budget, or when it replies `BLOCKED:` with evidence.
4. Still skim the diff. A gate checks the mechanical rules, not whether the fix is good.

## Tips

- **Long prompt → plan in one tool, run in the other.** Ask Claude for "plan only, I'll run it in Codex" (or the reverse) and paste the result.
- **Small task?** It will say no loop is needed and just do it. That's intended.
- **Budgets.** Defaults are about 3 rounds for writing, 5 per failing check for code, and 3 attempts per open question before marking it Unknown. Ask for tighter or looser budgets if you need them.
