# Simple Operating System Simulation — README

**Course:** CO2018 — Operating Systems | HCMC University of Technology

---

## Overview

This project simulates core OS components:
- **Scheduler** — Multi-Level Queue (MLQ) scheduling
- **Memory Management** — Paging-based virtual/physical memory
- **System Calls** — Kernel interface via syscall table

---

## Build

```bash
make all
```

---

## How to Run

```bash
./os <config_file>
```

Where `<config_file>` is a name in the `input/` directory.

### Config file format (without MM_PAGING)
```
[time_slice] [num_CPUs] [num_processes]
[start_time_0] [proc_path_0] [priority_0]
[start_time_1] [proc_path_1] [priority_1]
...
```

### Config file format (with MM_PAGING, `MM_FIXED_MEMSZ` disabled)
```
[time_slice] [num_CPUs] [num_processes]
[KERNEL_BASE_ADDR]
[RAM_SZ] [SWP_SZ_0] [SWP_SZ_1] [SWP_SZ_2] [SWP_SZ_3]
[start_time] [proc_path] [priority]
...
```

---

## Test Cases

### 1. Scheduler (MLQ)

| Command | Description |
|---|---|
| `./os sched_0` | 1 CPU, 4 processes — verify priority ordering (prio=0 before prio=4), round-robin within same priority |
| `./os sched_1` | 1 CPU, 5 processes — verify prio=0 → prio=1 → ... → prio=4 dispatch order |
| `./os sched` | 2 CPUs, 4 processes — verify parallel independent dispatch per CPU |

**Expected behavior:**
- Lower `prio` value = higher priority → dispatched first.
- Same-priority processes alternate in round-robin.
- Each priority level gets `slot = MAX_PRIO - prio` slots before moving to the next queue.

---

### 2. MLQ Paging — Single CPU

| Command | Description |
|---|---|
| `./os os_1_singleCPU_mlq_paging` | Basic paging: verify `liballoc:178` + `print_pgtbl` on ALLOC, `libwrite:502` + `print_pgtbl` on WRITE |
| `./os os_1_mlq_paging_small_1K` | RAM = 1K — forces page swap; program must not crash |
| `./os os_1_mlq_paging_small_4K` | RAM = 4K — similar swap test, slightly larger RAM |
| `./os os_2_singleCPU_mlq_paging` | 2 processes sharing RAM — verify no data corruption between processes |

---

### 3. MLQ Paging — Multi CPU

| Command | Description |
|---|---|
| `./os os_1_mlq_paging` | 4 CPUs — concurrent dispatch, no deadlock, mutex correct |
| `./os os_2_mlq_paging` | 4 CPUs, more processes |
| `./os os_0_mlq_paging` | 4 CPUs, no paging |

---

### 4. System Calls

| Command | Expected Output |
|---|---|
| `./os os_syscall_list` | Must include `0-sys_listsyscall` and `17-sys_memmap` |
| `./os os_syscall` | `liballoc:178` and `libwrite:502` interleaved with time slots |
| `./os os_sc` | Your custom syscall output (e.g., `The first system call parameter 1`) |

---

## Source Structure

```
include/
  common.h       # PCB, kernel structs
  os-cfg.h       # Feature toggles: MLQ_SCHED, MM_PAGING, MM_FIXED_MEMSZ, MM64
  os-mm.h        # VM/paging structs: mm_struct, vm_area_struct, memphy_struct
  sched.h        # Scheduler interface
  syscall.h      # Syscall headers
src/
  sched.c        # TODO: enqueue(), dequeue(), get_proc()
  queue.c        # TODO: queue operations
  mm.c           # Memory management
  mm-vm.c        # Virtual memory ops
  mm-memphy.c    # Physical memory (RAM + SWAP)
  mm64.c         # 64-bit paging support
  libmem.c       # ALLOC/FREE/READ/WRITE library (liballoc, libwrite)
  syscall.c      # Syscall dispatch
  syscall.tbl    # Syscall table (index, name, handler)
  sys_*.c        # Individual syscall handlers
input/           # Config files and process programs
output/          # Sample expected outputs
```

---

## Key Configurations (`include/os-cfg.h`)

```c
#define MLQ_SCHED 1      // Enable MLQ scheduler
#define MAX_PRIO  140    // Priority levels (0 = highest)

#define MM_PAGING        // Enable paging memory management
#define MM_FIXED_MEMSZ   // Use fixed memory sizes (comment out for custom sizes)
// #define MM64          // Enable 64-bit 5-level paging
```

---

## MLQ Policy Summary

- `MAX_PRIO = 140`, priorities 0..139
- Each priority queue gets **`slot = MAX_PRIO - prio`** CPU slots per round
- `get_proc()` traverses queues from prio=0 upward, consuming slots
- Processes return to their original priority queue after their time slice

---

## Process File Format (`input/proc/`)

```
[priority] [N]
instruction_0
instruction_1
...
instruction_N-1
```

**Instructions:** `calc`, `alloc [size] [reg]`, `free [reg]`, `read [src] [offset] [dst]`, `write [data] [dst] [offset]`, `kmalloc`, `kmem_cache_create`, `kmem_cache_alloc`, `copy_from_user`, `copy_to_user`

---

## Adding a System Call

1. Create `src/sys_xxxhandler.c`
2. Add to `Makefile`: `SYSCALL_OBJ += $(OBJ)/sys_xxxhandler.o`
3. Register in `src/syscall.tbl`: `440  xxx  sys_xxxhandler`
4. `make all` → `./os os_sc`

---

## Notes

- Multiple correct outputs exist due to concurrent loader + scheduler.
- Compare with `output/` samples — your result need not match exactly, but must be **explainable** by the theory.
- `run_queue` is obsolete — ignore it.
- User space ↔ kernel space data transfer must use `copy_from_user` / `copy_to_user`.
- Process PCB must **not** be passed directly outside kernel mode — access via `krnl_t` using PID.
