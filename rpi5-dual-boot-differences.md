# Raspberry Pi 5 Dual Boot - Changes from RPi 4 Guide

This document lists **ONLY** the differences you need to apply to the main [rpi4-dual-boot-guide.md](rpi4-dual-boot-guide.md) to make it work on **Raspberry Pi 5**.

---

## STEP 2: Build U-Boot for Raspberry Pi 5

**Change:**
```bash
# Instead of: make rpi_4_defconfig
make rpi_5_defconfig

# Build remains the same
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

---

## STEP 4: Install RPi Firmware and U-Boot

**Change firmware files:**
```bash
# Instead of start4.elf and fixup4.dat, use:
sudo cp firmware/boot/start5.elf /mnt/boot/
sudo cp firmware/boot/fixup5.dat /mnt/boot/

# Different device tree:
sudo cp firmware/boot/bcm2712-rpi-5-b.dtb /mnt/boot/

# U-Boot remains: kernel8.img (same)
```

---

## STEP 5: Create U-Boot Configuration Files

**config.txt - Update device tree:**
```bash
sudo tee /mnt/boot/config.txt << 'EOF'
# Raspberry Pi 5 Configuration
arm_64bit=1
enable_uart=1
uart_2ndstage=1

# Kernel is actually U-Boot
kernel=kernel8.img

# Device tree for RPi 5
device_tree=bcm2712-rpi-5-b.dtb

# GPU memory
gpu_mem=128
EOF
```

**boot.scr.txt - Update console and device tree:**
```bash
sudo tee /mnt/boot/boot.scr.txt << 'EOF'
# Raspberry Pi 5 Dual Boot Script

echo "==================================="
echo "Raspberry Pi 5 Dual Boot (U-Boot)"
echo "==================================="

# Set default values
setenv bootdelay 2
setenv mmc_dev 0

# Set memory addresses for Raspberry Pi 5
setenv kernel_addr_r 0x00080000
setenv kernel_comp_addr_r 0x0A000000
setenv kernel_comp_size 0x04000000
setenv fdt_addr_r 0x02600000
setenv ramdisk_addr_r 0x02700000

# Load environment from file
fatload mmc ${mmc_dev}:1 0x02000000 uboot.env && env import -b 0x02000000

# Check boot_partition variable (default to 'a')
if test "${boot_partition}" = "b"; then
    echo "Booting OS2 (rootfs_b) from partition 3..."
    setenv root_part 3
    setenv root_label "rootfs_b"
else
    echo "Booting OS1 (rootfs_a) from partition 2..."
    setenv root_part 2
    setenv root_label "rootfs_a"
fi

# RPi 5 serial console arguments
# Note: RPi 5 uses different UART - verify your setup
setenv bootargs "console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p${root_part} rootfstype=ext4 rootwait rw"

# Load kernel and device tree
echo "Loading kernel from partition 1..."
fatload mmc ${mmc_dev}:1 ${kernel_addr_r} Image

echo "Loading device tree for RPi 5..."
fatload mmc ${mmc_dev}:1 ${fdt_addr_r} bcm2712-rpi-5-b.dtb

# Load initrd if it exists
if fatload mmc ${mmc_dev}:1 ${ramdisk_addr_r} initrd.img; then
    echo "Loading initrd..."
    setenv initrd_size ${filesize}
    echo "Starting Linux kernel with initrd..."
    booti ${kernel_addr_r} ${ramdisk_addr_r}:${initrd_size} ${fdt_addr_r}
else
    echo "No initrd found, booting without it..."
    booti ${kernel_addr_r} - ${fdt_addr_r}
fi
EOF
```

---

## STEP 6: Install Operating Systems

**Change kernel selection:**
```bash
# RPi 5 uses rpi-2712 kernel instead of rpi-v8

# Copy kernel
sudo cp /mnt/rootfs_a/boot/vmlinuz-*-rpi-2712 /mnt/boot/Image

# Copy initrd
sudo cp /mnt/rootfs_a/boot/initrd.img-*-rpi-2712 /mnt/boot/initrd.img

# Verify files
ls -lh /mnt/boot/Image /mnt/boot/initrd.img
```

---

## Important Notes for Raspberry Pi 5

### Serial Console
- **RPi 5 uses different UART:** `ttyAMA0` instead of `ttyS0`
- Console bootargs: `console=ttyAMA0,115200` instead of `console=ttyS0,115200`
- No need for `8250.nr_uarts=1` or `earlycon=bcm2835aux` on RPi 5

### Kernel Naming
- **RPi 4:** `vmlinuz-*-rpi-v8` and `initrd.img-*-rpi-v8`
- **RPi 5:** `vmlinuz-*-rpi-2712` and `initrd.img-*-rpi-2712`

### Device Tree
- **RPi 4:** `bcm2711-rpi-4-b.dtb`
- **RPi 5:** `bcm2712-rpi-5-b.dtb`

### Firmware Files
- **RPi 4:** `start4.elf`, `fixup4.dat`
- **RPi 5:** `start5.elf`, `fixup5.dat`

### U-Boot Configuration
- **RPi 4:** `rpi_4_defconfig`
- **RPi 5:** `rpi_5_defconfig`

---

## Testing Checklist for RPi 5

After implementing these changes:

1. ✓ Verify U-Boot boots and shows RPi 5 in output
2. ✓ Check serial console works on ttyAMA0
3. ✓ Confirm kernel loads (rpi-2712)
4. ✓ Test OS1 boots successfully
5. ✓ Test OS2 boots successfully
6. ✓ Verify switch-boot script works
7. ✓ Test update-inactive-os script

---

## Quick Summary: What Changes

| Component | RPi 4 | RPi 5 |
|-----------|-------|-------|
| U-Boot config | `rpi_4_defconfig` | `rpi_5_defconfig` |
| Firmware | `start4.elf`, `fixup4.dat` | `start5.elf`, `fixup5.dat` |
| Device Tree | `bcm2711-rpi-4-b.dtb` | `bcm2712-rpi-5-b.dtb` |
| Kernel | `vmlinuz-*-rpi-v8` | `vmlinuz-*-rpi-2712` |
| Serial Console | `ttyS0` (Mini UART) | `ttyAMA0` (PL011 UART) |
| Console Args | `earlycon=bcm2835aux,mmio32,0xfe215040 console=ttyS0,115200 8250.nr_uarts=1` | `console=ttyAMA0,115200` |

---

## Everything Else Stays the Same

All other steps, scripts, and procedures from the RPi 4 guide work identically on RPi 5:
- Partition layout
- U-Boot environment (uboot.env)
- switch-boot script
- update-inactive-os script
- connect-wifi script
- fix-uboot-env script
- WiFi configuration
- fstab configuration
- All management scripts

**Just apply the 6 changes above and you're ready for RPi 5!**

---

## Password Recovery / Emergency Boot

If you can't login (wrong password, locked account, etc.), use this emergency boot method:

### Method 1: Boot with init=/bin/bash (From U-Boot)

**At U-Boot prompt (interrupt boot by pressing any key):**

**For RPi 5:**
```
# Modify bootargs to boot directly to bash
setenv bootargs "console=ttyAMA0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rw init=/bin/bash"

# Load kernel and boot
fatload mmc 0:1 ${kernel_addr_r} Image
fatload mmc 0:1 ${fdt_addr_r} bcm2712-rpi-5-b.dtb
fatload mmc 0:1 ${ramdisk_addr_r} initrd.img
setenv initrd_size ${filesize}
booti ${kernel_addr_r} ${ramdisk_addr_r}:${initrd_size} ${fdt_addr_r}
```

**For RPi 4:**
```
# Modify bootargs to boot directly to bash
setenv bootargs "console=ttyS0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rw init=/bin/bash 8250.nr_uarts=1"

# Load kernel and boot
fatload mmc 0:1 ${kernel_addr_r} Image
fatload mmc 0:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb
fatload mmc 0:1 ${ramdisk_addr_r} initrd.img
setenv initrd_size ${filesize}
booti ${kernel_addr_r} ${ramdisk_addr_r}:${initrd_size} ${fdt_addr_r}
```

**Once you get the root shell:**
```bash
# Remount root as read-write (if needed)
mount -o remount,rw /

# Reset password for user 'pi'
passwd pi
# Enter new password twice

# Sync and reboot
sync
reboot -f
```

### Method 2: Use the Other OS (Chroot Method)

If you can boot into the other OS, you can fix the broken one:

**From OS1, to fix OS2 (or vice versa):**
```bash
# Mount the inactive OS
sudo mkdir -p /mnt/inactive_os
sudo mount /dev/mmcblk0p3 /mnt/inactive_os  # Use p2 if fixing OS1 from OS2

# Mount required filesystems for chroot
sudo mount --bind /dev /mnt/inactive_os/dev
sudo mount --bind /proc /mnt/inactive_os/proc
sudo mount --bind /sys /mnt/inactive_os/sys

# Enter chroot
sudo chroot /mnt/inactive_os /bin/bash

# Reset password
passwd pi

# Exit and cleanup
exit
sudo umount /mnt/inactive_os/dev
sudo umount /mnt/inactive_os/proc
sudo umount /mnt/inactive_os/sys
sudo umount /mnt/inactive_os
```

**Or one-liner without chroot:**
```bash
# Mount target partition
sudo mount /dev/mmcblk0p3 /mnt/inactive_os

# Generate password hash and update shadow file
NEWPASS=$(openssl passwd -6 "your_new_password")
sudo sed -i "s|^pi:[^:]*:|pi:$NEWPASS:|" /mnt/inactive_os/etc/shadow

# Unmount
sudo umount /mnt/inactive_os
```

### Method 3: Add New User to Inactive OS (Chroot)

You can create new users in the inactive OS without booting into it:

**Interactive method:**
```bash
# Mount the inactive OS
sudo mkdir -p /mnt/inactive_os
sudo mount /dev/mmcblk0p3 /mnt/inactive_os  # p3 if on OS1, p2 if on OS2

# Mount required filesystems for chroot
sudo mount --bind /dev /mnt/inactive_os/dev
sudo mount --bind /proc /mnt/inactive_os/proc
sudo mount --bind /sys /mnt/inactive_os/sys

# Enter chroot
sudo chroot /mnt/inactive_os /bin/bash

# Create new user with home directory
useradd -m -s /bin/bash newuser

# Set password
passwd newuser

# Add to groups (optional)
usermod -aG sudo,adm,video,audio newuser

# Verify user was created
id newuser

# Exit chroot
exit

# Cleanup
sudo umount /mnt/inactive_os/dev
sudo umount /mnt/inactive_os/proc
sudo umount /mnt/inactive_os/sys
sudo umount /mnt/inactive_os
```

**One-liner method (non-interactive):**
```bash
# Mount partition
sudo mount /dev/mmcblk0p3 /mnt/inactive_os
sudo mount --bind /dev /mnt/inactive_os/dev
sudo mount --bind /proc /mnt/inactive_os/proc
sudo mount --bind /sys /mnt/inactive_os/sys

# Create user with password in one command
sudo chroot /mnt/inactive_os /bin/bash -c "useradd -m -s /bin/bash newuser && echo 'newuser:password123' | chpasswd && usermod -aG sudo,adm,video,audio newuser"

# Verify
sudo chroot /mnt/inactive_os /bin/bash -c "id newuser"

# Cleanup
sudo umount /mnt/inactive_os/dev
sudo umount /mnt/inactive_os/proc
sudo umount /mnt/inactive_os/sys
sudo umount /mnt/inactive_os
```

**Other useful user management in chroot:**
```bash
# Delete user
sudo chroot /mnt/inactive_os /bin/bash -c "userdel -r username"

# List all users
sudo chroot /mnt/inactive_os /bin/bash -c "cat /etc/passwd"

# Add SSH key
sudo chroot /mnt/inactive_os /bin/bash -c "mkdir -p /home/username/.ssh && echo 'ssh-rsa AAAA...' >> /home/username/.ssh/authorized_keys && chown -R username:username /home/username/.ssh"
```

### When to Use Each Method

| Scenario | Method |
|----------|--------|
| Can't login to either OS | Use `init=/bin/bash` at U-Boot |
| Locked out of OS2, can access OS1 | Use chroot from OS1 |
| Locked out of OS1, can access OS2 | Use chroot from OS2 |
| Want to automate password changes | Use shadow file editing |
| Need to add users before first boot | Use chroot with user management commands |

---

## Common Issues on RPi 5

### Issue: No serial console output
**Solution:** Verify you're using `console=ttyAMA0,115200` (not ttyS0)

### Issue: Kernel doesn't load
**Solution:** Make sure you're using `rpi-2712` kernel, not `rpi-v8`

### Issue: Wrong device tree
**Solution:** Use `bcm2712-rpi-5-b.dtb`, not `bcm2711-rpi-4-b.dtb`

### Issue: Default password rejected
**Solution:** Use the password recovery methods above
