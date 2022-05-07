# Compiling musl toolchain

use musl-cross-make, If you use a 64 bit target, make sure to build your kernel with 64 bit support.

# Compile busybox

At the root of the busybox source tree run:

```sh
make allnoconfig
make menuconfig
```

I enabled the following set of utilities, but you can add less or more:
- **Settings -> Include busybox applet**

- **Settings -> Build static library**

- **Settings -> Use -static-libc**

- **Settings -> Command line editing**

- **Settings -> Tab completion**

- **Coreutils -> cat**

- **Coreutils -> echo**

- **Coreutils -> cp**

- **Coreutils -> ls**

- **Editors -> vi**

- **Linux System Utilities -> mount**

- **Shells -> ash**

- **Shells -> ash -> Echo Builtin**

To cross compile with musl, Set **Settings -> Cross compiler prefix** to ``/path/to/musl-cross-make/build/local/x68_64-linux-musl/output/bin/x86_64-linux-musl-`` and **Settings -> Path to sysroot** ``/path/to/musl-cross-make/build/local/x86_64-linux-musl/output``.

Finally, run ``make -j$(nproc)`` to compile busybox.

You can use ``readelf -a busybox`` to confirm that the binary is static linked

With my configuration (64 bit) the binary was 200K.

# Compile linux

``make tinyconfig`` is a good starting point for kernel configuration, but you will need to enable some things.

```
make tinyconfig
make menuconfig
```

- **General Setup -> Initial RAM filesystem and RAM disk (initramfs/initrd) support** = y
- **General Setup -> Initial RAM filesystem and RAM disk (initramfs/initrd) support -> Support initial ramdisk/ramfs compressed using gzip** = y
- **General Setup -> Configure standard kernel features (expert users) -> Enable support for printk** = y
- **Executable file formats -> Kernel support for ELF binaries** = y
- **Executable file formats -> Kernel support for scripts starting with #!** = y
- **Device Drivers -> Character devices -> Virtual terminal** = y
- **Device Drivers -> Character devices -> Support for console on virtual terminal** = y
- **Device Drivers -> Graphics support -> Support for Frame buffer Devices** = y
- **Device Drivers -> Graphics support -> Support for Frame buffer Devices -> Simple framebuffer support** = y
- **Device Drivers -> Graphics support -> Firmware Drivers -> Mark VGA/VBE/EFI FB as generic system framebuffer** = y

To build it just run
```
make -j$(nproc)
```

The resultant kernel on my system (64 bit) was 688K

# setup rootfs

First create required structure and copy busybox.

```sh
mkdir root
mkdir root/{bin,proc,dev,sys}
cp /path/to/busybox/binary root/bin
ln -sr root/bin/sh root/bin/busybox
```

Then create ``root/init``, A basic one follows (remember to ``chmod a+x root/init``).

```sh
#!/bin/init
# Put anything you want to run on startup here
/bin/sh
```

Finally you can run this one liner to create an initramfs:

```
find root -printf "%P\0" | cpio --null -o -H newc -D root | gzip > initramfs.cpio.gz
```

# Create image

You will need SYSLINUX for both of these, prebuilt is fine. https://mirrors.edge.kernel.org/pub/linux/utils/boot/syslinux/

## CD rom image

First create a direcory for the iso fs:

```
mkdir isoroot
```

Then copy the kernel and initramfs:

```
mkdir isoroot/boot
cp initramfs.cpio.gz isoroot/boot/ramfs.img
cp path/to/linux/kernel isoroot/boot/vmlinuz
```

The ISOLINUX/SYSLINUX files also have to be copied:

```
cp path/to/syslinux/bios/core/isolinux.bin isoroot
cp path/to/syslinux/bios/com32/elflink/ldlinux/ldlinux.c32 isoroot
```

ISOLINUX needs to be configured, create the file ``isoroot/isolinux.cfg``, with the following:

```
default 1
label 1
    kernel boot/vmlinuz
    initrd boot/ramfs.img
    append quiet
```
finally you can run:

```
mkisofs -no-pad -o tinyroot.iso -b isolinux.bin -no-emul-boot -boot-load-size 4 -boot-info-table isoroot
```

The iso should be at ``tinyroot.iso``


You can test it with QEMU: ``qemu-system-x68_64 -cdrom tinyroot.iso``

## Disk image

###TODO
