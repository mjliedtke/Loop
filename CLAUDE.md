# CLAUDE.md

Guidance for AI coding agents (and humans) working in this repository.

## What this is

**Loop** is an app template for building an **automated insulin delivery (AID)** system for iOS / watchOS — a "closed-loop" controller that reads continuous glucose monitor (CGM) data, predicts future glucose, and commands an insulin pump (temp basal + automatic bolus) to keep glucose in range.

- Written in **Swift**, built with **Xcode** (`Loop.xcodeproj`). It **cannot be built on Linux** — it requires macOS + Xcode + the iOS SDK.
- Built on top of **[LoopKit](https://github.com/LoopKit/LoopKit)** (a separate repo) which provides the data stores, device-manager protocols, the dose-math/algorithm, and shared UI.
- This repo is normally compiled **inside the LoopWorkspace super-repo**, which vends LoopKit and the other framework dependencies. See "Building & testing".
- The project is **highly experimental and not approved for therapy**. Treat the dosing path as safety-critical.

## ⚠️ Safety-critical context

This software decides how much insulin a person receives. Bugs here can cause real harm.

- The dosing brain is **`Loop/Managers/LoopDataManager.swift`** (the central orchestrator) plus the algorithm dependency in LoopKit. Dose *enactment* goes through **`Loop/Managers/DoseEnactor.swift`** → the pump manager.
- When touching prediction, recommendation, enactment, safety guards, or the data-recency checks, be conservative: preserve existing guards, add tests, and prefer the most cautious behavior. Do **not** loosen clamps, recency windows, or suspend/headroom checks without explicit instruction.
- Key guards already in place: configuration completeness (schedules/limits must exist), data recency (`LoopCoreConstants.inputDataRecencyInterval = 15 min`), recommendation expiration, pump-suspended checks, IOB headroom clamping, "bolus in progress" suppression of automatic dosing, and crash recovery (`Loop/Models/CrashRecoveryManager.swift`).

## Building & testing

> Requires **macOS + Xcode**. Do not attempt to build/test in a Linux container — there is no Swift Package manifest to `swift build` and the iOS SDK is unavailable.

**Dependency model:** There is **no `Cartfile`, `Package.swift`, or `Podfile` in this repo**. Loop expects its dependency frameworks (LoopKit, LoopKitUI, CGM/pump plugins, TrueTime, etc.) to be prebuilt and present in `BUILT_PRODUCTS_DIR`. That wiring is owned by **LoopWorkspace** — clone/build through LoopWorkspace, not this repo standalone.

Commands (mirrors `.travis.yml`; run on a Mac):

```bash
# Build the app (skip code signing)
xcodebuild -project Loop.xcodeproj -scheme Loop build \
  CODE_SIGN_IDENTITY="" CODE_SIGNING_ALLOWED=NO | xcpretty

# Build the Learn companion app
xcodebuild -project Loop.xcodeproj -scheme Learn build \
  CODE_SIGN_IDENTITY="" CODE_SIGNING_ALLOWED=NO | xcpretty

# Unit/integration tests
xcodebuild -project Loop.xcodeproj -scheme LoopTests \
  -destination 'platform=iOS Simulator,name=iPhone 8' test EXCLUDED_ARCHS=arm64 | xcpretty

# Dose-math tests
xcodebuild -project Loop.xcodeproj -scheme DoseMathTests \
  -destination 'name=iPhone 8' test | xcpretty
```

**Shared schemes** (`Loop.xcodeproj/xcshareddata/xcschemes/`): `Loop`, `Learn`, `LoopTests`, `DoseMathTests`, `WatchApp`, `Loop Status Extension`, `Loop Intent Extension`, `SmallStatusWidgetExtension`.

**Configuration files:**

| File | Purpose |
| --- | --- |
| `Loop.xcconfig` | Master build settings: bundle id (`com.${DEVELOPMENT_TEAM}.loopkit`), display name, entitlements, code signing/provisioning, `IPHONEOS_DEPLOYMENT_TARGET = 15.1`, `LOOP_LOCAL_CACHE_DURATION_DAYS = 7`. `#include?`s `Version.xcconfig` and the optional overrides below. |
| `Version.xcconfig` | `LOOP_MARKETING_VERSION` / `CURRENT_PROJECT_VERSION`. For DIY builds the real version is set in LoopWorkspace. |
| `LoopOverride.xcconfig`, `../LoopConfigOverride.xcconfig`, `VersionOverride.xcconfig`, `../VersionOverride.xcconfig` | Optional, git-ignored local/workspace overrides (signing, team, version). |

**Build-phase scripts** (`Scripts/`):

| Script | Role |
| --- | --- |
| `build-derived-assets.sh` | Generates `DerivedAssets.xcassets` from `DerivedAssetsBase.xcassets` (+ optional override catalogs) so the icon/assets can be customized per build. |
| `capture-build-details.sh` | Writes `BuildDetails.plist` (git revision/branch, Xcode version, build date, provisioning expiration). |
| `apply-info-customizations.sh` | Applies `../InfoCustomizations.txt` key/values onto build metadata. |
| `copy-plugins.sh` | Copies `.loopplugin` bundles found in `BUILT_PRODUCTS_DIR` into the app's `Frameworks/` (renamed to `.framework`) and code-signs them — this is how pump/CGM/service plugins get embedded. |
| `install-scenarios.sh` | If a `Scenarios/` dir exists at the workspace root, copies it into the app bundle (debug testing scenarios). |
| `make_scenario.py` | Manual helper to generate scenario JSON (not run during build). |

## Repository layout / module map

Layering: **`LoopCore` (framework)** and **`Common` (shared sources)** sit below everything and are consumed by the app, the watch app, and the extensions. The **`Loop`** app target depends on **`LoopUI`** (reusable UI framework) and on external LoopKit frameworks.

| Path | Kind | Purpose |
| --- | --- | --- |
| `Loop/` | iOS app target | Main app: managers (control loop, devices, services, alerts), models, view controllers, view models, SwiftUI views, plugins loader. |
| `LoopCore/` | iOS framework | Shared domain + persistence primitives (`LoopSettings`, `PersistenceController`, `PersistedProperty`, `LoopCoreConstants`, `LoopCompletionFreshness`, `NSUserDefaults` app-group accessor, `HKUnit`, `LocalizedString`). Imported by app, watch, and all extensions. |
| `Common/` | shared sources | Cross-target models and extensions compiled into multiple targets, including the IPC payloads (`StatusExtensionContext`, `WatchContext`, `*UserInfo`) and **`FeatureFlags.swift`**. |
| `LoopUI/` | iOS framework | Reusable UI: HUD views, charts (`ChartsManager` subclasses), shared view types. Used by the app and the status extension. |
| `Learn/` | iOS app target | Standalone educational/analysis companion app. |
| `WatchApp/` | watchOS app | Watch app bundle: storyboards, assets, localizations (almost no Swift). |
| `WatchApp Extension/` | watchOS extension | The watch app's actual code: `ExtensionDelegate`, `ComplicationController`, a local `LoopDataManager`, UI controllers. |
| `Loop Widget Extension/` | iOS extension | Modern **WidgetKit** widgets (home/lock screen). |
| `Loop Status Extension/` | iOS extension | Legacy Today/status widget (`NCWidgetProviding`). |
| `Loop Intent Extension/` | iOS extension | Siri **Intents** handler (e.g. enable override preset). |
| `Shortcuts/` | resources | Prebuilt `.shortcut` files for the Shortcuts app. |
| `LoopTests/` | unit test bundle | XCTest tests for the app (managers, view models, models) + mock stores + JSON fixtures. |
| `DoseMathTests/` | test scheme/target | Exercises dose-math; the directory here holds only resources — the test sources come from the LoopKit dependency. |
| `Scripts/` | build tooling | See "Build-phase scripts" above. |
| `Documentation/` | docs/assets | Screenshots, customization images, `Testing/Scenarios.md`. |

## Architecture & key components

### The control loop (dosing pipeline)
1. **Trigger** — New CGM samples arrive via `DeviceDataManager` and are stored; a timer also drives a loop run roughly every 5 minutes. `CGMStalenessMonitor` gates automatic dosing when CGM data is stale.
2. **Gather & predict** — `LoopDataManager` (`Loop/Managers/LoopDataManager.swift`) fetches glucose history and computes insulin, carb, momentum, and retrospective-correction effects (via LoopKit stores), then projects a glucose forecast.
3. **Recommend** — From the prediction it computes either a temp-basal-only adjustment or an automatic bolus. The bolus fraction is chosen by an **application-factor strategy** (Strategy pattern): `Loop/Models/ApplicationFactorStrategy.swift` with `ConstantApplicationFactorStrategy` (fixed fraction) and `GlucoseBasedApplicationFactorStrategy` (fraction scales with glucose). Recommendations are clamped by max bolus/basal and IOB headroom.
4. **Gate** — Automatic dosing requires both `automaticDosingEnabled` and `isAutomaticDosingAllowed` (`Loop/Models/AutomaticDosingStatus.swift`), combined in `LoopAppManager`. Checks include config completeness, data recency, recommendation expiration, and pump-not-suspended.
5. **Enact** — `DoseEnactor` issues the temp basal then the bolus to the pump manager.
6. **Record** — The full decision (inputs + outputs, plus any `LoopWarning`) is persisted to the dosing-decision store; widgets/watch are refreshed; `.LoopCompleted` is posted.

Errors are modeled in `Loop/Models/LoopError.swift`; non-fatal issues in `Loop/Models/LoopWarning.swift`. Tunable constants: `Loop/Models/LoopConstants.swift` (app-level) and `LoopCore/LoopCoreConstants.swift` (shared, e.g. recency intervals).

### Devices, services & plugins (protocol-oriented, pluggable)
- **`DeviceDataManager`** is the hub owning the pump manager, CGM manager, and the LoopKit data stores; it coordinates alerts, settings, and remote upload.
- **Plugins**: `Loop/Plugins/PluginManager.swift` discovers pump/CGM/service/support plugins from the app's private frameworks (`.loopplugin` bundles embedded by `copy-plugins.sh`), keyed by Info.plist identifiers.
- **Services** implement the LoopKit `Service` protocol and are aggregated by `ServicesManager`, which fans them out to `AnalyticsServicesManager`, `LoggingServicesManager`, and `RemoteDataServicesManager` (uploads glucose/carbs/doses/settings/etc. to remote services such as Nightscout, tracked per-service via query anchors). `StatefulPluginManager` persists plugin state via `@PersistedProperty`.
- **Alerts**: `Loop/Managers/Alerts/` — `AlertManager` issues/retracts alerts, schedules them through `InAppModalAlertScheduler` and `UserNotificationAlertScheduler`, persists them as Core Data `StoredAlert`, replays unacknowledged alerts on launch, and honors `AlertMuter`. `NotificationManager` sends one-off rich notifications.
- **Settings**: `SettingsManager` mirrors `LoopSettings` into a Core Data `SettingsStore` and triggers remote upload on change.

### UI layer (mid-migration UIKit → SwiftUI, MVVM)
- The app is **part UIKit, part SwiftUI**. The main status screen (`Loop/View Controllers/StatusTableViewController.swift`, on a `LoopChartsTableViewController` base) is UIKit + table view, driven by `.LoopDataUpdated`/`.LoopRunning` notifications with a `RefreshContext` batching pattern.
- Newer screens (bolus entry, carb entry, settings) are **SwiftUI** with `ObservableObject` view models in `Loop/View Models/` (`@MainActor`, `@Published`, Combine), presented from UIKit via `UIHostingController`. View models talk to backends through delegate protocols (e.g. `BolusEntryViewModelDelegate`).
- **Charts**: `Loop/Managers/StatusChartsManager.swift` (extends LoopKit's `ChartsManager`) manages the glucose/IOB/dose/COB charts; SwiftUI wraps them via `UIViewRepresentable` (e.g. `PredictedGlucoseChartView`). Reusable HUD/chart components live in `LoopUI/`.
- **Convention:** build new UI in SwiftUI + a view model; keep business logic out of views and behind delegate seams.

### Data sharing to watch, widgets & extensions
- **App group** is the backbone: `UserDefaults.appGroup` (`LoopCore/NSUserDefaults.swift`, suite name derived from the bundle id in `Common/Extensions/NSBundle.swift`) and a shared Core Data container (`PersistenceController`). Extensions open the Core Data store **read-only** (`PersistenceController` detects `isAppExtension`).
- **Widgets / Status extension**: `Loop/Managers/ExtensionDataManager.swift` writes a versioned `StatusExtensionContext` (`Common/Models/`) to the app group; the WidgetKit timeline provider and the legacy status VC read it (plus the local glucose cache).
- **Watch**: `Loop/Managers/WatchDataManager.swift` sends a versioned `WatchContext` over `WCSession` (application context / complication transfers), and `*UserInfo` payloads (settings, bolus requests, backfill). The watch replies with bolus/carb requests. Complication updates are budget-throttled.
- **Intents/Shortcuts**: `ExtensionDataManager` publishes override preset names; the Intent extension reads them and signals the app to apply an override.
- **Gotcha:** these IPC models (`StatusExtensionContext`, `WatchContext`, `*UserInfo`) are **versioned** and decode all-or-nothing — bump the version and handle migration on both sides when changing their shape.

## Conventions

- **Managers**: app-wide coordinators live in `Loop/Managers/*Manager.swift`, typically long-lived reference types with delegate protocols. Follow the existing manager pattern when adding cross-cutting behavior.
- **Localization**: user-facing strings go through `NSLocalizedString`; framework code (LoopCore/LoopUI) uses the `LocalizedString` helper so the correct framework bundle is used. There are ~22 `.lproj` locales under `Loop/` and `Common/`; keys must be **static string literals** (not interpolated) for extraction to work. Root `Interface.strings` holds storyboard strings.
- **Feature flags**: `Common/FeatureFlags.swift` gates optional/experimental behavior (e.g. automatic bolus, nonlinear carb model, scenarios). Flags use compile-time conditions with some `UserDefaults.appGroup` fallbacks (`allowDebugFeatures`, `allowExperimentalFeatures`, `allowSimulators`). Check the relevant flag before assuming a feature is on.
- **Persistence**: prefer the existing layers — `@PersistedProperty` (plist files, app-group or local), `LoopSettings`/`*Context` `RawRepresentable` dictionaries (versioned), and Core Data via `PersistenceController` (write only from the app target, read-only in extensions).
- **Plugins**: integrate pump/CGM/service support as LoopKit plugins (Info.plist identifier + principal class), not by editing the app target directly.

## Testing conventions & scenarios

- **Framework**: XCTest. Tests live in `LoopTests/` grouped by type (`Managers/`, `ViewModels/`, `Models/`, `LoopCore/`); `DoseMathTests` is a separate scheme. Name files `<Type>Tests.swift` and methods `test…`. Use `@testable import Loop`.
- **Mocks & fixtures**: reusable mocks are in `LoopTests/Mock Stores/` (`MockCarbStore`, `MockGlucoseStore`, `MockDoseStore`, etc.); algorithm tests load JSON from `LoopTests/Fixtures/<scenario>/`. The `DosingTestScenario` enum (in `LoopTests/Managers/LoopDataManagerTests.swift`) parametrizes a scenario's fixtures and baseline date. Use `expectation`/`waitForExpectations` (and the `waitOnMain` helper) for async tests.
- **Testing scenarios (debug-only)**: a "scenario" is a JSON file of timed `glucoseValues`/`basalDoses`/`bolusDoses`/`carbEntries` (offsets relative to "now"). With the Simulator pump/CGM selected, `LocalTestingScenariosManager` (gated by `FeatureFlags.scenariosEnabled`) loads scenarios from the app bundle's `Scenarios/` dir and can step the loop forward/back in time. Author scenarios with `Scripts/make_scenario.py`, ship them via `Scripts/install-scenarios.sh`. See `Documentation/Testing/Scenarios.md`.

## Gotchas & non-obvious constraints

- **Cannot build/test on Linux** and **cannot build this repo alone** — use macOS + Xcode via LoopWorkspace (dependencies come from `BUILT_PRODUCTS_DIR`).
- **Dosing code is safety-critical** — keep guards/clamps/recency checks; add tests; never loosen limits casually.
- **Versioned IPC models** (`StatusExtensionContext`, `WatchContext`, `*UserInfo`) decode all-or-nothing; changing shape needs a version bump + migration on both producer and consumer.
- **Extensions are read-only** consumers of the shared Core Data store; only the app target writes.
- **App-group entitlement** must be shared by the app + all extensions + watch for data flow to work.
- **`LoopSettings` invariants**: pre-meal and (legacy) workout overrides are mutually exclusive (enforced with preconditions) — set them through the proper APIs.
- **Localization extraction** breaks if you pass non-literal strings to `NSLocalizedString`/`LocalizedString`.
- **Complication updates are budget-limited** by watchOS; `WatchDataManager` deliberately throttles them.

## References

- Loop docs: https://loopkit.github.io/loopdocs/ — installation, algorithm, configuration.
- Wiki: https://github.com/LoopKit/Loop/wiki — contributing, FAQs, tips.
- `README.md`, `LICENSE.md`, `CODE_OF_CONDUCT.md`.
- In-repo: `Documentation/Testing/Scenarios.md` (testing scenarios), `Loop.xcconfig` / `Version.xcconfig` (build config).
- Upstream framework: https://github.com/LoopKit/LoopKit (stores, device protocols, dose math, shared UI).
