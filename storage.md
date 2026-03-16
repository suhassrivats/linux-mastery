# **1. Physical Storage Installed**

Example: you plug in an **SSD, NVMe, or HDD**.

* Hardware sees: SATA, NVMe bus, PCIe interface.
* BIOS/UEFI detects device.
* Kernel detects device during boot (or hot-plug via `udev`).

Kernel creates a **block device**:

```text
/dev/sda   (HDD)
```

* Kernel structures:

  * `struct gendisk` → represents the whole disk
  * `struct block_device` → kernel handle for the device
  * `struct request_queue` → queue for I/O requests to that disk
* Device driver manages communication over the bus (SATA/NVMe driver).

**Flow so far:**

```text
Physical disk → Bus → Kernel driver → Block device /dev/sdX → request queue
```

---

# **2. Partitioning**

If you want multiple logical disks:

1. Use a **partition table**: MBR or GPT.
2. Kernel creates block devices per partition:

```text
/dev/sda1, /dev/sda2
```

* Each partition is still a **block device**.
* Kernel maintains **metadata**: start sector, size, type.
* Structures: `struct hd_geometry`, still mapped to `struct gendisk`.

---

# **3. Filesystem Creation**

* Before storing files, you **format a partition** with a filesystem (ext4, XFS, Btrfs…):

```bash
mkfs.ext4 /dev/sda1
```

* Creates:

  * **Superblock** → metadata about filesystem (size, block count, free blocks)
  * **Inodes** → metadata for files
  * **Block bitmap** → tracks which blocks are free/used
  * **Directory structures** → map filenames → inode numbers

* Kernel mounts the filesystem:

```bash
mount /dev/sda1 /mnt/data
```

* Kernel now knows **which blocks belong to which files**.

* Kernel structures involved:

```text
struct super_block → entire filesystem
struct inode → per file/directory
struct dentry → directory entry mapping
struct address_space → maps file data to memory (page cache)
```

---

# **4. Virtual File System (VFS)**

Linux uses **VFS** to give a **uniform interface** to all filesystems.

* System call like `read()` doesn’t know ext4 vs XFS.
* VFS structures:

```text
struct file → open file instance
struct inode → metadata for file
struct dentry → name → inode mapping
```

* VFS provides operations: `read()`, `write()`, `open()`, `close()` for all files.

---

# **5. Block Layer / I/O Scheduling**

When a file is accessed:

1. Kernel checks **page cache** first.

   * If data is cached → return from RAM (fast)
   * If not cached → issue **block I/O request**
2. Block layer receives request:

```text
struct request_queue → queues read/write requests
```

3. Scheduler orders requests (I/O elevator):

* `noop` → FIFO (SSD)
* `deadline` → deadlines for requests
* `cfq` → fair I/O among processes

4. Request sent to device driver → controller → disk.

---

# **6. Device Driver → Storage Hardware**

* Device driver translates **block request → command for disk**.
* Example: NVMe driver sends **read/write commands over PCIe**.
* Disk controller reads/writes **physical blocks on platters or NAND cells**.
* Hardware reports completion → interrupt generated.

---

# **7. Data Return Path**

* Disk sends data → driver → kernel **fills page cache**.
* VFS reads from page cache → `read()` returns to process.
* If multiple processes read same file, page cache prevents multiple physical reads.

---

# **8. Writing Data**

* `write()` call:

```text
Process → VFS → page cache → mark dirty → schedule write-back
```

* Linux **does not write immediately**:

  * Dirty pages are flushed later by **pdflush / writeback threads**
  * Journaling filesystem ensures metadata consistency

* Physical disk updated in blocks asynchronously.

---

# **9. Memory Integration**

* Each open file has **struct file**
* Each file has **struct inode**
* Each inode maps to **pages in RAM**
* Pages are synced to disk asynchronously

```text
task_struct → files_struct → struct file → inode → page cache → disk
```

---

# **10. Process-Level Flow**

1. `task_struct` represents process.
2. `files_struct` → tracks all open FDs.
3. `struct file` → per open file (offset, flags)
4. `inode` → file metadata
5. `address_space` → maps file blocks → memory pages
6. Page cache → cached blocks
7. Block layer → request queue → driver → physical disk
8. Disk reads/writes actual sectors.

---

# **11. Diagrammatic Flow**

```text
+-------------------+
|   Process (task_struct)   |
|   FD table (files_struct) |
+-----------+-------+
            |
       struct file
            |
       struct inode
            |
   struct address_space
            |
       Page Cache (RAM)
            |
       VFS / filesystem (ext4/XFS)
            |
       Block Layer (request_queue)
            |
       Device Driver
            |
   Physical Disk (SATA/NVMe/HDD)
```

* Reads → check page cache → block device → disk
* Writes → page cache → writeback → block device → disk

---

# **12. Key Kernel Structures**

| Layer         | Structure                  | Notes                                |
| ------------- | -------------------------- | ------------------------------------ |
| Process       | task_struct                | contains files_struct pointer        |
| FD table      | files_struct               | maps integer FD → struct file        |
| Open file     | struct file                | file offset, flags, pointer to inode |
| File metadata | struct inode               | inode info, block pointers           |
| Filesystem    | super_block, dentry        | maps inode → blocks                  |
| Memory cache  | address_space, struct page | page cache                           |
| Block layer   | request_queue, bio         | queues block I/O                     |
| Driver        | struct block_device        | interfaces hardware                  |
| Hardware      | disk controller, disk      | reads/writes physical blocks         |

---

# **1. What is RAID?**

**RAID** = **Redundant Array of Independent/Inexpensive Disks**.

Purpose:

1. **Redundancy** → survive disk failures
2. **Performance** → faster reads/writes
3. **Capacity aggregation** → multiple disks appear as one

Think of RAID as **a layer that sits above physical disks and presents them as a single logical device**. The key is to see RAID as a layer between the filesystem/block layer and the physical disks.

---

# **2. RAID Levels**

| RAID Level  | How it works                                      | Pros                            | Cons                            |
| ----------- | ------------------------------------------------- | ------------------------------- | ------------------------------- |
| **RAID 0**  | Striping (data split across disks)                | High performance, full capacity | No redundancy                   |
| **RAID 1**  | Mirroring (data duplicated)                       | Redundancy, easy recovery       | 50% capacity efficiency         |
| **RAID 5**  | Striping + parity (1 block for parity per stripe) | Redundancy + performance        | Slow writes, 1 disk can fail    |
| **RAID 6**  | Striping + double parity                          | Can survive 2 disks failing     | Slower writes, storage overhead |
| **RAID 10** | Mirrored stripes (RAID 1+0)                       | High performance + redundancy   | 50% capacity efficiency         |

**Conceptual diagram of RAID 5**:

```
Disk1  Disk2  Disk3  Disk4
A1     B1     C1     P1
A2     B2     P2     C2
A3     P3     C3     B3
```

* `A1, B1, C1` = data blocks
* `P1` = parity block
* Parity allows recovery if any one disk fails.

---

# **3. RAID in Linux**

Linux has two main RAID approaches:

### 3.1 Software RAID (`mdadm`)

* Kernel manages RAID via **MD (Multiple Device) driver**
* `/dev/md0`, `/dev/md1` → RAID devices
* Kernel structures: `struct mddev` → RAID metadata, state
* Supports RAID 0, 1, 5, 6, 10
* Disk failures handled by kernel and `md` subsystem
* Example commands:

```bash
# create RAID 5 on 3 disks
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# check status
cat /proc/mdstat
```

### 3.2 Hardware RAID

* RAID controller card manages redundancy and striping
* OS sees **one logical disk**
* Advantages: offloads computation (parity)
* Kernel sees it as `/dev/sdX`
* Can use vendor tools for monitoring (MegaRAID, HP Smart Array, Dell PERC)

---

# **4. What are RAID Groups?**

A **RAID group** is **a set of physical disks configured together in a RAID level**.

* It forms **one logical unit** exposed to Linux as a **block device**.
* For example:

```text
Physical Disks: sdb, sdc, sdd, sde
RAID Group (RAID 5) → /dev/md0
```

* If a storage array has multiple RAID groups, each RAID group → separate logical volume.
* In enterprise storage, **RAID group** is the base for LUNs (Logical Unit Numbers).

---

# **5. RAID in the Kernel Stack**

**Flow of storage I/O in RAID context:**

```text
Application read/write
       │
VFS / filesystem
       │
Page cache (struct page / address_space)
       │
Block layer → request_queue
       │
Device mapper (RAID software) / MD driver
       │
RAID group → physical disks
```

* Kernel handles **striping, mirroring, parity computation** for software RAID
* Hardware RAID offloads these to the RAID card

---

# **6. RAID Group Properties to Know**

| Property       | Explanation                                                 |
| -------------- | ----------------------------------------------------------- |
| RAID level     | 0,1,5,6,10 determines redundancy and performance            |
| Member disks   | Physical disks in the group                                 |
| Stripe size    | Size of data chunk on each disk (important for performance) |
| Parity         | Calculated for redundancy (RAID 5/6)                        |
| Logical device | What Linux sees (e.g., `/dev/md0`)                          |
| Hot spare      | Disk that replaces failed disk automatically                |

---

# **7. Why RAID Groups Matter in Linux Storage**

* **Performance tuning**:

  * Stripe size should match workload
  * Random vs sequential access performance differs
* **Redundancy planning**:

  * RAID 1 vs RAID 5/6 for tolerance
* **Disk failures / recovery**:

  * Kernel rebuilds RAID using parity or mirrored copies
* **Integration with LVM / filesystems**:

  * RAID groups → LVM → filesystem → VFS → processes

---

# **8. Storage Hierarchy Including RAID**

```
Application
   │
VFS / filesystem
   │
Page cache
   │
Block layer
   │
Device mapper / md driver
   │
RAID group (logical device)
   │
Member disks (physical disks)
```

* RAID group abstracts multiple disks
* Kernel sees **one logical device** from RAID
* Provides **redundancy and/or performance**

---

# **9. Practical Example in Linux**

```bash
# Check RAID devices
cat /proc/mdstat

# Example output
md0 : active raid5 sdb[0] sdc[1] sdd[2]
      1953384 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
```

* md0 → RAID5 logical device
* [UUU] → all 3 disks healthy
* Chunk size → 512 KB stripe per disk

---

# Full Flow with commands

```text
Physical disks → RAID → LVM → Filesystem → Mount
```

Example disks:

```text
/dev/sdb
/dev/sdc
/dev/sdd
/dev/sde
```

---

# 1. Verify Disks Exist

First check available disks.

```bash
lsblk
```

Example output:

```text
sdb   100G
sdc   100G
sdd   100G
sde   100G
```

Another useful command:

```bash
fdisk -l
```

---

# 2. Create RAID Array

Linux software RAID is created using the **mdadm tool**.

Example: create RAID5 from 4 disks.

```bash
sudo mdadm --create --verbose /dev/md0 \
  --level=5 \
  --raid-devices=4 \
  /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

Check RAID status:

```bash
cat /proc/mdstat
```

Example output:

```text
md0 : active raid5 sdb[0] sdc[1] sdd[2] sde[3]
```

Now Linux exposes:

```text
/dev/md0
```

This is a **logical RAID disk**.

---

# 3. Create LVM Physical Volume

Now convert the RAID device into an LVM physical volume.

Using **LVM**.

```bash
sudo pvcreate /dev/md0
```

Verify:

```bash
pvs
```

Output:

```text
PV        VG   Fmt  Attr
/dev/md0       lvm2
```

---

# 4. Create Volume Group

A volume group pools storage.

```bash
sudo vgcreate vg_data /dev/md0
```

Check:

```bash
vgs
```

Example output:

```text
VG      #PV  #LV  VSize
vg_data  1    0   1.5T
```

---

# 5. Create Logical Volumes

Now split the storage into logical volumes.

Example:

```bash
sudo lvcreate -L 200G -n lv_logs vg_data
```

```bash
sudo lvcreate -L 500G -n lv_db vg_data
```

```bash
sudo lvcreate -l 100%FREE -n lv_data vg_data
```

Check logical volumes:

```bash
lvs
```

Example output:

```text
LV       VG       Size
lv_logs  vg_data  200G
lv_db    vg_data  500G
lv_data  vg_data  800G
```

Devices created:

```text
/dev/vg_data/lv_logs
/dev/vg_data/lv_db
/dev/vg_data/lv_data
```

---

# 6. Create Filesystems

Now create filesystems on logical volumes.

Example EXT4:

```bash
sudo mkfs.ext4 /dev/vg_data/lv_logs
```

Example XFS:

```bash
sudo mkfs.xfs /dev/vg_data/lv_db
```

---

# 7. Create Mount Points

Create directories where filesystems will appear.

```bash
sudo mkdir /logs
sudo mkdir /database
```

---

# 8. Mount the Filesystems

```bash
sudo mount /dev/vg_data/lv_logs /logs
```

```bash
sudo mount /dev/vg_data/lv_db /database
```

Verify:

```bash
df -h
```

Example:

```text
Filesystem                Size  Mounted on
/dev/vg_data/lv_logs      200G  /logs
/dev/vg_data/lv_db        500G  /database
```

---

# 9. Make Mount Permanent

Edit:

```bash
sudo vi /etc/fstab
```

Add entries:

```text
/dev/vg_data/lv_logs   /logs      ext4   defaults  0 0
/dev/vg_data/lv_db     /database  xfs    defaults  0 0
```

---

# 10. Final Storage Stack

Your system now looks like this:

```text
Applications
      ↓
Filesystem (ext4 / xfs)
      ↓
Logical Volumes
/dev/vg_data/lv_logs
/dev/vg_data/lv_db
      ↓
Volume Group
vg_data
      ↓
Physical Volume
/dev/md0
      ↓
RAID5 Array
      ↓
Physical Disks
/dev/sdb /dev/sdc /dev/sdd /dev/sde
```

---

# 11. Commands to Inspect Each Layer

These are extremely useful in production troubleshooting.

| Layer            | Command                   |
| ---------------- | ------------------------- |
| disks            | `lsblk`                   |
| RAID status      | `cat /proc/mdstat`        |
| RAID details     | `mdadm --detail /dev/md0` |
| physical volumes | `pvs`                     |
| volume groups    | `vgs`                     |
| logical volumes  | `lvs`                     |
| filesystems      | `df -h`                   |
| mounts           | `mount`                   |

---

# 12. Interview Insight

This layered architecture exists because each layer solves a different problem.

| Layer      | Purpose                       |
| ---------- | ----------------------------- |
| RAID       | redundancy + performance      |
| LVM        | flexible storage allocation   |
| filesystem | file storage structure        |
| VFS        | unified kernel file interface |

---

✅ **Interview explanation (short version)**

> In Linux, multiple disks can be combined using mdadm to create a RAID array, which appears as a block device like `/dev/md0`. This device is often used as an LVM physical volume, from which volume groups and logical volumes are created. Filesystems such as ext4 or XFS are then created on the logical volumes and mounted to directories for application use.

---

If you'd like, I can also show you **how large distributed storage systems (NetApp, Ceph, Dell EMC) implement the same concepts internally**, which is **very relevant to the “distributed storage systems” requirement in your interview.**



```
Physical disks
/dev/sda
/dev/sdb
/dev/sdc
/dev/sdd
        ↓
RAID5 array
        ↓
/dev/md0
        ↓
LVM Physical Volume
        ↓
Volume Group
        ↓
Logical Volumes
/dev/vg_data/lv_logs
/dev/vg_data/lv_db
        ↓
Filesystem
XFS / EXT4
        ↓
Mounted directories
/logs
/db
```

Applications interact with:

```
/logs
/db
```
