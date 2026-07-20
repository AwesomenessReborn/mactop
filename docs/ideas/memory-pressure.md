# Idea: macOS Memory Pressure

Status: proposed (planning only — no implementation yet)
Upstream issue: filed on metaspartan/mactop (see repo tracker)
Base branch: `feature/memory-pressure` off `upstream/development` @ `9ce10f9`

## One-line

Surface the macOS kernel's **memory pressure state** (green / yellow / red — Normal /
Warning / Critical) as a first-class metric, distinct from "Memory Used %".

## Why this is a distinct signal

mactop already reports **Memory Used** matching Activity Monitor (`activityMonitorMemoryUsed`,
`native_stats.go:874`), swap, and DRAM bandwidth. High "used %" is normal and healthy on Apple
Silicon; it does not tell you the system is *under stress*. Memory **pressure** is the kernel's
own verdict on whether it is having to compress / swap / reclaim aggressively. For the MLX / LLM
workloads mactop is popular with, you can watch usage sit at 80%+ while pressure stays green, and
only flip yellow when compression/swap actually ramps.

## The product distinction we must keep (non-negotiable)

| Signal | Provenance | Claim we can make |
|---|---|---|
| **Pressure state** (Normal/Warning/Critical) | `kern.memorystatus_vm_pressure_level` sysctl — the kernel's own level | **Authoritative.** This *is* the green/yellow/red state. |
| **0–100 pressure "curve"** | Heuristic from VM counters we already read | **Approximate only.** Apple's Activity Monitor curve is private/smoothed/undocumented. Must be labeled "(approx.)" and must NOT claim parity. |

## Recommended shape

**First PR = state only.** Ship the authoritative state indicator. Defer the approximate 0–100
history graph to a *separate* follow-up PR with its own validation section. See
`docs/specs/memory-pressure.md` and `tasks/plan.md`.

## Hard constraints (from repo conventions)

- Native CGO only — read the sysctl in `native_stats.go`, mirroring the existing `vm.swapusage`
  block. **Do not** shell out to the `memory_pressure(1)` CLI.
- Poll per tick (the existing metrics loop). No `DISPATCH_SOURCE_TYPE_MEMORYPRESSURE` event source
  in v1.
- Pure, side-effect-free mapping helpers so they are table-test-able (mirror
  `activityMonitorMemoryUsed` + `TestActivityMonitorMemoryUsed`).
- `.cursorrules`: no code comments, table-driven tests, `make sexy` must pass (gofmt/vet/gocyclo<15/ineffassign).
