# pathdns OpenWrt package

OpenWrt `Makefile` + service files for [pathdns](https://github.com/jasonwei23/pathdns),
a policy-based DNS forwarder. Builds the upstream Rust crate via the
`lang/rust` feed's `rust-package.mk`.

## Layout

- `Makefile` — package definition (git snapshot source, Rust build, install rules).
- `files/pathdns.init` — procd init script (`/etc/init.d/pathdns`).
- `files/pathdns.config` — default UCI config (`/etc/config/pathdns`), disabled by default.
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
