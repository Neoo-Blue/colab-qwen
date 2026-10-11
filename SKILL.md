---
name: colab-qwen
description: Offload a self-contained task to a Qwen model (standard tier qwen3.8:27b, max tier Qwen3.8-Flash-Next 125B on the Strata engine) running on a Google Colab A100 that is started for the task and stopped right after. Use for heavy, batchable, or long-context work where a strong open model is good enough and Claude's own budget or context is the constraint - bulk summarization, classification or extraction over many files, a long-log or long-document digest, drafting many independent variants, an alternate solution to compare. Spends Colab compute units (A100 80 GB about 7 CU/hour) and has a ~3 minute cold start (~21 on the max tier), so never use it for quick questions, live facts, anything secret, or work a free peer (Grok, Antigravity) handles. Also the reference for the qwenx CLI and the colab-qwen-worker sub-agent, and invocable as /colab-qwen.
---

# Colab Qwen worker

An open model on a rented GPU, as a one-shot worker. Every call is a full GPU lifetime:
`colab new` (A100) -> install and serve Ollama -> pull the model -> answer -> `colab stop`.
Nothing stays up between calls. The bridge is the `qwenx` CLI in this folder
(`~/.claude/skills/colab-qwen/qwenx`, linked to `~/.local/bin/qwenx`), exposed two ways:

| Surface | What | When |
|---|---|---|
| Sub-agent `colab-qwen-worker` (`~/.claude/agents/colab-qwen-worker.md`) | thin relay: one `qwenx` call, output verbatim | default: dispatch via the Agent tool in the background, keep working, reconcile |
| CLI `qwenx` | `run`, `batch`, `stop`, `status`, `selftest` | inline, scripts, humans, batching many prompts into one GPU session |

## Hard rules

1. **Never idle.** The GPU exists only inside one `qwenx` command. Never run `colab new` by
   hand, never pass `--keep` to `colab run`, never leave a session up "for the next task".
   If more work is coming, collect it first and send it as one `qwenx batch`.
2. **Always stop.** `qwenx` stops the session in a `finally` block, on Ctrl-C, SIGTERM and
   SIGHUP, and arms a detached watchdog that runs `colab stop` after `--max-minutes`
   (default 60) even if `qwenx` itself is killed. If a run ends with
   `NOT CONFIRMED STOPPED`, or you are unsure, run `qwenx stop` then `qwenx status`
   immediately. Stopping a stopped session is harmless.
3. **Batch, don't loop.** Cold start is ~3 minutes on the standard tier (~21 on max) and
   costs more than most single answers. Ten prompts = one `qwenx batch`, never ten `qwenx run`.
4. **No secrets.** The prompt goes to a Google VM. Never send credentials, `.env` contents,
   customer data, or anything from `~/.claude/sesame`.
5. **Verify the output.** It is a 27B open model, below Claude and the peer bench on hard
   reasoning. Treat it as a fast junior: check facts, run its code, read its diffs.

## When to use it, and when not

Use it when **all** of these hold: the task is self-contained (everything it needs fits in
the prompt and attached files), an open 27B-class model is good enough, and it is heavy
enough to be worth ~0.5 CU of startup - usually because it is **many prompts** or **a lot
of text**.

| Good fit | Why |
|---|---|
| Summarize, classify, tag, or extract over dozens of files or records | one batch, parallel to your own work, saves Claude context |
| Digest a long log, transcript, or document (up to ~32k tokens by default, `--ctx` raises it) | keeps the bulk out of Claude's context |
| Draft many independent variants (copy, test cases, fixtures, translations) | cheap volume |
| An alternate implementation or answer to compare against yours | a different model family; disagreement is signal |
| Work the user explicitly sends to Qwen / Colab | their call |

Do **not** use it for:

- quick questions, single short answers, anything under a minute of your own work
- live facts, docs, versions, news (no web; use `grok-researcher` / `antigravity-researcher`)
- a second opinion on a hard call (use `grok-analyst` / `antigravity-analyst`: free, stronger)
- diff review (use `grok-reviewer`), or edits in the real working tree (it has no tools; it
  only returns text)
- anything secret, or the critical path you are blocked on (startup latency is minutes)

Before a big batch, check `qwenx status` (balance and active sessions).

## Workflow

1. **Decide and collect.** Confirm the task fits the table above. Gather every prompt that
   could go in the same GPU session.
2. **Write self-contained prompts.** Each prompt carries its own instructions, inputs, and
   output format. The model sees nothing else: no repo, no files unless attached, no web.
3. **Dispatch.**
   - One task, in the background: Agent tool, `subagent_type: colab-qwen-worker`, prompt =
     the brief (header lines below). Keep working while it runs.
   - One task, inline: `qwenx run "prompt" -f input.txt` (`-f -` attaches stdin).
   - Many tasks: write one file per prompt, then `qwenx batch -o outdir/ tasks/*.md`.
4. **Read the result.** The answer is on stdout (or `outdir/<task>.md`); the meta line on
   stderr shows tokens, tok/s, minutes, and the estimated CU spent.
5. **Confirm teardown.** The last stderr line must read `session stopped`. If it does not:
   `qwenx stop && qwenx status`.
6. **Verify and integrate.** You own the result; check it before using it.

## Sub-agent brief header lines

```
tier: max                 # optional: Qwen3.8-Flash-Next 125B on Strata (~21 min start, stronger, faster)
model: gemma4:31b         # optional Ollama tag, overrides the tier's model
think: yes                # optional reasoning mode (slower, better on hard problems)
ctx: 65536                # optional context window, default 32768
files: /abs/a.log,/abs/b.md   # optional files attached as context
<the task, self-contained>
```

## Tiers and models

Two tiers, benchmarked 2026-10-10 on the 80 GB A100 with the same coding and reasoning prompts:

| Tier | Model / engine | Ready in | Speed | Short batch | Pick when |
|---|---|---|---|---|---|
| `standard` (default) | `qwen3.8:27b` on Ollama | ~3 min | ~78 tok/s | ~1 CU | almost always: digests, extraction, drafts, most code |
| `max` (`-t max`) | Qwen3.8-Flash-Next 125B-A6B, IQ3_S, on [Strata](https://github.com/Niko1221/Strata) | ~21 min | 75-105 tok/s | ~2.5 CU | a big batch where answer quality matters: the startup only pays off over many prompts |

`max` is the stronger model and, once up, the faster one. Its cost is startup: Strata compiles
its engine for the A100 (~3 min), downloads ~85 GB (~9 min), prepares the draft layer that
gives its 1.6-1.8x guess-ahead speedup (~6 min), and loads (~3 min). In the benchmark both
tiers got the reasoning task right; on the coding task only `max` also wrote correct tests.
Running the same model on Ollama (`-t max --engine ollama -m qwen3.8-flash-next:125b-a6b-q4_K_M`)
took as long to start and ran at only 47 tok/s, so Strata is the default engine for `max`.

Other choices: `-m` with any Ollama tag (e.g. `gemma4:31b`) uses Ollama whatever the tier;
`-t max -m IQ2_XS` picks a smaller Strata size (sizes: IQ3_S, IQ3_XXS, IQ2_XS, Q2_0).
`QWENX_MODEL` changes the standard tier's model. Not usable: Ollama's `nvfp4`/`mlx`
Flash-Next builds (Apple MLX only) and the 40 GB A100 for `max`.

Every run downloads the model again, because Colab wipes the VM on stop. A cache does not
help: Ollama's registry measured ~1.3 GB/s and Strata's download ~100-380 MB/s, while Google
Drive needs a manual approval on every new VM.

## Cost

Measured 2026-10-10 (rates come from `colab usage` and move with demand):

| Shape | GPU | Rate |
|---|---|---|
| A100 `--high-mem` (default) | A100-SXM4 **80 GB**, 167 GB RAM | ~6.8 CU/h |
| A100 `--no-high-mem` | A100-SXM4 40 GB | ~5.3 CU/h |

- Standard tier: a two-prompt batch took 8.5 min and ~1 CU (ready in ~3 min; the first
  answer also pays ~70 s to load the model into VRAM).
- Max tier (Strata): the same batch took 22 min and ~2.5 CU (ready in ~21 min, then both
  answers in 17 s).
- `qwenx` reads the live rate on every run and prints the estimated CU spent; trust that line.
- `--gpu L4` costs less and is fine for small batches with a 27B model.
- `qwenx status` shows the remaining balance. Tell the user what a large batch cost.

## CLI recipes

```bash
qwenx run "Summarize the failure modes in this log as a table" -f /var/log/app.log
git diff main | qwenx run "Write a changelog entry for this diff" -f -
qwenx batch -t max -o /tmp/out tasks/*.md  # Flash-Next on Strata, ~21 min start: batches only
qwenx batch -o /tmp/out tasks/*.md          # one GPU session for all of them
qwenx batch --gpu L4 --max-minutes 30 -o out tasks/*.md
qwenx run --json "..." | jq -r .content     # machine-readable result
qwenx status                                # sessions + CU balance and rate
qwenx stop                                  # cleanup, always safe
qwenx selftest                              # offline wrapper check, no Colab call
```

Flags shared by `run` and `batch`: `-t {standard,max}`, `-m MODEL`, `--engine {ollama,strata}`, `-s SESSION` (default `qwen-a100`),
`--gpu {T4,L4,G4,A100,H100}`, `--high-mem/--no-high-mem` (default on: 80 GB A100), `--ctx N`, `--think/--no-think`,
`--system TEXT`, `--max-minutes N`, `--boot-timeout S`, `--gen-timeout S`, `--json`.

## Setup (once)

Same steps on every OS; Windows runs them inside WSL2 (the Colab CLI is macOS/Linux only).

| OS | Before the common steps |
|---|---|
| macOS | `brew install uv` (or `curl -LsSf https://astral.sh/uv/install.sh \| sh`) |
| Linux | `curl -LsSf https://astral.sh/uv/install.sh \| sh`; make sure `~/.local/bin` is on `PATH` |
| Windows | PowerShell as admin: `wsl --install -d Ubuntu`, reboot, open Ubuntu, then follow the Linux row. Use the skill from tools running inside WSL |

Common steps:

```bash
uv tool install google-colab-cli      # the official Colab CLI, provides `colab`
colab usage                           # first call signs in; on a headless box or WSL it
                                      # prints a URL and asks you to paste the code back
qwenx selftest
```

Requires Colab Pro or Pro+ for A100 access and compute units. Full install, including the
links for other agent tools, is in `README.md`.

## How it works, and the gotchas it handles

- `colab exec` runs **Python** in the session's Jupyter kernel, not shell. `qwenx` sends
  Python that installs Ollama with `subprocess`, starts `ollama serve` detached (it outlives
  each exec because the kernel does), and calls `http://127.0.0.1:11434/api/chat` from
  inside the VM. No tunnel, no public port.
- `colab exec` defaults to a **30 s timeout**; `qwenx` passes `--boot-timeout` /
  `--gen-timeout` explicitly.
- `colab exec` **exits 0 even when the remote code raised**. `qwenx` wraps every result in
  `<<<QWENX_BEGIN>>>`/`<<<QWENX_END>>>` markers and treats a missing marker as failure.
- Prompts travel base64-encoded inside the exec code, so quoting never breaks.
- A lock file per session name stops two runs from fighting over one VM; use `-s NAME` to
  run two in parallel (each one costs its own GPU).
- `<think>` blocks are stripped from answers; `--json` keeps the separate `thinking` field.
- State lives in `~/.cache/colab-qwen/` (per-session lock and watchdog pid).
