# Maintainer: Echo J. <aidas957 at gmail dot com>
# shellcheck shell=bash disable=SC2034,SC2164

pkgname=vulkan-nouveau-git
pkgdesc="Nouveau Vulkan (NVK) EXPERIMENTAL Mesa driver with some additions (Git version)"
pkgver=24.0.branchpoint.r1743.g876db91
pkgrel=1
arch=('x86_64')
depends=('libdrm' 'libxshmfence' 'libx11' 'systemd-libs' 'vulkan-icd-loader' 'wayland')
makedepends=('elfutils' 'git' 'glslang' 'libunwind' 'libxrandr' 'meson>=1.3.0rc2' 'python-mako'
             'rust' 'rust-bindgen' 'systemd' 'valgrind' 'wayland-protocols' 'xorgproto' 'zstd') # -rc1 has weird crate issues
optdepends=('vulkan-mesa-layers: Additional Vulkan layers'
            'linux>=6.6.arch1: Minimum required kernel for new uAPI support'
            'lib32-vulkan-nouveau-git: 32-bit application support')
provides=('vulkan-driver')
url="https://gitlab.freedesktop.org/mesa/mesa"
license=('MIT AND BSD-3-Clause AND SGI-B-2.0')
source=("git+${url}.git"
        nak-iadd3-imad.patch
        nvk-memory-budget.patch
        nvk-vr-support.patch
        LICENSE)
sha512sums=('SKIP'
            '6c4ed4c9c7dce79debb77cd9b828f628088101936c4e2b2994e56723f86e61799b278a9333f08813082d0a4153ac41870669da8ac47106aa20c7fc7dee8812e8'
            '4da33481cf0c4ccab193ff4fff652a98ef7940b10d12c0e3a69280236ddc9463c7813d390c5461af0478f98a6f223736a1eec14986d72a4b0f51edb28e713d95'
            'dd4c1cd059fa103b2effb3d835cf6485fb4e294c1217a94482052f803a7e88d63f452db7e1cc2d86656bf7b3340606ab685578d346f157d19e35f1b283bd04f3'
            'f9f0d0ccf166fe6cb684478b6f1e1ab1f2850431c06aa041738563eb1808a004e52cdec823c103c9e180f03ffc083e95974d291353f0220fe52ae6d4897fecc7')
install="${pkgname}.install"

prepare() {
  # HACK: Don't copy Mesa defaults (they're basically useless for this standalone driver)
  # TODO: replace this with a build option if possible
  cd mesa
  sed -i 's/install_data/#install_data/' src/util/meson.build

  # HACK: Disable xcb-util-keysyms dependency
  # It's only used for a RADV-specific trace feature so it's useless for us
  sed -i 's/with_xcb_keysyms = dep_xcb_keysyms.found()/with_xcb_keysyms = false/' meson.build

  # Set some common patch command options
  _patch_opts="--no-backup-if-mismatch -Np1 -i"

  ### DXVK v2.0+ FIRE FESTIVAL (that is somehow working) ###

  # HACK: Always expose Vulkan memory model/Vulkan 1.3
  # NAK does properly support those now but the compiler is still WIP for pre-Volta GPUs (so I'll enable these at the cost of CTS tests)
  sed -i 's/KHR_vulkan_memory_model = nvk_use_nak(info)/KHR_vulkan_memory_model = true/' src/nouveau/vulkan/nvk_physical_device.c
  sed -i 's/vulkanMemoryModel = nvk_use_nak(info)/vulkanMemoryModel = true/' src/nouveau/vulkan/nvk_physical_device.c
  sed -i 's/VK_MAKE_VERSION(1, 0/VK_MAKE_VERSION(1, 3/' src/nouveau/vulkan/nvk_physical_device.c

  ### Misc stuff ###

  # Add EXT_memory_budget (https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/172)
  # (fixes a vulkaninfo warning)
  patch ${_patch_opts} ../nvk-memory-budget.patch

  # Add imad/iadd3 support (https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27159)
  # (improves performance greatly in certain cases)
  patch ${_patch_opts} ../nak-iadd3-imad.patch

  # Add support for VR-related extensions (https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27654)
  # (seems to work with xrgears and Monado direct mode)
  patch ${_patch_opts} ../nvk-vr-support.patch

  # Add ESO/GPL support (https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/27024)
  # (enables shader precompile in DXVK and some Vulkan games; helps greatly in Overwatch 2)
  # TODO: Add this once fatal driver crashes on Overwatch 2 are fixed (I get a lot of BAR errors)
  rm src/vulkan/runtime/vk_shader.c src/vulkan/runtime/vk_shader.h || true
  #patch ${_patch_opts} ../nvk-eso-gpl.patch

  # Mark this NVK package with a signature (so I could track who's using it for bug report purposes)
  sed -i 's/"Mesa " PACKAGE_VERSION/"Mesa DodoNVK " PACKAGE_VERSION/' src/nouveau/vulkan/nvk_physical_device.c
}

pkgver() {
  cd mesa
  git describe --long --tags --abbrev=7 | sed 's/\([^-]*-g\)/r\1/;s/-/./g'
}

build() {
  # Auto-download Rust crates for NAK (removes extra code for crate handling)
  _nak_crate="--force-fallback-for=syn"

  # HACK: Remove crate .rlib files before build
  # (This prevents build errors after a Rust update: https://github.com/mesonbuild/meson/issues/10706)
  [ -d build/subprojects ] && find build/subprojects -iname "*.rlib" -delete
  [ -d build/src/nouveau/compiler ] && find build/src/nouveau/compiler -iname "*.rlib" -delete

  # As you can see, I optimized the build options pretty well 🐸
  arch-meson mesa build \
    --reconfigure \
    --wrap-mode=nofallback \
    ${_nak_crate} \
    -D b_ndebug=false \
    -D platforms=x11,wayland \
    -D gallium-drivers= \
    -D vulkan-drivers=nouveau-experimental \
    -D vulkan-layers= \
    -D dri3=enabled \
    -D egl=disabled \
    -D gallium-extra-hud=false \
    -D gallium-nine=false \
    -D gallium-omx=disabled \
    -D gallium-opencl=disabled \
    -D gallium-va=disabled \
    -D gallium-vdpau=disabled \
    -D gallium-xa=disabled \
    -D gbm=disabled \
    -D gles1=disabled \
    -D gles2=disabled \
    -D glvnd=false \
    -D glx=disabled \
    -D libunwind=enabled \
    -D llvm=disabled \
    -D lmsensors=disabled \
    -D osmesa=false \
    -D shared-glapi=disabled \
    -D microsoft-clc=disabled \
    -D valgrind=enabled \
    -D android-libbacktrace=disabled

  meson compile -C build
}

package() {
  meson install -C build --destdir "${pkgdir}"

  install -m644 -Dt "${pkgdir}/usr/share/licenses/${pkgname}" LICENSE
}
