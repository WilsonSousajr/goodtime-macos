# GitHub issues — initial backlog

Grouped by phase. Each row becomes a single issue created via `gh issue create`. Labels in `[brackets]`.

## Phase 0 · Project setup

1. **Initialize GitHub project board** `[infra]` — Create a GH Projects v2 board with columns: Backlog / Up Next / In Progress / In Review / Done.
2. **Add issue templates** `[infra]` — `.github/ISSUE_TEMPLATE/{bug,feature,task}.yml`.
3. **Add PR template** `[infra]` — `.github/pull_request_template.md` with Summary, Test Plan, Related Issues sections.
4. **Document the build/run loop** `[docs]` — `docs/CONTRIBUTING.md` covering how to build APK, build Mac `.app`, run local Supabase, run all tests.

## Phase 1 · KMP foundations (BLOCKER for Mac work)

5. **Add `macosX64` / `macosArm64` KMP targets to `composeApp`** `[kmp]`
6. **Add KSP Room compiler for macOS targets** `[kmp][persistence]`
7. **Smoke test: "hello from Kotlin" framework loaded by stub Swift app** `[kmp][mac]`
8. **Add Supabase Kotlin client dependency to `commonMain`** `[kmp][sync]`
9. **Implement `expect` for secure token storage (Keychain / EncryptedSharedPreferences)** `[kmp][auth]`

## Phase 2 · Backend

10. **Scaffold `backend/` directory with Supabase CLI workspace** `[backend]`
11. **Define `sessions` table + initial SQL migration** `[backend][schema]`
12. **Define `labels` table + initial SQL migration** `[backend][schema]`
13. **Define `settings` table + initial SQL migration** `[backend][schema]`
14. **Add RLS policies for single-user model** `[backend][security]`
15. **Configure Google OAuth provider in Supabase** `[backend][auth]`
16. **Add `docker-compose.yml` for local Supabase development** `[backend]`
17. **Add `backend/README.md` with local setup instructions** `[backend][docs]`

## Phase 3 · Infrastructure (VPS + Coolify)

18. **Procure Hostinger KVM 4 VPS in São Paulo** `[infra][manual]` — manual purchase; track in this issue.
19. **Set up DNS A records for `goodtime.<your-domain>` and `coolify.<your-domain>`** `[infra][manual]`
20. **Provision VPS: deploy user, SSH key auth, baseline packages, UFW, fail2ban, unattended-upgrades** `[infra][security]`
21. **Install Coolify via the official one-line installer** `[infra]`
22. **Configure Coolify instance domain + enable 2FA on admin account** `[infra][security]`
23. **Deploy Supabase via Coolify's one-click service template** `[infra]`
24. **Mount `backend/supabase/migrations/` into the Supabase service and configure post-deploy migration command** `[infra][backend]`
25. **Close port 8000 in UFW after Coolify admin domain is reachable over HTTPS** `[infra][security]`
26. **Configure Coolify Supabase backups → Backblaze B2 (daily, 30-day retention)** `[infra][ops]`
27. **Deploy Uptime Kuma via Coolify, configure monitors + push notifications** `[infra][ops]`
28. **Document Coolify-driven operational runbook** `[infra][docs]`

## Phase 4 · Sync layer in commonMain

29. **Implement Supabase Kotlin client wrapper (auth + REST)** `[kmp][sync]`
30. **Implement `SyncWorker` push path (local → remote)** `[kmp][sync]`
31. **Implement `SyncWorker` pull path (remote → local)** `[kmp][sync]`
32. **Implement tombstone-based soft delete propagation** `[kmp][sync]`
33. **Implement last-write-wins conflict resolution** `[kmp][sync]`
34. **Implement debouncing + periodic trigger scheduling** `[kmp][sync]`
35. **Unit tests: schema mapping for sessions, labels, settings** `[kmp][sync][test]`
36. **Unit tests: conflict resolution edge cases** `[kmp][sync][test]`
37. **Integration tests: round-trip via Testcontainers Supabase** `[kmp][sync][test]`

## Phase 5 · Android integration

38. **Wire `SyncWorker` into Android app lifecycle** `[android][sync]`
39. **Implement Google OAuth flow on Android (Supabase redirect)** `[android][auth]`
40. **Add sync sign-in + status to Settings UI** `[android][ui]`
41. **Remove `composeApp/src/androidGoogle/.../billing/`** `[android][cleanup]`
42. **Remove `composeApp/src/commonMain/.../billing/` and `expect` declarations** `[kmp][cleanup]`
43. **Remove `composeApp/src/iosMain/.../billing/`** `[ios][cleanup]`
44. **Remove all Pro-gated feature checks** `[kmp][cleanup]`
45. **Remove `composeApp/src/androidGoogle/.../backup/` (Google Drive)** `[android][cleanup]`
46. **Drop `google` / `fdroid` flavors from `composeApp/build.gradle.kts`** `[android][cleanup]`
47. **Remove `androidGoogle/` and `androidFdroid/` source sets** `[android][cleanup]`
48. **Smoke test: build single APK, install, complete round-trip sync** `[android][test]`

## Phase 6 · Mac app shell

49. **Create `macApp/macApp.xcodeproj` SwiftUI project** `[mac]`
50. **Configure `macApp` to consume `ComposeApp.framework`** `[mac][kmp]`
51. **Initialize Koin in Swift app startup** `[mac][kmp]`
52. **Implement `RootView` with `NavigationSplitView`** `[mac][ui]`
53. **Implement Today pane (today's sessions + start button)** `[mac][ui]`
54. **Implement Stats pane (week / month charts)** `[mac][ui]`
55. **Implement Labels pane (CRUD)** `[mac][ui]`
56. **Implement Settings pane (durations, notifications, sync sign-in)** `[mac][ui]`
57. **Implement `MenuBarExtra` shell with idle/running states** `[mac][ui]`
58. **Implement menu bar popover with Pause/Skip/+5m** `[mac][ui]`

## Phase 7 · Mac-native integrations

59. **Add global keyboard shortcuts (Sindre Sorhus library)** `[mac][native]`
60. **Implement `UNUserNotificationCenter` end-of-session notification** `[mac][native]`
61. **Implement Focus mode auto-toggle (Shortcuts-app fallback)** `[mac][native]`
62. **Implement `IOPMAssertion` sleep prevention** `[mac][native]`
63. **Implement live menu bar countdown** `[mac][native]`
64. **Add Settings UI for shortcut rebinding** `[mac][ui]`

## Phase 8 · CI/CD

65. **Add `.github/workflows/ci-kmp.yml`** `[ci]`
66. **Add `.github/workflows/ci-mac.yml`** `[ci]`
67. **Add `.github/workflows/ci-backend.yml`** `[ci]`
68. **Add `.github/workflows/release-android.yml` (signed APK to Releases)** `[ci][release]`
69. **Add `.github/workflows/release-mac.yml` (`.dmg` to Releases)** `[ci][release]`
70. **Add `.github/workflows/deploy-backend.yml` (Coolify webhook)** `[ci][release]`
71. **Configure branch protection on `master`** `[ci][infra]`
72. **Configure all required secrets in repo settings** `[ci][infra][manual]`

## Phase 9 · Testing infrastructure

73. **Add Testcontainers dependency + base test class for Supabase integration tests** `[test]`
74. **Add JaCoCo coverage reporting** `[test][android]`
75. **Add Xcode coverage reporting** `[test][mac]`
76. **Implement `scripts/smoke.sh` end-to-end smoke test** `[test]`
77. **Document testing conventions in `docs/testing.md`** `[docs][test]`

## Phase 10 · Documentation

78. **Author `docs/idea.md`** `[docs]` — done in the bootstrap PR
79. **Author `docs/infrastructure.md`** `[docs]` — done in the bootstrap PR
80. **Author `docs/security.md`** `[docs]` — done in the bootstrap PR
81. **Author `docs/ci-cd.md`** `[docs]` — done in the bootstrap PR
82. **Author `docs/testing.md`** `[docs]` — done in the bootstrap PR
83. **Update `AGENTS.md` to reflect new repo structure (post-Pro-removal)** `[docs]` — `CLAUDE.md` is now only a pointer to it.
84. **Update root `README.md` to reflect the fork's identity** `[docs]`
