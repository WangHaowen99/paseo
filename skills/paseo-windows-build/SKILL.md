---
name: paseo-windows-build
description: Build and publish Paseo Windows desktop artifacts from the Paseo monorepo. Use when asked to compile a Windows exe/zip, produce x64-only desktop builds, create or update GitHub release assets, or document/repeat the Windows release packaging flow.
---

# Paseo Windows Build

Use this workflow from the Paseo monorepo root on Windows PowerShell.

## Preconditions

- Work from the repository root, for example `C:\claude\paseo-src\paseo-main`.
- Confirm Node dependencies are already installed. If not, run the repo's normal install command before building.
- Confirm GitHub CLI auth before upload:

```powershell
& 'C:\Program Files\GitHub CLI\gh.exe' auth status -h github.com
```

If auth fails, ask the user to re-authenticate or provide a valid token through an approved channel. Do not print tokens.

## X64 Builder Config

Create a temporary `packages/desktop/electron-builder.x64.yml` from the normal desktop builder config, with:

- `directories.output` set to a unique release output directory.
- Windows target arch restricted to `x64` for both `nsis` and `zip`.
- `artifactName` set to `"Paseo-Setup-${version}-${arch}.${ext}"`.

Keep this file out of source commits unless the user explicitly wants a permanent x64 config. Delete it after packaging.

## Build Commands

Run this from the repo root. Change `$outputName` when rebuilding the same version for a different release.

```powershell
$ErrorActionPreference = 'Stop'
$outputName = 'release-windows-x64'
$releaseDir = Join-Path (Get-Location) "packages\desktop\$outputName"

if (Test-Path $releaseDir) {
  $resolved = (Resolve-Path $releaseDir).Path
  $expected = (Join-Path (Get-Location) "packages\desktop\$outputName")
  if ($resolved -ne $expected) { throw "Unexpected release dir: $resolved" }
  Remove-Item -LiteralPath $resolved -Recurse -Force
}

npm run build:terminal-webview --workspace=@getpaseo/app
npm run typecheck --workspace=@getpaseo/app
npm run build:app-deps
npm run build:server-deps
npm run build --workspace=@getpaseo/server

Push-Location packages\app
$env:PASEO_WEB_PLATFORM = 'electron'
npx expo export --platform web
Pop-Location

npm run build:main --workspace=@getpaseo/desktop

Push-Location packages\desktop
$env:CSC_IDENTITY_AUTO_DISCOVERY = 'false'
npx electron-builder --config electron-builder.x64.yml --win
Pop-Location

Get-ChildItem "packages\desktop\$outputName" | Select-Object Name,Length,LastWriteTime
```

Expected x64 artifacts:

- `Paseo-Setup-<version>-x64.exe`
- `Paseo-Setup-<version>-x64.zip`
- `Paseo-Setup-<version>-x64.exe.blockmap`
- `latest.yml`
- `win-unpacked/` for local inspection

Electron Builder may log npm dependency metadata warnings such as `ELSPROBLEMS` for workspace packages. Treat them as warnings only if the exe, zip, blockmap, and `latest.yml` are produced and the command exits successfully.

## GitHub Release Upload

Create a release with the four redistributable files only. Do not upload `builder-debug.yml` or `win-unpacked`.

```powershell
$repo = 'OWNER/REPO'
$tag = 'windows-x64-0.1.87-YYYYMMDD'
$target = '<commit-sha>'
$outputName = 'release-windows-x64'
$gh = 'C:\Program Files\GitHub CLI\gh.exe'

$assets = @(
  "packages/desktop/$outputName/Paseo-Setup-0.1.87-x64.exe",
  "packages/desktop/$outputName/Paseo-Setup-0.1.87-x64.zip",
  "packages/desktop/$outputName/Paseo-Setup-0.1.87-x64.exe.blockmap",
  "packages/desktop/$outputName/latest.yml"
)

$notes = @'
Windows x64 build.
'@

& $gh release create $tag @assets --repo $repo --target $target --title 'Windows x64 build' --notes $notes --latest
& $gh release view $tag --repo $repo --json url,tagName,targetCommitish,assets
```

If `gh release view --json isLatest` fails, omit `isLatest`; older GitHub CLI builds do not support that field.

## Source Commit Checklist

Before uploading a release, make sure the source corresponding to the build is on the target branch/tag:

- Commit source edits and generated files such as `packages/app/src/terminal/webview/terminal-emulator-webview-html.ts`.
- Do not commit local runtime config such as `~/.paseo/config.json`, app data, secrets, or release build directories.
- Do not commit temporary `electron-builder.x64.yml` unless requested.
- Verify remote source with raw GitHub URLs or `git ls-remote` when local `.git` state is unreliable.

## Cleanup

After packaging:

- Delete the temporary x64 builder config.
- Keep build artifacts if the user may need local inspection.
- Remove temporary API JSON files under `.tmp` only after checking the resolved path is inside the intended `.tmp` directory.
