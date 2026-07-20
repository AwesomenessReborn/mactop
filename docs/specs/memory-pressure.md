# Feature Spec: macOS Memory Pressure

Base: `feature/memory-pressure` off `upstream/development` @ `9ce10f9`
Scope of this doc: technical contract for the **state-only MVP** (PR 1), with the approximate
graph specified separately as **deferred (PR 2)**.

## 1. Problem statement & goals

Expose the macOS kernel memory-pressure *state* as a metric alongside the existing memory
metrics, so users can distinguish "lots of RAM in use" (normal) from "the system is under memory
stress" (the green/yellow/red signal in Activity Monitor).

Goals (PR 1):
- Collect the kernel pressure level natively (no subprocess).
- Map it to a stable enum: `Normal` / `Warning` / `Critical` / `Unknown`.
- Show it as a colored indicator in at least one TUI layout.
- Expose it in headless (JSON/YAML/etc.) and Prometheus output, consistent with existing memory
  metrics.
- Unit-test the pure mapping helpers.

Non-goals (PR 1): the 0–100 approximate curve, a dedicated history chart, a Dispatch event
source, per-process pressure attribution, non-macOS behavior beyond a clean no-op/Unknown.

## 2. Authoritative vs approximate (the core distinction)

- **Authoritative:** `kern.memorystatus_vm_pressure_level`. Integer; **observed** values `1` =
  Normal, `2` = Warning, `4` = Critical (do not treat as a maskable bitfield — the sysctl returns
  the current level, and anything outside {1,2,4} maps to Unknown). This is the kernel's own
  pressure level and *is* the green/yellow/red state. Reported available OS X 10.9+ (supporting
  note, not a guarantee — the `Unknown` fallback is what makes it safe).
- **Approximate (DEFERRED):** a 0–100 number derived from VM counters (compressor pages, swap-in
  rate, available vs total). This is a heuristic. Apple's Activity Monitor plotted curve is
  undocumented and appears time-smoothed; we will **not** claim parity. If/when shipped it is
  labeled `Memory Pressure (approx.)`.

## 3. Verified facts (see docs/ideas + plan for sources)

- Values 1/2/4 confirmed by XNU `kern_memorystatus.c`, newosxbook.com, and Chromium's
  `memory_pressure_monitor_mac.cc` (which reads the same sysctl).
- The sysctl name itself is not in Apple's public API docs; the *Dispatch* equivalent
  (`DISPATCH_SOURCE_TYPE_MEMORYPRESSURE`, flags NORMAL/WARN/CRITICAL) is Apple-documented and is
  the officially-blessed event-driven path — noted as a possible future enhancement, not used now.
- Chromium maps NORMAL→none, WARN→moderate, CRITICAL→critical. We use Normal/Warning/Critical.

## 4. Data types & interfaces (anchored to real code)

### 4a. Native struct — `native_stats.go:832` `NativeMemoryMetrics`
Add:
```go
PressureLevel int    // raw sysctl value: 1/2/4, 0 if unavailable
PressureState string // "Normal"|"Warning"|"Critical"|"Unknown"
```

### 4b. Public metric — `types.go:82` `MemoryMetrics`
Add (with json tags matching existing style):
```go
PressureLevel int    `json:"pressure_level"`
PressureState string `json:"pressure_state"`
```

### 4c. Pure helpers (new, in `native_stats.go`, mirror `activityMonitorMemoryUsed`)
```go
func pressureStateFromLevel(level int) string // 1->Normal 2->Warning 4->Critical else Unknown
```
Deferred (PR 2): `func approxPressurePercent(...) float64`.

## 5. Collector / native implementation

In `GetNativeMemoryMetrics()` (`native_stats.go:886`), after the swap block, read the sysctl,
mirroring the `vm.swapusage` pattern at `native_stats.go:918-931`:
```c
// int level; size = sizeof(int);
// sysctlbyname("kern.memorystatus_vm_pressure_level", &level, &size, NULL, 0)
```
- On non-zero return (unsupported / older OS): `PressureLevel = 0`, `PressureState = "Unknown"`,
  and continue — never fail the whole memory read for a missing pressure value.
- Map via `pressureStateFromLevel`.

Bridge in `getMemoryMetrics()` (`metrics.go:695`): copy the two new fields through, same as the
existing `Total/Used/...` copy.

## 6. Polling / update flow

No new loop. The existing metrics tick calls `getMemoryMetrics()`; the pressure fields ride along.
Cost is one extra `sysctlbyname` per tick — negligible, same class as the swap read already done.

## 7. UI integration (PR 1)

Minimal and additive:
- Reuse the existing `memoryGauge` (`app.go:72-78`) area — append the pressure **state** to the
  gauge title/label (mirroring `updateMemoryGaugeTitle`, `app.go:1910`), colored inline by state
  (Normal→green, Warning→yellow, Critical→red) using existing theme color constants.
- **Do not recolor the gauge bar** (`BarColor`) for pressure: the bar already encodes fill/usage
  and is re-set each tick, so overloading it would be overwritten and would conflate "high usage"
  with "high pressure" — the exact distinction this feature exists to make. State goes in the
  title/label text. (See `tasks/doubt-review.md` finding 6.)
- Do NOT add a new layout in PR 1. A dedicated pressure widget / new layout entry is optional and
  can be its own change.
- i18n: any new visible string goes through `i18n.T(...)` and gets a key in
  `internal/i18n/locales/active.en.toml` (other locales can fall back).

## 8. Headless / Prometheus (PR 1)

- Headless: `MemoryMetrics` is embedded in the snapshot (`headless.go:100`). JSON is automatic via
  the struct's `json` tags. **Before claiming YAML/XML/TOON parity, verify how the existing memory
  fields render in each format and match that exactly** — the sibling fields carry only `json`
  tags, so add only a `json` tag to the new fields (do not invent yaml/xml/toon tags the siblings
  lack; that would be inconsistent). See `tasks/doubt-review.md` finding 2. For the CSV path, add a
  `Mem_Pressure` column to the header (`headless.go:231`) and the value row (`headless.go:353`).
- Prometheus: extend the existing `memoryUsage` GaugeVec (`globals.go:254`, `mactop_memory_gb`) with
  a `type="pressure_level"` series **OR** add a dedicated gauge `mactop_memory_pressure_level`
  (0/1/2/4). Publish alongside `metrics.go:379-382`. Decide in plan; a dedicated gauge is cleaner
  since the value is a level, not GB. (Open question for maintainer.)

## 9. Unavailable / error behavior

- sysctl missing/unsupported → `Unknown` state, level 0, no error surfaced to the user beyond the
  indicator reading "Unknown".
- Never panic; never abort the memory collector.

## 10. Platform / build tags

The whole `native_stats.go` is already CGO/macOS-gated by the existing build setup; the new code
lives in the same file and inherits it. No new build tags. Confirm `go vet`/build still pass.

## 11. Test strategy

- `pressureStateFromLevel`: table-driven test in `app_test.go` mirroring
  `TestActivityMonitorMemoryUsed` (`app_test.go:384`): cases 1→Normal, 2→Warning, 4→Critical,
  0→Unknown, 3→Unknown, 99→Unknown. `t.Parallel()`.
- No CGO in the unit test — the helper is pure Go.
- Deferred (PR 2): tests for `approxPressurePercent` bounds/monotonicity.

## 12. Manual / integration validation

- `sudo memory_pressure -S -l warn` and `-l critical` to force transitions; confirm the indicator
  flips Normal→Warning→Critical and back. (Used only for manual QA, never called by mactop.)
- Cross-check the color against Activity Monitor's Memory tab under a real MLX/LLM load and a
  gradual RAM hog.
- Verify graceful `Unknown` on any chip/OS that doesn't return the sysctl.

## 13. Acceptance criteria (PR 1)

- `make build` succeeds; `make test` green including the new helper test; `make sexy` clean.
- `mactop` shows a colored Normal/Warning/Critical indicator that matches Activity Monitor's state.
- Headless JSON contains `pressure_state`/`pressure_level`; Prometheus exposes the level.
- Missing sysctl degrades to `Unknown` with no crash and no failed memory read.
