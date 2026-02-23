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
══════════════════ MEMORY SIDE ══════════════════    ═══════════ FILES SIDE ═══════════
```

### Memory Management (`mm_struct`)

```text
mm_struct (process address space)
┌───────────────────────────────────────────────────┐
│  Page table root (CR3 / PGD)                      │
│  List of VMAs (vm_area_struct)                    │
└───────────────────────────────────────────────────┘
         │
         ▼
vm_area_struct (Virtual Memory Area)
┌───────────────────────────────────┐
│  start / end addresses            │
│  permissions (R/W/X)              │
│  backing (file or anonymous)      │
└───────────────────────────────────┘
         │
         ▼
Page Tables (PGD → PUD → PMD → PTE)
┌───────────────────────────────────┐
│  Maps virtual pages → physical    │
│  4 KB pages                       │
└───────────────────────────────────┘
         │
         ▼
Physical Memory (RAM)
┌───────────────────────────────────┐
│  Stack pages (function frames)    │
│  Heap pages (malloc/new)         │
│  File-backed pages (mmap)        │
│  Code pages (instructions)       │
└───────────────────────────────────┘
         │
         ▼
Swap / Disk (if page swapped)
┌───────────────────────────────────┐
│  Pages moved out to swap          │
│  when physical RAM is low        │
└───────────────────────────────────┘
```

### File Management (`files_struct`)

```text
files_struct (open files)
┌─────────────────────────────────────────┐
│  File Descriptor Table                  │
├─────────────────────────────────────────┤
│  0 → stdin                             │
│  1 → stdout                            │
│  2 → stderr                            │
│  3 → file.txt                          │
│  4 → socket / pipe / etc                │
└─────────────────────────────────────────┘
         │
         ▼
struct file (OPEN FILE OBJECT)
┌─────────────────────────────────────────┐
│  current file offset                    │
│  access mode & flags                    │
│  pointer to inode                       │
└─────────────────────────────────────────┘
         │
         ▼
inode (ACTUAL FILE METADATA)
┌─────────────────────────────────────────┐
│  owner / permissions                   │
│  timestamps                            │
│  file size                             │
│  disk block pointers                   │
└─────────────────────────────────────────┘
         │
         ▼
Disk Blocks (actual file contents)
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
