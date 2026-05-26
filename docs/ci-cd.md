# CI/CD pipelines

All pipelines run on **GitHub Actions**. The existing CircleCI config is left as-is for now; new work flows through GHA.

## 1. Workflows overview

| Workflow file | Trigger | What it does |
|---|---|---|
| `.github/workflows/ci-kmp.yml` | PR + push to `master` | Builds composeApp, runs `:composeApp:check`, `spotlessCheck`, uploads JaCoCo report |
| `.github/workflows/ci-mac.yml` | PR + push to `master` (paths: `macApp/**`, `composeApp/**`) | Builds the Mac app on a `macos-14` runner, runs Swift tests |
| `.github/workflows/ci-backend.yml` | PR + push to `master` (paths: `backend/**`) | Validates SQL migrations against an ephemeral Postgres, lints `docker-compose.yml` |
| `.github/workflows/release-android.yml` | Tag `v*.*.*-android` | Builds release APK, signs with keystore, uploads to GitHub Releases |
| `.github/workflows/release-mac.yml` | Tag `v*.*.*-mac` | Builds release Mac `.app`, packages as `.dmg`, uploads to GitHub Releases |
| `.github/workflows/deploy-backend.yml` | Tag `v*.*.*-backend` or manual `workflow_dispatch` | Hits Coolify deploy webhook to redeploy the Supabase service (which pulls latest migrations from this repo) |

## 2. Secrets needed

In repo Settings → Secrets and variables → Actions:

- `ANDROID_KEYSTORE_BASE64` — keystore file, base64-encoded
- `ANDROID_KEYSTORE_PASSWORD`
- `ANDROID_KEY_ALIAS`
- `ANDROID_KEY_PASSWORD`
- `MAC_SIGNING_IDENTITY` (only if user has Apple Developer Program — defaults to ad-hoc signing)
- `MAC_NOTARY_API_KEY_ID` / `MAC_NOTARY_API_KEY` / `MAC_NOTARY_ISSUER_ID` (notarization, optional)
- `COOLIFY_DEPLOY_WEBHOOK` — full webhook URL from Coolify → Supabase service → Webhooks → Deploy
- `COOLIFY_API_TOKEN` — bearer token from Coolify → Profile → API Tokens (read+deploy scope is enough)

## 3. KMP CI workflow sketch

```yaml
# .github/workflows/ci-kmp.yml
name: CI · KMP
on:
  pull_request:
  push:
    branches: [master]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v3
      - run: ./gradlew :composeApp:check spotlessCheck
      - uses: actions/upload-artifact@v4
        with:
          name: jacoco-report
          path: composeApp/build/reports/jacoco/
```

## 4. Mac CI workflow sketch

```yaml
# .github/workflows/ci-mac.yml
name: CI · Mac
on:
  pull_request:
    paths: [macApp/**, composeApp/**]
  push:
    branches: [master]
    paths: [macApp/**, composeApp/**]
jobs:
  build-test-mac:
    runs-on: macos-14
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - run: ./gradlew :composeApp:linkDebugFrameworkMacosArm64
      - run: xcodebuild test -project macApp/macApp.xcodeproj -scheme macApp -destination 'platform=macOS'
```

## 5. Release workflows

Tag-driven. Bumping `gradle.properties` `appVersionName` + `appVersionCode`, committing, then pushing a tag like `v0.1.0-android` triggers the APK build.

Mac `.dmg` release: tag `v0.1.0-mac`. Builds `.app`, code-signs (ad-hoc by default; Developer ID if `MAC_SIGNING_IDENTITY` secret is set), runs `create-dmg`, uploads.

## 6. Backend deploy workflow

```yaml
# .github/workflows/deploy-backend.yml
name: Deploy · Backend
on:
  push:
    tags: ['v*.*.*-backend']
  workflow_dispatch:
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Coolify redeploy
        run: |
          curl -fsSL -X POST "${{ secrets.COOLIFY_DEPLOY_WEBHOOK }}" \
            -H "Authorization: Bearer ${{ secrets.COOLIFY_API_TOKEN }}"
```

The Supabase service in Coolify is wired with a **post-deployment command** that runs `psql -v ON_ERROR_STOP=1 -f /migrations/*.sql` against the local Postgres. Coolify mounts the repo's `backend/supabase/migrations/` directory as `/migrations` via its persistent storage tab. So a webhook tick → Coolify pulls the latest repo state → redeploys Supabase → runs migrations in order.

## 7. Branch protection

- `master` requires green `ci-kmp`, `ci-mac` (if changed), `ci-backend` (if changed) before merge.
- Linear history (rebase merges only).
- No force-push to `master`.
