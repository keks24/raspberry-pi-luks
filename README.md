Table of Contents
=================
* [Introduction](#introduction)
* [Known issues](#known-issues)

# Introduction
This repository shall describe all necessary steps in order to encrypt the `root partition` of the Raspberry Pi stock image `Raspberry Pi OS Lite`.

Refer to the different branches for your target distribution:
* `[Debian 10 (Buster)](https://codeberg.org/keks24/raspberry-pi-luks/src/branch/debian_10_buster)` on a `Raspberry Pi Model B Rev 2`.
* [Debian 11 (Bullseye)(https://about_blank) on a `<when_the_time_is_ripe>`.

The instructions are adaptable for `other Raspberry Pi revisions` as well. They should also work on `image files with partition information` in general.

**Please read [Known issues](#known-issues) first before following any of the instructions!**

# Known issues
**Note: For some reason, following the instructions [Encrypting the root partition manually](#encrypting-the-root-partition-manually) will provide an image file, which is **only compatible** with the Raspberry Pi revision, on which the `root partition` was resized.**

So, using an image, where the `root partition` was previously resized on a `Raspberry Pi Model B Rev 2` is **incompatible** with a `Raspberry Pi 4 Model B Rev 1.4`.

It will boot into the `initramfs`, but the `root partition` cannot be decrypted. Skipping [Resizing the root partition](#resizing-the-root-partition) will also preserve the partition size and its `UUID`, but it still will not work.
