# Codex Prebuilt Binaries

This repository hosts prebuilt native binaries for the [Codex Editor](https://codexeditor.app) application.
Binaries are mirrored from their original upstream sources for reliability and published to both npm (`@codex-editor` scope) and GitHub Releases.

## Why This Exists

Codex Editor depends on three native binaries that are **not** bundled with VS Code: SQLite3, FFmpeg, and Git. These are downloaded at runtime with a **three-tier fallback**:

1. **Codex-owned npm packages** (`@codex-editor/sqlite3-darwin-arm64`, etc.) — fastest, most reliable
2. **Codex-owned GitHub Releases** (this repo) — backup if npm is down or blocked
3. **Original upstream source** (TryGhost, @ffmpeg-installer, dugite-native) — last resort

If all three tiers fail, the user sees a warning modal explaining what's missing and how to proceed.

## Tools

| Tool | License | Upstream |
|------|---------|----------|
| SQLite3 (node-sqlite3) | BSD 3-Clause | [TryGhost/node-sqlite3](https://github.com/TryGhost/node-sqlite3) |
| FFmpeg | **LGPL-2.1 only** | [ffmpeg.org](https://ffmpeg.org) / [@ffmpeg-installer](https://www.npmjs.com/package/@ffmpeg-installer/ffmpeg) |
| Git | GPL-2.0 | [desktop/dugite-native](https://github.com/desktop/dugite-native) |

### FFmpeg: LGPL Only — No GPL Builds

FFmpeg can be compiled in two modes:

- **LGPL-2.1** (default) — Safe to redistribute in proprietary applications. Includes all audio codecs Codex needs.
- **GPL-2.0** (opt-in via `--enable-gpl`) — Unlocks extra **video** encoders (libx264, libx265, libxvid) but would require the entire host application to comply with GPL.

**Codex uses LGPL builds exclusively.** Our CI pipeline verifies every FFmpeg binary by running `ffmpeg -version` (Linux) or `strings` (Windows/macOS) and **rejects** any build containing `--enable-gpl`. Upstream packages that ship GPL-compiled binaries are skipped.

This is safe because Codex only uses FFmpeg for **audio** operations:

| Format | Decoder | Encoder | Needs GPL? |
|--------|---------|---------|------------|
| WAV (PCM) | Built-in | Built-in | No |
| MP3 | Built-in | Built-in (libshine) | No |
| M4A / AAC | Built-in | Built-in | No |
| OGG / Vorbis | Built-in | Built-in | No |
| FLAC | Built-in | Built-in | No |
| Opus | Built-in | Built-in | No |

The GPL codecs (libx264, libx265, libxvid) are **video encoders** that Codex does not use. Even video *decoding* (reading H.264/H.265) works with LGPL builds — only encoding requires the GPL extras.

If an upstream source only provides GPL-compiled binaries, the CI will skip that platform and log a warning. Those platforms would need an LGPL build from an alternative source or a custom compile.

## Platforms

All 7 Codex shipping targets:
- darwin-arm64, darwin-x64
- linux-arm64, linux-arm, linux-x64
- win32-arm64, win32-x64

Plus Alpine (linuxmusl-arm64, linuxmusl-x64) and win32-ia32 for SQLite3.

## Custom Builds

Some platforms are not available upstream and are cross-compiled by CI:
- **sqlite3-linux-arm**: Cross-compiled with `gcc-arm-linux-gnueabihf`
- **sqlite3-win32-arm64**: Cross-compiled with MSVC ARM64
- **ffmpeg-win32-arm64**: Sourced from community builds (LGPL verified)

## Releases

Releases are tagged by tool+version: `sqlite3-v5.1.7`, `ffmpeg-v1.0.0`, `git-v2.47.3-1`

Each release contains platform-specific tarballs as assets.

## npm Packages

Published under `@codex-editor` scope:
- `@codex-editor/sqlite3-{platform}`
- `@codex-editor/ffmpeg-{platform}`
- `@codex-editor/git-{platform}`

---

## Maintenance Guide

This section is the operational reference for team members maintaining the binary pipeline.

### Prerequisites

| Tool | Install | Purpose |
|------|---------|---------|
| `gh` CLI | `brew install gh` (macOS) / [cli.github.com](https://cli.github.com) | GitHub repo management, secrets, workflow triggers |
| `npm` | Comes with Node.js | Publishing `@codex-editor` packages |
| Python 3.8+ | System or `brew install python3` | Running check/update scripts |
| `python3 -m pip install requests` | | Required by check/update scripts |

### One-Time Setup

#### 1. Authenticate `gh` CLI

```bash
gh auth login
```

Choose `GitHub.com`, then `HTTPS`, then `Login with a web browser`. Ensure your account has write access to the `genesis-ai-dev` org.

Verify:

```bash
gh auth status
```

#### 2. Create the `@codex-editor` npm org (if not already done)

Go to [npmjs.com/org/create](https://www.npmjs.com/org/create) and create the `codex-editor` organization. Add team members who need publish access.

#### 3. Add `NPM_TOKEN` as a repository secret

Generate a fine-grained token at [npmjs.com/settings/codex-editor/tokens](https://www.npmjs.com/settings/codex-editor/tokens) with publish access to the `@codex-editor` scope. Then:

```bash
gh secret set NPM_TOKEN --repo genesis-ai-dev/codex-prebuilt-binaries
```

Paste the token when prompted. This secret is used by all three CI workflows.

> **Token expiry**: If CI fails with `npm ERR! 403`, the token has expired. Generate a new one and re-run the command above.

#### 4. Trigger initial CI mirror runs

```bash
gh workflow run mirror-sqlite3.yml --repo genesis-ai-dev/codex-prebuilt-binaries
gh workflow run mirror-ffmpeg.yml --repo genesis-ai-dev/codex-prebuilt-binaries
gh workflow run mirror-git.yml --repo genesis-ai-dev/codex-prebuilt-binaries
```

Monitor progress in the [Actions tab](https://github.com/genesis-ai-dev/codex-prebuilt-binaries/actions).

### Checking for Upstream Updates

From the **codex-editor** repo root:

```bash
python3 scripts/check_binary_updates.py
```

This queries GitHub and npm APIs for each tool and compares against the versions pinned in `scripts/binary-registry.json`. It reports:
- New upstream versions available
- Platform coverage changes (e.g., a platform was added or dropped upstream)
- Any network errors

No tokens required — uses public APIs only.

### Updating a Binary Version

From the **codex-editor** repo root:

```bash
NPM_TOKEN=npm_xxxxx GITHUB_TOKEN=ghp_xxxxx python3 scripts/update_binary_version.py {tool} {version}
```

For example:

```bash
NPM_TOKEN=npm_abc123 GITHUB_TOKEN=ghp_def456 python3 scripts/update_binary_version.py sqlite3 5.1.8
```

**What it does:**
1. Validates `NPM_TOKEN` (checks expiry — exits with code 3 if expired)
2. Downloads binaries from upstream for all supported platforms
3. Repackages them as `@codex-editor/{tool}-{platform}` tarballs with license files
4. Publishes to npm (skips if version already exists)
5. Creates a GitHub Release on this repo with all tarballs as assets (skips if release exists)
6. Updates `scripts/binary-registry.json` with the new version
7. Updates the version constants in the relevant manager files (`sqliteNativeBinaryManager.ts`, `ffmpegManager.ts`, or `gitBinaryManager.ts`)

**If a platform requires a custom build** (e.g., sqlite3 linux-arm), the script will pause and instruct you to trigger the CI workflow for cross-compilation, then resume.

### Handling Expired Tokens

#### npm Token

If the update script exits with code 3, or CI fails with `403 Forbidden`:

1. Go to [npmjs.com/settings/codex-editor/tokens](https://www.npmjs.com/settings/codex-editor/tokens)
2. Generate a new token with publish access to `@codex-editor` scope
3. Update the repo secret: `gh secret set NPM_TOKEN --repo genesis-ai-dev/codex-prebuilt-binaries`
4. Re-run the failed workflow or script

#### GitHub Token

For the update script, use a personal access token with `repo` scope:
1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. Generate a token with `repo` scope (or fine-grained with Contents + Releases write access to `genesis-ai-dev/codex-prebuilt-binaries`)
3. Pass it as `GITHUB_TOKEN=ghp_xxx` when running the script

### Re-running CI Workflows Manually

Each workflow can be triggered from the [Actions tab](https://github.com/genesis-ai-dev/codex-prebuilt-binaries/actions) or via CLI:

```bash
# Mirror all SQLite3 platforms
gh workflow run mirror-sqlite3.yml --repo genesis-ai-dev/codex-prebuilt-binaries

# Mirror all FFmpeg platforms
gh workflow run mirror-ffmpeg.yml --repo genesis-ai-dev/codex-prebuilt-binaries

# Mirror all Git platforms
gh workflow run mirror-git.yml --repo genesis-ai-dev/codex-prebuilt-binaries
```

Workflows are idempotent — they skip publishing if the version already exists on npm or GitHub Releases.

### Where the Code Lives (codex-editor repo)

| File | Purpose |
|------|---------|
| `scripts/binary-registry.json` | Single source of truth: pinned versions, platform coverage, URLs |
| `scripts/check_binary_updates.py` | Check for upstream updates (read-only, no tokens needed) |
| `scripts/update_binary_version.py` | Automate full version update (download → repackage → publish → update code) |
| `src/utils/binaryDownloadUtils.ts` | Shared three-tier download helper used by all managers |
| `src/utils/sqliteNativeBinaryManager.ts` | SQLite3 binary download + platform detection |
| `src/utils/ffmpegManager.ts` | FFmpeg binary download + platform detection |
| `src/utils/toolsManager.ts` | Startup tool availability checks |
| `.cursor/rules/binary-dependency-management.mdc` | AI agent instructions for maintaining this system |

In the **frontier-authentication** repo:

| File | Purpose |
|------|---------|
| `src/git/gitBinaryManager.ts` | Git (dugite) binary download + platform detection |

### Key Rules

- **Never** update a version constant in code without also updating `scripts/binary-registry.json`
- **Never** remove a platform from the registry — if upstream drops it, set `"buildRequired": true`
- **Always** verify all 7 required platforms have binaries before merging a version bump
- **Never** compile FFmpeg with `--enable-gpl` or `--enable-nonfree` without explicit legal review
- **Never** publish a binary package without its license file included in the tarball
- If an upstream source disappears entirely, the Codex-owned copies become the sole source — treat as **critical**

---

## License

See the [LICENSES/](LICENSES/) directory for individual tool licenses and [LICENSES/NOTICE.md](LICENSES/NOTICE.md) for a summary.
