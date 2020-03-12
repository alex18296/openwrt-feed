include $(TOPDIR)/rules.mk

PKG_NAME:=libgost
PKG_VERSION:=1.1.0.3
PKG_RELEASE:=1

PKG_LICENSE:=Apache-2.0
PKG_LICENSE_FILES:=COPYING
PKG_MAINTAINER:=

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk

define Package/$(PKG_NAME)
	SECTION:=libs
	SUBMENU:=SSL
	CATEGORY:=Libraries
	DEPENDS:=+libopenssl +libopenssl-conf +@OPENSSL_ENGINE +@OPENSSL_WITH_GOST
	TITLE:=The engine support for Russian GOST crypto algorithms.
endef

define Package/$(PKG_NAME)/description
	A reference implementation of the Russian GOST crypto algorithms for OpenSSL.
	Original sources in https://github.com/gost-engine/engine version 1.1.0.3
endef

define Build/Prepare
	mkdir -p $(PKG_BUILD_DIR)
	if [ ! -d "$(PKG_BUILD_DIR)/src" ]; then git clone https://github.com/gost-engine/engine -b v1.1.0.3 $(PKG_BUILD_DIR)/src || true; fi
	$(CP) ./files/* $(PKG_BUILD_DIR)/
endef

define Package/$(PKG_NAME)/install
	$(INSTALL_DIR) $(1)/usr/lib/engines-1.1/ $(1)/etc/uci-defaults/
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/gost.so $(1)/usr/lib/engines-1.1/
	$(LN) gost.so $(1)/usr/lib/engines-1.1/gost.so.1.1
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/gost_init $(1)/etc/uci-defaults/
endef

define Package/$(PKG_NAME)/postinst
#!/bin/sh
( . /etc/uci-defaults/gost_init ) && rm -f /etc/uci-defaults/gost_init
exit 0
endef

define Package/$(PKG_NAME)/postrm
#!/bin/sh
if [ -f /etc/ssl/openssl.bak ] ; then
  mv -f /etc/ssl/openssl.bak /etc/ssl/openssl.cnf
fi
exit 0
endef

$(eval $(call BuildPackage,$(PKG_NAME)))
