# autofix

AI-powered behavioral attestation and fix generation for Python agent tools. Runs your tool in an isolated OpenShell sandbox, collects runtime evidence, and generates fixes using an LLM. Your source code never leaves your environment.

> **Language support:** Static analysis and sandbox execution are currently supported for Python only. Support for additional languages is coming in future releases.

## How It Works

1. Static analysis scans your Python source for hardcoded credentials, unauthorized network calls, missing timeouts, and more
2. Your tool is uploaded to a sandboxed OpenShell environment and executed
3. Runtime evidence is collected — network denials, crashes, warnings, policy violations
4. Each issue is sent to an LLM with the full source context
5. You get back a fixed version of the code, a least-privilege policy YAML, and a signed behavioral attestation

## Requirements

- [OpenShell](https://build.nvidia.com/openshell) — required for sandboxed execution (`--sandbox` flag)
- Access to an OpenAI-compatible LLM endpoint (OpenAI, Anthropic, Ollama, LiteLLM, LM Studio, etc.)
- Python 3.8+ on the host machine — required for dependency wheel downloads when using `--sandbox`

## Setup

### 1. Install OpenShell

Download from [build.nvidia.com/openshell](https://build.nvidia.com/openshell), then:

```bash
openshell gateway start   # keep running in a separate terminal
openshell sandbox list    # verify CLI is on PATH
```

### 2. Configure an LLM

**Local (Ollama):**
```bash
brew install ollama       # macOS
ollama serve
ollama pull qwen2.5-coder:32b
```

**Cloud:** set `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` — the endpoint is auto-detected from the model name.

### 3. Install autofix

**macOS (Homebrew):**
```bash
brew tap trustabl/autofix
brew install autofix
```

**Windows (Scoop):**
```powershell
scoop bucket add trustabl https://github.com/trustabl/scoop-autofix
scoop install autofix
```

**Linux / manual:**
```bash
curl -L https://github.com/trustabl/autofix/releases/latest/download/autofix_linux_amd64.tar.gz | tar xz
sudo mv autofix /usr/local/bin/
autofix version
```

---

## Commands

### `fix` — Scan and fix an agent tool

```bash
autofix fix --source ./my-tool
autofix fix --source ./my-tool --entrypoint tool.py --sandbox --model gpt-4o
autofix fix --source ./my-tool --existing-policy ./policy.yaml --model claude-3-5-sonnet-20241022
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--source` | required | Path to the tool directory |
| `--entrypoint` | auto-detected | Entry point script(s) — can repeat for multi-file scanning |
| `--model` | required | Model name. Cloud: `gpt-4o`, `claude-3-5-sonnet-20241022`. Local: `qwen2.5-coder:32b` (recommended) |
| `--llm-url` | auto-detected | OpenAI-compatible base URL. Auto-detected for `gpt-*` and `claude-*`. Required for all other models. |
| `--api-key` | env var | API key — defaults to `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `LLM_API_KEY` |
| `--llm-timeout` | `5m` | Per-request LLM timeout (increase for slow local models) |
| `--sandbox` | off | Run the tool in a sandbox for runtime findings |
| `--runner` | `openshell` | Sandbox runner: `openshell` or `docker` |
| `--base-image` | auto | Sandbox base image (default: auto-detected from entrypoint extension) |
| `--sandbox-args` | — | Arguments passed to the tool inside the sandbox. Value is a raw shell token — avoid shell metacharacters. |
| `--existing-policy` | — | Path to an existing OpenShell policy YAML to diff against the generated least-privilege policy |
| `--auto-accept` | off | Accept all fixes without interactive review (useful for CI) |
| `--output` | `text` | Output format: `text` or `json` |

**Output files** (written to `--source` directory):

| File | Description |
|---|---|
| `autofix-policy.yaml` | Least-privilege OpenShell network policy |
| `autofix-attestation.json` | Signed behavioral manifest with source hash |
| `autofix-session.json` | Full session log (findings, diffs, accept/reject decisions) |
| `autofix-session.patch` | Unified diff of all accepted fixes |

---

### `watch` — Continuous improvement loop

Polls source files for changes. On every save, re-runs static analysis, rewrites `autofix-policy.yaml` and `autofix-attestation.json`, and prints a delta (score change, resolved/new findings, new source hash).

```bash
autofix watch --source ./my-tool
autofix watch --source ./my-tool --entrypoint tool.py --interval 1s
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--source` | required | Path to the tool directory |
| `--entrypoint` | auto-detected | Entry point script(s) to watch |
| `--interval` | `2s` | Poll interval |

---

### `version` — Print version

```bash
autofix version
```

