# Gating CI on autofix attestations

`autofix verify` checks a behavioral attestation's integrity and signature,
then optionally fails the build on a low production-readiness score or
critical findings. Run it in your own CI, right after `autofix fix`.

## How signing works

When `autofix fix` writes `autofix-attestation.json`, it attempts to sign it
with [Sigstore](https://www.sigstore.dev/) keyless signing: an ephemeral
keypair is generated, exchanged for a short-lived certificate from the
public-good Fulcio CA bound to your CI's OIDC identity, and a Rekor
transparency log entry is recorded. No private key is ever stored or managed.

This is what makes the gate meaningful — the signature asserts "this
attestation was produced by a run of *this* workflow, in *this* repo." An
attacker on a malicious PR can't forge a passing attestation without
controlling that CI identity.

Signing is best-effort: if no OIDC credential is available (no CI, or no
`id-token: write` permission), `autofix fix` still completes and writes an
unsigned attestation with a warning on stderr. Use `--require-signature` on
`verify` to make an unsigned attestation fail the gate once you're ready to
enforce it. Pass `--no-sign` to `fix`/`watch` to skip signing entirely —
useful for local runs where you don't want a browser popping up or your
identity recorded in Sigstore's public transparency log.

## GitHub Actions example

```yaml
name: autofix-gate
on: [pull_request]

permissions:
  id-token: write   # required — Fulcio needs this OIDC token. Without it,
                     # signing silently falls back to an interactive flow
                     # that fails non-interactively in CI.
  contents: read

jobs:
  attest:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download autofix
        run: |
          curl -L https://github.com/trustabl/autofix/releases/latest/download/autofix_linux_amd64.tar.gz | tar xz
          chmod +x autofix

      - name: Scan and attest
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: ./autofix fix --source . --entrypoint tool.py --model gpt-4o --output json --auto-accept

      - name: Verify attestation and gate
        run: |
          ./autofix verify ./autofix-attestation.json \
            --require-signature \
            --identity "https://github.com/${{ github.repository }}/.*" \
            --issuer "https://token.actions.githubusercontent.com" \
            --min-score 70 \
            --fail-on-critical
```

## `verify` flags

| Flag | Default | Description |
|---|---|---|
| `--min-score` | `0` (no gate) | Fail if `production_readiness_score` is below this value |
| `--fail-on-critical` | `false` | Fail if any critical findings are present |
| `--require-signature` | `false` | Fail if no valid Sigstore signature is present |
| `--identity` | — | Regex the signer's certificate SAN must match |
| `--issuer` | — | Regex the signer's OIDC issuer must match |
| `--output` | `text` | Output format: `text` or `json` |

Start with `--require-signature` unset while rolling this out across teams —
plain score/critical gating still works on unsigned attestations. Once every
team's CI has `id-token: write` configured, add `--require-signature` to
close the loop.
