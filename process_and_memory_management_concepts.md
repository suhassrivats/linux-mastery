# Process and Memory Management in Linux

---

## Overview

This document illustrates how Linux organizes processes and their memory, from the user-space view through the kernel's internal structures.

---

## 1. User Space — Process View

```text
   Running Program: cat file.txt

   Virtual Address Space
   ┌──────────────────────────────────────┐
   │ Stack  (function frames, locals)     │
   ├──────────────────────────────────────┤
   │ mmap region / shared libraries       │
   ├──────────────────────────────────────┤
   │ Heap   (malloc/new allocations)      │
   ├──────────────────────────────────────┤
   │ Data + BSS (globals/statics)         │
   ├──────────────────────────────────────┤
   │ Code  (machine instructions)         │
   └──────────────────────────────────────┘

                │
                │ System calls (read, write, open, mmap, ...)
                ▼
```

---

## 2. Kernel Space — Process Representation

### Process Descriptor (`task_struct`)

```text
┌─────────────────────────────────────────────────────┐
│  task_struct  — PROCESS DESCRIPTOR                  │
├─────────────────────────────────────────────────────┤
│  • PID, state, CPU registers                        │
│  • scheduling info                                  │
│  • credentials (UID/GID)                            │
│  • parent/child links                               │
│  • pointers → mm_struct, files_struct, signals      │
└─────────────────────────────────────────────────────┘
         │                              │
         │                              │
         ▼                              ▼
```

---

## 3. Memory Side vs. Files Side

```text
══════════ MEMORY SIDE ════════════         ══════════ FILES SIDE ══════════

mm_struct (process address space)           files_struct (open files)
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│  Page table root (CR3 / PGD)    │         │  File Descriptor Table           │
│  List of VMAs (vm_area_struct)  │         ├─────────────────────────────────┤
└─────────────────────────────────┘         │  0 → stdin                      │
         │                                   │  1 → stdout                     │
         ▼                                   │  2 → stderr                     │
vm_area_struct (Virtual Memory Area)        │  3 → file.txt                   │
┌─────────────────────────────────┐         │  4 → socket / pipe / etc        │
│  start / end addresses          │         └─────────────────────────────────┘
│  permissions (R/W/X)            │                    │
│  backing (file or anonymous)   │                    ▼
└─────────────────────────────────┘         struct file (OPEN FILE OBJECT)
         │                                   ┌─────────────────────────────────┐
         ▼                                   │  current file offset            │
Page Tables (PGD → PUD → PMD → PTE)          │  access mode & flags            │
┌─────────────────────────────────┐         │  pointer to inode               │
│  Maps virtual pages → physical  │         └─────────────────────────────────┘
│  4 KB pages                     │                    │
└─────────────────────────────────┘                    ▼
         │                                   inode (ACTUAL FILE METADATA)
         ▼                                   ┌─────────────────────────────────┐
Physical Memory (RAM)                        │  owner / permissions            │
┌─────────────────────────────────┐         │  timestamps                     │
│  Stack pages (function frames)   │         │  file size                      │
│  Heap pages (malloc/new)        │         │  disk block pointers            │
│  File-backed pages (mmap)       │         └─────────────────────────────────┘
│  Code pages (instructions)      │                    │
└─────────────────────────────────┘                    ▼
         │                                   Disk Blocks (actual file contents)
         ▼
Swap / Disk (if page swapped)
┌─────────────────────────────────┐
│  Pages moved out to swap         │
│  when physical RAM is low        │
└─────────────────────────────────┘
```

---

## 4. Pages, Frames & Page Tables — How Mapping Works

### Key Terms

| Term | Meaning |
| ---- | ------- |
| **Page** | A fixed-size chunk of *virtual* memory. On x86-64 Linux, typically **4 KB** (12 bits). Virtual address space is divided into contiguous pages. |
| **Frame** | A fixed-size chunk of *physical* RAM. Same size as a page (4 KB). Frames are what actually hold data in hardware. |
| **Page frame** | Another name for "frame" — the physical container in RAM. |
| **Page table** | A data structure that maps *virtual pages* → *physical frames*. The CPU's Memory Management Unit (MMU) uses it to translate addresses. |

### Why Paging?

- **Virtual address space** lets each process see a large, private address range (e.g. 48 bits on x86-64) without needing that much physical RAM.
- **Isolation**: Process A cannot access Process B's memory because each has its own page tables.
- **Efficiency**: Only used pages need physical frames; the rest can stay on disk (swap) or not be allocated yet.

### How the Mapping Happens

```text
VIRTUAL ADDRESS (64 bits on x86-64, 48 usable)
┌──────────┬──────────┬──────────┬──────────┬─────────────┐
│   PGD    │   PUD    │   PMD    │   PTE    │   Offset    │
│  (9 b)   │  (9 b)   │  (9 b)   │  (9 b)   │   (12 b)   │
└──────────┴──────────┴──────────┴──────────┴─────────────┘
     │          │          │          │           │
     │          │          │          │           └── Byte offset within the 4 KB page (0–4095)
     │          │          │          │
     │          │          │          └── Index into Page Table → PTE holds: frame number, flags (R/W/X, present)
     │          │          │
     │          │          └── Index into Page Middle Directory
     │          │
     │          └── Index into Page Upper Directory
     │
     └── Index into Page Global Directory (top level)
```

1. **CPU issues a virtual address** (e.g. when loading from `0x7fff12345678`).
2. **MMU splits the address** into indices (PGD, PUD, PMD, PTE) and offset.
3. **MMU walks the page tables** (each level points to the next table or to a page):
   - Start at **CR3** (holds PGD base for current process).
   - Use PGD index → get PUD table.
   - Use PUD index → get PMD table.
   - Use PMD index → get PTE table.
   - Use PTE index → get **Physical Frame Number (PFN)** and flags.
4. **Physical address** = `(PFN × 4096) + offset`.
5. **MMU checks flags**: present? permissions (R/W/X) match the access? If not → page fault.

### What’s in a PTE (Page Table Entry)?

```text
┌─────────────────────────────────────────────────────────┐
│  Present | R/W | U/S | PWT | PCD | A | D | ... | PFN    │
└─────────────────────────────────────────────────────────┘
     │       │     │
     │       │     └── User/Supervisor (kernel vs user access)
     │       └── Read/Write
     └── 1 = page in RAM, 0 = not present (page fault: load from swap or allocate)
```

- **Present bit = 0**: Page fault. Kernel may load from swap, map from a file, or allocate a new frame.
- **R/W/X**: Read, Write, Execute. Enforced by the MMU (e.g. no execute in data regions).

### Page Faults

When the MMU cannot complete a translation (e.g. present = 0, or permission violation):

1. CPU traps into the kernel.
2. Kernel inspects the faulting address and process state.
3. **Valid fault**: load page from swap, copy-on-write, demand paging, etc.
4. **Invalid fault**: send SIGSEGV to the process (segmentation fault).

### TLBs (Translation Lookaside Buffers)

Walking 4 levels of page tables for every access would be slow. The CPU caches recent virtual→physical translations in a **TLB**. On TLB miss, the MMU does a full page-table walk and then fills the TLB.

---

## Summary

| Component          | Purpose                                           |
| ------------------ | ------------------------------------------------- |
| `task_struct`      | Process descriptor; central process state         |
| `mm_struct`        | Virtual address space, page tables, VMAs          |
| `vm_area_struct`   | Maps virtual regions with permissions             |
| Page               | 4 KB chunk of virtual address space               |
| Frame              | 4 KB chunk of physical RAM                        |
| PTE                | Maps one virtual page → one physical frame        |
| `files_struct`     | Open file descriptor table                        |
| `struct file`      | Per-process view of an open file                  |
| `inode`            | On-disk file metadata                             |
