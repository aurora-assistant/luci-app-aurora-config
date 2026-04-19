# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build

This is a pure OpenWrt LuCI package — no npm/node toolchain. The package is built by the OpenWrt build system:

```sh
# From the OpenWrt build root with this package in feeds:
make package/luci-app-aurora-config/compile
```

Release artifacts (`.ipk` / `.apk`) are produced by the GitHub Actions workflow in `.github/workflows/build-and-release-aurora-config.yml`, triggered on `v*` tags via the shared `eamonxg/build-luci-package` action.

There are no lint, test, or format commands — code quality is enforced through review.

## Architecture

The app is split into three layers:

### Frontend — `htdocs/luci-static/resources/view/aurora/`

LuCI JavaScript views using the `form.*`, `uci.*`, and `rpc.*` APIs from `luci-base`.

- **`theme.js`** — Main settings UI with tabbed sections: Color (light/dark pickers), Structure (layout spacing), Icons & Toolbar. Calls RPC methods for all read/write operations. Also handles preset application, config import/export, and version-check display.
- **`version.js`** — Update management: compares installed vs. GitHub release versions, downloads and installs packages sequentially (theme + config + i18n). Caches results in `localStorage` with a 30-minute TTL.
- **`color.global.js`** — Color math library (Color.js-based). Loaded dynamically by `theme.js`. Handles oklch↔hex conversions and matrix-based color space transforms.

### Backend RPC — `root/usr/libexec/rpcd/luci.aurora`

Shell script implementing the `ubus` RPC interface. Exposes ~13 methods including: `get_theme_config`, `apply_theme_preset`, `list_icons`, `upload_icon`, `remove_icon`, `export_config`, `import_config`, `get_installed_versions`, `check_updates`, `download_package`, `install_package`. Handles both `opkg` and `apk` package managers. Path traversal protection is applied to icon file operations.

### Configuration

- **`root/usr/share/aurora/*.template`** — Five built-in theme presets (Classic, Monochrome, Sage Green, Amber Sand, Sky Blue) in UCI format using `oklch()` color values.
- **`root/etc/uci-defaults/80_aurora`** — Runs once on install; bootstraps `/etc/config/aurora` from the Classic template if absent.
- **`root/usr/share/luci/menu.d/luci-app-aurora.json`** — Registers two menu entries under `admin/system/aurora`: Theme Settings (order 10) and Version Management (order 20).
- **`root/usr/share/rpcd/acl.d/luci-app-aurora.json`** — Grants read/write ACL for UCI aurora config and all RPC methods.

### Internationalisation — `po/`

14 language packs (zh_Hans, zh_Hant, de, es, fr, id, it, ja, ko, nl, pl, ru, tr, uk, vi). `.po` files are compiled into LuCI's catalogue format by the OpenWrt build system.

## Key Patterns

- All frontend↔backend communication goes through LuCI's `rpc.declare` / `callXxx` pattern — never direct shell execution from JS.
- Color values in templates and UCI config use `oklch()` notation.
- Version strings follow semver; the RPC `version_compare()` shell function handles ordering.
- The `localStorage` cache key scheme is `aurora_version_cache` / `aurora_version_cache_time` for the 30-minute TTL check.
