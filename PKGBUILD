# Maintainer: Joe Maples <joe@maples.dev>
# Contributor: Levente Polyak <anthraxx[at]archlinux[dot]org>
# Contributor: Daniel Micay <danielmicay@gmail.com>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

## The following variables can be customized at build time
##
##  Usage:  env _microarchitecture=98 use_numa=n use_tracers=n makepkg -sc
##     or:  makepkg -sc -- _microarchitecture=98 use_numa=n use_tracers=n
##     or:  export use_numa=n use_tracers=n; makepkg -sc

## Look inside 'choose-gcc-optimization.sh' to choose your microarchitecture
## Valid numbers between: 0 to 99
## Default is: 93 => x86-64-v3
## Good option if your package is for one machine: 98 (Intel native) or 99 (AMD native)
: "${_microarchitecture:=93}"

## Compress modules by default (following Arch's kernel)
## Set variable "_compress_modules" to: n to disable
##                                      y to enable (default)
: "${_compress_modules:=y}"

# Compile ONLY used modules to VASTLY reduce the number of modules built
# and the build time.
#
# To keep track of which modules are needed for your specific system/hardware,
# give module_db script a try: https://aur.archlinux.org/packages/modprobed-db
# This PKGBUILD read the database kept if it exists
#
# More at this wiki page ---> https://wiki.archlinux.org/index.php/Modprobed-db
: "${_localmodcfg:=n}"

pkgbase=linux-hardened-xanmod-tt-rog
_srcname=${pkgbase/-git/}
_gitbranch=5.16
pkgver=5.16.11.r1060404.gb7a9f15bc280
pkgrel=1
pkgdesc='Security-Hardened Linux with Xanmod, TT Sched, and ROG patches'
url='https://github.com/anthraxx/linux-hardened'
arch=(x86_64)
license=(GPL2)
makedepends=(
  bc kmod libelf
  clang llvm lld python
  git
)
options=('!strip')
source=(
  "${_srcname}::git+https://github.com/anthraxx/linux-hardened#branch=${_gitbranch}?signed"
  config # the main kernel config files
  choose-gcc-optimization.sh
  linux-5.16.11.patch
  xanmod.patch # Xanmod patch
  tt.patch
  fix-tt-build.patch
  rog.patch
)
validpgpkeys=(
  'ABAF11C65A2970B130ABE3C479BE3E4300411886'  # Linus Torvalds
  '647F28654894E3BD457199BE38DBBDC86092693E'  # Greg Kroah-Hartman
  'E240B57E2C4630BA768E2F26FC1B547C8D8172C8'  # Levente Polyak
)
sha256sums=('SKIP'
            'b4e9c35c7ff5c5a7f61b60899cfb792794ec677a5a7c413261bbca0191cb3455'
            '278118011d7a2eeca9971ac97b31bf0c55ab55e99c662ab9ae4717b55819c9a2'
            '0eb2abe265702256c83858a8c27531d07226d71f24dd6ad66ecd9a095547bd4f'
            '97fcda12d597a98578a0a4421f0819a7a757888fe86a291d4dd10dc567333b16'
            '2528d7c98924ca883f99e642e406f4a7e53b207212a8c83527caf493273d38f3'
            '26a74fbbda76e7765761a8437325626cf4b2fcdc231380b6af57866ed071c7a4'
            'ce9b6ca13e93f6dba983c1167861baa1a2208c24d57a9a3a206f8cca18c7801e'
)

export KBUILD_BUILD_HOST=archlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

pkgver() {
  cd $_srcname
  printf "%s.%s%s%s.r%s.g%s" \
    "$(grep '^VERSION = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^PATCHLEVEL = ' Makefile|awk -F' = ' '{print $2}')" \
    "$(grep '^SUBLEVEL = ' Makefile|awk -F' = ' '{print $2}'|grep -vE '^0$'|sed 's/.*/.\0/')" \
    "$(grep '^EXTRAVERSION = ' Makefile|awk -F' = ' '{print $2}'|tr -d -|sed -E 's/hardened[0-9]+//')" \
    "$(git rev-list --count HEAD)" \
    "$(git rev-parse --short HEAD)"
}

prepare() {
  cd $_srcname

  msg2 "Setting version..."
  rm -f localversion* include/config/kernel.release
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname
  echo "-r$(git rev-list --count HEAD)" > localversion.30-revision

  local src
  for src in "${source[@]}"; do
    src="${src%%::*}"
    src="${src##*/}"
    case "$src" in
      *.patch|*.patch.xz|*.diff|*.diff.xz)
        # Apply any other patches
        msg2 "Applying patch ${src%\.xz} ..."
        patch -Np1 -i "../${src%\.xz}"
        ;;
    esac
  done

  msg2 "Setting config..."
  cp ../config .config

  # Select microarchitecture optimization target
  sh "${srcdir}/choose-gcc-optimization.sh" $_microarchitecture

  # Compress modules
  if [ "$_compress_modules" = "y" ]; then
    scripts/config --disable CONFIG_MODULE_COMPRESS_NONE \
                   --enable CONFIG_MODULE_COMPRESS_ZSTD
  fi

  ### Optionally load needed modules for the make localmodconfig
  # See https://aur.archlinux.org/packages/modprobed-db
  if [ "$_localmodcfg" = "y" ]; then
    if [ -f "$HOME/.config/modprobed.db" ]; then
      msg2 "Running Steven Rostedt's make localmodconfig now"
      make LLVM=1 LSMOD="$HOME/.config/modprobed.db" localmodconfig
    else
      msg2 "No modprobed.db data found"
      exit 1
    fi
  fi

  msg2 "Finalizing kernel config..."
  
  make LLVM=1 olddefconfig

  make LLVM=1 -s kernelrelease > version
  msg2 "Prepared $pkgbase version $(<version)"
}

build() {
  cd $_srcname
  make LLVM=1 all
}

_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(coreutils kmod initramfs)
  optdepends=('crda: to set the correct wireless channels of your country'
              'linux-firmware: firmware images needed for some devices'
              'usbctl: deny_new_usb control')
  provides=(VIRTUALBOX-GUEST-MODULES WIREGUARD-MODULE)

  cd $_srcname
  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  msg2 "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$(make -s image_name)" "$modulesdir/vmlinuz"

  # Used by mkinitcpio to name the kernel
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  msg2 "Installing modules..."
  make LLVM=1 INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 modules_install

  # remove build and source links
  rm "$modulesdir"/{source,build}
}

_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"

  cd $_srcname
  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  msg2 "Installing build files..."
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  msg2 "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  msg2 "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  msg2 "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    echo "Removing $(basename "$arch")"
    rm -r "$arch"
  done

  msg2 "Removing documentation..."
  rm -r "$builddir/Documentation"

  msg2 "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  msg2 "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  msg2 "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  msg2 "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  msg2 "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase"
}

pkgname=(linux-hardened-xanmod-tt-rog-git linux-hardened-xanmod-tt-rog-headers-git)
for _p in "${pkgname[@]}"; do
  _p=${_p/-git/}
  eval "package_$_p-git() {
    provides=(${_p})
    $(declare -f "_package${_p#linux-hardened-xanmod-tt-rog}")
    _package${_p#linux-hardened-xanmod-tt-rog}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
