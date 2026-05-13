# ccr-shared-workflows

Reusable GitHub Actions workflows shared across all CCR service repositories.

Centralised here so security/CI logic is defined once and consumed by every service
(booking, catalog, payment, frontends, etc.) and every environment (dev, uat, prod).

## Contents

| Reusable workflow | Path | Purpose |
| --- | --- | --- |
| **SonarCloud scan** | `.github/workflows/sonarqube-scan.yml` | Static analysis + coverage upload to SonarCloud |
| **Dependency-Track scan** | `.github/workflows/dependency-track-scan.yml` | Generates a CycloneDX SBOM, uploads it to Dependency-Track, and optionally enforces severity thresholds |

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
| `DTRACK_API_URL` | dependency-track-scan | Base URL of the Dependency-Track **API server** (not the UI) |
| `DTRACK_API_KEY` | dependency-track-scan | DT team API key with `BOM_UPLOAD`, `PROJECT_CREATION_UPLOAD`, `VIEW_PORTFOLIO`, `VIEW_VULNERABILITY` |

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
```

## Inputs

### `sonarqube-scan.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `solution-name` | yes | — | Filename of the `.sln` (e.g. `BookingService.sln`) |
| `working-directory` | no | `.` | Directory where the solution lives, relative to repo root |
| `sonar-project-key` | yes | — | SonarCloud project key |
| `sonar-organization` | yes | — | SonarCloud organization key |
| `sonar-project-name` | yes | — | Display name shown in SonarCloud UI |
| `sonar-host-url` | no | `https://sonarcloud.io` | Override only for self-hosted SonarQube |
| `dotnet-version` | no | `10.x` | .NET SDK to install |

### `dependency-track-scan.yml`

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `solution-name` | yes | — | Filename of the `.sln` |
| `working-directory` | no | `.` | Directory where the solution lives |
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
