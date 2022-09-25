Table of Contents
=================
* [Introduction](#introduction)
* [Using the modified image](#using-the-modified-image)
   * [Prerequisites](#prerequisites)
   * [Downloading the image](#downloading-the-image)
   * [Copying the image to the SD card](#copying-the-image-to-the-sd-card)
   * [Resizing the root partition](#resizing-the-root-partition)
      * [Creating a backup of the SD card](#creating-a-backup-of-the-sd-card)
      * [Analysing the root partition](#analysing-the-root-partition)
      * [Extending the root partition](#extending-the-root-partition)
      * [Calculating new LUKS partition size](#calculating-new-luks-partition-size)
      * [Rebooting and verifying](#rebooting-and-verifying)
   * [Further steps](#further-steps)
      * [Re-encrypting the root partition](#re-encrypting-the-root-partition)
      * [Changing the UUID of the root partition](#changing-the-uuid-of-the-root-partition)
* [Encrypting the root partition manually](#encrypting-the-root-partition-manually)
   * [Prerequisites](#prerequisites-1)
      * [Separate system](#separate-system)
      * [Raspberry Pi](#raspberry-pi)
   * [Downloading the stock image](#downloading-the-stock-image)
   * [Configuration](#configuration)
      * [Preparation](#preparation)
      * [Encrypting the root partition](#encrypting-the-root-partition)
      * [Entering the chroot environment](#entering-the-chroot-environment)
         * [Installing necessary tools](#installing-necessary-tools)
         * [Configuration](#configuration-1)
         * [Generating the initramfs](#generating-the-initramfs)
         * [Exiting the chroot environment](#exiting-the-chroot-environment)
* [Installing the modified image](#installing-the-modified-image)
* [Further steps](#further-steps-1)
   * [Updating all installed packages](#updating-all-installed-packages)
* [Optional steps](#optional-steps)
   * [Decrypting the root partition via SSH](#decrypting-the-root-partition-via-ssh)
      * [Install dropbear-initramfs](#install-dropbear-initramfs)
      * [Configure dropbear](#configure-dropbear)
      * [Configuring kernel parameters](#configuring-kernel-parameters)
      * [Rebuilding the initramfs](#rebuilding-the-initramfs)
      * [Rebooting](#rebooting)
      * [Testing remote decryption](#testing-remote-decryption)
      * [Optional fancy SSH ASCII banner](#optional-fancy-ssh-ascii-banner)
         * [Configuring dropbear](#configuring-dropbear)
         * [Generating a fancy ASCII banner](#generating-a-fancy-ascii-banner)
         * [Rebuilding the initramfs](#rebuilding-the-initramfs-1)
         * [Rebooting](#rebooting-1)
* [Debugging](#debugging)
   * [Examining the initramfs](#examining-the-initramfs)
   * [Unarchiving the initramfs](#unarchiving-the-initramfs)
      * [Easy method](#easy-method)
      * [Elaborated method](#elaborated-method)
* [Additional information](#additional-information)
   * [Credentials](#credentials)
      * [LUKS password](#luks-password)
      * [User credentials](#user-credentials)
      * [Changing the user password](#changing-the-user-password)
      * [Changing the LUKS password](#changing-the-luks-password)
   * [Changing the UUID of the root partition](#changing-the-uuid-of-the-root-partition-1)
      * [Installing necessary tools](#installing-necessary-tools-1)
      * [Changing the UUID](#changing-the-uuid)
      * [Verifying the modification](#verifying-the-modification)
      * [Adapting configuration files](#adapting-configuration-files)
      * [Rebuilding the initramfs](#rebuilding-the-initramfs-2)
      * [Rebooting](#rebooting-2)
   * [Decrypting the root partition from the image](#decrypting-the-root-partition-from-the-image)
   * [Re-encrypting the root partition](#re-encrypting-the-root-partition-1)
      * [Prerequisites](#prerequisites-2)
      * [Creating a backup of the SD card](#creating-a-backup-of-the-sd-card-1)
      * [Configuring the bootloader](#configuring-the-bootloader)
      * [Adapting the modifications of the bootloader](#adapting-the-modifications-of-the-bootloader)
      * [Re-encrypting the partition](#re-encrypting-the-partition)
      * [Reverting the modifications of the bootloader](#reverting-the-modifications-of-the-bootloader)
      * [Verifying the new cipher method](#verifying-the-new-cipher-method)
* [Known issues](#known-issues)

# Introduction
This repository shall describe all necessary steps in order to encrypt the `root partition` of the Raspberry Pi stock image `Raspberry Pi OS Lite`; currently `Debian 10 (Buster)` on a `Raspberry Pi Model B Rev 2`.

The instructions are adaptable for `other Raspberry Pi revisions` as well. They should also work on `image files with partition information` in general.

The entire setup was done on a `Banana Pi Pro` with [`Armbian Buster (mainline based kernel 5.10.y)`](https://www.armbian.com/banana-pi-pro/).

**Please read [Known issues](#known-issues) first before following any of the instructions!**

# Using the modified image
## Prerequisites
* The following packages are installed:
```no-highlight
aria2c
coreutils
cryptsetup-2.0.6 or higher
e2fsprogs
gnupg
linux-image-5.0 or higher
parted
util-linux
```
* `linux-image-5.0 (Linux Kernel 5.0)` or higher and `cryptsetup-2.0.6` or higher are required to support the fast `software-based` encryption method `aes-adiantum-plain64`, since the Raspberry Pi's CPU does not support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`).
* The capacity of the `SD card` must be greater than `8 GiB`.

## Downloading the image
Either download the files manually from the [release page](https://codeberg.org/keks24/raspberry-pi-luks/releases) or download them via `direct links`:
```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" --force-sequential="true" \
    "https://srv-store4.gofile.io/download/48Rnkz/ee3a464731dc8453f7d9b214cdc445dc/raspberrypi_sd_card_backup.img" \
    "https://srv-store4.gofile.io/download/48Rnkz/c57f476a29378ae9ce21dff9e3c8120c/raspberrypi_sd_card_backup.img.asc" \
    "https://srv-store4.gofile.io/download/48Rnkz/2e907656ce9f9c30c31058a1d0e06091/raspberrypi_sd_card_backup.img.b2" \
    "https://srv-store4.gofile.io/download/48Rnkz/26dccd90de56da6cfdf4d00df63291e6/raspberrypi_sd_card_backup.img.sha256" \
    "https://srv-store1.gofile.io/download/48Rnkz/fa7c7877a25f02d9071eb712971917ad/LICENSE"
```

If the links are dead, due to infrequent downloads, please `proceed with` [Encrypting the root partition manually](#encrypting-the-root-partition-manually).

Check the `data integrity` and `verify` the signature:
```bash
$ b2sum --check "raspberrypi_sd_card_backup.img.b2"
raspberrypi_sd_card_backup.img: OK
$ gpg --verify "raspberrypi_sd_card_backup.img.asc" "raspberrypi_sd_card_backup.img"
gpg: Signature made Mon Mar 22 06:16:43 2021 CET
gpg:                using RSA key 598398DA5F4DA46438FDCF87155BE26413E699BF
gpg: Good signature from "Ramon Fischer (ramon@sharkoon) <RamonFischer24@googlemail.com>" [ultimate]
gpg:                 aka "Ramon Fischer (ramon@sharkoon) <Ramon_Fischer@hotmail.de>" [ultimate]
```

## Copying the image to the SD card
Copy the image to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" conv="fsync" status="progress"
```

## Resizing the root partition
When copying the image to another SD card with a `higher capacity`, the `encrypted root partition` will stay at `~8 GiB`. Therefore, it needs to be `extended` in order to use the `unused free space`.

### Creating a backup of the SD card
Before doing any changes, create a `backup` of the SD card, since the following commands can corrupt data:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_resize.img" bs="512b" conv="fsync" status="progress"
```

### Analysing the root partition
After that, boot into `Raspbian` and check the partition structure via `parted`:
```bash
$ parted --list
Model: Linux device-mapper (crypt) (dm)
Disk /dev/mapper/cryptroot: 7659MB
Sector size (logical/physical): 4096B/4096B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  7659MB  7659MB  ext4


Model: SD SC32G (sd/mmc)
Disk /dev/mmcblk0: 31.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  273MB   268MB   primary  fat32        lba
 2      273MB   7948MB  7676MB  primary
```

This indicates, that `/dev/mmcblk0p2` (`/dev/mapper/cryptroot`) only has a size of `7676 MB`, but the SD card's total capacity is `31.9 GB`.

### Extending the root partition
In order to `extend` the `second partition`, execute the following commands:
```bash
$ parted "/dev/mmcblk0"
GNU Parted 3.2
Using /dev/mmcblk0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: SD SC32G (sd/mmc)
Disk /dev/mmcblk0: 31.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  273MB   268MB   primary  fat32        lba
 2      273MB   7948MB  7676MB  primary

(parted) resizepart
Partition number? 2
End?  [7948MB]? -1
(parted) print
Model: SD SC32G (sd/mmc)
Disk /dev/mmcblk0: 31.9GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  273MB   268MB   primary  fat32        lba
 2      273MB   31.9GB  31.6GB  primary

(parted) quit
```

The command `resizepart` will be used to `extend` the partition, where `-1` defines the last sector of the SD card.

### Calculating new LUKS partition size
After `extending` the partition via `parted`, the new `LUKS partition size` needs to be `calculated` via `cryptsetup` as well:
```bash
$ cryptsetup resize cryptroot
Enter passphrase for /dev/mmcblk0p2: raspberry
```

Once this is done, use `resize2fs` to `resize` the partition:
```bash
$ resize2fs /dev/mapper/cryptroot
resize2fs 1.44.5 (15-Dec-2018)
Filesystem at /dev/mapper/cryptroot is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 4
The filesystem on /dev/mapper/cryptroot is now 7720844 (4k) blocks long.
```

Note, that the partition size of `7720844 Bytes (~8 GB)` is still indicated.

### Rebooting and verifying
After rebooting, all changes are applied properly:
```bash
$ reboot
$ cryptsetup status cryptroot
/dev/mapper/cryptroot is active and is in use.
  type:    LUKS2
  cipher:  xchacha20,aes-adiantum-plain64
  keysize: 256 bits
  key location: keyring
  device:  /dev/mmcblk0p2
  sector size:  4096
  offset:  32768 sectors
  size:    7720844 sectors
  mode:    read/write
```

To check, if the values are correct, the following formula can be used:
```no-highlight
(<sector_size> * <size>) / 1024^3 = <size_in_gib>
```

That is:
```no-highlight
(4096 Bytes * 7720844) / 1024^3 = 29.45 GiB
```

The result differs slightly from the output of `parted`, since the unit is in `Gibibyte (base 2)` and not `Gigabyte (base 10)`.

## Further steps
### Re-encrypting the root partition
**It is mandatory to proceed with [Re-encrypting the root partition](#re-encrypting-the-root-partition-1) below!**

### Changing the UUID of the root partition
After that, proceed with [Changing the UUID of the root partition](#changing-the-uuid-of-the-root-partition-1) below for further instructions.

# Encrypting the root partition manually
## Prerequisites
### Separate system
* The following packages are installed:
```no-highlight
aria2c
coreutils
cryptsetup-2.0.6 or higher
e2fsprogs
parted
qemu-user-static
unzip
util-linux
```

### Raspberry Pi
* The following packages are installed:
```no-highlight
cryptsetup-2.0.6 or higher
raspberrypi-kernel-1.20200527-1 or higher
```

* `qemu-user-static` is needed, if one is working on a `non-ARM operating system`.
* `raspberrypi-kernel-1.20200527-1 (Linux Kernel 5.0)` or higher and `cryptsetup-2.0.6` or higher are required to support the fast `software-based` encryption method `aes-adiantum-plain64`, since the Raspberry Pi's CPU does not support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`).
* Free space of at least `1.5 times` the capactiy of the `SD card`

## Downloading the stock image
Download the image `Raspberry Pi OS Lite` from the [official page](https://www.raspberrypi.org/software/operating-systems/) and also save its `SHA256` checksum:

```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" --force-sequential="true" \
    "https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip" \
    "https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip.sha256"
```

Verify the checksum of the archive:
```bash
$ sha256sum --check "2021-01-11-raspios-buster-armhf-lite.zip.sha256"
2021-01-11-raspios-buster-armhf-lite.zip: OK
```

## Configuration
### Preparation
Clone the repository to the `current working directory`:
```bash
$ git clone "https://codeberg.org/keks24/raspberry-pi-luks.git"
```

Unarchive `2021-01-11-raspios-buster-armhf-lite.zip`:
```bash
$ unzip "2021-01-11-raspios-buster-armhf-lite.zip"
Archive:  2021-01-11-raspios-buster-armhf-lite.zip
  inflating: 2021-01-11-raspios-buster-armhf-lite.img
```

Copy the image `2021-01-11-raspios-buster-armhf-lite.img` to the `SD card`:
```bash
$ dd if="2021-01-11-raspios-buster-armhf-lite.img" of="/dev/sdx" bs="512b" conv="fsync" status="progress"
```

Boot into `Raspbian`, so the `root partition` is extended to its full capacity. Then, log in with the [predefined user credentials](#user-credentials) and update the kernel:
```bash
$ sudo apt update
$ sudo apt install raspberrypi-kernel --only-upgrade
$ reboot
```

After logging in again, take notes of the following output. These information are important for later:
```bash
$ uname --kernel-release
5.10.17+
$ grep "Model" "/proc/cpuinfo"
Model           : Raspberry Pi Model B Rev 2
```

Make sure, that the `Kernel version` is `1.20200527-1` or higher and `shut down` the system:
```bash
$ dpkg --list | grep "raspberrypi-kernel"
ii  raspberrypi-kernel                   1.20210303-1                        armhf        Raspberry Pi bootloader
ii  raspberrypi-kernel-headers           1.20210303-1                        armhf        Header files for the Raspberry Pi Linux kernel
$ sudo poweroff
```

After that, create a `backup` of the `SD card`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup.img" bs="512b" conv="fsync" status="progress"
```

Analyse the image for its partition `start sectors` and the `logical sector size`:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 31116288s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start    End        Size       Type     File system  Flags
 1      8192s    532479s    524288s    primary  fat32        lba
 2      532480s  15523839s  14991360s  primary  ext4
```

In this case, the `first partition (boot)` starts at `sector 8192` and the `second partition (root)` at `sector 532480`. The `logical sector size` is `512 bytes`.

These values can be used to mount the partitions from the image.

Before doing that, check, that there are free `loop devices` at `/dev/`:
```bash
$ losetup --list
```

Next, make the partitions available at `/dev/loop1` and `/dev/loop2`:
```bash
$ losetup --offset="$(( 512 * 8192 ))" "/dev/loop1" "raspberrypi_sd_card_backup.img"
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ losetup --list
NAME       SIZELIMIT    OFFSET AUTOCLEAR RO BACK-FILE                                DIO LOG-SEC
/dev/loop1         0   4194304         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
/dev/loop2         0 272629760         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
```

Using `losetup` here is important, since it is necessary for the `chroot` later on. Using `mount --options="offset"` might return the error `overlapping loop device exists`.

Mount and create a `backup` of the `root partition` before encrypting it. The `trailing slash` for the `rsync` command is important here!:
```bash
$ mount "/dev/loop2" "/mnt/"
$ rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" "/mnt/" "root_backup"
```

After that unmount `/mnt/`:
```bash
$ umount "/mnt/"
```

### Encrypting the root partition
Since the preparation is done, the `root partition` can now be `formatted and encrypted` via `cryptsetup`:
```bash
$ cryptsetup --cipher="xchacha20,aes-adiantum-plain64" --key-size="256" --sector-size="4096" luksFormat "/dev/loop2"
WARNING: Device /dev/loop2 already contains a 'ext4' superblock signature.

WARNING!
========
This will overwrite data on /dev/loop2 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/loop2: raspberry
Verify passphrase: raspberry
```

It is recommended to use `aes-adiantum-plain64`, since the CPU does **not** support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`).

The `LUKS header information` looks like so:
```bash
$ cryptsetup luksDump "/dev/loop2"
LUKS header information
Version:       	2
Epoch:         	3
Metadata area: 	16384 [bytes]
Keyslots area: 	16744448 [bytes]
UUID:          	4b1b525d-da43-4316-83a3-39a67b8403db
Label:         	(no label)
Subsystem:     	(no subsystem)
Flags:       	(no flags)

Data segments:
  0: crypt
	offset: 16777216 [bytes]
	length: (whole device)
	cipher: xchacha20,aes-adiantum-plain64
	sector: 4096 [bytes]

Keyslots:
  0: luks2
	Key:        256 bits
	Priority:   normal
	Cipher:     xchacha20,aes-adiantum-plain64
	Cipher key: 256 bits
	PBKDF:      argon2i
	Time cost:  4
	Memory:     190840
	Threads:    4
	Salt:       f4 e7 c8 63 26 ee 0d ae 2b dc 8a 25 bc cd 62 a5
	            6f d4 ea 91 da c0 a9 b4 1a 4b 6a 04 ee b8 d5 92
	AF stripes: 4000
	AF hash:    sha256
	Area offset:32768 [bytes]
	Area length:131072 [bytes]
	Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
	Hash:       sha256
	Iterations: 68124
	Salt:       11 23 2d 69 d3 3a e6 b3 1b b0 78 1a 78 b5 4c 3e
	            68 6d ec cc 65 69 4b 61 fd ee 51 eb a1 01 58 f4
	Digest:     d7 bd 62 df ba 94 d2 b6 ec 10 6a 7d 93 9b d1 5d
	            08 56 1c 92 fb e7 6a 48 37 64 73 7d 61 e9 8c 0f
```

Other `encryption methods` are supported as well and can be looked up [here](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/LUKS-standard/on-disk-format.pdf#Cipher%20and%20Hash%20specification%20registry).

If the encryption method `aes-xts-plain64` is preferred, make absolutely sure, that the `key size` is `at least 512 Bytes`, [since `XTS` splits the key size in half](https://wiki.archlinux.org/index.php/dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode).

Now, open/decrypt the encrypted `root partition` and format it via `mkfs.ext4`:
```bash
$ cryptsetup open "/dev/loop2" cryptroot
Enter passphrase for /dev/loop2: raspberry
$ mkfs.ext4 "/dev/mapper/cryptroot"
mke2fs 1.44.5 (15-Dec-2018)
Creating filesystem with 384000 4k blocks and 96000 inodes
Filesystem UUID: 1fd31646-340c-47ed-8c66-8efb2e730d0f
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
```

After that, it is possible to mount the encrypted partition `/dev/mapper/cryptroot`:
```bash
$ mount "/dev/mapper/cryptroot" "/mnt/"
```

Finally, restore the backup:
```bash
$ rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" "root_backup/" "/mnt/"
```

### Entering the chroot environment
Once this is done, it is time to go into a `chroot environment`.

Before doing so, mount the `boot partition` to `/mnt/boot/`:
```bash
$ mount "/dev/loop1" "/mnt/boot/"
```

Then, prepare the `chroot environment`:
```bash
$ mount --types proc "/proc/" "/mnt/proc/"
$ mount --rbind "/sys" "/mnt/sys/"
$ mount --make-rslave "/mnt/sys/"
$ mount --rbind "/dev/" "/mnt/dev/"
$ mount --make-rslave "/mnt/dev/"
$ cp "/usr/bin/qemu-arm-static" "/mnt/usr/bin/"
$ cp --dereference "/etc/resolv.conf" "/mnt/etc/"
```

**`qemu-arm-static` is mandatory, if one is working on a `Non-ARM operating system`!**

`/etc/resolv.conf` contains entries of `DNS name servers`, which are required within the `chroot environment` in order to make `reverse DNS lookups` possible.

Enter the new environment:
```bash
$ chroot "/mnt/" qemu-arm-static "/bin/bash"
$ source "/etc/profile"
$ export PS1="(chroot) ${PS1}"
(chroot) $ cd
```

#### Installing necessary tools
An `initramfs` is needed in order to decrypt the `root partition` on boot. The following packages will provide all tools to build it:
```bash
(chroot) $ apt update
(chroot) $ apt install busybox cryptsetup initramfs-tools
```

#### Configuration
The binary `cryptsetup` must be included in the `initramfs`. This can be configured in `/etc/cryptsetup-initramfs/conf-hook`:
```bash
(chroot) $ vi "/etc/cryptsetup-initramfs/conf-hook"
CRYPTSETUP="y"
```

The `initramfs` should always be generated, when a new kernel was installed. Enable this by setting `RPI_INITRD=Yes` in `/etc/default/raspberrypi-kernel`:
```bash
(chroot) $ vi "/etc/default/raspberrypi-kernel"
RPI_INITRD=Yes
```

As of writing, the package `rpi-initramfs-tools` is not available, yet. So `custom hook scripts` have to be created for this.

The next commands and explanations contain the kernel version `5.10.17+`. Replace the version according to the `Raspberry Pi revision` (`grep "Model" "/proc/cpuinfo"`) and the current kernel version (`uname --kernel-release`):

Type                | Kernel version naming convention   | Kernel filename   | Initramfs filename
------------------- | ---------------------------------- | ----------------- |  -----------------
Raspberry Pi 1      | `<kernel_version>+`                | `kernel.img`      | `initrd.img-<kernel_version>+`
Raspberry Pi Zero   | `<kernel_version>+`                | `kernel.img`      | `initrd.img-<kernel_version>+`
Raspberry Pi Zero W | `<kernel_version>+`                | `kernel.img`      | `initrd.img-<kernel_version>+`
Raspberry Pi 2      | `<kernel_version>-v7`              | `kernel7.img`     | `initrd.img-<kernel_version>-v7`
Raspberry Pi 3      | `<kernel_version>-v7`              | `kernel7.img`     | `initrd.img-<kernel_version>-v7`
Raspberry Pi 3+     | `<kernel_version>-v7`              | `kernel7.img`     | `initrd.img-<kernel_version>-v7`
Raspberry Pi 4      | `<kernel_version>-v7l`             | `kernel7l.img`    | `initrd.img-<kernel_version>-v7l`
Raspberry Pi 4      | `<kernel_version>-v7l+`            | `kernel7l.img`    | `initrd.img-<kernel_version>-v7l+`

[Source](https://www.raspberrypi.com/documentation/computers/linux_kernel.html#default_configuration)

Copy the `custom hook scripts`. **This step must be done in a `separate shell` outside of the `chroot environment`!**:
```bash
$ install -D --owner="root" --group="root" --mode="755" --verbose "raspberry-pi-luks/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools" "/mnt/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools"
install: creating directory '/mnt/usr/local/share/kernel'
install: creating directory '/mnt/usr/local/share/kernel/postinst.d'
'raspberry-pi-luks/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools' -> '/mnt/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools'
$ install -D --owner="root" --group="root" --mode="755" --verbose "raspberry-pi-luks/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools" "/mnt/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools"
install: creating directory '/mnt/usr/local/share/kernel/postrm.d'
'raspberry-pi-luks/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools' -> '/mnt/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools'
$ install -D --owner="root" --group="root" --mode="755" --verbose "raspberry-pi-luks/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools" "/mnt/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools"
install: creating directory '/mnt/usr/local/share/kernel/preinst.d'
'raspberry-pi-luks/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools' -> '/mnt/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools'
```

The `custom hook scripts` are placed in the directory `/usr/local/share/kernel/` and have the following tasks:
* `postinst.d/01-rpi-initramfs-tools`
    * `Appends` the entry `initramfs initramfs.cpio.gz followkernel` to the configuration file `/boot/config.txt`.
        * `Uncomments` the entry `#initramfs [...]` to `initramfs [...]`, if it was commented before, but does not verify the subsequent entries. **Make absolutely sure, that these are correct!**
    * `Renames` the newly generated `initramfs` file `/boot/initrd-5.10.17+` to `/boot/initramfs.cpio.gz`.
* `postrm.d/01-rpi-initramfs-tools`
    * `Comments` the entry `initramfs [...]` to `#initramfs [...]`.
    * `Removes` the `initramfs` files `/boot/initrd-5.10.17+` and `/boot/initramfs.cpio.gz`.
* `preinst.d/01-rpi-initramfs-tools`
    * `Installs` the `post-install` hook script `/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools` to `/etc/kernel/postinst.d/5.10.17+/01-rpi-initramfs-tools` by creating a `symbolic link`.
    * `Installs` the `post-remove` hook script `/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools` to `/etc/kernel/postrm.d/5.10.17+/01-rpi-initramfs-tools` by creating a `symbolic link`.
    * `Renames` the hook script directory names of `/etc/kernel/postinst.d/5.10.17+` and `/etc/kernel/postrm.d/5.10.17+` after each update of the package `raspberrypi-kernel`.
        * `Skips renaming`, if the `current kernel version` and the `package kernel version` are identical

Back to the `chroot environment`. Install the `hook scripts` via `symbolic links`:
```bash
(chroot) $ mkdir --parents /etc/kernel/{postinst.d,postrm.d}/5.10.17+/ "/etc/kernel/preinst.d/"
(chroot) $ ln --symbolic --verbose "/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools" "/etc/kernel/postinst.d/5.10.17+/"
'/etc/kernel/postinst.d/01-rpi-initramfs-tools' -> '/usr/local/share/kernel/postinst.d/01-rpi-initramfs-tools'
(chroot) $ ln --symbolic --verbose "/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools" "/etc/kernel/postrm.d/5.10.17+/"
'/etc/kernel/postrm.d/01-rpi-initramfs-tools' -> '/usr/local/share/kernel/postrm.d/01-rpi-initramfs-tools'
(chroot) $ ln --symbolic --verbose "/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools" "/etc/kernel/preinst.d/"
'/etc/kernel/preinst.d/01-rpi-initramfs-tools' -> '/usr/local/share/kernel/preinst.d/01-rpi-initramfs-tools'
```

Be aware, that the hook script directories `/etc/kernel/postinst.d/5.10.17+/` and `/etc/kernel/postrm.d/5.10.17+` must always match the `current` or `newer` kernel version of the `Raspberry Pi`. **Otherwise, generating the `initramfs` will fail; rendering the `Raspberry Pi` unbootable**.

Note, that the custom hook script `/etc/kernel/preinst.d/01-rpi-initramfs-tools` silently fails, if working on a different system in a `chroot environment`, since it uses `uname` to determine the `current kernel version` and compares its `suffix` with the `suffix` of the `package kernel version` in order to copy the `post-install` and `post-remove` hook scripts into their respective directories. If one is using a `Raspberry Pi` for this setup, make sure, that the `kernel version` of the `Raspberry Pi` and within the `chroot environment` are identical.

Next, get the `UUID` of `/dev/loop2`, which will be used later on:
```bash
(chroot) $ blkid "/dev/loop2"
/dev/loop2: UUID="1fd31646-340c-47ed-8c66-8efb2e730d0f" TYPE="crypto_LUKS"
```

Edit the kernel parameters in `/boot/cmdline.txt` and add an entry in `/etc/crypttab`, so the `root partition` can be decrypted on boot:
```bash
(chroot) $ vi "/boot/cmdline.txt"
root=/dev/mapper/cryptroot cryptdevice=UUID=1fd31646-340c-47ed-8c66-8efb2e730d0f:cryptroot
(chroot) $ vi "/etc/crypttab"
cryptroot UUID=1fd31646-340c-47ed-8c66-8efb2e730d0f none luks,initramfs
```

Note, that adding the option `initramfs` is very important here. Otherwise, the following error might occur and the file `cryptroot/crypttab` within the `initramfs` might be empty; **rendering the system unbootable**:
```no-highlight
cryptsetup: WARNING: target '<some_target>' not found in /etc/crypttab
cryptsetup: ERROR: cryptroot: Source mismatch
```

Adapt the file `/etc/fstab`, so the `decrypted root partition` will be mounted automatically:
```bash
(chroot) $ vi "/etc/fstab"
#PARTUUID=e8af6eb2-02  /               ext4    defaults,noatime  0       1
/dev/mapper/cryptroot  /               ext4    defaults,noatime  0       1
```

#### Generating the initramfs
There are three ways to generate the `initramfs`.

1. Either reinstall the package `raspberrypi-kernel`, which will install all `Raspberry Pi kernels` and then executes the `hook scripts` in `/etc/kernel/preinst.d/`, `/etc/kernel/postrm.d/` and `/etc/kernel/postinst.d/`:
```bash
(chroot) $ apt install raspberrypi-kernel --reinstall
```

2. Execute the command `mkinitramfs` manually:
```bash
(chroot) $ mkinitramfs -o "/boot/initramfs.cpio.gz" 5.10.17+
```

3. Or execute `update-initramfs` and rename the file `initrd.img-5.10.17+` manually:
```bash
(chroot) $ update-initramfs -vuk 5.10.17+
update-initramfs: Generating /boot/initrd.img-5.10.17+
Copying module directory kernel/drivers/usb/dwc2
Adding module /lib/modules/5.10.17+/kernel/drivers/usb/roles/roles.ko
[...]
(chroot) $ mv "/boot/initrd.img-5.10.17+" "/boot/initramfs.cpio.gz"
```

For this setup, the `first method is preferred` in order to test the hook scripts in `preinst.d`, `postrm.d` and `postinst.d`.

After the reinstallation has been completed, there should be the entry `initramfs initramfs.cpio.gz followkernel` in the configuration file `/boot/config.txt`:
```bash
$ (chroot) tail --lines="3" "/boot/config.txt"
[all]
#dtoverlay=vc4-fkms-v3d
initramfs initramfs.cpio.gz followkernel
```

Also, the `initramfs` file `/boot/initramfs.cpio.gz` should be `created/updated`:
```bash
$ (chroot) stat "/boot/initramfs.cpio.gz" | grep "Modify"
Modify: 2021-04-10 22:59:16.000000000 +0100
```

Make sure, that the following important files are present in the `initramfs` file `initramfs.cpio.gz`:
```bash
(chroot) $ lsinitramfs "/boot/initramfs.cpio.gz" | grep --extended-regexp "adiantum.ko|crypttab|sbin/cryptsetup"
cryptroot/crypttab
usr/lib/modules/5.10.17+/kernel/crypto/adiantum.ko
usr/sbin/cryptsetup
```

Also make sure, that the content of `cryptroot/crypttab` is correct.

Detailed debugging is explained [below](#debugging).

#### Exiting the chroot environment
Exit the `chroot`, unmount the `boot partition` and all `pseudo filesystems`:
```bash
(chroot) $ exit
$ cd
$ umount "/mnt/boot/"
$ umount --lazy --recursive /mnt/{proc/,sys/,dev/}
```

Remove `qemu-arm-static` from `/mnt/usr/bin/`:
```bash
$ rm "/mnt/usr/bin/qemu-arm-static"
```

Unmount the encrypted `root partition` and detach all `loop devices`:
```bash
$ umount "/mnt/"
$ cryptsetup close cryptroot
$ losetup --detach "/dev/loop1"
$ losetup --detach "/dev/loop2"
$ losetup --list
```

# Installing the modified image
The image is now prepared and can be copied to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" conv="fsync" status="progress"
```

On boot there should be a message to decrypt the `root partition`:
```no-highlight
Please unlock disk cryptroot: raspberry
```

After entering the password, the `Raspberry Pi` should boot.

# Further steps
## Updating all installed packages
As time progresses, the probability is very high, that `new packages` are available. Update them using the following commands:
```bash
$ apt update
$ apt list --upgradable
$ apt upgrade
```

# Optional steps
## Decrypting the root partition via SSH
In order to decrypt the `root partition` via `SSH`, further configuration is needed. `dropbear` suits here well, since it does not require much memory.

### Install dropbear-initramfs
Log into the `Raspberry Pi` and install the package `dropbear-initramfs`:
```bash
$ apt install dropbear-initramfs
```

### Configure dropbear
All configuration files can be found at `/etc/dropbear-initramfs/` and are self-explained.

Next, configure `dropbear` by editing `/etc/dropbear-initramfs/config`:
```bash
$ vi "/etc/dropbear-initramfs/config"
DROPBEAR_OPTIONS="-p 22222 -I 60 -sjk"
```

This sets the `SSH port` to `22222`, sets the `idle timeout` to `60 seconds` and disables `logins via password`, `local port forwarding` and `remote port forwarding`.

Further information can be looked up at `man 8 dropbear`.

Before adding the public key, the file `authorized_keys` should be created like so:
```bash
$ touch "/etc/dropbear-initramfs/authorized_keys"
$ chmod 600 "/etc/dropbear-initramfs/authorized_keys"
```

As of writing, `Raspbian` (`Debian 10 (Buster)`) uses `dropbear` version [`2018.76-5`](https://packages.debian.org/buster/dropbear), which does not support `ed25519 keys`. The next stable release of `Debian 11 (Bullseye)` will have version [`2020.81-3`](https://packages.debian.org/bullseye/dropbear) available, which supports [`ed25519 keys`](https://github.com/mkj/dropbear/releases/tag/DROPBEAR_2020.79).

This means, that a strong `RSA 8192` key should be generated on the `host`, from which the partition should be decrypted:
```bash
$ ssh-keygen -t rsa -b 8192 -f "/home/<some_username>/.ssh/dropbear_root_rsa8192"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/<some_username>/.ssh/dropbear_root_rsa8192):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/<some_username>/.ssh/dropbear_root_rsa8192
Your public key has been saved in /home/<some_username>/.ssh/dropbear_root_rsa8192.pub
The key fingerprint is:
SHA256:XaASDCS2YOFPyNQzO7IalEvMC8EE7ZZhrpzPFRymwjI <some_username>@<some_hostname>
The key's randomart image is:
+---[RSA 8192]----+
|           o++.  |
|          o.. . .|
|         o+= o o |
|         o=+O o +|
|        S.oO o o+|
|          =o+. ..|
|         .+.+Boo |
|        .o =.o@  |
|        .. .=E.. |
+----[SHA256]-----+
```

Copy the contents of the `SSH public key` `/home/<some_username>/.ssh/dropbear_root_rsa8192.pub` to the `Raspberry Pi` to `/etc/dropbear-initramfs/authorized_keys` with the following `SSHD options`:
```bash
no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/usr/bin/cryptroot-unlock" ssh-rsa [...]
```

The option `command` restricts the logged in user `root` to only execute the command `/usr/bin/cryptroot-unlock` within the `initramfs`. All other options should be self-explained, but can be looked up at `man 8 sshd`. The path to the binary can be determined by [examining](#examining-the-initramfs) or [unarchiving the content of the `initramfs`](#unarchiving-the-initramfs).

Removing all options will grant access to the `busybox` as user `root`.

Make sure, that the `SSH public key` is copied correctly, since `logins via password` are disabled.

### Configuring kernel parameters
In order to make the `initramfs` available in the network on boot, set a `static IP address` via kernel parameters. Append the following line in `/boot/cmdline.txt`:
```bash
ip=192.168.1.80:::255.255.255.0
```

The kernel parameter `ip` has the following syntax:
```no-highlight
ip=<client_ip>:<server_ip>:<gateway_ip>:<netmask>:<hostname>:<network_interface>:<autoconf>:<dns0_ip>:<dns1_ip>:<ntp0_ip>
```

[Source](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)

This will set the `private Class C IP address` to `192.168.1.80` and the `subnet mask` to `255.255.255.0`; these values may differ depending on the network infrastructure. The `initramfs` does not have any connection to the `internet`, since no `gateway IP address` is set.

Make sure, that the `Raspberry Pi` and the `host` from which the partition should be decrypted, are in the same network.

### Rebuilding the initramfs
`Rebuild` the `initramfs` to adapt all changes:
```bash
$ mkinitramfs -o "/boot/initramfs.cpio.gz"
```

Make sure, that the binary `dropbear` and its `configuration files` are present in the file `initramfs.cpio.gz`:
```bash
$ lsinitramfs "/boot/initramfs.cpio.gz" | grep --extended-regexp "dropbear|authorized_keys"
etc/dropbear
etc/dropbear/config
etc/dropbear/dropbear_dss_host_key
etc/dropbear/dropbear_ecdsa_host_key
etc/dropbear/dropbear_rsa_host_key
root-0sldnV/.ssh/authorized_keys
scripts/init-bottom/dropbear
scripts/init-premount/dropbear
usr/sbin/dropbear
```

### Rebooting
After that, `reboot` the `Raspberry Pi`:
```bash
$ reboot
```

### Testing remote decryption
The `initramfs` should be now accessable via `SSH` on `port 22222`, which can be accessed like so:
```bash
$ ssh -p 22222 root@192.168.1.80 -i "/home/<some_username>/.ssh/dropbear_root_rsa8192"
```

The following message should appear:
```no-highlight
Please unlock disk cryptroot: raspberry
```

After entering the correct password, the `SSH connection` will cut off and the `Raspberry Pi` will boot:
```no-highlight
cryptsetup: cryptroot set up successfully
Shared connection to 192.168.1.80 closed.
```

### Optional fancy SSH ASCII banner
`dropbear` allows to set a custom `ASCII banner`, which is shown, when connecting to the `initramfs`.

#### Configuring dropbear
To do this, the parameter `-b` has to be appended to the configuration file `/etc/dropbear-initramfs/config`:
```bash
$ vi "/etc/dropbear-initramfs/config"
$ DROPBEAR_OPTIONS="-p 22222 -I 60 -sjk -b etc/dropbear/ssh_banner.net"
```

Leaving out the `leading slash` is important.

#### Generating a fancy ASCII banner
Then, generate a `fancy ASCII banner` via `figlet` and save it to `/etc/dropbear-initramfs/ssh_banner.net`:
```bash
$ apt update
$ apt install figlet
$ figlet -w 100 -f slant "decrypt cryptroot" > "/etc/dropbear-initramfs/ssh_banner.net"
$ echo "" >> "/etc/dropbear-initramfs/ssh_banner.net"
```

There are also websites to generate some fancy ASCII art:
* http://patorjk.com/software/taag
* http://www.network-science.de/ascii/

After that, implement the `ASCII banner` by using a `custom hook script` for `initramfs-tools`. One is already prepared in the repository:
```bash
$ git clone "https://codeberg.org/keks24/raspberry-pi-luks.git"
$ install -D --owner="root" --group="root" --mode="755" --verbose "raspberry-pi-luks/etc/initramfs-tools/hooks/dropbear" "/etc/initramfs-tools/hooks/dropbear"
'raspberry-pi-luks/etc/initramfs-tools/hooks/dropbear' -> '/etc/initramfs-tools/hooks/dropbear'
```

#### Rebuilding the initramfs
Finally, rebuild the `initramfs`:
```bash
$ mkinitramfs -o "/boot/initramfs.cpio.gz"
```

#### Rebooting
After rebooting the `Raspberry Pi`, `dropbear` will now display a `fancy ASCII banner`:
```bash
$ ssh -p 22222 root@192.168.1.80 -i "/home/<some_username>/.ssh/dropbear_root_rsa8192"
       __                           __                           __                   __
  ____/ /__  ____________  ______  / /_   ____________  ______  / /__________  ____  / /_
 / __  / _ \/ ___/ ___/ / / / __ \/ __/  / ___/ ___/ / / / __ \/ __/ ___/ __ \/ __ \/ __/
/ /_/ /  __/ /__/ /  / /_/ / /_/ / /_   / /__/ /  / /_/ / /_/ / /_/ /  / /_/ / /_/ / /_
\__,_/\___/\___/_/   \__, / .___/\__/   \___/_/   \__, / .___/\__/_/   \____/\____/\__/
                    /____/_/                     /____/_/

Please unlock disk cryptroot: raspberry
```

# Debugging
## Examining the initramfs
There is a tool, called `lsinitramfs`, which can output the content of a compressed `initramfs` to `stdout`:
```bash
$ lsinitramfs "/boot/initramfs.cpio.gz" | less
```

This is useful, if one wants to check a recent-built `initramfs` in a quick way.

## Unarchiving the initramfs
The `initramfs` can be unarchived on the system in order to analyse its content.

### Easy method
The following command `unarchives` the `initramfs` directly to the `current working directory`:
```bash
$ unmkinitramfs -v "/boot/initramfs.cpio.gz" .
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
cryptroot
cryptroot/crypttab
etc
etc/console-setup
[...]
```

It is also possible to directly unarchive it to a custom directory:
```bash
$ unmkinitramfs -v "/boot/initramfs.cpio.gz" "initramfs/"
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
cryptroot
cryptroot/crypttab
etc
etc/console-setup
[...]
```

Note, that this method comes with a caveat (`man 8 unmkinitramfs`):
```no-highlight
unmkinitramfs cannot deal with multiple-segmented initramfs images, except where an early (uncompressed) initramfs with system firmware is prepended to the regular compressed initramfs.
```

### Elaborated method
Copy the initramfs `initramfs.cpio.gz` to the `current working directory`:
```bash
$ cp --archive "/boot/initramfs.cpio.gz" .
```

Then, analyse which type of compression was used:
```bash
$ file "initramfs.cpio.gz"
/boot/initramfs.cpio.gz: gzip compressed data, last modified: Tue Mar 23 00:35:15 2021, from Unix, original size 24183808
```

To unarchive it, `gzip` is required and the file must have the suffix `.gz`:
```bash
$ apt update
$ apt install gzip
$ gzip --decompress "initramfs.cpio.gz"
```

Once this is done, the file is still compressed as `ASCII cpio archive`, which can be unarchived to the `current working directory` like so:
```bash
$ file "initramfs.cpio"
initramfs.cpio: ASCII cpio archive (SVR4 with no CRC)
$ apt install cpio
$ cpio --extract --make-directories --preserve-modification-time --verbose < "initramfs.cpio"
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
cryptroot
cryptroot/crypttab
etc
etc/console-setup
[...]
```

# Additional information
## Credentials
It is **highly recommended** to change these passwords!

### LUKS password
Password: `raspberry`

The `American keyboard layout` applies here.

See also [Changing the LUKS password](#changing-the-luks-password).

### User credentials
Username: `pi`

Password: `raspberry`

The `American keyboard layout` applies here.

See also [Changing the user password](#changing-the-user-password).

### Changing the user password
When logged in as the user `pi`, change the password as follows:
```bash
$ passwd
Changing password for pi.
Current password: raspberry
New password: <some_strong_personal_password>
Retype new password: <some_strong_personal_password>
passwd: password updated successfully
```

### Changing the LUKS password
The password of the `root partition` of the image can be changed like so:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 31116288s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start    End        Size       Type     File system  Flags
 1      8192s    532479s    524288s    primary  fat32        lba
 2      532480s  15523839s  14991360s  primary
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup luksChangeKey "/dev/loop2"
Enter passphrase to be changed: raspberry
Enter new passphrase: <some_strong_personal_password>
Verify passphrase: <some_strong_personal_password>
$ losetup --detach "/dev/loop2"
$ losetup --list
```

This is also possible on the `Raspberry Pi` itself:
```bash
$ cryptsetup luksChangeKey "/dev/mmcblk0p2"
Enter passphrase to be changed: raspberry
Enter new passphrase: <some_strong_personal_password>
Verify passphrase: <some_strong_personal_password>
```

## Changing the UUID of the root partition
When using the `modified` or a `self-prepared` image on `several Raspberry Pis`, all `UUIDs` are identical. There might be the case to change these.

Before doing any changes, create a `backup` of the SD card, since the following commands can corrupt data:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_changing_uuid.img" bs="512b" conv="fsync" status="progress"
```

### Installing necessary tools
Once this is done, boot into `Raspbian`, install the package `uuid-runtime` in order to install the tool `uuidgen`
```bash
$ apt update
$ apt install uuid-runtime
[...]
Created symlink /etc/systemd/system/sockets.target.wants/uuidd.socket → /lib/systemd/system/uuidd.socket.
```

After that, disable its `systemd service` and `socket unit`, since `time-based UUIDs` are not required:
```bash
$ systemctl disable uuidd.service uuidd.socket --now
Synchronizing state of uuidd.service with SysV service script with /lib/systemd/systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable uuidd
Removed /etc/systemd/system/sockets.target.wants/uuidd.socket.
$ systemctl is-active uuidd.service uuidd.socket
inactive
inactive
$ systemctl is-enabled uuidd.service uuidd.socket
indirect
disabled
```

Further details about the `systemd service unit` can be found [here](https://packages.debian.org/buster/uuid-runtime).

### Changing the UUID
Next, generate a `new random UUID` via `uuidgen` and `modify` the `root partition` via `cryptsetup`:
```bash
$ uuidgen --random
a00be720-f82f-452c-96cf-669601d1d57e
$ cryptsetup --uuid="a00be720-f82f-452c-96cf-669601d1d57e" luksUUID "/dev/mmcblk0p2"

WARNING!
========
Do you really want to change UUID of device?

Are you sure? (Type 'yes' in capital letters): YES
```

### Verifying the modification
Verify, if the `new UUID` has been applied successfully:
```bash
$ blkid "/dev/mmcblk0p2"
/dev/mmcblk0p2: UUID="a00be720-f82f-452c-96cf-669601d1d57e" TYPE="crypto_LUKS" PARTUUID="e8af6eb2-02"
```

### Adapting configuration files
After that, adapt the entries in `/boot/cmdline` and `/etc/crypttab`
```bash
$ vi "/boot/cmdline.txt"
root=/dev/mapper/cryptroot cryptdevice=UUID=a00be720-f82f-452c-96cf-669601d1d57e:cryptroot
$ vi "/etc/crypttab"
cryptroot UUID=a00be720-f82f-452c-96cf-669601d1d57e none luks,initramfs
```

### Rebuilding the initramfs
Rebuild the `initramfs`:
```bash
$ mkinitramfs -o "/boot/initramfs.cpio.gz"
```

Make sure, that the following important files are present in the `initramfs` file `initramfs.cpio.gz`:
```bash
$ lsinitramfs "/boot/initramfs.cpio.gz" | grep --extended-regexp "adiantum.ko|crypttab|sbin/cryptsetup"
cryptroot/crypttab
usr/lib/modules/5.10.17+/kernel/crypto/adiantum.ko
usr/sbin/cryptsetup
```

Also make sure, that the content of `cryptroot/crypttab` is correct.

Detailed debugging is explained [here](#debugging).

### Rebooting
Finally, `reboot` the `Raspberry Pi`:
```bash
$ reboot
```

## Decrypting the root partition from the image
The encrypted `root partition` can be opened via `cryptsetup` as follows:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 31116288s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start    End        Size       Type     File system  Flags
 1      8192s    532479s    524288s    primary  fat32        lba
 2      532480s  15523839s  14991360s  primary
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcard
Enter passphrase for /dev/loop2: raspberry
```

After that, mount it like so:
```bash
$ mount "/dev/mapper/cryptsdcard" "/mnt/"
```

## Re-encrypting the root partition
When using the `modified` or a `self-prepared` image on `several Raspberry Pis`, all `key slot hashes and salts` are identical. **It is mandatory to change these.**

This method can also be used to apply a `new cipher method` to the `root partition`. This can only be done `offline` and **not** `on-the-fly`. Further configuration is needed for this.

### Prerequisites
* `LUKS` partition `version 2` (`cryptsetup luksDump "/dev/mmcblk0p2" | grep "Version"`)
* Raspberry Pi 4
* Bootable `USB stick` with `Raspbian`, which is accessable via `SSH`
    * `cryptsetup-2.0.6` or higher

If the `LUKS` partition version is `1`, please upgrade it to version `2` first, using [these instructions](https://gist.github.com/kravietz/d7ea4d98c5ffb79fc7a1b3d98be4de94/a306dba941cd6a6c56c65a08df879fa3033608ba#upgrade-luks).

If one does not use a `Raspberry Pi 4` with an on-board `EEPROM`, on which the `bootloader` is installed, but a separate Linux system, please `proceed with` [Creating a backup of the SD card](#creating-a-backup-of-the-sd-card-1) and then `skip to` [Re-encrypting the partition](#re-encrypting-the-partition).

### Creating a backup of the SD card
Before doing any changes, create a `backup` of the SD card, since the following commands can corrupt data:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_reencrypt.img" bs="512b" conv="fsync" status="progress"
```

### Configuring the bootloader
Boot into `Raspbian` of the Raspberry Pi, where the `root partition` should be re-encrypted and change the `boot order` of the `bootloader`:
```bash
$ rpi-eeprom-config --edit
#BOOT_ORDER=0xf41
# change boot order to "USB (0x4)", "SD (0x1)", "RESTART (0xf)"
BOOT_ORDER=0xf14
```

[Source](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#BOOT_ORDER)

### Adapting the modifications of the bootloader
Connect the `USB stick` to the Raspberry Pi and `adapt all changes` of the bootloader by `rebooting` the system:
```bash
$ reboot
```

The Raspberry Pi should now boot from the `USB stick`. Connect to it via `SSH` and proceed with the next step.

### Re-encrypting the partition
`Re-encrypt` the `root-partition` via `cryptsetup-reencrypt`:
```bash
$ cryptsetup-reencrypt --cipher="xchacha20,aes-adiantum-plain64" --key-size="256" "/dev/mmcblk0p2"
Enter passphrase for key slot 0: raspberry
Progress:   5.0%, ETA 19:38,  744 MiB written, speed  12.0 MiB/s
```

Note, newer versions of `cryptsetup` use `cryptsetup reencrypt`.

This process may take up to `30 minutes`.

### Reverting the modifications of the bootloader
After that, `revert the boot order` of the bootloader and `reboot` to apply all changes:
```bash
$ rpi-eeprom-config --edit
BOOT_ORDER=0xf41
$ reboot
```

### Verifying the new cipher method
Finally, verify, if the re-encryption was successful:
```bash
$ cryptsetup status cryptroot
/dev/mapper/cryptroot is active and is in use.
  type:    LUKS2
  cipher:  xchacha20,aes-adiantum-plain64
  keysize: 256 bits
  key location: keyring
  device:  /dev/mmcblk0p2
  sector size:  4096
  offset:  32768 sectors
  size:    7720844 sectors
  mode:    read/write
```

The `USB stick` can now be disconnected.

# Known issues
**Note: For some reason, following the instructions [Encrypting the root partition manually](#encrypting-the-root-partition-manually) will provide an image file, which is **only compatible** with the Raspberry Pi revision, on which the `root partition` was resized.**

So, using an image, where the `root partition` was previously resized on a `Raspberry Pi Model B Rev 2` is **incompatible** with a `Raspberry Pi 4 Model B Rev 1.4`.

It will boot into the `initramfs`, but the `root partition` cannot be decrypted. Skipping [Resizing the root partition](#resizing-the-root-partition) will also preserve the partition size and its `UUID`, but it still will not work.
