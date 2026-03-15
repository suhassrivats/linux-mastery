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

Perfect, Suhas — now we’re diving into **RAID and storage grouping**, which is crucial for **high-availability and performance in Linux storage systems**. I’ll explain this from **concept → Linux implementation → kernel interaction → production relevance**.

---

# **1. What is RAID?**

**RAID** = **Redundant Array of Independent/Inexpensive Disks**.

Purpose:

1. **Redundancy** → survive disk failures
2. **Performance** → faster reads/writes
3. **Capacity aggregation** → multiple disks appear as one

Think of RAID as **a layer that sits above physical disks and presents them as a single logical device**.

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
