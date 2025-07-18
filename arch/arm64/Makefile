#
# arch/arm64/Makefile
#
# This file is included by the global makefile so that you can add your own
# architecture-specific flags and dependencies.
#
# This file is subject to the terms and conditions of the GNU General Public
# License.  See the file "COPYING" in the main directory of this archive
# for more details.
#
# Copyright (C) 1995-2001 by Russell King

LDFLAGS_vmlinux	:=--no-undefined -X -z norelro
CPPFLAGS_vmlinux.lds = -DTEXT_OFFSET=$(TEXT_OFFSET)
GZFLAGS		:=-9

ifeq ($(CONFIG_RELOCATABLE), y)
# Pass --no-apply-dynamic-relocs to restore pre-binutils-2.27 behaviour
# for relative relocs, since this leads to better Image compression
# with the relocation offsets always being zero.
LDFLAGS_vmlinux		+= -Bsymbolic -z notext \
			$(call ld-option, --no-apply-dynamic-relocs)

# Pass -shared for bfd and -pie for lld to minimize Image size on both
ifeq ($(ld-name),bfd)
LDFLAGS_vmlinux		+= -shared
endif
ifeq ($(ld-name),lld)
LDFLAGS_vmlinux		+= -pie
endif
endif

ifeq ($(CONFIG_ARM64_ERRATUM_843419),y)
  ifeq ($(call ld-option, --fix-cortex-a53-843419),)
$(warning ld does not support --fix-cortex-a53-843419; kernel may be susceptible to erratum)
  else
    ifeq ($(call gold-ifversion, -lt, 114000000, y), y)
$(warning This version of GNU gold may generate incorrect code with --fix-cortex-a53-843419;\
	see https://sourceware.org/bugzilla/show_bug.cgi?id=21491)
    endif
LDFLAGS_vmlinux	+= --fix-cortex-a53-843419
  endif
else
  ifeq ($(ld-name),gold)
# Pass --no-fix-cortex-a53-843419 to ensure the erratum fix is disabled
LDFLAGS	+= --no-fix-cortex-a53-843419
  endif
endif

KBUILD_DEFCONFIG := defconfig

# Check for binutils support for specific extensions
lseinstr := $(call as-instr,.arch_extension lse,-DCONFIG_AS_LSE=1)

ifeq ($(CONFIG_ARM64_LSE_ATOMICS), y)
  ifeq ($(lseinstr),)
$(warning LSE atomics not supported by binutils)
  endif
endif

ifeq ($(CONFIG_ARM64), y)
brokengasinst := $(call as-instr,1:\n.inst 0\n.rept . - 1b\n\nnop\n.endr\n,,-DCONFIG_BROKEN_GAS_INST=1)

  ifneq ($(brokengasinst),)
$(warning Detected assembler with broken .inst; disassembly will be unreliable)
  endif
endif

ifeq ($(CONFIG_COMPAT_VDSO), y)
  CROSS_COMPILE_ARM32 ?= $(CONFIG_CROSS_COMPILE_ARM32:"%"=%)

  # Check that the user has provided a valid prefix for the 32-bit toolchain.
  # To prevent selecting the system $(cc-name) by default, the prefix is not
  # allowed to be empty, unlike CROSS_COMPILE. In the unlikely event that the
  # system $(cc-name) is actually the 32-bit ARM compiler to be used, the
  # variable can be set to the dirname (e.g. CROSS_COMPILE_ARM32=/usr/bin/).
  # Note: this Makefile is read both before and after regenerating the config
  # (if needed). Any warning appearing before the config has been regenerated
  # should be ignored. If the error is triggered and you set
  # CONFIG_CROSS_COMPILE_ARM32, set CROSS_COMPILE_ARM32 to an appropriate value
  # when invoking make and fix CONFIG_CROSS_COMPILE_ARM32.
  ifeq ($(CROSS_COMPILE_ARM32),)
    $(error CROSS_COMPILE_ARM32 not defined or empty, the compat vDSO will not be built)
  else ifeq ($(cc-name),clang)
    export CLANG_TRIPLE_ARM32 ?= $(CROSS_COMPILE_ARM32)
    export CLANG_TARGET_ARM32 := --target=$(notdir $(CLANG_TRIPLE_ARM32:%-=%))
    export GCC_TOOLCHAIN32_DIR := $(dir $(shell which $(CROSS_COMPILE_ARM32)ld))
    export GCC_TOOLCHAIN32 := $(realpath $(GCC_TOOLCHAIN32_DIR)/..)
    export CLANG_PREFIX32 := --prefix=$(GCC_TOOLCHAIN32_DIR)$(notdir $(CROSS_COMPILE_ARM32))
    export CLANG_GCC32_TC := --gcc-toolchain=$(GCC_TOOLCHAIN32)
    export CONFIG_VDSO32 := y
    vdso32 := -DCONFIG_VDSO32=1
  else ifeq ($(shell which $(CROSS_COMPILE_ARM32)$(cc-name) 2> /dev/null),)
    $(error $(CROSS_COMPILE_ARM32)$(cc-name) not found, check CROSS_COMPILE_ARM32)
  else
    export CROSS_COMPILE_ARM32
    export CONFIG_VDSO32 := y
    vdso32 := -DCONFIG_VDSO32=1
  endif
endif

ifeq ($(cc-name),clang)
# This is a workaround for https://bugs.llvm.org/show_bug.cgi?id=30792.
# TODO: revert when this is fixed in LLVM.
KBUILD_CFLAGS	+= -mno-implicit-float
else
KBUILD_CFLAGS	+= -mgeneral-regs-only
endif
KBUILD_CFLAGS	+= $(lseinstr) $(brokengasinst) $(vdso32)
#ifdef CONFIG_MACH_MT6768
CL_FLAGS += -O3 -mcpu=cortex-a55+crypto+crc
GC_FLAGS += -O3 -mcpu=cortex-a75.cortex-a55+crypto+crc
ifeq ($(cc-name),gcc)
KBUILD_CFLAGS  += $(GC_FLAGS)
KBUILD_AFLAGS  += $(GC_FLAGS)
KBUILD_LDFLAGS += $(GC_FLAGS)
endif
ifeq ($(cc-name),clang)
KBUILD_CFLAGS  += $(CL_FLAGS)
KBUILD_AFLAGS  += $(CL_FLAGS)
KBUILD_LDFLAGS += $(CL_FLAGS)
endif
#endif
KBUILD_CFLAGS	+= -fno-asynchronous-unwind-tables
KBUILD_CFLAGS	+= $(call cc-option, -mpc-relative-literal-loads)
KBUILD_CFLAGS	+= -fno-pic
KBUILD_AFLAGS	+= $(lseinstr) $(brokengasinst) $(vdso32)

KBUILD_CFLAGS	+= $(call cc-option,-mabi=lp64)
KBUILD_AFLAGS	+= $(call cc-option,-mabi=lp64)

ifeq ($(CONFIG_SHADOW_CALL_STACK), y)
KBUILD_CFLAGS	+= -ffixed-x18
endif

ifeq ($(CONFIG_CPU_BIG_ENDIAN), y)
KBUILD_CPPFLAGS	+= -mbig-endian
CHECKFLAGS	+= -D__AARCH64EB__
AS		+= -EB
LD		+= -EB
ifeq ($(ld-name),gold)
LDFLAGS		+= -maarch64_elf64_be_vec
else
LDFLAGS		+= -maarch64elfb
endif
UTS_MACHINE	:= aarch64_be
else
KBUILD_CPPFLAGS	+= -mlittle-endian
CHECKFLAGS	+= -D__AARCH64EL__
AS		+= -EL
LD		+= -EL
ifeq ($(ld-name),gold)
LDFLAGS		+= -maarch64_elf64_le_vec
else
LDFLAGS		+= -maarch64elf
endif
UTS_MACHINE	:= aarch64
endif

CHECKFLAGS	+= -D__aarch64__ -m64

ifeq ($(CONFIG_ARM64_MODULE_CMODEL_LARGE), y)
KBUILD_CFLAGS_MODULE	+= -mcmodel=large
ifeq ($(CONFIG_LTO_CLANG), y)
# Code model is not stored in LLVM IR, so we need to pass it also to LLVMgold
KBUILD_LDFLAGS_MODULE	+= -plugin-opt=-code-model=large
endif
endif

ifeq ($(CONFIG_ARM64_MODULE_PLTS),y)
KBUILD_LDFLAGS_MODULE	+= -T $(srctree)/arch/arm64/kernel/module.lds
endif

# Default value
head-y		:= arch/arm64/kernel/head.o

# The byte offset of the kernel image in RAM from the start of RAM.
ifeq ($(CONFIG_ARM64_RANDOMIZE_TEXT_OFFSET), y)
TEXT_OFFSET := $(shell awk "BEGIN {srand(); printf \"0x%06x\n\", \
		 int(2 * 1024 * 1024 / (2 ^ $(CONFIG_ARM64_PAGE_SHIFT)) * \
		 rand()) * (2 ^ $(CONFIG_ARM64_PAGE_SHIFT))}")
else
TEXT_OFFSET := 0x00080000
endif

ifeq ($(cc-name),clang)
KBUILD_CFLAGS += $(call cc-disable-warning, asm-operand-widths)
endif

ifeq ($(CONFIG_KASAN_SW_TAGS), y)
KASAN_SHADOW_SCALE_SHIFT := 4
else
KASAN_SHADOW_SCALE_SHIFT := 3
endif

KBUILD_CFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_CPPFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)
KBUILD_AFLAGS += -DKASAN_SHADOW_SCALE_SHIFT=$(KASAN_SHADOW_SCALE_SHIFT)

# KASAN_SHADOW_OFFSET = VA_START + (1 << (VA_BITS - KASAN_SHADOW_SCALE_SHIFT))
#				 - (1 << (64 - KASAN_SHADOW_SCALE_SHIFT))
# in 32-bit arithmetic
KASAN_SHADOW_OFFSET := $(shell printf "0x%08x00000000\n" $$(( \
	(0xffffffff & (-1 << ($(CONFIG_ARM64_VA_BITS) - 32))) \
	+ (1 << ($(CONFIG_ARM64_VA_BITS) - 32 - $(KASAN_SHADOW_SCALE_SHIFT))) \
	- (1 << (64 - 32 - $(KASAN_SHADOW_SCALE_SHIFT))) )) )

export	TEXT_OFFSET GZFLAGS

core-y		+= arch/arm64/kernel/ arch/arm64/mm/
core-$(CONFIG_NET) += arch/arm64/net/
core-$(CONFIG_KVM) += arch/arm64/kvm/
core-$(CONFIG_XEN) += arch/arm64/xen/
core-$(CONFIG_CRYPTO) += arch/arm64/crypto/
libs-y		:= arch/arm64/lib/ $(libs-y)
core-$(CONFIG_EFI_STUB) += $(objtree)/drivers/firmware/efi/libstub/lib.a

# Default target when executing plain make
boot		:= arch/arm64/boot
ifeq ($(CONFIG_BUILD_ARM64_APPENDED_DTB_IMAGE),y)
KBUILD_IMAGE	:= $(boot)/$(subst $\",,$(CONFIG_BUILD_ARM64_APPENDED_KERNEL_IMAGE_NAME))
else
KBUILD_IMAGE	:= $(boot)/Image.gz
endif

KBUILD_DTBS	:= dtbs

all:	Image.gz $(KBUILD_DTBS) $(subst $\",,$(CONFIG_BUILD_ARM64_APPENDED_KERNEL_IMAGE_NAME))

MTK_PLATFORM := $(subst ",,$(CONFIG_MTK_PLATFORM))
MTK_PROJECT := $(subst ",,$(CONFIG_ARCH_MTK_PROJECT))
export MTK_PLATFORM MTK_PROJECT

Image: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.%: Image
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

zinstall install:
	$(Q)$(MAKE) $(build)=$(boot) $@

%.dtb: scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts $(boot)/dts/$@

%.dtbo: scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts $(boot)/dts/$@

PHONY += dtbs dtbs_install

dtbs: prepare scripts
	$(Q)$(MAKE) $(build)=$(boot)/dts

dtbs_install:
	$(Q)$(MAKE) $(dtbinst)=$(boot)/dts

Image-dtb: vmlinux scripts dtbs
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

Image.gz-dtb: vmlinux scripts dtbs Image.gz
	$(Q)$(MAKE) $(build)=$(boot) $(boot)/$@

PHONY += vdso_install
vdso_install:
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso $@

# We use MRPROPER_FILES and CLEAN_FILES now
archclean:
	$(Q)$(MAKE) $(clean)=$(boot)
	$(Q)$(MAKE) $(clean)=$(boot)/dts

ifeq ($(KBUILD_EXTMOD),)
# We need to generate vdso-offsets.h before compiling certain files in kernel/.
# In order to do that, we should use the archprepare target, but we can't since
# asm-offsets.h is included in some files used to generate vdso-offsets.h, and
# asm-offsets.h is built in prepare0, for which archprepare is a dependency.
# Therefore we need to generate the header after prepare0 has been made, hence
# this hack.
prepare: vdso_prepare
vdso_prepare: prepare0
	$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso include/generated/vdso-offsets.h
	$(if $(CONFIG_VDSO32),$(Q)$(MAKE) $(build)=arch/arm64/kernel/vdso32 \
					  include/generated/vdso32-offsets.h)
endif

define archhelp
  echo  '* Image.gz      - Compressed kernel image (arch/$(ARCH)/boot/Image.gz)'
  echo  '  Image         - Uncompressed kernel image (arch/$(ARCH)/boot/Image)'
  echo  '* dtbs          - Build device tree blobs for enabled boards'
  echo  '  dtbs_install  - Install dtbs to $(INSTALL_DTBS_PATH)'
  echo  '  install       - Install uncompressed kernel'
  echo  '  zinstall      - Install compressed kernel'
  echo  '                  Install using (your) ~/bin/installkernel or'
  echo  '                  (distribution) /sbin/installkernel or'
  echo  '                  install to $$(INSTALL_PATH) and run lilo'
endef
