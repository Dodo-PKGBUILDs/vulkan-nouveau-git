# Maintainer: Echo J. <aidas957 at gmail dot com>
# shellcheck shell=bash disable=SC2034,SC2164

pkgname=vulkan-nouveau-git
pkgdesc="Nouveau Vulkan (NVK) EXPERIMENTAL Mesa driver with some additions (Git version)"
pkgver=23.3.branchpoint.r1282.g03a7cb2
pkgrel=1
arch=('x86_64')
depends=('libdrm' 'libxshmfence' 'libx11' 'systemd-libs' 'vulkan-icd-loader' 'wayland')
makedepends=('elfutils' 'git' 'glslang' 'libunwind' 'libxrandr' 'meson>=1.3.0rc2' 'python-mako'
             'rust' 'rust-bindgen' 'systemd' 'valgrind' 'wayland-protocols' 'xorgproto' 'zstd') # -rc1 has weird crate issues
optdepends=('vulkan-mesa-layers: Additional Vulkan layers'
            'linux>=6.6.arch1: Minimum required kernel for new uAPI support')
provides=('vulkan-driver')
url="https://gitlab.freedesktop.org/mesa/mesa"
license=('custom')
source=("git+${url}.git"
        nvk-memory-budget.patch
        nvk-pipeline-cache.patch
        nvk-synchr2-memmodel.patch
        nvk-vulkan11.patch
        LICENSE)
sha512sums=('SKIP'
            '770d195f571aabc0e9dddf254576c29bbfff34ff0af0edfb6ede9864d25ef12247f2f5afd770d5ca70e8a9ac900623b92892211d73bd8bd4075d95c012367742'
            'c8918bbee4849fb481aeb17800f435e35d695a1f06ded8b3ce6ca8d0d7cf27c05d77c90160c6ef021d57766d0dd10da5c0fa20e12c3b445e3115a79908d71b10'
            '48a3de59f4528548a7df3f8582cb922057f10b8ac3b5cf4a3f664e6ee926b94327860190b54cd976048b04a14b438778c206dbb4b417d69428aea86b621dc219'
            '8f308ef9cb662613a537d1cbb2e72a9d7dcf9186cfc9fd4dbdaf9d0e0fdeae5d4d4e4c47d2b4a5da9c5a77b56c29d5a3222ea7404bb98b63c1c61f7085953df9'
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

  ### Required patches for DXVK (up to v1.10.3)/PCSX2 to work (WIP) ###

  # Advertise Vulkan 1.1 to expose functions required for PCSX2/DXVK v1.5.2+
  # (https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/251)
  patch ${_patch_opts} ../nvk-vulkan11.patch

  # Set ICD version to 5 (required to get rid of a loader warning)
  sed -i 's/*pSupportedVersion, 4u/*pSupportedVersion, 5u/' src/nouveau/vulkan/nvk_instance.c

  ### DXVK v2.0+ FIRE FESTIVAL (that is somehow working) ###

  # HACK turned up to 11: Advertise Vulkan 1.3 support
  sed -i 's/VK_MAKE_VERSION(1, 1/VK_MAKE_VERSION(1, 3/' src/nouveau/vulkan/nvk_instance.c
  sed -i 's/VK_MAKE_VERSION(1, 1/VK_MAKE_VERSION(1, 3/' src/nouveau/vulkan/nvk_physical_device.c

  # Expose Vulkan memory model/synchronization2
  # I highly doubt this passes CTS (and it doesn't) but it works well enough for DXVK
  patch ${_patch_opts} ../nvk-synchr2-memmodel.patch

  ### Misc stuff ###

  # Add EXT_memory_budget (https://gitlab.freedesktop.org/nouveau/mesa/-/merge_requests/172)
  # (fixes a vulkaninfo warning)
  patch ${_patch_opts} ../nvk-memory-budget.patch

  # Pipeline caching (https://gitlab.freedesktop.org/mesa/mesa/-/merge_requests/25550)
  # (might improve performance) (this patch is rebased for the NVK shader code rework)
  patch ${_patch_opts} ../nvk-pipeline-cache.patch

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

  ninja -C build
  meson compile -C build
}

package() {
  meson install -C build --destdir "${pkgdir}"

  install -m644 -Dt "${pkgdir}/usr/share/licenses/${pkgname}" LICENSE
}