# Arch Linux Encrypted Root Auto-Install Guide

## 1. Kernel Parameters for Encrypted Root

To unlock the encrypted root partition at boot, kernel parameters must be set:

- Using the encrypt hook (common Arch method):

```text
cryptdevice=UUID=device-UUID:root root=/dev/mapper/root
````

* Using the systemd `rd.luks.name` alternative:

```text
rd.luks.name=device-UUID=root root=/dev/mapper/root
```

Where `device-UUID` refers to the **UUID of the LUKS superblock**.

---

## 2. Getting the LUKS UUID Dynamically

Instead of hardcoding a device, detect the device currently mounted at `/`:

```bash
ROOT_SRC=$(findmnt -n -o SOURCE /)
```

This returns the block device, e.g., `/dev/mapper/root`.

Then find the underlying LUKS device:

```bash
LUKS_DEV=$(cryptsetup status "$ROOT_SRC" | awk '/device:/ {print $2}')
```

Finally, get the UUID:

```bash
LUKS_UUID=$(cryptsetup luksUUID "$LUKS_DEV")
```

---

## 3. One-liner Version (Script-Friendly)

```bash
LUKS_UUID=$(cryptsetup luksUUID $(cryptsetup status $(findmnt -n -o SOURCE /) | awk '/device:/ {print $2}'))
```

This gives the UUID directly for kernel parameters.

---

## 4. Bootloader Integration

### GRUB:

```bash
KERNEL_PARAMS="cryptdevice=UUID=$LUKS_UUID:root root=/dev/mapper/root"

sed -i "s|^GRUB_CMDLINE_LINUX=\"|GRUB_CMDLINE_LINUX=\"$KERNEL_PARAMS |" /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```

### systemd-boot:

```bash
cat <<EOF > /boot/loader/entries/arch.conf
title   Arch Linux
linux   /vmlinuz-linux
initrd  /initramfs-linux.img
options $KERNEL_PARAMS rw
EOF
```

---

## 5. Notes for chroot

* Works inside `arch-chroot` as long as the LUKS device is unlocked.
* Ensure `/dev`, `/sys`, `/proc`, and `/run` are mounted inside chroot:

```bash
mount --types proc /proc /proc
mount --rbind /sys /sys
mount --rbind /dev /dev
mount --rbind /run /run
```

* If using LVM on LUKS, adjust accordingly (same principle applies).

---

## 6. Full Snippet for Inside Chroot

```bash
# Get the device mounted at /
ROOT_SRC=$(findmnt -n -o SOURCE /)

# Get the underlying LUKS device
LUKS_DEV=$(cryptsetup status "$ROOT_SRC" | awk '/device:/ {print $2}')

# Get the LUKS UUID
LUKS_UUID=$(cryptsetup luksUUID "$LUKS_DEV")

# Generate the kernel parameter for encrypt hook
KERNEL_PARAMS="cryptdevice=UUID=$LUKS_UUID:root root=/dev/mapper/root"

# Show for verification
echo "Root device: $ROOT_SRC"
echo "LUKS device: $LUKS_DEV"
echo "LUKS UUID: $LUKS_UUID"
echo "Kernel parameters: $KERNEL_PARAMS"

# Example for GRUB
sed -i "s|^GRUB_CMDLINE_LINUX=\"|GRUB_CMDLINE_LINUX=\"$KERNEL_PARAMS |" /etc/default/grub
grub-mkconfig -o /boot/grub/grub.cfg
```

---

This snippet can be adapted for:

* Automated Arch install scripts
* systemd-boot instead of GRUB
* Detecting root device dynamically in both live environment and chroot

```

---

If you paste this into a file named `arch-encrypt-guide.md`, you’ll have the **full session in Markdown** with code blocks and explanations ready to use.  

If you want, I can also make a **ready-to-run shell command** that **creates this `.md` file automatically in one shot**. Do you want me to do that?
```
