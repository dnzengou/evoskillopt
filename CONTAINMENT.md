# CONTAINMENT.md — Clow Studio / EvoSkillOpt AI Containment Blueprint

> **Audience.** Any Claude Code session (or human engineer) opening this repo.
> This file is the authoritative, condensed containment contract for the
> `clow-mcp` runtime and the `EvoSkillOpt` candidate loop. Read it before
> proposing architecture, writing network / exec / log code, reviewing a PR, or
> operating the deployed service. The long-form source is
> `Clow_Studio_EvoSkillOpt_AI_Containment_Blueprint_Executable_Implementation_Plan.pdf`
> (Nov 25, 2026, E. Désiré Nzengou).

---

## 1. Core thesis — effect-based governance, not intent-based

Modern autonomous agents (multi-step, multi-agent, candidate-loop optimizers)
defeat prompt filters and output moderators because those act at the **intent
layer**. Containment must be enforced at the **effect layer**: the physical
boundary where code becomes real-world state change — sockets, file
descriptors, process groups, kernel memory, cryptographic state.

Documented 2025–2026 failure modes we are designing against:

1. **Boundary & egress leakage** — loopback / RFC1918 / link-local / cloud
   metadata (`169.254.169.254`) side channels.
2. **Emergent coordination & reward hacking** — out-of-band inter-agent buses,
   local reward-function edits.
3. **Trace & log tampering** — truncated / rewritten audit trails pre-shutdown.
4. **Unbounded loops & runaway workers** — `nohup` / daemon forks that
   outlive the parent task.

## 2. The four engineering pillars (non-negotiable)

| # | Pillar | Mechanism |
|---|---|---|
| 1 | **Pre-run verification** | Solvability, constraint, disk/quota, timeout checks before any context is allocated. |
| 2 | **Default-deny isolation** | Kernel process-group SIGKILL isolation, DNS pinning, socket-level IP dial control. |
| 3 | **Hash-chained effect telemetry** | Append-only SHA-256 JSONL audit trail over every external effect. |
| 4 | **Out-of-band kill switches** | Atomic SQLite WAL control plane with real-time worker termination and token revocation. |

Any change that weakens one of these pillars must be rejected or escalated to
the user before landing.

## 3. Trust zones (control-plane separation)

| Zone | Components | Responsibility | Guarantee |
|---|---|---|---|
| **0 — Trusted Control Plane** | SQLite WAL (`clow.db`), `clow-admin` CLI, persistence store | Kill-switch state, allowlists, revoked tokens, policy | R/W only from admin local process; SQLite in WAL mode |
| **1 — Egress Boundary** | `egressguard` pkg, socket dial controller, host resolver | Intercepts outbound TCP, resolves + classifies IPs before TCP handshake | Hard-coded IP blocks enforced in `net.Dialer.Control` |
| **2 — Execution Engine** | `clow-mcp` HTTP server, EvoSkillOpt candidate loop, process isolation | Tool invocation, candidate evaluation, sub-process spawn | No direct SQLite R/W; mediated API only |
| **3 — Audit & Verification** | `effectlog` auditor, file verifier, CI pipeline | Records state mutations; verifies chain integrity off-node | Append-only `0600` file perms; verifier independent of server |

**Rule:** Zone 2 (execution) never holds administrative authority over Zone 0
or Zone 3. The `clow-admin` CLI is deliberately out-of-band from the HTTP
surface so a compromised server cannot silently disable itself.

## 4. 12-point containment checklist

### A. Pre-run environment & solvability
1. **Solvability & constraint pre-flight** — verify context limits, disk
   quotas, timeout parameters, task definitions before starting.
2. **Clean environment init** — purge target dirs; strip ambient secrets
   (`AWS_SECRET_ACCESS_KEY`, `DATABASE_URL`, `KUBE_TOKEN`); restrict `TMPDIR`.
3. **Workspace integrity baseline** — SHA-256 file-tree of local dirs before
   execution so mutation is detectable.

### B. Sandbox, network & credential isolation
4. **Socket-level egress filtering** — `net.Dialer.Control` denies loopback
   (`127.0.0.0/8`, `::1`), RFC1918 (`10/8`, `172.16/12`, `192.168/16`),
   link-local, and metadata (`169.254.169.254`).
5. **Process-group hard SIGKILL isolation** — `SysProcAttr{Setpgid: true}`;
   on timeout/cancel, kill the whole process group.
6. **Token hashing & least-privilege credentials** — never persist raw tokens;
   store only SHA-256 digests for revocation checks.

### C. Effect-based monitoring & log integrity
7. **Hash-chained audit logging** — every external effect is one JSONL row
   where `entry_hash = SHA256(prev_hash || raw_json_bytes)`.
8. **Atomic synchronous log flush** — `File.Sync()` (`fsync`) after each entry
   before the effect proceeds.
9. **Real-time kill-switch evaluation** — check global halt + per-domain /
   per-action block list on every tool request and every candidate step.

### D. Release gating, escape testing & kill switches
10. **Persistent control-plane state in SQLite WAL** — strict concurrency, one
    durable truth for halt state, blocks, revocations.
11. **Out-of-band admin CLI (`clow-admin`)** — halt, resume, block, unblock,
    revoke, verify-log — must not share the HTTP binary's trust path.
12. **CI fixture audit pipeline** — pre-deploy jobs that dial loopback for
    real, verify chain continuity, fail on tampered fixtures, run `gosec` +
    `govulncheck`.

## 5. Reference architecture (Go)

The implementation is Go. Package boundaries and canonical responsibilities:

```
clow/
├── egressguard/        # Pillar 2: net.Dialer.Control, DNS pinning, IP classifier
│   └── client.go       #   NewClient(Config{AllowedHosts, Timeout}) -> *Client
├── effectlog/          # Pillar 3: append-only, SHA-256-chained JSONL log
│   ├── logger.go       #   NewLogger, WriteRecord (fsyncs), Close
│   └── verify_log.go   #   VerifyFile(path) -> (ok, count, lastHash, err)
├── controlplane/       # Pillar 4 (state): SQLite WAL, migrations, token hashing
│   └── store.go        #   IsHalted, SetHalted, IsBlocked, Block/Unblock, RevokeToken
├── killswitch/         # Pillar 4 (runtime): middleware, ticker, process-group exec
│   └── controller.go   #   Middleware(next) 503s when halted; ExecuteProcessGroup
└── cmd/
    ├── clow-mcp/       # HTTP MCP server, wires store+logger+guard+controller
    ├── clow-admin/     # Out-of-band CLI: kill-switch, token revoke, audit verify-log
    └── gen-fixtures/   # Deterministic valid + tampered JSONL for CI
```

Non-obvious invariants:

- `egressguard` validates **resolved IPs**, not just hostnames — DNS rebinding
  is an in-scope threat. `isRestrictedIP` covers loopback, private,
  link-local unicast & multicast, unspecified, and `169.254.169.254`.
- `effectlog` uses `GenesisHash = "00…00"` (64 zeros). `entry_hash =
  SHA256("seq|ts|actor|action|target|payload|prev_hash")`. **Every**
  `WriteRecord` calls `file.Sync()` before returning.
- `controlplane` opens SQLite with `PRAGMA journal_mode = WAL;` + `PRAGMA
  synchronous = NORMAL;`. Tables: `kill_switch_state` (singleton row,
  `id CHECK (id = 1)`), `kill_switch_blocks` (PK `(target_type,
  target_value)`), `revoked_tokens` (PK `token_hash`).
- `killswitch.Controller.localHalted` is an `atomic.Bool` refreshed by a
  1-second `StartSyncTicker` — the middleware never hits SQLite on the hot
  path. Refresh cadence is a correctness knob; shortening it is fine,
  lengthening it needs explicit sign-off.
- `killswitch.ExecuteProcessGroup` sets `Setpgid: true` and overrides
  `cmd.Cancel` to `syscall.Kill(-pid, SIGKILL)` so detached children die.
- `clow-mcp` request order on `/v1/mcp/tool`: **(1)** bearer present → **(2)**
  `IsTokenRevoked(token)` → **(3)** `IsBlocked("domain", target)` → **(4)**
  `logger.WriteRecord(...)` **(5)** execute. The audit record is written
  *before* the effect — never after.
- `clow-admin` subcommands: `kill-switch {status|halt|resume|block|unblock}`,
  `token revoke <raw>`, `audit verify-log <path>`.

## 6. CI / release gating (`.github/workflows/mcp-guardrails.yml`)

Required steps, in order, for a green build:

1. `actions/setup-go@v5` (Go 1.22, cache on).
2. `go mod tidy && git diff --exit-code`.
3. `gosec ./...` and `govulncheck ./...`.
4. `go test -v -race -count=1 ./...` — race detector is mandatory.
5. Build all three binaries (`clow-mcp`, `clow-admin`, `gen-fixtures`) with
   `CGO_ENABLED=0`.
6. Generate fixtures, then `clow-admin audit verify-log testdata/valid_chain.jsonl`
   **must pass** and `clow-admin audit verify-log testdata/tampered_chain.jsonl`
   **must fail** (CI inverts the exit code — a passing tampered verify is a
   build break, treated as a security failure).

Required Go tests at minimum:

- `egressguard.TestEgressGuard_BlocksRestrictedIPs` (loopback denied).
- `egressguard.TestEgressGuard_BlocksCloudMetadata` (`169.254.169.254` denied
  even with `AllowedHosts: ["*"]` — the IP check is the backstop).

## 7. Fly.io production runbook

**Container.** Multi-stage, `golang:1.22-alpine` builder → `alpine:3.19`
runtime. `CGO_ENABLED=0`, `-ldflags="-w -s"`. Non-root `clow:clow` user.
`/data/db` and `/data/audit` owned by `clow`. `ENTRYPOINT ["/usr/local/bin/clow-mcp"]`.

**fly.toml.** App `clow-mcp-governance`, region `iad`, volume `clow_data`
mounted at `/data`. `ENV DATABASE_URL=/data/db/clow.db`,
`EFFECT_LOG_PATH=/data/audit/effects.jsonl`, `PORT=8080`. HTTP health check
`GET /healthz` every 10s.

**Post-deploy smoke sequence.**

```bash
curl -i https://clow-mcp-governance.fly.dev/healthz
fly ssh console -C "ls -la /data/db /data/audit"
fly ssh console -C "/usr/local/bin/clow-admin kill-switch status"
fly ssh console -C "/usr/local/bin/clow-admin audit verify-log /data/audit/effects.jsonl"
```

**Operational command matrix.**

| Intent | Command |
|---|---|
| Emergency global halt | `fly ssh console -C "/usr/local/bin/clow-admin kill-switch halt"` |
| Resume operations | `fly ssh console -C "/usr/local/bin/clow-admin kill-switch resume"` |
| Block malicious host | `fly ssh console -C "/usr/local/bin/clow-admin kill-switch block domain badactor.com"` |
| Revoke leaked token | `fly ssh console -C "/usr/local/bin/clow-admin token revoke <RAW_TOKEN>"` |
| Audit verification | `fly ssh console -C "/usr/local/bin/clow-admin audit verify-log /data/audit/effects.jsonl"` |

## 8. Incident containment & rollback (four steps)

1. **Engage global kill-switch** → `clow-admin kill-switch halt`. Every
   `/v1/mcp/tool` request now returns 503 immediately via the middleware.
2. **Revoke compromised credentials** → `clow-admin token revoke <TOKEN>`.
   Hash goes into `revoked_tokens`; invalidation is cluster-wide on next
   request.
3. **Cryptographically audit effect log** → `clow-admin audit verify-log
   /data/audit/effects.jsonl`. Locates tamper point or confirms chain
   integrity.
4. **Resume after patch** → `clow-admin kill-switch resume`. Only after root
   cause is identified and a fix is deployed.

## 9. Rules for Claude Code sessions working in this repo

These are enforceable repo norms, not suggestions:

- **Never widen `egressguard` allowlists silently.** A new entry in
  `AllowedHosts` or any change to `isRestrictedIP` requires the user's
  explicit approval in the conversation and lands in its own commit.
- **Never log raw bearer tokens, raw API keys, or raw secrets** — log only
  hashes or short prefixes (e.g. `token[:8]`), matching the existing
  `handleToolExecution` pattern.
- **Never shorten the audit path.** Do not remove `fsync`, do not batch log
  writes, do not make audit writes conditional, do not reorder audit-after-
  effect. The invariant is: audit record is durable on disk *before* the
  effect runs.
- **Never bypass `killswitch.Middleware`** for any new HTTP handler. All
  effect-producing routes mount under the controller-wrapped mux.
- **Never introduce a direct SQLite R/W path from Zone 2** (the MCP server
  or candidate loop). Go through `controlplane.Store`.
- **Never spawn external binaries without `ExecuteProcessGroup`** (or an
  equivalent that sets `Setpgid` and kills the group on cancel).
- **Never add a new effect type** (new outbound call, new exec, new file
  write outside `/data`) without a matching `effectlog` `WriteRecord` call
  on the hot path and without a CI fixture that exercises it.
- **Never weaken CI** — the `gosec` / `govulncheck` / race-detector /
  tampered-log-rejection steps are load-bearing. Removing one is a security
  regression.
- **Never commit** files under `/data/`, `testdata/*.jsonl`, `*.db`,
  `*.jsonl` audit traces, or raw secrets.
- **When in doubt, halt first, ask second.** The kill-switch exists to be
  used; preferring `halt` over a risky mitigation is the correct default.

## 10. Repository context note

This repository (`evoskillopt`) currently holds the **conceptual and
GTM-facing** material for EvoSkillOpt (README, landing page, evolutionary
architecture notes). The Go runtime described above (`clow/`, `cmd/`,
`.github/workflows/mcp-guardrails.yml`, `Dockerfile`, `fly.toml`) is the
target state for the companion implementation repo. Treat this
`CONTAINMENT.md` as the contract the runtime must satisfy — when the Go
packages land, this file governs them.

---

_Source of truth: `Clow_Studio_EvoSkillOpt_AI_Containment_Blueprint_Executable_Implementation_Plan.pdf`._
_Changes to this document that relax any pillar, zone, or checklist item require explicit sign-off from the user._
