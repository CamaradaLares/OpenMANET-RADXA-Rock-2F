include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/package.mk

STAMP_CONFIGURED_DEPENDS := $(STAGING_DIR)/usr/include/mac80211-backport/backport/autoconf.h

PKG_NAME:=aic8800-usb
AIC_TAG:=4.0+git20250410.b99ca8b6-5
PKG_VERSION:=20250410
PKG_RELEASE:=8
PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/aic8800-$(PKG_VERSION)

define KernelPackage/aic8800-usb
  SUBMENU:=Wireless Drivers
  TITLE:=Radxa AIC8800 USB Wi-Fi driver
  DEPENDS:=+kmod-usb-core +kmod-usb2 +kmod-cfg80211 +kmod-mac80211 +aic8800-firmware
  FILES:= \
	$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic_load_fw/aic_load_fw.ko \
	$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko
  AUTOLOAD:=$(call AutoProbe,aic_load_fw aic8800_fdrv)
endef

define KernelPackage/aic8800-usb/description
 Radxa AIC8800D80 USB Wi-Fi driver.
endef

define Build/Prepare
	rm -rf $(PKG_BUILD_DIR)
	mkdir -p $(PKG_BUILD_DIR)
	cp -a /tmp/aic8800-src/. $(PKG_BUILD_DIR)/
	# Apply debian patches
	cd $(PKG_BUILD_DIR) && \
	for p in $$$$(cat debian/patches/series); do \
		echo "Applying $$$$p"; \
		patch -p1 < debian/patches/$$$$p || true; \
	done
	# Force all version guards to take the new API path
	# The backported headers use 6.12 APIs but LINUX_VERSION_CODE is 6.6
	$(SED) 's|LINUX_VERSION_CODE > KERNEL_VERSION(6, 7, 0)|1|g' \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_main.c
	$(SED) 's|LINUX_VERSION_CODE >= KERNEL_VERSION(6, 7, 0)|1|g' \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_main.c
	$(SED) 's|LINUX_VERSION_CODE >= KERNEL_VERSION(6, 9, 0)|1|g' \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_main.c
	$(SED) 's|LINUX_VERSION_CODE >= KERNEL_VERSION(6, 11, 0)|1|g' \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_main.c
	$(SED) 's|LINUX_VERSION_CODE >= KERNEL_VERSION(6, 12, 0)|1|g' \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_main.c \
		$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/rwnx_radar.c
endef

define Build/Compile
	+$(KERNEL_MAKE) \
		M="$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800" \
		NOSTDINC_FLAGS=" \
			$(KERNEL_NOSTDINC_FLAGS) \
			-I$(STAGING_DIR)/usr/include/mac80211-backport \
			-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
			-I$(STAGING_DIR)/usr/include/mac80211 \
			-I$(STAGING_DIR)/usr/include/mac80211/uapi \
			-include backport/backport.h" \
		CONFIG_PLATFORM_ROCKCHIP=y \
		CONFIG_PLATFORM_UBUNTU=n \
		KBUILD_EXTRA_SYMBOLS="$(shell find $(KERNEL_BUILD_DIR)/.. -path '*/symvers/*.symvers' 2>/dev/null)" \
		USER_EXTRA_CFLAGS="-w" \
		KCFLAGS="-w" \
		modules
endef

$(eval $(call KernelPackage,aic8800-usb))
