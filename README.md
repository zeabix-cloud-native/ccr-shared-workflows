# ccr-shared-workflows

Reusable GitHub Actions workflows shared across all CCR service repositories.

Centralised here so security/CI logic is defined once and consumed by every service
(booking, catalog, payment, frontends, etc.) and every environment (dev, uat, prod).

## Contents

| Reusable workflow | Path | Purpose |
| --- | --- | --- |
| **SonarCloud scan** | `.github/workflows/sonarqube-scan.yml` | Static analysis + coverage upload to SonarCloud |
| **Dependency-Track scan** | `.github/workflows/dependency-track-scan.yml` | Generates a CycloneDX SBOM, uploads it to Dependency-Track, and optionally enforces severity thresholds |
| **Dependency-Track (npm)** | `.github/workflows/dependency-track-npm-scan.yml` | Same as above for Node/npm apps via `@cyclonedx/cyclonedx-npm` |
| **.NET build** | `.github/workflows/dotnet-build.yml` | Restore → test → publish → upload artifact for any .NET solution |
| **Next.js standalone build** | `.github/workflows/nextjs-standalone-build.yml` | Node.js → `npm run build` → Next standalone tarball (`deploy.tar.gz`) |
| **Azure Web App deploy** | `.github/workflows/azure-webapp-deploy.yml` | Download artifact → optional tarball extract → deploy to Azure App Service |

## Setup (one-time)

1. Push this folder to a new GitHub repository, e.g. `zeabix-cloud-native/ccr-shared-workflows`.
2. Tag a stable release the first time you want consumers to pin: `git tag v1 && git push --tags`.
3. In each consuming service repository:
   - `Settings → Actions → General → Access` → allow Actions in this repo to access `zeabix-cloud-native/ccr-shared-workflows` (for private orgs).
   - Add the required org-/repo-level secrets (see each reusable workflow for the exact list).

## Versioning

- Tag the repo with `v1`, `v2`, … on breaking changes.
- Consumers should pin: `uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/sonarqube-scan.yml@v1`.
- During development you can use `@main` for "always latest"; switch to a pinned tag before going to UAT/prod.

## Required secrets in each consumer

| Secret | Used by | Notes |
| --- | --- | --- |
| `SONAR_TOKEN` | sonarqube-scan | SonarCloud user/analysis token |
| `DTRACK_API_URL` | dependency-track-scan, dependency-track-npm-scan | Base URL of the Dependency-Track **API server** (not the UI) |
| `DTRACK_API_KEY` | dependency-track-scan, dependency-track-npm-scan | DT team API key with `BOM_UPLOAD`, `PROJECT_CREATION_UPLOAD`, `VIEW_PORTFOLIO`, `VIEW_VULNERABILITY` |

## Calling these workflows

See `examples/cicd_dev.yml` in any CCR service repo, or the snippet below:

```yaml
jobs:
  sonarqube:
    uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/sonarqube-scan.yml@v1
    with:
      solution-name: BookingService.sln
      working-directory: .
      sonar-project-key: zeabix-cloud-native_ccr-booking-service
      sonar-organization: zeabix-cloud-native
      sonar-project-name: CCR Booking Service
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  dependency-track:
    uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/dependency-track-scan.yml@v1
    with:
      solution-name: BookingService.sln
      working-directory: .
      project-name: CCR-Booking-Service
      project-version: v1.0.0
      # Optional severity gate (defaults are all -1 = no gating)
      # max-critical: 0
      # max-high: 0
    secrets:
      DTRACK_API_URL: ${{ secrets.DTRACK_API_URL }}
      DTRACK_API_KEY: ${{ secrets.DTRACK_API_KEY }}

  build:
    uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/dotnet-build.yml@v1
    with:
      solution-name: BookingService.sln
      project-name: BookingService.csproj
      cache-key-prefix: nuget-booking

  deploy:
    needs: build
    uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/azure-webapp-deploy.yml@v1
    with:
      app-name: app-booking-wccr-nonprod-sea-01
      slot-name: Production
    secrets:
      AZURE_PUBLISH_PROFILE: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_DC4831A6E0034A55BFA7F2991766196D }}

  # Next.js example (tarball artifact):
  # deploy:
  #   needs: build
  #   uses: zeabix-cloud-native/ccr-shared-workflows/.github/workflows/azure-webapp-deploy.yml@v1
  #   with:
  #     app-name: app-fe-user-wccr-nonprod-sea-01
  #     artifact-name: node-app
  #     extract-tarball: true
  #     tarball-filename: deploy.tar.gz
  #     verify-next-standalone: true
  #   secrets:
  #     AZURE_PUBLISH_PROFILE: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_XXX }}

```

## Inputs

### `sonarqube-scan.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `solution-name` | yes | — | Filename of the `.sln` (e.g. `BookingService.sln`) |
| `working-directory` | no | `.` | Directory where the solution lives when autodetect is `false` |
| `autodetect-working-directory` | no | `false` | If `true`, use repo root when the `.sln` exists there, else `monorepo-package-path` |
| `monorepo-package-path` | no | `''` | Second path to try when autodetect is `true` (e.g. `application_service/ccr-payment-service`) |
| `sonar-project-key` | yes | — | SonarCloud project key |
| `sonar-organization` | yes | — | SonarCloud organization key |
| `sonar-project-name` | yes | — | Display name shown in SonarCloud UI |
| `sonar-host-url` | no | `https://sonarcloud.io` | Override only for self-hosted SonarQube |
| `dotnet-version` | no | `10.x` | .NET SDK to install |

### `dependency-track-scan.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `solution-name` | yes | — | Filename of the `.sln` |
| `working-directory` | no | `.` | Directory where the solution lives when autodetect is `false` |
| `autodetect-working-directory` | no | `false` | If `true`, resolve `.sln` at repo root or under `monorepo-package-path` |
| `monorepo-package-path` | no | `''` | Fallback path when autodetect is `true` |
| `project-name` | yes | — | Dependency-Track project name (e.g. `CCR-Booking-Service-DEV`) |
| `project-version` | no | `v1.0.0` | DT project version label |
| `is-latest` | no | `true` | Mark this BOM as the project's "latest" |
| `auto-create` | no | `true` | Create the DT project if it doesn't exist |
| `cyclonedx-spec-version` | no | `1.6` | CycloneDX spec to emit (DT 4.13 supports up to 1.6) |
| `max-critical` | no | `-1` | Fail if Critical count > N. `-1` = ignore |
| `max-high` | no | `-1` | Fail if High count > N. `-1` = ignore |
| `max-medium` | no | `-1` | Fail if Medium count > N. `-1` = ignore |
| `max-low` | no | `-1` | Fail if Low count > N. `-1` = ignore |
| `polling-timeout-seconds` | no | `180` | Max time to wait for DT to finish processing the BOM (only when gating is on) |
| `dotnet-version` | no | `10.x` | .NET SDK to install |

### `dependency-track-npm-scan.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `working-directory` | no | `.` | Directory with `package.json` when autodetect is `false` |
| `autodetect-working-directory` | no | `false` | If `true`, resolve `package.json` at repo root or under `monorepo-package-path` |
| `monorepo-package-path` | no | `application_service/ccr-frontend-user` | Fallback app path when autodetecting |
| `node-version` | no | `22.x` | Node.js version |
| `npm-install-mode` | no | `ci` | `ci` (needs lockfile) or `install` |
| `cache-key-prefix` | no | `dtrack-npm` | npm cache key prefix |
| `project-name` | yes | — | Dependency-Track project name (e.g. `CCR-FE-User-UAT`) |
| `project-version` | no | `v1.0.0` | DT project version label |
| `is-latest` | no | `true` | Mark this BOM as the project's "latest" |
| `auto-create` | no | `true` | Create the DT project if it doesn't exist |
| `cyclonedx-spec-version` | no | `1.6` | CycloneDX spec (DT 4.13 supports up to 1.6) |
| `cyclonedx-npm-version` | no | `2` | Semver range for `@cyclonedx/cyclonedx-npm` (e.g. `2` or `2.7.0`) |
| `package-lock-only` | no | `true` | Lockfile-based SBOM (much smaller upload; avoids common nginx **413**). Skips `npm install` / `npm ci`. Adds `--ignore-npm-errors` with this mode. Requires `package-lock.json` or `npm-shrinkwrap.json`. Set `false` for full `node_modules` SBOM (may need a larger API upload limit). |
| `omit-dev-dependencies` | no | `false` | If `true`, adds `--omit dev` for a smaller production-oriented BOM |
| `max-critical` | no | `-1` | Fail if Critical count > N. `-1` = ignore |
| `max-high` | no | `-1` | Fail if High count > N. `-1` = ignore |
| `max-medium` | no | `-1` | Fail if Medium count > N. `-1` = ignore |
| `max-low` | no | `-1` | Fail if Low count > N. `-1` = ignore |
| `polling-timeout-seconds` | no | `180` | Max wait for DT processing when gating is on |

### `dotnet-build.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `solution-name` | yes | — | `.sln` filename used for restore and test |
| `project-name` | yes | — | `.csproj` filename used for publish |
| `working-directory` | no | `.` | Directory containing the solution and project files when autodetect is `false` |
| `autodetect-working-directory` | no | `false` | If `true`, resolve `.sln` at repo root or under `monorepo-package-path` |
| `monorepo-package-path` | no | `''` | Fallback path when autodetect is `true` |
| `configuration` | no | `Release` | Build configuration (`Debug` or `Release`) |
| `dotnet-version` | no | `10.x` | .NET SDK to install |
| `artifact-name` | no | `.net-app` | Name of the uploaded artifact consumed by the deploy job |
| `run-tests` | no | `true` | Set to `false` to skip `dotnet test` |
| `cache-key-prefix` | no | `nuget` | NuGet cache key prefix — use the service name (e.g. `nuget-booking`) to keep caches isolated per service |

### `nextjs-standalone-build.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `working-directory` | no | `.` | Directory containing `package.json` when `autodetect-working-directory` is `false` |
| `autodetect-working-directory` | no | `false` | If `true`, use root `package.json` or `monorepo-package-path` |
| `monorepo-package-path` | no | `application_service/ccr-frontend-backoffice` | Fallback app path when autodetecting |
| `node-version` | no | `22.x` | Node.js version |
| `npm-install-mode` | no | `install` | `install` or `ci` |
| `build-command` | no | `npm run build` | Command that produces `.next/standalone` |
| `copy-public-optional` | no | `false` | If `true`, `cp public` uses `\|\| true` (missing `public` folder) |
| `artifact-name` | no | `node-app` | Uploaded artifact name |
| `tarball-filename` | no | `deploy.tar.gz` | Tarball written in the resolved working directory |
| `cache-key-prefix` | no | `nextjs` | Prefix for the npm cache key |
| `github-environment` | no | `''` | When non-empty, the build job uses `environment: <name>` on the **caller** repo and sets build `env` from `vars` / `secrets` (Vite-style). When empty, use `next-public-*` inputs / optional `secrets` below. |
| `next-public-base-url` | no | `''` | `NEXT_PUBLIC_BASE_URL` during `npm run build` |
| `next-public-omise-public-key` | no | `''` | `NEXT_PUBLIC_OMISE_PUBLIC_KEY` during build |
| `next-public-turnstile-site-key` | no | `''` | `NEXT_PUBLIC_TURNSTILE_SITE_KEY` during build |

Optional **secrets** (when `github-environment` is empty): `NEXT_PUBLIC_*` override the matching `with` input when non-empty. When `github-environment` is set, the build job reads those names from the Environment (and repo) via `vars` / `secrets` on the Build step instead.

### `azure-webapp-deploy.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `app-name` | yes | — | Azure Web App name (e.g. `app-booking-wccr-nonprod-sea-01`) |
| `slot-name` | no | `Production` | Deployment slot |
| `artifact-name` | no | `.net-app` | Name of the artifact uploaded by the build job |
| `package-path` | no | `.` | Path passed to `azure/webapps-deploy` |
| `extract-tarball` | no | `false` | If `true`, run `tar -xzf` on `tarball-filename` after download (Next.js flow) |
| `tarball-filename` | no | `deploy.tar.gz` | Tarball file inside the artifact |
| `verify-next-standalone` | no | `false` | If `true` (and extract is `true`), list `.next/` and `cat .next/BUILD_ID` before deploy |
| `clean` | no | `false` | If `true`, pass `clean` to `azure/webapps-deploy` (remove extra files at destination) |

Secret: **`AZURE_PUBLISH_PROFILE`** — pass the publish-profile XML from a repo secret, e.g.:

```yaml
secrets:
  AZURE_PUBLISH_PROFILE: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_XXXXXXXXXX }}
```
