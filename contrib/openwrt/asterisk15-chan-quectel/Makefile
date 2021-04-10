#
# Copyright (C) 2011 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=asterisk15-chan-quectel
#PKG_REV:=10
PKG_VERSION:=1.1-20210410
#.r$(PKG_REV)
PKG_RELEASE:=1

#PKG_SOURCE_URL:=http://asterisk-chan-quectel.googlecode.com/svn/trunk/
#PKG_SOURCE_VERSION:=$(PKG_RELEASE)
#PKG_SOURCE:=chan_quectel-$(PKG_VERSION).tar.gz
PKG_SOURCE_PROTO:=git
PKG_FIXUP:=autoreconf
# or
PKG_SOURCE:=chan_quectel-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL=https://github.com/t4rd15/asterisk-chan-quectel.git
PKG_MD5SUM:=0b7495179f5ef7b7fc4b1bd6c82557b6275cb8ad

PKG_SOURCE_SUBDIR:=chan_quectel-$(PKG_VERSION)
PKG_BUILD_DIR=$(BUILD_DIR)/$(PKG_SOURCE_SUBDIR)

WITH_ASTERISK=asterisk-15.3.0

include $(INCLUDE_DIR)/package.mk

define Package/asterisk15-chan-quectel
	SUBMENU:=asterisk15 (Complete Open Source PBX), v15.x
	SECTION:=net
	CATEGORY:=Network
	URL:=http://www.asterisk.org/
	MAINTAINER:=Hans Zandbelt <hans.zandbelt@gmail.com>
	DEPENDS:= +asterisk15 +libiconv-full
	TITLE:=Quectel LTE cards support
endef

define Package/asterisk15-chan-quectel/description
	Asterisk channel driver for Quectel LTE cards
endef

MAKE_ARGS:= \
	CC="$(TARGET_CC)" \
	LD="$(TARGET_CC)" \
	CFLAGS="$(TARGET_CFLAGS) -DLOW_MEMORY -D_XOPEN_SOURCE=600 $(TARGET_CPPFLAGS) -I$(STAGING_DIR)/usr/lib/libiconv-full/include -I$(BUILD_DIR)/$(WITH_ASTERISK)/include -DHAVE_CONFIG_H -I. -fPIC" \
	LDFLAGS="$(TARGET_LDFLAGS) -L$(STAGING_DIR)/usr/lib/libiconv-full/lib -liconv" \
	DESTDIR="$(PKG_INSTALL_DIR)/usr/lib/asterisk/modules"

define Build/Configure
	$(call Build/Configure/Default, \
	    --with-asterisk=$(BUILD_DIR)/$(WITH_ASTERISK)/include \
	    $(MAKE_ARGS) \
	)
endef

define Build/Compile
	mkdir -p $(PKG_INSTALL_DIR)/usr/lib/asterisk/modules
	$(MAKE) -C "$(PKG_BUILD_DIR)" $(MAKE_ARGS) all install
endef

define Package/asterisk15-chan-quectel/conffiles
/etc/asterisk/quectel.conf
endef

define Package/asterisk15-chan-quectel/install
	$(INSTALL_DIR) $(1)/etc/asterisk
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/etc/quectel.conf $(1)/etc/asterisk/
	$(INSTALL_DIR) $(1)/usr/lib/asterisk/modules
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/lib/asterisk/modules/chan_quectel.so $(1)/usr/lib/asterisk/modules/
endef

$(eval $(call BuildPackage,asterisk15-chan-quectel))
