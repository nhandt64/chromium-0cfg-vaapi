# Maintainer: Evangelos Foutras <evangelos@foutrelis.com>
# Contributor: Pierre Schmitz <pierre@archlinux.de>
# Contributor: Jan "heftig" Steffens <jan.steffens@gmail.com>
# Contributor: Daniel J Griffiths <ghost1227@archlinux.us>

# Building for x86_64 requires lib32-glibc & lib32-zlib from [multilib]. These
# libraries are linked from the NaCl toolchain, and are only needed during
# build time.

pkgname=chromium
pkgver=16.0.912.63
pkgrel=1
pkgdesc="The open-source project behind Google Chrome, an attempt at creating a safer, faster, and more stable browser"
arch=('i686' 'x86_64')
url="http://www.chromium.org/"
license=('BSD')
depends=('gtk2' 'dbus-glib' 'nss' 'alsa-lib' 'xdg-utils' 'bzip2' 'libevent'
         'libxss' 'libgcrypt' 'ttf-dejavu' 'desktop-file-utils'
         'hicolor-icon-theme')
makedepends=('python2' 'perl' 'gperf' 'yasm' 'mesa' 'libgnome-keyring'
             'elfutils')
# Needed for the NaCl toolchain
[[ $CARCH == x86_64 ]] && makedepends+=('lib32-zlib')
provides=('chromium-browser')
conflicts=('chromium-browser')
options=('!buildflags')
install=chromium.install
source=(http://commondatastorage.googleapis.com/chromium-browser-official/$pkgname-$pkgver.tar.bz2
        http://commondatastorage.googleapis.com/nativeclient-mirror/nacl/nacl_sdk/nacl_sdk.zip
        chromium.desktop
        chromium.sh
        gcc-4.6.patch
        fix-downloads-on-ntfs.patch)
sha256sums=('da806829adee04c0701444b1842975eec2e4b745956933d760153a414b53c588'
            '964fe3a5ec56f2505649aba00f900abe4205674b7fdaa16772647d347173bb01'
            '09bfac44104f4ccda4c228053f689c947b3e97da9a4ab6fa34ce061ee83d0322'
            'c53bfc4db9dde684fbaed6a4bbecb207e3e7a0a2703233426fe076a6d3c557f3'
            '9c5e0803904d1a0e71ab7444c92a7046a34a9518eeba7a70f2eec7abecb8bf4e'
            '6364c464d1885b2ec21076f01f993725925ccc066805f1ecbbeaf6f79b93c209')

build() {
  cd "$srcdir/chromium-$pkgver"

  # Fix build with gcc 4.6
  # http://code.google.com/p/chromium/issues/detail?id=80071
  patch -Np0 -i "$srcdir/gcc-4.6.patch"

  # Fix build with CUPS 1.5
  sed -i '/#include <cups\/cups.h>/ a #include <cups/ppd.h>' \
    chrome/browser/ui/webui/print_preview_handler.cc

  # Fix downloading on NTFS partitions
  # http://code.google.com/p/chromium/issues/detail?id=102200
  patch -Np2 -i "$srcdir/fix-downloads-on-ntfs.patch"

  # Use Python 2
  find . -type f -exec sed -i -r \
    -e 's|/usr/bin/python$|&2|g' \
    -e 's|(/usr/bin/python2)\.4$|\1|g' \
    {} +
  # There are still a lot of relative calls which need a workaround
  mkdir "$srcdir/python2-path"
  ln -s /usr/bin/python2 "$srcdir/python2-path/python"
  export PATH="$srcdir/python2-path:$PATH"

  pushd "$srcdir/nacl_sdk"
  ./naclsdk update pepper_15
  popd

  ln -s "$srcdir/nacl_sdk/pepper_15/toolchain/linux_x86_newlib" \
    native_client/toolchain/linux_x86_newlib

  # We need to disable system_ssl until "next protocol negotiation" support is
  # available in our nss package.
  # (See https://bugzilla.mozilla.org/show_bug.cgi?id=547312)

  build/gyp_chromium -f make build/all.gyp --depth=. \
    -Dno_strict_aliasing=1 \
    -Dwerror= \
    -Dlinux_sandbox_path=/usr/lib/chromium/chromium-sandbox \
    -Dlinux_strip_binary=1 \
    -Drelease_extra_cflags="$CFLAGS" \
    -Dffmpeg_branding=Chrome \
    -Dproprietary_codecs=1 \
    -Duse_system_bzip2=1 \
    -Duse_system_ffmpeg=0 \
    -Duse_system_libevent=1 \
    -Duse_system_libjpeg=1 \
    -Duse_system_libpng=1 \
    -Duse_system_libxml=0 \
    -Duse_system_ssl=0 \
    -Duse_system_yasm=1 \
    -Duse_system_zlib=1 \
    -Duse_gconf=0 \
    $([[ $CARCH == i686 ]] && echo '-Ddisable_sse2=1')

  make chrome chrome_sandbox BUILDTYPE=Release
}

package() {
  cd "$srcdir/chromium-$pkgver"

  install -D out/Release/chrome ${pkgdir}/usr/lib/chromium/chromium

  install -Dm4755 -o root -g root out/Release/chrome_sandbox \
    "$pkgdir/usr/lib/chromium/chromium-sandbox"

  cp out/Release/{{chrome,resources}.pak,libffmpegsumo.so} \
    out/Release/nacl_helper{,_bootstrap} \
    out/Release/{libppGoogleNaClPluginChrome.so,nacl_irt_x86_*.nexe} \
    "$pkgdir/usr/lib/chromium/"

  # These links are only needed when building with system ffmpeg
  #ln -s /usr/lib/libavcodec.so.52 ${pkgdir}/usr/lib/chromium/
  #ln -s /usr/lib/libavformat.so.52 ${pkgdir}/usr/lib/chromium/
  #ln -s /usr/lib/libavutil.so.50 ${pkgdir}/usr/lib/chromium/

  cp -a out/Release/locales out/Release/resources "$pkgdir/usr/lib/chromium/"

  find "$pkgdir/usr/lib/chromium/" -name '*.d' -type f -delete

  install -Dm644 out/Release/chrome.1 "$pkgdir/usr/share/man/man1/chromium.1"

  install -Dm644 "$srcdir/chromium.desktop" \
    "$pkgdir/usr/share/applications/chromium.desktop"

  for size in 16 22 24 32 48 64 128 256; do
    install -Dm644 "chrome/app/theme/chromium/product_logo_$size.png" \
      "$pkgdir/usr/share/icons/hicolor/${size}x${size}/apps/chromium.png"
  done

  install -D "$srcdir/chromium.sh" "$pkgdir/usr/bin/chromium"

  install -Dm644 LICENSE "$pkgdir/usr/share/licenses/chromium/LICENSE"
}

# vim:set ts=2 sw=2 et:
