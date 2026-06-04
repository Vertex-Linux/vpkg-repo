# vpkg-repo
The repository with a bunch of files for VPKG

# vpkg Package Format

This document describes how to add packages to the Vertex Linux package repository so they can be installed with `vpkg`.

## Repository Layout

Each package lives in its own folder at the root of this repository. The folder name is what users type when they install the package.

```
vpkg-repo/
├── calla/
│   └── pkg.json
├── my-tool/
│   ├── pkg.json
│   └── my-tool-1.0.0-x86_64.pkg.tar.zst
└── another-package/
    ├── pkg.json
    └── pkgbuild.zip
```

Users install packages with:
```bash
vpkg vl install calla
vpkg install calla        # also works — auto-detects source
```

---

## `pkg.json` Reference

Every package folder must contain a `pkg.json` file.

### All fields

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | The package name shown in the TUI and used as the install name |
| `version` | string | yes | Version string shown in search results |
| `description` | string | no | Short description shown in the TUI |
| `type` | string | yes | Install method — `"pacman"`, `"makepkg"` / `"pkgbuild"`, or `"binary"` |
| `file` | string | when source is repo | Filename of the package file in this folder |
| `source` | string | no | `"repo"` (default) or `"github"` |
| `github_repo` | string | when source is github | GitHub repository in `owner/repo` format |
| `github_asset` | string | no | Release asset filename to download. Supports `{arch}`. If omitted, the repo is cloned instead |
| `github_tag_keyword` | string | no | Only use releases whose tag contains this string. Omit to use the latest release |
| `rename-file` | string | no | For `binary` installs — rename the downloaded file to this before placing it in `/usr/local/bin/` |
| `pm` | array | no | Pacman packages to install as dependencies before this package |
| `aur` | array | no | AUR packages to install as dependencies before this package |
| `fp` | array | no | Flatpak packages to install as dependencies before this package |

---

## Install Types

### `"pacman"`
Downloads the file and installs it with `sudo pacman -U`. Use this for pre-built Arch packages (`.pkg.tar.zst`).

### `"makepkg"` or `"pkgbuild"`
Either unzips an archive and runs `makepkg -si` from the directory containing a `PKGBUILD`, or clones a GitHub repo and does the same. Both values are accepted.

### `"binary"`
Downloads a file and copies it directly into `/usr/local/bin/` with executable permissions. Use this for pre-built standalone binaries. Use `rename-file` to control the final filename.

---

## Examples

### 1 — Pre-built pacman package hosted in this repo

The `.pkg.tar.zst` file sits next to `pkg.json` in the folder.

```json
{
  "name": "my-package",
  "version": "1.0.0",
  "description": "A pre-built Arch package",
  "type": "pacman",
  "file": "my-package-1.0.0-1-x86_64.pkg.tar.zst"
}
```

---

### 2 — PKGBUILD zip hosted in this repo

A zip file containing a `PKGBUILD` (in the root or one folder deep).

```json
{
  "name": "my-package",
  "version": "1.0.0",
  "description": "Built from a PKGBUILD",
  "type": "makepkg",
  "file": "pkgbuild.zip"
}
```

---

### 3 — Pre-built binary hosted in this repo

```json
{
  "name": "my-tool",
  "version": "1.0.0",
  "description": "A standalone binary",
  "type": "binary",
  "file": "my-tool-linux",
  "rename-file": "my-tool"
}
```

---

### 4 — Pre-built binary from a GitHub release (auto arch detection)

`{arch}` is automatically replaced with `x86_64` or `aarch64` depending on the user's system.

```json
{
  "name": "my-tool",
  "version": "latest",
  "description": "A standalone binary from GitHub releases",
  "type": "binary",
  "source": "github",
  "github_repo": "owner/my-tool",
  "github_asset": "my-tool-{arch}-unknown-linux-gnu",
  "rename-file": "my-tool"
}
```

---

### 5 — GitHub binary with tag keyword filter

Only grabs releases whose tag contains `"stable"`. Useful if a repo publishes both stable and nightly releases.

```json
{
  "name": "my-tool",
  "version": "latest",
  "description": "Stable releases only",
  "type": "binary",
  "source": "github",
  "github_repo": "owner/my-tool",
  "github_asset": "my-tool-{arch}-unknown-linux-gnu",
  "github_tag_keyword": "stable",
  "rename-file": "my-tool"
}
```

---

### 6 — PKGBUILD built directly from a GitHub repo

No `github_asset` is specified, so vpkg clones the repo with `git clone --depth=1` and runs `makepkg -si` from wherever it finds a `PKGBUILD`.

```json
{
  "name": "calla",
  "version": "1.0.4",
  "description": "A super fast and efficient desktop environment based on Awesome.",
  "type": "makepkg",
  "source": "github",
  "github_repo": "Vertex-Linux/Calla",
  "aur": ["awesome-git", "noto-color-emoji-fontconfig", "lua-pam-git"],
  "pm":  ["playerctl", "noto-fonts", "picom", "brightnessctl", "xorg-server"]
}
```

---

### 7 — Package with dependencies

Dependencies are installed before the main package. When installing, vpkg will list all deps and ask the user to confirm before proceeding. Answering `n` skips the deps and installs only the main package.

```json
{
  "name": "my-app",
  "version": "2.1.0",
  "description": "An app with many dependencies",
  "type": "pacman",
  "source": "github",
  "github_repo": "owner/my-app",
  "github_asset": "my-app-{arch}.pkg.tar.zst",
  "pm":  ["gtk3", "libpng", "ffmpeg"],
  "aur": ["some-aur-library"],
  "fp":  ["org.freedesktop.Platform"]
}
```

---

## How the GitHub source fallback works

When `source` is `"github"`:

```
github_asset specified?
├── yes → look for that asset in the release
│         ├── found → download it → install
│         └── not found → git clone the repo → install from there
└── no  → git clone the repo → install from there
```

This means you can ship a PKGBUILD-only repo with zero releases and vpkg will still install it correctly.

---

## Adding a Package

1. Create a folder with the package name at the repo root
2. Add a `pkg.json` following the examples above
3. If hosting files directly, add the package file to the same folder
4. Commit and push — vpkg reads directly from the `main` branch
