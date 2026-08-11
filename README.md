# MinkaPKG

PKGBUILDs for software that isn't in the Arch repos or the AUR, plus the helper
that publishes them into a local pacman repository.

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

Add to the end of `/etc/pacman.conf`, after `[kotontrion]`, so it can't shadow
a system package:

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
| `lxmf-rs` | [FreeTAKTeam/LXMF-rs](https://github.com/FreeTAKTeam/LXMF-rs) | Reticulum + LXMF in Rust. Not in the AUR. `rns-tools` ships eight binary names that collide with `python-rns`, so they install to `/usr/lib/lxmf-rs/bin` instead of `/usr/bin` — see the PKGBUILD comment. |
