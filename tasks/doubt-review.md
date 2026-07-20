# Doubt review — adversarial pass on the memory-pressure plan

Purpose: attack this plan before building. Findings are ordered by how likely they are to bite.

## Unsupported / over-stated claims

1. **"1/2/4 bit-flag valued" (spec §2, §4c).** The *Dispatch* flags (NORMAL/WARN/CRITICAL) are
   combinable bit flags; the *sysctl* is only *observed* to return one of 1/2/4. Do not describe
   the sysctl as a bitmask or OR values. Safe statement: "observed values 1/2/4; treat anything
   else as Unknown." The helper's `default→Unknown` already enforces this — keep it.
2. **"Structured headless picks up the fields automatically" (spec §8, T4.1).** Only true per
   serializer. `MemoryMetrics` fields currently carry **only `json` tags** (types.go:82-88), while
   the outer `SnapshotOutput` carries json/yaml/xml/toon. So JSON is automatic; YAML/XML/TOON keys
   fall back to Go field names / json tag depending on the encoder. **Action:** before claiming
   parity, check how the *existing* memory fields render in each format and match that exactly —
   probably just add a `json` tag like its siblings, nothing more. Don't invent yaml/xml/toon tags
   the sibling fields don't have (that would be inconsistent).
3. **"OS X 10.9+" availability.** Sourced for the mechanism, but the exact sysctl-name floor isn't
   from Apple docs. It's a supporting note, not a guarantee — the `Unknown` fallback is what makes
   this safe, not the version claim. Don't put a hard minimum-OS assertion in user-facing text.

## Overengineering / scope creep

4. **PR 1 may be bigger than "state MVP."** It currently bundles TUI + CSV + Prometheus + i18n.
   The truest minimum is **TUI state + structured headless (json)**. CSV column, Prometheus series,
   and README edits could each be trimmed to shrink review surface. Recommend keeping them (low
   cost, consistency) but be ready to drop CSV/Prometheus if the maintainer wants the smallest
   possible first PR. Flagged, not blocking.
5. **T5.3 (README update)** is mild scope creep — gated on "check first," acceptable.

## Hidden assumptions

6. **Recoloring `memoryGauge` by pressure (T3.1) is risky.** The gauge's bar color likely already
   encodes fill/usage and is re-set every tick. Overloading `BarColor` for pressure could be
   overwritten, or visually conflate "high usage" with "high pressure" — the exact distinction the
   feature exists to make. **Safer design:** put the state in the gauge **title/label text**
   (colored inline if the widget supports it), and only reconsider bar color if a maintainer asks.
   Update spec §7 / plan §12 accordingly before building.
7. **Assumes the sysctl exists on every supported chip.** Not verified across M1–M5. The `Unknown`
   fallback covers it, but T5.2 must actually exercise/inspect that path, not assume it.
8. **Assumes one current level per read.** If a future OS returned something unexpected, `default→
   Unknown` degrades gracefully — good — but don't log-spam per tick if it's persistently unknown.

## Tasks that are too large / need splitting

9. **T4.3 (Prometheus)** hides a design decision (dedicated gauge vs label) *and* register+publish
   wiring. Resolve the decision (plan Q2) **before** starting T4.3, or it stalls mid-task.
10. Everything else is appropriately atomic.

## Net verdict

Plan is sound and honest about the authoritative/approximate split. Three concrete edits to make
**before** `/build`:
- (a) soften the "bit-flag"/"OS 10.9+" phrasing (findings 1, 3),
- (b) verify per-format headless serialization instead of asserting "automatic" (finding 2),
- (c) change TUI integration from bar-recolor to title/label text (finding 6).

None of these change scope or the recommended PR-split; they tighten correctness and reduce review
risk. No task is oversized except the latent decision inside T4.3 (finding 9).
