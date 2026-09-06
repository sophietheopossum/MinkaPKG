# MinkaPKG

MinkaDE's packaging repo: PKGBUILDs for everything this desktop needs built from
source, plus the helper that publishes them into a local pacman repository.

One directory per package. `minkarepo` builds them and adds the results to
`/srv/localrepo/x86_64`, which appears in pamac alongside `[extra]` and
`[kotontrion]` — searchable, installable and upgradable through the GUI.

## Why a repo and not a local AUR

pamac has `https://aur.archlinux.org` compiled into libpamac with no config key
to redirect it, so a private AUR-alike can't be made searchable. A pacman
repository can be, and it installs prebuilt packages instead of compiling on
demand.

The trade-off: nothing tells you when upstream publishes a new version, because
`CheckAURUpdates` only queries the real AUR. Updates themselves work normally —
bump `pkgver`, rebuild, and `pacman -Syu` offers it like any other package.

## One-time setup

The repo directory is owned by your user so everything afterwards is sudo-free:

```sh
sudo mkdir -p /srv/localrepo/x86_64
sudo chown "$(id -un):$(id -gn)" /srv/localrepo/x86_64
```

Then create an empty database — **before** touching `pacman.conf`:

```sh
./minkarepo init
```

`repo-add` refuses to create a database with no packages in it, so configuring
the repo first gives `failed retrieving file 'minka.db' from disk` on the next
`pacman -Sy`. `init` writes an empty `.db` and `.files` so the repo can be
configured while still empty.

Now add to the end of `/etc/pacman.conf`, after `[kotontrion]`, so it can't
shadow a system package:

```ini
[minka]
SigLevel = Optional TrustAll
Server = file:///srv/localrepo/$arch
```

`SigLevel = Optional TrustAll` skips signature checks, which is reasonable for
a root-owned directory holding packages you built yourself. If this repo ever
gets served over a network, sign with `makepkg --sign` and switch to
`SigLevel = Required` instead.

Then `sudo pacman -Sy`.

## Usage

```sh
./minkarepo init               # create an empty db (one-time, before pacman.conf)
./minkarepo publish lxmf-rs    # build, then add to the repo
./minkarepo build   lxmf-rs    # build only
./minkarepo add     lxmf-rs    # add an already-built package
./minkarepo list               # what the repo holds
./minkarepo remove  lxmf-rs    # drop it and delete the file
```

`MINKAREPO_DIR` and `MINKAREPO_DB` override the location and database name.

Symlink it onto PATH if you'd rather not type the path:

```sh
ln -s "$PWD/minkarepo" ~/.local/bin/minkarepo
```

## Packages

| Directory | Upstream | Notes |
|---|---|---|
| `firestorm-nolvc` | [firestormviewer.org](https://www.firestormviewer.org) | Second Life / OpenSim viewer, built without LVC. Carries four local patches (fortify, config dir, GCC 16 SFINAE, WebRTC multiple-definition). |
| `floorp` | [Floorp-Projects/Floorp](https://github.com/Floorp-Projects/Floorp) | **Patched fork of the AUR package.** Floorp 12.17.1 will not configure: its branding sets `MOZ_APP_VENDOR` through confvars, but upstream mozilla-central made that a `project_flag` (settable only via `imply_option`), so configure dies with `InvalidOptionError`. `0002-vendor-must-be-implied-not-confvars.patch` drops the confvars assignment and moves the value to the `imply_option` in `browser/moz.configure`, which keeps the vendor as "Ablaze" — deleting the line alone would silently make it "Mozilla" and move the profile path. It must be a `.patch`, not a `sed` early in `prepare()`: the branding is rsynced in from `.github/assets/branding/` at the top of `prepare()`, and the patch loop deliberately runs after it. |
| `hypercat` | [savannah-i-g/HyperCat-Agent](https://github.com/savannah-i-g/HyperCat-Agent) | Personality-driven multi-agent workspace application. |
| `lxmf-rs` | [FreeTAKTeam/LXMF-rs](https://github.com/FreeTAKTeam/LXMF-rs) | Reticulum + LXMF in Rust. Not in the AUR. `rns-tools` ships eight binary names that collide with `python-rns`, so they install to `/usr/lib/lxmf-rs/bin` instead of `/usr/bin` — see the PKGBUILD comment. |
| `rbrowser` | [fr33n0w/rBrowser](https://github.com/fr33n0w/rBrowser) | Standalone web UI browser for NomadNet nodes and pages over Reticulum. |
| `renbrowser` | [Quad4-Software/Ren-Browser](https://github.com/Quad4-Software/Ren-Browser) | Reticulum browser for NomadNet pages. |

## Building a package that is also in the AUR

`floorp` shares its `pkgname` with the AUR package, so the local build shadows it
once `[minka]` is configured. That is intended — but `CheckAURUpdates` still
queries the real AUR, so pamac may offer the unpatched AUR version as an
"update". Rebuild from here rather than accepting it, or rename the package if
that ever becomes annoying.
