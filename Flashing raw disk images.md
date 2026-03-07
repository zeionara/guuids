# Flashing raw disk images

Sometimes OS should be installed by flashing a raw disk image to a memory unit. For instance, this installation method is applied for configuring the [official OS](https://drive.google.com/drive/folders/1jrgZeoYCjDAeRD_4uIC0PWPGO8njqxu9?usp=sharing) for `Orange Pi 6 Plus`. After you've downloaded the file, extract the image from archive:

```sh
7zz x Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.7z
```

Then verify the image integrity:

```sh
sha256sum -c Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.img.sha
```

Expected output:

```sh
Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.img: OK
```

If output from your call says `FAILED`, then download the image again. When the `sha256sum` says `OK`, proceed.

## Flashing all partitions

To **erase all data** on the target memory unit and flash all partitions (the safest and easiest approach), run the following command (replace `/dev/sdb` with path to your memory unit):

```sh
sudo dd if=Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.img of=/dev/sdb bs=4M status=progress conv=fsync
```

Verify that the changes have been applied correctly:

```sh
sudo fdisk -l /dev/sdb
```

Reference output:

```sh
Disk /dev/sdb: 931.51 GiB, 1000204886016 bytes, 1953525168 sectors
Disk model: Tech
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: AB29D93A-B7E0-4B3D-BCB2-8A7A6D7B5C2C

Device       Start      End  Sectors  Size Type
/dev/sdb1    20480   430079   409600  200M EFI System
/dev/sdb2   430080   438271     8192    4M unknown
/dev/sdb3   438272  4534271  4096000    2G Linux root (ARM-64)
/dev/sdb4  4534272  4636671   102400   50M unknown
/dev/sdb5  4636672  5660671  1024000  500M unknown
/dev/sdb6  5660672  5763071   102400   50M unknown
/dev/sdb7  5763072  6172671   409600  200M unknown
/dev/sdb8  6172672 16775167 10602496  5.1G unknown
```

If it says something like:

```sh
GPT PMBR size mismatch (16777215 != 1953525167) will be corrected by write.
The backup GPT table is not on the end of the device. This problem will be corrected by write.
```

Then fix `GPT` size using `fdisk`:

```sh
sudo fdisk /dev/sdb
> w
```

## Flashing some partitions

To keep existing partiton layout on the device and append some partitions, execute the following steps. First, list partitions present in the image file:

```sh
sudo fdisk -l Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.img
```

Then associate the image file with the first available loop device:

```sh
sudo losetup -fP Orangepi6plus_openharmony_5.0.3v1.0.0_linux6.6.img
```

List available partitions:

```sh
lsblk /dev/loop0
```

Make sure that you have tools for working with `f2fs`, on gentoo they can be installed like this:

```sh
sudo emerge --ask sys-fs/f2fs-tools
```

Then mount `EFI` partition:

```sh
sudo mkdir /mnt/orangepi
sudo mount /dev/loop0p1 /mnt/orangepi
```

Print `grub` config:

```sh
cat /mnt/orangepi/GRUB/GRUB.CFG
```

Reference output:

```sh
set debug=loader,mm
set term=vt100
set default=0
set timeout=2


menuentry '0 Cix Sky1 CPIO on EVB (ACPI)' {
    linux /Image \
        console=ttyAMA2,115200 \
        earlycon=pl011,0x040d0000 \
        arm-smmu-v3.disable_bypass=0 \
        cma=640M \
        default_boot_device=CIXH2020:00 \
        ohos.boot.hardware=orangepi_6p \
        cpuidle.off=1 \
        firmware_class.path=/vendor/firmware \
        acpi=force \
        sn=bf9c9492397d \
        udid=e1c1a7d36ef5
    initrd /ramdisk.img
}
menuentry '1 Cix Sky1 CPIO on EVB (Device Tree)' {
    devicetree /sky1-evb.dtb
    linux /Image \
        console=ttyAMA2,115200 \
        earlycon=pl011,0x040d0000 \
        arm-smmu-v3.disable_bypass=0 \
        default_boot_device=soc@0/a070000.pcie \
        ohos.boot.hardware=orangepi_6p \
        firmware_class.path=/vendor/firmware \
        acpi=off
    initrd /ramdisk.img
}
```

Then mount system files partition:

```sh
sudo mount /dev/loop0p3 /mnt/orangepi
```

Copy files from this partition to some local folder:

```sh
mkdir ~/Downloads/orangepi
sudo cp -R /mnt/orangepi/* ~/Downloads/orangepi/
```

Mount partition with binaries:

```sh
sudo mount /dev/loop0p5 /mnt/orangepi
```

Copy system tools from this partition to some local folder:

```sh
mkdir ~/Downloads/orangepi.old
sudo cp -R /mnt/orangepi/* ~/Downloads/orangepi.old/
```

Mount partition with kernel objects:

```sh
sudo umount /mnt/orangepi
sudo mount /dev/loop0p7 /mnt/orangepi
```

Copy kernel objects to some local folder:

```sh
mkdir ~/Downloads/orangepi/kernel
cp /mnt/orangepi/* ~/Downloads/orangepi/kernel
```

When you are finished, unmount the `loop` partitions:

```sh
sudo umount /dev/loop0p1
sudo umount /dev/loop0p3
sudo umount /dev/loop0p5
sudo umount /dev/loop0p7
```

Detach the image from `loop` device:

```sh
sudo losetup -d /dev/loop0
```
