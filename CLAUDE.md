# CLAUDE.md

This file provides guidance for AI assistants (Claude Code) working in this repository.

## What this repo is

This is `actions/runner-images` — the source that builds the VM images used by
GitHub-hosted Actions runners and Microsoft-hosted Azure Pipelines agents. It does not
contain application code; it contains **Packer templates, shell/PowerShell provisioning
scripts, toolset configs, and Pester validation tests** that together define what software
ends up on each OS image (Ubuntu, Windows, macOS).

There is no "build and run the app" workflow here — the deliverable is a VM image built
by Packer in Azure (Linux/Windows) or Anka (macOS), which takes significant time and cloud
resources. Most day-to-day changes are additive: install/update/remove one tool and prove
it works via a validation script.

## Repository layout

```text
images/
  ubuntu/            Ubuntu 22.04/24.04/26.04 (x64 + arm64)
  ubuntu-slim/        Minimal Ubuntu image
  windows/            Windows Server 2022/2025, Windows 11 arm64
  macos/              macOS 14/15/26 (x64 + arm64) — built via Anka, CI is not open to external PRs
    <platform>/
      templates/       Packer *.pkr.hcl files (source/build/locals/variable per OS)
      toolsets/         toolset-<version>.json — declarative tool/version config, validated by schemas/toolset-schema.json
      scripts/
        build/           Install scripts, one (or a few) per tool — run during image generation
        tests/           Pester (*.Tests.ps1) validation scripts, run at the end of the matching build script
        helpers/         Shared helper functions/modules sourced by build & test scripts
        docs-gen/        Generate-SoftwareReport.ps1 + SoftwareReport.*.psm1 — generates the per-image *-Readme.md
      assets/            Static files copied onto the image (incl. post-gen scripts)
images.CI/              CI-only helper scripts (build-image.ps1, cleanup.ps1, create-release.ps1, shebang-linter.ps1)
helpers/                 Top-level PowerShell modules for manual image generation & Azure resource management,
                          plus helpers/software-report-base (shared software-report engine + its own Pester tests)
docs/                    create-image-and-azure-resources.md (full Packer/Azure walkthrough), dotnet-ubuntu.md
schemas/                 toolset-schema.json — JSON schema enforced on all toolset-*.json files
.github/workflows/       CI: linting, per-OS build triggers, release automation, schema validation
```

## Core workflow: adding/updating a tool

This is the most common task in this repo. For each platform:

1. **Install script** — add/edit a script in `images/<platform>/scripts/build/`. Use an
   existing script (e.g. `images/ubuntu/scripts/build/install-github-cli.sh`) as a template.
   - Source only the helpers you actually use (Ubuntu: `$HELPER_SCRIPTS/install.sh`, `os.sh`,
     `etc-environment.sh`; macOS: `~/utils/utils.sh`).
   - Prefer existing helper functions over ad hoc code: Windows has
     `Install-ChocoPackage`, `Install-Binary`, `Install-VSIXFromFile`, `Install-VSIXFromUrl`,
     `Invoke-DownloadWithRetry`, `Test-IsWin22`, `Test-IsWin25` (see
     `images/windows/scripts/helpers/ImageHelpers.psm1`); Ubuntu has helpers in
     `images/ubuntu/scripts/helpers/install.sh`.
   - Verify checksums/signatures for downloaded binaries where the upstream project
     publishes them (supply-chain security — see the checksum pattern in
     `install-github-cli.sh`).
2. **Validation script** — add a Pester test alongside the install (Windows/macOS use
   Pester v5 in `scripts/tests/*.Tests.ps1`, or add to the shared `Tools.Tests.ps1` for
   simple checks; Ubuntu validation typically lives inline in the build script and must
   `exit 1` on failure). Call `Invoke-PesterTests -TestFile <file> [-TestName <describe>]`
   at the end of the install script so tests run automatically as part of image generation.
   Validation must not mutate image state — it only checks what's already installed.
3. **Toolset config** — if the tool supports multiple installable versions, list them in
   the relevant `images/<platform>/toolsets/toolset-*.json` instead of hardcoding a version
   in the script. This file is validated against `schemas/toolset-schema.json` in CI
   (`validate-json-schema.yml`); pinned/non-semver entries need a `pinnedDetails` block.
4. **Software report** — update the report generator so the tool shows up in the
   generated per-image README (e.g. `Ubuntu2404-Readme.md`, `Windows2022-Readme.md`):
   `images/<platform>/scripts/docs-gen/Generate-SoftwareReport.ps1` and its
   `SoftwareReport.*.psm1` modules (uses MarkdownPS). **Do not hand-edit the generated
   `*-Readme.md` files** — they're regenerated from these modules during the build.
5. Keep naming consistent across the install script, test, toolset entry, and report —
   this is checked informally in review, not by CI.

Before adding a brand-new tool (not updating an existing one), an approved GitHub issue is
expected per `CONTRIBUTING.md` — flag this to the user if it looks like a net-new addition
rather than an update.

## Script conventions

Full style guide: `CONTRIBUTING.md`. Highlights an assistant should follow automatically:

**Bash** (Ubuntu/Ubuntu-slim, and macOS shell helpers)
- Shebang: `#!/bin/bash -e`
- File header block:
  ```bash
  ################################################################################
  ##  File:  <filename>
  ##  Desc:  <short description>
  ################################################################################
  ```
- 4-space indentation; `then`/`do` on their own line; one-line form for short conditionals.
- Use `[[ ... ]]` not `[ ... ]`; use `$()` not backticks; prefer long option flags except
  common short forms (`tar -xzf`, `apt-get -yqq`, `curl -sSLf`, `wget -qO-`).
- lowercase_snake_case variables, UPPERCASE constants.

**PowerShell** (Windows scripts, and cross-platform helpers/tests)
- Same file header block as Bash.
- `Verb-Noun` PascalCase function names, camelCase variables, UPPERCASE constants.
- 1TBS brace style, 4-space indent, avoid aliases, use splatting for long parameter lists.
- Follow the [PowerShell Practice and Style](https://github.com/PoshCode/PowerShellPracticeAndStyle) guide.
- `.vscode/settings.json` encodes the exact formatter rules (OTBS preset, whitespace around
  pipes/braces/operators) — match it rather than reformatting by hand.

**General**
- Every file ends with a newline; no trailing whitespace; `.gitattributes` forces `lf` line
  endings for all text files — don't introduce CRLF.
- `toolset-*.json` files must validate against `schemas/toolset-schema.json`
  (`.vscode/settings.json` wires this up for editor validation too).

## CI

- `linter.yml` — runs `github/super-linter/slim` over changed JSON/Markdown/Shell files on
  every PR to `main` (excludes generated `*-Readme.md`), plus `images.CI/shebang-linter.ps1`.
- `validate-json-schema.yml` — validates all `toolset-*.json` against `schemas/toolset-schema.json`
  via `helpers/CheckJsonSchema.ps1`.
- `check-pinned-versions.yml` — flags outdated pinned tool versions via
  `helpers/CheckOutdatedVersionPinning.ps1`.
- `powershell-tests.yml` — runs Pester tests for `helpers/software-report-base/**` only.
- Per-OS workflows (`ubuntu2204.yml`, `ubuntu2404.yml`, `ubuntu2604.yml`, `windows2022.yml`,
  `windows2025.yml`, `windows2025-vs2026.yml`) are triggered by applying a `CI <os>` label to
  a PR and delegate to `trigger-ubuntu-win-build.yml`, which kicks off an actual Packer build.
  **This is expensive and slow** — don't suggest applying these labels casually; they're for
  maintainers validating real image builds. macOS has no public CI; PRs touching
  `images/macos/**` cannot be validated by external contributors' CI runs.
- `codeql-analysis.yml`, `create_sbom_report.yml`, `docker-images.yml`,
  `create_github_release.yml` / `update_github_release.yml` / `merge_pull_request.yml` handle
  security scanning and the release process — not typically touched by tool-addition PRs.

## Manual/local testing

There is no local "run the image." To validate a change end-to-end you either:
- Trust the install script's own inline checks / Pester assertions (fast, no cloud needed —
  this is the practical option in an assistant session), or
- Actually build an image in Azure via `helpers/GenerateResourcesAndImage.ps1`
  (see `docs/create-image-and-azure-resources.md` for the full Packer/Azure walkthrough,
  required variables, and Service Principal setup) — requires real Azure credentials/cost,
  do not attempt this without explicit user direction.

`helpers/software-report-base` has its own Pester test suite
(`helpers/software-report-base/tests`) runnable directly with
`Invoke-Pester -Output Detailed "helpers/software-report-base/tests"` — this one *can* be
run locally without cloud resources if `pwsh` + Pester are available.

## Key docs to consult before larger changes

- `README.md` — supported images, label scheme, version/deprecation policy, package-manager
  usage per OS, preinstallation policy (what gets accepted at all).
- `CONTRIBUTING.md` — full contribution + style guide (source of truth for conventions above).
- `docs/create-image-and-azure-resources.md` — Packer/Azure build mechanics.
- `docs/dotnet-ubuntu.md` — Ubuntu-specific .NET version details.

## Constraints an assistant should respect

- Don't hand-edit generated `*-Readme.md` files under `images/<platform>/` — edit the
  `docs-gen` report generator instead.
- Don't add a brand-new tool without noting that repo policy expects prior issue approval.
- Don't suggest triggering the labeled CI build workflows or real Packer/Azure builds
  unless the user explicitly wants to kick off an actual image build.
- macOS PRs can't be validated by this repo's CI for external contributions — say so if
  relevant.
- Keep changes scoped to one platform/tool at a time where possible, matching this repo's
  preference for small, focused PRs (`CONTRIBUTING.md`).
