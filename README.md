Table of Contents
=================
* [Introduction](#introduction)
* [Using the modified image](#using-the-modified-image)
* [Encrypting the root partition manually](#encrypting-the-root-partition-manually)
   * [Prerequisites](#prerequisites)
   * [Downloading the stock image](#downloading-the-stock-image)
   * [Configuration](#configuration)
      * [Preparation](#preparation)
      * [Encrypting the root partition](#encrypting-the-root-partition)
      * [Entering the chroot](#entering-the-chroot)
         * [Installing necessary packages in order to build an initramfs](#installing-necessary-packages-in-order-to-build-an-initramfs)
         * [Configuration](#configuration-1)
         * [Generating the initramfs](#generating-the-initramfs)
         * [Exiting the chroot](#exiting-the-chroot)
* [Installing the modified image](#installing-the-modified-image)
* [Further steps](#further-steps)
   * [Updating all installed packages](#updating-all-installed-packages)
* [Optional steps](#optional-steps)
   * [Decrypting the root partition via SSH](#decrypting-the-root-partition-via-ssh)
* [Additional information](#additional-information)
   * [Opening the root partition from the image](#opening-the-root-partition-from-the-image)
   * [Changing the LUKS password](#changing-the-luks-password)

# Introduction
This repository shall describe all necessary steps in order to encrypt the `root partition` of the Raspberry Pi stock image `Raspberry Pi OS Lite`; currently `Debian 10 (Buster)` on a `Raspberry Pi Model B Rev 2`. The instructions should be adaptable for other Raspberry Pi revisions as well.

The entire setup was done on a `Banana Pi Pro` with [`Armbian Buster (mainline based kernel 5.10.y)`](https://www.armbian.com/banana-pi-pro/).

# Using the modified image
The `SD card` must be bigger than `8 GiB`.

Either download the files manually from the [release page](https://codeberg.org/keks24/raspberry-pi-luks/releases) or download them via `direct links`:
```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" --force-sequential \
    "https://srv-store4.gofile.io/download/48Rnkz/ee3a464731dc8453f7d9b214cdc445dc/raspberrypi_sd_card_backup.img" \
    "https://srv-store4.gofile.io/download/48Rnkz/c57f476a29378ae9ce21dff9e3c8120c/raspberrypi_sd_card_backup.img.asc" \
    "https://srv-store4.gofile.io/download/48Rnkz/2e907656ce9f9c30c31058a1d0e06091/raspberrypi_sd_card_backup.img.b2" \
    "https://srv-store4.gofile.io/download/48Rnkz/26dccd90de56da6cfdf4d00df63291e6/raspberrypi_sd_card_backup.img.sha256" \
    "https://srv-store6.gofile.io/download/48Rnkz/588be43ada286493bab2fd309bc9eb99/LICENSE" \
    "https://srv-store6.gofile.io/download/48Rnkz/4a2f7d2a2895bbf8ab72933afc9de0e6/README.md"
```

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

Copy the image to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" status="progress"
```

# Encrypting the `root partition` manually
## Prerequisites
* The following packages are installed:
```no-highlight
aria2c
busybox
coreutils
cryptsetup
e2fsprogs
qemu-arm-static
unzip
util-linux
```
* Free space of at least `1.5 times` the capactiy of the `SD card`
* `qemu-arm-static` is needed, if one is working on a `non-ARM operating system`.

## Downloading the stock image
Download the image `Raspberry Pi OS Lite` from the [official page](https://www.raspberrypi.org/software/operating-systems/) and also save its `SHA256` checksum:

```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" "https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip.torrent"
$ echo "d49d6fab1b8e533f7efc40416e98ec16019b9c034bc89c59b83d0921c2aefeef" > "2021-01-11-raspios-buster-armhf-lite.zip.sha256"
```

Verify the checksum of the archive:
```bash
$ sha256sum --check "2021-01-11-raspios-buster-armhf-lite.zip.sha256"
2021-01-11-raspios-buster-armhf-lite.zip: OK
```

## Configuration
### Preparation
Clone the repository in the current working directory:
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
$ dd if="2021-01-11-raspios-buster-armhf-lite.img" of="/dev/sdx" bs="512b" status="progress"
$ sync
```

Boot into `Raspbian` once, so the `root partition` is extended to its full capacity. Then, log in with the username `pi` and the password `raspberry`.

Take notes of the following commands and shutdown the system. These are important for later:
```bash
$ uname --release
5.10.17+
$ grep "Model" "/proc/cpuinfo"
Model           : Raspberry Pi Model B Rev 2
$ sudo poweroff
```

After that, create a `backup` of the `SD card`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup.img" bs="512b" status="progress" && sync
```

Analyse the image for its partition `startsectors` and the `logical sector size`:
```bash
$ fdisk --list "raspberrypi_sd_card_backup.img"
Disk raspberry_pi_sd_card_backup.img: 7.41 GiB, 7948206080 bytes, 15523840 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x2cf0beee

Device                           Boot  Start      End  Sectors  Size Id Type
raspberry_pi_sd_card_backup.img1        8192   532479   524288  256M  c W95 FAT32 (LBA)
raspberry_pi_sd_card_backup.img2      532480 15523839 14991360  7.2G 83 Linux
```

In this case, the `first partition (boot)` starts at `startsector 8192 bytes` and the `second partition (root)` at `startsector 532480 bytes`. The `logical sector size` is `512 bytes`.

These values can be used to mount the partitions from the image.

Before doing that, check, that there are free `loop devices` at `/dev/`:
```bash
$ losetup
```

Next, make the partitions available at `/dev/loop1` and `/dev/loop2`:
```bash
$ losetup --offset="$(( 512 * 8192 ))" "/dev/loop1" "raspberrypi_sd_card_backup.img"
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ losetup
NAME       SIZELIMIT    OFFSET AUTOCLEAR RO BACK-FILE                                DIO LOG-SEC
/dev/loop1         0   4194304         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
/dev/loop2         0 272629760         0  0 /root/tmp/raspberrypi_sd_card_backup.img   0     512
```

Using `losetup` here is important, since it is necessary for the `chroot` later on. Using `mount --offset` might return the error `overlapping loop device exists`.

Mount and create a `backup` of the `root partition` before encrypting it. The `trailing slash` for the `rsync` command is important here!:
```bash
$ mount "/dev/loop2" "/mnt/"
$ rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" "/mnt/" "root_backup"
```

After that unmount `/mnt/`:
```bash
$ umount "/mnt/"
```

### Encrypting the `root partition`
Since the preparation is done, the `root partition` can now be `formatted and encrypted` via `cryptsetup`:
```bash
$ cryptsetup --cipher="aes-xts-plain64" --key-size="512" luksFormat "/dev/loop2"
WARNING: Device /dev/loop2 already contains a 'ext4' superblock signature.

WARNING!
========
This will overwrite data on /dev/loop2 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/loop2: raspberry
Verify passphrase: raspberry
```

Make absolutely sure, that the `key size` is `at least 512 Bytes`, [since `XTS` splits the key size in half](https://wiki.archlinux.org/index.php/dm-crypt/Device_encryption#Encryption_options_for_LUKS_mode).

The `LUKS header information` looks like so:
```bash
$ cryptsetup luksDump "/dev/loop2"
LUKS header information
Version:        2
Epoch:          3
Metadata area:  16384 [bytes]
Keyslots area:  16744448 [bytes]
UUID:           1fd31646-340c-47ed-8c66-8efb2e730d0f
Label:          (no label)
Subsystem:      (no subsystem)
Flags:          (no flags)

Data segments:
  0: crypt
        offset: 16777216 [bytes]
        length: (whole device)
        cipher: aes-xts-plain64
        sector: 512 [bytes]

Keyslots:
  0: luks2
        Key:        512 bits
        Priority:   normal
        Cipher:     aes-xts-plain64
        Cipher key: 512 bits
        PBKDF:      argon2i
        Time cost:  4
        Memory:     45128
        Threads:    2
        Salt:       45 b9 b2 2f 2d 8a 8d e2 26 63 fa 2d 2f cd d0 1d
                    05 de 0a da 84 1e c2 93 ad 86 b7 d0 ec 13 69 4e
        AF stripes: 4000
        AF hash:    sha256
        Area offset:32768 [bytes]
        Area length:258048 [bytes]
        Digest ID:  0
Tokens:
Digests:
  0: pbkdf2
        Hash:       sha256
        Iterations: 7816
        Salt:       bc fd f1 88 bf 9f 4c 24 d9 98 c2 49 cc 2d 20 7b
                    05 d5 2f c4 5b c3 b6 de a9 2a a9 d6 73 bc cc 88
        Digest:     22 8e 63 90 ec 64 9d 0a b8 62 50 06 80 ca e0 90
                    cb 5b 14 ae 38 87 7a f0 0a 21 1b a7 8a a0 28 86
```

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

Next, restore the backup:
```bash
$ rsync --archive --hard-links --acls --xattrs --one-file-system --numeric-ids --info="progress2" "root_backup/" "/mnt/"
```

Finally, mount the `boot partition` to `/mnt/boot/`:
```bash
$ mount "/dev/loop1" "/mnt/boot/"
```

### Entering the `chroot`
Once this is done, it is time to go into a `chroot` environment.

Prepare the `chroot` environment:
```bash
$ mount --types proc "/proc/" "/mnt/proc/"
$ mount --rbind "/sys" "/mnt/sys/"
$ mount --make-rslave "/mnt/sys/"
$ mount --rbind "/dev/" "/mnt/dev/"
$ mount --make-rslave "/dev/"
$ cp "/usr/bin/qemu-arm-static" "/mnt/usr/bin/"
```

**`qemu-arm-static` is mandatory, if one is working on a `Non-ARM operating system`!**

Enter the new environment:
```bash
$ chroot "/mnt/" qemu-arm-static "/bin/bash"
$ source "/etc/profile"
$ export PS1="(chroot) ${PS1}"
(chroot) $ cd
```

#### Installing necessary packages in order to build an initramfs
An `initramfs` is needed in order to decrypt the `root partition`. The following packages will provide all tools to build it:
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

As of writing the package `rpi-initramfs-tools` is not available, yet. So `custom hook scripts` have to be used for this.

The next commands contain the kernel version `5.10.17+`. Replace the version according to the `Raspberry Pi revision` (`grep "Model" "/proc/cpuinfo"`) and the current kernel version (`uname --release`):

Type                | Kernel version naming convention   | Kernel filename   | Initramfs filename
------------------- | ---------------------------------- | ----------------- |  -----------------
Raspberry Pi 1      | `<kernel_version>+`                  | `kernel.img`    | `initrd.img-<kernel_version>+`
Raspberry Pi Zero   | `<kernel_version>+`                  | `kernel.img`    | `initrd.img-<kernel_version>+`
Raspberry Pi Zero W | `<kernel_version>+`                  | `kernel.img`    | `initrd.img-<kernel_version>+`
Raspberry Pi 2      | `<kernel_version>-v7`                | `kernel7.img`   | `initrd.img-<kernel_version>-v7`
Raspberry Pi 3      | `<kernel_version>-v7`                | `kernel7.img`   | `initrd.img-<kernel_version>-v7`
Raspberry Pi 3+     | `<kernel_version>-v7`                | `kernel7.img`   | `initrd.img-<kernel_version>-v7`
Raspberry Pi 4      | `<kernel_version>-v7l`               | `kernel7l.img`  | `initrd.img-<kernel_version>-v7l`

Copy the `custom hook scripts`. **This step must be done in a `separate shell` outside of the `chroot` environment!**:
```bash
$ install -D --verbose --owner="root" --group="root" --mode="755" "raspberry-pi-luks/etc/kernel/postinst.d/5.10.17+/rpi-iniramfs-tools" "/mnt/etc/kernel/postinst.d/5.10.17+/rpi-initramfs-tools"
install: creating directory '/mnt/etc/kernel/postinst.d/5.10.17+'
'raspberry-pi-luks/etc/kernel/postinst.d/5.10.17+/rpi-iniramfs-tools' -> '/mnt/etc/kernel/postinst.d/5.10.17+/rpi-initramfs-tools'
$ install -D --verbose --owner="root" --group="root" --mode="755" "raspberry-pi-luks/etc/kernel/postrm.d/5.10.17+/rpi-iniramfs-tools" "/mnt/etc/kernel/postrm.d/5.10.17+/rpi-initramfs-tools"
install: creating directory '/mnt/etc/kernel/postrm.d/5.10.17+'
'raspberry-pi-luks/etc/kernel/postrm.d/5.10.17+/rpi-iniramfs-tools' -> '/mnt/etc/kernel/postrm.d/5.10.17+/rpi-initramfs-tools'
```

Be aware, that the directory `/etc/kernel/postinst.d/5.10.17+/` must always match the `kernel version`, which is currently in use. Otherwise, generating the `initramfs` will fail.

Get the `UUID` of `/dev/loop2`, which will be used later on:
```bash
$ blkid "/dev/loop2"
/dev/loop2: UUID="1fd31646-340c-47ed-8c66-8efb2e730d0f" TYPE="crypto_LUKS"
```

Edit the kernel parameters in `/boot/cmdline.txt` and add an entry in `/etc/crypttab`, so the `root partition` can be decrypted:
```bash
(chroot) $ vi "/boot/cmdline.txt"
root=/dev/mapper/cryptroot cryptdevice=UUID=1fd31646-340c-47ed-8c66-8efb2e730d0f:cryptroot
(chroot) $ vi "/etc/crypttab"
cryptroot UUID=1fd31646-340c-47ed-8c66-8efb2e730d0f none luks
```

Adapt the file `/etc/fstab`, so the `root partition` will be mounted automatically:
```bash
(chroot) $ vi "/etc/fstab"
#PARTUUID=e8af6eb2-02  /               ext4    defaults,noatime  0       1
/dev/mapper/cryptroot  /               ext4    defaults,noatime  0       1
```

#### Generating the initramfs
There are two ways to generate the `initramfs`.

1. Either reinstall the package `raspberrypi-kernel`, which will install all kernels and then execute the `hook scripts` in `/etc/kernel/postinst.d/`:
```bash
(chroot) $ apt install raspberrypi-kernel --reinstall
```

2. Or execute the command `mkinitramfs` manually:
```bash
(chroot) $ mkinitramfs -o "/boot/initrd.img" 5.10.17+
```

For the initial setup, the first method is preferred.

Make sure, that the binary `cryptsetup` is present in the file `initrd.img`:
```bash
(chroot) $ lsinitramfs "/boot/initrd.img" | grep "sbin/cryptsetup"
usr/sbin/cryptsetup
```

#### Exiting the `chroot`
Exit the `chroot`, unmount the `boot partition` and all `pseudo filesystems`:
```bash
(chroot) $ exit
$ cd
$ umount "/mnt/boot/"
$ umount --lazy --recursive "/mnt/proc" "/mnt/sys/" "/mnt/dev/"
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
```

# Installing the modified image
The image is now prepared and can be copied to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" status="progress" && sync
```

On boot there should be a message to decrypt the `root partition`:
```no-highlight
Please unlock disk cryptroot: raspberry
```

After entering the password, the system should start.

# Further steps
## Updating all installed packages
```bash
$ apt update
$ apt upgrade
```

# Optional steps
## Decrypting the `root partition` via `SSH`
bookmark
bookmark
bookmark
bookmark
bookmark
bookmark
bookmark
dropbear

# Additional information
## Opening the root partition from the image
The encrypted `root partition` can be opened via `cryptsetup` as follows:
```bash
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcard
Enter passphrase for /dev/loop2: raspberry
$ mount "/dev/mapper/cryptsdcard" "/mnt/"
```

## Changing the `LUKS` password
Changing the password within the image:
```bash
$ losetup --offset="$(( 512 * 532480 ))" /dev/loop2 raspberrypi_sd_card_backup.img
$ cryptsetup luksChangeKey "/dev/loop2"
Enter passphrase to be changed: raspberry
Enter new passphrase: 1234
Verify passphrase: 1234
```

Changing the password on the `Raspberry Pi`:
```bash
$ cryptsetup luksChangeKey "/dev/mmcblk0p2"
Enter passphrase to be changed: raspberry
Enter new passphrase: 1234
Verify passphrase: 1234
```
