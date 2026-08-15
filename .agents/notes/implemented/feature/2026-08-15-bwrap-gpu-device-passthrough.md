# Agent Note: Configurable GPU device passthrough in bwrap profiles

Status: implemented

English | [中文](2026-08-15-bwrap-gpu-device-passthrough.zh.md)

## Problem

The bwrap profile builds a fresh `/dev` through `--dev /dev`, so a confined command sees none of the host's device nodes. On WSL2 the GPU is the paravirtualized dxgkrnl device `/dev/dxg` — CUDA/NVML reach the GPU through it and there are no `/dev/nvidia*` nodes — so GPU-backed ML work (`torch.cuda.is_available()`, `nvidia-smi`) fails under confinement, and `dsh-sandbox-local` had no knob to carry device nodes into the sandbox short of replacing the runner wholesale via `runnerCommand`.

## Decision

`dsh-sandbox-local` gains the `gpuDeviceNodes: string[]` config key (default empty). Every bwrap profile the provider builds — the platform bwrap rung and the `runnerCommand` override path alike — bind-mounts each listed node with `--dev-bind <node> <node>` into the fresh `/dev`, but only when the node exists on the host. Entries are validated at plugin load: non-empty absolute paths, duplicates collapsed, any violation fails loud. The existence guard lets one list serve heterogeneous hosts (bare-metal `/dev/nvidia*` vs WSL2 `/dev/dxg`) and leaves non-GPU hosts untouched.

The default is empty on purpose: device exposure is a dimension outside the file-effect mode vocabulary, so the default composition's sandbox surface does not change; a deployment that wants confined commands to reach a GPU opts in explicitly. Landlock and Seatbelt rungs are unaffected — the former has no mount concept, the latter no matching mechanism.

## Testing

`local.spec.ts` pins the behavior deterministically on every host by passing real temp-file paths as fake nodes (the profile builder only existence-checks): no binds by default, binds only existing nodes, the `runnerCommand` and platform-rung paths both receive the configured list, duplicates collapse, and blank or relative entries fail plugin load.

## Alternatives considered

- **Hardcoded `['/dev/dxg']` list** — rejected: a deployment-varying choice as a constant violates the no-hardcoded-tunables convention, and bare-metal NVIDIA or AMD ROCm hosts need different node sets.
- **`/dev/dxg` bound by default** — rejected: silently widens every default composition's sandbox with GPU access, a resource the mode vocabulary never promised.
- **A separate GPU-passthrough plugin** — rejected: it would require inventing a profile extension point on the provider with no consumer to justify it; the behavior belongs to the owning provider.
- **Binding all of `/dev`** — rejected: far wider than one device; the point of `--dev /dev` is a blank device namespace.

## Consequences

- A WSL2 deployment enables GPU in confined commands with one `cordis.yml` line (`gpuDeviceNodes: ['/dev/dxg']`); both sandbox modes expose the node once configured, and the README documents this.
- Unconfigured deployments and non-GPU hosts see zero change beyond one `existsSync` stat per listed node per wrap.
- Misconfiguration fails loud at load, while the existence guard keeps cross-host configs from failing.
- The sandbox's file-effect promise is unchanged; device access is an explicitly chosen, separate deployment dimension.
