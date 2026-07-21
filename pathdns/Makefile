include $(TOPDIR)/rules.mk

PKG_NAME:=pathdns
PKG_VERSION:=0.1.2
PKG_RELEASE:=1

PKG_SOURCE:=pathdns-linux-aarch64
PKG_SOURCE_URL:=https://github.com/jasonwei23/pathdns/releases/download/v$(PKG_VERSION)
PKG_HASH:=330eb780414d4c35a11d5441185ef87dd4ea7dd6ca7d2633fd169b897b0d7105

PKG_MAINTAINER:=jasonwei23 <jasonwei23@users.noreply.github.com>
PKG_LICENSE:=MIT

include $(INCLUDE_DIR)/package.mk

define Package/pathdns
  SECTION:=net
  CATEGORY:=Network
  SUBMENU:=IP Addresses and Names
  TITLE:=Policy-based DNS forwarder for split-horizon deployments
  URL:=https://github.com/jasonwei23/pathdns
  DEPENDS:=@aarch64 +jsonfilter +nftables
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

  This package repackages the prebuilt static aarch64-musl binary from
  pathdns' own GitHub release rather than building from source.
endef

define Package/pathdns/conffiles
/etc/config/pathdns
/etc/pathdns/config.json
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	$(CP) $(DL_DIR)/$(PKG_SOURCE) $(PKG_BUILD_DIR)/pathdns
endef

define Build/Compile
endef

define Package/pathdns/install
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/pathdns $(1)/usr/sbin/pathdns

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
