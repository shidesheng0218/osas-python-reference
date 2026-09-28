# OSAS Python Reference Implementation

A Python implementation of the [Open Support Agent Spec (OSAS)](https://github.com/shidesheng0218/open-support-agent-spec)
v0.2 proposal/execution surface plus the v0.3 Controlled Execution Sandbox
profile — self-contained in a single `server.py`.

This repository is the extraction of `implementations/python-reference/` from
the OSAS spec repository, maintained separately so it can serve as a second
implementation on the path to OSAS v1.0.

> **Independence note.** Per the OSAS
> [registry criteria](https://github.com/shidesheng0218/open-support-agent-spec/blob/main/conformance/README.md),
> an implementation counts as *independent* only with a separate repository
> **and** separate maintainership. This repo satisfies the first; for it to
> count toward the v1.0 gate it also needs a maintainer outside the founding
> organization. Interested? Open an issue.

## What it implements

- Discovery (`/.well-known/osas`, `/v1/capabilities`, `/v1/meta/tools`,
  `/v1/schemas[/…]`) served from the vendored `schemas/` directory.
- Real JSON Schema (draft 2020-12) validation for `/v1/validate` and policy
  drafts, via `jsonschema` + the `referencing` registry (the only runtime
  dependency — see `requirements.txt`).
- The deterministic policy-evaluation algorithm (spec §5), including
  param-transform passthrough.
- The policy version lifecycle (`draft → simulated → approved → active →
  retired`) with supersession, `POLICY_IMMUTABLE`, and RBAC
  (`policy_admin`-only mutations; cross-tenant access → `TENANT_MISMATCH`).
- Idempotent execution keyed by `(tenantId, idempotencyKey)`, `uncertain` →
  reconciliation (never retried), `/v1/proposals/:id/reconcile`.
- Per-tenant append-only audit hash chains (SHA-256 over canonical JSON — see
  the spec repo's `docs/implementing-osas.md`, "Reproducing the audit hash
  chain") with `/v1/audit/verify`.
- Conformance Mode (test-only, fail-closed) and the deterministic Sandbox
  execution path for the v0.3 profile.

Scope notes:

- It does **not** implement the approvals or shadow-run endpoints (its
  capability manifest declares only what it serves); the black-box suites do
  not exercise those surfaces, so its `conforming` registration covers the
  surfaces it does implement. Handoffs are record-only.
- RFC 0008 action-bound approvals: not yet implemented (approvals are not
  served here); the schema field `actionDigest` is optional, so conformance is
  unaffected.
- No external network calls, no real provider credentials, `live` execution
  refuses startup.

## Running

```bash
python3.12 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
python server.py --port 3010
```

Environment knobs mirror the spec repo: `OSAS_EXECUTION_MODE` (default
`shadow`; `sandbox` enables synthetic execution), `OSAS_CONFORMANCE_MODE` +
`OSAS_CONFORMANCE_KEY` (test-only), `OSAS_PROVIDER_EVENT_KEY`.

## Testing

```bash
python -m unittest        # 14 unit tests
```

Black-box conformance (requires Node ≥ 20 for the runner):

```bash
OSAS_CONFORMANCE_MODE=true OSAS_CONFORMANCE_KEY=test-key \
OSAS_PROVIDER_EVENT_KEY=test-provider-key OSAS_EXECUTION_MODE=sandbox \
  python server.py --port 3010 &

npx -y @osas/compat-runner --target http://127.0.0.1:3010 \
  --conformance-key test-key --out v02-report.json
npx -y @osas/compat-runner --target http://127.0.0.1:3010 \
  --profile controlled-execution \
  --conformance-key test-key --provider-event-key test-provider-key \
  --out v03-report.json
grep -q '"gateOk": true' v02-report.json && grep -q '"gateOk": true' v03-report.json
```

CI runs exactly this against the published runner — see
`.github/workflows/ci.yml`.

## Relationship to the spec repo

- `schemas/` and `conformance/fixtures/demo-tenant.json` are vendored copies of
  the spec's authoritative schema set, pinned per release; sync them when the
  claimed spec line changes (the black-box runner's discovery checks will catch
  drift).
- `server.py` differs from the in-repo original in exactly one line: the
  repository-root resolution (`ROOT = Path(__file__).resolve().parent`),
  because the layout is flat here.

## License

Apache-2.0 — see [LICENSE](LICENSE).
