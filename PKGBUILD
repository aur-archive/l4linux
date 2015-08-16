# Mantainer Andrei Zaikin (andzai1995@gmail.com)

pkgbase=l4linux
pkgname=${pkgbase}-svn

pkgver=3.14

#this PKGBUILD is making 3 different components/packages
# (1) The Fiasco L4 microkernel
# (2) The L4Re runtime environment running on (1)
# (3) A patched Linux kernel running on (2)
# packaging aims to make all of this bootable...

pkgdesc="The Linux Kernel running on the L4 microkernel"
depends=('coreutils' 'linux-firmware' 'mkinitcpio')
optdepends=('crda: to set the correct wireless channels of your country')
backup=("etc/mkinitcpio.d/${pkgbase}.preset")
install=l4linux.install

pkgrel=2
url="http://os.inf.tu-dresden.de/L4/LinuxOnL4/"
arch=('i686' 'x86_64')
license=('GPL2')
makedepends=('gcc' 'perl' 'binutils' 'subversion' 'make' 'gawk' 'pkg-config') 
options=(!strip)

#http://os.inf.tu-dresden.de/download/snapshots-oc/
#configs = Arch linux configs + ACPI and PM removed
source=('l4linux.preset' 'l4config-32' 'l4config-64' \
'globalconfig.32' 'globalconfig.64' 'kconfig.64' 'kconfig.32')

md5sums=('4b28af9263a7fedecedb9724ee40a8de' 'bdcdccc95e847d810e82a29ee72dc986'\
 '602f24b0858b9757112d5438a21d66e0' 'd77f71a516442b82c74c6de99f60424e'\
 '083b764da087480235deba00b2960d27' 'fde9c4da94032157d72ba0b7d0252c24'\
 '6feb00bf4693cb593d1711a4d2bd50ea')

export KARCH=x86


prepare() {
	#decide on which configs to use
	if [ ${CARCH} == x86_64 ]; then
	  _fiasconfig=${srcdir}/globalconfig.64
	  _l4reconf=${srcdir}/kconfig.64
	  _linuxconfig=${srcdir}/l4config-64
	else
	  _fiasconfig=${srcdir}/globalconfig.32
	  _l4reconf=${srcdir}/kconfig.32
	  _linuxconfig=${srcdir}/l4config-32
	fi
	_src="${srcdir}/${pkgname}/src"

	cd ${srcdir}/${pkgname}
	if [[ $? == 1 ]]; then
	  msg "getting sources"
	  mkdir -p ${srcdir}/${pkgname}
	  cd ${srcdir}/${pkgname}
	  svn cat http://svn.tudos.org/repos/oc/tudos/trunk/repomgr | perl - init http://svn.tudos.org/repos/oc/tudos fiasco l4re l4linux_requirements
	  cd src/l4/pkg
	  svn up acpica drivers examples fb-drv input io libevent libio-io libgfortran libirq libvcpu lxfuxlibc mag mag-gfx rtc shmc x86emu zlib
	  cd ${_src}
	  svn co http://svn.tudos.org/repos/oc/l4linux/trunk l4linux
	else
	  msg "updating sources" 
	  cd ${_src}
	  svn up
	  cd l4linux
	  svn up
	  cd ..
	  cd ${_src}/l4/pkg
	  svn up acpica drivers examples fb-drv input io libevent libio-io libgfortran libirq libvcpu lxfuxlibc mag mag-gfx rtc shmc x86emu zlib
	fi

	msg "prepare fiasco"
	mkdir -p ${_src}/kernel/fiasco
	cd ${_src}/kernel/fiasco
	rm -rf ${srcdir}/build
	make BUILDDIR="${srcdir}/build/fiasco"
	cp ${_fiasconfig} ${srcdir}/build/fiasco/globalconfig.out
	cd ${srcdir}/build/fiasco
	make oldconfig
	make menuconfig
	#alternatively: make menuconfig

	msg "prepare L4Re"
	cd ${_src}/l4
	make B=${srcdir}/build/l4re
	cp ${_l4reconf} ${srcdir}/build/l4re/.kconfig
	make O=${srcdir}/build/l4re oldconfig
	make O=${srcdir}/build/l4re menuconfig

	msg "prepare l4linux"
	cd ${_src}/l4linux
	cp ${_linuxconfig} .config
	mkdir -p ${srcdir}/build/linux
	make O=${srcdir}/build/linux oldconfig
#Placeholder if you want to do your own config
#do not enable features like ACPI, apic/ioapic, HPET, highmem, MTRR, MCE, power management and similar
#Make sure to enable Support for frame buffer devices and and Framebuffer Console support for the console to work.
	make O=${srcdir}/build/linux menuconfig
}

build() {
	msg "build fiasco"
	cd ${srcdir}/build/fiasco
	make

	msg "build L4Re"
	cd ${_src}/l4
	make O=${srcdir}/build/l4re

	msg "build l4linux"
	cd ${_src}/l4linux
	make O=${srcdir}/build/linux
}

package() {
	msg "package fiasco"
	mkdir -p ${pkgdir}/boot/l4
	install -m755 ${srcdir}/build/fiasco/fiasco ${pkgdir}/boot/l4/

	msg "package L4Re"
	install -m755 O=${srcdir}/build/l4re/* ${pkgdir}/boot/l4/

	msg "package l4linux"
	cd ${_linuxsrc}

	# don't run depmod on 'make install'. We'll do this ourselves in packaging
  	sed -i '2iexit 0' scripts/depmod.sh

	# get kernel version
	_kernver="$(make LOCALVERSION= kernelrelease)"
	_basekernel=${_kernver%%-*}
	_basekernel=${_basekernel%.*}

	mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
	make LOCALVERSION= INSTALL_MOD_PATH="$pkgdir" modules_install
	cp arch/$KARCH/boot/bzImage "${pkgdir}/boot/vmlinuz-${pkgbase}"

	# add vmlinux
  	install -D -m644 vmlinux "${pkgdir}/usr/src/linux-${_kernver}/vmlinux"

  	# set correct depmod command for install
  	cp -f "${startdir}/${install}" "${startdir}/${install}.pkg"
  	true && install=${install}.pkg
  	sed \
    	 -e  "s/KERNEL_NAME=.*/KERNEL_NAME=${_kernelname}/" \
    	 -e  "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/" \
    	 -i "${startdir}/${install}"

  	# install mkinitcpio preset file for kernel
  	install -D -m644 "${srcdir}/l4linux.preset" "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"
  	sed \
    	 -e "1s|'linux.*'|'${pkgbase}'|" \
    	 -e "s|ALL_kver=.*|ALL_kver=\"/boot/vmlinuz-${pkgbase}\"|" \
    	 -e "s|default_image=.*|default_image=\"/boot/initramfs-${pkgbase}.img\"|" \
    	 -e "s|fallback_image=.*|fallback_image=\"/boot/initramfs-${pkgbase}-fallback.img\"|" \
    	 -i "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  	# remove build and source links
  	rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
  	# remove the firmware
  	rm -rf "${pkgdir}/lib/firmware"
  	# gzip -9 all modules to save 100MB of space
  	find "${pkgdir}" -name '*.ko' -exec gzip -9 {} \;

  	# Now we call depmod...
  	depmod -b "$pkgdir" -F System.map "$_kernver"

  	# move module tree /lib -> /usr/lib
  	mv "$pkgdir/lib" "$pkgdir/usr"
}
