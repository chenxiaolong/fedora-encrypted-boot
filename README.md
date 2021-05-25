# Fedora Encrypted /boot

This repo provides a simple patch to the Anaconda installer to allow Fedora 34+ systems to be installed with LUKS1-encrypted `/boot` partitions.

## Usage

1. Boot into the Fedora live image. Make sure the Anaconda installer is closed if it is open.

2. Ensure the `patch` command is installed.

    ```bash
    sudo dnf install patch
    ```

3. Clone this git repository.

    ```bash
    git clone https://github.com/chenxiaolong/fedora-encrypted-boot.git
    cd fedora-encrypted-boot
    ```

4. Apply the Anaconda patch.

    ```bash
    source /etc/os-release
    sudo patch -p2 \
        -i "$(pwd)/patches/${VERSION_ID}/0001-Add-support-for-encrypted-boot.patch" \
        -d /usr/lib64/python3.*/site-packages/pyanaconda
    ```

5. Open the Anaconda installer and install Fedora as usual.

## Details

The patch makes the following changes to the Anaconda installer:

* Removes the checks that prevent `/boot` from being located on an encrypted volume.
* Adds an additional check to ensure that `/boot` cannot be located on LUKS2-encrypted volumes.
* On UEFI systems, adds the appropriate `cryptomount -u <UUID>` line to `/boot/efi/EFI/fedora/grub.cfg` so that GRUB can decrypt LUKS and find the boot volume.
* Adds `GRUB_ENABLE_CRYPTODISK=y` to `/etc/default/grub` so that the relevant cryptodisk modules are included in the GRUB image. This has no effect on UEFI Secure Boot systems since those use a prebuilt signed build of `grubx64.efi`.

## Limitations

The patch does not change the suggested partition layouts that Anaconda offers. For example, the default btrfs partition layout still creates a separate ext4-formatted `/boot` partition. This default layout can be freely customized as desired or a new partition layout can be created from scratch.

Also, the patch adds a check to make sure `/boot` is on a LUKS1-formatted volume instead of LUKS2 (if encrypted). This is because GRUB2's support for LUKS2 is not quite ready yet. On UEFI (with Secure Boot), Fedora's prebuilt `grubx64.efi` executable does not include the required modules. On other systems, `grub2-install` currently does not detect the correct modules to include either. Additionally, the default KDF of LUKS2 (Argon2i) is not supported.

## Tested Scenarios

While it's not very likely for the patch to break things, it has only been tested in the following scenarios:

* Fedora 34 on UEFI with encrypted boot on btrfs
    * /dev/sda1
        * FAT32 -> /boot/efi
    * /dev/sda2
        * LUKS1
            * btrfs
                * root -> /
                * boot -> /boot
* Fedora 34 on UEFI with single encrypted ext4 filesystem
    * /dev/sda1
        * FAT32 -> /boot/efi
    * /dev/sda2
        * LUKS1
            * ext4 -> /
* Fedora 34 on UEFI without encryption
    * /dev/sda1
        * FAT32 -> /boot/efi
    * /dev/sda2
        * ext4 -> /
* Fedora 34 on BIOS with encrypted boot on btrfs
    * /dev/sda1
        * LUKS1
            * btrfs
                * root -> /
                * boot -> /boot

## LICENSE

The patch is under the same license as the upstream Anaconda project: GPLv2+. Please see [`LICENSE`](./LICENSE) for the full license text.
