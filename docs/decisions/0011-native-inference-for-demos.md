# 0011 — Native llama-server for demos; containers stay for hermetic dev/CI

Date: 2026-05-18
Status: accepted (supersedes 0001 only for the native deployment surface)

## Decision

For demo runs on a developer workstation, llama-server runs **natively on the
host** (`brew install llama.cpp` on macOS; whatever package is appropriate
on Linux). Two processes serve the small (Qwen 2.5 3B) and mid (Qwen 2.5
7B) tiers on `:11434` and `:11435` respectively, hitting Metal/CUDA directly.
The orchestrator in compose points at `host.docker.internal:11434/11435`
via env-substituted `SMALL_INFERENCE_URL` / `MID_INFERENCE_URL`, and the
in-cluster `small-inference` / `mid-inference` containers are simply not
started in this mode.

For hermetic dev, CI, and machines without GPU acceleration, the
containerized inference path from ADR 0001 stays. Both paths produce the
same audit events with the same orchestrator code — the only thing that
changes is the URL the orchestrator hits.

## Context

`smoke_full` on Docker-CPU takes ~8-9 minutes per happy run: small classify
~70-80 s, mid diagnosis ~440 s for ~73 output tokens. That cadence is not
demoable. The single biggest lever is inference throughput, and the single
biggest lever on inference throughput is hardware acceleration.

Docker Desktop on macOS does not pass Metal through to containers. The
realistic options were:

1. **Native llama-server on the host**, via `brew install llama.cpp`. Single
   binary per tier; Metal works out of the box on Apple Silicon. Pointed at
   from inside the compose network via `host.docker.internal`.
2. **Run on a Linux box with NVIDIA + the container toolkit.** Works, not
   applicable here.
3. **Ollama natively.** Already installed on this machine, also Metal-native.
   Cleaner UX (`ollama pull qwen2.5:3b`), but single port for all models
   would require the orchestrator's `InferenceClient` to pass the model
   name in chat requests, diverging from the container path. Considered
   and rejected for that reason.

Native llama-server matches the container architecture exactly: one process,
one model, one port. No `InferenceClient` change. The only delta is which
URL the orchestrator dials.

## Alternatives considered

- **Stay on container inference, accept the cadence.** Documented in the
  README is fine for "this exists," but not for "we should show this to a
  buyer." Demo polish requires acceptable wall-clock.
- **Ollama.** Already installed on this machine, would shave one install
  step. Rejected because its API requires model selection per call, which
  forces a fork in `InferenceClient` between native and containerized
  paths. The principle that "the only thing that changes between deployment
  modes is a URL" is worth preserving.
- **vLLM, TGI, or other server.** Heavier, more capability than this PoC
  needs, and breaks the one-binary story.
- **GPU passthrough via Docker Desktop.** Not supported on macOS.
- **A separate compose file for native mode (`docker-compose.native.yml`).**
  Considered, rejected in favor of env-substituted URLs + explicit service
  list on `docker compose up`. Less file proliferation, same effect.

## Consequences

Easier:
- A happy run drops from ~9 min to ~30-60 s on Apple Silicon Metal
  (expected; verify in step 7 of this checklist).
- The container architecture stays exactly as ADR 0001 specified for the
  hermetic case (CI, anyone without GPU).
- A new contributor on macOS can `brew install llama.cpp`, run two
  background processes, and demo the system at a watchable cadence.

Harder:
- One more thing to install. Mitigated by `scripts/start-native-inference.sh`
  which checks for `llama-server` on PATH and fails with a clear message
  pointing at the brew command.
- Model weights now live in TWO places (Docker volumes for the container
  path, `~/.castellan/models/` for the native path). The script can copy
  out of the Docker volumes on first run if they're populated, avoiding a
  second 6 GB download.
- `host.docker.internal` is Docker Desktop magic; Linux compose needs
  `--add-host=host.docker.internal:host-gateway` or similar. Documented.

To revisit:
- Linux dev path with NVIDIA — likely doable with the container toolkit;
  out of scope for this ADR.
- If/when Castellan grows beyond a single-orchestrator PoC, native inference
  becomes a deployment-doc question more than an ADR-worthy decision.

## The actual contract

The orchestrator's two env vars decide everything:

| Mode | `SMALL_INFERENCE_URL`                       | `MID_INFERENCE_URL`                         | Inference containers? |
|------|---------------------------------------------|---------------------------------------------|-----------------------|
| Containerized (default) | `http://small-inference:8080`     | `http://mid-inference:8080`                | up                    |
| Native (demo)           | `http://host.docker.internal:11434` | `http://host.docker.internal:11435`      | not started           |

`make demo-native` and `make smoke-native` set the env and start the native
processes via `scripts/start-native-inference.sh`. `make stop-native` tears
them down.
