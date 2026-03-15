Understanding **how BIOS/UEFI detects storage devices** requires looking at **what happens when a computer powers on**. I’ll walk through the process from **power button → hardware detection → kernel taking over**.

---

# 1. What Happens When You Press the Power Button

When you power on a computer, the CPU cannot immediately run Linux because:

* RAM is empty
* Devices are not initialized
* The kernel is not loaded

So the system starts with **firmware**.

Two types of firmware:

| Firmware | Description                           |
| -------- | ------------------------------------- |
| BIOS     | older firmware used in legacy systems |
| UEFI     | modern firmware replacing BIOS        |

Firmware is stored in **non-volatile memory on the motherboard**.

Boot flow:

```text
Power ON
   ↓
CPU starts executing firmware
   ↓
BIOS / UEFI initializes hardware
   ↓
Detect storage devices
   ↓
Find bootloader
   ↓
Load Linux kernel
```

---

# 2. CPU Startup (Important)

When the CPU starts, it executes code from a **fixed memory address**.

Example:

```text
x86 reset vector → FFFF0
```

This address contains a jump to the firmware code.

So:

```text
CPU
 ↓
BIOS / UEFI firmware starts
```

---

# 3. POST (Power-On Self Test)

The firmware runs **POST**, which checks basic hardware.

Checks include:

| Hardware            | What is checked        |
| ------------------- | ---------------------- |
| CPU                 | registers working      |
| RAM                 | memory test            |
| GPU                 | display initialization |
| Keyboard            | controller check       |
| Storage controllers | SATA / NVMe detection  |

If something fails, the system may produce **beep codes**.

---

# 4. Detecting Storage Devices

Now firmware scans **hardware buses**.

Remember earlier:

| Bus  | Devices         |
| ---- | --------------- |
| SATA | HDD / SATA SSD  |
| PCIe | NVMe SSD        |
| USB  | external drives |

Firmware queries **controllers attached to these buses**.

---

# 5. SATA Device Detection

For SATA drives, detection happens through the **SATA controller**.

Flow:

```text
BIOS / UEFI
     ↓
SATA controller
     ↓
Scan SATA ports
     ↓
Ask each port if a device is attached
```

Each SATA device responds with **IDENTIFY command**.

Device returns information such as:

* model number
* capacity
* serial number
* supported features

Example info returned:

```text
Model: Samsung SSD
Size: 1TB
Sector size: 512 bytes
```

Firmware stores this information.

---

# 6. NVMe Device Detection (PCIe)

NVMe devices are detected differently.

Because NVMe devices connect through **PCIe bus**.

Flow:

```text
BIOS / UEFI
     ↓
PCIe root complex
     ↓
Scan PCIe bus
     ↓
Enumerate devices
```

PCIe devices have **configuration space** containing identifiers.

Example fields:

| Field      | Meaning             |
| ---------- | ------------------- |
| Vendor ID  | device manufacturer |
| Device ID  | specific hardware   |
| Class code | type of device      |

Example NVMe device:

```text
Vendor ID: 144d (Samsung)
Device class: NVMe storage controller
```

Firmware recognizes it as an **NVMe storage controller**.

---

# 7. PCIe Device Enumeration

PCIe detection is called **enumeration**.

Steps:

```text
Firmware scans PCIe bus
      ↓
Each device responds with ID
      ↓
Firmware assigns resources
      ↓
Device becomes usable
```

Resources assigned include:

| Resource        | Purpose              |
| --------------- | -------------------- |
| Memory address  | device registers     |
| Interrupt lines | device notifications |
| DMA mapping     | data transfer        |

---

# 8. Firmware Creates Boot Device List

After detecting disks, firmware creates a list of **bootable devices**.

Example boot order:

```text
1. NVMe SSD
2. SATA SSD
3. USB drive
4. Network boot
```

Firmware looks for a **bootloader**.

---

# 9. Bootloader Discovery

Depending on firmware type:

### Legacy BIOS

BIOS reads:

```text
first 512 bytes of disk
```

called the **MBR (Master Boot Record)**.

MBR contains:

* partition table
* bootloader code

---

### UEFI

UEFI looks for:

```text
EFI System Partition
```

Inside:

```text
/EFI/BOOT/bootx64.efi
```

This is the **bootloader executable**.

Example bootloaders:

* GRUB
* systemd-boot

---

# 10. Bootloader Loads Linux Kernel

Bootloader then loads:

```text
Linux kernel
initramfs
```

into RAM.

Kernel starts executing.

---

# 11. Linux Kernel Detects Devices Again

Even though firmware detected devices, **Linux does its own detection**.

Kernel loads drivers such as:

| Driver      | Device         |
| ----------- | -------------- |
| AHCI        | SATA devices   |
| NVMe        | NVMe SSDs      |
| USB-storage | external disks |

Kernel scans hardware again.

---

# 12. Kernel Creates Block Devices

Once the driver detects the disk, Linux creates:

```text
/dev/sda
/dev/sdb
/dev/nvme0n1
```

Example mapping:

```text
NVMe SSD → /dev/nvme0n1
SATA SSD → /dev/sda
```

These represent **block devices**.

---

# 13. Kernel Structures Created

The kernel builds structures like:

| Structure              | Purpose                |
| ---------------------- | ---------------------- |
| `struct gendisk`       | represents the disk    |
| `struct block_device`  | device abstraction     |
| `struct request_queue` | I/O request scheduling |

Now the disk is usable by Linux.

---

# 14. Final Flow (Complete)

Complete detection path:

```text
Power button
   ↓
CPU reset vector
   ↓
BIOS / UEFI firmware
   ↓
POST hardware checks
   ↓
Bus scanning
   ↓
SATA / PCIe device detection
   ↓
Bootloader located
   ↓
Linux kernel loaded
   ↓
Kernel drivers detect devices
   ↓
Block devices created (/dev/sdX)
```

---

# 15. Simple Analogy

Think of the firmware as a **hotel receptionist**.

```text
Computer starts
   ↓
Receptionist checks all rooms
   ↓
Registers devices
   ↓
Hands control to operating system
```

Then Linux **manages the devices** during runtime.

---

✅ **One-line summary**

> BIOS/UEFI detects storage devices by scanning hardware buses like SATA and PCIe during boot, identifying connected devices through controller queries or PCIe enumeration, and then passing this information to the operating system which loads drivers and creates block devices.
