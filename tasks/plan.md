# Plan: macOS Memory Pressure

Branch: `feature/memory-pressure` (base `upstream/development` @ `9ce10f9`)
Companion docs: `docs/ideas/memory-pressure.md`, `docs/specs/memory-pressure.md`
Status: **planning only.** Do not run `/build` from this doc.

---

## 1. Problem statement & goals

Add the macOS kernel **memory-pressure state** (Normal/Warning/Critical = green/yellow/red) as a
metric distinct from "Memory Used %". Goal for the first PR: authoritative state, collected
natively, shown in TUI + headless + Prometheus, with tested pure helpers. Keep it small and
honest.

## 2. Authoritative vs approximate behavior

- **Authoritative:** `kern.memorystatus_vm_pressure_level` → 1 Normal / 2 Warning / 4 Critical.
  This is the green/yellow/red state; we can claim it is accurate.
- **Approximate (deferred):** any 0–100 curve derived from VM counters. Heuristic, labeled
  "(approx.)", no parity claim. Not in PR 1.

## 3. Assumptions & verified facts

Verified:
- Level values 1/2/4, OS X 10.9+ — XNU `kern_memorystatus.c`, newosxbook.com, Chromium.
- Dispatch source (`DISPATCH_SOURCE_TYPE_MEMORYPRESSURE`) is the Apple-documented event path;
  polling the sysctl is the simpler equivalent and fits mactop's per-tick loop.
- Repo already reads memory via `sysctlbyname` in `native_stats.go` (`vm.swapusage` block) — the
  new read mirrors it exactly.

Assumptions (to validate during build, flagged as risks):
- The sysctl returns a single current level (not a bitmask OR of multiple) — treat unknown values
  as `Unknown`.
- All currently-supported Apple Silicon chips return the sysctl; guard with `Unknown` if not.

## 4. Recommended architecture

Data flows through the **existing** memory pipeline — no new subsystem:

```
kern.memorystatus_vm_pressure_level (sysctl)
  └─ GetNativeMemoryMetrics()  native_stats.go:886   (read + map via pressureStateFromLevel)
       └─ NativeMemoryMetrics{PressureLevel,PressureState}   native_stats.go:832
            └─ getMemoryMetrics()  metrics.go:695            (copy through)
                 └─ MemoryMetrics{PressureLevel,PressureState}  types.go:82
                      ├─ TUI: memoryGauge title/color   app.go:1910 / app.go:72
                      ├─ Headless: snapshot.Memory       headless.go:100 (+CSV 231/353)
                      └─ Prometheus: mactop_memory_pressure_level  metrics.go:379
```

Pure helper `pressureStateFromLevel(int) string` mirrors `activityMonitorMemoryUsed` and is the
only unit-tested logic.

## 5. Alternative designs considered

| # | Design | Pros | Cons | Verdict |
|---|---|---|---|---|
| A | **State-only MVP** | Authoritative, tiny, low review risk, honest | No graph "wow" | Ship as **PR 1** |
| B | State + approximate gauge in one PR | One-shot feature | Larger diff, unverified heuristic mixed with authoritative signal, higher review risk | Reject for PR 1 |
| C | **State-first PR, then separate approx/validation PR** | Clean review, authoritative ships fast, approx gets its own validation | Two PRs | **Recommended overall** |
| D | Dispatch event source instead of polling | Event-driven, Apple-blessed | New concurrency surface, callback lifecycle, doesn't match per-tick model | Defer / maybe never |
| E | Shell out to `memory_pressure(1)` | Trivial | Violates native-only convention, subprocess cost | Reject |

**Recommended: C** — PR 1 = design A (state only); approximate curve is a later PR.

## 6. Explicit MVP scope (PR 1)

- Two fields on `NativeMemoryMetrics` + `MemoryMetrics`.
- Native sysctl read + `pressureStateFromLevel` helper.
- Bridge through `getMemoryMetrics`.
- Colored state on `memoryGauge` (title text + color).
- Headless (automatic for structured; CSV column added) + one Prometheus series.
- Table-driven unit test for the helper.

## 7. Explicit non-goals (PR 1)

Approx 0–100 curve, dedicated history chart, new layout, Dispatch source, per-process attribution,
non-macOS support beyond `Unknown`.

## 8. Proposed data types & interfaces

See spec §4. Summary:
- `NativeMemoryMetrics`: `PressureLevel int`, `PressureState string`.
- `MemoryMetrics`: `PressureLevel int json:"pressure_level"`, `PressureState string json:"pressure_state"`.
- `func pressureStateFromLevel(level int) string`.

## 9. Collector / native implementation

Add after the swap block in `GetNativeMemoryMetrics()`, mirroring `native_stats.go:918-931`:
read `kern.memorystatus_vm_pressure_level` into a C `int`; on failure set level 0 / `Unknown`;
map non-failure via helper. No comments (per `.cursorrules`).

## 10. Mapping helper design

`pressureStateFromLevel`: pure switch — `1→"Normal"`, `2→"Warning"`, `4→"Critical"`, default
`"Unknown"`. String constants (or small package-level consts) for the four states to avoid typos.

## 11. Polling / update flow

Reuse existing tick. One extra `sysctlbyname` per interval. No new goroutine, timer, or channel.

## 12. UI integration

`memoryGauge` only. Extend title (like `updateMemoryGaugeTitle`, `app.go:1910`) with the state and
set gauge color by state using existing theme colors. Route visible text through `i18n.T`.

## 13. Unavailable / error behavior

Missing sysctl → `Unknown`/0, no error to user, memory read still succeeds. Never panic.

## 14. Platform / build-tag behavior

Lives in already-macOS/CGO-gated `native_stats.go`; inherits gating. No new tags.

## 15. Unit-test strategy

Table-driven `TestPressureStateFromLevel` in `app_test.go`, `t.Parallel()`, cases {1,2,4,0,3,99}.
Mirrors `TestActivityMonitorMemoryUsed` (`app_test.go:384`). Pure Go, no CGO in test.

## 16. Integration / manual validation

`sudo memory_pressure -S -l warn|critical` to force transitions; compare against Activity
Monitor's Memory tab under real MLX load + a RAM hog; confirm `Unknown` degradation.

## 17. Activity Monitor comparison methodology

Compare **state color only** (green/yellow/red), not curve height — that is all we claim. Log
`vm_stat`, swap, and the sysctl level over a ramping workload and confirm our state transitions
line up with Activity Monitor's color transitions.

## 18. Exact likely files to modify (PR 1)

- `internal/app/native_stats.go` — struct fields, sysctl read, helper.
- `internal/app/types.go` — `MemoryMetrics` fields.
- `internal/app/metrics.go` — `getMemoryMetrics` bridge + Prometheus publish/register.
- `internal/app/globals.go` — Prometheus gauge decl (if dedicated).
- `internal/app/app.go` — gauge title/color update.
- `internal/app/headless.go` — CSV header + row.
- `internal/i18n/locales/active.en.toml` — new visible string key(s).
- `internal/app/app_test.go` — helper test.

## 19. Dependency ordering

types/struct fields → helper (+test) → native read → bridge → Prometheus → TUI → headless CSV →
i18n. Everything after "helper" depends on the fields existing.

## 20. Implementation tasks

See `tasks/todo.md` for the atomic checklist with per-task verification commands and checkpoints.

## 21. Acceptance criteria

Per task in `tasks/todo.md`; overall in spec §13.

## 22. Atomic commit plan

1. `feat(mem): add pressure fields + pressureStateFromLevel helper + test`
2. `feat(mem): read kern.memorystatus_vm_pressure_level natively`
3. `feat(mem): surface pressure state in memory gauge`
4. `feat(mem): expose pressure in headless CSV + Prometheus`

(Follow repo commit style — inspect `git log` before finalizing prefixes; no Co-Authored-By, user
is sole author.)

## 23. Risks & mitigations

| Risk | Mitigation |
|---|---|
| sysctl name unavailable on some chip/OS | `Unknown` fallback, never fail memory read |
| sysctl returns unexpected value | default→`Unknown` in helper |
| Maintainer wants exact AM curve | scope PR 1 to state; approx is separate PR with validation |
| gocyclo>15 from added branches | keep helper a flat switch; don't grow `GetNativeMemoryMetrics` cyclomatic count (extract if needed) |
| Prometheus label vs dedicated gauge choice | ask maintainer; default to dedicated `mactop_memory_pressure_level` |
| i18n missing keys break other locales | add `en` key; confirm fallback behavior with existing i18n tests |

## 24. Open questions for maintainer

1. Presentation: reuse `memoryGauge` (title+color) or a dedicated pressure widget/layout?
2. Prometheus: `type="pressure_level"` on `mactop_memory_gb`, or a dedicated
   `mactop_memory_pressure_level` gauge?
3. Should the approximate 0–100 graph ship at all, or is authoritative state enough?
4. State-only PR 1 acceptable, with approx as a follow-up? (This plan assumes yes.)

## 25. PR split strategy

- **PR 1:** state-only (this plan). Small, authoritative, testable.
- **PR 2 (later):** approximate 0–100 curve + history chart + Activity Monitor comparison writeup,
  clearly labeled approximate. Only if the maintainer wants it.
