1. Cài Đặt Toolchain
1.1 Sử dụng Crosstool-NG để Build Toolchain
1.1.1 clone Crosstool-NG:
git clone https://github.com/crosstool-ng/crosstool-ng.git
cd crosstool-ng
git checkout crosstool-ng-1.24.0

1.1.2 intall
./bootstrap
./configure --prefix=${PWD}
make
make install

1.1.3 select this target configuration
show all toolchain:
bin/ct-ng list-samples

bin/ct-ng arm-unknown-linux-gnueabi

1.1.3 config
bin/ct-ng menuconfig
In Paths and misc options, disable Render the toolchain read-only
(CT_PREFIX_DIR_RO)

1.1.4 Build Toolchain:
bin/ct-ng build
Toolchain sẽ được cài đặt vào thư mục /home/podman/x-tools.

2. Build Bootloader (U-Boot)
2.1 Tải và Cấu Hình U-Boot
Tải mã nguồn U-Boot:
cd /home/podman
git clone git://git.denx.de/u-boot.git
cd u-boot
git checkout v2021.01

2.2 Cấu hình U-Boot:
# make CROSS_COMPILE=/home/podman/x-tools/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi- am335x_evm_defconfig

Build U-Boot:
# make CROSS_COMPILE=/home/podman/x-tools/arm-unknown-linux-gnueabi/bin/arm-unknown-linux-gnueabi-

3. Build Kernel
3.1 Tải và Cấu Hình Kernel
cd /home/podman
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.50.tar.xz
tar xf linux-5.4.50.tar.xz
mv linux-5.4.50 linux-stable

3.2 Cấu hình Kernel:
export PATH=/home/podman/x-tools/arm-unknown-linux-gnueabi/bin:$PATH
make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- mrproper
make ARCH=arm versatile_defconfig

3.3 Build Kernel:
make -j4 ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- zImage
make -j4 ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- modules
make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- dtbs

mkdir ~/rootfs
cd ~/rootfs
mkdir bin dev etc home lib proc sbin sys tmp usr var
mkdir usr/bin usr/lib usr/sbin
mkdir -p var/log

cd ~/rootfs
git clone git://busybox.net/busybox.git
cd busybox
git checkout 1_31_1

make distclean
make defconfig

make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi-
make ARCH=arm CROSS_COMPILE=arm-unknown-linux-gnueabi- CONFIG_PREFIX=/home/podman/rootfs install


cd ~/rootfs
arm-unknown-linux-gnueabi-readelf -a bin/busybox | grep "program interpreter"
arm-unknown-linux-gnueabi-readelf -a bin/busybox | grep "Shared library"

arm-unknown-linux-gnueabi-gcc -print-sysroot

export SYSROOT=$(arm-unknown-linux-gnueabi-gcc -printsysroot)


cd ~/rootfs
cp -a $SYSROOT/lib/ld-linux.so.3 lib
cp -a $SYSROOT/lib/ld-2.29.so lib
cp -a $SYSROOT/lib/libc.so.6 lib
cp -a $SYSROOT/lib/libc-2.29.so lib
cp -a $SYSROOT/lib/libm.so.6 lib
cp -a $SYSROOT/lib/libm-2.29.so lib
cp -a $SYSROOT/lib/libresolv.so.2 lib
cp -a $SYSROOT/lib/libresolv-2.29.so lib

arm-unknown-linux-gnueabi-strip rootfs/lib/libc-2.29.so

cd ~/rootfs
sudo mknod -m 666 dev/null c 1 3
sudo mknod -m 600 dev/console c 5 1
ls -l dev

find . | cpio -H newc -ov --owner root:root >../initramfs.cpio

QEMU_AUDIO_DRV=none qemu-system-arm -m 256M -nographic -M versatilepb -kernel /home/podman/linux-stable/arch/arm/boot/zImage -append "console=ttyAMA0 rdinit=/bin/sh" -dtb /home/podman/linux-stable/arch/arm/boot/dts/versatile-pb.dtb -initrd initramfs.cpio.gz
