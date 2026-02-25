# U-Boot Environment Troubleshooting Guide

## Problem: fw_printenv fails with "Cannot read environment"

 Symptoms
```bash
$ fw_printenv boot_partition
Cannot read environment, using default
Cannot read default environment from file
```

 Root Cause
The `/boot/uboot.env` file is either:
- Wrong size (should be 131,072 bytes / 128KB)
- Corrupted or malformed
- Missing proper CRC32 checksum
- Not in correct binary format

 Quick Diagnostic

```bash
# Check if config file exists
ls -l /etc/fw_env.config

# Check uboot.env file size
ls -l /boot/uboot.env

# Expected output: -rwxr-xr-x 1 root root 131072 <date> /boot/uboot.env
#                                        ^^^^^^
#                                        Must be 131072 bytes (128KB)

# Check if boot partition is mounted
mount | grep /boot
```

---

## Solution: Recreate uboot.env File

 Step 1: Determine Current OS

```bash
# Check which OS you're currently on
findmnt -n -o SOURCE /

# If /dev/mmcblk0p2 → You're on OS1 (partition a)
# If /dev/mmcblk0p3 → You're on OS2 (partition b)
```

 Step 2: Create Proper Environment File

**If you're on OS1 (mmcblk0p2):**
```bash
# Create environment text file
cat > /tmp/uboot_env.txt << 'EOF'
boot_partition=a
boot_count=0
scriptaddr=0x02400000
bootcmd=fatload mmc 0:1 ${scriptaddr} boot.scr && source ${scriptaddr}
EOF

# Convert to binary format (128KB)
mkenvimage -s 131072 -o /tmp/uboot.env /tmp/uboot_env.txt

# Backup old file
sudo cp /boot/uboot.env /boot/uboot.env.backup

# Install new file
sudo cp /tmp/uboot.env /boot/uboot.env

# Verify size
ls -l /boot/uboot.env
# Should show: 131072 bytes

# Test
fw_printenv boot_partition
# Should show: boot_partition=a
```

**If you're on OS2 (mmcblk0p3):**
```bash
# Create environment text file (boot_partition=b for OS2)
cat > /tmp/uboot_env.txt << 'EOF'
boot_partition=b
boot_count=0
scriptaddr=0x02400000
bootcmd=fatload mmc 0:1 ${scriptaddr} boot.scr && source ${scriptaddr}
EOF

# Convert to binary format (128KB)
mkenvimage -s 131072 -o /tmp/uboot.env /tmp/uboot_env.txt

# Backup old file
sudo cp /boot/uboot.env /boot/uboot.env.backup

# Install new file
sudo cp /tmp/uboot.env /boot/uboot.env

# Verify size
ls -l /boot/uboot.env
# Should show: 131072 bytes

# Test
fw_printenv boot_partition
# Should show: boot_partition=b
```

 Step 3: Create /etc/fw_env.config (if missing)

```bash
# Check if config exists
cat /etc/fw_env.config

# If missing, create it:
sudo tee /etc/fw_env.config << 'EOF'
# Device name       Offset      Size        Erase block size
/boot/uboot.env     0x0000      0x20000     0x20000
EOF

# Verify
cat /etc/fw_env.config
```

 Step 4: Verify Everything Works

```bash
# View all environment variables
fw_printenv

# Expected output:
# boot_partition=a    (or b if on OS2)
# boot_count=0
# scriptaddr=0x02400000
# bootcmd=fatload mmc 0:1 ${scriptaddr} boot.scr && source ${scriptaddr}

# Test switch-boot script
switch-boot status

# Expected output:
# Current Boot: OSA (/dev/mmcblk0p2)    [or OSB if on OS2]
# Next Boot: a                           [or b if on OS2]
```

---

## Understanding the File Format

 uboot.env Structure

```
┌─────────────────────────────────────────────┐
│ CRC32 Checksum (4 bytes)                   │
├─────────────────────────────────────────────┤
│ Environment Variables (key=value pairs)     │
│ - boot_partition=a                          │
│ - boot_count=0                              │
│ - scriptaddr=0x02400000                     │
│ - bootcmd=fatload mmc 0:1 ...              │
│ - (null-terminated strings)                 │
├─────────────────────────────────────────────┤
│ Padding (zeros to fill 128KB)              │
└─────────────────────────────────────────────┘
Total Size: 131,072 bytes (128KB = 0x20000)
```

 Why 128KB?

The size **must match** what's configured in `/etc/fw_env.config`:
```
/boot/uboot.env     0x0000      0x20000     0x20000
                    offset      size        erase size
                                ^^^^^^
                                0x20000 = 131,072 bytes
```

U-Boot expects a fixed-size environment block for:
1. **Atomic updates** - Can write new env without corruption
2. **CRC validation** - Checksum covers entire block
3. **Memory mapping** - U-Boot maps this into memory at boot

---

## Common Issues

 Issue 1: Size Mismatch
**Problem:** File is 16KB instead of 128KB
```bash
ls -l /boot/uboot.env
-rwxr-xr-x 1 root root 16384 Feb 18 11:39 /boot/uboot.env
                              ^^^^^
                              Wrong! Should be 131072
```

**Fix:** Recreate with `mkenvimage -s 131072`

 Issue 2: Missing /etc/fw_env.config
**Problem:** fw_printenv can't find config
```bash
fw_printenv: Cannot parse config file '/etc/fw_env.config'
```

**Fix:** Create the config file with correct parameters

 Issue 3: Wrong boot_partition Value
**Problem:** System boots wrong OS after reboot
```bash
# You're on OS2 but file says boot_partition=a
fw_printenv boot_partition
boot_partition=a    # Wrong! Should be 'b'
```

**Fix:** Set correct value
```bash
sudo fw_setenv boot_partition b
```

 Issue 4: Corrupted After Update
**Problem:** Environment becomes unreadable after switch-boot
```bash
sudo switch-boot switch
# After reboot
fw_printenv
Cannot read environment
```

**Fix:** Recreate uboot.env with correct partition value

---

## Prevention: Automated Setup Script

Create this script to automatically fix uboot.env issues:

```bash
sudo tee /usr/local/bin/fix-uboot-env << 'EOF'
#!/bin/bash
# Fix U-Boot Environment Script

set -e

echo "==================================="
echo "U-Boot Environment Repair Tool"
echo "==================================="

# Detect current OS partition
CURRENT_ROOT=$(findmnt -n -o SOURCE /)

if [[ "$CURRENT_ROOT" == *"mmcblk0p2"* ]]; then
    BOOT_PART="a"
    OS_NAME="OS1"
elif [[ "$CURRENT_ROOT" == *"mmcblk0p3"* ]]; then
    BOOT_PART="b"
    OS_NAME="OS2"
else
    echo "Error: Cannot determine current partition"
    exit 1
fi

echo "Current OS: $OS_NAME (partition $BOOT_PART)"
echo ""

# Check if mkenvimage is available
if ! command -v mkenvimage &> /dev/null; then
    echo "Error: mkenvimage not found. Installing u-boot-tools..."
    sudo apt-get update
    sudo apt-get install -y u-boot-tools
fi

# Create environment text file
echo "Creating new environment file..."
cat > /tmp/uboot_env.txt << ENVEOF
boot_partition=$BOOT_PART
boot_count=0
scriptaddr=0x02400000
bootcmd=fatload mmc 0:1 \${scriptaddr} boot.scr && source \${scriptaddr}
ENVEOF

# Convert to binary
mkenvimage -s 131072 -o /tmp/uboot.env /tmp/uboot_env.txt

# Backup existing file
if [ -f /boot/uboot.env ]; then
    echo "Backing up existing uboot.env..."
    sudo cp /boot/uboot.env /boot/uboot.env.backup.$(date +%Y%m%d_%H%M%S)
fi

# Install new file
echo "Installing new environment file..."
sudo cp /tmp/uboot.env /boot/uboot.env

# Create fw_env.config if missing
if [ ! -f /etc/fw_env.config ]; then
    echo "Creating /etc/fw_env.config..."
    sudo tee /etc/fw_env.config > /dev/null << 'CONFIGEOF'
# Device name       Offset      Size        Erase block size
/boot/uboot.env     0x0000      0x20000     0x20000
CONFIGEOF
fi

# Verify
echo ""
echo "Verification:"
echo "============="
ls -lh /boot/uboot.env
echo ""
echo "Environment contents:"
fw_printenv

echo ""
echo "✓ U-Boot environment repaired successfully!"
echo "  boot_partition set to: $BOOT_PART ($OS_NAME)"
EOF

sudo chmod +x /usr/local/bin/fix-uboot-env
```

 Usage:
```bash
# Run anytime uboot.env is corrupted
sudo fix-uboot-env
```

---

## Testing After Repair

 Test 1: Read Environment
```bash
fw_printenv
# Should show all variables without errors
```

 Test 2: Check Status
```bash
switch-boot status
# Should show:
#   Current Boot: OSA (or OSB)
#   Next Boot: a (or b)
```

 Test 3: Test Switching
```bash
# Switch to other OS
sudo switch-boot switch

# Check what will boot next
fw_printenv boot_partition

# Reboot and verify you boot into the other OS
sudo reboot
```

 Test 4: Verify Persistence
```bash
# After reboot, change boot_partition
sudo fw_setenv boot_partition a

# Read it back
fw_printenv boot_partition
# Should show: boot_partition=a

# Reboot again and verify it persists
sudo reboot
```

---

## Summary

**The uboot.env file must be:**
- ✅ Exactly 131,072 bytes (128KB)
- ✅ Created with `mkenvimage -s 131072`
- ✅ Contains valid CRC32 checksum
- ✅ Has correct boot_partition value (a or b)
- ✅ Matches /etc/fw_env.config settings

**If fw_printenv fails, recreate the file using the steps above.**

**Key Commands:**
```bash
# Fix corrupted environment
sudo fix-uboot-env

# View environment
fw_printenv

# Change boot partition
sudo fw_setenv boot_partition a    # or b

# Check status
switch-boot status
```
