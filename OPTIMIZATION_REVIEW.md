# QUI Codebase Optimization Review

## Scope reviewed
- `core/main.lua`
- `modules/trackers/spellscanner.lua`
- `modules/cooldowns/cdm_manager.lua`
- `modules/cooldowns/effects.lua`
- `modules/cooldowns/cdm_viewer.lua`

## Top optimization opportunities

### 1) Remove redundant string work in profile import path (quick win)
**Where:** `QUICore:ImportProfileFromString` in `core/main.lua`.

The import path strips `QUI1:` **twice**:
- `str = str:gsub("^QUI1:", "")`  
- `str = str:gsub("^QUI1:", "")`

This is minor on a single call, but it is unnecessary work in a hot user-flow path and can hide logic mistakes.

**Recommendation:** Keep one `QUI1:` strip and one compatibility strip (`CDM1:`), then decode.

---

### 2) Reduce per-call DB side effects in SpellScanner
**Where:** `GetDB()` in `modules/trackers/spellscanner.lua`.

`GetDB()` currently:
- Creates global scanner tables lazily.
- Re-syncs `SpellScanner.autoScan` from DB every call.

Because many helper functions call `GetDB()`, this turns lookups into side-effectful operations and adds repeated branching/assignment.

**Recommendation:**
- Split into:
  - `EnsureDB()` for one-time initialization and migration.
  - `GetDB()` for pure lookup.
- Load `autoScan` once at module init/login and when option changes.

**Expected benefit:** lower overhead and simpler reasoning about scanner behavior.

---

### 3) Avoid repeated full child-table allocations during cooldown layout
**Where:** `RemovePadding()` in `modules/cooldowns/cdm_manager.lua` and similar patterns in `modules/cooldowns/effects.lua`.

Current pattern repeatedly builds arrays like:
- `local children = {viewer:GetChildren()}`
- `local visibleChildren = { ... }`

for every scheduled layout pass.

**Recommendation:**
- Iterate with `select(i, viewer:GetChildren())` style loops or cache child references per viewer when possible.
- Reuse scratch tables (`wipe(tbl)`) instead of allocating fresh tables every pass.
- Skip re-sort when icon ordering/stride unchanged since last pass.

**Expected benefit:** lower GC churn and fewer frame-time spikes during frequent viewer relayouts.

---

### 4) Replace broad `pcall` usage in tight aura loops with explicit guards
**Where:** `ScanSpellFromBuffs()` in `modules/trackers/spellscanner.lua`.

The scanner uses `pcall` inside the 1..40 aura scan loop for arithmetic and comparisons. Protected calls are expensive when used repeatedly in combat-relevant paths.

**Recommendation:**
- Use existing helper functions in `core/utils.lua` (`SafeValue`, `SafeArithmetic`, `SafeCompare`) and early guard checks on `duration`/`expirationTime` types.
- Reserve `pcall` for truly exceptional boundaries, not normal control flow.

**Expected benefit:** faster aura scan passes and cleaner error handling semantics.

---

### 5) Deduplicate timer bursts and post-layout work in cooldown effects
**Where:** event handling in `modules/cooldowns/effects.lua`.

The file schedules multiple timers for overlapping work:
- `ApplyToAllViewers()` + `HideExistingBlizzardGlows()` at 0.5s
- extra glow cleanup at 1s
- additional calls on `PLAYER_ENTERING_WORLD` and `PLAYER_LOGIN`

**Recommendation:**
- Introduce a single debounced scheduler (shared bucket pattern like `cdm_manager.lua`) for effect refresh.
- Track per-viewer dirty flags and process only dirty viewers.

**Expected benefit:** fewer duplicate traversals of viewer icons and smoother startup/login behavior.

---

### 6) Hook minimization for per-icon frames
**Where:** `HideCooldownEffects()` in `modules/cooldowns/effects.lua` and icon setup flows in `modules/cooldowns/cdm_viewer.lua`.

`HideCooldownEffects()` loops over effect frame names and may attach OnShow-related hooks repeatedly across many icons and children.

**Recommendation:**
- Consolidate to one per-child hide hook where possible.
- Track a compact processed state and avoid repeated hooksecurefunc attachment checks in nested loops.

**Expected benefit:** reduced hook overhead and less long-session memory growth.

---

## Suggested implementation order
1. **Quick wins first:** remove duplicate `gsub`, split SpellScanner DB init/lookup.
2. **High impact runtime:** reduce allocations/re-sorts in cooldown viewer passes.
3. **Stability/perf cleanup:** replace hot-path `pcall` usage and unify cooldown effect scheduling.
4. **Long-tail hardening:** hook deduplication audit for all icon-related modules.

## Validation checklist for optimization PRs
- Add lightweight profiling counters around:
  - layout passes per minute,
  - average icon count per pass,
  - scanner aura-loop duration.
- Compare before/after in:
  - login sequence,
  - combat with frequent procs,
  - Mythic+ pull (high buff/debuff churn).
- Confirm no regressions in:
  - cooldown glow behavior,
  - custom tracker scanning accuracy,
  - profile import compatibility (`QUI1` + `CDM1`).
