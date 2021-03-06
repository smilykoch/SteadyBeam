#
# Copyright (C) 2006-2016 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#
include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/image.mk

define Build/Clean
	$(MAKE) -C lzma-loader clean
endef

define Image/Prepare
	# Optimized LZMA compression (with dictionary), handled by lzma-loader.
	cat $(KDIR)/vmlinux | $(STAGING_DIR_HOST)/bin/lzma e -si -so -eos -lc1 -lp2 -pb2 > $(KDIR)/vmlinux.lzma

	# Less optimal LZMA compression (no dictionary), handled by CFE.
	$(STAGING_DIR_HOST)/bin/lzma e -so -d16 $(KDIR)/vmlinux > $(KDIR)/vmlinux-nodictionary.lzma

	gzip -nc9 $(KDIR)/vmlinux > $(KDIR)/vmlinux.gz
ifneq ($(CONFIG_TARGET_ROOTFS_INITRAMFS),)
	cat $(KDIR)/vmlinux-initramfs | $(STAGING_DIR_HOST)/bin/lzma e -si -so -eos -lc1 -lp2 -pb2 > $(KDIR)/vmlinux-initramfs.lzma
	$(STAGING_DIR_HOST)/bin/lzma e -so -d16 $(KDIR)/vmlinux-initramfs > $(KDIR)/vmlinux-initramfs-nodictionary.lzma
endif
	rm -f $(KDIR)/loader.gz
	$(MAKE) -C lzma-loader \
		BUILD_DIR="$(KDIR)" \
		TARGET="$(KDIR)" \
		clean install
	echo -ne "\\x00" >> $(KDIR)/loader.gz
	rm -f $(KDIR)/fs_mark
	echo -ne '\xde\xad\xc0\xde' > $(KDIR)/fs_mark
	$(call prepare_generic_squashfs,$(KDIR)/fs_mark)
endef

define trxalign/jffs2-128k
-a 0x20000 -f $(KDIR)/root.$(1)
endef
define trxalign/jffs2-64k
-a 0x10000 -f $(KDIR)/root.$(1)
endef
define trxalign/squashfs
-a 1024 -f $(KDIR)/root.$(1) $(if $(2),-f $(2)) -a 0x10000 -A $(KDIR)/fs_mark
endef

#################################################
# Images
#################################################

define Build/trx-with-loader
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $@ \
		-f $(KDIR)/loader.gz \
		-f $(word 1,$^) \
		$(call trxalign/$(FILESYSTEM),$(FILESYSTEM))
endef

define Build/trx-v2-with-loader
	$(STAGING_DIR_HOST)/bin/trx \
		-2 \
		-m 33554432 \
		-o $@.new \
		-f $(KDIR)/loader.gz \
		-f $(KDIR)/vmlinux.lzma \
		$(call trxalign/$(FILESYSTEM),$(FILESYSTEM),$@)
	mv $@.new $@
endef

define Build/trx-without-loader
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $@ \
		-f $(word 1,$^) \
		$(call trxalign/$(FILESYSTEM),$(FILESYSTEM))
endef

define Build/asus-trx
	$(STAGING_DIR_HOST)/bin/asustrx -p "$(PRODUCTID)" -i $@ -o $@.new
	mv $@.new $@
endef

define Build/edimax-bin
	$(STAGING_DIR_HOST)/bin/trx2edips $@ $@.new
	mv $@.new $@
endef

define Build/huawei-bin
	dd if=/dev/zero of=$@.new bs=92 count=1
	echo -ne 'HDR0\x08\x00\x00\x00' >> $@.new
	cat $@ >> $@.new
	mv $@.new $@
endef

define Build/linksys-bin
	$(STAGING_DIR_HOST)/bin/addpattern -4 -p $(DEVICE_ID) -v v$(VERSION) $(if $(SERIAL),-s $(SERIAL)) -i $@ -o $@.new
	mv $@.new $@
endef

define Build/linksys-pattern-partition
	$(STAGING_DIR_HOST)/bin/addpattern -5 -p $(DEVICE_ID) -v v$(VERSION) $(if $(SERIAL),-s $(SERIAL)) -i /dev/null -o $@
endef

define Build/motorola-bin
	$(STAGING_DIR_HOST)/bin/motorola-bin -$(MOTOROLA_DEVICE) $@ $@.new
	mv $@.new $@
endef

define Build/prepend-with-elf
	mv $@ $@.old
	dd if=$(KDIR)/loader.elf of=$@ bs=131072 conv=sync
	cat $@.old >> $@
endef

define Build/tailed-bin
	echo $(BIN_TAIL) >> $@
endef

define Build/usrobotics-bin
	$(STAGING_DIR_HOST)/bin/trx2usr $@ $@.new
	mv $@.new $@
endef

#################################################
# Devices
#################################################

DEVICE_VARS += PRODUCTID
DEVICE_VARS += DEVICE_ID VERSION SERIAL
DEVICE_VARS += NETGEAR_BOARD_ID NETGEAR_REGION
DEVICE_VARS += MOTOROLA_DEVICE
DEVICE_VARS += BIN_TAIL

define Device/Default
	KERNEL := kernel-bin
	IMAGE_NAME = $$(IMAGE_PREFIX)-$$(1).$$(2)
	KERNEL_NAME = vmlinux.lzma
	FILESYSTEMS := $(FS_64K)
	IMAGES := trx
	IMAGE/trx := trx-with-loader
endef

define Device/asus
	IMAGES := trx
	IMAGE/trx := trx-with-loader | asus-trx
endef

define AsusDevice
  define Device/asus-$(1)
	$$(Device/asus)
	PRODUCTID := $(2)
  endef
  TARGET_DEVICES += asus-$(1)
endef

define Device/linksys
	IMAGES := bin
	IMAGE/bin := trx-with-loader | linksys-bin
endef

define LinksysDevice
  define Device/linksys-$(1)
	$$(Device/linksys)
	DEVICE_ID := $(2)
	VERSION := $(3)
  endef
  TARGET_DEVICES += linksys-$(1)
endef

define Device/motorola
	IMAGES := bin
	IMAGE/bin := trx-with-loader | motorola-bin
endef

define MotorolaDevice
  define Device/motorola-$(1)
	$$(Device/motorola)
	MOTOROLA_DEVICE := $(2)
  endef
  TARGET_DEVICES += motorola-$(1)
endef

define Device/netgear
	IMAGES := chk
	IMAGE/chk := trx-with-loader | netgear-chk
endef

define NetgearDevice
  define Device/netgear-$(1)
	$$(Device/netgear)
	NETGEAR_BOARD_ID := $(2)
	NETGEAR_REGION := $(3)
  endef
  TARGET_DEVICES += netgear-$(1)
endef

#################################################
# Subtarget generic
#################################################

ifeq ($(SUBTARGET),generic)
  # BCM4705 with tg3
  $(eval $(call LinksysDevice,wrt300n-v1.1,EWC2,1.51.2))
  $(eval $(call LinksysDevice,wrt310n-v1,310N,1.0.10))
  $(eval $(call LinksysDevice,wrt350n-v1,EWCG,1.04.1))
  $(eval $(call LinksysDevice,wrt610n-v1,610N,1.0.1))

  # BCMA SoC with SSB WiFi
  $(eval $(call LinksysDevice,wrt610n-v2,610N,2.0.0))
  $(eval $(call LinksysDevice,e3000-v1,61XN,1.0.3))
endif

#################################################
# Subtarget legacy
#################################################

define Device/dlink-dwl-3150
	IMAGES := bin
	IMAGE/bin := trx-with-loader | tailed-bin
	BIN_TAIL := BCM-5352-2050-0000000-01
endef

define Device/edimax-ps1208-mfg
	IMAGES := bin
	IMAGE/bin := trx-with-loader | edimax-bin
endef

define Device/huawei-e970
	KERNEL_NAME = vmlinux.gz
	IMAGES := bin
	IMAGE/bin := trx-without-loader | huawei-bin
endef

define Device/linksys-wrt54g3gv2-vf
	FILESYSTEMS := $(FS_128K)
	IMAGES := noheader.bin bin
	IMAGE/noheader.bin := linksys-pattern-partition | trx-v2-with-loader
	IMAGE/bin := linksys-pattern-partition | trx-v2-with-loader | linksys-bin
	DEVICE_ID := 3G2V
	VERSION := 3.00.24
	SERIAL := 6
endef

define Device/linksys-wrt54gs
	$(Device/linksys)
	FILESYSTEMS := $(FS_128K)
	DEVICE_ID := W54S
	VERSION := 4.80.1
endef

define Device/linksys-wrtsl54gs
	$(Device/linksys)
	FILESYSTEMS := $(FS_128K)
	DEVICE_ID := W54U
	VERSION := 2.08.1
endef

define Device/netgear-wgt634u
	FILESYSTEMS := $(FS_128K)
	IMAGES := bin
	IMAGE/bin := trx-with-loader | prepend-with-elf
endef

define Device/usrobotics-usr5461
	IMAGES := bin
	IMAGE/bin := trx-with-loader | usrobotics-bin
endef

ifeq ($(SUBTARGET),legacy)
  TARGET_DEVICES += \
	dlink-dwl-3150 \
	edimax-ps1208-mfg \
	huawei-e970 \
	linksys-wrt54g3gv2-vf \
	linksys-wrt54gs \
	linksys-wrtsl54gs \
	netgear-wgt634u \
	usrobotics-usr5461

  $(eval $(call AsusDevice,wl-300g,WL300g      ))
  $(eval $(call AsusDevice,wl-320gp,WL320gP     ))
  $(eval $(call AsusDevice,wl-330ge,WL-330gE    ))
  $(eval $(call AsusDevice,wl-500gp-v1,WL500gp     ))
  $(eval $(call AsusDevice,wl-500gp-v2,WL500gpv2   ))
  $(eval $(call AsusDevice,wl-500w,WL500W      ))
  $(eval $(call AsusDevice,wl-520gu,WL520gu     ))
  $(eval $(call AsusDevice,wl-550ge,WL550gE     ))
  $(eval $(call AsusDevice,wl-hdd25,WLHDD       ))
  $(eval $(call LinksysDevice,wrt54g3g,W54F,2.20.1))
  $(eval $(call LinksysDevice,wrt54g3g-em,W3GN,2.20.1))
  $(eval $(call LinksysDevice,wrt54g,W54G,4.71.1))
  $(eval $(call LinksysDevice,wrt54gs-v4,W54s,1.09.1))
  $(eval $(call LinksysDevice,wrt150n,N150,1.51.3))
  $(eval $(call LinksysDevice,wrt160n-v1,N150,1.50.1))
  $(eval $(call LinksysDevice,wrt300n-v1,EWCB,1.03.6))
  $(eval $(call MotorolaDevice,wa840g,2))
  $(eval $(call MotorolaDevice,we800g,3))
  $(eval $(call MotorolaDevice,wr850g,1))
  $(eval $(call NetgearDevice,wgr614-v8,U12H072T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wndr3300-v1,U12H093T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr834b-v2,U12H081T00_NETGEAR,2))
endif

#################################################
# Subtarget mips74k
#################################################

ifeq ($(SUBTARGET),mips74k)
  $(eval $(call AsusDevice,rt-ac53u,RT-AC53U))
# $(eval $(call AsusDevice,rt-ac66u,RT-AC66U))
  $(eval $(call AsusDevice,rt-n10,RT-N10      ))
  $(eval $(call AsusDevice,rt-n10p,RT-N10P))
  $(eval $(call AsusDevice,rt-n10p-v2,RT-N10PV2))
  $(eval $(call AsusDevice,rt-n10u,RT-N10U))
  $(eval $(call AsusDevice,rt-n10u-b,RT-N10U))
  $(eval $(call AsusDevice,rt-n12,RT-N12      ))
  $(eval $(call AsusDevice,rt-n12-b1,RT-N12B1))
  $(eval $(call AsusDevice,rt-n12-c1,RT-N12C1))
  $(eval $(call AsusDevice,rt-n12-d1,RT-N12D1))
  $(eval $(call AsusDevice,rt-n12hp,RT-N12HP))
  $(eval $(call AsusDevice,rt-n14uhp,RT-N14UHP))
  $(eval $(call AsusDevice,rt-n15u,RT-N15U))
  $(eval $(call AsusDevice,rt-n16,RT-N16))
  $(eval $(call AsusDevice,rt-n53,RT-N53))
  $(eval $(call AsusDevice,rt-n66u,RT-N66U))
  $(eval $(call AsusDevice,rt-n66w,RT-N66U))
  $(eval $(call LinksysDevice,wrt160n-v3,N150,3.0.3))
  $(eval $(call LinksysDevice,wrt310n-v2,310N,2.0.1))
  $(eval $(call LinksysDevice,wrt320n-v1,320N,1.0.5))
  $(eval $(call LinksysDevice,e900-v1,E900,1.0.4))
  $(eval $(call LinksysDevice,e1000-v1-v2-v2.1,E100,1.1.3))
  $(eval $(call LinksysDevice,e1200-v1,E120,1.0.3))
  $(eval $(call LinksysDevice,e1200-v2,E122,1.0.4))
  $(eval $(call LinksysDevice,e1500-v1,E150,1.0.5))
  $(eval $(call LinksysDevice,e1550-v1,1550,1.0.3))
  $(eval $(call LinksysDevice,e2000-v1,32XN,1.0.4))
  $(eval $(call LinksysDevice,e2500-v1,E25X,1.0.7))
  $(eval $(call LinksysDevice,e2500-v2,E25X,2.0.0))
  $(eval $(call LinksysDevice,e2500-v2.1,25RU,2.1.0))
  $(eval $(call LinksysDevice,e2500-v3,25V3,3.0.0))
  $(eval $(call LinksysDevice,e3200-v1,3200,1.0.1))
  $(eval $(call LinksysDevice,e4200-v1,4200,1.0.5))
  $(eval $(call NetgearDevice,wgr614-v10_north-america,U12H139T01_NETGEAR,2))
  $(eval $(call NetgearDevice,wgr614-v10_other-regions,U12H139T01_NETGEAR,1))
  $(eval $(call NetgearDevice,wn2500rp-v1,U12H197T00_NETGEAR,1))
  $(eval $(call NetgearDevice,wndr3400-v1,U12H155T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wndr3400-v2,U12H187T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wndr3400-v3,U12H208T00_NETGEAR,1))
  $(eval $(call NetgearDevice,wndr3700-v3,U12H194T00_NETGEAR,2))
#  $(eval $(call NetgearDevice,wndr3400-vcna,U12H155T01_NETGEAR,2))
  $(eval $(call NetgearDevice,wndr4000,U12H181T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr1000-v3,U12H139T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr2000v2,U12H114T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr3000rp,U12H163T01_NETGEAR,1))
  $(eval $(call NetgearDevice,wnr3500l-v1-north-america,U12H136T99_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr3500l-v1-other-regions,U12H136T99_NETGEAR,1))
  $(eval $(call NetgearDevice,wnr3500l-v2,U12H172T00_NETGEAR,1))
#  $(eval $(call NetgearDevice,wnr3500u,U12H136T00_NETGEAR,2))
  $(eval $(call NetgearDevice,wnr3500-v2,U12H127T00_NETGEAR,2))
#  $(eval $(call NetgearDevice,wnr3500-v2-vc,U12H127T70_NETGEAR,2))
endif

#################################################
# Shared BuildImage defines
#################################################

define Image/Build/Initramfs
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $(BIN_DIR)/$(IMG_PREFIX)-initramfs.trx \
		-f $(KDIR)/loader.gz \
		-f $(KDIR)/vmlinux-initramfs.lzma
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $(BIN_DIR)/$(IMG_PREFIX)-initramfs-noloader-nodictionary.trx \
		-f $(KDIR)/vmlinux-initramfs-nodictionary.lzma
endef

# $(1): filesystem type.
define Image/Build
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $(BIN_DIR)/$(IMG_PREFIX)-$(1).trx \
		-f $(KDIR)/loader.gz -f $(KDIR)/vmlinux.lzma \
		$(call trxalign/$(1),$(1))
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $(BIN_DIR)/$(IMG_PREFIX)-$(1)-noloader-nodictionary.trx \
		-f $(KDIR)/vmlinux-nodictionary.lzma \
		$(call trxalign/$(1),$(1))
	$(STAGING_DIR_HOST)/bin/trx \
		-m 33554432 \
		-o $(BIN_DIR)/$(IMG_PREFIX)-$(1)-gz.trx \
		-f $(KDIR)/vmlinux.gz \
		$(call trxalign/$(1),$(1))
ifneq ($(CONFIG_TARGET_ROOTFS_INITRAMFS),)
	$(call Image/Build/Initramfs)
endif
endef

$(eval $(call BuildImage))
