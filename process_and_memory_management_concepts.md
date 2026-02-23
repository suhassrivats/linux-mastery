# Process and Memory Management in Linux

```text
USER SPACE (Process View)
=========================

   Running Program: cat file.txt

   Virtual Address Space
   +--------------------------------------+
   | Stack  (function frames, locals)     |
   +--------------------------------------+
   | mmap region / shared libraries       |
   +--------------------------------------+
   | Heap   (malloc/new allocations)      |
   +--------------------------------------+
   | Data + BSS (globals/statics)         |
   +--------------------------------------+
   | Code  (machine instructions)         |
   +--------------------------------------+

                |
                | System calls (read, write, open, mmap, ...)
                v


KERNEL SPACE (Process Representation)
=====================================

task_struct  ← PROCESS DESCRIPTOR
+------------------------------------------------------+
| PID, state, CPU registers                           |
| scheduling info                                     |
| credentials (UID/GID)                               |
| parent/child links                                  |
|                                                      |
| pointers → mm_struct, files_struct, signals, ...    |
+------------------------------------------------------+
         |                                  |
         |                                  |
         v                                  v

================ MEMORY SIDE =================    ============== FILES SIDE ==============

mm_struct (process address space)             files_struct (open files)
+---------------------------------------------------+   +-----------------------------+
| Page table root (CR3 / PGD)                      |   | File Descriptor Table       |
| List of VMAs (vm_area_struct)                    |   |-----------------------------|
+---------------------------------------------------+   | 0 → stdin                   |
          |                                           | 1 → stdout                  |
          v                                           | 2 → stderr                  |
    vm_area_struct (Virtual Memory Area)            | 3 → file.txt                |
    +-------------------------------+               | 4 → socket / pipe / etc     |
    | start / end addresses         |               +-----------------------------+
    | permissions (R/W/X)           |                        |
    | backing (file or anonymous)   |                        v
    +-------------------------------+                  struct file
          |                                           (OPEN FILE OBJECT)
          v                                           +--------------------------+
     Page Tables (PGD → PUD → PMD → PTE)             | current file offset      |
     +-------------------------------+               | access mode & flags      |
     | Maps virtual pages → physical  |               | pointer to inode         |
     | 4 KB pages                     |               +--------------------------+
     +-------------------------------+                        |
          |                                                 v
          v                                             inode
    Physical Memory (RAM)                            (ACTUAL FILE METADATA)
    +-------------------------------+               +--------------------------+
    | Stack pages (function frames)  |               | owner / permissions      |
    | Heap pages (malloc/new)       |               | timestamps               |
    | File-backed pages (mmap)      |               | file size                |
    | Code pages (instructions)     |               | disk block pointers      |
    +-------------------------------+               +--------------------------+
          |                                                 |
          v                                                 v
     Swap / Disk (if page swapped)                       Disk Blocks
     +-------------------------------+               (actual file contents)
     | Pages moved out to swap       |
     | if physical RAM is low        |
     +-------------------------------+
```

