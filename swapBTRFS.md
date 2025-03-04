I copied and pasted the content from Deepseek because I got lazy. I put it here to document it for future reference and research. Need to understand it first
---

Here's the **best approach for swap configuration** with Btrfs on your ThinkPad T460p (i5-6440HQ/16GB RAM/1TB SSD):

---

### **Recommended Setup: Swap File on Btrfs**
**Why this works**:
- Modern Btrfs (kernel 5.0+) supports swap files **if CoW is disabled**.
- Avoids partitioning headaches and maximizes flexibility.
- Works seamlessly with ZRAM for tiered memory management.

---

### **Step-by-Step Configuration**
1. **Create Dedicated Swap Subvolume**:
   ```bash
   # While installing:
   btrfs subvolume create /mnt/@swap
   mount -o nodatacow,subvol=@swap /dev/sda2 /mnt/swap
   ```

2. **Create Swap File**:
   ```bash
   truncate -s 0 /mnt/swap/swapfile
   chattr +C /mnt/swap/swapfile     # Disable CoW
   fallocate -l 4G /mnt/swap/swapfile
   chmod 600 /mnt/swap/swapfile
   mkswap /mnt/swap/swapfile
   ```

3. **Add to fstab**:
   ```bash
   echo "/swap/swapfile none swap defaults,discard 0 0" >> /mnt/etc/fstab
   ```

---

### **Why This Beats a Separate Partition**
| **Factor**          | Swap File on Btrfs                          | Separate Swap Partition      |
|----------------------|---------------------------------------------|-------------------------------|
| **Flexibility**      | Easily resize (e.g., `fallocate -l 8G ...`) | Fixed size, requires repartitioning |
| **Space Efficiency** | Shares Btrfs space dynamically              | Dedicated unused partition    |
| **SSD Optimization** | Supports `discard` for TRIM                 | Requires manual TRIM setup    |
| **Stability**        | Works flawlessly if CoW disabled            | Risk of misaligned partitions |

---

### **ZRAM Integration**
Add **ZRAM as primary swap** and **Btrfs swap file as fallback**:
```bash
# Install ZRAM generator
pacman -S zram-generator

# Configure ZRAM (uses 50% RAM)
echo "[zram0]
zram-size = min(ram / 2, 8192)
compression-algorithm = zstd
swap-priority = 100" > /etc/systemd/zram-generator.conf

# Configure swap file (lower priority)
echo "/swap/swapfile none swap defaults,discard,pri=50 0 0" >> /etc/fstab
```

---

### **Verification**
After reboot:
```bash
# Check swap devices (ZRAM first, then file)
swapon --show

NAME       TYPE      SIZE   USED PRIO
/dev/zram0 partition   8G   0B    100
/swapfile  file       4G    0B    50
```

---

### **When to Use a Swap Partition**
Only consider a separate partition if:
- Using legacy filesystems (ext4/XFS)
- Older kernel (<5.0)
- Enterprise systems requiring 100% uptime guarantees

---

### **Your Case (T460p Optimized)**
1. **Keep your existing Btrfs layout** (`@`, `@home`, etc.).
2. **Add `@swap` subvolume** with CoW disabled.
3. **Combine ZRAM + swap file** for:
   - Fast compressed swapping (ZRAM)
   - Physical fallback (swap file)
   - No partition table changes

This setup gives you **maximum flexibility** while maintaining stability and performance on Btrfs.
