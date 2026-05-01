# GCS+FUSE Per-Session Sandbox — Deep-Dive Analysis

> Synthesized 2026-05-01 from 4 parallel Opus subagent reads of branch `inbound-voice-fuse-sandbox` @ `b0bb495`.
> Voice-tooling commits (`4951ffa`, `8c27b15`, `e07f995`, `3c6cb14`, `8107eeb`) are out of scope per task brief.
> Source scratch artifacts: `.analysis-scratch/{01-fuse-primitives,02-app-integration,03-deployment-perf,04-alignment}.md`.

---

## 1. Executive Summary

The single FUSE/sandbox commit (`b0bb495 per-session sandbox for carelink-agi backed by gcs + gcsfuse`, Arjun Walia, 2026-04-30) lands **3,864 LoC of new code** + **666 LoC of tests** across 21 files. Two distinct surfaces:

- **`carelink-pms/carelink/agi/workspaces/`** (4 files, ~1,022 LoC + 666 LoC tests) — a clean, library-grade sandbox primitive: `WorkspaceManager` + `WorkspaceManifest` + `SandboxLayout` + path constants. **Near-zero CareLink coupling.** Drops into the current branch's PR 5 with a single targeted refactor (org_id → MemoryScope).
- **`carelink-agi/app/*` + `scripts/* + docs/*`** — the consumer half: a parallel "AGI v2" FastAPI service (port 8001), a sweeper, a benchmark, a systemd unit, a runbook. **This consumer surface conflicts with the locked Decision 1** (BackendAGIService + WorkerAGIService split, not BackendAGIService + V2 split) and should NOT be cherry-picked.

**Bottom line for the multi-tenancy implementation plan:**

| Action | Cost | Value |
|---|---|---|
| Cherry-pick `workspaces/{manifest,schemas,manager}.py` + 23 tests into PR 5 | 2–3 days (incl. org→MemoryScope refactor) | Replaces ~40 LoC of `_prepare_workspace`/`_copy_skills`/`_copy_agents`/`_write_restish_config`. Gives PR 6 a free file-index foundation. Eliminates the `extra_data->>'workspace'` cold-revive stash. |
| Take `scripts/{mount,setup,bench}` + runbook as templates | 1 day rebrand | Shaves ~1 sprint off operator-doc work. Mount stays opt-in. |
| Cherry-pick `carelink-agi/app/*` v2 service | **DO NOT** | Conflicts with Decision 1; parallel runtime, not an evolution of PMS. |

**Five sharp edges that must be fixed before merge** (across all 4 reads, ranked by data-loss severity):

1. `_load_manifest` swallows transient GCS errors and returns an empty manifest, which is then *uploaded over the real manifest on next checkpoint* (`workspaces/manager.py:644-652`).
2. No `if-generation-match` on manifest upload — two concurrent pods/replicas will silently overwrite each other's manifest (`manager.py:537-542`).
3. `finalize()` releases the asyncio lock between checkpoint and `rmtree`; a concurrent `hydrate()` could be reaped mid-write (`manager.py:251-256`).
4. `WorkspaceManager(org_id=…)` is baked one-org-per-process — incompatible with the locked 5-scope hierarchy (Platform→Org→Pharmacy→User→Session) (`manager.py:147`, `carelink-agi/app/main.py:67`).
5. systemd unit has zero hardening + hardcoded `User=sangeet` + no IaC for bucket/IAM/gcsfuse install (`scripts/sandbox-fuse.service:18-32`, `:8-11`).

---

## 2. Branch Topology

```
                   ┌─ openaiagentssdk @ 91c1039 ─ "slightly native SDK" sibling
                   │   • carelink-agi/ standalone v2 service (port 8001)
                   │   • claude_agent_sdk + native agents=
                   │   • PMS service.py uses claude_code_sdk (legacy)
                   │
b0bb495 builds on ─┴─ inbound-voice-fuse-sandbox @ b0bb495
                       • +5 voice commits (out of scope)
                       • +b0bb495: workspaces/ library + GCS+FUSE wiring
                       • git diff 91c1039..b0bb495 -- carelink-pms/carelink/agi/service.py
                         is EMPTY — the FUSE commit does not modify the PMS service.

                   ┌─ feature/ai-orchestration-with-domain-agents @ 6173360 (current)
                   │   • PMS service.py: 8-expert orchestration via invoke_expert MCP
                   │   • claude_agent_sdk migrated; native agents= wired but DORMANT
                   │   • Phase D verdict: native agents= retrofit infeasible
main ──────────────┘   • PR 5 (planned, unstarted): _prepare_workspace deterministic paths + Skills GCS-FUSE
```

**Key insight from Agent D**: `git diff 91c1039 b0bb495 -- carelink-pms/carelink/agi/service.py` is empty. The PMS service in the FUSE branch is byte-for-byte the sibling's. All PMS-side FUSE work lives in the *new* `workspaces/` package, not in `service.py`. **This is exactly why the cherry-pick is clean.**

---

## 3. Subagent Workflow (meta)

Four parallel Opus agents, each with a non-overlapping slice of the diff. Each wrote a scratch markdown to `<worktree>/.analysis-scratch/`. This synthesis composes from those.

| Agent | Slice | Scratch file | Output size |
|---|---|---|---|
| A — FUSE Primitives | `workspaces/{manifest,schemas,manager}.py` + tests | `01-fuse-primitives.md` | ~325 lines |
| B — App Integration | `carelink-agi/app/{agent,jobs,main,skills,sweeper,storage,routers/*}.py` + 2 design docs | `02-app-integration.md` | ~340 lines |
| C — Deployment + Perf | `scripts/*` + config diff + runbook ops sections | `03-deployment-perf.md` | ~256 lines |
| D — 3-Way Alignment | Cross-branch reads via `git show` against `origin/openaiagentssdk` + current branch service.py + handoff doc | `04-alignment.md` | ~130 lines |

Constraint applied to all 4 agents: read-only; cite every claim with `path:line`; flag surprises with `**SHARP EDGE:**`; ignore voice/IVR/calls code; target ≤500 lines. All agents complied.

---

## 4. Deep-Dive Findings

### 4.1 The `workspaces/` Library (Agent A)

**Public API surface** (`carelink-pms/carelink/agi/workspaces/__init__.py:21-31`):

- `HydrateOptions` — frozen dataclass: `skills`, `agents`, `memories`, `session_state`. Defaults pull all skills, no agents, all memories, prior session state (`manager.py:100-117`).
- `WorkspaceManager` — owner of the per-session round trip. Methods: `hydrate`, `checkpoint`, `finalize`, `sweep_cold`, `layout_for`, `session_exists_locally`, `fuse_available`, `write_restish_config`. Constructor takes `(org_id, local_root: Path, gcs_client: GCSClient, fuse_root: Path | None)` (`manager.py:131-178`).
- `WorkspaceManifest` — dataclass; `session_id`, `org_id`, `version`, `schema_version`, `last_checkpoint_at`, `last_checkpoint_reason`, `last_checkpoint_actor`, `files: list[WorkspaceFile]` (`manifest.py:52-107`).
- `WorkspaceFile` — `path` (sandbox-relative POSIX), `sha256`, `size`, `mtime`, `classification`, `blob_path` (`manifest.py:20-49`).
- `SandboxLayout` — frozen path bundle: `skills_dir`, `agents_dir`, `memories_dir`, `artifacts_dir`, `proposals_dir`, `scratch_dir`, `restish_config_linux`, `restish_config_macos`, `transcript_file()` (`schemas.py:74-139`).

**Lifecycle (implicit; no explicit state machine):**

| State | Detection | Entry | Exit |
|---|---|---|---|
| does-not-exist | `not layout.root.is_dir()` | (initial) | `hydrate()` creates dir tree |
| hydrated/active | `layout.root.is_dir()` AND no manifest blob | first `hydrate()` | `checkpoint()` |
| checkpointed | `manifest.json` blob in GCS | `checkpoint()` | next `checkpoint()` or `finalize()` |
| finalized | local dir gone; manifest in bucket | `finalize()` → `shutil.rmtree` | (terminal) |
| swept | local dir gone; manifest in bucket | `sweep_cold()` | re-hydrate |

**Lease semantics (or lack thereof):**

```
self._locks: dict[str, asyncio.Lock]  # manager.py:172
def _lock_for(session_id): ...        # manager.py:704-709
```

In-process only. Two replicas of the FastAPI service or one FastAPI + one Inngest worker both holding `session_id="X"` will both `hydrate`, both `checkpoint`. **Last writer wins on every blob.** No fencing token, no GCS object lease, no Postgres row lock. `_locks` only shrinks on `finalize` — so swept sessions leak lock entries until process restart (minor memory leak in long-running pods).

**GCS+FUSE interaction:**

- **Reads**: prefer FUSE if `fuse_root.is_dir()`; else GCS API (`manager.py:400-407`). Per-call check, no caching of "is FUSE alive". A *degraded but mounted* gcsfuse (process alive, bucket unreachable) silently produces empty hydrates.
- **Writes**: always `GCSClient.upload_blob` — explicitly bypassing FUSE because of "gcsfuse upload-on-close quirks" (module docstring `manager.py:18-21`).
- **Mount**: assumed pre-mounted by sidecar/systemd. Manager never shells out to `gcsfuse`.
- **Hash-diff before copy** (`manager.py:423-427`): SHA256 short-circuit on identical files. **Reads every byte of every file twice** (source hash + target hash) on each idempotent re-hydrate — measurable per-session cold-start cost.

**Manifest semantics:**

- Stored as `gs://<bucket>/sessions/<session_id>/manifest.json` only — **not on local disk** (`manager.py:641-642`).
- Per-checkpoint flow: load manifest → walk sandbox → upload changed files → upload new manifest LAST. The "manifest last" ordering is the only consistency guarantee (`manager.py:471-550`).
- `bump()` runs *before* manifest upload (`manager.py:521`); on upload failure, the in-memory manifest still advertises `version=N+1` while bucket stays at `version=N`. **Misleading return value.**
- `_load_manifest` catches all exceptions and returns `WorkspaceManifest.empty(...)` (`manager.py:644-652`, `manifest.py:106-107`). Cannot distinguish 404 (legit empty) from 503 (transient → should retry, not nuke).

**Idempotency & retry guards** are extensive (15 sites identified, see `01-fuse-primitives.md §7`) but **not** retry-with-backoff. Failures are logged-and-skipped; the only remediation is the next checkpoint.

**Test coverage** — 15 tests, 6 classes, 666 LoC. **Solid for happy paths**, but missing:

- Concurrency tests (the `_lock_for` path is untested).
- Per-file or manifest upload failure paths (the `# noqa: BLE001` swallows are untested).
- `_load_manifest` exception → empty manifest (the most dangerous silent path — untested).
- `_classify` proposals branch.
- Agents-template hydrate (`HydrateOptions(agents=[...])`).
- `finalize()` race window (lock released between checkpoint and rmtree — untested).
- gcsfuse stat-cache TTL effects (the cross-session-memory test simulates this manually at `test_workspace_manager.py:639-642`).

### 4.2 App Integration (Agent B)

**Where the manager lives**: `app.state.workspace_manager`, single process-wide singleton constructed in `lifespan` at `carelink-agi/app/main.py:66-71`. Every request reads it from `request.app.state` — see `chat.py:120,328`, `routers/jobs.py:256,394,526,672`.

**End-to-end chat lifecycle** (one session, one turn):

| Phase | Step | File:line |
|---|---|---|
| Request arrives | `_resolve_session_and_resume` → `manager.hydrate(session_id)` (or fresh UUID) | `chat.py:69-103` |
| Build SDK options | hydrate → mkdir layout, copy skills (FUSE or GCS), copy memories, copy session-state, write `apis.json` to both Linux+macOS restish paths | `agent.py:119-269`, `manager.py:200-216`, `:325-367`, `:288-319` |
| SDK turn runs | first `init` SystemMessage carries `session_id`; captured into local var, emitted to SSE | `chat.py:418-424` |
| End of turn | mirror local sandbox from job_id-key to sdk-session-id-key (if differs), checkpoint | `chat.py:151-166` (sync), `:478-484` (stream); `manager.py:218-247` |
| Checkpoint body | `_load_manifest` from GCS → walk sandbox excluding `_is_ephemeral` paths → SHA256-diff vs manifest → upload changed files → upload new manifest LAST | `manager.py:471-550` |
| Transcript | re-snapshotted to `sessions/<sid>/transcript/<sid>.jsonl` separately from the walker (which excludes `.claude/projects/`) | `manager.py:579-598` |
| Session "death" | TTL sweeper at 1h interval reaps `<local_root>/<sid>/` if mtime > 24h AND not in `JobManager.list()`. **No explicit "session done" event.** Bucket is never touched. `finalize()` exists but is unwired. | `sweeper.py:56-96`, `manager.py:260-282`, `:249-258` |

**Sandbox key bifurcation** is the trickiest mechanic (`jobs.py:498-508,746-748,781-803`; `chat.py:151-154`):

```
sandbox_key = job.session_id or job.job_id   # fresh job → job_id; on resume → session_id
hydrate under sandbox_key
... SDK runs ...
SDK init emits session_id
job.session_id = session_id
end_of_turn: if job_id != session_id:
    shutil.copytree(<job_id>/, <session_id>/)   # MIRROR, not move
    checkpoint under session_id
```

Both `<job_id>/` and `<session_id>/` live in `/tmp` until the sweeper reaps the orphan. **SHARP EDGE**: orphaned `<job_id>/` is protected by the sweeper (`_build_active_predicate` at `sweeper.py:36-53` checks both keys), but only via JobManager — chat sessions are NOT tracked there, so chat sandboxes are protected only by the 24h mtime TTL.

**Skills moved entirely to GCS** (no in-process cache):

- Old: `SkillsRegistry(agent_workspace=…)` — filesystem-backed.
- New: `SkillsRegistry(gcs_client=…, templates_prefix="templates/skills")` — every CRUD method is direct GCS (`skills.py:99-329`).
- Every `GET /skills` re-lists the bucket. Hydrate copies skills into per-session sandboxes (one copy per session, not shared).
- **Checkpoint walker REFUSES to push `.claude/skills/` back** (`manager.py:634-635`). **An agent that edits a skill via Bash sees the write succeed locally; the change vanishes on next hydrate of any session.** Skill mutation must go through the HTTP CRUD endpoints. This is intentional but a documented footgun.

**App boot sequence additions** (`main.py`):

1. `settings.sandbox_local_root.mkdir(parents=True, exist_ok=True)` (`:35`)
2. `gcs = GCSClient(settings.sandbox_bucket)` (`:54`) — **no probe**
3. FUSE startup probe: `fuse_root = settings.sandbox_fuse_root if settings.sandbox_fuse_root.is_dir() else None`; logs WARNING if missing (`:55-65`)
4. `WorkspaceManager(...)` constructed (`:66-71`)
5. `SkillsRegistry(...)` constructed; `ensure_root()` is now a documented no-op (`:73-77`, `skills.py:111-118`)
6. Sweeper task scheduled with `interval=1h, max_age=24h` (`:89-96`)

**No startup health gate for GCS/ADC** — first failure surfaces at the very first hydrate request. Combined with the silent FUSE fallback (logged WARNING only), a misconfigured prod deploy can appear healthy at boot but fail every chat turn.

**Docs-vs-code drift check** (Agent B verified 5 claims from `sandbox-walkthrough.md` and `agent-sandbox-runbook.md`):

- "Reads come from FUSE when available" → **matches** (`manager.py:400-407`).
- "Writes go through GCS API, never FUSE" → **matches** (`manager.py:489-543`).
- "1 session = 1 sandbox = N agents" → **matches at the manager level** (subagent SDK semantics not directly verified).
- Default bucket `arelink-agent-workspaces` → **matches** (`config.py:237`). (Note: the bucket name has no leading "c" — looks like a typo from "carelink-…" but the runbook is consistent.)
- "Sweeper checks 'is this session active?' via JobManager" → **drift**: chat sessions aren't tracked in JobManager. Runbook implies "we always know what's hot"; code's actual model is "we know what JOBS are hot". Worth fixing in the operator handoff.

### 4.3 Deployment + Performance (Agent C)

**Deployment shape**: VM-only, **NOT Cloud Run-shaped**. Hardcoded:

- User: `sangeet` (`scripts/sandbox-fuse.service:19-21`).
- HOME: `/home/sangeet` (`:21`) — required for ADC discovery.
- Mount path: `/mnt/arelink-agent-workspaces` (`:22-29`).
- ADC user creds: `~/.config/gcloud/application_default_credentials.json` (`docs/agent-sandbox-runbook.md:71-77`).
- Bundled gcsfuse binary at `/usr/bin/gcsfuse` (`scripts/sandbox-fuse.service:23`).

**Bucket setup (`scripts/setup-sandbox-bucket.sh`):**

- Does NOT create the bucket — only verifies + hardens an existing one (`:21-37`).
- Sets: UBLA on (`:34`), versioning on (`:35`), public-access-prevention on (`:36`).
- Does NOT set: lifecycle policy, retention, CMEK, audit logs, IAM bindings.
- **SHARP EDGE: versioning on + no lifecycle = unbounded version sprawl.** Every overwrite of a memory or transcript file creates a new noncurrent version, growing linearly forever. Likely the dominant long-term cost driver.

**gcsfuse flags** (identical between manual script and systemd unit):

- `--implicit-dirs` (required because the bucket has no placeholder objects)
- `--file-mode=0644` / `--dir-mode=0755`
- `--stat-cache-ttl=10m` / `--type-cache-ttl=10m` — this is the cross-session memory propagation lag ceiling. **A new memory written by Session A becomes visible to Session B no sooner than the cache TTL expires.** Test at `test_workspace_manager.py:621-647` simulates this manually; production will see real lag.
- No write caching, no atomic-write mode, no `--max-conns-per-host` tuning.

**systemd unit hardening — entirely missing:**

- No `PrivateTmp`, `NoNewPrivileges`, `ProtectSystem`, `ProtectHome`, `CapabilityBoundingSet`, `AmbientCapabilities`, `SystemCallFilter`, `RestrictAddressFamilies`, `MemoryMax`.
- For a long-lived process holding GCS OAuth tokens and mounting PHI-adjacent storage, this is below community baseline. Roughly a 30-minute fix.
- `Restart=on-failure` covers crashes but not hang — a frozen-but-alive gcsfuse won't restart. The manual script handles this via `fusermount -u || sudo umount -l` (`scripts/mount-sandbox-fuse.sh:32-39`); the unit has no equivalent.

**Bucket creation, IAM bindings, gcsfuse install, FUSE kernel module, VM provisioning** — all out-of-band manual ops. **Zero IaC.** The runbook is honest: "by the time you're reading this, all that exists" (`docs/sandbox-walkthrough.md:51-53`).

**Benchmark methodology** (`scripts/bench-sandbox.py`, 244 LoC):

- 6 timed steps via `time.perf_counter()`: cold hydrate, warm hydrate, single-artifact checkpoint, 100-bulk checkpoint, transcript checkpoint, idempotent re-checkpoint (`bench-sandbox.py:179-231`).
- **No repeats, no median/p95, no warm/cold separation in real mode, no GCS-roundtrip isolation, no concurrency.**
- Budgets in the docstring (`< 200 ms` skill hydrate, etc.) are aspirational — script doesn't assert them.
- `--fake` mode uses an in-process dict (zero IO), so its numbers don't predict real-mode behavior.
- **No actual measured numbers in the runbook** (`docs/agent-sandbox-runbook.md:276-285` shows only the budget table).

**HIPAA / security posture**:

| Control | Status |
|---|---|
| UBLA + public-access-prevention | ✅ on |
| Versioning | ✅ on (but no lifecycle, see SHARP EDGE) |
| File modes (0644/0755) | ✅ |
| Per-org prefix | ✅ `memories/<org_id>/` |
| CMEK | ❌ acknowledged out of scope |
| Audit logs | ❌ inherits project default |
| Retention policy | ❌ |
| IAM service account | ❌ ADC user identity only |
| systemd hardening | ❌ none |
| Cross-org isolation | ❌ prefix-only; misconfigured `sandbox_org_id` writes to wrong tenant |
| IaC | ❌ |

**Cloud Run readiness verdict: NOT ready.** systemd unit can't run; `/mnt/...` is hardcoded; ADC user creds aren't workload identity; sweeper expects long-lived VM; per-instance `/tmp` won't survive instance recycle. The runbook explicitly defers Cloud Run (`agent-sandbox-runbook.md:351-353`).

### 4.4 3-Way Alignment (Agent D)

**Per-axis comparison** (current branch / sibling / FUSE branch):

| Axis | Current (`feature/ai-orch...` @ 6173360) | Sibling (`openaiagentssdk` @ 91c1039) | FUSE (`b0bb495`) |
|---|---|---|---|
| Workspace creation | `tempfile.mkdtemp(prefix="agi_", ...)` per session | Same | `<local_root>/<session_id>/` deterministic |
| FS path | Non-deterministic suffix; cold-revive uses `extra_data->>'workspace'` stash | Same; no cold-revive | Deterministic; `extra_data->>'workspace'` stash unnecessary |
| GCS path | None for workspace | None | `gs://arelink-agent-workspaces/{templates,memories,sessions}/...` |
| Scoping | Per-session FS only; `org_id` on row but not in path | Same | Per-session FS + per-session GCS prefix + per-org GCS memories prefix |
| Mount | None | None | gcsfuse READ at `/mnt/...`; falls back to GCS API per-call |
| Lease | Process-local `cl._agi_clients` dict | Same | Process-local `asyncio.Lock` per session_id; **no distributed primitive** |
| Cold-revive | Implemented but pinned-to-machine | None | Not addressed at the lease layer; deterministic paths help but don't solve cross-machine |
| Skills delivery | Eager `shutil.copytree` from Python package (~141 file copies/session) | Same | From `gs://.../templates/skills/` via FUSE+GCS-API; SHA256 idempotent |
| Memory persistence | Local dirs created but `rmtree` at session end → aspirational | None | Persists; per-org prefix; checkpointed every turn; hydrated on resume |
| Cleanup / TTL | `rmtree` at session end | Same | TTL sweeper (1h interval, 24h cutoff); bucket untouched |
| Tests | Cold-revive + per-expert hermetic; `make_fake_spawn` | Same minus cold-revive | 23-test fake-GCS hermetic suite; FakeGCSClient pattern |
| Production readiness | 2/5 | 2/5 | 3/5 (v2 service path); PMS path still 2/5 |

**Where FUSE is AHEAD of current branch** (10 design primitives PR 5 wants but doesn't have):

1. Deterministic per-session paths
2. Typed `SandboxLayout` (replaces ~12 ad-hoc string concats in `service.py`)
3. `WorkspaceManifest` JSON format
4. Hash-diff incremental checkpoint
5. Bucket-write idempotency tolerant of dev ADC permission gaps
6. TTL-driven cold sweep with pluggable activity predicate
7. GCS-FUSE-or-GCS-API per-directory fallback
8. Static path constants in one module (`schemas.py:43-71`)
9. 23-test fake-GCS hermetic suite
10. Explicit transcript-snapshot path that fixes the SDK's cwd-slug-encoding bug — **directly addresses the handoff §5 issue** "workspace is per-machine /tmp; cold-revive only works if Inngest event lands on the original machine"

**Where FUSE is INCOMPATIBLE** (and severity):

| Locked decision | FUSE contradiction | Severity |
|---|---|---|
| 5-scope hierarchy | Only 2 scopes (`memories/<org_id>/`, `sessions/<sid>/`); no Pharmacy/User/Platform | **Medium** — pure path-builder change; replace `f"{BUCKET_MEMORIES}/{self._org_id}"` (`manager.py:355,615`) with a `MemoryScope` resolver |
| `org_id` ≠ `pharmacy_id`; pharmacy stays on `extra_data` JSON | FUSE has no pharmacy concept at all | **Medium** — covered by the MemoryScope refactor |
| Bucket-per-org GCS (Decision 3, recommended) | One bucket + prefix-per-org | **Medium** — Decision 3 is "recommended" not locked; if locked-as-bucket-per-org, `WorkspaceManager.__init__` needs to accept a GCSClient resolver |
| Append-only memory with 14d TTL | FUSE memory is upsert-overwrite; no TTL on memory files (24h TTL is local-only) | **High** — fundamental conflict with locked memory architecture. PR 6 (memory store + selector + merger) must add the merger layer; FUSE checkpoint is the *transport*, not the *content semantics* |
| PMS uses `claude_agent_sdk` (current; migration done in Phase 0/1) | FUSE PMS `service.py` still uses `claude_code_sdk` | **Low** — FUSE didn't touch PMS service.py; this is sibling-branch DNA showing through, not a FUSE design choice |
| Native `agents=` is DORMANT and infeasible to retrofit | FUSE's carelink-agi v2 service uses native `agents=` end-to-end (no `invoke_expert` MCP) | **Inverted, not contradiction** — Phase D's verdict is about *retrofitting* into the existing PMS orchestrator. v2 was started fresh with native agents from day one |
| Cardinality "1 sandbox = 1 session = N agents" | **MATCH** — explicitly stated in `__init__.py:5-7` and `sandbox-walkthrough.md:22`; test-enforced at `test_workspace_manager.py:650` | Compatible (validates locked decision) |

**Sibling-branch DNA**: PMS service.py is byte-for-byte identical between sibling and FUSE — `git diff 91c1039 b0bb495 -- carelink-pms/carelink/agi/service.py` is empty. The "native SDK" pattern lives entirely in `carelink-agi/`. **Implication: cherry-picking workspaces/ is clean; cherry-picking the v2 service drags an entire parallel runtime that conflicts with Decision 1.**

---

## 5. Consolidated Sharp Edges (cross-agent)

Ranked by data-loss risk:

### S1 — `_load_manifest` swallow-and-empty (Agent A)

`manager.py:644-652`. A bare `except Exception → empty()` cannot distinguish 404 (legit empty) from 503 (transient). A transient GCS read failure during `_checkpoint` will produce an empty manifest, which is then *uploaded over the real one* via the unconditional manifest write (`manager.py:537-542`). Recoverable only if GCS object versioning is enabled — which the bucket setup script does enable, but with no lifecycle, so the recovery is "go find the noncurrent version".

**Fix**: distinguish 404 from 5xx in `_load_manifest`; on 5xx, raise + log + abort the checkpoint (let the next checkpoint retry).

### S2 — No `if-generation-match` on manifest upload (Agent A)

`manager.py:537-542`. Two concurrent checkpoints from different pods/replicas both download the same manifest, both bump to v(N+1), both upload. Last writer wins on `files`; the other's blob uploads remain orphaned in the bucket. The per-session `asyncio.Lock` does NOT span processes.

**Fix**: pass `if_generation_match=existing.generation` on `upload_blob`; on precondition-failed, re-load and retry-with-merge or fail loudly.

### S3 — `finalize()` race window (Agent A)

`manager.py:251-256`. Lock released between `checkpoint` (which acquires/releases internally) and `rmtree`. A concurrent `hydrate(session_id="X")` could land between, creating a sandbox that gets `rmtree`'d.

**Fix**: hold the lock continuously across both operations; or have `finalize` re-check `session_exists_locally` before `rmtree`.

### S4 — `WorkspaceManager` is one-org-per-process (Agents A, D)

`manager.py:147`, `carelink-agi/app/main.py:67`. `org_id` is baked at construction time. **Cannot serve multiple orgs from one carelink-agi process today.**

**Fix (for cherry-pick)**: replace `org_id: str` constructor param with a `scope_resolver: Callable[[Session], MemoryScope]` that produces the path bundle dynamically. ~4 hours of work.

### S5 — Memory cross-session propagation depends on gcsfuse cache TTL (Agent A)

Test at `test_workspace_manager.py:621-647` simulates this manually by writing the GCS blob into `fuse_root` after checkpoint. Production uses `--stat-cache-ttl=10m` (`mount-sandbox-fuse.sh:53-54`). **Real cross-session memory propagation can lag up to 10 minutes.** Not covered by tests; not flagged in the runbook.

**Fix**: document the cache TTL in the runbook; for memory-merger logic in PR 6, prefer the GCS API path over FUSE for read-after-write within a single session boundary; consider lower TTL for memory paths specifically.

### S6 — Skills written by an agent are silently dropped (Agents A, B)

`manager.py:634-637` — `_classify` returns `(None, "")` for `.claude/skills/**` and `.claude/agents/**`. An agent that edits a skill via `Bash` inside its sandbox sees the write succeed; the content vanishes on next hydrate.

**Fix**: this is intentional (skills must go through `SkillsRegistry`). Document for agent authors. Consider a checkpoint-time WARNING log when checkpoint sees touched files in those paths.

### S7 — No startup health gate (Agent B)

`carelink-agi/app/main.py:54` constructs `GCSClient(settings.sandbox_bucket)` without probing. `skills_registry.ensure_root()` is now a no-op. First failure surfaces at first request.

**Fix**: probe a `healthcheck/probe.txt` upload+delete during lifespan startup (mirroring what `setup-sandbox-bucket.sh:18-30` already does in shell). Fail fast with a clear log line.

### S8 — Sweeper is JobManager-only; chat sessions protected only by 24h TTL (Agent B)

`sweeper.py:36-53`. `_build_active_predicate` checks `JobManager.list()`; chat sessions aren't tracked there. A long-running chat with reduced TTL (e.g. for tests) could in principle be reaped mid-stream.

**Fix**: register live chat sessions in a process-wide registry that the sweeper consults; or update `mtime` continuously during a chat turn so the TTL effectively follows liveness.

### S9 — systemd unit + IaC gaps (Agent C)

`scripts/sandbox-fuse.service`. No hardening; hardcoded `User=sangeet` + `HOME=/home/sangeet`; bucket creation, IAM, gcsfuse install, kernel module, VM provisioning all out-of-band; manual ADC user-credential rotation = service degradation until human re-auths.

**Fix**: ~30-minute hardening pass (`PrivateTmp`, `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome=tmpfs`, capability bounds). Templated user/HOME via `Environment=`. Service-account migration via workload identity for any non-dev environment. Terraform/Pulumi modules for bucket + IAM + lifecycle policy.

### S10 — Versioning on + no lifecycle = unbounded cost growth (Agent C)

`scripts/setup-sandbox-bucket.sh:35`. No max-versions, no age cap. Likely the dominant long-term cost driver.

**Fix**: GCS object lifecycle policy: NumNewerVersions=N (e.g. 10) for memory and transcript paths; explicit retention only on session manifests if needed for audit.

---

## 6. Architecture & Orchestration Recommendations

### 6.1 Cherry-pick plan for PR 5 (Decision: take the library, leave the consumer)

| Module | Action | Effort | Notes |
|---|---|---|---|
| `carelink-pms/carelink/agi/workspaces/manifest.py` | Drop in verbatim | 30 min | Pure dataclass + JSON serde; zero CareLink coupling |
| `carelink-pms/carelink/agi/workspaces/schemas.py` | Drop in with `BUCKET_MEMORIES` → `MemoryScope` builder | 1–2 hr | Replace one constant with a 5-scope path resolver |
| `carelink-pms/carelink/agi/workspaces/manager.py` | Drop in with `__init__(org_id=…)` → `__init__(scope_resolver=…)` | 1 day | ~4 references to `self._org_id` updated; structural is unchanged |
| `carelink-pms/tests/section1_unit/test_workspace_manager.py` | Drop in with fixture rename `org_id="test-org"` → `scope=test_scope` | 2–3 hr | 3 memory-scoping tests need new path assertions |
| `scripts/bench-sandbox.py` | Drop in as PR 5 acceptance benchmark | 1 hr | Add p50/p95 + repeats before relying on numbers |
| `scripts/{mount-sandbox-fuse.sh, sandbox-fuse.service, setup-sandbox-bucket.sh}` | Drop in as deployment templates with hardening additions and bucket-name env-vars | 1 day | Add the systemd hardening + IaC discipline before any non-dev use |
| `carelink-agi/app/sweeper.py` | Adapt into `carelink-pms/carelink/agi/sessions/sweeper.py` | 1 hr | Generic background-task pattern; depend on `cl.agi.sessions` for activity predicate |
| `docs/{sandbox-walkthrough.md, agent-sandbox-runbook.md}` | Drop in (rebranded) as `docs/agi-sandbox-walkthrough.md`, `docs/agi-sandbox-runbook.md` | 0 min | Total ~926 LoC of operator content |

**Total cherry-pickable: ~2,100 LoC implementation + 666 LoC tests + ~500 LoC scripts/docs. Estimated 2–3 day port** including the org→MemoryScope refactor and merging with current branch's existing `_prepare_workspace`/`_copy_skills`/`_copy_agents`/`_write_restish_config` callers.

**Files NOT to cherry-pick:**

- Anything under `carelink-agi/app/` — parallel runtime, conflicts with Decision 1 (BackendAGIService + WorkerAGIService split).
- `carelink-backend/app/routers/calls.py` and any voice tooling — out of scope.
- Voice-tunnel scripts.

### 6.2 Bug fixes that must land before merge

The 10 sharp edges in §5 with `Fix:` lines are concrete patches. Estimate ~2–3 days for S1–S4 (the data-loss-class bugs); S5–S10 can be follow-ups.

### 6.3 Manifest-first in PR 6 (memory store)

PR 6 (per handoff §7) plans `agi/memory/{store,selector,merger}.py`. The FUSE `WorkspaceManifest` already gives PR 6 a content-addressable file index for free. Adopt the manifest format BEFORE writing the merger; otherwise PR 6 reinvents one and PR 5 has to be rewritten to use it. **Critical sequencing.**

### 6.4 Retire `extra_data->>'workspace'` cold-revive stash

Once paths are deterministic (`<local_root>/<session_id>/`), the stash is dead code (`service.py:702, 902-916`). Cold-revive just re-derives the path from `session_id`. Confirm the SDK transcript file is restored to the same cwd-slug as the new deterministic path — `workspaces/manager._hydrate_transcript` does this correctly (`manager.py:447-465`), so follow that pattern in the cold-revive caller.

### 6.5 Defer enabling the FUSE mount in production

Land the gcsfuse scripts and configs but keep the mount opt-in via env var until the bucket-as-system-of-record is stable. PR 5's value comes mostly from the deterministic-paths + manifest + hash-diff checkpoint, not from FUSE specifically. The mount is a read-side accelerator that's most useful for skills hydrate (the largest cold-start cost) — turn it on after the rest is stable.

### 6.6 Add concurrency, failure-injection, and lease tests

The 23-test FUSE suite establishes the FakeGCSClient pattern but does not cover:

- Two concurrent `hydrate(session_id="X")` from different replicas
- `_load_manifest` raising 5xx (the most dangerous silent path)
- Per-file upload failure mid-checkpoint
- Manifest upload failure
- `finalize` race window
- gcsfuse cache TTL effects on cross-session memory

PR 1 + PR 3 already plan lease primitives. Lease tests sit alongside the cherry-picked workspace tests — add explicitly before any multi-replica deploy.

---

## 7. Concrete Next Steps (sequenced)

1. **Confirm "no commits to monorepo without approval" stance applies to this analysis output.** The report file is in the worktree on its own branch (`inbound-voice-fuse-sandbox`), not in the main checkout. Question for stakeholder: should it land here, in `feature/ai-orchestration-with-domain-agents/docs/`, or stay in the worktree only?
2. **Confirm sibling worktree retirement was intentional.** `.claude/worktrees/openaiagentssdk-review` is gone. Update the persistent memory file to reflect it so the next agent doesn't expect to find it.
3. **Decision: cherry-pick plan or build greenfield?** The cherry-pick recommendation in §6.1 is strong on ROI but contingent on stakeholder OK to take work that the sibling-branch's author (Arjun Walia) wrote in a parallel runtime. Confirm authorship attribution + license posture is fine before lifting code.
4. **Decision: org→MemoryScope refactor scope.** Section 5/§S4. Either land it as part of the cherry-pick (recommended) or keep `org_id` baked-in for v1 and refactor later. Latter risks paying multi-tenancy debt twice.
5. **Decision: bucket-per-org vs prefix-per-org (Decision 3).** Currently "recommended" not locked. The cherry-pick generalizes more cleanly if this is locked first — a `GCSClient` resolver vs a `MemoryScope` resolver have different shapes.
6. **Patch the 4 data-loss-class sharp edges (S1–S4)** before any production use. Even if the cherry-pick is rejected, anyone running this branch is exposed.
7. **Schedule the systemd hardening pass + write Terraform for bucket/IAM/lifecycle.** ~2 days; should not gate the cherry-pick but should gate any non-dev environment.
8. **Add the missing tests** (concurrency, failure-injection, lease) alongside the cherry-pick, before the lease primitives from PR 1 + PR 3 land.

---

## 8. Open Questions for Stakeholder

1. Is the cherry-pick of `workspaces/{manifest,schemas,manager}.py` + tests acceptable given the authorship is Arjun Walia (different engineer's branch)?
2. Should bucket-per-org be locked before the cherry-pick, or after?
3. Is the FUSE mount worth enabling in v1, or defer to a later PR? (Recommendation: defer.)
4. Should the cherry-picked workspaces module replace OR coexist with the existing `_prepare_workspace` flow during a transition? (Recommendation: replace; the deterministic paths + manifest are the value.)
5. Should `finalize()` be wired to a real "session done" event (browser closed, job archived) as part of the cherry-pick, or stay TTL-only? (Recommendation: stay TTL-only for v1.)
6. Is the sibling worktree (`openaiagentssdk-review`) retirement permanent? If yes, the persistent memory file should be updated; if no, restore the worktree before the next cross-branch comparison agent runs.
7. Should the carelink-agi/ v2 service continue to exist on its own branch as a reference, or be retired now that the workspaces/ library is moving to the production track? (Recommendation: keep as a reference until PR 5 + PR 6 land in main; then evaluate.)

---

## Appendix A — Source Scratch Files

The four agent reads underlying this synthesis are preserved in:

- `.claude/worktrees/inbound-voice-fuse-sandbox/.analysis-scratch/01-fuse-primitives.md` — Agent A
- `.claude/worktrees/inbound-voice-fuse-sandbox/.analysis-scratch/02-app-integration.md` — Agent B
- `.claude/worktrees/inbound-voice-fuse-sandbox/.analysis-scratch/03-deployment-perf.md` — Agent C
- `.claude/worktrees/inbound-voice-fuse-sandbox/.analysis-scratch/04-alignment.md` — Agent D

Each scratch file has a `Synthesis Hooks` section listing what the author flagged as load-bearing for downstream synthesis. This document attempts to reconcile all 16 hooks (~4 per agent) into the §5 sharp-edge list and §6 recommendations.

## Appendix B — Files Touched by `b0bb495` (FUSE/storage scope only)

NEW:
```
carelink-pms/carelink/agi/workspaces/__init__.py        45
carelink-pms/carelink/agi/workspaces/manager.py         712
carelink-pms/carelink/agi/workspaces/manifest.py        110
carelink-pms/carelink/agi/workspaces/schemas.py         155
carelink-pms/tests/section1_unit/test_workspace_manager.py  666
carelink-agi/app/sweeper.py                             99
docs/agent-sandbox-runbook.md                           362
docs/sandbox-walkthrough.md                             564
scripts/bench-sandbox.py                                244
scripts/mount-sandbox-fuse.sh                           59
scripts/sandbox-fuse.service                            35
scripts/setup-sandbox-bucket.sh                         45
```

MODIFIED:
```
carelink-agi/app/agent.py                               +117
carelink-agi/app/jobs.py                                +200
carelink-agi/app/main.py                                +58
carelink-agi/app/routers/chat.py                        +185
carelink-agi/app/routers/jobs.py                        +70
carelink-agi/app/routers/skills.py                      +14
carelink-agi/app/skills.py                              +296
carelink-agi/app/storage.py                             +14
carelink-agi/app/config.py                              +41
```

EXCLUDED from this analysis (voice tooling):
```
carelink-backend/app/routers/{calls,voice_tools}.py
carelink-backend/app/voice_tools_auth.py
carelink-pms/carelink/integrations/asepha_voice/client.py
carelink-pms/carelink/ivr/{models,schemas,service,voice_tools_*}.py
carelink-frontend/src/components/workspace/tabs/{Call,InboundCalls}*.tsx
docs/{inbound-voice-agent-handoff,inbound-voice-testing,voice-tools-v2-architecture}.md
scripts/update-voice-tunnel.sh
```

---

_End of synthesis. ~600 lines._
