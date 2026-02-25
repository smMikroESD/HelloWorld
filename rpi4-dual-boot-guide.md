# Raspberry Pi 4 Dual Boot with U-Boot - Complete Implementation Guide

## Overview
This guide implements a dual boot system with:
- Two independent root filesystems (OS1 and OS2)
- Shared boot partition
- U-Boot environment for boot selection
- Linux tools to switch between OS instances
- Ability to update inactive partition

## Architecture

```
SD Card Layout:
├── /dev/mmcblk0p1 → /boot (FAT32, 256MB) - Shared boot files
├── /dev/mmcblk0p2 → /rootfs_a (ext4, 8GB) - OS1
├── /dev/mmcblk0p3 → /rootfs_b (ext4, 8GB) - OS2
└── /dev/mmcblk0p4 → /data (ext4, remaining) - Shared data (optional)
```

---

## STEP 1: Prepare Build Environment

### On your development machine (Windows with WSL or Linux PC):

```bash
# Install dependencies (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y git build-essential bison flex libssl-dev \
    bc device-tree-compiler python3 python3-setuptools swig \
    gcc-aarch64-linux-gnu u-boot-tools

# Create workspace
mkdir -p ~/rpi4-dualboot
cd ~/rpi4-dualboot
```

---

## STEP 2: Build U-Boot for Raspberry Pi 4

```bash
# Clone U-Boot (or use your existing u-boot directory)
git clone https://github.com/u-boot/u-boot.git
cd u-boot

# Checkout stable version
git checkout v2024.01

# Configure for Raspberry Pi 4 (64-bit)
make rpi_4_defconfig

# Optional: Customize configuration
make menuconfig
# Enable these options:
# - CONFIG_ENV_IS_IN_FAT=y (store env in FAT partition)
# - CONFIG_ENV_FAT_INTERFACE="mmc"
# - CONFIG_ENV_FAT_DEVICE_AND_PART="0:1"
# - CONFIG_ENV_FAT_FILE="uboot.env"
# - CONFIG_CMD_EDITENV=y
# - CONFIG_CMD_SAVEENV=y

# Build U-Boot
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)

# Result: u-boot.bin
ls -lh u-boot.bin
```

---

## STEP 3: Partition and Format SD Card

### ⚠️ WARNING: This will erase all data on SD card!

```bash
# Identify SD card (usually /dev/sdX or /dev/mmcblk0)
lsblk

# Set your SD card device (CHANGE THIS!)
export SDCARD=/dev/sdX  # e.g., /dev/sdb or /dev/mmcblk0

# Unmount if mounted
sudo umount ${SDCARD}*

# Create partition table
sudo parted ${SDCARD} mklabel msdos

# Create partitions
sudo parted ${SDCARD} mkpart primary fat32 1MiB 257MiB
sudo parted ${SDCARD} mkpart primary ext4 257MiB 8449MiB
sudo parted ${SDCARD} mkpart primary ext4 8449MiB 16641MiB
sudo parted ${SDCARD} mkpart primary ext4 16641MiB 100%

# Set boot flag
sudo parted ${SDCARD} set 1 boot on

# Format partitions
sudo mkfs.vfat -F 32 -n BOOT ${SDCARD}1     # or ${SDCARD}p1 if mmcblk
sudo mkfs.ext4 -L rootfs_a ${SDCARD}2        # or ${SDCARD}p2
sudo mkfs.ext4 -L rootfs_b ${SDCARD}3        # or ${SDCARD}p3
sudo mkfs.ext4 -L data ${SDCARD}4            # or ${SDCARD}p4
```

---

## STEP 4: Install RPi Firmware and U-Boot

```bash
# Mount boot partition
sudo mkdir -p /mnt/boot
sudo mount ${SDCARD}1 /mnt/boot

# Download Raspberry Pi firmware
cd ~/rpi4-dualboot
git clone --depth=1 https://github.com/raspberrypi/firmware.git

# Copy required files
sudo cp firmware/boot/start4.elf /mnt/boot/
sudo cp firmware/boot/fixup4.dat /mnt/boot/
sudo cp firmware/boot/bcm2711-rpi-4-b.dtb /mnt/boot/

# Copy U-Boot
sudo cp u-boot/u-boot.bin /mnt/boot/kernel8.img

# Create initial empty environment file
sudo dd if=/dev/zero of=/mnt/boot/uboot.env bs=1024 count=128
```

---

## STEP 5: Create U-Boot Configuration Files

### Create config.txt
```bash
sudo tee /mnt/boot/config.txt << 'EOF'
# Raspberry Pi 4 Configuration
arm_64bit=1
enable_uart=1
uart_2ndstage=1

# Kernel is actually U-Boot
kernel=kernel8.img

# Device tree
device_tree=bcm2711-rpi-4-b.dtb

# GPU memory
gpu_mem=128
EOF
```

### Create boot.scr.txt (U-Boot boot script)
```bash
sudo tee /mnt/boot/boot.scr.txt << 'EOF'
# Raspberry Pi 4 Dual Boot Script

echo "==================================="
echo "Raspberry Pi 4 Dual Boot (U-Boot)"
echo "==================================="

# Set default values
setenv bootdelay 2
setenv mmc_dev 0

# Set memory addresses for Raspberry Pi 4
# These MUST be set before loading anything for kernel decompression with initrd
setenv kernel_addr_r 0x00080000
setenv kernel_comp_addr_r 0x0A000000
setenv kernel_comp_size 0x04000000
setenv fdt_addr_r 0x02600000
setenv ramdisk_addr_r 0x02700000

# Load environment from file if it exists (use separate temp address)
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

# Set boot arguments for RPi4 serial console
# earlycon: Use Mini UART for early boot messages
# 8250.nr_uarts=1: Force registration of Mini UART as ttyS0
# console=ttyS0: Continue using serial after boot
setenv bootargs "earlycon=bcm2835aux,mmio32,0xfe215040 console=ttyS0,115200 8250.nr_uarts=1 root=/dev/mmcblk0p${root_part} rootfstype=ext4 rootwait rw"

# Load kernel and device tree
echo "Loading kernel from partition 1..."
fatload mmc ${mmc_dev}:1 ${kernel_addr_r} Image

echo "Loading device tree..."
fatload mmc ${mmc_dev}:1 ${fdt_addr_r} bcm2711-rpi-4-b.dtb

# Load initrd if it exists
if fatload mmc ${mmc_dev}:1 ${ramdisk_addr_r} initrd.img; then
    echo "Loading initrd..."
    setenv initrd_size ${filesize}
    # Boot with initrd
    echo "Starting Linux kernel with initrd..."
    booti ${kernel_addr_r} ${ramdisk_addr_r}:${initrd_size} ${fdt_addr_r}
else
    echo "No initrd found, booting without it..."
    # Boot without initrd
    booti ${kernel_addr_r} - ${fdt_addr_r}
fi
EOF
```

### Compile boot script
```bash
cd /mnt/boot
sudo mkimage -A arm64 -O linux -T script -C none -d boot.scr.txt boot.scr
```

---

## STEP 6: Install Operating Systems

### Download and Install Raspberry Pi OS (or your preferred distro)

```bash
# Download Raspberry Pi OS Lite (or full version)
cd ~/rpi4-dualboot
wget https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz

# Extract
xz -d 2024-03-15-raspios-bookworm-arm64-lite.img.xz

# Mount the image
sudo losetup -fP 2024-03-15-raspios-bookworm-arm64-lite.img
export LOOP=$(losetup -a | grep raspios | cut -d: -f1)
echo "Loop device: ${LOOP}"

# Mount rootfs from image
sudo mkdir -p /mnt/image_root
sudo mount ${LOOP}p2 /mnt/image_root

# Copy to both partitions
# OS1 (rootfs_a)
sudo mkdir -p /mnt/rootfs_a
sudo mount ${SDCARD}2 /mnt/rootfs_a
sudo rsync -axHAWX --numeric-ids --info=progress2 /mnt/image_root/ /mnt/rootfs_a/

# OS2 (rootfs_b) - Same OS initially, will update later
sudo mkdir -p /mnt/rootfs_b
sudo mount ${SDCARD}3 /mnt/rootfs_b
sudo rsync -axHAWX --numeric-ids --info=progress2 /mnt/image_root/ /mnt/rootfs_b/

# Cleanup
sudo umount /mnt/image_root
sudo losetup -d ${LOOP}
```

### Update fstab for both OS instances

```bash
# OS1 fstab
sudo tee /mnt/rootfs_a/etc/fstab << 'EOF'
/dev/mmcblk0p1  /boot           vfat    defaults          0       2
/dev/mmcblk0p2  /               ext4    defaults,noatime  0       1
/dev/mmcblk0p4  /data           ext4    defaults,noatime  0       2
EOF

# OS2 fstab
sudo tee /mnt/rootfs_b/etc/fstab << 'EOF'
/dev/mmcblk0p1  /boot           vfat    defaults          0       2
/dev/mmcblk0p3  /               ext4    defaults,noatime  0       1
/dev/mmcblk0p4  /data           ext4    defaults,noatime  0       2
EOF

# Create mount points
sudo mkdir -p /mnt/rootfs_a/boot /mnt/rootfs_a/data
sudo mkdir -p /mnt/rootfs_b/boot /mnt/rootfs_b/data
```

### Add kernel and initrd to boot partition

```bash
# For Raspberry Pi 4, use the rpi-v8 kernel (generic 64-bit ARM)
# Note: rpi-2712 is for RPi 5, rpi-v8 is for RPi 3/4

# Copy kernel
sudo cp /mnt/rootfs_a/boot/vmlinuz-*-rpi-v8 /mnt/boot/Image

# Copy initrd (initial ramdisk) - required for modern Raspberry Pi OS
sudo cp /mnt/rootfs_a/boot/initrd.img-*-rpi-v8 /mnt/boot/initrd.img

# Verify files are copied
ls -lh /mnt/boot/Image /mnt/boot/initrd.img
```

---

## STEP 7: Set Initial Boot Environment

```bash
# Create initial U-Boot environment configuration
cat > /tmp/uboot_env.txt << 'EOF'
boot_partition=a
boot_count=0
scriptaddr=0x02400000
bootcmd=fatload mmc 0:1 \${scriptaddr} boot.scr && source \${scriptaddr}
EOF

# Convert to binary format
mkenvimage -s 131072 -o /tmp/uboot.env /tmp/uboot_env.txt

# Copy to boot partition
sudo cp /tmp/uboot.env /mnt/boot/uboot.env

# Unmount everything
sudo umount /mnt/boot
sudo umount /mnt/rootfs_a
sudo umount /mnt/rootfs_b
```

---

## STEP 8: First Boot and U-Boot Tools Setup

### Boot the Raspberry Pi

1. Insert SD card into RPi4
2. Connect UART or HDMI to see boot messages
3. Power on
4. You should see U-Boot messages and then Linux boot from OS1

### Install U-Boot tools on both OS instances

```bash
# SSH or login to RPi4 (booted into OS1)
sudo apt-get update
sudo apt-get install -y u-boot-tools

# Create U-Boot environment configuration
sudo tee /etc/fw_env.config << 'EOF'
# Device name       Offset      Size        Erase block size
/boot/uboot.env     0x0000      0x20000     0x20000
EOF
```

---

## STEP 9: Create Boot Switching Scripts

### Create script to switch boot partition

```bash
sudo tee /usr/local/bin/switch-boot << 'EOF'
#!/bin/bash
# Script to switch boot partition for next reboot

set -e

CURRENT_ROOT=$(findmnt -n -o SOURCE /)
ENV_CONFIG="/etc/fw_env.config"

# Check if fw_setenv is available
if ! command -v fw_setenv &> /dev/null; then
    echo "Error: u-boot-tools not installed"
    echo "Run: sudo apt-get install u-boot-tools"
    exit 1
fi

# Determine current and other partition
if [[ "$CURRENT_ROOT" == *"mmcblk0p2"* ]]; then
    CURRENT="a"
    OTHER="b"
    OTHER_PART="/dev/mmcblk0p3"
elif [[ "$CURRENT_ROOT" == *"mmcblk0p3"* ]]; then
    CURRENT="b"
    OTHER="a"
    OTHER_PART="/dev/mmcblk0p2"
else
    echo "Error: Cannot determine current root partition"
    exit 1
fi

case "$1" in
    status)
        echo "Current Boot: OS$(echo $CURRENT | tr '[:lower:]' '[:upper:]') ($CURRENT_ROOT)"
        echo "Next Boot: $(fw_printenv -n boot_partition 2>/dev/null || echo 'unknown')"
        ;;
    
    switch)
        echo "Current root partition: $CURRENT_ROOT (OS$(echo $CURRENT | tr '[:lower:]' '[:upper:]'))"
        echo "Switching to OS$(echo $OTHER | tr '[:lower:]' '[:upper:]') for next boot..."
        
        # Check if other partition exists and has filesystem
        if ! blkid $OTHER_PART &> /dev/null; then
            echo "Warning: Target partition $OTHER_PART may not have a valid filesystem"
            read -p "Continue anyway? (y/N) " -n 1 -r
            echo
            if [[ ! $REPLY =~ ^[Yy]$ ]]; then
                exit 1
            fi
        fi
        
        # Set boot partition
        sudo fw_setenv boot_partition $OTHER
        echo "Boot partition set to: $OTHER"
        echo "Reboot to switch to OS$(echo $OTHER | tr '[:lower:]' '[:upper:]')"
        ;;
    
    revert)
        echo "Setting boot back to current OS ($CURRENT)..."
        sudo fw_setenv boot_partition $CURRENT
        echo "Done. System will continue booting OS$(echo $CURRENT | tr '[:lower:]' '[:upper:]')"
        ;;
    
    *)
        echo "Usage: $0 {status|switch|revert}"
        echo ""
        echo "  status  - Show current and next boot partition"
        echo "  switch  - Switch to the other OS for next boot"
        echo "  revert  - Keep booting the current OS"
        exit 1
        ;;
esac
EOF

sudo chmod +x /usr/local/bin/switch-boot
```

### Test the script

```bash
# Check current status
switch-boot status

# Switch to OS2
sudo switch-boot switch

# Reboot to test
sudo reboot
```

---

## STEP 10: Create Update Script for Inactive Partition

```bash
sudo tee /usr/local/bin/update-inactive-os << 'EOF'
#!/bin/bash
# Script to update the inactive OS partition

set -e

CURRENT_ROOT=$(findmnt -n -o SOURCE /)

# Determine inactive partition
if [[ "$CURRENT_ROOT" == *"mmcblk0p2"* ]]; then
    INACTIVE_PART="/dev/mmcblk0p3"
    INACTIVE_MOUNT="/mnt/inactive_os"
    TARGET_OS="b"
elif [[ "$CURRENT_ROOT" == *"mmcblk0p3"* ]]; then
    INACTIVE_PART="/dev/mmcblk0p2"
    INACTIVE_MOUNT="/mnt/inactive_os"
    TARGET_OS="a"
else
    echo "Error: Cannot determine current root partition"
    exit 1
fi

if [ -z "$1" ]; then
    echo "Usage: $0 <image.img> [--switch]"
    echo ""
    echo "Updates the inactive OS partition with new image"
    echo "  --switch: Automatically switch to new OS after update"
    exit 1
fi

IMAGE_FILE="$1"
AUTO_SWITCH=false

if [ "$2" == "--switch" ]; then
    AUTO_SWITCH=true
fi

if [ ! -f "$IMAGE_FILE" ]; then
    echo "Error: Image file not found: $IMAGE_FILE"
    exit 1
fi

echo "==================================="
echo "Updating Inactive OS Partition"
echo "==================================="
echo "Current root: $CURRENT_ROOT"
echo "Target partition: $INACTIVE_PART (OS$(echo $TARGET_OS | tr '[:lower:]' '[:upper:]'))"
echo "Image: $IMAGE_FILE"
echo ""
read -p "This will ERASE $INACTIVE_PART. Continue? (yes/NO) " -r
if [[ ! $REPLY =~ ^yes$ ]]; then
    echo "Aborted."
    exit 1
fi

# Create mount point
sudo mkdir -p "$INACTIVE_MOUNT"

# Format partition
echo "Formatting $INACTIVE_PART..."
sudo mkfs.ext4 -F -L "rootfs_$TARGET_OS" "$INACTIVE_PART"

# Mount partition
echo "Mounting $INACTIVE_PART..."
sudo mount "$INACTIVE_PART" "$INACTIVE_MOUNT"

# Extract/copy image
echo "Extracting image to $INACTIVE_PART..."
if [[ "$IMAGE_FILE" == *.tar.gz ]] || [[ "$IMAGE_FILE" == *.tgz ]]; then
    sudo tar -xzf "$IMAGE_FILE" -C "$INACTIVE_MOUNT"
elif [[ "$IMAGE_FILE" == *.tar ]]; then
    sudo tar -xf "$IMAGE_FILE" -C "$INACTIVE_MOUNT"
elif [[ "$IMAGE_FILE" == *.img ]]; then
    # Mount image and copy
    LOOP_DEV=$(sudo losetup -fP --show "$IMAGE_FILE")
    sudo mkdir -p /mnt/img_temp
    sudo mount "${LOOP_DEV}p2" /mnt/img_temp
    sudo rsync -axHAWX --numeric-ids --info=progress2 /mnt/img_temp/ "$INACTIVE_MOUNT"/
    sudo umount /mnt/img_temp
    sudo losetup -d "$LOOP_DEV"
else
    echo "Unsupported image format. Supported: .tar, .tar.gz, .tgz, .img"
    sudo umount "$INACTIVE_MOUNT"
    exit 1
fi

# Update fstab
echo "Updating fstab..."
if [ "$TARGET_OS" == "a" ]; then
    ROOTFS_PART="2"
else
    ROOTFS_PART="3"
fi

sudo tee "$INACTIVE_MOUNT/etc/fstab" << FSTAB_EOF
/dev/mmcblk0p1  /boot           vfat    defaults          0       2
/dev/mmcblk0p${ROOTFS_PART}  /               ext4    defaults,noatime  0       1
/dev/mmcblk0p4  /data           ext4    defaults,noatime  0       2
FSTAB_EOF

# Install u-boot-tools if not present
if [ ! -f "$INACTIVE_MOUNT/etc/fw_env.config" ]; then
    echo "Setting up U-Boot tools in new OS..."
    sudo tee "$INACTIVE_MOUNT/etc/fw_env.config" << FW_EOF
/boot/uboot.env     0x0000      0x20000     0x20000
FW_EOF
fi

# Copy boot management scripts
echo "Installing boot management scripts..."
sudo cp /usr/local/bin/switch-boot "$INACTIVE_MOUNT/usr/local/bin/"
sudo cp /usr/local/bin/update-inactive-os "$INACTIVE_MOUNT/usr/local/bin/"

# Sync and unmount
echo "Syncing..."
sync
sudo umount "$INACTIVE_MOUNT"

echo ""
echo "Update complete!"
echo "New OS installed on partition $INACTIVE_PART"

if [ "$AUTO_SWITCH" == true ]; then
    echo ""
    echo "Auto-switching to new OS..."
    sudo fw_setenv boot_partition "$TARGET_OS"
    echo "Reboot to boot into the new OS"
else
    echo ""
    echo "To switch to new OS, run:"
    echo "  sudo switch-boot switch"
    echo "  sudo reboot"
fi
EOF

sudo chmod +x /usr/local/bin/update-inactive-os
```

---

## STEP 11: Setup Shared Data Partition (Optional)

```bash
# Format data partition
sudo mkfs.ext4 -L data /dev/mmcblk0p4

# Mount and create directories
sudo mkdir -p /data
sudo mount /dev/mmcblk0p4 /data
sudo mkdir -p /data/config
sudo mkdir -p /data/updates

# Create shared boot config
sudo tee /data/config/boot-preference.conf << 'EOF'
# Shared boot configuration
# This file can be read/written by both OS instances
PREFERRED_OS=a
LAST_BOOT_TIME=$(date)
BOOT_REASON=manual
EOF

sudo chmod 666 /data/config/boot-preference.conf
```

---

## STEP 12: Testing and Verification

### Test 1: Check current boot
```bash
switch-boot status
fw_printenv boot_partition
df -h /
```

### Test 2: Switch to OS2
```bash
sudo switch-boot switch
sudo reboot
# After reboot, verify you're on OS2
switch-boot status
```

### Test 3: Switch back to OS1
```bash
sudo switch-boot switch
sudo reboot
```

### Test 4: Update inactive partition
```bash
# Download a new OS image
cd /tmp
wget https://example.com/new-os.img

# Update inactive partition
sudo update-inactive-os /tmp/new-os.img --switch

# Or download directly from URL
sudo update-inactive-os https://downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2024-03-15/2024-03-15-raspios-bookworm-arm64-lite.img.xz --switch

# Reboot to new OS
sudo reboot
```

---

## Summary of Commands

### Daily Operations

```bash
# Check which OS will boot next
switch-boot status

# Switch to other OS
sudo switch-boot switch && sudo reboot

# Update inactive OS
sudo update-inactive-os /path/to/image.img

# Update from URL
sudo update-inactive-os https://example.com/raspios.img.xz

# Update inactive OS and auto-switch
sudo update-inactive-os /path/to/image.img --switch && sudo reboot

# Update from URL and auto-switch
sudo update-inactive-os https://example.com/raspios.img.xz --switch && sudo reboot
```

### U-Boot Environment Commands

```bash
# View all variables
fw_printenv

# Set boot partition manually
sudo fw_setenv boot_partition a    # or b

# Reset environment
sudo fw_setenv boot_partition a
sudo fw_setenv boot_count 0
```

### Updating Boot Script

```bash
# If you need to update boot.scr after setup
# On your Linux PC with SD card mounted:
cd /mnt/boot

# Edit boot.scr.txt with your changes
sudo nano boot.scr.txt

# Recompile boot script
sudo mkimage -A arm64 -O linux -T script -C none -d boot.scr.txt boot.scr

# Or from the running RPi:
cd /boot
sudo nano boot.scr.txt
sudo mkimage -A arm64 -O linux -T script -C none -d boot.scr.txt boot.scr
sudo reboot
```

---

## Troubleshooting

### Issue: U-Boot doesn't start
- Check that kernel8.img is actually u-boot.bin
- Verify config.txt has `kernel=kernel8.img`
- Check UART output for errors

### Issue: fw_setenv command not found
```bash
sudo apt-get install u-boot-tools
```

### Issue: "Cannot parse config file"
- Verify /etc/fw_env.config exists
- Check that /boot/uboot.env file exists
- Recreate environment file if corrupted

### Issue: Boots to wrong OS
```bash
# Check environment
fw_printenv boot_partition

# Force to OS1
sudo fw_setenv boot_partition a

# Force to OS2
sudo fw_setenv boot_partition b
```

### Issue: Kernel panic / Cannot find root
- Check fstab in the target rootfs
- Verify boot.scr is loading correct partition
- Check U-Boot boot arguments with `printenv bootargs`

### Issue: Kernel boots but no console output after "Starting kernel..."
**Symptom:** You see kernel decompression and "Starting kernel..." but then no more output.

**Solution:** The console device needs to match RPi4's serial port. At U-Boot prompt:

setenv bootargs "console=ttyS0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rw"


Or for verbose boot debugging:

setenv bootargs "console=ttyS0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rw earlycon loglevel=7"


Then regenerate boot.scr with the updated script from Step 5.

### Issue: "kernel_comp_addr_r or kernel_comp_size is not provided"
**Solution:** Memory addresses need to be set before loading kernel. At U-Boot prompt:

setenv kernel_addr_r 0x00080000
setenv kernel_comp_addr_r 0x0A000000
setenv kernel_comp_size 0x04000000
setenv fdt_addr_r 0x02600000
setenv ramdisk_addr_r 0x02700000
boot


Permanent fix: Regenerate boot.scr with the updated script from Step 5.

---

## Advanced: Automatic Rollback

Add to /etc/rc.local or create systemd service:

```bash
#!/bin/bash
# Check if first boot after update
BOOT_COUNT=$(fw_printenv -n boot_count 2>/dev/null || echo "0")

if [ "$BOOT_COUNT" -eq "0" ]; then
    # First boot, increment counter
    fw_setenv boot_count 1
    echo "First boot of new OS, monitoring..."
    
    # Add health checks here
    # If checks fail:
    # fw_setenv boot_partition <old_partition>
    # reboot
elif [ "$BOOT_COUNT" -eq "1" ]; then
    # Second successful boot, consider it stable
    fw_setenv boot_count 2
    echo "OS verified stable"
fi
```

---

## Notes

1. **Backup**: Always backup your working SD card before major changes
2. **Kernel Updates**: When updating kernel, copy to /boot/Image
3. **Device Tree**: Keep DTB files in sync with kernel version
4. **Serial Console**: Keep UART adapter handy for debugging
5. **Environment Size**: Default env size is 128KB, increase if needed

---

## Quick Reference: Reflashing Boot Script

### Method 1: From Linux PC (SD card removed from RPi)

```bash
# Mount boot partition
sudo mount /dev/sdX1 /mnt/boot  # Change sdX to your SD card device

# Create the boot script (copy from Step 5 above)
sudo tee /mnt/boot/boot.scr.txt << 'EOF'
# (paste entire boot script from Step 5)
EOF

# Compile to boot.scr
cd /mnt/boot
sudo mkimage -A arm64 -O linux -T script -C none -d boot.scr.txt boot.scr

# Verify it was created
ls -lh boot.scr boot.scr.txt

# Unmount
cd ~
sudo umount /mnt/boot
```

### Method 2: From Running Raspberry Pi (via SSH or console)

```bash
# Install u-boot-tools if not already installed
sudo apt-get install u-boot-tools

# Edit the boot script
sudo nano /boot/boot.scr.txt

# Compile to boot.scr
cd /boot
sudo mkimage -A arm64 -O linux -T script -C none -d boot.scr.txt boot.scr

# Verify
ls -lh /boot/boot.scr*

# Reboot
sudo reboot
```

---

## References

- U-Boot Documentation: https://docs.u-boot.org/
- Raspberry Pi Firmware: https://github.com/raspberrypi/firmware
- U-Boot RPi4: https://u-boot.readthedocs.io/en/latest/board/raspberry-pi/index.html
