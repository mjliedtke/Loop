# Loop and Learn — Customization Options Research

> Research note prepared by intaking this repository and cross-referencing the
> customizations published by **Loop and Learn** (loopandlearn.org). The goal is
> to document _what_ customizations exist, _how_ they are applied, and _where_
> they touch this codebase.
>
> ⚠️ This repository is **highly experimental and not approved for therapy** (see
> `README.md`). The customizations below modify safety-relevant behavior (carb
> limits, bolus authentication, insulin damping). Treat everything here as
> research, not medical guidance.

---

## 1. Repository intake

This repo is **Loop for iOS** — the LoopKit automated insulin delivery (AID)
app template.

| Fact | Value |
|------|-------|
| App | Loop (iOS) automated insulin delivery |
| Marketing version | **3.3.0** (`Version.xcconfig`, `LOOP_MARKETING_VERSION = 3.3.0`) |
| Build | `CURRENT_PROJECT_VERSION = 57` |
| Min iOS | 15.1 (`Loop.xcconfig`) |
| Built on | [LoopKit](https://github.com/LoopKit/LoopKit) frameworks (Git submodules) |
| Default branch base | LoopKit `master` lineage; latest commit adds NFC support for Libre scanning |

> **Version caveat:** Loop and Learn's customization catalog tracks the *current*
> Loop release (3.6 → 3.14+). This checkout is **Loop 3.3.0**, so a few catalog
> items are either version-gated the other way (e.g. `live_activity`,
> `dexcom_upload_readings`, `xdrip4ios` are flagged invalid/automated for
> 3.10+/3.14+) or land on code that has since moved. Anchors below were verified
> against the files in *this* checkout.

### 1.1 Built-in customization hooks (no third-party scripts needed)

Loop already ships two first-class extension points that custom builds use:

1. **xcconfig overrides** — `Loop.xcconfig` ends with:
   ```
   #include? "LoopOverride.xcconfig"
   #include? "../LoopConfigOverride.xcconfig"
   ```
   The `#include?` (note the `?`) means "include if present." Dropping a
   `LoopConfigOverride.xcconfig` next to the repo lets a build override the
   bundle id, display name, icon, team id, version, and — importantly —
   `SWIFT_ACTIVE_COMPILATION_CONDITIONS` **without editing tracked files**.
   Neither override file exists in this checkout (by design — they are local).

2. **Compile-time feature flags** — `Common/FeatureFlags.swift` reads
   `SWIFT_ACTIVE_COMPILATION_CONDITIONS` to toggle behavior at build time. Many
   flags are *inverse* (feature on by default, a `*_DISABLED` symbol turns it
   off). Flags exposed today include:

   | Flag symbol | Effect |
   |-------------|--------|
   | `AUTOMATIC_BOLUS_DISABLED` | disable automatic bolus dosing strategy |
   | `FEATURE_OVERRIDES_DISABLED` | disable sensitivity overrides |
   | `FIASP_INSULIN_MODEL_DISABLED` / `LYUMJEV_..._DISABLED` / `AFREZZA_..._DISABLED` | hide insulin models |
   | `MANUAL_DOSE_ENTRY_DISABLED` | disable manual/untracked dose entry |
   | `NONLINEAR_CARB_MODEL_DISABLED` | revert to linear carb model |
   | `OBSERVE_HEALTH_KIT_*_DISABLED` | stop importing HealthKit carbs/doses/glucose |
   | `REMOTE_COMMANDS_DISABLED` / `REMOTE_OVERRIDES_DISABLED` | disable remote control |
   | `SIRI_DISABLED`, `SIMPLE_BOLUS_CALCULATOR_ENABLED`, `MOCK_THERAPY_SETTINGS_ENABLED`, … | misc |
   | `DEBUG_FEATURES_ENABLED[_CONDITIONALLY]`, `SIMULATORS_ENABLED[_CONDITIONALLY]` | dev gates |

   These are the *officially sanctioned* toggles. Loop and Learn's catalog goes
   **beyond** these by patching source directly.

---

## 2. How Loop and Learn customizations are applied

Loop and Learn (LnL) distributes its customizations through the
**[`loopandlearn/lnl-scripts`](https://github.com/loopandlearn/lnl-scripts)**
repo (successor to `loopnlearn/loopbuildscripts`).

- The entry point is the **Build Select Script** (`BuildSelectScript.sh`), a
  menu with: *Build Loop · Build Related Apps · Maintenance Utilities ·
  **Customization Select** · Exit*.
- **Customization Select** (`CustomizationSelect.sh`) lets a user add or remove
  individual customizations against a downloaded Loop source tree.
- Mechanically, customizations are **source patches** applied to specific Swift
  files (the patch payloads live under `patch/`, `patch_cto/`, `patch_profile/`
  in the scripts repo). Scripts are assembled from `src/` via an `#!inline`
  build step.
- Two delivery paths exist for end users:
  - **Mac-Xcode Build** — run the script locally before building in Xcode.
  - **[Browser Build](https://www.loopandlearn.org/custom-code-browser-build/)**
    — apply customizations through the GitHub-Actions-based browser build flow.

Because these are textual patches against known files, each customization maps
to a concrete edit. Section 3 ties each one back to *this* repo.

---

## 3. The customization catalog (mapped to this codebase)

Source of truth: <https://www.loopandlearn.org/custom-code/>. Anchors verified
against files in this checkout.

### Standard customizations

| # | Identifier | What it does | Where it lands in this repo |
|---|-----------|--------------|------------------------------|
| 1 | `dexcom_upload_readings` | Default-enables uploading Dexcom readings to remote services (Nightscout). **Invalid for v3.10.0+.** | CGM/Dexcom manager defaults (LoopKit `CGMBLEKit`/`G7SensorKit` submodule, not in this repo) |
| 2 | `future_carbs_4h` | Allows entering carbs up to **4 h** in the future (removes the 1 h cap). | `Loop/Models/LoopConstants.swift:28` — `maxCarbEntryFutureTime = TimeInterval(hours: 1)` |
| 3 | `low_carb_limit` | Warn at **49 g**, hard cap **99 g** single entry. *(pick one carb-limit option)* | `Loop/Models/LoopConstants.swift:18,20` — `maxCarbEntryQuantity = 250`, `warningCarbEntryQuantity = 99` |
| 4 | `high_carb_limit` | Warn at **201 g**, hard cap **300 g**. *(pick one carb-limit option)* | same as #3 — `maxCarbEntryQuantity` / `warningCarbEntryQuantity` |
| 5 | `no_auth` | Removes auth (Face/Touch ID / passcode) prompt before bolus & therapy changes. **Disables a safety feature.** | Bolus/auth path: `Loop/View Models/BolusEntryViewModel.swift`, `ManualEntryDoseViewModel.swift` (authentication gate) |
| 6 | `override_sens` | Override insulin-needs picker range → **50–200 % in 5 % steps** (helps younger/very-sensitive users). | Override editor picker (LoopKit `LoopKitUI` `OverrideSelectionView` / scale-factor picker submodule) |
| 7 | `now_line` | Draws a **vertical "now" line** on the status/predicted-glucose charts. | Chart rendering (LoopUI `ChartsManager` / `Loop Status Extension/StatusChartsManager.swift`) |
| 8 | `lnl_icon` | Swaps the app icon for the Loop-and-Learn branded icon. | `Loop/DerivedAssetsBase.xcassets/AppIcon.appiconset/` + `APPICON_NAME` in `Loop.xcconfig` |
| 9 | `watch_title` | Removes the "Loop" title row from the Apple Watch display. | `WatchApp Extension/` status interface |
| 10 | `2hlollipop` | Sets the 🍭 *fast* ("lollipop") absorption default to **2 h** (vs 30 min). | `LoopCore/LoopCoreConstants.swift:19` — `defaultCarbAbsorptionTimes = (fast: .minutes(30), …)` |
| 11 | `meal_days` | Carb-history page shows **2 days**. *(mutually exclusive with `meal_week`)* | `Loop/View Controllers/CarbAbsorptionViewController.swift` (chart window / `totalHours`, ~L129–142) |
| 12 | `meal_week` | Carb-history page shows **1 week** (slower to load). *(mutually exclusive with `meal_days`)* | same as #11 |

### Advanced customizations

| # | Identifier | What it does | Notes |
|---|-----------|--------------|-------|
| 13 | `profiles` | Save/load full **therapy-setting profiles**; backward-compatible with the old `2002` identifier. | Patches therapy-settings + a new profiles UI (`patch_profile/`). Large change set. |
| 14 | `basal_lock` | Adds a **basal-rate lock** with adjustable guardrails, exposed as a Preferences row in Settings. | Settings + dosing guardrail patch. |
| 15 | `live_activity` | Adds **Live Activity / Dynamic Island** glucose display. **Invalid for v3.10.0+** (folded into mainline). | Newer Loop ships this natively. |
| 16 | `negative_insulin` | "Negative-insulin damper" — alters how strongly negative IOB suppresses dosing. | Dosing-math patch (algorithm-level; LoopKit). |
| 17 | `remote_window` | Widens the **remote-command validity window to 10 min**. | Remote command staleness check (LoopKit `RemoteCommands` / Nightscout service). This repo wires remote commands via `Loop/Models/Remote/*Action.swift` + `RemoteDataServicesManager.swift`; the validity window itself is in the command layer. |
| 18 | `xdrip4ios` | Adds **xDrip4iOS** as a CGM client (clones `xdrip-client-swift`). | Works v3.14.2+; **now automated** for newer Loop. |

### "Recommended for new Loopers"

LnL maintains a short, curated subset it suggests new users start with (the
page links it but does not enumerate it inline). In practice this is the
low-risk cosmetic/quality-of-life set — typically `now_line`, `lnl_icon`,
`watch_title`, the carb-history extensions, and one carb-limit option — while
explicitly steering newcomers **away** from `no_auth` and algorithm-level
changes (`negative_insulin`, `basal_lock`).

---

## 4. Risk tiering (research opinion)

| Tier | Customizations | Why |
|------|----------------|-----|
| 🟢 Cosmetic / display only | `now_line`, `lnl_icon`, `watch_title`, `meal_days`, `meal_week` | No effect on dosing. |
| 🟡 Input-bounds / UX | `future_carbs_4h`, `low_carb_limit`, `high_carb_limit`, `override_sens`, `2hlollipop`, `remote_window` | Change limits/defaults the user interacts with; affects dosing *indirectly* via what can be entered. |
| 🔴 Safety / algorithm | `no_auth`, `negative_insulin`, `basal_lock`, `profiles` | Remove guardrails or change dosing math. Highest scrutiny. |
| ⚙️ Integration / version-gated | `dexcom_upload_readings`, `live_activity`, `xdrip4ios` | Version-dependent; some now native or automated. |

---

## 5. If we wanted to adopt any of these here

Two clean integration strategies, in order of preference:

1. **Reuse the built-in hooks.** For anything mappable to a feature flag, add a
   `SWIFT_ACTIVE_COMPILATION_CONDITIONS` symbol + a branch in
   `Common/FeatureFlags.swift`, and let users opt in via a local
   `LoopConfigOverride.xcconfig`. This keeps tracked source pristine and matches
   Loop's own pattern. Good fit for on/off behaviors (e.g. gating `no_auth`,
   `now_line`).

2. **Constant edits** for the bounds-style ones (`future_carbs_4h`,
   `*_carb_limit`, `2hlollipop`) are one-liners in `LoopConstants.swift` /
   `LoopCoreConstants.swift`. Lowest effort, but they edit tracked files — so
   prefer wrapping them behind a feature flag if upstreaming.

3. **Don't hand-port the algorithm/profile patches** (`negative_insulin`,
   `profiles`, `basal_lock`) from scratch — they live partly in the LoopKit
   submodule and are maintained as cohesive patch sets by LnL. If needed, pull
   them via the LnL scripts against a matching Loop version rather than
   re-deriving.

> Note: items marked "(LoopKit submodule)" above are **not** editable from this
> repo alone — they require changes in the pinned LoopKit/LoopKitUI/CGM
> dependencies.

---

## 6. Sources

- [Code Customization — Available Customizations (Loop and Learn)](https://www.loopandlearn.org/custom-code/)
- [Code Customization — Browser Build](https://www.loopandlearn.org/custom-code-browser-build/)
- [Build Select Script](https://www.loopandlearn.org/build-select/)
- [loopandlearn/lnl-scripts (GitHub)](https://github.com/loopandlearn/lnl-scripts)
- [loopnlearn/loopbuildscripts (legacy, GitHub)](https://github.com/loopnlearn/loopbuildscripts)
- [LoopDocs — Build Loop](https://loopkit.github.io/loopdocs/build/build-app/)
- In-repo anchors: `Loop.xcconfig`, `Version.xcconfig`, `Common/FeatureFlags.swift`,
  `Loop/Models/LoopConstants.swift`, `LoopCore/LoopCoreConstants.swift`,
  `Common/Models/CarbAbsorptionTime.swift`,
  `Loop/View Controllers/CarbAbsorptionViewController.swift`.
