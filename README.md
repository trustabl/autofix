# autofix

AI-powered behavioral attestation and fix generation for agent tools. Runs your tool in an isolated OpenShell sandbox, collects runtime evidence, and generates fixes using an LLM. Your source code never leaves your environment.

> **Language support:** Static analysis is supported for Python, Go, JavaScript/TypeScript, Ruby, and Shell. Sandbox execution is available for all languages.

## How It Works

1. Static analysis scans your source for hardcoded credentials, unauthorized network calls, missing timeouts, and more
2. Your tool is uploaded to a sandboxed OpenShell environment and executed
3. Runtime evidence is collected — network denials, crashes, warnings, policy violations
4. Each issue is sent to an LLM with the full source context
5. You get back a fixed version of the code, a least-privilege policy YAML, and a signed behavioral attestation

The attestation is signed with [Sigstore](https://www.sigstore.dev/) keyless signing — no private key to manage. In CI (e.g. GitHub Actions with `permissions: id-token: write`), it signs with your workflow's identity automatically. Run locally without that, and it falls back to an interactive browser login bound to your personal identity, recorded in Sigstore's public transparency log — pass `--no-sign` to skip signing entirely instead.

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
# Replace v0.1.0 with the latest version from https://github.com/trustabl/autofix/releases
curl -L https://github.com/trustabl/autofix/releases/download/v0.1.0/autofix_0.1.0_linux_amd64.tar.gz | tar xz
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
| `--no-sign` | off | Skip Sigstore signing — no network call, no interactive browser flow, no public transparency log entry |
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
| `--no-sign` | off | Skip Sigstore signing — no network call, no interactive browser flow, no public transparency log entry |

---

### `verify` — Verify an attestation and gate CI

Checks an attestation's content digest and Sigstore signature, then optionally fails (non-zero exit code) on a low score or critical findings. See [docs/ci-gate.md](docs/ci-gate.md) for a full CI setup.

```bash
autofix verify ./autofix-attestation.json --min-score 70 --fail-on-critical
autofix verify ./autofix-attestation.json --require-signature --identity "https://github.com/acme-corp/.*"
```

**Flags:**

| Flag | Default | Description |
|---|---|---|
| `--min-score` | `0` (no gate) | Fail if `production_readiness_score` is below this value |
| `--fail-on-critical` | off | Fail if any critical findings are present |
| `--require-signature` | off | Fail if no valid Sigstore signature is present |
| `--identity` | — | Regex the signer's certificate SAN must match |
| `--issuer` | — | Regex the signer's OIDC issuer must match |
| `--output` | `text` | Output format: `text` or `json` |

---

### `version` — Print version

```bash
autofix version
```

### `license` — Print the full EULA

```bash
autofix license
```

---

## License

Autofix is free to use. It is a commercial product — all rights reserved.
Resale and redistribution are not permitted. Trustabl reserves the right to
change licensing terms in future versions.

See [legal/EULA.md](legal/EULA.md) for the full End-User License Agreement,
or run `autofix license` to read it in the terminal.
