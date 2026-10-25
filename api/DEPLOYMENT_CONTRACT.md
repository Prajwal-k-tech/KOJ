# KOJ Judge — Deployment Contract (Phase 2 Commit 1)

Scope: the FastAPI judge service in `api/` plus migration
`0003_judge_hardening.sql`. Parent (Next.js `app/`, contests, UX)
owns dispatch wiring, recovery, and verdict display — open items are
listed at the bottom. `rlimit` is a compatibility mode with process limits,
not a security boundary for untrusted submissions. A host that cannot run
containers must not accept public submissions unless execution is delegated
to an equivalent isolated worker.

> **Deployment gate:** `auto` uses Docker and fails closed when Docker is
> unavailable. The explicit `rlimit` mode runs code on the host without
> filesystem or network namespaces and is only for controlled development.
> Verify the backend from `/health` and keep public judging disabled until a
> Docker-capable isolated worker (or equivalent) is provisioned and verified.

## 1. Host

- Linux x86_64 with Docker Engine (daemon + CLI) and cgroup v1 or v2
  with memory/pids/cpu controllers enabled. Verify:
  `docker info`, `docker run --rm hello-world`,
  `cat /sys/fs/cgroup/cgroup.controllers` (v2) shows `memory pids cpu`.
- The service runs as a container or systemd unit **with access to the
  Docker socket or TCP daemon** (see §3). A serverless host (Cloud Run,
  no Docker daemon, no `--privileged`, no sibling containers) cannot
  satisfy §1 — use a VM (GCE e2-medium or larger), GCE
  Container-Optimized OS, or equivalent. Standard Cloud Run cannot satisfy
  the Docker-per-case requirements; use a separate isolated worker for code
  execution.

## 1a. Sandbox backend selection (`JUDGE_SANDBOX_MODE`)

`api/app/judge.py` implements two backends:

| Mode | Behaviour |
|---|---|
| `docker` | Everything in this document. Isolation via container namespaces, seccomp, cgroups. |
| `rlimit` | Host toolchain with `RLIMIT_CPU/AS/NPROC/FSIZE`, wall-clock timeout, and a fresh process session per run. No filesystem or network namespace; controlled development only. |
| `auto` (code default) | Docker only. Fails closed if Docker is unavailable. |

`GET /health` reports the active backend as `sandbox`; check it after every
deployment and alert if it differs from the configured backend. The deployed
backend and host must be verified directly; do not infer them from `auto` or
from this document. rlimit-specific properties you must not
forget when changing judge code:

- `RLIMIT_NPROC` counts every task of the real UID (threads included). It must be derived
  from the host's live task count (`/proc/loadavg` field 4) plus headroom, or `g++`/`javac`
  fail with `posix_spawn: Resource temporarily unavailable`.
- `RLIMIT_AS` cannot be used for the JVM, the Go arena, or V8 — those runtimes reserve far
  more virtual address space than they touch. Java is bounded by `-Xmx`, Go by `GOMEMLIMIT`,
  Node by `--max-old-space-size`, all derived from the problem's memory limit.
- `RLIMIT_FSIZE` applies to the **run** phase only. Applying it while compiling makes
  linkers (e.g. rustc) fail on binaries larger than the bound and surfaces as a bogus
  compilation error.
- Compiling with a missing toolchain is infrastructure failure
  (`SandboxUnavailable` → infra-flagged `runtime_error`), never a contestant CE. Each
  language declares its required binaries in `_LOCAL_TOOLCHAIN`.
- Go needs a writable `GOCACHE`; it is shared process-wide
  (`tempfile.gettempdir()/koj-go-build-cache`) and warmed in a background thread at start-up,
  otherwise every submission recompiles the standard library (10s+ each).
- Kernel with seccomp enforcement (`CONFIG_SECCOMP`, `CONFIG_SECCOMP_FILTER`).
  Verify: `grep -i seccomp /boot/config-$(uname -r)`.
- Single uvicorn worker (`--workers 1`, see `api/Dockerfile`).
  Judging is CPU-bound and fans out to `docker run`; more workers only
  contend on the same daemon. Scale by sizing the host (§7), not workers.

## 2. cgroups / resource enforcement

- Every `docker run` sets `--memory`/`--memory-swap` (equal, no swap),
  `--cpus`, `--pids-limit`, `fsize`/`nproc` ulimits. Problem limits map
  through `_clamp_memory_mb` (64–1024 MB); compile uses a 256 MB floor
  (`_compile_memory_mb`); Java run uses a 256 MB floor
  (`_run_memory_mb`) so the JVM never OOMs on hello-world.
- OOM signal: cgroup kill surfaces as container exit **137**
  (128+SIGKILL) or -9. The judge maps 137/-9 (plus `Out of memory`,
  `MemoryError`, Java `OutOfMemoryError`, C++ `bad_alloc`) to
  `memory_limit_exceeded`. Post-mortem `docker inspect` OOMKilled is
  deliberately unavailable: containers run with `--rm` and timed-out
  containers are force-removed (`docker rm -f`) so no debris accumulates.
- If the host cannot enforce memory/pids (e.g. cgroup controllers
  missing), Docker still starts the container but limits are advisory.
  Treat `docker info` warnings about missing cgroup support as a
  deployment blocker for contests.

## 3. Docker daemon / socket risk

- The judge shells out to the `docker` CLI with a fixed argv; **no
  request field sets images, mounts, flags, or commands**. Images come
  only from `JUDGE_DOCKER_IMAGE_*` env vars.
- Daemon access is full host root-equivalent. Mitigations, all required:
  - Run the FastAPI service as an unprivileged UID; grant **only**
    membership in the `docker` group (socket `/var/run/docker.sock`
    mode `660 root:docker`) or a TLS-protected TCP daemon. Never expose
    the socket to contestant code (containers get `--network none`,
    no socket mount).
  - Socket mount (if the service itself is containerized):
    `-v /var/run/docker.sock:/var/run/docker.sock:ro` — read-only mount
    of the socket path (daemon access is still privileged; the `ro`
    flag only prevents remount tricks).
  - Audit: restrict SSH, rotate daemon TLS certs, monitor
    `docker events` for unexpected image pulls/runs.
- Daemon unreachable, missing `docker` binary, or seccomp errors map to
  `SandboxUnavailable` → infra-flagged pending (never contestant
  `runtime_error`). No host fallback: without Docker nothing executes.

## 4. Seccomp profile provisioning

- Repo source: `api/seccomp-koj.json` (vendored Docker/Moby-default
  derivative, `defaultAction: SCMP_ACT_ERRNO`, fail-closed). It is NOT a
  minimal hand-rolled allowlist and makes no exhaustive-audit claim;
  dangerous families (mount, kexec, modules, `open_by_handle_at`,
  `clock_settime`, ptrace, bpf, io_uring, userfaultfd, perf, keyctl)
  are omitted by design, preserving Docker default denials.
- Provision to the **daemon-host** path (default
  `/etc/koj/seccomp-koj.json`, override via `JUDGE_SECCOMP_PROFILE`):
  ```bash
  sudo mkdir -p /etc/koj
  sudo cp api/seccomp-koj.json /etc/koj/seccomp-koj.json
  sudo chmod 644 /etc/koj/seccomp-koj.json
  docker run --rm --security-opt seccomp=/etc/koj/seccomp-koj.json \
    python:3.11-slim true && echo SECOMP_OK
  ```
- Every judge container passes
  `--security-opt seccomp=<JUDGE_SECCOMP_PROFILE>` plus
  `no-new-privileges`. If the daemon reports a missing/invalid profile,
  the run raises `SandboxUnavailable` (fail closed, infra-flagged).
- If the daemon runs on a different host than the FastAPI process, copy
  the profile to the **daemon** host, not the API host. Remote-daemon
  setups must document which host holds the file.

## 5. Images: pre-pull and inspection

- Pre-pull (or override consistently in env + contract):
  ```bash
  docker pull python:3.11-slim gcc:13-bookworm eclipse-temurin:17-jdk-jammy \
              golang:1.22-bookworm rust:1.79-bookworm node:20-bookworm-slim
  ```
- Judge runs with `--pull never`: deployments never pull at judge time
  (no registry-in-the-loop, no tag mutation mid-contest). Pin digests in
  production:
  `docker inspect --format='{{.RepoDigests}}' <image>` and record them
  in the contest runbook.
- Inspect before contests: `docker images`, `docker inspect <image> |
  grep -i seccomp`, and a smoke run per language (§9).

## 6. UID/GID and filesystem boundary

- Containers run `--user 65534:65534` (`JUDGE_DOCKER_USER`), `--read-only`,
  `--cap-drop ALL`. Sources mount **read-only** (`/sandbox:ro`); build
  artifacts go to the writable exec tmpfs `/scratch` (64 MB):
  C/C++ `-o /scratch/solution`, Java `javac -d /scratch` + `java -cp
  /scratch`, Python runs from `/sandbox` with bytecode redirected to
  `/tmp` (`PYTHONDONTWRITEBYTECODE=1`, `PYTHONPYCACHEPREFIX=/tmp`).
  `/tmp` is a separate noexec tmpfs (64 MB). Host tempdirs
  (`koj-judge-*`) hold only sources and are removed on return.
- The host workdir path is server-generated (`tempfile`), never
  user-controlled. No user-controlled `-v`, `--tmpfs`, `--env`, image,
  or flag is ever interpolated.

## 7. Pool and concurrency sizing

- `JUDGE_MAX_CONCURRENT` (default 4, validated 1–32) bounds concurrent
  judgements per worker via a non-blocking semaphore. Saturated `/judge`
  and `/judge-async` callers get **503 + `Retry-After: 5`**; nothing
  blocks indefinitely. Size guidance: `JUDGE_MAX_CONCURRENT ≈
  host_CPUs` for C/Python, `≈ host_CPUs / 2` when Java is heavy
  (JVMRSS + Metaspace). A 4-vCPU host with default limits handles 4
  concurrent judges; raise only with load evidence.
- psycopg pool (`api/app/db.py`): `min_size=1, max_size=10` per worker,
  `timeout=5s`, autocommit. Keep `max_size ≥ JUDGE_MAX_CONCURRENT + 2`
  (judge slot + health + unlock paths). Defaults (10 ≥ 4+2) satisfy this;
  raise both together if `JUDGE_MAX_CONCURRENT` grows.
- Next.js pool (`db/index.ts`): `max: 10`. Neon pooled DSN required
  (`sslmode=require`). Total backend connections stay under Neon's
  pooled limit: Next(10) + judge(10) + margin.

## 8. Secrets and env

- Required: `DATABASE_URL` (Neon pooled + `sslmode=require`),
  `JUDGE_INTERNAL_SECRET` (identical on Next + FastAPI),
  `FRONTEND_URL` (Vercel origin for CORS). Never commit `.env`;
  only `.env.example` is tracked. Never log `settings` or the DSN.
- `JUDGE_INTERNAL_SECRET` empty disables judging (endpoints 500, lifespan
  raises). Rotate by updating both sides; mismatched secrets 401 without
  leaking which side is wrong (constant-time compare).
- CORS: `http://localhost:3000` + `FRONTEND_URL` only; methods
  `GET/POST/OPTIONS`; headers `Content-Type, X-Judge-Secret,
  Authorization, Idempotency-Key`; no credentials; never `*`.

## 9. Health and smoke matrix

- Liveness: `GET /health` → 200 `{"status":"ok"|"degraded","db":bool}`.
  Degraded (DB down) stays 200 so the LB keeps the task; judging then
  infra-flags pending.
- Smoke (Docker-capable Linux host only; record unavailable hosts as a
  limitation, never a pass):
  1. `POST /judge` python hello (sample) → `accepted`, sample stdout kept.
  2. `POST /judge` python hello with `is_sample=false` → `accepted`,
     case stdout/stderr `""` (redacted).
  3. `POST /judge` C with `-fstack-protector` smoke → `accepted`.
  4. `POST /judge` Java `class Solution` + `// class Solution` decoy →
     accepted only for the real declaration; `/* class Solution */`-only
     source → `compilation_error`.
  5. OOM probe (e.g. `x = bytearray(2**31)` under small `memory_mb`) →
     `memory_limit_exceeded` (exit 137 path on Docker).
  6. Timeout probe (`while True: pass`, tiny `time_limit_ms`) →
     `time_limit_exceeded`, container gone (`docker ps -a | grep koj-judge`
     empty).
  7. Seccomp negative: temporarily rename the host profile → judgment
     returns infra-flagged pending/503, service stays up.
  8. Saturation: set `JUDGE_MAX_CONCURRENT=1`, hold a slow judgement,
     second `POST /judge` → 503 + `Retry-After`.
  9. Idempotency: `POST /judge-async` twice (same `submission_id`) →
     second returns stored status without re-judging; post-completion
     retry → 200 `{"deduped":true}`.
  10. Adversarial (record results, do not run unobserved on shared hosts):
      fork bomb, `/proc`/`/sys` reads, socket connect, binary writing
      >1 MB stdout, `open('/sandbox/...','w')` (must fail read-only),
      `__pycache__` write (must land in `/tmp`), Java thread spray
      (pids-limit holds), seccomp-blocked syscall (EPERM/ERRNO, no escape).
- Request-body logging check: `POST /judge` with a canary string, then
  `grep` service logs — the canary must appear nowhere. Uvicorn access
  logs record method/path/status only; no code path logs bodies or case
  content (enforced by review, not by a log scrubber).

## 10. Logging contract

- Application logs carry submission/problem IDs, verdicts, counts, and
  truncated daemon/cleanup errors only. Case `stdin`, `expected_stdout`,
  program stdout/stderr, and `code` are never logged.
- Cleanup failures (`docker rm -f` non-zero/timeout) log a warning with
  the truncated daemon message; the original timeout/verdict is always
  preserved (cleanup never masks it).
- Uvicorn: keep the default access log (no body logging). Do NOT add
  `--access-log`-with-bodies proxies, request-dump middleware, or APM
  body capture in front of `/judge*`. This file is the configuration
  record: body logging is prohibited by policy, not by a code flag.

## 11. Limitations (must not be presented as passes)

- No Docker-capable Linux host was available in this environment; the
  Docker-mode matrix above is unrunnable here and must run before contest
  use. The rlimit backend has been exercised per language in the recorded
  environment; that does not establish safe isolation or verify the current
  deployed backend.
- rlimit mode provides no filesystem or network namespace: contestant code
  runs as the service UID, can access files available to that user and can
  reach the network. Do not use it for untrusted public submissions. The
  Docker backend is the intended isolated path, subject to its host and
  configuration requirements.
- rlimit memory accounting is `RLIMIT_AS` for C/C++/Python and a runtime flag
  for Java/Go/Node, so a container-level (cgroup) memory kill does not apply.
  Go/Node programs that exceed the limit are more likely to be reported as a
  runtime error than MLE.
- Timing is wall-clock (`elapsed_ms` vs `time_limit_ms`) with +2s daemon
  headroom: variance under load is expected; strict CPU accounting is
  future work.
- Memory accounting is container-level (`--memory`), not per-process RSS;
  JVM overhead is bounded by `-Xmx` + Metaspace/code-cache caps.
- `notes` table is untouched in this phase (no evidence it is safe to
  drop; cleanup flagged separately — see parent follow-ups).
- Advisory locks are per-submission (`koj-judge-async`, submission ID);
  there is no durable idempotency-key table (by design — no new schema).

## 12. Parent integration required (Next.js owner)

1. **Retry wiring:** `app/api/submissions/route.ts` currently sends
   `{submission_id}` with no `Idempotency-Key` and no retry. On 503
   (`Retry-After`) or infra-pending from `/judge-async`, resend the same
   `submission_id` with a stable header `Idempotency-Key:
   submission-<id>` under exponential backoff (e.g. 5s, 15s, 30s; max
   ~3 retries), then surface "judge busy — retrying" instead of
   `runtime_error`. Duplicate-after-completion returns 200
   `{"deduped":true}` — treat as success.
2. **Infra display:** `GET /api/submissions` and the verdict UI must
   expose `judge_infra_error` (pending + flag = "judge infra error —
   retrying", never "your code crashed"). Recovery
   (`app/api/submissions/recovery.ts`, `recover/route.ts`) must set the
   flag when marking stale rows and must re-dispatch `pending +
   judge_infra_error=true` rows (set `running` + call `/judge-async`)
   instead of terminally blaming them.
3. **Migration:** apply `0003_judge_hardening.sql` (`drizzle-kit migrate`)
   before deploying this judge; backfill is a plain `DEFAULT false`.
4. **Provisioning:** copy `api/seccomp-koj.json` to
   `/etc/koj/seccomp-koj.json` on the judge host and pre-pull/pin images
   per §5. Record digests in the contest runbook.
5. **`notes` cleanup:** separate decision — confirm no production rows
   depend on `notes` before any drop; not in this phase.
