#
# Copyright (C) 2024 Bin Jin <bjin@protonmail.com>
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
# OpenWrt package for the phantun kernel module.
#
# Compatible target: OpenWrt 24.10+ (kernel 6.6+). Drop this directory into
# an OpenWrt buildroot's package/ (or a feed) as package/phantun/ and build
# with `make package/phantun/compile` or select kmod-phantun in menuconfig
# under Kernel modules -> Network Support.

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=phantun
PKG_VERSION:=0.4.3.1
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/Paull/phantun-dkms.git
PKG_SOURCE_DATE:=2026-09-23
PKG_SOURCE_VERSION:=7a5fb487e4d48e298031ff3e5702d160d1e3f30e
PKG_SOURCE_SUBDIR:=$(PKG_NAME)-$(PKG_VERSION)
# Paull/phantun-dkms has no tagged releases yet, so the source is fetched
# straight from a pinned commit instead of a release tarball; there is no
# stable per-release hash to pin against.
PKG_MIRROR_HASH:=skip

PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

PKG_MAINTAINER:=Bin Jin <bjin@protonmail.com>
PKG_LICENSE:=GPL-2.0-or-later
PKG_LICENSE_FILES:=LICENSE

PKG_FIXUP:=autoreconf
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk

define KernelPackage/phantun
  SUBMENU:=Network Support
  TITLE:=Phantun fake-TCP transport for UDP applications
  URL:=https://github.com/Paull/phantun-dkms
  FILES:=$(PKG_BUILD_DIR)/phantun.ko
  AUTOLOAD:=$(call AutoProbe,phantun)
endef

define KernelPackage/phantun/description
  Kernel module re-implementation of Phantun: transforms UDP streams into
  fake-TCP streams in-kernel via netfilter hooks, so that UDP applications
  such as WireGuard can traverse networks that only allow (or prioritize)
  TCP traffic, without a TUN device or userspace relay.

  All module parameters (managed_local_ports, managed_remote_peers, and
  friends) can be set through /etc/modules.d/ or /etc/modprobe.d/, or via
  UCI in /etc/config/phantun combined with /etc/init.d/phantun.
endef

define Build/Configure
	(cd $(PKG_BUILD_DIR) && \
		MAKE="make ARCH=$(LINUX_KARCH) CROSS_COMPILE=$(KERNEL_CROSS) CC=$(KERNEL_CC)" \
		CONFIG_SHELL="$(BASH)" \
		./configure --with-kernel="$(LINUX_DIR)" \
	)
endef

define Build/Compile
	$(KERNEL_MAKE) M="$(PKG_BUILD_DIR)" modules
endef

define KernelPackage/phantun/install
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) ./files/phantun.config $(1)/etc/config/phantun
	$(INSTALL_DIR) $(1)/etc/init.d
	$(INSTALL_BIN) ./files/phantun.init $(1)/etc/init.d/phantun
endef

define KernelPackage/phantun/conffiles
/etc/config/phantun
endef

$(eval $(call KernelPackage,phantun))
