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

The trade-off: pamac's `CheckAURUpdates` only queries the real AUR, so it never
says when upstream publishes a new version. `minkarepo check` does that instead
(see Keeping packages current), and `minkarepo update` bumps, builds and
publishes; `pacman -Syu` then offers it like any other package.

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
./minkarepo check              # what upstream has, per package
./minkarepo update  lxmf-rs    # bump to upstream's newest, then publish
./minkarepo update  --all      # the same for every package marked auto=yes
```

`MINKAREPO_DIR` and `MINKAREPO_DB` override the location and database name.

Builds run in their own systemd user scope by default, rather than in the
graphical login session's scope, with `MemoryHigh=8G`, `MemoryMax=10G`, and
`MemorySwapMax=4G`. Override those limits with `MINKAREPO_MEMORY_HIGH`,
`MINKAREPO_MEMORY_MAX`, and `MINKAREPO_MEMORY_SWAP_MAX`, or set
`MINKAREPO_NO_ISOLATE=1` to run `makepkg` directly in the calling session.
Builds also get a disk-backed per-package `TMPDIR` under
`~/.cache/minkarepo/tmp` (override the parent with `MINKAREPO_TMPDIR`), so
compiler temporary files cannot exhaust the tmpfs per-user quota on `/tmp`.

Symlink it onto PATH if you'd rather not type the path:

```sh
ln -s "$PWD/minkarepo" ~/.local/bin/minkarepo
```

## Keeping packages current

Each package directory has an `upstream` file: plain `key=value` lines, read
and never sourced, saying where its newest version comes from and whether it is
safe to bump unattended. `minkarepo check` asks every upstream and prints the
PKGBUILD's version, the published one, upstream's newest and what to do:

| kind | newest version from | `update` bumps |
|---|---|---|
| `github-release` | the repo's latest release (`repo=`) | `pkgver`, `pkgrel=1`, checksums |
| `github-tag` | its highest tag matching `tag_regex=`, turned into a pkgver by `version_sed=` (default `s/^v//`) | the same |
| `git-commit` | the newest commit on `branch=` (default: the repo's) | `_commit=` and pkgrel + 1 (pkgver stays, as Arch and CachyOS do for a rebuild) |
| `local` | the PKGBUILD's own `pkgver()`: the release of a checkout in this repo (`checkout=`) | written by `publish` itself: a new release resets pkgrel, a new commit raises it |
| `follows` | another package dir's pkgver (`package=`) | `pkgver`, `pkgrel=1` |
| `watch-package` | a sync-repo package's version | nothing: reported for a hand bump |

`auto=yes` lets `minkarepo update --all` bump, build and publish a package
without asking; today that is `minka-syu`, `floorp-dev` and `lxmf-rs`. The rest
(`auto=no`) are only reported by `--all`, with their `note=` saying what to
re-check: boot-critical (`bcachefs-dkms`), hours-long builds (`floorp`,
`firestorm-novlc`), bump steps in the PKGBUILD header, or a vanished upstream
(`rbrowser`). `minkarepo update <name>` still does their mechanical part on
request and prints that note first. GitHub is read with `git ls-remote` where
possible; set `GITHUB_TOKEN` if the API's 60 requests an hour run out.

Picking by hand is easier in MinkaSyu's **Sources** tab: it lists `minkarepo
check --tsv`, ticks what to bump and runs `minkarepo update <picked>` in a
terminal. A package you don't care to follow gets `track=no` ("don't track"
there, or `minkarepo track <pkg> no`): `check` lists it as untracked without
asking upstream, and `update --all` leaves it alone. `pkgrel_new=N` sets the
pkgrel a bump starts from (bcachefs-dkms uses 2, so CachyOS's `x-1.1` rebuild
never sorts above it).

Versions stay release versions everywhere: nothing is versioned by commit.
`minka-syu`'s `pkgver()` takes the higher of the MinkaSyu checkout's newest
numeric tag and its committed `Cargo.toml` version; with a `kind=local` upstream
file, `publish` and `update` then raise pkgrel for each newer commit
(`0.6.0-1`, `0.6.0-2`, …) and reset it on a new release. They refuse a version
below the published one (`MINKAREPO_ALLOW_DOWNGRADE=1` overrides). Commit
first: uncommitted changes are built but not tracked. A package directory
without an `upstream` file is simply not tracked.

## Packages

| Directory | Upstream | Notes |
|---|---|---|
| `bs-manager-bin` | [Zagrios/bs-manager](https://github.com/Zagrios/bs-manager) | **Fixed repack of the AUR package of the same name.** Beat Saber version, map and mod manager, from upstream's `.rpm`. In the AUR build, Download does nothing: makepkg strips the bundled `DepotDownloader`, a .NET single-file program whose payload sits after the ELF image, and it then exits with code 159. This one sets `options=('!strip')`, runs the helper in `check()`, depends on the OpenSSL 3 library (the helper does not load OpenSSL 4) and adds `/usr/bin/bs-manager`. The PKGBUILD header has the measurements. |
| `firestorm-novlc` | [firestormviewer.org](https://www.firestormviewer.org) | Second Life / OpenSim viewer, built without LVC. Carries five local patches (fortify, config dir, GCC 16 SFINAE, WebRTC multiple-definition, SLURL handoff). |
| `floorp` | [Floorp-Projects/Floorp](https://github.com/Floorp-Projects/Floorp) | **Patched fork of the AUR package.** Floorp 12.17.1 will not configure: its branding sets `MOZ_APP_VENDOR` through confvars, but upstream mozilla-central made that a `project_flag` (settable only via `imply_option`), so configure dies with `InvalidOptionError`. `0002-vendor-must-be-implied-not-confvars.patch` drops the confvars assignment and moves the value to the `imply_option` in `browser/moz.configure`, which keeps the vendor as "Ablaze" — deleting the line alone would silently make it "Mozilla" and move the profile path. It must be a `.patch`, not a `sed` early in `prepare()`: the branding is rsynced in from `.github/assets/branding/` at the top of `prepare()`, and the patch loop deliberately runs after it. |
| `floorp-dev` | [Floorp-Projects/Floorp](https://github.com/Floorp-Projects/Floorp) | Fileless metapackage holding the runtime and build dependencies for a local Floorp checkout. It neither depends on nor provides `floorp`, and installs no browser files. |
| `hypercat` | [savannah-i-g/HyperCat-Agent](https://github.com/savannah-i-g/HyperCat-Agent) | Personality-driven multi-agent workspace application. |
| `lib32-libcaca` | [cacalabs/libcaca](https://github.com/cacalabs/libcaca) | **Fixed rebuild of the AUR package of the same name.** The 32-bit libcaca, needed by `lib32-gst-plugins-good` and `steam-native-runtime`; it left `[multilib]` for the AUR. The AUR package still depends on `lib32-imlib2`, whose AUR build is broken, although the library never links imlib2: only the tools `cacaview` and `img2txt` do, and neither is shipped. This one drops that dependency and builds the source of Arch's 64-bit `libcaca` (its tag and two patches) with every driver set by a configure switch. It has the same files, exported symbols and drivers (x11, raw, null) as the 0.99.beta20-2 installed now, the last `[multilib]` build (the AUR's PKGBUILD, built here, would add ncurses). Its version is `0.99.beta20-2`, the same as the AUR package's. Once it is installed, `lib32-imlib2` can be removed. |
| `minka-syu` | MinkaDE (this repo) | Update checker/notifier: Rust core + AntiWebUI window (`minka-syu-gui`), with the QML UI kept as the fallback. Built from the sibling `MinkaSyu` submodule (no source tarball); the QML UI's Proustite/MinkaLink symlinks are dereferenced into `/usr/share/minka-syu/ui`. After installing, `systemctl --user enable --now minka-syu-check.timer` for periodic checks. |
| `lxmf-rs` | [FreeTAKTeam/LXMF-rs](https://github.com/FreeTAKTeam/LXMF-rs) | Reticulum + LXMF in Rust. Not in the AUR. `rns-tools` ships eight binary names that collide with `python-rns`, so they install to `/usr/lib/lxmf-rs/bin` instead of `/usr/bin` — see the PKGBUILD comment. |
| `rbrowser` | [fr33n0w/rBrowser](https://github.com/fr33n0w/rBrowser) | Standalone web UI browser for NomadNet nodes and pages over Reticulum. |
| `renbrowser` | [Quad4-Software/Ren-Browser](https://github.com/Quad4-Software/Ren-Browser) | Reticulum browser for NomadNet pages. |

## Building a package that is also in the AUR

`floorp`, `bs-manager-bin` and `lib32-libcaca` share their `pkgname` with
an AUR package, so the local build shadows it once `[minka]` is configured.
That is intended — but `CheckAURUpdates` still queries the real AUR, so
pamac may offer the unpatched AUR version as an "update". Rebuild from here
rather than accepting it, or rename the package if that ever becomes annoying.
