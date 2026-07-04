# pathdns OpenWrt package

OpenWrt `Makefile` + service files for [pathdns](https://github.com/jasonwei23/pathdns),
a policy-based DNS forwarder. Builds the upstream Rust crate via the
`lang/rust` feed's `rust-package.mk`.

## Layout

- `Makefile` — package definition (git snapshot source, Rust build, install rules).
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
`package/pathdns` or as a feed entry), with the official `packages` feed
present at `feeds/packages` (needed for `lang/rust/rust-package.mk` and the
Rust host toolchain), then:

```sh
./scripts/feeds update -a && ./scripts/feeds install -a
make menuconfig   # Network > pathdns
make package/pathdns/{clean,compile} V=s
```

`.github/workflows/build-pathdns-apk.yml` automates this against a
downloaded OpenWrt SDK (no local buildroot checkout needed) and produces
an aarch64 `.apk`, modeled on
[douglarek/mihomo-openwrt](https://github.com/douglarek/mihomo-openwrt)'s
approach: download the target's SDK + `config.buildinfo`, copy this
package in, `download`/`check`/`compile` it, sign with an ephemeral CI
key so `apk` will accept the result with `--allow-untrusted`. Run it via
the Actions tab (`workflow_dispatch`) or by pushing to `main` under
`pathdns/`. `OPENWRT_RELEASE`/`OPENWRT_TARGET`/`OPENWRT_SUBTARGET` at the
top of the workflow pin the SDK; bump `OPENWRT_RELEASE` as OpenWrt cuts
new point releases.

## Known follow-ups for a maintainer

- `PKG_MIRROR_HASH:=skip` — no release tarball/mirror hash exists yet for this
  git snapshot. Run `make package/pathdns/download V=s` against a real
  OpenWrt tree once and fill in the printed hash.
- Upstream has no `LICENSE` file yet despite `Cargo.toml` declaring
  `license = "MIT"`; add one upstream, then set `PKG_LICENSE_FILES:=LICENSE`.
- pathdns requires Linux kernel >= 6.0 with io_uring available (it self-checks
  and refuses to start otherwise) — not something `DEPENDS` can express, so
  it's runtime-enforced by pathdns itself.
- Optional Cargo features `doq` (DNS-over-QUIC) and `h3` (DoH3) are not
  enabled by default; add `RUST_PKG_FEATURES:=doq,h3` in the Makefile if
  those upstream transports are needed.
- The `redirect_dns` nftables rule's target port is fixed when the service
  starts; if you hot-edit `bind.port` in a running config, restart the
  service (`/etc/init.d/pathdns restart`) so the redirect picks it up.
- With `redirect_dns` enabled, `config_file` must set `bind.port` explicitly
  (the init script refuses to start rather than assume pathdns' own 65353
  default — see the note in Layout above).
