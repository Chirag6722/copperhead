---
title: Your first run
description: A guided first hour with copperhead on a board you already have — including what it looks like when something goes wrong.
sidebar:
  order: 3
---

The [Quickstart](/getting-started/quickstart/) lists the commands. This page walks the path, in order, on a board you already own — and spends most of its length on the parts where people get stuck.

Budget about twenty minutes. At the end you will have one small, ERC-verified, committed change, and enough of a feel for the loop to decide whether you trust it with a bigger one.

## Before you start

You need a KiCad project in a git repository. Not a copy on the desktop — an actual repo, because copperhead snapshots with git before it edits and rolls back to that snapshot when verification fails. No repo, no safety net, and the preflight will refuse to run.

If your board is not in git yet:

```bash
cd my-board
git init && git add -A && git commit -m "baseline before copperhead"
```

Work on a branch for the first run. Nothing here is destructive, but a branch makes "throw it all away" a one-liner:

```bash
git switch -c copperhead-trial
```

## Step 1: Ask what is missing

Run this before anything else. It calls no model and touches no network — it just tells you whether the machine is ready.

```bash
copperhead doctor
```

On a machine that is not ready yet:

```text
  [ok]   node      v24.16.0 (>= 20)
  [ok]   kicad-cli 8.0.9
  [ok]   git       2.54.0.windows.1
  [FAIL] provider  no model configured
         hint: pass --model, set COPPERHEAD_MODEL, or export an API key; see
               https://docs.copperhead.sh/reference/configuration/
  [info] project   no .copperhead/config.json (run `copperhead init` to
                   scaffold)
not ready: fix the [FAIL] items above
```

`[info]` lines are notes, not problems — the missing `config.json` on that last line is exactly what Step 3 creates. Only `[FAIL]` blocks you. Exit code is 0 when ready, 1 when not, so this is safe to put in a setup script.

:::caution[kicad-cli must be on PATH]
Every copperhead command probes `kicad-cli` before doing anything, so if this check fails, nothing else will run. On macOS the binary ships inside the app bundle and is often not linked:

```bash
export PATH="/Applications/KiCad/KiCad.app/Contents/MacOS:$PATH"
```
:::

## Step 2: Choose one model backend

You need exactly one. Not zero, and — this trips people up — not two.

If you already use Codex CLI or Claude Code, reuse that login and skip API keys entirely:

```bash
export COPPERHEAD_MODEL=codex          # or: claude-code, cursor
```

Otherwise export a single key:

```bash
export ANTHROPIC_API_KEY=...           # or OPENAI_API_KEY, not both
```

:::danger[Two keys in your environment is a hard stop]
If you have both `OPENAI_API_KEY` and `ANTHROPIC_API_KEY` exported — common on a developer machine — copperhead refuses to guess:

```text
  [FAIL] provider  ambiguous: multiple credentials, no model selected
         hint: 2 credentials found (OPENAI_API_KEY, ANTHROPIC_API_KEY) and no
               model was selected; pass --model, set COPPERHEAD_MODEL, or set
               "model" in .copperhead/config.json.
```

This is deliberate. Silently picking one could send your design to a provider you did not intend, and bill you for it. Name the model and the error goes away:

```bash
export COPPERHEAD_MODEL=claude
```
:::

Re-run `copperhead doctor` until it says `ready`:

```text
  [ok]   node      v24.16.0 (>= 20)
  [ok]   kicad-cli 8.0.9
  [ok]   git       2.54.0.windows.1
  [ok]   provider  gpt-5 -> openai: OPENAI_API_KEY set
  [info] project   no .copperhead/config.json (run `copperhead init` to
                   scaffold)
ready
```

Your key is read from the environment only. It is never written into `.copperhead/config.json`, and it is redacted from run transcripts as they are written.

## Step 3: Adopt the board

```bash
copperhead init
```

This reads your schematic and scaffolds `docs/` — `SPEC.md`, `SUBSYSTEMS.md`, `BOM.md`, `PINOUT.md`, `LAYOUT.md` — plus `.copperhead/config.json` pointing at your schematic and board. It also installs a `pre-commit` hook that runs `copperhead check`, and it will not clobber a hook you already have.

`init` is idempotent. Re-run it freely: it reports `unchanged` for files it already wrote, and it **refuses** to overwrite a generated doc you have since hand-edited, exiting non-zero and naming the files it skipped. Pass `--force` only when you genuinely want them regenerated.

Read what it produced before continuing. The scaffolded docs are the agent's memory — every later run reads them first, so a wrong value here becomes a wrong assumption in every change after it. This is the highest-leverage ten minutes in the whole process.

## Step 4: Establish a baseline

```bash
copperhead check          # alias: copperhead verify
```

This runs ERC, DRC, doc-drift detection, constraint checks, and spec validation. It makes no LLM calls and opens no network connections, by contract — which is why it is safe in CI and in that pre-commit hook.

:::caution[A green check on an un-adopted repo means nothing]
If `init` has not run, or `config.json` does not point at your files, `check` skips everything and still exits 0:

```text
ERC skipped (no schematic configured; run copperhead init)
DRC skipped (no board configured)
```

That is a pass over an empty set, not a clean board. Read the lines, not the exit code — if you see `skipped`, fix the config before you trust anything downstream.
:::

Expect real findings on a real board. Fix them, or note them, before the agent starts making changes — otherwise you cannot tell its mistakes from your pre-existing ones.

## Step 5: One small change

Pick something narrow and easy to eyeball. A rename is ideal for a first run: unambiguous, and wrong is obvious.

See the proposal without writing anything:

```bash
copperhead do "rename net KEY_DAH to KEY_DASH" --dry-run
```

Then run it for real, pausing for your approval once the plan validates:

```bash
copperhead do "rename net KEY_DAH to KEY_DASH" --interactive
```

What happens, in order:

1. **Propose.** The agent writes an OpenSpec change proposal — why, what changes, the task list. Until this validates, its edit tools do not exist. Not discouraged: absent from the tool list it is given.
2. **Edit.** Anchored exact-match replacements in the `.kicad_sch` and in every doc that mentions the net.
3. **Verify.** ERC runs, and DRC too if the board changed. Failures come back to the agent as its own error report to repair.
4. **Commit.** One commit, with the verification result in the message, plus a line in `docs/CHANGELOG.md` and any real decision appended to `docs/DECISIONS.md`.

If verification cannot be made to pass, the run rolls back to the pre-run snapshot and your tree is left exactly as it was.

## What a run costs

Worth calibrating before you reach for the big commands:

| Command | Typical shape |
|---|---|
| `check`, `doctor`, `draft`, `score`, `export bom` | Seconds. No model, no network, no cost. |
| `do "<small change>"` | A handful of turns, a few minutes. |
| `create --brief brief.md` | Eight stages. Hours, not minutes. |

`create` is a full pipeline — spec, architecture, part selection, schematic, layout, outputs, firmware, dev plan. Individual stages can legitimately run over an hour. It is not hung; it prints a heartbeat every 30 seconds while a turn is in flight. Try it first on a small brief from [examples/simple](https://github.com/chouhanindustries/copperhead/tree/main/examples/simple).

## When it goes wrong

| What you see | What it means | Fix |
|---|---|---|
| `kicad-cli` check fails in `doctor` | KiCad's CLI is not on `PATH`; nothing will run | Add KiCad's binary directory to `PATH` |
| `ambiguous: 2 credentials found` | Two API keys exported, no model named | `export COPPERHEAD_MODEL=claude` |
| `no model configured` | No key and no saved login found | Export one key, or set `COPPERHEAD_MODEL` to a saved-login backend |
| Preflight refuses on a dirty tree | Uncommitted changes would be caught in the snapshot | Commit or stash first, or pass `--allow-dirty` |
| `ERC skipped (no schematic configured)` | `init` has not run, or config points nowhere | `copperhead init` |
| `run failed: ... working tree restored` | Verification never passed; changes were rolled back | See the preserved-work note below |
| `session/usage limit reached` | Saved-login quota, not a bug | Wait for the stated reset, re-run the same command |

Two things worth knowing about that last column.

**Failed work is not destroyed.** A failed run preserves everything it touched as a named git stash entry before rolling back, and prints the recovery command:

```bash
git stash apply     # get the failed run's work back
git stash drop      # or discard it
```

**A rate-limited saved-login run resumes cheaply.** Every completed turn is cached on disk, so re-running the same command after the reset replays that work at roughly zero tokens and picks up where it stopped.

## What it will refuse to do

For a first run, the refusals matter more than the features. copperhead is built to fail closed:

- **No edit without a validated proposal.** The edit tools are structurally withheld, not merely discouraged.
- **No "done" without ERC.** And DRC too, whenever the board changed.
- **No edit that breaks a KiCad file.** Every text edit to a schematic or board is probed for loadability, and reverted if it would make the file unopenable.
- **No hand-edits to an engine-drafted sheet.** Sheets drafted from an intent file are regenerated wholesale; direct geometry edits are refused rather than silently lost on the next re-draft.
- **No silent resolution of a requirement violation.** `sync` will fix docs that drifted from the as-built schematic, but a violated budget or constraint is reported for you to decide, never auto-resolved.
- **No LLM in `check`.** That command imports no provider at all, so it cannot make a network call.

## Next

- [Guardrails](/concepts/guardrails/): the two invariants, in depth
- [The agent loop](/concepts/agent-loop/): what one run actually does, turn by turn
- [Docs as memory](/concepts/docs-as-memory/): what lives in `docs/` and why it matters
- [Design from a brief](/workflows/create-from-brief/): when you are starting from nothing
- [CLI reference](/reference/cli/): every command and flag
