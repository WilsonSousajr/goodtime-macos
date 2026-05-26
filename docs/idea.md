# Goodtime macOS — design idea

Personal fork of [adrcotfas/Goodtime](https://github.com/adrcotfas/goodtime). Adds a **native macOS app** alongside the existing Android app, with **cross-device sync** via a self-hosted Supabase backend. iOS stays as-is. All Pro/paywall code is removed since this is a single-user project. Distribution is sideloaded APK + directly built `.app`. A web client is possible future scope.

## 1. Decision summary

| Area | Decision |
|---|---|
| Mac UI stack | SwiftUI, native — consumes KMP `ComposeApp.framework` for business logic |
| Mac form factor | Hybrid: `NavigationSplitView` window + `MenuBarExtra` with live countdown |
| Mac product identity | Productivity hub (Today / Stats / Labels / Settings) |
| Sync backend | Self-hosted Supabase (via Coolify), eventually-consistent |
| Sync scope | Sessions + labels + settings. **No** live timer state. |
| Auth | Google OAuth only, single user |
| Persistence (all platforms) | KMP Room — Mac adds `macosX64` / `macosArm64` targets |
| iOS | Untouched — no Supabase, no UI changes, keeps existing iCloud backup |
| Android | Add Supabase sync; remove all Pro/RevenueCat/billing; drop flavors; APK-only |
| Mac-native v1 features | Global hotkeys, macOS notifications, Focus mode auto-toggle, sleep prevention, live menu bar countdown |
| macOS minimum | 14 (Sonoma) |
| Repo layout | Monorepo: `composeApp/`, `iosApp/`, **`macApp/`** (new), **`backend/`** (new) |

## 2. System topology

```
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│  Android phone  │ ◄─────► │    Supabase     │ ◄─────► │       Mac       │
│   (APK only,    │  HTTPS  │   (via Coolify  │  HTTPS  │  (SwiftUI app + │
│   sideloaded)   │         │ on Hostinger BR)│         │   ComposeApp)   │
└─────────────────┘         └─────────────────┘         └─────────────────┘
        │                                                        │
        ▼ canonical local Room DB              canonical local Room DB ◄
   eventually-consistent sync (push local, pull remote, last-write-wins)
```

- **Android phone** — existing KMP/Compose app with Supabase sync client added; Pro/billing code removed. Local Room remains canonical offline. Single APK, no flavors.
- **Supabase** — Postgres + GoTrue + PostgREST, deployed via [Coolify](https://coolify.io) on a Hostinger KVM 4 VPS in São Paulo. Schema as SQL migrations in `backend/`. RLS trivially restricts to one user.
- **Mac** — fresh SwiftUI shell consuming the same `ComposeApp.framework` plus two new macOS-native KMP targets. Same Room schema, same sync layer as Android.
- **iOS** — out of scope. Stays in repo for posterity; iCloud backup unchanged.

## 3. Repo structure

```
goodtime-macos/
├── composeApp/            KMP shared code
│   └── src/
│       ├── commonMain/    + Supabase sync layer (NEW)
│       ├── androidMain/   unchanged platform actuals
│       ├── iosMain/       unchanged
│       └── macosMain/     NEW — macOS platform actuals
├── iosApp/                unchanged
├── macApp/                NEW — SwiftUI Xcode project
├── backend/               NEW — Supabase config & SQL migrations
├── docs/                  NEW/expanded — idea, infrastructure, security, ci-cd, testing, issues
└── .github/               NEW — workflows, issue templates, PR template
```

KMP `composeApp` additions:
- `macosX64`, `macosArm64` Kotlin/Native targets
- `src/macosMain/` source set for macOS `actual` implementations
- KSP for Room: `kspMacosX64`, `kspMacosArm64`
- Supabase Kotlin client in `commonMain` dependencies

## 4. Sync architecture

### What syncs
- **Sessions** (`LocalSession` rows): completed sessions — timestamps, duration, label, interruptions, notes
- **Labels** (`LocalLabel` rows): names, colors, archived flag, per-label timer profile
- **Settings** (synced subset): default durations, notification preferences, label↔Focus mappings. Pure-UI state (theme, window position, sidebar selection) stays device-local.

### What does NOT sync
- Live timer state (active session, pause/resume, remaining time)
- Device-local UI preferences
- Backups / exports

### Data flow

Both clients:
1. Local writes go to Room first — offline-first; UI never blocks on network.
2. `SyncWorker` (KMP, in `commonMain`) coalesces writes and batches them to Supabase.
3. Same worker pulls remote changes since `lastSyncAt` and applies to Room.
4. Triggers: app foreground, debounced on-write (~5 s), periodic while active (~60 s).

### Schema strategy
- Postgres tables mirror Room entities 1:1.
- Every synced row: `id` (uuid), `device_id` (uuid of origin), `updated_at` (timestamptz), `deleted` (boolean tombstone).
- **Conflict resolution: last-write-wins per row** using `updated_at`. Sessions are mostly immutable so conflicts are rare; labels and settings — last edit wins.
- Soft deletes via tombstones so deletions propagate.

### Auth
- Supabase Google OAuth provider; one Google account.
- Android: reuses existing `libs.google.id` / `libs.google.play.auth`, redirected to Supabase.
- Mac: `ASWebAuthenticationSession` for the OAuth handshake.
- Token storage: Keychain (Mac), EncryptedSharedPreferences (Android).

## 5. Mac app architecture

### App shell
- `@main App` declares two scenes: `WindowGroup` → `RootView`, `MenuBarExtra` → `MenuBarView`.
- macOS 14+ only — no fallback paths.
- Bundle ID: `com.apps.adrcotfas.goodtime.mac`.

### Window structure
- `NavigationSplitView` (sidebar + detail).
- Sidebar items: **Today**, **Stats**, **Labels**, **Settings**.
- Detail pane swaps with selection; toolbar holds context actions (Start session, Add label).

### Menu bar extra
- Always present in system menu bar.
- Idle: clock icon only.
- Running: icon + `"<label> · MM:SS"` updated every second.
- Click → small popover with current session info, Pause/Skip/+5min, "Open hub" link.

### Consuming KMP
- `ComposeApp.framework` (static) extended to two macOS targets.
- Swift consumes Kotlin types via generated headers.
- Thin Swift adapters bridge Kotlin `StateFlow` to `@Observable` / `AsyncSequence` for SwiftUI.
- Koin DI initialized in Swift app startup.

### Persistence
- Room DB file at `~/Library/Application Support/com.apps.adrcotfas.goodtime.mac/goodtime.db`.
- Same `ProductivityDatabase` schema as Android.
- KSP generates Room code for `kspMacosX64` and `kspMacosArm64`.

## 6. Mac-native integrations (v1)

1. **Global keyboard shortcuts** — [`KeyboardShortcuts`](https://github.com/sindresorhus/KeyboardShortcuts) (Sindre Sorhus) or `NSEvent.addGlobalMonitorForEvents`. Default actions: Start/Pause, Skip, Open hub. User-rebindable in Settings.
2. **macOS user notifications** — `UNUserNotificationCenter` on session end. Per-label sound configurable (reuses existing `SoundPickerDialog` settings in DataStore).
3. **Focus mode auto-toggle** — Per-label mapping (label → Focus mode). On session start, attempt via `INFocus` / Shortcuts integration; restore on end. macOS Focus APIs are constrained; fallback is invoking a user-configured Shortcut.
4. **Sleep prevention** — `IOPMAssertionCreateWithName(kIOPMAssertionTypePreventUserIdleDisplaySleep, ...)` on start; release on end / pause. ~30 lines of Swift.
5. **Live menu bar countdown** — `MenuBarExtra` label computed from a `Timer` ticking each second. `"<label> · MM:SS"` running; icon-only idle.

## 7. What gets removed

**Code deletions:**
- `composeApp/src/iosMain/.../billing/` — entire directory (RevenueCat)
- `composeApp/src/androidGoogle/.../billing/` — entire directory (Play Billing)
- `composeApp/src/commonMain/.../billing/` — `PurchaseManager` interface and `expect` declarations
- All Pro-gated feature checks in `commonMain` — make unconditional, then remove the conditional entirely
- `composeApp/src/androidGoogle/.../backup/` — Google Drive backup (Supabase supersedes)

**Dependency drops from `composeApp/build.gradle.kts`:**
- `libs.purchases.core`, `libs.purchases.ui`
- `googleImplementation` lines for `app.update.ktx`, `review.ktx`, billing variants
- `libs.google.drive`, `libs.google.api.client`

**Flavor consolidation:**
- Drop `google` / `fdroid` flavors → single APK
- Remove `androidGoogle/` and `androidFdroid/` source sets
- Drop `flavorDimensions` and `productFlavors` blocks

**What stays:**
- Google Sign-In — repurposed for Supabase OAuth
- `LocalAutoBackupManager` (Storage Access Framework) — manual export escape hatch
- iCloud backup in `iosMain/` — iOS is out of scope

## 8. Testing strategy

### Layers

1. **commonTest (KMP unit tests)** — pure-Kotlin tests for business logic. The existing `composeApp/src/commonTest/` already has a small footprint; expand to cover:
   - `TimerManager` state transitions (full event-driven coverage)
   - `FinishedSessionsHandler` persistence side-effects
   - Sync layer: push payload construction, pull-and-merge, tombstone propagation, last-write-wins conflict resolution
   - Schema mapping between Room `LocalSession`/`LocalLabel` and Supabase row shapes
   - `SyncWorker` debouncing/coalescing semantics under simulated workloads

2. **Sync integration tests** — `SyncWorker` against a real Supabase instance using **Testcontainers** (Postgres + GoTrue + PostgREST started ephemerally per test class). Validates round-trip: client A writes → Supabase → client B reads.

3. **androidUnitTest** — Android-specific actuals (already a source set). Cover platform service implementations, particularly notification scheduling and the new Supabase OAuth handshake.

4. **macOS Swift tests** — `XCTest` in `macApp/` for Swift adapters (the bridges from Kotlin `StateFlow` to SwiftUI `@Observable`), `MenuBarExtra` rendering invariants, and the IOPMAssertion lifecycle (start session → assertion held; end session → assertion released).

5. **End-to-end smoke (manual, scripted)** — a `scripts/smoke.sh` that:
   - Brings up Supabase via Docker Compose locally
   - Builds + installs the APK on a connected device
   - Builds the Mac `.app` and launches it
   - Runs an instrumented session, asserts both clients converge

### Conventions
- Mock external I/O behind **named fake classes** (per `CLAUDE.md` conventions): `FakeSupabaseClient`, `FakeLocalDataRepository`, `FakeUserNotificationCenter`, etc. No inline lambdas as test doubles.
- Tests run with `./gradlew :composeApp:check` (Kotlin) and `xcodebuild test -scheme macApp` (Swift).
- New features require tests; bug fixes require a regression test that fails before the fix.
- Coverage measurement via **JaCoCo** (Android) and **Xcode coverage** (Mac), surfaced in the CI summary. No hard coverage gate — single user, judgment-driven.

## 9. Out of scope (explicit)

- Web client (future, separate spec)
- iOS — no Supabase, no UI changes, no Pro removal
- Live timer state sync (defer; reconsider if usage warrants)
- App Store / Play Store distribution
- Multi-user / family sharing
- Notification Center widget, Shortcuts.app actions, Calendar awareness, URL scheme, dock badge (named as stretch in brainstorm; defer)
- Migration tooling for previously-purchased Pro users (single-user fork; no upgrade path)
