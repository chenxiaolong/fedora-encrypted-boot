**NOTE**: This is no longer maintained because I've switched to using UKIs + systemd-boot.

# Fedora Encrypted /boot

This repo provides a simple patch to the Anaconda installer to allow Fedora 34+ systems to be installed with LUKS1-encrypted `/boot` partitions.

## Usage

1. Boot into the Fedora live image. Make sure the Anaconda installer is closed if it is open.

2. Ensure `git` and `patch` are installed.

    ```bash
    sudo dnf install git patch
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

### Default partition layout

The patch does not change the suggested partition layouts that Anaconda offers. For example, the default btrfs partition layout still creates a separate ext4-formatted `/boot` partition. This default layout can be freely customized as desired or a new partition layout can be created from scratch.

### LUKS2-encrypted `/boot` not allowed

The patch adds a check to make sure `/boot` is on a LUKS1-formatted volume instead of LUKS2 (if encrypted). This is because GRUB2's support for LUKS2 is not quite ready yet. On UEFI (with Secure Boot), Fedora's prebuilt `grubx64.efi` executable does not include the required modules. On other systems, `grub2-install` currently does not detect the correct modules to include either. Additionally, the default KDF of LUKS2 (Argon2i) is not supported.

### LUKS decryption in GRUB2 is slow

GRUB's implementation of LUKS is not as fast as the implementation in the Linux kernel. As a very rough estimate, with a Kaby Lake Core i7 7700HQ, a LUKS key slot using ~2 million PBKDF2 iterations can be decrypted by the kernel in a couple seconds while it takes GRUB more than 30 seconds.

The current number of PBKDF2 iterations can be queried by running:

```bash
sudo cryptsetup luksDump <LUKS block device>
```

The number of iterations can be reduced to speed up decryption in GRUB, but should only be done if the passphrase has sufficient entropy. Before making any changes, see cryptsetup's documentation about this topic: https://gitlab.com/cryptsetup/cryptsetup/-/wikis/FrequentlyAskedQuestions#3-common-problems (section 3.4).

If the tradeoffs of reducing the PBKDF2 iterations are acceptable, create a new key slot with the desired number of iterations. The new passphrase can be the same as the existing passphrase.

```bash
sudo cryptsetup luksAddKey --pbkdf-force-iterations <iterations> <LUKS block device>
```

Then, disable the old key slot (default slot is 0).

```bash
sudo cryptsetup luksKillSlot <LUKS block device> 0
```

### Passphrase needs to be entered twice

With an encrypted `/boot` volume, the LUKS passphrase needs to be entered twice during the boot process: once in GRUB and once in plymouth (Linux). There is currently no way for GRUB to securely pass the passphrase to Linux.

This can be annoying and can be worked around by including a key file in the initramfs. **While the key file is encrypted at rest, any process with root privileges will be able to read it.** This is far less secure than having a passphrase that only exists in (human) memory.

To add a key to the initramfs anyway:

1. Generate a key file.

2. Add the key file as a new key slot.

    ```bash
    sudo cryptsetup luksAddKey <LUKS block device> <key file>
    ```

3. Update `/etc/crypttab` to point to the key file.

4. Add a dracut config file to include the key file in the initramfs image.

    ```bash
    echo 'install_items+=" <key file> "' | sudo tee /etc/dracut.conf.d/99-key.conf
    ```

5. Regenerate all the initramfs images.

    ```bash
    sudo dracut -vf --regenerate-all
    ```

6. Verify that the initramfs images are only readable by root.

    ```bash
    sudo ls -l /boot/initramfs-*
    ```

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
