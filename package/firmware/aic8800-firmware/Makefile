include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/package.mk

PKG_NAME:=aic8800-firmware
AIC_TAG:=4.0+git20250410.b99ca8b6-5
PKG_VERSION:=20250410
PKG_RELEASE:=5

PKG_BUILD_DIR:=$(BUILD_DIR)/aic8800-$(PKG_VERSION)

define Package/aic8800-firmware
  SECTION:=firmware
  CATEGORY:=Firmware
  TITLE:=Radxa AIC8800 firmware
endef

define Package/aic8800-firmware/description
 Firmware for Radxa AIC8800 USB devices.
endef

define Build/Prepare
	rm -rf $(PKG_BUILD_DIR)
	mkdir -p $(PKG_BUILD_DIR)
	cp -a /tmp/aic8800-src/. $(PKG_BUILD_DIR)/
endef

define Build/Compile
	true
endef

define Package/aic8800-firmware/install
	$(INSTALL_DIR) $(1)/lib/firmware/aic8800_fw/USB
	$(CP) $(PKG_BUILD_DIR)/src/USB/driver_fw/fw/* $(1)/lib/firmware/aic8800_fw/USB/
endef

$(eval $(call BuildPackage,aic8800-firmware))
