# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A community fork of Microsoft's experimental `Microsoft.AspNetCore.Authentication.DeviceBoundSessions` package
(Device Bound Session Credentials, DBSC), published to nuget.org as
`ThorstenSauter.AspNetCore.Authentication.DeviceBoundSessions`. Microsoft shipped it once (`0.11.0-rc.1`, .NET 11 RC1)
and removed it before RC2 ([dotnet/aspnetcore#69462](https://github.com/dotnet/aspnetcore/pull/69462)), possibly
returning in .NET 12 ([#66478](https://github.com/dotnet/aspnetcore/issues/66478)). This fork exists so
[ewlv-website](https://github.com/ThorstenSauter/ewlv-website) can keep its device-bound sessions in production. Once
Microsoft ships an official package, consumers migrate and this package is deprecated on nuget.org.

Source origin: `dotnet/dotnet@3551975be08744f0418857c5bed8ab1545c5dd47`,
`src/aspnetcore/src/Security/Authentication/DeviceBoundSessions/src` and `.../test/DeviceBoundSessions`.

## Commands

```bash
dotnet build
dotnet test                       # xUnit v3 on Microsoft.Testing.Platform (runner set in global.json)
dotnet pack src/ThorstenSauter.AspNetCore.Authentication.DeviceBoundSessions -c Release -o artifacts/pkg
```

## Rules for changes

- **Stay close to upstream.** Keep the public API shape (`AddDeviceBoundSession`, `DeviceBoundSessionOptions`,
  `WriteDeviceBoundSessionRegistration`, scheme names `{source}.Dbsc*`) so migrating to an official package stays a
  mechanical swap. When fixing a bug that has an upstream issue, link it in a code comment and in the README's
  "Differences" list.
- **Keep the .NET Foundation MIT headers** on imported files. New files may omit them.
- **Every behaviour change gets a test.** Proof JWTs for tests come from `DbscProofKey` (ES256/RS256 keys, tampering,
  malformed UTF-8); reuse it instead of hand-building tokens.
- Imported upstream tests are kept verbatim where possible; `xUnit1051` is suppressed in the test project for them.
  New tests should pass `TestContext.Current.CancellationToken` anyway.
- `AddDeviceBoundSession` deliberately stays in the `Microsoft.Extensions.DependencyInjection` namespace, as upstream.
- This is security-critical code (proof JWT validation, challenge protection, cookie scoping). Prefer returning
  `null`/400 over throwing on malformed client input; a client must never be able to provoke a 500.
- Code style: `.editorconfig`, copied from ewlv-website. Warnings are errors (`Directory.Build.props`). CI runs
  `dotnet format --verify-no-changes`; run `dotnet format` before committing.
- **Public API baseline:** `PublicAPI.Shipped.txt` / `PublicAPI.Unshipped.txt` (Microsoft.CodeAnalysis.PublicApiAnalyzers).
  Any public API change fails the build (RS0016/RS0017) until recorded: new members go into `Unshipped`; at release
  time move them to `Shipped`. Removing or changing a `Shipped` entry is a breaking change: bump the major version
  (or minor while 0.x) and label the PR `breaking`.
- `EnablePackageValidation` checks the package on pack. After the first stable release, set
  `PackageValidationBaselineVersion` to the previous version so API breaks against it fail the pack too.

## Packages and dependencies

- Central package management: versions live in `Directory.Packages.props`, `PackageReference`s carry no version.
- **Lock files:** `packages.lock.json` per project is committed; CI (`CI=true`) restores in locked mode and fails if a
  lock file is stale. After changing a package, run `dotnet restore` and commit the updated lock files.
- **SBOM:** [Sbom](https://github.com/SimonCropp/Sbom) embeds an SPDX 3.0.1 SBOM at
  `_manifest/spdx_3.0/manifest.spdx.json`, built from the lock file. `pack-and-publish` attests it (with the DLL) as a
  provenance subject; GitHub's SBOM-mode attestation only accepts SPDX 2.x/CycloneDX, so that mode isn't used.
- `NuGet.config` clears all sources and maps everything to nuget.org; the package must never depend on a private
  feed.
- `System.Formats.Cbor` and the Data Protection packages come from the shared framework (`net11.0` +
  `FrameworkReference Microsoft.AspNetCore.App`); referencing them as packages fails restore (NU1510).
- Renovate (`.github/renovate.json`, extending `local>ThorstenSauter/renovate-config`) updates packages, the SDK in
  `global.json`, and the SHA-pinned actions.

## CI and releases

- `.github/workflows/ci.yml`: `ThorstenSauter/dotnet-actions/build-and-test` on Ubuntu and Windows, a format check,
  and a pack-only run of `pack-and-publish`. The `main` ruleset requires only the `CI passed` gate job: add every new
  job to its `needs:`. The ruleset also requires a squash-merged PR, signed commits and resolved review threads.
- `.github/workflows/release.yml`: on a published GitHub release, tests again, then `pack-and-publish` pushes to
  nuget.org via trusted publishing (OIDC, `id-token: write`, `nuget` environment, `NUGET_USER` environment variable).
  No API key is stored anywhere. The `nuget` environment only deploys from `v*` tags, with no admin bypass, and the
  nuget.org trusted publishing policy is bound to `release.yml` + environment `nuget`.
- The release tag is the package version: `v0.1.0-rc.2` → `0.1.0-rc.2`. Use a prerelease suffix while the target
  .NET version is itself a prerelease.
- Release notes are generated from PR labels (`.github/release.yml`): `enhancement`, `bug`, `documentation`,
  `breaking`, `github_actions`, `dependencies`; `ignore-for-release` drops a PR. Label every PR.
- Pin third-party actions by full commit SHA with a trailing `# vX.Y.Z` comment.
- Commit messages use Conventional Commits.
