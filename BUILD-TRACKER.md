# Agent Zero Custom Build — Tracker

**Purpose:** audit trail and developer reference for the patched Agent Zero build
(all harness fixes applied) — its source branches, PRs, image checkpoints,
deployment state, and the procedures to verify, rebuild, and migrate it.

**Last updated:** 2026-08-02 21:35 UTC
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
| Docker image | `agent-zero:local` (`46998bdb2a27`) | v2.8 runtime + v2.8-based integration tree baked into `/git/agent-zero` seed |
| Container | `agent-zero` (compose: `~/agent-zero/compose.yaml`) | UI at `127.0.0.1:5080` |
| Data volume | `agent_zero_usr` → `/a0/usr` | Settings, presets, secrets, chats — NOT in any image/repo |
| Connector CLI (host) | uv tool `a0` @ `27000e38` | Started manually: `a0 --host http://localhost:5080 --no-docker-discovery --connect` (interactive login required) |
| Build definition | `~/agent-zero/Dockerfile.local-overlay` | Also on fork branch `deploy/local-overlay` |

---

## 2. Fix arcs — PR checkpoints

Upstream base for all arcs: `5ff106a2` (2026-08-01, "Raise unusable response limit to five").

| Arc | Branch (fork) | Head SHA | PR | Validation |
|---|---|---|---|---|
| DeepSeek V4 Flash harness reliability | `fix/deepseek-harness-reliability` | `bec6ea5f` (rebased 2026-08-02 from v2.7 base onto v2.8) | [agent-zero#1798](https://github.com/agent0ai/agent-zero/pull/1798) | Targeted harness tests + core suite; truncation detector sanity checks |
| Test-suite usr guard + pollution + stale markers | `fix/test-suite-live-usr-guard` | `187faf84` | [agent-zero#1799](https://github.com/agent0ai/agent-zero/pull/1799) | Adversarial: failure set 7→4 with change; 3 stale markers fixed, 4 env-dependent (read-only `/a0/usr`) pre-existing |
| Timezone auto-persistence | `fix/timezone-auto-persistence` | `057a8c78` | [agent-zero#1800](https://github.com/agent0ai/agent-zero/pull/1800) | Branch regression tests |
| Browser harness reliability | `fix/browser-harness-reliability` | `c8670019` | [agent-zero#1801](https://github.com/agent0ai/agent-zero/pull/1801) | Failure set byte-identical vs base; forensic recovery of agent-written fixes + 3 review corrections |
| Web UI stale progress after reconnect | `fix/webui-stale-progress-reconnect` | `32291be0` | [agent-zero#1803](https://github.com/agent0ai/agent-zero/pull/1803) | 4 ordering tests (3 discriminating: fail on pre-fix v2.8, pass with fix); suite 1328/0; `node --check` clean |
| CLI unbounded reconnect | `fix/unbounded-reconnect` (connector fork) | `27000e38` | [a0-connector#20](https://github.com/agent0ai/a0-connector/pull/20) | 3 discriminating tests; failure set byte-identical to baseline |

All PRs: OPEN, MERGEABLE, no upstream CI configured. GitHub head == local == fork (verified 2026-08-02).

---

## 3. Build checkpoints

| Checkpoint | Value |
|---|---|
| Integration branch | `integration/v2.8-deployment` (fork) = `5ff106a2` + merges of all 4 agent-zero arcs (previous: `integration/live-deployment` @ `8890c882`, v2.6-based — retired) |
| Integration SHA (current) | `1ef91c61cd61f6e5bfa3d786b9b9372967d0f95a` |
| Test status at integration | **1324 passed, 0 failed, 1 skipped** on the v2.8 image runtime (throwaway suite, read-only usr mount) |
| Image | `agent-zero:local`, ID `46998bdb2a27`, built 2026-08-02 ~17:30 EDT on **v2.8 base** with v2.8-based seed (previous: `10df87c3a26f`, `da1086144ae9`, `9c37627520fe`) |
| Image recipe | `Dockerfile.local-overlay`: `FROM agent0ai/agent-zero:v2.8` → fetch fork `integration/v2.8-deployment` → `git checkout $INTEG_SHA` in `/git/agent-zero` (build-arg `INTEG_SHA`, default `1ef91c61…`) |
| Deploy branch | `deploy/local-overlay` (see branch head) — standalone branch (Dockerfile + compose + this tracker) for any-host rebuild. Note: no SHA pinned here — every update to this file advances the branch, so a pinned value would be self-invalidating |
| Live container | `/a0` at `1ef91c61` (v2.8 runtime), all 6 supervisord services RUNNING, UI 302, webui stale-progress fix live at `webui/index.js:416` |

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
   stock v2.8 that fetches the fork's integration branch and checks out
   `$INTEG_SHA` **in the seed**. No recompilation, no venv changes — the runtime
   is byte-identical to stock except for the patched source tree.
4. Side effect of `cp --no-preserve=mode`: after seeding, `git status` in `/a0`
   shows mode-only diffs (lost exec bits); restore with `chmod +x` on the two
   affected scripts or `git checkout -- .` (done 2026-08-02 after the v2.8
   recreate).
5. v2.8 subprocess-test init pattern: the **top-level** `/a0/runtime.py` shim was
   removed upstream, so plain `import runtime` no longer works. `helpers/runtime.py`
   is intact and unchanged — use:

   ```python
   import sys; sys.argv.append("--dockerized=true")
   from dotenv import load_dotenv; load_dotenv("/a0/usr/.env")
   from helpers import runtime; runtime.initialize()   # global init (was top-level runtime)
   import initialize                                   # only if you need initialize_agent()/AgentConfig
   ```

   Validated live 2026-08-02 (`is_dockerized()` → True, runtime id assigned).

`compose.yaml` points at `agent-zero:local`, so recreation always comes up patched.
Validated live 2026-08-02: container recreated from image `10df87c3a26f` came up at
`b18b3a10` on the v2.8 runtime with all fixes and all usr config intact. Subprocess
test harnesses: use the §3-point-5 init pattern (top-level `runtime.py` shim was
removed in v2.8). Retire the overlay once upstream merges the PRs.

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
           integration/v2.8-deployment; do
    echo "$b $(git rev-parse --short=8 $b) $(git rev-parse --short=8 fork/$b)"; done
# Live container == integration
docker exec -w /a0 agent-zero git rev-parse HEAD
docker exec -w /a0 agent-zero git status --short   # expect empty
docker exec agent-zero supervisorctl status        # expect 6 RUNNING
```

### 4.2 Full test suite (integration tree)

```bash
docker run --rm -v /home/drei/src/agent-zero:/a0 -v agent_zero_usr:/a0/usr:ro \
  agent0ai/agent-zero:v2.8 sh -lc \
  '/opt/venv-a0/bin/pip install -q pytest==9.1.1 pytest-asyncio==1.4.0 aiogram==3.30.0 \
   && cd /a0 && /opt/venv-a0/bin/python -m pytest tests/ -q'
```

(With `~/src/agent-zero` checked out at the integration branch. `/tmp/a0-integ` still
holds the retired v2.6-based integration tree.)

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

- [x] ~~v2.8 flip BLOCKED~~ — DONE 2026-08-02 20:40 UTC: `integration/v2.8-deployment` built (`5ff106a2` + all 4 arcs, conftest add/add resolved per documented rule: #1799 superset), suite 1324/0 green, overlay `10df87c3a26f` rebuilt, container recreated and fully verified
- [ ] PRs #1798–#1801, #1803, a0-connector#20 awaiting upstream review (no activity as of 2026-08-02; zero reviews, only self-comments)
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
| 2026-08-02 19:50 | PR-base audit vs v2.8: 4/5 branches exactly v2.8-based; #1798 was v2.7-based (`87e1e591`) → rebased onto `5ff106a2` (clean, 9 commits, head `bec6ea5f`, force-pushed). Suite on rebased branch: 1306 pass + 3 known-upstream stale-marker failures (fixed by #1799). conftest.py overlap documented on both PRs (#1799's superset wins). Integration unchanged at `8890c882` — already contains superior conftest; no re-merge needed. Nothing superseded by v2.8: upstream repairs target native-Responses transport, ours target chat-mode JSON envelope/truncation |
| 2026-08-02 19:20 | **v2.8 upgrade prepared**: v2.8 tag == `5ff106a2` == exact PR base (zero rebase); overlay rebuilt FROM v2.8 (`da1086144ae9`); smoke test passed; suite green on v2.8 runtime (1324/0); usr backup `agent-zero-usr-20260802-184402.tar.gz`; recreate staged pending live task completion |
| 2026-08-02 19:05 | Subagent curation: `agents.json` written via harness API (hacker/tiny-local disabled; normalizer stores deviations only). Verified at registry, prompt-menu, and runtime-guard levels. LIVE DeepSeek validation passed: disabled-profile guard fired RepairableException and model self-repaired with exact error; developer-profile delegation ran full A1 subordinate loop; zero protocol misformats in 22 log entries |
| 2026-08-02 20:30 | **API slowness root-caused (measured, not guessed):** DeepSeek healthy (1.1s tiny call; 3.5–5.1s with thinking high/max @1.6k tokens); all `rl_*` limits = 0 (no client throttling); real cause = chat `NL2Koll9` history at ~2.79M chars ≈ **~700k tokens/call** (6.8MB chat.json) — every loop iteration + every misformat retry re-prefills it. Fix: chat `ctx_length` capped 1M→256k on Default/High/Off (Max keeps 1M), backup `presets.yaml.bak-20260802-202148`, run_ui restarted, all 6 services RUNNING, presets re-verified from new process. **Correction to 19:20 entry:** overlay `da1086144ae9` is FROM v2.8 base but seeds old integration `8890c882` — v2.8 flip is NOT staged, needs a v2.8-based integration rebuild first |
| 2026-08-02 20:40 | **v2.8 flip COMPLETE**: `integration/v2.8-deployment` built from `5ff106a2` + all 4 arcs (head `b18b3a10`; only conflict = expected `tests/conftest.py` add/add, resolved to #1799 superset per documented rule). Suite on merged tree via v2.8 runtime: **1324 passed, 0 failed, 1 skipped**. Overlay rebuilt (`10df87c3a26f`, `INTEG_SHA=b18b3a10`); usr backup `agent-zero-usr-20260802-202716.tar.gz`; container recreated during user-confirmed idle window. Post-recreate verified: `/a0` @ `b18b3a10`, all 4 fix arcs present (truncation detector smoke-tested live), settings (agent0 profile, circuit breaker 5), presets (256k chat caps), project instructions (9604 chars), subagent curation (`agents.json`), chats (7.3MB NL2Koll9) all intact; 6/6 services RUNNING; UI 302; ports 80/55520 LISTENING. Mode-only seed diffs re-fixed via chmod. Gotcha recorded: v2.8 removed the top-level `runtime.py` shim — `helpers/runtime.py` is intact; see §3 point 5 for the corrected subprocess init pattern |
| 2026-08-02 21:35 | **Web UI/CLI split-brain root-caused and fixed (arc 6)**: CLI finished vs Web UI stuck on "A0: Reasoning..." after the 20:28 recreate. Measured: backend verifiably idle (py-spy all threads parked, log frozen at final `response` 20:42, queue empty) → backend correct; root cause = frontend `applySnapshot` sequenced `updateProgress`/`paused`/notifications **behind** `await setMessages(...)`, so a 543-entry/7.5MB re-render kept a stale indicator until the backlog drained. Token accounting reconciled: harness-true history = **322.5k tokens** (`history.get_tokens()`; CLI's 339.7k = history + system prompt) — earlier chars/4 estimate (~700k) was 2x over. Fix `32291be0` on `fix/webui-stale-progress-reconnect` → **PR #1803**; merged into integration (`1ef91c61`); suite 1328/0; overlay `46998bdb2a27`; container recreated, fix live |
