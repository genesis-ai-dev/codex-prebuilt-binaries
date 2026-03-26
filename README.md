# Codex Prebuilt Binaries

This repository hosts prebuilt native binaries for the [Codex Editor](https://codexeditor.app) application.
Binaries are mirrored from their original upstream sources for reliability and published to both npm (`@codex-editor` scope) and GitHub Releases.

## Tools

| Tool | License | Upstream |
|------|---------|----------|
| SQLite3 (node-sqlite3) | BSD 3-Clause | [TryGhost/node-sqlite3](https://github.com/TryGhost/node-sqlite3) |
| FFmpeg | LGPL-2.1 | [ffmpeg.org](https://ffmpeg.org) / [@ffmpeg-installer](https://www.npmjs.com/package/@ffmpeg-installer/ffmpeg) |
| Git | GPL-2.0 | [desktop/dugite-native](https://github.com/desktop/dugite-native) |

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

## License

See the [LICENSES/](LICENSES/) directory for individual tool licenses and [LICENSES/NOTICE.md](LICENSES/NOTICE.md) for a summary.
