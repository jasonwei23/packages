include $(TOPDIR)/rules.mk

PKG_NAME:=pathdns
PKG_VERSION:=0.1.0
PKG_RELEASE:=1

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/jasonwei23/pathdns.git
PKG_SOURCE_DATE:=2026-07-04
PKG_SOURCE_VERSION:=afe83a714fda2318ae0dc5dafae416b0affb45ce
PKG_MIRROR_HASH:=skip

PKG_MAINTAINER:=jasonwei23 <jasonwei23@users.noreply.github.com>
PKG_LICENSE:=MIT

PKG_BUILD_DEPENDS:=rust/host
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk
include $(TOPDIR)/feeds/packages/lang/rust/rust-package.mk

define Package/pathdns
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=IP Addresses and Names
  TITLE:=Policy-based DNS forwarder for split-horizon deployments
  URL:=https://github.com/jasonwei23/pathdns
  DEPENDS:=$(RUST_ARCH_DEPENDS) +jsonfilter +nftables
endef

define Package/pathdns/description
  PathDNS routes DNS queries to different upstream resolvers based on
  ruleset domain classification and custom routing rules, with optional
  ipset/nftset integration for IP-based routing decisions. It supports
  DoT/DoH/DoQ/DoH3 upstreams, EDNS-aware caching, an answer-ip test
  fallback for primary/secondary routing, and a built-in HTTP dashboard.

  Linux-only: uses io_uring (multishot recvmsg), sendmmsg, SO_REUSEPORT
  and netlink directly. Requires Linux kernel 6.0 or newer.

  Optionally hijacks all LAN DNS traffic (tcp/udp port 53) to pathdns via
  a dedicated nftables table, regardless of clients' configured resolver
  (see the redirect_dns option in /etc/config/pathdns).
endef

define Package/pathdns/conffiles
/etc/config/pathdns
/etc/pathdns/config.json
endef

define Package/pathdns/install
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/bin/pathdns $(1)/usr/sbin/pathdns

	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/pathdns.init $(1)/etc/init.d/pathdns

	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/pathdns.config $(1)/etc/config/pathdns

	$(INSTALL_DIR) $(1)/etc/pathdns
	$(INSTALL_DATA) ./files/pathdns.json $(1)/etc/pathdns/config.json
endef

define Package/pathdns/postinst
#!/bin/sh
[ -n "$${IPKG_INSTROOT}" ] || {
	echo "pathdns is disabled by default; edit /etc/pathdns/config.json"
	echo "and /etc/config/pathdns, then run:"
	echo "  /etc/init.d/pathdns enable"
	echo "  /etc/init.d/pathdns start"
}
exit 0
endef

$(eval $(call BuildPackage,pathdns))
