# Agent Zero Custom Build — Tracker

**Purpose:** audit trail and developer reference for the patched Agent Zero build
(all harness fixes applied) — its source branches, PRs, image checkpoints,
deployment state, and the procedures to verify, rebuild, and migrate it.

**Last updated:** 2026-08-02 19:05 UTC
**Maintained at:** this file (`~/agent-zero/BUILD-TRACKER.md`) and mirrored on the
fork branch `deploy/local-overlay` (`Zenetusken/agent-zero`).

---

## 1. Component inventory

| Component | Identity | Role |
|---|---|---|
| Upstream repo | `github.com/agent0ai/agent-zero` | Official product; maintainers review/merge our PRs |
| Fork repo | `github.com/Zenetusken/agent-zero` (public) | Carries all fix/integration/deploy branches; PR sources |
| Connector upstream | `github.com/agent0ai/a0-connector` | Official CLI repo |
| Connector fork | `github.com/Zenetusken/a0-connector` | Carries `fix/unbounded-reconnect` |
| Local clone (framework) | `~/src/agent-zero` | Remotes: `origin`=upstream, `fork`=Zenetusken |
| Local clone (connector) | `~/src/a0-connector` | Same remote layout |
| Worktrees | `/tmp/a0-integ` (integration), `/tmp/a0-browser-fix`, `/tmp/a0-test-suite`, `/tmp/a0-deploy`, `/tmp/a0-browser-base` (base ref `5ff106a2`) | Ephemeral; recreatable |
| Docker image | `agent-zero:local` (`9c37627520fe`) | v2.6 + integration tree baked into `/git/agent-zero` seed |
| Container | `agent-zero` (compose: `~/agent-zero/compose.yaml`) | UI at `127.0.0.1:5080` |
| Data volume | `agent_zero_usr` → `/a0/usr` | Settings, presets, secrets, chats — NOT in any image/repo |
| Connector CLI (host) | uv tool `a0` @ `27000e38` | Started manually: `a0 --host http://localhost:5080 --no-docker-discovery --connect` (interactive login required) |
| Build definition | `~/agent-zero/Dockerfile.local-overlay` | Also on fork branch `deploy/local-overlay` |

---

## 2. Fix arcs — PR checkpoints

Upstream base for all arcs: `5ff106a2` (2026-08-01, "Raise unusable response limit to five").

| Arc | Branch (fork) | Head SHA | PR | Validation |
|---|---|---|---|---|
| DeepSeek V4 Flash harness reliability | `fix/deepseek-harness-reliability` | `f31ce233` | [agent-zero#1798](https://github.com/agent0ai/agent-zero/pull/1798) | Targeted harness tests + core suite; truncation detector sanity checks |
| Test-suite usr guard + pollution + stale markers | `fix/test-suite-live-usr-guard` | `187faf84` | [agent-zero#1799](https://github.com/agent0ai/agent-zero/pull/1799) | Adversarial: failure set 7→4 with change; 3 stale markers fixed, 4 env-dependent (read-only `/a0/usr`) pre-existing |
| Timezone auto-persistence | `fix/timezone-auto-persistence` | `057a8c78` | [agent-zero#1800](https://github.com/agent0ai/agent-zero/pull/1800) | Branch regression tests |
| Browser harness reliability | `fix/browser-harness-reliability` | `c8670019` | [agent-zero#1801](https://github.com/agent0ai/agent-zero/pull/1801) | Failure set byte-identical vs base; forensic recovery of agent-written fixes + 3 review corrections |
| CLI unbounded reconnect | `fix/unbounded-reconnect` (connector fork) | `27000e38` | [a0-connector#20](https://github.com/agent0ai/a0-connector/pull/20) | 3 discriminating tests; failure set byte-identical to baseline |

All PRs: OPEN, MERGEABLE, no upstream CI configured. GitHub head == local == fork (verified 2026-08-02).

---

## 3. Build checkpoints

| Checkpoint | Value |
|---|---|
| Integration branch | `integration/live-deployment` (fork) = merges of all 4 agent-zero arcs |
| Integration SHA (current) | `8890c882b4f177d8c5e38c74065c4d931febaf2e` |
| Test status at integration | **1324 passed, 0 failed, 1 skipped** (throwaway-container suite, read-only usr mount) |
| Image | `agent-zero:local`, ID `9c37627520fe`, built 2026-08-02 12:22 EDT |
| Image recipe | `Dockerfile.local-overlay`: `FROM agent0ai/agent-zero:v2.6` → fetch fork → `git checkout $INTEG_SHA` in `/git/agent-zero` (build-arg `INTEG_SHA`, default `8890c882…`) |
| Deploy branch | `deploy/local-overlay` (see branch head) — standalone branch (Dockerfile + compose + this tracker) for any-host rebuild. Note: no SHA pinned here — every update to this file advances the branch, so a pinned value would be self-invalidating |
| Live container | `/a0` and `/git/agent-zero` both at `8890c882`, tree clean; all 6 supervisord services RUNNING |

### How the overlay works (why `/git/agent-zero` is the lever)

A container = read-only **image** + ephemeral **writable layer**. Edits made to `/a0`
in a running container live in the writable layer: they survive `docker restart`
but are discarded on **recreation** (compose change, `docker compose up`, host
migration). The stock image's boot mechanism is what makes a durable patch possible:

1. At upstream image build time, the repo is cloned into **`/git/agent-zero`** — a
   pristine seed baked into the image (`/ins/install_A0.sh`).
2. At every container start, `/exe/run_A0.sh` sources **`/ins/copy_A0.sh`**, which
   copies the seed into `/a0` **only if `/a0/run_ui.py` is missing** and with
   `cp -rn` (no-clobber) — so it populates a fresh `/a0` once and never overwrites
   an existing install afterward. Startup scripts perform **no** git resets.
3. Therefore: whatever tree the seed contains is what every recreated container
   starts with. The overlay (`Dockerfile.local-overlay`) adds one thin layer to
   stock v2.6 that fetches the fork's integration branch and checks out
   `$INTEG_SHA` **in the seed**. No recompilation, no venv changes — the runtime
   is byte-identical to stock except for the patched source tree.
4. Side effect of `cp --no-preserve=mode`: after seeding, `git status` in `/a0`
   shows mode-only diffs (lost exec bits); a one-time `git checkout -- .` restores
   canonical modes (done in §4.3).

`compose.yaml` points at `agent-zero:local`, so recreation always comes up patched.
Validated live 2026-08-02: container recreated from the image came up at
`8890c882` with a clean tree. Retire the overlay once upstream merges the PRs.

### Data checkpoints (secrets-bearing — keep private)

| Checkpoint | Value |
|---|---|
| usr volume backups | `agent-zero-usr-20260801-200256.tar.gz` (2.5 MB), `agent-zero-usr-20260802-180437.tar.gz` (0.8 MB, latest — includes agent0 profile + new project instructions) in `~/agent-zero/backups/` |
| Forensic artifacts | `~/agent-zero/debug/` (mode 700; browser-harness patch, failure extracts, connector log) |
| compose backups | `~/agent-zero/compose.yaml.backup-20260801-222814`, `…-20260802-122238` |

---

## 4. Procedures

### 4.1 Verify full sync (run any time)

```bash
# PRs state
gh pr list --repo agent0ai/agent-zero --author Zenetusken --state open
gh pr list --repo agent0ai/a0-connector --author Zenetusken --state open
# Branch parity (local == fork)
cd ~/src/agent-zero && git fetch fork && \
  for b in fix/deepseek-harness-reliability fix/test-suite-live-usr-guard \
           fix/timezone-auto-persistence fix/browser-harness-reliability \
           integration/live-deployment; do
    echo "$b $(git rev-parse --short=8 $b) $(git rev-parse --short=8 fork/$b)"; done
# Live container == integration
docker exec -w /a0 agent-zero git rev-parse HEAD
docker exec -w /a0 agent-zero git status --short   # expect empty
docker exec agent-zero supervisorctl status        # expect 6 RUNNING
```

### 4.2 Full test suite (integration tree)

```bash
docker run --rm -v /tmp/a0-integ:/a0 -v agent_zero_usr:/a0/usr:ro \
  agent0ai/agent-zero:v2.6 sh -lc \
  '/opt/venv-a0/bin/pip install -q pytest==9.1.1 pytest-asyncio==1.4.0 aiogram==3.30.0 \
   && cd /a0 && /opt/venv-a0/bin/python -m pytest tests/ -q'
```

Note: 4 tests (`test_http_auth_csrf` ×2, `test_defer_lifecycle`,
`test_parallel_child_contexts_are_chats_not_tasks`) fail on branches lacking the
`.env`-write guards when `/a0/usr` is mounted read-only — a harness artifact, not
a product bug. They pass on the integration tree.

### 4.3 Rebuild image after integration advances

```bash
cd /tmp/a0-integ && git rev-parse HEAD   # new SHA
cd ~/agent-zero && docker build -f Dockerfile.local-overlay \
  --build-arg INTEG_SHA=<new-sha> -t agent-zero:local .
docker compose up -d                      # recreates container onto new image
docker exec -w /a0 agent-zero git checkout -- . 2>/dev/null  # clear seed mode-artifacts
```

### 4.4 Deploy on a new host

```bash
curl -O https://raw.githubusercontent.com/Zenetusken/agent-zero/deploy/local-overlay/Dockerfile.local-overlay
curl -O https://raw.githubusercontent.com/Zenetusken/agent-zero/deploy/local-overlay/compose.yaml
docker build -f Dockerfile.local-overlay -t agent-zero:local .
docker compose up -d
# Restore model config/keys (optional but needed for DeepSeek presets):
docker run --rm -v agent_zero_usr:/data -v "$PWD:/backup" alpine \
  sh -c 'tar -xzf /backup/agent-zero-usr-20260801-200256.tar.gz -C /data'
# Connector CLI:
uv tool install git+https://github.com/Zenetusken/a0-connector.git@fix/unbounded-reconnect
```

Registry publishing (true `docker pull`) is blocked until: `docker login`
(Docker Hub) or `gh auth refresh -s write:packages` (GHCR). Then tag + push
`agent-zero:local` and record the registry ref in §3.

### 4.5 When upstream merges

1. `git fetch origin` in `~/src/agent-zero`; check which PR SHAs landed in `origin/main`.
2. Reconcile fork `main` with upstream; retire merged fix branches.
3. If ALL arcs merged: switch `compose.yaml` to the next official image tag,
   `docker compose up -d`, and retire `agent-zero:local` + overlay + integration branch.
4. If PARTIAL: rebuild overlay with an integration branch containing only unmerged arcs.

---

## 5. Watch list / open items

- [ ] PRs #1798–#1801, a0-connector#20 awaiting upstream review (no activity as of 2026-08-02; zero reviews, only self-comments)
- [ ] Image not yet published to any registry (local daemon only; fork rebuild is the portable path)
- [ ] `INTEG_SHA` is pinned — bump + rebuild when integration advances (§4.3)
- [ ] Do NOT delete fork or PR branches while PRs are open (kills the PRs)
- [x] ~~usr volume backup predates latest settings~~ — refreshed 2026-08-02 18:04 UTC (`agent-zero-usr-20260802-180437.tar.gz`)
- [ ] 4 read-only-mount test artifacts documented in §4.2 are not product bugs

## 6. Changelog

| Date (UTC) | Event |
|---|---|
| 2026-08-01 | Diagnostic arc: root-caused DeepSeek V4 Flash misformats + utility-model failure; presets recovered; circuit breaker raised |
| 2026-08-01 | PRs #1798–#1800 opened; connector `fix/unbounded-reconnect` pushed |
| 2026-08-02 | Browser fixes forensically recovered → PR #1801; regime-dependent test fixed |
| 2026-08-02 | a0-connector PR #20 opened; 5-PR set complete |
| 2026-08-02 | Live-agent test edit folded into #1801 (`c8670019`); 3 stale-marker fixes added to #1799 (`187faf84`); integration `8890c882` fully green (1324/0) |
| 2026-08-02 | Overlay image `9c37627520fe` built; container recreated onto `agent-zero:local`; seed mechanism validated live |
| 2026-08-02 | `deploy/local-overlay` branch published (`46284073`); this tracker created |
| 2026-08-02 17:59 | Tracker §3 expanded with the full overlay/seed mechanism explanation; deploy-branch SHA reference de-pinned (self-invalidating) |
| 2026-08-02 18:15 | Harness config: agent profile `default`→`agent0` (settings.json); project instructions replaced with closed-loop coding workflow (9604 chars, `project.json`); verified via `build_system_prompt_vars` + `initialize_agent`; run_ui restarted; fresh usr backup taken |
| 2026-08-02 19:05 | Subagent curation: `agents.json` written via harness API (hacker/tiny-local disabled; normalizer stores deviations only). Verified at registry, prompt-menu, and runtime-guard levels. LIVE DeepSeek validation passed: disabled-profile guard fired RepairableException and model self-repaired with exact error; developer-profile delegation ran full A1 subordinate loop; zero protocol misformats in 22 log entries |
