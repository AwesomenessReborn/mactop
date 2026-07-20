# TODO: macOS Memory Pressure — PR 1 (state-only)

Branch `feature/memory-pressure` @ base `9ce10f9`. Planning only — do not start until approved.
Each task is atomic, has acceptance criteria, and an exact verification command. Checkpoints run
`make build && make test && make sexy`.

Repo-wide gate (must pass after every task that touches Go):
```
cd $(git rev-parse --show-toplevel) && make build && make test && make sexy
```

---

## Phase 0 — Setup (done)
- [x] Branch `feature/memory-pressure` off `upstream/development` @ `9ce10f9`.
- [x] Planning docs written (`docs/ideas`, `docs/specs`, `tasks/plan.md`, this file).

## Phase 1 — Types + pure helper + test  (no CGO, no behavior change yet)
- [ ] **T1.1** Add `PressureLevel int` + `PressureState string` to `MemoryMetrics` (`types.go:82`)
      with json tags `pressure_level` / `pressure_state`.
      - Accept: builds; json tags match existing snake_case style.
      - Verify: `go build ./...`
- [ ] **T1.2** Add the same two fields to `NativeMemoryMetrics` (`native_stats.go:832`).
      - Accept: builds.
      - Verify: `go build ./...`
- [ ] **T1.3** Add pure helper `pressureStateFromLevel(level int) string` in `native_stats.go`
      (switch 1/2/4→Normal/Warning/Critical, default Unknown). No comments.
      - Accept: builds; helper is side-effect-free.
      - Verify: `go build ./...`
- [ ] **T1.4** Table-driven `TestPressureStateFromLevel` in `app_test.go` mirroring
      `TestActivityMonitorMemoryUsed` (cases 1,2,4,0,3,99), `t.Parallel()`.
      - Accept: test passes.
      - Verify: `go test -run TestPressureStateFromLevel ./internal/app/...`
- [ ] **CHECKPOINT 1**: `make build && make test && make sexy` all green.
      Commit: `feat(mem): add pressure fields + pressureStateFromLevel helper + test`.

## Phase 2 — Native collection
- [ ] **T2.1** In `GetNativeMemoryMetrics()` (`native_stats.go:886`), after the swap block, read
      `kern.memorystatus_vm_pressure_level` into a C `int`, mirroring the `vm.swapusage` pattern
      (`native_stats.go:918-931`). On sysctl failure: `PressureLevel=0`, `PressureState="Unknown"`,
      continue. On success: set level + `pressureStateFromLevel(level)`.
      - Accept: builds; both success and failure paths set the fields; memory read never fails
        because of pressure.
      - Verify: `go build ./...` then run `./mactop --help`-style smoke (build only; runtime needs a Mac).
- [ ] **T2.2** Bridge in `getMemoryMetrics()` (`metrics.go:695`): copy `PressureLevel` +
      `PressureState` from native into `MemoryMetrics`.
      - Accept: builds; fields propagate.
      - Verify: `go build ./...`
- [ ] **CHECKPOINT 2**: `make build && make test && make sexy`; manual: run mactop, confirm no
      regression and (if reachable) inspect headless JSON shows `pressure_state`.
      Commit: `feat(mem): read kern.memorystatus_vm_pressure_level natively`.

## Phase 3 — TUI
- [ ] **T3.1** Extend `memoryGauge` title/color by state (near `updateMemoryGaugeTitle`,
      `app.go:1910`): Normal→green, Warning→yellow, Critical→red, using existing theme colors.
      Route any new visible string via `i18n.T`.
      - Accept: builds; gauge shows state; color changes with state.
      - Verify: `go build ./...` + manual run + `sudo memory_pressure -S -l warn` to see it flip.
- [ ] **T3.2** Add i18n key(s) to `internal/i18n/locales/active.en.toml`.
      - Accept: i18n tests pass; string resolves.
      - Verify: `go test ./internal/i18n/...`
- [ ] **CHECKPOINT 3**: full gate + manual color check vs Activity Monitor.
      Commit: `feat(mem): surface pressure state in memory gauge`.

## Phase 4 — Headless + Prometheus
- [ ] **T4.1** Structured headless (JSON/YAML/XML/TOON) is automatic via `MemoryMetrics` embedding
      (`headless.go:100`) — verify fields appear.
      - Verify: run headless JSON path; grep for `pressure_state`.
- [ ] **T4.2** CSV: add `Mem_Pressure` to header (`headless.go:231`) and the value to the row
      (`headless.go:353`).
      - Accept: CSV column count matches header; value present.
      - Verify: run CSV headless path; confirm column aligns.
- [ ] **T4.3** Prometheus: add dedicated `mactop_memory_pressure_level` gauge in `globals.go`,
      register at `metrics.go:26`, publish near `metrics.go:379-382`. (Or maintainer-preferred
      label form — see plan Q2.)
      - Accept: `/metrics` exposes the new series; value is 0/1/2/4.
      - Verify: `go test ./internal/app/...` (Prometheus type test) + manual `curl :PORT/metrics`.
- [ ] **CHECKPOINT 4**: full gate + headless + `/metrics` spot check.
      Commit: `feat(mem): expose pressure in headless CSV + Prometheus`.

## Phase 5 — Validation & PR
- [ ] **T5.1** Manual matrix: `memory_pressure -S -l warn|critical`; real MLX/LLM load; RAM hog.
      Record state transitions vs Activity Monitor color.
- [ ] **T5.2** Confirm `Unknown` fallback path (simulate sysctl-miss if possible / code-review it).
- [ ] **T5.3** Update README memory section if the project documents metrics there (check first).
- [ ] **T5.4** Open PR against `upstream/development`, narrow scope, link the issue, state clearly:
      authoritative state only, approx curve deferred. Draft first.

## Deferred to PR 2 (not in this branch's first PR)
- [ ] `approxPressurePercent` helper + bounds/monotonicity tests.
- [ ] History chart / dedicated pressure widget or layout.
- [ ] Activity Monitor comparison writeup for the approximate curve.
- [ ] (Maybe never) Dispatch `DISPATCH_SOURCE_TYPE_MEMORYPRESSURE` event source.

## Global acceptance for PR 1
- `make build` + `make test` + `make sexy` green.
- Colored Normal/Warning/Critical indicator matches Activity Monitor state.
- `pressure_state`/`pressure_level` in headless JSON; level in Prometheus.
- Missing sysctl → `Unknown`, no crash, memory read intact.
