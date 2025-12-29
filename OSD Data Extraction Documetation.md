# Manual Data Extraction (Advanced) 
If you cannot rebuild the cluster, you can attempt to export specific Placement Groups (PGs) or objects directly.
### Before proceed to data extraction 
- Prepare any linux based machine and install ceph(latest version is safe) on it (In this guidline we use proxmox 8).
- Prepare a recovery disk and mount it to a directory in my case 400GB disk is mounted to /mnt/recovery (size of disk must be comparable to ceph disk)
### To manually extract data from offline OSDs in 2025, you will use 
We use `ceph-objectstore-tool` to pull individual pieces (objects) and then reassemble them. This is the "last resort" if the cluster monitors cannot be rebuilt.
We use `ceph-volume lvm list` to see which OSDs are present on your LVM volumes and grep the **cluster FSID**, because ceph-volume expects a valid cluster configuration to perform its tasks, even for "offline" activation. Since your original cluster is gone, you must provide a minimal configuration and then manually "prime" the OSD directories. use the Cluster FSID found in your ceph-volume lvm list output.
- Create a Minimal ceph.conf
```
cat <<EOF > /etc/ceph/ceph.conf
[global]
fsid = 46158e5c-c2e7-4169-a12a-16cb77e49f79
mon_host = 127.0.0.1
auth_cluster_required = none
auth_service_required = none
auth_client_required = none
EOF
```
# The steps taken to recover data:
## 1. Locate the OSD Data Paths
First, identify where your OSDs are mounted or mapped. If you have just attached the disks to a new server, use:
```
ceph-volume lvm activate --all --no-systemd
```
--all: Attempts to find and mount all OSDs.
--no-systemd: Crucial for recovery. It maps the disks to /var/lib/ceph/osd/ but does not try to start the OSD background service (which would fail anyway without a monitor).
The data paths are typically /var/lib/ceph/osd/ceph-'$ID'. Verify this by looking for a block or data file in those directories. 
`ls -l /var/lib/ceph/osd/`
You should see folders like ceph-0, ceph-1, etc. If you enter one (e.g., cd /var/lib/ceph/osd/ceph-0), you should see a symlink named block pointing to your LVM volume (like /dev/ceph-...).
## 2. List Content on an OSD
Before extracting, see what is on a specific OSD. Use the --op list command:
```
# List all objects on OSD 0
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-0 --op list
```
This will output a large list of object IDs (e.g., rbd_data.10a46b2ae31a1.0000000000000000). 
## 3. Kill any active Ceph OSD processes
Even if systemctl status says they are stopped, ghost processes may be holding the lock.
```
# Kill any lingering OSD processes
killall -9 ceph-osd
systemctl stop ceph-osd.target
systemctl stop ceph-osd@0
systemctl stop ceph-osd@1
systemctl stop ceph-osd@2
rm -f /var/lib/ceph/osd/ceph-0/lock
rm -f /var/lib/ceph/osd/ceph-1/lock
rm -f /var/lib/ceph/osd/ceph-2/lock
```
## 4. Clear existing locks
Sometimes a lock file persists even after the process dies. In BlueStore, the tool needs exclusive access to the fsid file in the OSD directory.
```
# Check if a process is still using the OSD directory
fuser -v /var/lib/ceph/osd/ceph-0
fuser -v /var/lib/ceph/osd/ceph-1
fuser -v /var/lib/ceph/osd/ceph-2
```
## 5. Extract Data from All OSDs (In my case 3 OSDs)
This script iterates through OSD 0, 1, and 2. It searches each for the specific RBD prefix (dcfe27b3b5a  -- in my case for you it is different string) and saves every chunk to your recovery disk (sde  --is my recovery disk).
```
PREFIX="rbd_data.dcfe27b3b5a"
RECOVERY_DIR="/mnt/recovery/chunks"
mkdir -p "$RECOVERY_DIR"

for ID in 0 1 2; do
    OSD_PATH="/var/lib/ceph/osd/ceph-$ID"
    echo "--- Processing OSD.$ID ---"
    
    if [ -L "$OSD_PATH/block" ]; then
        # Step 1: Generate the list of objects and save to a temp file
        LIST_FILE="/tmp/osd${ID}_list.txt"
        ceph-objectstore-tool --data-path "$OSD_PATH" --no-mon-config --op list | grep "$PREFIX" > "$LIST_FILE"
        
        # Step 2: Iterate through the file (no nested tool calls)
        while read -r line; do
            OID=$(echo "$line" | grep -oP 'oid":"\K[^"]+')
            PGID=$(echo "$line" | awk -F'["'\'']' '{print $2}')
            
            echo "  Extracting $OID..."
            # Step 3: Extract the specific object
            ceph-objectstore-tool --data-path "$OSD_PATH" --no-mon-config --pgid "$PGID" "$line" get-bytes > "$RECOVERY_DIR/$OID"
        done < "$LIST_FILE"
        
        rm "$LIST_FILE"
    else
        echo "  OSD.$ID block not found, skipping."
    fi
done
```
After the script finishes, check the chunk directory:
```
ls /mnt/recovery/chunks | wc -l
```
If your RBD image was 32GB (in my case) and used 4MB objects, you should ideally see up to 8,192 files (4MB times 8192 = 32GB). If you see significantly fewer, some data may be missing or stored on a different OSD you haven't processed yet.

## 6. Advanced: Reassembling RBD Images (VM Disks)
Ceph stores VM disks (RBD images) as many 4MB objects. To recover a full disk manually: 
Extract all objects belonging to the RBD image ID (e.g., dcfe27b3b5a) from all OSDs.
    Use rbd-recover-tool: This specialized tool (if available in your Proxmox Ceph version) can automate the reassembly of these objects into a single .img file without a running cluster.
    
```
FINAL_IMAGE="/mnt/recovery/recovered_vm.img"
CHUNK_DIR="/mnt/recovery/chunks"
OBJ_SIZE=4194304  # Standard 4MB Ceph object size

echo "Starting reassembly..."
# 1. Create a blank sparse file (set size to what the disk originally was, e.g., 32G)
truncate -s 32G "$FINAL_IMAGE"

# 2. Iterate through extracted chunks and place them at the correct offset
cd "$CHUNK_DIR"
for file in rbd_data.dcfe27b3b5a.*; do
    # Get the hex suffix (e.g., 0000000000001c8b -> 1c8b)
    HEX=$(echo "$file" | awk -F. '{print $NF}')
    
    # Convert hex to decimal index
    INDEX=$((16#$HEX))
    
    # Write the 4MB chunk into the final image at the calculated block offset
    dd if="$file" of="$FINAL_IMAGE" bs=$OBJ_SIZE seek=$INDEX conv=notrunc status=none
done

echo "Reassembly finished: $FINAL_IMAGE"
```

## 7. How to verify the recovered file
After reassembly, verify if the partition table is readable. If it is, you have successfully recovered the disk structure:

Check partitions: 
```
fdisk -l /mnt/recovery/recovered_vm.img
```
Mount and browse:
```
mkdir -p /mnt/temp_recovery
# Use -o loop,ro to protect against further corruption
mount -o loop,ro,offset=540016640 /mnt/recovery/recovered_vm_disk.img /mnt/temp_recovery
# (Note: 'offset' is usually 1048576 for standard Linux partitions, 
# but use 'fdisk' output to find the exact start sector * 512)
```
 
## Optional for step 7
Create a vm then attach this /mnt/recovery/recovered_vm.img as scsi0 and boot the vm from scsi0. Lets assume VM id is 100
```
qm importdisk 100 /mnt/recovery/recovered_vm_disk.img local-lvm
# Attach the imported disk as SCSI 0
qm set 100 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-100-disk-0
# Set the boot order to use this disk
qm set 100 --boot order=scsi0
qm start 100
```
# References
https://forum.proxmox.com/threads/recover-data-from-ceph-osds.138259/
