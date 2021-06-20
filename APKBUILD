# Maintainer: Morelcia <nat@nekopon.pl>
# Arch Linux contributors:
# Contributor: Levente Polyak <anthraxx[at]archlinux[dot]org>
# Contributor: Daniel Micay <danielmicay@gmail.com>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

_flavor=hardened
pkgname=linux-${_flavor}
pkgver=5.12.12
case $pkgver in
	*.*.*)	_kernver=${pkgver%.*};;
	*.*) _kernver=$pkgver;;
esac
pkgbase=${pkgver}.hardened1
pkgrel=0
pkgdesc="Security-Hardened Linux"
url="https://github.com/anthraxx/linux-hardened"
depends="mkinitfs"
_depends_dev="perl gmp-dev elfutils-dev bash flex bison"
makedepends="$_depends_dev sed installkernel bc linux-headers linux-firmware-any openssl-dev
	diffutils findutils xz mpc1-dev mpfr-dev elfutils-dev zstd"
options="!strip !check" # no tests
_config=${config:-config-hardened.${CARCH}}
_srcname=linux-${pkgbase%.*}
_srctag=${pkgbase%.*}-${pkgbase##*.}
install=
subpackages="$pkgname-dev:_dev:$CBUILD_ARCH"
source="https://cdn.kernel.org/pub/linux/kernel/v${pkgbase%%.*}.x/linux-$_kernver.tar.xz"
case $pkgver in
	*.*.0)	source="$source";;
	*.*.*)	source="$source
	https://cdn.kernel.org/pub/linux/kernel/v${pkgver%%.*}.x/patch-$pkgver.xz" ;;
esac

source="$source
	https://github.com/anthraxx/${pkgname}/releases/download/${_srctag}/${pkgname}-${_srctag}.patch
	config-hardened.x86_64
"

arch="x86_64"
license="GPL-2.0"

_flavors=
for _i in $source; do
	case $_i in
	config-*.$CARCH)
		_f=${_i%.$CARCH}
		_f=${_f#config-}
		_flavors="$_flavors ${_f}"
		if [ "linux-$_f" != "$pkgname" ]; then
			subpackages="$subpackages linux-${_f}::$CBUILD_ARCH linux-${_f}-dev:_dev:$CBUILD_ARCH"
		fi
		;;
	esac
done

_carch=${CARCH}
case "$_carch" in
aarch64*) _carch="arm64" ;;
arm*) _carch="arm" ;;
riscv64) _carch="riscv" ;;
esac

prepare() {
	local _patch_failed=
	cd "$srcdir"/linux-$_kernver
	case $pkgver in
		*.*.0);;
		*)
		msg "Applying patch-$pkgver.xz"
		unxz -c < "$srcdir"/patch-$pkgver.xz | patch -p1 -N ;;
	esac

	# first apply patches in specified order
	for i in $source; do
		local file=${i%::*}
		case $file in
		*.patch)
			msg "Applying $file..."
			if ! patch -s -p1 -N -i "$srcdir"/${file##*/}; then
				echo $file >>failed
				_patch_failed=1
			fi
			;;
		esac
	done

	if ! [ -z "$_patch_failed" ]; then
		error "The following patches failed:"
		cat failed
		return 1
	fi

	# remove localversion from patch if any
	rm -f localversion*
	oldconfig
}

oldconfig() {
	for i in $_flavors; do
		local _config=config-$i.${CARCH}
		local _builddir="$srcdir"/build-$i.$CARCH
		mkdir -p "$_builddir"
		echo "-$pkgrel-$i" > "$_builddir"/localversion-alpine \
			|| return 1

		cp "$srcdir"/$_config "$_builddir"/.config
		make -C "$srcdir"/linux-$_kernver \
			O="$_builddir" \
			ARCH="$_carch" \
			listnewconfig oldconfig
	done
}

build() {
	unset LDFLAGS
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"
	for i in $_flavors; do
		cd "$srcdir"/build-$i.$CARCH
		make ARCH="$_carch" CC="${CC:-gcc}" \
			KBUILD_BUILD_VERSION="$((pkgrel + 1 ))-Alpine"
	done
}

_package() {
	local _buildflavor="$1" _outdir="$2"
	local _abi_release=${pkgver}-${pkgrel}-${_buildflavor}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	cd "$srcdir"/build-$_buildflavor.$CARCH
	# modules_install seems to regenerate a defect Modules.symvers on s390x. Work
	# around it by backing it up and restore it after modules_install
	cp Module.symvers Module.symvers.backup

	mkdir -p "$_outdir"/boot "$_outdir"/lib/modules

	local _install
	case "$CARCH" in
		arm*|aarch64|riscv64) _install="zinstall dtbs_install";;
		*) _install=install;;
	esac

	make -j1 modules_install $_install \
		ARCH="$_carch" \
		INSTALL_MOD_PATH="$_outdir" \
		INSTALL_PATH="$_outdir"/boot \
		INSTALL_DTBS_PATH="$_outdir/boot/dtbs-$_buildflavor"

	cp Module.symvers.backup Module.symvers

	rm -f "$_outdir"/lib/modules/${_abi_release}/build \
		"$_outdir"/lib/modules/${_abi_release}/source
	rm -rf "$_outdir"/lib/firmware

	install -D include/config/kernel.release \
		"$_outdir"/usr/share/kernel/$_buildflavor/kernel.release
}

# main flavor installs in $pkgdir
package() {
	depends="$depends linux-firmware-any"

	_package hardened "$pkgdir"
}

_dev() {
	local _flavor=$(echo $subpkgname | sed -E 's/(^linux-|-dev$)//g')
	local _abi_release=${pkgver}-${pkgrel}-$_flavor
	# copy the only the parts that we really need for build 3rd party
	# kernel modules and install those as /usr/src/linux-headers,
	# simlar to what ubuntu does
	#
	# this way you dont need to install the 300-400 kernel sources to
	# build a tiny kernel module
	#
	pkgdesc="Headers and script for third party modules for $_flavor kernel"
	depends="$_depends_dev"
	local dir="$subpkgdir"/usr/src/linux-headers-${_abi_release}
	export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"

	# first we import config, run prepare to set up for building
	# external modules, and create the scripts
	mkdir -p "$dir"
	cp "$srcdir"/config-$_flavor.${CARCH} "$dir"/.config
	echo "-$pkgrel-$_flavor" > "$dir"/localversion-alpine

	make -j1 -C "$srcdir"/linux-$_kernver O="$dir" ARCH="$_carch" \
		syncconfig prepare modules_prepare scripts

	# remove the stuff that points to real sources. we want 3rd party
	# modules to believe this is the soruces
	rm "$dir"/Makefile "$dir"/source

	# copy the needed stuff from real sources
	#
	# this is taken from ubuntu kernel build script
	# http://kernel.ubuntu.com/git/ubuntu/ubuntu-zesty.git/tree/debian/rules.d/3-binary-indep.mk
	cd "$srcdir"/linux-$_kernver
	find .  -path './include/*' -prune \
		-o -path './scripts/*' -prune -o -type f \
		\( -name 'Makefile*' -o -name 'Kconfig*' -o -name 'Kbuild*' -o \
		   -name '*.sh' -o -name '*.pl' -o -name '*.lds' -o -name 'Platform' \) \
		-print | cpio -pdm "$dir"

	cp -a scripts include "$dir"

	find $(find arch -name include -type d -print) -type f \
		| cpio -pdm "$dir"

	install -Dm644 "$srcdir"/build-$_flavor.$CARCH/Module.symvers \
		"$dir"/Module.symvers

	mkdir -p "$subpkgdir"/lib/modules/${_abi_release}
	ln -sf /usr/src/linux-headers-${_abi_release} \
		"$subpkgdir"/lib/modules/${_abi_release}/build
}

sha512sums="
be03b6fee1d1ea8087b09874d27c0a602c0b04fd90ad38b975bd2c8455a07e83c29b56814aaf1389e82305fae0e4c2d1701075a7f0a7295dd28149f967ec5b3d  linux-5.12.tar.xz
b1d623d03692b03f322b5686bf7dad12a00955688097d53d607e470c3b32dd3ddc89d00f0689a82ed5a406c71cf4dcc09c5c1779f9df82eda2c0436bf012a6a7  patch-5.12.12.xz
e716ac0b8ce4cc859066c17e9e280b674423024a2ceac9886bbef08dbc989a8385c1cdc8bedfd5b961731a5de37b2e4fdd75015d1670d0d5b7e00cf29c007d8e  linux-hardened-5.12.12-hardened1.patch
293827bcc0c92e81e147a24103f0f0a0f0d95456d3ece36e40b6a37184711636ea02a4d947769a8c1c017fdd6672b5eedece36a607c27c3e6da58e34c898e288  config-hardened.x86_64
"
