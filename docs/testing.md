# Testing strategy

(See also `docs/idea.md` § 8.)

## Test layers

### 1. commonTest — KMP unit tests
- Location: `composeApp/src/commonTest/`
- Run: `./gradlew :composeApp:allTests` (cross-platform) or `:composeApp:jvmTest` for fastest feedback
- Coverage targets: business logic, sync layer, conflict resolution, schema mapping

### 2. Sync integration tests
- Location: `composeApp/src/commonTest/kotlin/.../sync/integration/`
- Uses **Testcontainers** to spin up Supabase ephemerally per test class
- Run: same `:composeApp:allTests`, but only on platforms with Docker (skipped on iOS native)

### 3. androidUnitTest
- Location: `composeApp/src/androidUnitTest/`
- Run: `./gradlew :composeApp:testDebugUnitTest`
- Focus: Android platform actuals, notification scheduling, Supabase OAuth handshake

### 4. macOS Swift tests
- Location: `macApp/macAppTests/`
- Run: `xcodebuild test -scheme macApp -destination 'platform=macOS'`
- Focus: Kotlin↔Swift bridges, `MenuBarExtra` rendering invariants, `IOPMAssertion` lifecycle

### 5. End-to-end smoke (manual / scripted)
- Location: `scripts/smoke.sh`
- Brings up Supabase locally, builds + installs APK on connected device, builds Mac `.app`, runs a scripted Pomodoro, asserts convergence

## Conventions

- **Named fake classes** per `CLAUDE.md` Conventions § Tests: `FakeSupabaseClient`, `FakeLocalDataRepository`, `FakeUserNotificationCenter`, etc.
- **F.I.R.S.T.** — Fast, Independent, Repeatable, Self-validating, Timely.
- Every new function: a test. Every bug fix: a regression test failing before the fix.
- No inline lambda doubles; no `mockk` reflection magic.

## CI integration

Per `docs/ci-cd.md`:
- `ci-kmp.yml` runs `:composeApp:check` (includes all KMP test layers)
- `ci-mac.yml` runs `xcodebuild test`
- Coverage from both jobs surfaced in the Actions summary; no hard gate
