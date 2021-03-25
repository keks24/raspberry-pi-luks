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
* [Encrypting the root partition manually](#encrypting-the-root-partition-manually)
   * [Prerequisites](#prerequisites-1)
   * [Downloading the stock image](#downloading-the-stock-image)
   * [Configuration](#configuration)
      * [Preparation](#preparation)
      * [Encrypting the root partition](#encrypting-the-root-partition)
      * [Entering the chroot](#entering-the-chroot)
         * [Installing necessary tools](#installing-necessary-tools)
         * [Configuration](#configuration-1)
         * [Generating the initramfs](#generating-the-initramfs)
         * [Exiting the chroot](#exiting-the-chroot)
* [Installing the modified image](#installing-the-modified-image)
* [Further steps](#further-steps)
   * [Updating all installed packages](#updating-all-installed-packages)
* [Optional steps](#optional-steps)
   * [Decrypting the root partition via SSH](#decrypting-the-root-partition-via-ssh)
      * [Install dropbear-initramfs](#install-dropbear-initramfs)
      * [Configure dropbear-initramfs](#configure-dropbear-initramfs)
      * [Configuring kernel parameters](#configuring-kernel-parameters)
      * [Rebuilding the initramfs](#rebuilding-the-initramfs)
      * [Rebooting](#rebooting)
      * [Testing remote decryption](#testing-remote-decryption)
      * [Optional fancy SSH ASCII banner](#optional-fancy-ssh-ascii-banner)
         * [Configuring dropbear-initramfs](#configuring-dropbear-initramfs)
         * [Generating a fancy ASCII banner](#generating-a-fancy-ascii-banner)
         * [Rebuilding the initramfs](#rebuilding-the-initramfs-1)
         * [Rebooting](#rebooting-1)
* [Debugging](#debugging)
   * [Examining the initramfs](#examining-the-initramfs)
   * [Unarchiving the initramfs](#unarchiving-the-initramfs)
* [Additional information](#additional-information)
   * [Credentials](#credentials)
      * [LUKS password](#luks-password)
      * [User credentials](#user-credentials)
   * [Decrypting the root partition from the image](#decrypting-the-root-partition-from-the-image)
   * [Changing the LUKS password](#changing-the-luks-password)
   * [Changing the user password](#changing-the-user-password)

# Introduction
This repository shall describe all necessary steps in order to encrypt the `root partition` of the Raspberry Pi stock image `Raspberry Pi OS Lite`; currently `Debian 10 (Buster)` on a `Raspberry Pi Model B Rev 2`. The instructions should be adaptable for other Raspberry Pi revisions as well.

The entire setup was done on a `Banana Pi Pro` with [`Armbian Buster (mainline based kernel 5.10.y)`](https://www.armbian.com/banana-pi-pro/).

# Using the modified image
## Prerequisites
* The following packages are installed:
```no-highlight
aria2c
coreutils
cryptsetup
e2fsprogs
gnupg
parted
util-linux
```
* The capacity of the `SD card` must be greater than `8 GiB`.

## Downloading the image
Either download the files manually from the [release page](https://codeberg.org/keks24/raspberry-pi-luks/releases) or download them via `direct links`:
```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" --force-sequential \
    "https://srv-store4.gofile.io/download/48Rnkz/ee3a464731dc8453f7d9b214cdc445dc/raspberrypi_sd_card_backup.img" \
    "https://srv-store4.gofile.io/download/48Rnkz/c57f476a29378ae9ce21dff9e3c8120c/raspberrypi_sd_card_backup.img.asc" \
    "https://srv-store4.gofile.io/download/48Rnkz/2e907656ce9f9c30c31058a1d0e06091/raspberrypi_sd_card_backup.img.b2" \
    "https://srv-store4.gofile.io/download/48Rnkz/26dccd90de56da6cfdf4d00df63291e6/raspberrypi_sd_card_backup.img.sha256" \
    "https://srv-store1.gofile.io/download/48Rnkz/fa7c7877a25f02d9071eb712971917ad/LICENSE"
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

## Copying the image to the SD card
Copy the image to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" conv="fdatasync" status="progress"
```

## Resizing the root partition
When copying the image to another SD card with a `higher capacity`, the `encrypted root partition` will stay at `~8 GiB`. Therefore, it needs to be `extended` in order to use the `unused free space`.

### Creating a backup of the SD card
Before doing any changes, create a `backup` of the SD card, since the following commands can corrupt data:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup_before_resize.img" bs="512b" conv="fdatasync" status="progress"
```

### Analysing the root partition
After that, boot into `Raspbian` and check the partition structure via `parted`:
```bash
$ parted --list
Model: Linux device-mapper (crypt) (dm)
Disk /dev/mapper/cryptroot: 7659MB
Sector size (logical/physical): 512B/512B
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

The command `resizepart` will be used to `extend` the partition, where `-1` defines the very end of the SD card.

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
  cipher:  aes-xts-plain64
  keysize: 512 bits
  key location: keyring
  device:  /dev/mmcblk0p2
  sector size:  512
  offset:  32768 sectors
  size:    61766752 sectors
  mode:    read/write
```

To check, if the values are correct, the following formula can be used:
```no-highlight
(<sector_size> * <sector_size>) / 1024^3 = <size_in_gib>
```

That is:
```no-highlight
(512 Bytes * 61766752) / 1024^3 = 29.45 GiB
```

The result differs slightly from the output of `parted`, since the unit is in `Gibibyte (base 2)` and not `Gigabyte (base 10)`.

# Encrypting the root partition manually
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
* `qemu-arm-static` is needed, if one is working on a `non-ARM operating system`.
* Free space of at least `1.5 times` the capactiy of the `SD card`

## Downloading the stock image
Download the image `Raspberry Pi OS Lite` from the [official page](https://www.raspberrypi.org/software/operating-systems/) and also save its `SHA256` checksum:

```bash
$ aria2c --min-split-size="20M" --split="4" --max-connection-per-server="8" "https://downloads.raspberrypi.org/raspios_lite_armhf/images/raspios_lite_armhf-2021-01-12/2021-01-11-raspios-buster-armhf-lite.zip"
$ echo "d49d6fab1b8e533f7efc40416e98ec16019b9c034bc89c59b83d0921c2aefeef  2021-01-11-raspios-buster-armhf-lite.zip" > "2021-01-11-raspios-buster-armhf-lite.zip.sha256"
```

Verify the checksum of the archive:
```bash
$ sha256sum --check "2021-01-11-raspios-buster-armhf-lite.zip.sha256"
2021-01-11-raspios-buster-armhf-lite.zip: OK
```

## Configuration
### Preparation
Clone the repository to the current working directory:
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
$ dd if="2021-01-11-raspios-buster-armhf-lite.img" of="/dev/sdx" bs="512b" conv="fdatasync" status="progress"
```

Boot into `Raspbian`, so the `root partition` is extended to its full capacity. Then, log in with the [predefined user credentials](#user-credentials) and update the kernel:
```bash
$ sudo apt update
$ sudo apt install raspberrypi-kernel --only-upgrade
$ reboot
```

After logging in again, take notes of the following output. These information are important for later:
```bash
$ uname --release
5.10.17+
$ grep "Model" "/proc/cpuinfo"
Model           : Raspberry Pi Model B Rev 2
$ sudo poweroff
```

After that, create a `backup` of the `SD card`:
```bash
$ dd if="/dev/sdx" of="raspberrypi_sd_card_backup.img" bs="512b" conv="fdatasync" status="progress"
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

Other `encryption methods` are supported as well and can be looked up [here](https://gitlab.com/cryptsetup/cryptsetup/-/wikis/LUKS-standard/on-disk-format.pdf#Cipher%20and%20Hash%20specification%20registry).

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

### Entering the chroot
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

[Source](https://www.raspberrypi.org/documentation/linux/kernel/building.md).

Copy the `custom hook scripts`. **This step must be done in a `separate shell` outside of the `chroot` environment!**:
```bash
$ install -D --verbose --owner="root" --group="root" --mode="755" "raspberry-pi-luks/etc/kernel/postinst.d/5.10.17+/rpi-iniramfs-tools" "/mnt/etc/kernel/postinst.d/5.10.17+/rpi-initramfs-tools"
install: creating directory '/mnt/etc/kernel/postinst.d/5.10.17+'
'raspberry-pi-luks/etc/kernel/postinst.d/5.10.17+/rpi-iniramfs-tools' -> '/mnt/etc/kernel/postinst.d/5.10.17+/rpi-initramfs-tools'
$ install -D --verbose --owner="root" --group="root" --mode="755" "raspberry-pi-luks/etc/kernel/postrm.d/5.10.17+/rpi-iniramfs-tools" "/mnt/etc/kernel/postrm.d/5.10.17+/rpi-initramfs-tools"
install: creating directory '/mnt/etc/kernel/postrm.d/5.10.17+'
'raspberry-pi-luks/etc/kernel/postrm.d/5.10.17+/rpi-iniramfs-tools' -> '/mnt/etc/kernel/postrm.d/5.10.17+/rpi-initramfs-tools'
```

Be aware, that the directory `/etc/kernel/postinst.d/5.10.17+/` must always match the `kernel version`, which is currently in use. Otherwise, generating the `initramfs` will fail and renders the `Raspberry Pi` unbootable.

Next, get the `UUID` of `/dev/loop2`, which will be used later on:
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

Adapt the file `/etc/fstab`, so the `decrypted root partition` will be mounted automatically:
```bash
(chroot) $ vi "/etc/fstab"
#PARTUUID=e8af6eb2-02  /               ext4    defaults,noatime  0       1
/dev/mapper/cryptroot  /               ext4    defaults,noatime  0       1
```

#### Generating the initramfs
There are two ways to generate the `initramfs`.

1. Either reinstall the package `raspberrypi-kernel`, which will install all `Raspberry Pi kernels` and then executes the `hook scripts` in `/etc/kernel/postinst.d/`:
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

Detailed debugging is explained [below](#debugging).

#### Exiting the chroot
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
```

# Installing the modified image
The image is now prepared and can be copied to the `SD card`:
```bash
$ dd if="raspberrypi_sd_card_backup.img" of="/dev/sdx" bs="512b" conv="fdatasync" status="progress"
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

### Configure dropbear-initramfs
All configuration files can be found at `/etc/dropbear-initramfs/` and are self-explained.

Next, configure `dropbear-initramfs` by editing `/etc/dropbear-initramfs/config`:
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

`dropbear` does not seem to support `ed25519` keys, yet; so a strong `RSA 8192` key should be generated on the `host` from which the partition should be decrypted:
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

This will set the `private Class C IP address` to `192.168.1.80` and the `subnet mask` to `255.255.255.0`; these values may differ depending on the network infrastructure. The `initramfs` does not have any connection to the `internet`, since no `gateway IP address` is set.

Make sure, that the `Raspberry Pi` and the `host` from which the partition should be decrypted, are in the same network.

### Rebuilding the initramfs
`Rebuild` the `initramfs` to adapt all changes:
```bash
$ mkinitramfs -o "/boot/initrd.img"
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
Please unlock disk cryptroot:
```

After entering the correct password, the `SSH connection` will cut off and the `Raspberry Pi` will boot:
```no-highlight
cryptsetup: cryptroot set up successfully
Shared connection to 192.168.1.80 closed.
```

### Optional fancy SSH ASCII banner
`dropbear-initramfs` allows to set a custom `ASCII banner`, which is shown, when logging into the `initramfs`.

#### Configuring dropbear-initramfs
To do this, the parameter `-b` has to be appended to the configuration file `/etc/dropbear-initramfs/config`:
```bash
$ vi "/etc/dropbear-initramfs/config"
$ DROPBEAR_OPTIONS="-p 22222 -I 60 -sjk -b etc/dropbear/ssh_banner.net"
```

Leaving out the `leading slash` is important, since the `initramfs` does not have the directory `/`.

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
$ install -D --verbose --owner="root" --group="root" --mode="755" "raspberry-pi-luks/etc/initramfs-tools/hooks/dropbear" "/etc/initramfs-tools/hooks/dropbear"
'raspberry-pi-luks/etc/initramfs-tools/hooks/dropbear' -> '/etc/initramfs-tools/hooks/dropbear'
```

#### Rebuilding the initramfs
Finally, rebuild the `initramfs`:
```bash
$ mkinitramfs -o "/boot/initrd.img"
```

#### Rebooting
After rebooting the `Raspberry Pi`, `dropbear-initramfs` will now display a `fancy ASCII banner`:
```bash
$ ssh -i 22222 root@192.168.1.80 -i "/home/<some_username>/.ssh/dropbear_root_rsa8192"
       __                           __                           __                   __
  ____/ /__  ____________  ______  / /_   ____________  ______  / /__________  ____  / /_
 / __  / _ \/ ___/ ___/ / / / __ \/ __/  / ___/ ___/ / / / __ \/ __/ ___/ __ \/ __ \/ __/
/ /_/ /  __/ /__/ /  / /_/ / /_/ / /_   / /__/ /  / /_/ / /_/ / /_/ /  / /_/ / /_/ / /_
\__,_/\___/\___/_/   \__, / .___/\__/   \___/_/   \__, / .___/\__/_/   \____/\____/\__/
                    /____/_/                     /____/_/

Please unlock disk cryptroot:
```

# Debugging
## Examining the initramfs
There is a tool, called `lsinitramfs`, which can output the content of an compressed `initramfs` to `stdout`:
```bash
$ lsinitramfs "/boot/initrd.img" | less
```

This is useful, when one wants to check a recent-built `initramfs` in a quick way.

## Unarchiving the initramfs
The `initramfs` can be unarchived on the system in order to analyse its content.

First, copy the initramfs `initrd.img` to the current working directory:
```bash
$ cp --archive "/boot/initrd.img" .
```

Then, analyse which type of compression was used:
```bash
$ file "initrd.img"
/boot/initrd.img: gzip compressed data, last modified: Tue Mar 23 00:35:15 2021, from Unix, original size 24183808
```

To unarchive it, `gzip` is required and the suffix `.gz` needs to be appended as well:
```bash
$ apt update
$ apt install gzip
$ mv initrd.img{,.gz}
$ gzip --decompress "initrd.img.gz"
```

Once this is done, the file is still compressed as `ASCII cpio archive`, which can be unarchived like so:
```bash
$ file "initrd.img"
initrd.img: ASCII cpio archive (SVR4 with no CRC)
$ apt install cpio
$ cpio --extract --make-directories --preserve-modification-time --verbose < "initrd.img"
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

This unarchives the entire `initramfs` into the current working directory.

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

## Decrypting the root partition from the image
The encrypted `root partition` can be opened via `cryptsetup` as follows:
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
$ losetup --offset="$(( 512 * 532480 ))" "/dev/loop2" "raspberrypi_sd_card_backup.img"
$ cryptsetup open "/dev/loop2" cryptsdcard
Enter passphrase for /dev/loop2: raspberry
$ mount "/dev/mapper/cryptsdcard" "/mnt/"
```

## Changing the LUKS password
The password of the `root partition` can be changed like so:
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
$ losetup --offset="$(( 512 * 532480 ))" /dev/loop2 raspberrypi_sd_card_backup.img
$ cryptsetup luksChangeKey "/dev/loop2"
Enter passphrase to be changed: raspberry
Enter new passphrase: <some_strong_personal_password>
Verify passphrase: <some_strong_personal_password>
```

This is also possible on the `Raspberry Pi` itself:
```bash
$ cryptsetup luksChangeKey "/dev/mmcblk0p2"
Enter passphrase to be changed: raspberry
Enter new passphrase: <some_strong_personal_password>
Verify passphrase: <some_strong_personal_password>
```

## Changing the user password
When logged in as the user `pi`, change the password as follows:
```bash
$ passwd
Changing password for pi.
Current password: raspberry
New password: <some_strong_personal_password>
Retype new password: <some_strong_personal_password>
passwd: password updated successfully
```
