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

## Summary

| Component          | Purpose                                           |
| ------------------ | ------------------------------------------------- |
| `task_struct`      | Process descriptor; central process state         |
| `mm_struct`        | Virtual address space, page tables, VMAs          |
| `vm_area_struct`   | Maps virtual regions with permissions             |
| `files_struct`     | Open file descriptor table                        |
| `struct file`      | Per-process view of an open file                  |
| `inode`            | On-disk file metadata                             |
