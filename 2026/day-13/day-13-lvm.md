# Day 13 – Linux Volume Management (LVM)

## 1️⃣ Commands Used

```bash
# Switch to root
sudo -i

# Check disks
lsblk

# Create Physical Volume
pvcreate /dev/nvme2n1
pvs

# Create Volume Group
vgcreate devops-vg /dev/nvme2n1
vgs

# Create Logical Volume
lvcreate -L 500M -n app-data devops-vg
lvs

# Format filesystem
mkfs.ext4 /dev/devops-vg/app-data

# Mount volume
mkdir -p /mnt/app-data
mount /dev/devops-vg/app-data /mnt/app-data
df -h /mnt/app-data

# Extend Logical Volume
lvextend -L +200M /dev/devops-vg/app-data
resize2fs /dev/devops-vg/app-data
df -h /mnt/app-data
