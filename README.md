Table of Contents
=================
* [Table of Contents](#table-of-contents)
* [Introduction](#introduction)
* [Using the modified image](#using-the-modified-image)
    * [Prerequisites](#prerequisites)
    * [Downloading the image](#downloading-the-image)
    * [Copying the image to the SD card](#copying-the-image-to-the-sd-card)
    * [Extending the root partition](#extending-the-root-partition)
        * [Creating a backup of the SD card](#creating-a-backup-of-the-sd-card)
        * [Analysing the root partition](#analysing-the-root-partition)
        * [Extending the root partition](#extending-the-root-partition-1)
        * [Calculating new LUKS device size](#calculating-new-luks-device-size)
        * [Extending the filesystem](#extending-the-filesystem)
        * [Verifying the filesystem](#verifying-the-filesystem)
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
    * [Copying the modified image to smaller data storage devices](#copying-the-modified-image-to-smaller-data-storage-devices)
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
    * [Shrinking the modified image](#shrinking-the-modified-image)
        * [Creating a backup of the image](#creating-a-backup-of-the-image)
        * [Analysing the filesystem](#analysing-the-filesystem)
        * [Shrinking the filesystem](#shrinking-the-filesystem)
        * [Shrinking the root partition](#shrinking-the-root-partition)
        * [Truncating the modified image](#truncating-the-modified-image)
        * [Verifying the data integrity](#verifying-the-data-integrity)
        * [Copying the modified image to the SD card](#copying-the-modified-image-to-the-sd-card)
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
    * [Compatibility](#compatibility)

# Introduction
This repository shall describe all necessary steps, in order to encrypt the `root partition` of the Raspberry Pi stock image `Raspberry Pi OS Lite` of `Debian 12 (Bookworm)` on a `Raspberry Pi 4 Model B Rev 1.4`.

The instructions are adaptable for `other Raspberry Pi revisions` as well. They should also work on `image files with partition information` in general.

The entire setup was done on an `amd64-based computer` with [`Gentoo Linux (Kernel 6.6.67)`](https://www.gentoo.org/downloads/).

**Please read [Known issues](#known-issues) first before following any of the instructions!**

# Using the modified image
## Prerequisites
* The following packages are installed:
```no-highlight
coreutils
cryptsetup-2.0.6 or higher
e2fsprogs
gnupg
linux-image-6.6.51+rpt-rpi-v8 or higher
linux-image-rpi-v8
parted
util-linux
```
* `Linux Kernel version 5.0` or higher and `cryptsetup-2.0.6` or higher are required to support the fast `software-based` encryption method `aes-adiantum-plain64`, since the Raspberry Pi's CPU does not support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`).
* The capacity of the `SD card` must be greater than `3.3 GiB`.

If one is using a `Raspberry Pi 5`, the encryption method `aes-xts-plain64` with a `key size` of [`512 bits`](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode) may be preferred.

## Downloading the image
Download the files from [gofile.io](https://gofile.io/d/3H8TnE).

If the links are dead, due to infrequent downloads, please `proceed with` [Encrypting the root partition manually](#encrypting-the-root-partition-manually).

Check the `data integrity` and `verify` the signature:
```bash
$ b2sum --check "raspberrypi_sd_card_backup.img.b2"
raspberrypi_sd_card_backup.img: OK
$ gpg --verify "raspberrypi_sd_card_backup.img.b2.asc" "raspberrypi_sd_card_backup.img.b2"
gpg: Signature made Sat Feb  8 15:37:15 2025 CET
gpg:                using RSA key 598398DA5F4DA46438FDCF87155BE26413E699BF
gpg: Good signature from "Ramon Fischer (ramon@sharkoon) <RamonFischer24@googlemail.com>" [ultimate]
gpg:                 aka "Ramon Fischer (ramon@sharkoon) <Ramon_Fischer@hotmail.de>" [ultimate]
```

## Copying the image to the SD card
Copy the image to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="1M" conv="fsync" status="progress"
```

## Extending the root partition
When copying the image to other data storage devices with `higher capacity`, the `encrypted root partition` will stay at `~3.3 GiB`. Therefore, it needs to be `extended`, in order to use the `unused free space`.

### Creating a backup of the SD card
Before making any changes, create a `backup` of the SD card, since the following commands `can corrupt data`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_extending.img" bs="1M" conv="fsync" status="progress"
```

### Analysing the root partition
After that, boot into `Raspberry Pi OS (Debian)` with [predefined user credentials](#user-credentials) and check the partition structure via `parted`:
```bash
$ parted --list
Model: Linux device-mapper (crypt) (dm)
Disk /dev/mapper/cryptroot: 3963MB
Sector size (logical/physical): 4096B/4096B
Partition Table: loop
Disk Flags:

Number  Start  End     Size    File system  Flags
 1      0.00B  3963MB  3963MB  ext4


Model: SD SC64G (sd/mmc)
Disk /dev/mmcblk0: 61.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  541MB   537MB   primary  fat32        lba
 2      541MB   4521MB  3980MB  primary
```

This indicates, that `/dev/mmcblk0p2` (`/dev/mapper/cryptroot`) only has a size of `~4 GB`, but the SD card's total capacity is `61.5 GB`.

### Extending the root partition
In order to `extend` the `second partition`, execute the following commands:
```bash
$ parted "/dev/mmcblk0"
GNU Parted 3.2
Using /dev/mmcblk0
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Model: SD SC64G (sd/mmc)
Disk /dev/mmcblk0: 61.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  541MB   537MB   primary  fat32        lba
 2      541MB   4521MB  3980MB  primary

(parted) resizepart
Partition number? 2
End?  [61.5GB]? -1
(parted) print
Model: SD SC64G (sd/mmc)
Disk /dev/mmcblk0: 61.5GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system  Flags
 1      4194kB  541MB   537MB   primary  fat32        lba
 2      541MB   61.5GB  61.0GB  primary

(parted) quit
```

The command `resizepart` is used to `extend` the partition, where `-1` defines the `last sector` of the SD card.

### Calculating new LUKS device size
After `extending` the partition via `parted`, the new `LUKS device size` needs to be `calculated` via `cryptsetup` as well:
```bash
$ cryptsetup resize cryptroot
Enter passphrase for /dev/mmcblk0p2: raspberry
```

### Extending the filesystem
Once this is done, use `resize2fs` to `extend` the filesystem:
```bash
$ resize2fs "/dev/mapper/cryptroot"
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/cryptroot is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/mapper/cryptroot is now 14884108 (4k) blocks long.
```

Note, that the filesystem size is now `~56.78 GiB`.

### Verifying the filesystem
All changes are now applied properly:
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
  size:    119072864 sectors
  mode:    read/write
```

Be aware, that the `size` is in `512-Byte-sectors`, even, if `sector size` is indicated as `4096 Bytes`. ["This is a relict from the time, when only `512-byte-sectors` were supported"](https://gitlab.com/cryptsetup/cryptsetup/-/issues/884#note_1899199290).

To check, if the values are correct, the following formula can be used:
```no-highlight
(<sector_size> * <size>) / 1024^3 = <size_in_gib>
```

That is:
```no-highlight
(512 Bytes * 119072864) / 1024^3 = ~56.78 GiB
```

The result differs slightly from the output of `parted`, since the unit is in `Gibibyte (base 2)` and not `Gigabyte (base 10)`.

For good measure, the command `dumpe2fs` can be used to see the `real sector size (block size)` and `real size (block count)` of the `decrypted root partition`:
```bash
$ dumpe2fs "/dev/mapper/cryptroot" | grep "Block " | head --lines="2"
Block count:              14884108
Block size:               4096
```

Which leads to the `same result` as above:
```no-highlight
(4096 Bytes * 14884108) / 1024^3 = ~56.78 GiB
```

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
fping
netcat
openssl
parted
qemu-user-static
util-linux
xz-utils
```

### Raspberry Pi
* The following packages are installed:
```no-highlight
cryptsetup-2.0.6 or higher
linux-image-6.6.51+rpt-rpi-v8 or higher
linux-image-rpi-v8
```

* `qemu-user-static` is needed, if one is working on a `non-ARM operating system`.
* `Linux Kernel version 5.0` or higher and `cryptsetup-2.0.6` or higher are required to support the fast `software-based` encryption method `aes-adiantum-plain64`, since the Raspberry Pi's CPU does not support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`).
* Free space of at least `1.5 times` the capactiy of the `SD card`

If one is using a `Raspberry Pi 5`, the encryption method `aes-xts-plain64` with a `key size` of [`512 bits`](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode) may be preferred.

## Downloading the stock image
Download the image `Raspberry Pi OS Lite` from the [official page](https://www.raspberrypi.org/software/operating-systems/) and also save its `SHA256` checksum file:

```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" --force-sequential="true" \
    "https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-11-19/2024-11-19-raspios-bookworm-arm64-lite.img.xz" \
    "https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2024-11-19/2024-11-19-raspios-bookworm-arm64-lite.img.xz.sha256"
```

Verify the checksum of the archive:
```bash
$ sha256sum --check "2024-11-19-raspios-bookworm-arm64-lite.img.xz.sha256"
2024-11-19-raspios-bookworm-arm64-lite.img.xz: OK
```

## Configuration
### Preparation
Clone the repository to the `current working directory`:
```bash
$ git clone "https://codeberg.org/keks24/raspberry-pi-luks.git"
```

Unarchive `2024-11-19-raspios-bookworm-arm64-lite.img.xz`:
```bash
$ xz --decompress --verbose "2024-11-19-raspios-bookworm-arm64-lite.img.xz"
2024-11-19-raspios-bookworm-arm64-lite.img.xz (1/1)
  100 %     437.7 MiB / 2,628.0 MiB = 0.167   427 MiB/s       0:06
```

Copy the image `2024-11-19-raspios-bookworm-arm64-lite.img` to the `SD card`:
```bash
$ dd if="2024-11-19-raspios-bookworm-arm64-lite.img" of="/dev/sdx" bs="1M" conv="fsync" status="progress"
```

Since `Debian Bullseye` a user needs to be added [manually](https://www.raspberrypi.com/news/raspberry-pi-bullseye-update-april-2022/), in order to be able to log in. Therefore the `boot partition` needs to be mounted:
```bash
$ mount "/dev/sdx1" "/mnt/"
```

Next, create the file [`userconf`](https://www.raspberrypi.com/documentation/computers/configuration.html#configuring-a-user) in the directory `/mnt/`:
```bash
$ echo "pi:$(openssl passwd -6)" > "/mnt/userconf"
Password: raspberry
Verifying - Password: raspberry
```

This file will be read `on boot` and the user `pi` will be created with [default credentials](#user-credentials). This user is preferred, since there are `predefined sudo rules` at `/etc/sudoers.d/010_pi-nopasswd` on the `root partition`, which allow to `escalate privileges without password`.

In order to be able to `establish an SSH connection` to the Raspberry Pi, the `empty` file `ssh` needs to be added:
```bash
$ touch "/mnt/ssh"
```

Unmount the `boot partition`:
```bash
$ umount "/mnt/"
```

Insert the `SD card` into the Raspberry Pi and boot into `Raspberry Pi OS (Debian)`. This will create the user `pi`, the daemon `sshd` will be started and the `root partition` will be extended to its full capacity.

It may be necessary to `scan the network` for `active devices` and `probe the port 22 (SSH)`, in order to find the Raspberry Pi:
```bash
$ fping --alive --generate 192.168.1.0/24 2>/dev/null
192.168.1.1
192.168.1.2
192.168.1.3
192.168.1.80
[...]
$ nc -zvn 192.168.1.80 22
(UNKNOWN) [192.168.1.80] 22 (ssh) open
```

In this case, it is `192.168.1.80`.

After that, `establish an SSH connection` to the Raspberry Pi, `log in` with the [predefined user credentials](#user-credentials) and `update the kernel`:
```bash
$ ssh pi@192.168.1.80
pi@192.168.0.30's password: raspberry
[...]
$ sudo apt update
$ uname --kernel-release
6.6.51+rpt-rpi-v8
$ sudo apt install linux-image-rpi-v8 --only-upgrade
$ sudo poweroff
```

All `installed kernel packages` can be looked up with the following command:
```bash
$ dpkg --list "linux-image*"
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name                                     Version         Architecture Description
+++-========================================-===============-============-=============================================
ii  linux-image-6.6.51+rpt-rpi-2712          1:6.6.51-1+rpt3 arm64        Linux 6.6 for Raspberry Pi 2712, Raspberry Pi
un  linux-image-6.6.51+rpt-rpi-2712-unsigned <none>          <none>       (no description available)
ii  linux-image-6.6.51+rpt-rpi-v8            1:6.6.51-1+rpt3 arm64        Linux 6.6 for Raspberry Pi v8, Raspberry Pi
un  linux-image-6.6.51+rpt-rpi-v8-unsigned   <none>          <none>       (no description available)
ii  linux-image-6.6.74+rpt-rpi-2712          1:6.6.74-1+rpt1 arm64        Linux 6.6 for Raspberry Pi 2712, Raspberry Pi
un  linux-image-6.6.74+rpt-rpi-2712-unsigned <none>          <none>       (no description available)
ii  linux-image-6.6.74+rpt-rpi-v8            1:6.6.74-1+rpt1 arm64        Linux 6.6 for Raspberry Pi v8, Raspberry Pi
un  linux-image-6.6.74+rpt-rpi-v8-unsigned   <none>          <none>       (no description available)
ii  linux-image-rpi-2712                     1:6.6.74-1+rpt1 arm64        Linux for Raspberry Pi 2712 (meta-package)
ii  linux-image-rpi-v8                       1:6.6.74-1+rpt1 arm64        Linux for Raspberry Pi v8 (meta-package)
```

Next, create a `backup` of the `SD card`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup.img" bs="1M" conv="fsync" status="progress"
```

Analyse the image for its partition `start sectors` and the `logical sector size`:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 120176640s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start     End         Size        Type     File system  Flags
 1      8192s     1056767s    1048576s    primary  fat32        lba
 2      1056768s  120176639s  119119872s  primary  ext4
```

In this case, the `first partition (boot)` starts at `sector 8192` and the `second partition (root)` at `sector 1056768`. The `logical sector size` is `512 Bytes`.

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
NAME       SIZELIMIT    OFFSET AUTOCLEAR RO BACK-FILE                                                                                 DIO LOG-SEC
/dev/loop1         0   4194304         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
/dev/loop2         0 541065216         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
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
Since the preparation is done, the `root partition` can now be `overwritten with random Bytes` before formatting, in order to `decrease the chance of external recovery` from `third parties`:
```bash
$ shred --iterations="1" --random-source="/dev/urandom" --zero --verbose "/dev/loop2"
```

The command `shred` overwrites the `root partition` with `random Bytes` first and then with `zeroes`; the latter `obfuscates` the shredding. The special character device [`/dev/urandom`](https://www.thomas-huehn.com/myths-about-urandom/) is used as preferred `entropy source`, since it behaves like `/dev/random` since [`Kernel version 5.6`](https://en.wikipedia.org/w/index.php?title=/dev/random&oldid=1268697417#Linux), but it will `not block` the process, if there is `insufficient entropy`. The command comes with an [internal pseudo-random generator](https://www.gnu.org/software/coreutils/manual/html_node/Random-sources.html#Random-sources), which will be used, if the parameter `--random-source` is left out; this would `accelerate` the write process, but it is `not true random`.

After that, `format` and `encrypt` the `root partition` via `cryptsetup`:
```bash
$ cryptsetup --cipher="xchacha20,aes-adiantum-plain64" --key-size="256" --sector-size="4096" luksFormat "/dev/loop2"

WARNING!
========
This will overwrite data on /dev/loop2 irrevocably.

Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase for /root/tmp/raspberrypi_sd_card_backup.img:
Verify passphrase: raspberry
```

It is recommended to use `aes-adiantum-plain64`, since the CPU does **not** support `hardware accelerated AES` (`grep "Features" "/proc/cpuinfo"`). The `sector size` of `4096 Bytes` is preferred, since it comes with a [performance gain](https://lwn.net/Articles/776959/). If one is using a `Raspberry Pi 5`, the encryption method `aes-xts-plain64` with a `key size` of [`512 bits`](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode) may be preferred.

The `LUKS header information` looks like so:
```bash
$ cryptsetup luksDump "/dev/loop2"
LUKS header information
Version:        2
Epoch:          3
Metadata area:  16384 [bytes]
Keyslots area:  16744448 [bytes]
UUID:           1eee5e92-e843-4dd8-b663-5a3cf7b32361
Label:          (no label)
Subsystem:      (no subsystem)
Flags:          (no flags)

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
        PBKDF:      argon2id
        Time cost:  4
        Memory:     1048576
        Threads:    4
        Salt:       fa 2b 44 06 38 59 32 32 ea 62 6a 56 ce 05 44 64
                    c1 89 05 94 3e 0f 7b be 1a 24 6a 7b d1 df ba 6d
        AF stripes: 4000
        AF hash:    sha256
        Area offset:32768 [bytes]
        Area length:131072 [bytes]
        Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
        Hash:       sha256
        Iterations: 173375
        Salt:       2f 3b 74 c8 e0 c9 1c d1 52 dc 69 d5 72 2b 3b 74
                    15 42 0a 17 e0 b1 3d e2 46 9f 18 54 c1 d4 78 6a
        Digest:     06 bf 10 72 07 c7 5e 01 ba cf ec 40 79 1a 08 e7
                    eb 44 63 b7 91 c1 7e 77 c4 e3 d6 b0 ca 6b 64 39
```

Other `encryption methods` are supported as well and can be looked up [here](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/LUKS-standard/on-disk-format.pdf#cipher-and-hash-specification-registry).

If the encryption method `aes-xts-plain64` is preferred, make absolutely sure, that the `key size` is `at least 512 bits`, [since `XTS` splits the key size in half](https://wiki.archlinux.org/index.php/dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode).

Now, open/decrypt the encrypted `root partition` and format it via `mkfs.ext4`:
```bash
$ cryptsetup open "/dev/loop2" cryptroot
Enter passphrase for /root/tmp/raspberrypi_sd_card_backup.img: raspberry
$ mkfs.ext4 "/dev/mapper/cryptroot"
mke2fs 1.47.1 (20-May-2024)
Creating filesystem with 14885888 4k blocks and 3727360 inodes
Filesystem UUID: 9f98f16c-9779-49b5-89d6-4be7be1937a5
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424

Allocating group tables: done
Writing inode tables: done
Creating journal (65536 blocks): done
Writing superblocks and filesystem accounting information: done
```

After that, it is possible to mount the decrypted partition `/dev/mapper/cryptroot`:
```bash
$ mount "/dev/mapper/cryptroot" "/mnt/"
```

Finally, restore the backup:
```bash
$ rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" "root_backup/" "/mnt/"
```

### Entering the chroot environment
Once this is done, it is time to go into a `chroot environment`.

Before doing so, mount the `boot partition` to `/mnt/boot/firmware/`:
```bash
$ mount "/dev/loop1" "/mnt/boot/firmware/"
```

Then, prepare the `chroot environment`:
```bash
$ mount --types proc "/proc/" "/mnt/proc/"
$ mount --rbind "/sys" "/mnt/sys/"
$ mount --make-rslave "/mnt/sys/"
$ mount --rbind "/dev/" "/mnt/dev/"
$ mount --make-rslave "/mnt/dev/"
$ cp "/usr/bin/qemu-aarch64-static" "/mnt/usr/bin/"
$ cp --dereference "/etc/resolv.conf" "/mnt/etc/"
```

**`qemu-aarch64-static` is mandatory, if one is working on a `non-ARM operating system`!**

`/etc/resolv.conf` contains entries of `DNS name servers`, which are required within the `chroot environment`, in order to make `reverse DNS lookups` possible.

Enter the new environment:
```bash
$ chroot "/mnt/" qemu-aarch64-static "/bin/bash"
$ source "/etc/profile"
$ export PS1="(chroot) ${PS1}"
(chroot) $ cd
```

#### Installing necessary tools
An `initramfs` is needed, in order to decrypt the `root partition` on boot. The following packages will provide all tools to build it:
```bash
(chroot) $ apt update
(chroot) $ apt install busybox cryptsetup cryptsetup-initramfs initramfs-tools
```

This will automatically trigger `update-initramfs`, which will install the `zstd-compressed` file `initramfs8` to `/boot/firmware/initramfs8`.

#### Configuration
Next, get the `UUID` of `/dev/loop2`, which will be used later on:
```bash
(chroot) $ blkid "/dev/loop2"
/dev/loop2: UUID="1eee5e92-e843-4dd8-b663-5a3cf7b32361" TYPE="crypto_LUKS"
```

Edit the kernel parameters in `/boot/firmware/cmdline.txt` and add an entry in `/etc/crypttab`, so the `root partition` can be decrypted on boot:
```bash
(chroot) $ vi "/boot/firmware/cmdline.txt"
root=/dev/mapper/cryptroot cryptdevice=UUID=1eee5e92-e843-4dd8-b663-5a3cf7b32361:cryptroot
(chroot) $ vi "/etc/crypttab"
cryptroot UUID=1eee5e92-e843-4dd8-b663-5a3cf7b32361 none luks,initramfs
```

Note, that adding the option `initramfs` is very important here. Otherwise, the following error might occur and the file `/cryptroot/crypttab` within the `initramfs` might be empty; **rendering the system unbootable**:
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

Edit the configuration file `/etc/initramfs-tools/modules`, in order to make `cryptsetup` work in the `initramfs`:
```bash
$ vi "/etc/initramfs-tools/modules"
# see in the following directories for available modules:
## "/lib/modules/<some_kernel_version>/kernel/crypto/"
## "/lib/modules/<some_kernel_version>/kernel/arch/arm64/crypto/"
## "/lib/modules/<some_kernel_version>/kernel/lib/crypto/"
adiantum
aes_arm64
algif_skcipher
dm-crypt
nhpoly1305
sha256
xchacha20
```

Once this is done, the `initramfs` can now be built.

#### Generating the initramfs
There are three ways to generate the `initramfs`.

1. Either reinstall the package `linux-image-6.6.74+rpt-rpi-v8`, which will only install that specific kernel and then executes the `hook scripts` in `/etc/kernel/postrm.d/` and `/etc/kernel/postinst.d/`:
```bash
(chroot) $ apt install linux-image-6.6.74+rpt-rpi-v8 --reinstall
```

2. Execute the command `mkinitramfs` manually:
```bash
(chroot) $ mkinitramfs -o "/boot/firmware/initramfs8" 6.6.74+rpt-rpi-v8
```

3. Or execute `update-initramfs` manually:
```bash
(chroot) $ update-initramfs -vuk 6.6.74+rpt-rpi-v8
[...]
Keeping /boot/initrd.img-6.6.74+rpt-rpi-v8.dpkg-bak
update-initramfs: Generating /boot/initrd.img-6.6.74+rpt-rpi-v8
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libpoly1305.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/adiantum.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libaes.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/aes_generic.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/aes-arm64.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/af_alg.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/algif_skcipher.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-mod.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-crypt.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/nhpoly1305.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha256-arm64.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha2-ce.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libchacha.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/chacha-neon.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/chacha_generic.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/uio/uio.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/uio/uio_pdrv_genirq.ko.xz
[...]
Building cpio /boot/initrd.img-6.6.74+rpt-rpi-v8.new initramfs
'/boot/initrd.img-6.6.74+rpt-rpi-v8' -> '/boot/firmware/initramfs8'
```

For this setup, the `third method is preferred`, which can be executed now.
```bash
(chroot) $ update-initramfs -vuk 6.6.74+rpt-rpi-v8
```

The file `modification date and time` of the `initramfs` can be checked with the command `stat`:
```bash
$ (chroot) stat "/boot/firmware/initramfs8" | grep "Modify"
Modify: 2025-02-07 21:37:16.000000000 +0000
```

After the reinstallation has been completed, add the entry `initramfs initramfs8 followkernel` at the end of the configuration file `/boot/firmware/config.txt`:
```bash
$ (chroot) vi "/boot/firmware/config.txt"
[all]
initramfs initramfs8 followkernel
```

Make sure, that the following important files are present in the initramfs file `/boot/firmware/initramfs8`:
```bash
(chroot) $ lsinitramfs "/boot/firmware/initramfs8" | grep --extended-regexp "adiantum|aes-arm|algif_skcipher|dm-crypt|nhpoly|sha256|chacha|crypttab|sbin/cryptsetup|usr/bin/cryptroot-unlock"
cryptroot/crypttab
usr/bin/cryptroot-unlock
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/aes-arm64.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/chacha-neon.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha256-arm64.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/adiantum.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/algif_skcipher.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/chacha_generic.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/nhpoly1305.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-crypt.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libchacha.ko
usr/sbin/cryptsetup
usr/bin/sha256sum
```

Also make sure, that the content of `/cryptroot/crypttab` is correct.

Detailed debugging is explained [below](#debugging).

#### Exiting the chroot environment
Exit the `chroot`, unmount the `boot partition` and all `pseudo filesystems`:
```bash
(chroot) $ exit
$ cd
$ umount "/mnt/boot/firmware/"
$ umount --lazy --recursive /mnt/{proc/,sys/,dev/}
```

Remove `qemu-aarch64-static` from `/mnt/usr/bin/`:
```bash
$ rm "/mnt/usr/bin/qemu-aarch64-static"
```

Unmount the decrypted `root partition` and detach all `loop devices`:
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
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="1M" conv="fsync" status="progress"
```

On boot there should be a message to decrypt the `root partition`:
```no-highlight
Please unlock disk cryptroot: raspberry
```

After entering the password, the `Raspberry Pi` should boot into `Raspberry Pi OS (Debian)`.

# Further steps
## Copying the modified image to smaller data storage devices
In order to copy the modified image to `smaller` data storage devices, the `filesystem` and `partition information` and the `image size` need to be `adjusted` accordingly; see [Shrinking the modified image](#shrinking-the-modified-image) below.

## Updating all installed packages
Since only some of the above mentioned packages have been upgraded, the probability is very high, that `new packages` are available. Update them using the following commands:
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
All configuration files can be found at `/etc/dropbear/initramfs/` and are self-explained.

Next, configure `dropbear` by editing `/etc/dropbear/initramfs/dropbear.conf`:
```bash
$ vi "/etc/dropbear/initramfs/dropbear.conf"
DROPBEAR_OPTIONS="-p 22222 -I 60 -sjk"
```

This sets the `SSH port` to `22222`, sets the `idle timeout` to `60 seconds` and disables `logins via password`, `local port forwarding` and `remote port forwarding`.

Further information can be looked up at `man 8 dropbear`.

Before adding the public key, the file `authorized_keys` should be created like so:
```bash
$ touch "/etc/dropbear/initramfs/authorized_keys"
$ chmod 600 "/etc/dropbear/initramfs/authorized_keys"
```

Since [`Debian 11 (Bullseye)`](https://packages.debian.org/bullseye/dropbear) dropbear supports [`ed25519 keys`](https://github.com/mkj/dropbear/releases/tag/DROPBEAR_2020.79), which will be created on the `host` like this:
```bash
$ ssh-keygen -t ed25519 -f "/home/<some_username>/.ssh/dropbear_root_ed25519"
Generating public/private ed25519 key pair.
Enter passphrase (empty for no passphrase): <some_password>
Enter same passphrase again: <some_password>
Your identification has been saved in /home/<some_username>/.ssh/dropbear_root_ed25519
Your public key has been saved in /home/<some_username>/.ssh/dropbear_root_ed25519.pub
The key fingerprint is:
SHA256:ZJmQeVDrJpSM+URlKV16DIxM+dhDkjrMXn33Al/FRvs <some_username>@<some_hostname>
The key's randomart image is:
+--[ED25519 256]--+
|     o=%=o.    o.|
|     =X+*B      =|
|   oo.=XB o    + |
|    =+o+=.o . . .|
|   . oo So + o  E|
|    .  o    o .  |
|             .   |
|                 |
|                 |
+----[SHA256]-----+
```

Copy the contents of the `SSH public key` `/home/<some_username>/.ssh/dropbear_root_ed25519.pub` to the `Raspberry Pi` to `/etc/dropbear/initramfs/authorized_keys` with the following `SSHD options`:
```bash
no-port-forwarding,no-agent-forwarding,no-x11-forwarding,command="/usr/bin/cryptroot-unlock" ssh-ed25519 [...]
```

The option `command` restricts the logged in user `root` to only execute the command `/usr/bin/cryptroot-unlock` within the `initramfs`. All other options should be self-explained, but can be looked up at `man 8 sshd`. The path to the binary can be determined by [examining](#examining-the-initramfs) or [unarchiving the content of the `initramfs`](#unarchiving-the-initramfs).

Removing the option `command` will grant access to the `busybox` as user `root`.

Make sure, that the `SSH public key` is copied correctly, since `logins via password` are disabled.

### Configuring kernel parameters
In order to make the `initramfs` available in the network on boot, set a `static IP address` via kernel parameters. Append the following line in `/boot/firmware/cmdline.txt`:
```bash
$ vi "/boot/firmware/cmdline.txt"
ip=192.168.1.80:::255.255.255.0
```

The kernel parameter `ip` has the following syntax:
```no-highlight
ip=<client_ip>:<server_ip>:<gateway_ip>:<netmask>:<hostname>:<network_interface>:<autoconf>:<dns0_ip>:<dns1_ip>:<ntp0_ip>
```

[Source](https://www.kernel.org/doc/Documentation/filesystems/nfs/nfsroot.txt)

This will set the `private Class C IP address` to `192.168.1.80` and the `subnet mask` to `255.255.255.0`; these values may differ depending on the network infrastructure. The `initramfs` does not have any connection to the `internet`, since no `gateway IP address` is set.

Make sure, that the `Raspberry Pi` and the `host`, from which the partition should be decrypted, are in the same network.

### Rebuilding the initramfs
`Rebuild` the `initramfs` to adapt all changes:
```bash
$ update-initramfs -vuk 6.6.74+rpt-rpi-v8
[...]
Keeping /boot/initrd.img-6.6.74+rpt-rpi-v8.dpkg-bak
update-initramfs: Generating /boot/initrd.img-6.6.74+rpt-rpi-v8
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libpoly1305.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/adiantum.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/af_alg.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/algif_skcipher.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/cryptd.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-mod.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-crypt.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/nhpoly1305.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha256-arm64.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha2-ce.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libchacha.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/chacha-neon.ko.xz
Adding module /usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/chacha_generic.ko.xz
[...]
Calling hook dropbear
Adding binary /usr/sbin/dropbear
[...]
Building cpio /boot/initrd.img-6.6.74+rpt-rpi-v8.new initramfs
'/boot/initrd.img-6.6.74+rpt-rpi-v8' -> '/boot/firmware/initramfs8'
```

Make sure, that the binary `dropbear` and its `configuration files` are present in the initramfs file `/boot/firmware/initramfs8`:
```bash
$ lsinitramfs "/boot/firmware/initramfs8" | grep --extended-regexp "dropbear|authorized_keys"
etc/dropbear
etc/dropbear/dropbear.conf
etc/dropbear/dropbear_ecdsa_host_key
etc/dropbear/dropbear_ed25519_host_key
etc/dropbear/dropbear_rsa_host_key
root-6a1XMV4VQT/.ssh/authorized_keys
scripts/init-bottom/dropbear
scripts/init-premount/dropbear
usr/sbin/dropbear
```

Also make sure, that the content of `/root-<some_random_alphanumeric_characters>/.ssh/authorized_keys` and `/etc/dropbear/dropbear.conf` are correct.

Detailed debugging is explained [here](#debugging).

### Rebooting
After that, `reboot` the `Raspberry Pi`:
```bash
$ reboot
```

### Testing remote decryption
The `initramfs` should be now accessable via `SSH` on `port 22222`, which can be accessed like so:
```bash
$ ssh -p 22222 root@192.168.1.80 -i "/home/<some_username>/.ssh/dropbear_root_ed25519"
The authenticity of host '[192.168.1.80]:22222 ([192.168.1.80]:22222)' can't be established.
ED25519 key fingerprint is SHA256:<some_random_alphanumeric_characters>.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[192.168.1.80]:22222' (ED25519) to the list of known hosts.
Enter passphrase for key '/home/<some_username>/.ssh/dropbear_root_ed25519': <some_password>
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

Logging in via `SSH (port 22)` should now be possible:
```bash
$ ssh pi@192.168.0.30
pi@192.168.0.30's password: raspberry
```

### Optional fancy SSH ASCII banner
`dropbear` allows to set a custom `ASCII banner`, which is shown, when connecting to the `initramfs`.

#### Configuring dropbear
To do this, the parameter `-b` has to be appended to the configuration file `/etc/dropbear/initramfs/dropbear.conf`:
```bash
$ vi "/etc/dropbear/initramfs/dropbear.conf"
DROPBEAR_OPTIONS="-p 22222 -I 60 -sjk -b etc/dropbear/ssh_banner.net"
```

**Leaving out the `leading slash` is important!**

#### Generating a fancy ASCII banner
Then, generate a `fancy ASCII banner` via `figlet` and save it to `/etc/dropbear/initramfs/ssh_banner.net`:
```bash
$ apt update
$ apt install figlet
$ figlet -w 100 -f slant "decrypt cryptroot" > "/etc/dropbear/initramfs/ssh_banner.net"
$ echo "" >> "/etc/dropbear/initramfs/ssh_banner.net"
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
Finally, rebuild the `initramfs` and reboot:
```bash
$ update-initramfs -vuk 6.6.74+rpt-rpi-v8
[...]
Calling hook dropbear
Adding file /etc/dropbear/initramfs/ssh_banner.net
[...]
Building cpio /boot/initrd.img-6.6.74+rpt-rpi-v8.new initramfs
'/boot/initrd.img-6.6.74+rpt-rpi-v8' -> '/boot/firmware/initramfs8'
$ reboot
```

#### Rebooting
After rebooting the `Raspberry Pi`, `dropbear` will now display a `fancy ASCII banner`:
```bash
$ ssh -p 22222 root@192.168.1.80 -i "/home/<some_username>/.ssh/dropbear_root_ed25519"
       __                           __                           __                   __
  ____/ /__  ____________  ______  / /_   ____________  ______  / /__________  ____  / /_
 / __  / _ \/ ___/ ___/ / / / __ \/ __/  / ___/ ___/ / / / __ \/ __/ ___/ __ \/ __ \/ __/
/ /_/ /  __/ /__/ /  / /_/ / /_/ / /_   / /__/ /  / /_/ / /_/ / /_/ /  / /_/ / /_/ / /_
\__,_/\___/\___/_/   \__, / .___/\__/   \___/_/   \__, / .___/\__/_/   \____/\____/\__/
                    /____/_/                     /____/_/

Please unlock disk cryptroot: raspberry
```

## Shrinking the modified image
When copying the image to other data storage devices with `lower capacity`, the `filesystem` and `partition information` and the `image size` need to be `shrunk`, in order to fit on the new device.

**This can only be done, if the image is not in use; it is not possbile to do this `on-the-fly`!**

### Creating a backup of the image
Before making any changes, create a `backup` of the `modified image`, since the following commands `can corrupt data`:
```bash
$ cp --archive --verbose raspberrypi_sd_card_backup{,_before_shrinking}.img
'raspberrypi_sd_card_backup' -> 'raspberrypi_sd_card_backup_before_shrinking.img'
```

### Analysing the filesystem
After that, check, that there are free `loop devices` at `/dev/`:
```bash
$ losetup --list
```

Then, decrypt the `root partition`:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 120176640s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start     End         Size        Type     File system  Flags
 1      8192s     1056767s    1048576s    primary  fat32        lba
 2      1056768s  120176639s  119119872s  primary
$ losetup --offset="$(( 512 * 1056768 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcardbackup
Enter passphrase for /dev/loop2: raspberry
```

Once this is done, `verify the filesystem integrity` via `e2fsck`:
```bash
$ e2fsck -f "/dev/mapper/cryptsdcardbackup"
e2fsck 1.47.1 (20-May-2024)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

       82247 inodes used (2.21%, out of 3727360)
          22 non-contiguous files (0.0%)
          68 non-contiguous directories (0.1%)
             # of inodes with ind/dind/tind blocks: 0/0/0
             Extent depth histogram: 76270/10
      922976 blocks used (6.20%, out of 14885888)
           0 bad blocks
           1 large file

       70052 regular files
        6126 directories
           8 character device files
           0 block device files
           0 fifos
         366 links
        6052 symbolic links (5951 fast symbolic links)
           0 sockets
------------
       82604 files
```

The parameter `-f` `forces` the filesystem check, even, if it is clean.

### Shrinking the filesystem
Next, `determine` the lowest filesystem size possible via `resize2fs`. This command can handle the `filesystem type ext4`. Moreover, it is capable of `moving data on-demand`, so defragmentation before this process via `e4defrag` is **not needed**:
```bash
$ resize2fs "/dev/mapper/cryptsdcardbackup" "1"
resize2fs 1.47.1 (20-May-2024)
resize2fs: New size smaller than minimum (889692)
```

This will return the `lowest filesystem size (889692)` in `4 Kibibyte sectors`, which was automatically determined by the underlying `ext4 filesystem`. **This value should not be used as is, since it is not taking the `LUKS header size` into consideration!**

Be aware, that the command was `intentionally` executed in a wrong way, since there is no available parameter for a `dry run`.

Further information can be looked up at `man 8 resize2fs`.

The following formula will be used to `determine` the `new filesystem size`:
```no-highlight
(<luks_header_binary_size> + <luks_header_json_size> + <luks_header_keyslots_size>) + (<encrypted_partition_size>) = <new_size_of_the_root_partition>
```

That is:
```no-highlight
(16,384 KiB + 16,384 KiB + 131,072 KiB) + (4 KiB * 889692) = 3,722,608 KiB
```

Using `Kibibytes` as unit is precise enough for the next steps. The values of the `LUKS header size` can be looked up in the diagram [below](#shrinking-the-root-partition).

Now, `resize` the `filesystem`:
```bash
$ resize2fs "/dev/mapper/cryptsdcardbackup" "3722608K"
resize2fs 1.47.1 (20-May-2024)
Resizing the filesystem on /dev/mapper/cryptsdcardbackup to 930652 (4k) blocks.
The filesystem on /dev/mapper/cryptsdcardbackup is now 930652 (4k) blocks long.
```

### Shrinking the root partition
Once this is done, the `partition information` need to be `adjusted` to the `new filesystem size`.

Before doing so, `close` the `LUKS device` and `detach` the `loop device`, in order to not have any other activity to the image:
```bash
$ cryptsetup close cryptsdcardbackup
$ losetup --detach "/dev/loop2"
$ losetup --list
```

The following diagram shows the `partition structure` in `Kibibytes`, which will be used later on:
```no-highlight
                      ┌────────────────────────────┐
                      │        LUKS 2 header       │
┌─────────┬───────────┼────────┬────────┬──────────┼────────────────┐
│ Offset  │ Boot data │ Binary │  JSON  │ Keyslots │ Encrypted data │
├─────────┼───────────┼────────┼────────┼──────────┼────────────────┤
│  4,096  │  524,288  │ 16,384 │ 16,384 │ 131,072  │    3,722,608   │
├─────────┴───────────┼────────┴────────┴──────────┴────────────────┤
│  Boot partiton (1)  │              Root partition (2)             │
└─────────────────────┴─────────────────────────────────────────────┘
```

[Source](https://gitlab.com/cryptsetup/LUKS2-docs/-/blob/main/luks2_doc_wip.pdf#luks2-on-disk-format)

As a side note: The [`keyslots limit`](https://gitlab.com/cryptsetup/cryptsetup/-/blob/main/lib/luks2/luks2.h?ref_type=heads#L27) is `hardcoded` to `32`.

This above diagram indicates, that the `boot partition size` and the `LUKS header size` need to be considered, when `shrinking` the `root partition`.

The following formula will be used, in order to `determine` the `end sector` of the `root partition` in `Kibibytes`:
```no-highlight
(<boot_offset> + <boot_data_size>) + (<luks_header_binary_size> + <luks_header_json_size> + <luks_header_keyslots_size>) + <encrypted_data_size> = <new_size_of_the_root_partition>
```

That is:
```no-highlight
(4,096 + 524,288) + (16,384 KiB + 16,384 KiB + 131,072 KiB) + 3,722,608 KiB = 4,414,832 KiB
```

Next, `shrink` the `root partition` via `parted`:
```bash
$ parted "raspberrypi_sd_card_backup.img"
GNU Parted 3.6
Using /root/tmp/raspberrypi_sd_card_backup.img
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) unit kib
(parted) print
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 60088320kiB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start      End          Size         Type     File system  Flags
 1      4096kiB    528384kiB    524288kiB    primary  fat32        lba
 2      528384kiB  60088320kiB  59559936kiB  primary

(parted) resizepart
Partition number? 2
End?  [3886448kiB]? 4414832
(parted) print
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 60088320kiB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start      End         Size        Type     File system  Flags
 1      4096kiB    528384kiB   524288kiB   primary  fat32        lba
 2      528384kiB  4414832kiB  3886448kiB  primary

(parted) quit
```

### Truncating the modified image
Once this is done, use the following formula, in order to `determine` the `total file size`:
```no-highlight
<boot_partition_end_sector> + <root_partition_end_sector> = <total_file_size>
```

That is:
```no-highlight
528,384 KiB + 4,414,832 KiB = 4,943,216 KiB
```

Finally, `truncate` the file to `its new total size`:
```bash
$ truncate --size="$(( 528384 + 4414832 ))K" "raspberrypi_sd_card_backup.img"
$ ls -l --block-size="K"
total 4943224K
-rw-r--r-- 1 root root 4943216K Feb 16 22:08 raspberrypi_sd_card_backup.img
```

### Verifying the data integrity
**Before copying the modified image to the SD card, checking the `data integrity` is mandatory!**

Therefore, the `root partition` needs to be `mounted again`:
```bash
$ parted "raspberrypi_sd_card_backup.img" "unit s print"
Model:  (file)
Disk /root/tmp/raspberrypi_sd_card_backup.img: 120176640s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start     End         Size        Type     File system  Flags
 1      8192s     1056767s    1048576s    primary  fat32        lba
 2      1056768s  120176639s  119119872s  primary
$ losetup --offset="$(( 512 * 1056768 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcardbackup
Enter passphrase for /dev/loop2: raspberry
```

After that, `verify the filesystem integrity` via `e2fsck`:
```bash
$ e2fsck -f "/dev/mapper/cryptsdcardbackup"
e2fsck 1.47.1 (20-May-2024)
Pass 1: Checking inodes, blocks, and sizes
Pass 2: Checking directory structure
Pass 3: Checking directory connectivity
Pass 4: Checking reference counts
Pass 5: Checking group summary information

       82247 inodes used (34.62%, out of 237568)
         238 non-contiguous files (0.3%)
          68 non-contiguous directories (0.1%)
             # of inodes with ind/dind/tind blocks: 0/0/0
             Extent depth histogram: 76229/51
      698832 blocks used (75.09%, out of 930652)
           0 bad blocks
           1 large file

       70052 regular files
        6126 directories
           8 character device files
           0 block device files
           0 fifos
         366 links
        6052 symbolic links (5951 fast symbolic links)
           0 sockets
------------
       82604 files
```

The parameter `-f` `forces` the filesystem check, even, if it is clean.

If this check `fails`, `one or more` of the above steps may have caused a `misalignment`; **rendering the `root partition` unusable!** Try to comprehend the [above steps](#shrinking-the-modified-image) from the beginning once again.

If the check was `successful`, the `root partition` should be `mountable`:
```bash
$ mount "/dev/mapper/cryptsdcardbackup" "/mnt/"
$ df --block-size="K" "/mnt/"
Filesystem                    1K-blocks     Used Available Use% Mounted on
/dev/mapper/cryptsdcardbackup  3368008K 2440728K   724776K  78% /mnt
$ ls -l "/mnt/"
total 76
lrwxrwxrwx  1 root root     7 Nov 19 14:30 bin -> usr/bin
drwxr-xr-x  3 root root  4096 Feb 16 18:55 boot
drwxr-xr-x  4 root root  4096 Feb 16 18:55 dev
drwxr-xr-x 93 root root  4096 Feb 16 18:55 etc
drwxr-xr-x  3 root root  4096 Feb 16 18:55 home
[...]
```

For good measure, check the status of the `root partition` via `cryptsetup`:
```bash
$ cryptsetup status cryptsdcardbackup
/dev/mapper/cryptsdcardbackup is active and is in use.
  type:    LUKS2
  cipher:  xchacha20,aes-adiantum-plain64
  keysize: 256 bits
  key location: keyring
  device:  /dev/loop2
  loop:    /root/tmp/raspberrypi_sd_card_backup.img
  sector size:  4096
  offset:  32768 sectors
  size:    8796896 sectors
  mode:    read/write
```

Be aware, that the `size` is in `512-Byte-sectors`, even, if the `sector size` is indicated as `4096 Bytes`. ["This is a relict from the time, when only `512-byte-sectors` were supported"](https://gitlab.com/cryptsetup/cryptsetup/-/issues/884#note_1899199290).

After that, `close` the `LUKS device` and `detach` the `loop device`:
```bash
$ cryptsetup close cryptsdcardbackup
$ losetup --detach "/dev/loop2"
$ losetup --list
```

### Copying the modified image to the SD card
Finally, copy the image to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="1M" conv="fsync" status="progress"
```

When booting into `Raspberry Pi OS (Debian)`, the `root partition` can be [`extended on-the-fly`](#extending-the-root-partition).

# Debugging
## Examining the initramfs
There is a tool, called `lsinitramfs`, which can output the content of a compressed `initramfs` to `stdout`:
```bash
$ lsinitramfs "/boot/firmware/initramfs8" | less
```

This is useful, if one wants to check a `recent-built initramfs` in a quick way.

## Unarchiving the initramfs
The `initramfs` can be unarchived on the system, in order to analyse its content.

### Easy method
The following command `unarchives` the `initramfs` directly to the `current working directory`:
```bash
$ unmkinitramfs -v "/boot/firmware/initramfs8" "."
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
conf/modules
cryptroot
cryptroot/crypttab
etc
etc/console-setup
[...]
```

It is also possible to directly unarchive it to a custom directory:
```bash
$ unmkinitramfs -v "/boot/initramfs8" "initramfs/"
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
conf/modules
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
Copy the initramfs `/boot/firmware/initramfs8` to the `current working directory`:
```bash
$ cp --archive "/boot/firmware/initramfs8" "."
```

Then, analyse which type of compression was used:
```bash
$ file "initramfs8"
initramfs8: Zstandard compressed data (v0.8+), Dictionary ID: None
```

To unarchive it, `zstd` is required and the file must have the suffix `.zst`:
```bash
$ apt update
$ apt install zstd
$ mv --verbose initramfs8{,.cpio.zst}
renamed 'initramfs8' -> 'initramfs8.cpio.zst'
$ zstd --decompress "initramfs8"
```

Once this is done, the file is still compressed as `ASCII cpio archive`, which can be unarchived to the `current working directory` like so:
```bash
$ file "initramfs8.cpio"
initramfs.cpio: ASCII cpio archive (SVR4 with no CRC)
$ apt install cpio
$ cpio --extract --make-directories --preserve-modification-time --verbose < "initramfs8.cpio"
.
bin
conf
conf/arch.conf
conf/conf.d
conf/initramfs.conf
conf/modules
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
Disk /root/tmp/raspberrypi_sd_card_backup.img: 120176640s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start     End         Size        Type     File system  Flags
 1      8192s     1056767s    1048576s    primary  fat32        lba
 2      1056768s  120176639s  119119872s  primary  ext4
$ losetup --offset="$(( 512 * 1056768 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
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
When using the `modified` or a `self-prepared` image on `several Raspberry Pis`, all `UUIDs` are identical. There might be the need to change these.

Before making any changes, create a `backup` of the SD card, since the following commands `can corrupt data`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_changing_uuid.img" bs="1M" conv="fsync" status="progress"
```

### Installing necessary tools
Once this is done, boot into `Raspberry Pi OS (Debian)` and install the package `uuid`:
```bash
$ apt update
$ apt install uuid
```

### Changing the UUID
Next, generate a `new random UUID` via the command `uuid` and `modify` the `root partition` via `cryptsetup`:
```bash
$ uuid -v 4
8be664e3-e89d-4c23-adda-657eb936b1e5
$ cryptsetup --uuid="8be664e3-e89d-4c23-adda-657eb936b1e5" luksUUID "/dev/mmcblk0p2"

WARNING!
========
Do you really want to change UUID of device?

Are you sure? (Type 'yes' in capital letters): YES
```

### Verifying the modification
Verify, if the `new UUID` has been applied successfully:
```bash
$ blkid "/dev/mmcblk0p2"
/dev/mmcblk0p2: UUID="8be664e3-e89d-4c23-adda-657eb936b1e5" TYPE="crypto_LUKS" PARTUUID="e8af6eb2-02"
```

### Adapting configuration files
After that, adapt the entries in `/boot/firmware/cmdline.txt` and `/etc/crypttab`
```bash
$ vi "/boot/firmware/cmdline.txt"
root=/dev/mapper/cryptroot cryptdevice=UUID=c8be664e3-e89d-4c23-adda-657eb936b1e5:cryptroot
$ vi "/etc/crypttab"
cryptroot UUID=8be664e3-e89d-4c23-adda-657eb936b1e5 none luks,initramfs
```

### Rebuilding the initramfs
Rebuild the `initramfs`:
```bash
$ update-initramfs -vuk 6.6.74+rpt-rpi-v8
```

Make sure, that the following important files are present in the `initramfs` file `/boot/firmware/initramfs8`:
```bash
$ lsinitramfs "/boot/firmware/initramfs8" | grep --extended-regexp "adiantum|aes-arm|algif_skcipher|dm-crypt|nhpoly|sha2|chacha|crypttab|sbin/cryptsetup|usr/bin/cryptroot-unlock"
cryptroot/crypttab
usr/bin/cryptroot-unlock
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/aes-arm64.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/chacha-neon.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha2-ce.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/arch/arm64/crypto/sha256-arm64.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/adiantum.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/algif_skcipher.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/chacha_generic.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/crypto/nhpoly1305.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/drivers/md/dm-crypt.ko
usr/lib/modules/6.6.74+rpt-rpi-v8/kernel/lib/crypto/libchacha.ko
usr/sbin/cryptsetup
usr/bin/sha256sum
```

Also make sure, that the content of `/cryptroot/crypttab` is correct.

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
Disk /root/tmp/raspberrypi_sd_card_backup.img: 120176640s
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start     End         Size        Type     File system  Flags
 1      8192s     1056767s    1048576s    primary  fat32        lba
 2      1056768s  120176639s  119119872s  primary  ext4
$ losetup --offset="$(( 512 * 1056768 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcardbackup
Enter passphrase for /dev/loop2: raspberry
```

After that, mount it like so:
```bash
$ mount "/dev/mapper/cryptsdcardbackup" "/mnt/"
```

## Re-encrypting the root partition
When using the `modified` or a `self-prepared` image on `several Raspberry Pis`, all `key slot hashes and salts` are identical. **It is mandatory to change these!**

This method can also be used to apply a `new cipher method` to the `root partition`.

**This can only be done, if the `partition is unmounted`; it is not possbile to do this `on-the-fly`!**

Further configuration is needed for this.

### Prerequisites
* `LUKS` partition `version 2` (`cryptsetup luksDump "/dev/mmcblk0p2" | grep "Version"`)
* Raspberry Pi 4
* Bootable `USB stick` with `Raspberry Pi OS (Debian)`, which is accessable via `SSH`
    * `cryptsetup-2.0.6` or higher

If the `LUKS` partition version is `1`, please upgrade it to version `2` first, using [these instructions](https://gist.github.com/kravietz/d7ea4d98c5ffb79fc7a1b3d98be4de94/a306dba941cd6a6c56c65a08df879fa3033608ba#upgrade-luks).

If one does not use a `Raspberry Pi 4` with an on-board `EEPROM`, on which the `bootloader` is installed, but a separate Linux system, please `proceed with` [Creating a backup of the SD card](#creating-a-backup-of-the-sd-card-1) and then `skip to` [Re-encrypting the partition](#re-encrypting-the-partition).

### Creating a backup of the SD card
Before making any changes, create a `backup` of the SD card, since the following commands `can corrupt data`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_reencrypt.img" bs="1M" conv="fsync" status="progress"
```

### Configuring the bootloader
Boot into `Raspberry Pi OS (Debian)` of the Raspberry Pi, where the `root partition` should be re-encrypted and change the `boot order` of the `bootloader`:
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
`Re-encrypt` the `root-partition` via `cryptsetup reencrypt`:
```bash
$ cryptsetup reencrypt --cipher="xchacha20,aes-adiantum-plain64" --key-size="256" "/dev/mmcblk0p2"
Enter passphrase for key slot 0: raspberry
Progress:   5.0%, ETA 19:38,  744 MiB written, speed  12.0 MiB/s
```

Note, older versions of `cryptsetup` use `cryptsetup-reencrypt`.

This process may take up to `30 minutes`.

### Reverting the modifications of the bootloader
After that, `revert the boot order` of the bootloader and `reboot` to apply all changes:
```bash
$ rpi-eeprom-config --edit
BOOT_ORDER=0xf41
$ reboot
```

### Verifying the new cipher method
Finally, `verify`, if the re-encryption was successful:
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
  size:    119087104 sectors
  mode:    read/write
```

Be aware, that the `size` is in `512-Byte-sectors`, even, if `sector size` is indicated as `4096 Bytes`. ["This is a relict from the time, when only `512-byte-sectors` were supported"](https://gitlab.com/cryptsetup/cryptsetup/-/issues/884#note_1899199290).

The `USB stick` can now be disconnected.

# Known issues
## Compatibility
**For some reason, following the instructions [Encrypting the root partition manually](#encrypting-the-root-partition-manually) will provide an image file, which is **only compatible** with the Raspberry Pi revision, on which the `root partition` was resized.**

So, using an image, where the `root partition` was previously resized on a `Raspberry Pi Model B Rev 2` is **incompatible** with a `Raspberry Pi 4 Model B Rev 1.4`.

It will boot into the `initramfs`, but the `root partition` cannot be decrypted. Skipping [Resizing the root partition](#resizing-the-root-partition) will also preserve the partition size and its `UUID`, but it still will not work.
