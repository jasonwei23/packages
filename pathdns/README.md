# pathdns OpenWrt package

OpenWrt `Makefile` + service files for [pathdns](https://github.com/jasonwei23/pathdns),
a policy-based DNS forwarder. This package does **not** compile pathdns —
it downloads and repackages the prebuilt static aarch64-musl binary that
pathdns' own release workflow already publishes
(`pathdns-linux-aarch64` on the GitHub release), verified against its
sha256. aarch64-only; there's no upstream release binary for other
architectures.

## Layout

- `Makefile` — package definition: fetches the release binary (`PKG_SOURCE`/
  `PKG_HASH`), `Build/Compile` is a no-op, `Package/pathdns/install` just
  installs it plus the service files below.
- `files/pathdns.init` — procd init script (`/etc/init.d/pathdns`). Also implements an
  optional DNS-hijack redirect: when `redirect_dns` is enabled, all tcp/udp port-53
  traffic on the router is DNAT'd (via a dedicated `inet pathdns` nftables table, so a
  `fw4 reload` won't wipe it) to whatever port pathdns is actually listening on. That
  port is read automatically from `config_file`'s `bind.port` at service start — no
  need to duplicate it into a second UCI option. If it can't be read (missing/invalid
  `bind.port`, bad JSON, wrong path), that's treated as a broken config and the service
  refuses to start at all, rather than guessing a port. This means `bind.port` must be
  set explicitly in `config_file` to use `redirect_dns`, even though pathdns itself is
  happy to fall back to its own built-in default (65353) when `bind` is omitted.
- `files/pathdns.config` — default UCI config (`/etc/config/pathdns`): `enabled`,
  `config_file`, `user`, `redirect_dns` — disabled by default.
- `files/pathdns.json` — default JSON config (`/etc/pathdns/config.json`): loopback bind on
  port 65353, single catch-all rule to two public resolvers, no dashboard. Edit this (or
  point `config_file` in `/etc/config/pathdns` elsewhere) before enabling the service —
  see the [upstream config reference](https://github.com/jasonwei23/pathdns#configuration)
  for ruleset-based split-horizon routing, DoT/DoH/DoQ upstreams, and the dashboard.

## Using this package

Drop this directory into an OpenWrt buildroot's package tree (e.g. as
`package/pathdns` or as a feed entry) — no external feed is required, since
there's nothing to compile, just `package/pathdns/{download,compile}` (a
no-op) and `install` — then:

```sh
make menuconfig   # Network > pathdns
make package/pathdns/{clean,compile} V=s
```

`.github/workflows/build-pathdns-apk.yml` automates this against a
downloaded OpenWrt SDK (no local buildroot checkout needed): download the
target's SDK + `config.buildinfo`, copy this package in,
`download`/`check`/`compile` (no-op) it, sign with an ephemeral CI key so
`apk` will accept the result with `--allow-untrusted` — same overall shape
as [douglarek/mihomo-openwrt](https://github.com/douglarek/mihomo-openwrt)'s
`build.yml`, minus the actual compilation step since pathdns already
publishes the binary. It's manual-only (`workflow_dispatch`, from the
Actions tab). `OPENWRT_RELEASE`/`OPENWRT_TARGET`/`OPENWRT_SUBTARGET` at the
top of the workflow pin the SDK; bump `OPENWRT_RELEASE` as OpenWrt cuts new
releases.

## Known follow-ups for a maintainer

- Bumping to a new upstream release means bumping `PKG_VERSION` and
  `PKG_HASH` together — get the hash from the release's published
  `pathdns-linux-aarch64.sha256` asset, or
  `curl -fsSL ".../releases/download/vX.Y.Z/pathdns-linux-aarch64" | sha256sum`.
  An earlier version of this Makefile built from source (first a pinned git
  commit, then a release source tarball via `lang/rust/rust-package.mk`);
  both were dropped once pathdns started publishing a release binary,
  since OpenWrt's `lang/rust` bootstraps a full from-source rustc+cargo
  host toolchain against OpenWrt's own musl (it can't use a prebuilt
  rustup toolchain, since that's built against upstream musl) — heavy
  enough to exhaust a default GitHub Actions runner's disk, and pointless
  next to a static binary pathdns already builds and signs itself.
- Upstream has no `LICENSE` file yet despite `Cargo.toml` declaring
  `license = "MIT"`; add one upstream, then set `PKG_LICENSE_FILES:=LICENSE`.
- pathdns requires Linux kernel >= 6.0 with io_uring available (it self-checks
  and refuses to start otherwise) — not something `DEPENDS` can express, so
  it's runtime-enforced by pathdns itself.
- The `redirect_dns` nftables rule's target port is fixed when the service
  starts; if you hot-edit `bind.port` in a running config, restart the
  service (`/etc/init.d/pathdns restart`) so the redirect picks it up.
- With `redirect_dns` enabled, `config_file` must set `bind.port` explicitly
  (the init script refuses to start rather than assume pathdns' own 65353
  default — see the note in Layout above).
