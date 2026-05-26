# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Goodtime is a productivity timer (Pomodoro, longer breaks, count-up flow). It is a **Kotlin Multiplatform** app targeting Android (primary, distributed via Google Play and F-Droid) and iOS (TestFlight). The repo folder is named `goodtime-macos` and the Gradle root is `goodtime-productivity`, but there is no separate macOS target — the macOS experience comes from running the iOS framework.

## Build & Test

All commands use the Gradle wrapper from the repo root. JVM toolchain is **JDK 17**.

- `./gradlew :composeApp:check` — full check (unit tests + lint) across all source sets. This is what CI runs.
- `./gradlew spotlessCheck` — verify Kotlin/XML/Gradle formatting. CI runs this too.
- `./gradlew spotlessApply` — auto-fix formatting. Also runs automatically before every `preBuild` (see [build.gradle.kts](build.gradle.kts)), so any local build will reformat staged files — don't be surprised when the working tree changes.
- `./gradlew :composeApp:testDebugUnitTest --tests "com.example.MyTest.myMethod"` — run a single Android unit test.
- `./gradlew :composeApp:iosSimulatorArm64Test` — iOS unit tests on the simulator. Or open `iosApp/iosApp.xcodeproj` in Xcode for full iOS dev.
- `./gradlew :composeApp:assembleGoogleDebug` / `assembleFdroidDebug` — Android APKs for the two flavors (`google` and `fdroid` differ in store auth/billing).

CI (`.circleci/config.yml`) only runs on `master`, so PR-targeted branches receive no automatic checks — run `./gradlew :composeApp:check spotlessCheck` locally before pushing.

Kotlin/Native compilation is memory-hungry; `gradle.properties` already sets 6 GB for Kotlin/Native and 3 GB for the Gradle daemon.

## Architecture

Single Gradle module (`:composeApp`) with KMP source sets:

- `commonMain/` — shared Compose UI, business logic, Room DB, Koin DI
- `androidMain/` — Android `actual` implementations (audio, vibration, foreground service, ACRA crash reporting)
- `iosMain/` — iOS `actual` implementations (audio, haptics, RevenueCat billing)
- `androidGoogle/` / `androidFdroid/` — product-flavor code (Play Store auth + billing vs. nothing)
- `iosApp/` — **separate Xcode project**; Swift app shell that consumes the KMP-compiled `ComposeApp.framework` and adds the Live Activity widget (`GoodtimeInProgress/`)

**Timer is event-driven.** `bl/TimerManager.kt` holds `DomainTimerData` (IDLE/RUNNING/PAUSED, duration, label, long-break state) as a `StateFlow`. Side effects live in `EventListener`s — `SoundVibrationAndTorchPlayer`, `FinishedSessionsHandler`, etc. — that react to lifecycle events. To add behavior on timer events, register a new `EventListener`; don't reach into the UI.

**Persistence is two-layer:**
- Room (SQLite) at `data/local/Database.kt` — sessions, labels, timer profiles. Schemas are committed under `composeApp/schemas/`. KSP runs the Room compiler for both Android and all three iOS targets (see the `kspAndroid` / `kspIosX64` / `kspIosArm64` / `kspIosSimulatorArm64` dependencies in [composeApp/build.gradle.kts](composeApp/build.gradle.kts)). Migrations are manual.
- DataStore (Preferences) at `data/settings/` — user settings. Always observe `settingsRepo.settings` as a `Flow`; don't read point-in-time.

**Navigation** is type-safe via Compose Navigation + kotlinx.serialization routes (sealed classes in `main/`). Use `navController.navigate<Dest>()` and `entry.toRoute<Dest>()`.

**DI:** Koin. Viewmodels are obtained via `koinViewModel()`.

**Platform-specific files use `.ios.kt` / `.android.kt` suffixes** with `expect` / `actual` declarations. Add a new `expect` in `commonMain` and the corresponding `actual`s in `iosMain` / `androidMain` rather than branching on platform at runtime.

## iOS-specific

- **Live Activities** (lock-screen timer) are in the `iosApp/GoodtimeInProgress/` widget extension. The bridging contract — how the Kotlin host app shares state with the Swift extension when the host process is killed — is documented in detail at [docs/architecture/IOS_LIVE_ACTIVITY_HOST_APP_KILLED_DESIGN.md](docs/architecture/IOS_LIVE_ACTIVITY_HOST_APP_KILLED_DESIGN.md). Read this before touching live-activity code.
- iOS framework name is `ComposeApp` (static, exports `touchlab.kermit.simple`).
- Several androidx artifacts don't link on iOS, so iOS configurations substitute them at the dependency-resolution layer (paging-common-ktx → paging-common, lifecycle-runtime-ktx → lifecycle-runtime, kotlinx-coroutines-android → kotlinx-coroutines-core, etc.) — see the `configurations.all` block in [composeApp/build.gradle.kts](composeApp/build.gradle.kts). If a new androidx dependency fails the iOS link, add an entry there.
- iOS version bumping is currently manual; the `syncIosVersion` Gradle task is commented out near the bottom of `composeApp/build.gradle.kts` ("re-enable after we're out of TestFlight").

## Formatting

Spotless enforces ktlint, GPL license headers, trailing-whitespace removal, and EOF newlines on `.kt`, `.xml`, and `.gradle.kts` files. Header templates: `.spotless/license.kt`, `.spotless/license.xml`. The `.editorconfig` tells ktlint to ignore function-name lint for `@Composable` and `@Test` functions and to allow trailing commas.

Don't hand-format — let `./gradlew spotlessApply` do it, and don't argue about style beyond what ktlint enforces.

## Conventions

### Code style

- Functions: 4–20 lines. Split when longer. Files: under 500 lines; split by responsibility.
- One thing per function, one responsibility per module (SRP).
- Names must be specific and grep-friendly — aim for under 5 hits in the codebase for a given symbol. Avoid `data`, `handler`, `info`. The existing `*Manager` types (`TimerManager`, `ReminderManager`, etc.) predate this rule and stay — but prefer a concrete role name (`TimerScheduler`, `SessionRecorder`) for new code.
- Explicit Kotlin types on public APIs. Avoid `Any?`, raw generic types, and untyped lambdas crossing module boundaries.
- No duplicated logic. Extract into a shared function — and when behavior diverges by platform, into an `expect` / `actual` pair (see existing `SoundPlayer`, `VibrationPlayer`, `TorchManager`).
- Early returns. Keep nesting to at most two levels of `if` / `when` / `for`.
- Exception messages must include the offending value and the expected shape, e.g. `require(durationMs > 0) { "expected positive timer duration, got $durationMs ms" }`.

### Comments

- Write **why**, not what. Skip `// increment counter` above `i++`.
- Don't strip existing comments during a refactor — they carry intent and provenance.
- KDoc on public functions: state intent and a one-line usage example.
- Reference issue numbers or commit SHAs when a line exists because of a specific bug or upstream constraint (e.g. the androidx→iOS substitutions in [composeApp/build.gradle.kts](composeApp/build.gradle.kts)).

### Tests

- Run all checks with `./gradlew :composeApp:check`; single-test invocation is under **Build & Test** above.
- Every new function gets a test. Bug fixes get a regression test that fails before the fix.
- Mock external I/O (network, DB, filesystem, system clock, platform audio/haptics) behind **named fake classes** (e.g. `FakeLocalDataRepository`) — not inline stubs or anonymous objects.
- Tests must be F.I.R.S.T: Fast, Independent, Repeatable, Self-validating, Timely.

### Dependencies

- Inject collaborators through the constructor and wire them in a Koin module. Never service-locate from inside business logic or `import` a singleton.
- Wrap third-party libraries behind an interface this project owns. The `expect` / `actual` split already does this for platform APIs; do the same for new vendor SDKs (billing, analytics, etc.).

### Structure

- Organize `commonMain` by feature (`bl/`, `data/`, `settings/`, `main/`, `notifications/`). Keep modules small and focused; avoid god files.
- Put code in `androidMain` or `iosMain` only when it genuinely needs a platform API — everything else belongs in `commonMain`.

### Logging

- Use [touchlab.kermit](https://github.com/touchlab/Kermit) (already a dependency in `commonMain`). Prefer structured key/value pairs over interpolated strings for anything an engineer will later filter or grep.
- This is a mobile app; there is no CLI surface. User-visible messages are localized strings in Compose, not log lines.
