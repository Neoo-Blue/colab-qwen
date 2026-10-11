# colab-qwen

A [Claude Code](https://claude.com/claude-code) skill that turns a Qwen model on a Google Colab
A100 into an on-demand worker. Claude decides when a task is worth offloading, `qwenx` starts the
GPU, serves the model with Ollama, gets the answer, and stops the GPU in the same command. Nothing
is left running between tasks.

- **`SKILL.md`**: when to use it (and when not), the workflow, hard rules, models, and cost
- **`qwenx`**: the CLI (stdlib-only Python 3.9+; macOS, Linux, Windows via WSL2). `run`, `batch`, `stop`, `status`, `selftest`
- **`agents/colab-qwen-worker.md`**: a sub-agent relay so Claude can dispatch a task in the
  background via the Agent tool

## Install

The Colab CLI runs on macOS and Linux only, so on Windows everything below runs inside WSL2.
The steps after the first one are identical on all three.

### 1. Prerequisite per OS

**macOS**
```bash
brew install uv            # or: curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Linux** (any distro with Python 3 and git)
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"   # add to ~/.bashrc or ~/.zshrc if not already there
```

**Windows**: open PowerShell as administrator:
```powershell
wsl --install -d Ubuntu
```
Reboot, open the **Ubuntu** app, and run the Linux step above inside it. Run Claude Code
(or Codex, OpenCode, etc.) inside WSL too, so the agent and `qwenx` share one environment.
Native Windows `qwenx` exits with a message pointing here.

**macOS certificate note.** The Colab CLI's live connection to the VM uses Python's default
certificate store, which is empty on python.org and uv-managed Pythons
(`CERTIFICATE_VERIFY_FAILED`). `qwenx` handles this by pointing `SSL_CERT_FILE` at the
certifi bundle inside the CLI's own environment; set `SSL_CERT_FILE` yourself to override.

### 2. Colab CLI and sign-in

```bash
uv tool install google-colab-cli
colab usage      # first call signs in. With a browser it opens one; on a headless server
                 # or WSL it prints a URL, you sign in anywhere and paste the code back
```

Needs Colab Pro or Pro+ (A100 access and compute units).

### 3. The skill

```bash
mkdir -p ~/.claude/skills ~/.claude/agents ~/.local/bin
git clone https://github.com/Neoo-Blue/colab-qwen ~/.claude/skills/colab-qwen
ln -sf ~/.claude/skills/colab-qwen/qwenx ~/.local/bin/qwenx
cp ~/.claude/skills/colab-qwen/agents/colab-qwen-worker.md ~/.claude/agents/
qwenx selftest
```

### 4. Other agent tools (optional)

Every tool runs the same `qwenx` CLI; these links just make each one load the skill.

| Tool | Needs |
|---|---|
| Claude Code | step 3 (skill + `colab-qwen-worker` sub-agent) |
| OpenCode, Muse, Grok Build | nothing more: they read `~/.claude/skills` |
| Codex (and others using the shared folder) | `mkdir -p ~/.agents/skills && ln -sfn ~/.claude/skills/colab-qwen ~/.agents/skills/colab-qwen` |
| Antigravity (`agy`) | `mkdir -p ~/.gemini/config/skills && ln -sfn ~/.claude/skills/colab-qwen ~/.gemini/config/skills/colab-qwen` |

Update everything later with `git -C ~/.claude/skills/colab-qwen pull`.

## Use

In Claude Code, `/colab-qwen <task>`, or just describe heavy batch or long-context work and let
Claude pick it up from the skill description. From a shell:

```bash
qwenx run "Summarize the failure modes in this log" -f app.log
qwenx batch -o out/ tasks/*.md        # many prompts, one GPU session
qwenx batch -t max -o out/ tasks/*.md # max tier: Qwen3.8-Flash-Next 125B, ~18 min start
qwenx status                          # active sessions + compute-unit balance
qwenx stop                            # cleanup, always safe
```

## Tiers

| Tier | Model | Ready in | Warm speed | Use |
|---|---|---|---|---|
| `standard` (default) | `qwen3.8:27b` | ~3 min | ~78 tok/s | almost everything |
| `max` | `qwen3.8-flash-next:125b-a6b-q4_K_M` | ~18 min | ~47 tok/s | big batches where quality matters |

Measured on Colab's 80 GB A100 (the default `--high-mem` shape). Details in `SKILL.md`.

## Lifetime guarantees

Every run stops its session in a `finally` block, on Ctrl-C / SIGTERM / SIGHUP, and through a
detached watchdog that runs `colab stop` after `--max-minutes` (default 60) even if `qwenx` is
killed outright. The default A100 (80 GB, high-RAM shape) burns about 6.8 compute units per hour (40 GB: ~5.3); `qwenx` prints the live rate and
the estimated spend at the end of each run.

## License

MIT
