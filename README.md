# colab-qwen

A [Claude Code](https://claude.com/claude-code) skill that turns a Qwen model on a Google Colab
A100 into an on-demand worker. Claude decides when a task is worth offloading, `qwenx` starts the
GPU, serves the model with Ollama, gets the answer, and stops the GPU in the same command. Nothing
is left running between tasks.

- **`SKILL.md`**: when to use it (and when not), the workflow, hard rules, models, and cost
- **`qwenx`**: the CLI (stdlib-only Python 3.10+). `run`, `batch`, `stop`, `status`, `selftest`
- **`agents/colab-qwen-worker.md`**: a sub-agent relay so Claude can dispatch a task in the
  background via the Agent tool

## Install

```bash
uv tool install google-colab-cli                     # official Colab CLI (`colab`)
colab usage                                          # one-time Google sign-in
git clone https://github.com/Neoo-Blue/colab-qwen ~/.claude/skills/colab-qwen
ln -sf ~/.claude/skills/colab-qwen/qwenx ~/.local/bin/qwenx
cp ~/.claude/skills/colab-qwen/agents/colab-qwen-worker.md ~/.claude/agents/
qwenx selftest
```

Needs Colab Pro or Pro+ (A100 access and compute units). macOS or Linux.

## Use

In Claude Code, `/colab-qwen <task>`, or just describe heavy batch or long-context work and let
Claude pick it up from the skill description. From a shell:

```bash
qwenx run "Summarize the failure modes in this log" -f app.log
qwenx batch -o out/ tasks/*.md        # many prompts, one GPU session
qwenx status                          # active sessions + compute-unit balance
qwenx stop                            # cleanup, always safe
```

## Lifetime guarantees

Every run stops its session in a `finally` block, on Ctrl-C / SIGTERM / SIGHUP, and through a
detached watchdog that runs `colab stop` after `--max-minutes` (default 60) even if `qwenx` is
killed outright. A100 burns roughly 7-8 compute units per hour; `qwenx` prints the live rate and
the estimated spend at the end of each run.

## License

MIT
