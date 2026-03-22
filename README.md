# CosmicWave — Bare-Metal x86-64 Kernel in Rust

[![Language](https://img.shields.io/badge/language-Rust%20(no__std)-orange)](https://www.rust-lang.org)
[![Architecture](https://img.shields.io/badge/arch-x86--64-blue)](https://en.wikipedia.org/wiki/X86-64)
[![Version](https://img.shields.io/badge/version-v8-green)](#version-history)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](#license)

CosmicWave is a research and educational operating system kernel written from scratch in `no_std` Rust, targeting bare-metal x86-64. It is designed to be read and understood in its entirety — every subsystem is implemented without external OS dependencies, documented at the design level, and progressively hardened across a series of well-defined versions.

---

## Table of Contents

1. [What CosmicWave Is](#what-cosmicwave-is)
2. [Quick Start](#quick-start)
3. [Architecture Overview](#architecture-overview)
4. [What Has Been Achieved](#what-has-been-achieved)
   - [v1–v5: Foundation](#v1v5-foundation)
   - [v6: Threading and VFS](#v6-threading-and-vfs)
   - [v6 (fixed): Correctness Passes](#v6-fixed-correctness-passes)
   - [v7: Hardening](#v7-hardening)
   - [v8: IST, Phased Scheduler, Preemption Control](#v8-ist-phased-scheduler-preemption-control)
5. [Subsystem Reference](#subsystem-reference)
6. [What Still Needs to Be Done](#what-still-needs-to-be-done)
   - [Near-Term (High Priority)](#near-term-high-priority)
   - [Medium-Term (Infrastructure)](#medium-term-infrastructure)
   - [Long-Term (Full OS)](#long-term-full-os)
7. [Potential Uses](#potential-uses)
8. [Project Layout](#project-layout)
9. [Building and Running](#building-and-running)
10. [Design Philosophy](#design-philosophy)
11. [Known Limitations](#known-limitations)
12. [Version History](#version-history)
13. [License](#license)

---

## What CosmicWave Is

CosmicWave is **not** a production operating system. It is a complete, self-contained implementation of the concepts that underpin real kernels — memory management, scheduling, interrupt handling, virtual memory, filesystems, and system calls — written in a language that makes the unsafe boundaries explicit.

The goals are:

- **Legibility**: every design decision is explained in the source or documentation. A competent systems programmer should be able to read any subsystem cold and understand it within minutes.
- **Correctness over cleverness**: algorithms are chosen for clarity. Where performance matters (e.g. the TLB shootdown ring, the slab allocator), the trade-off is documented.
- **Progressive hardening**: each version fixes a class of real kernel bugs — double-frees, unbounded spin-waits, unvalidated syscall arguments, stack overflows — before adding new features.
- **Fidelity to real hardware**: no emulation shims. CosmicWave runs on QEMU with standard x86-64 hardware virtualization and exercises real LAPIC, ACPI, SMP, and page-table mechanics.

---

## Quick Start

### Prerequisites

- `rustup` with the nightly toolchain
- `qemu-system-x86_64`
- `lld` (comes with LLVM)

### Build and run

```bash
git clone https://github.com/your-org/cosmicwave
cd cosmicwave
chmod +x build.sh
./build.sh
```

The kernel boots, runs its built-in test suite, and prints results to the serial port (mapped to stdout by QEMU).

Expected output:
```
🌌 CosmicWave v8 — full IST, phased scheduler, preemption control
🔍 CPU discovery:
 ACPI: 4 CPUs
🔥 SMP bringup
 AP 1 online LAPIC 1
 AP 2 online LAPIC 2
 AP 3 online LAPIC 3
✅ SMP 4 CPUs
[BSP] rsp0=0xffff8000...
[BSP] IST1=0xffff8000...  [BSP] IST2=...  ...  [BSP] IST7=...
kernel .text write-protected [0xffff800000100000, 0xffff8000001a0000)
ramfs: /dev/null, /dev/zero, /etc/hostname registered
kthread 'test-runner' PID 1
--- Running 11 tests ---
[PASS] ramfs/open+read
[PASS] devnull+devzero
[PASS] pipe/read-write
[PASS] preempt-count/disable-enable
[PASS] spinlock/preempt-disable
[PASS] ist/all-slots-populated
[PASS] stack-canary/intact
[PASS] kthread/separate-pml4
[PASS] arg-val/read-bad-fd
[PASS] arg-val/write-null-ptr
[PASS] arg-val/mmap-bad-prot
[PASS] arg-val/kill-sig-zero
[PASS] parallel-counter
[PASS] address-space/drop-frees-frames
--- Tests complete: 11 pass, 0 fail ---
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CosmicWave v8                               │
├──────────────┬─────────────────┬──────────────┬─────────────────────┤
│  Physical    │  Virtual Memory │  Interrupts  │     Scheduler       │
│  Memory      │                 │  & Exceptions│                     │
│              │  4-level PTEs   │              │  sched_pick_next()  │
│  Frame alloc │  CoW fork       │  Full IDT    │  sched_steal()      │
│  + refcounts │  SMEP / SMAP    │  20 vectors  │  sched_idle_wait()  │
│  Slab cache  │  NX bit         │              │  perform_ctx_switch │
│  Large bump  │  Guard pages    │  7-slot IST  │  perform_first_run  │
│  + free list │  Write-protect  │  per-vector  │                     │
│              │  kernel .text   │  dedicated   │  Per-CPU run-queues │
│              │  Private kthread│  8 KiB stacks│  Work-stealing      │
│              │  PML4s          │              │  preempt_count      │
├──────────────┴─────────────────┴──────────────┴─────────────────────┤
│                              SMP                                    │
│  ACPI MADT parse · LAPIC init · INIT-SIPI-SIPI bringup             │
│  Per-CPU GDT/TSS · TLB shootdown ring · IPI (TLB/resched/wake)     │
├─────────────────────────────────────────────────────────────────────┤
│                           VFS / RamFs                               │
│  Vnode trait · RegularFile · Pipe (WaitQueue-based blocking)        │
│  DevNull · DevZero · FileTable (shared via CLONE_FILES)             │
│  RamFs: /dev/null, /dev/zero, /etc/hostname                         │
├─────────────────────────────────────────────────────────────────────┤
│                         Syscall Layer                               │
│  int 0x80 dispatch · SyscallArgs validation · errno constants       │
│  read/write/open/close/pipe/dup2/mmap/munmap                        │
│  fork/clone/exit/waitpid/kill/sigaction/sigreturn                   │
├─────────────────────────────────────────────────────────────────────┤
│                        Kernel Hardening                             │
│  Stack canaries (per-task, PRNG-seeded from TSC)                    │
│  Guard pages (1 × 4 KiB unmapped below every kernel stack)          │
│  Write-protected .text after boot                                   │
│  Private PML4 per kernel thread                                     │
│  Signal delivery only from user-mode frames                         │
│  Full IST for all critical exceptions                               │
└─────────────────────────────────────────────────────────────────────┘
```

**Virtual memory map**

| Region | Base address | Notes |
|--------|-------------|-------|
| Kernel image | `0xFFFF_8000_0010_0000` | Loaded at 1 MiB physical |
| Kernel heap | `0xFFFF_8000_4444_0000` | 16 MiB slab + bump |
| Frame pool | `0xFFFF_8000_5444_0000` | Grows upward; 512 MiB hard limit |
| User code (min) | `0x0000_0000_0010_0000` | `USER_VADDR_MIN` |
| User stack top | `0x0000_7FFF_FFFF_F000` | Grows down; 4 pages pre-mapped |

---

## What Has Been Achieved

### v1–v5: Foundation

The earliest versions established the hardware substrate every subsequent feature builds on.

**Boot and paging.** The kernel enters in 64-bit long mode, sets up a 4-level page table that identity-maps the first 512 MiB of physical memory at `KERNEL_VMA` (using 2 MiB huge pages), enables NX via `EFER.NXE`, and activates SMEP and SMAP via CR4. SMEP prevents the kernel from executing user-space code; SMAP prevents accidental kernel reads from user memory without an explicit `clac`/`stac` bracket.

**Physical memory management.** A bump allocator hands out 4 KiB frames from a pool above the heap. A per-frame `AtomicU32` refcount supports copy-on-write sharing. A free list (protected by an IRQ-safe spinlock) recycles frames when their refcount drops to zero, with an `Ordering::Acquire` fence after the last `fetch_sub` to ensure all prior writes to the freed frame are visible.

**Hybrid kernel allocator.** Eight slab caches (16 B – 2 KiB, powers of two) handle small `alloc::` requests. Larger requests go through a bump allocator with a best-fit free list. This is registered as `#[global_allocator]`, enabling `Vec`, `BTreeMap`, `Arc`, and `String` in the kernel.

**IRQ-safe spinlock.** `Spinlock<T>` saves and restores the interrupt flag in RFLAGS on every lock/unlock, preventing deadlock from an interrupt handler preempting a lock holder on the same CPU. v8 adds `preempt_count` management to the same acquire/release path.

**SMP bringup.** ACPI MADT is parsed for LAPIC IDs. A real-mode trampoline is written to physical address `0x8000` and INIT-SIPI-SIPI is sent to each AP. Each AP initialises its own per-CPU GDT/TSS, reloads the IDT, starts its LAPIC timer, and halts. Up to 16 CPUs are supported.

**TLB shootdown ring.** A per-CPU lock-free ring buffer (`TlbRing`, capacity 64 entries) receives `(pml4_phys, va)` pairs. Targeted IPIs (vector `0x42`) tell sibling CPUs to drain their rings. The initiating CPU waits for all ring heads to equal their tails before returning, ensuring full TLB coherence across all CPUs that have the affected address space active.

**IDT and exception handling.** A 256-entry IDT. Two ISR stub macros: `isr_sw!` performs the user→kernel stack switch for ordinary interrupts; `isr_ist!` is used where the CPU switches to an IST stack automatically. The LAPIC timer fires at approximately 100 Hz.

**Inline exception table.** Kernel user-memory accessors (`copy_from_user`, `copy_to_user`, `get_user_u64`, `put_user_u64`) emit `(fault_va, fixup_va)` pairs into `.ex_table`. The #PF handler does a binary search and redirects to the fixup address on a faulting user access, rather than killing the kernel.

---

### v6: Threading and VFS

**`clone()` with shared resources.** `sys_clone` supports `CLONE_VM`, `CLONE_FILES`, `CLONE_SIGHAND`, and `CLONE_THREAD`. Threads sharing `CLONE_VM` share an `Arc<AddressSpace>` — no page table copy occurs. `CLONE_FILES` shares an `Arc<FileTable>`. `tgid` tracks thread group membership.

**Copy-on-write `fork`.** `sys_fork` walks the parent's user-half PTEs, creates new page table structure for the child, clears `PTE_W`, and sets a software-defined `PTE_COW` bit (page table bit 9) on all writable pages in both parent and child. The #PF handler detects write faults on CoW pages, allocates a fresh zeroed frame, copies the old page, and clears `PTE_COW`. Targeted TLB shootdowns keep all CPUs coherent after the PTE update.

**Per-CPU run queues with work-stealing.** Each CPU owns a `VecDeque<Task>` run queue. `enqueue_task` selects the CPU with the shortest queue (O(n CPUs) scan, reasonable for ≤16 cores). `sched_steal` migrates up to half the tasks from the busiest sibling queue. This prevents any CPU from sitting idle while others have a backlog.

**Userspace preemption.** The LAPIC timer handler checks `cs & 3 == 3` (user mode bit in the interrupted code segment). If the interrupted code was in user mode, `schedule()` is called immediately inside the handler before `iretq`. Kernel-mode contexts set `need_resched` and are preempted at the next safe kernel exit.

**IPI-driven remote reschedule.** `resched(cpu_idx)` sets `need_resched` on the target CPU and, if the target is remote, sends IPI vector `0x43`. The receiving handler sets its flag; `schedule()` is called at the next preemptible point.

**Minimal VFS.** A `Vnode` trait with `read`, `write`, and `is_seekable`. Implementations: `RegularFile` (heap-backed `Vec<u8>`), `DevNull`, `DevZero`. `FileTable` maps `fd → Arc<OpenFile>`. `RamFs` holds named vnodes in a `BTreeMap<String, Arc<dyn Vnode>>`. Pre-populated at boot with `/dev/null`, `/dev/zero`, `/etc/hostname`.

**Pipes.** `PipeInner` is a bounded ring buffer (`PIPE_BUF` = 4 KiB). `PipeReadEnd` and `PipeWriteEnd` implement `Vnode`. A `writers: AtomicUsize` refcount enables read-side EOF detection when all write ends close.

**Signals.** `pending` (64-bit bitmask), `sigmask`, per-signal `SigAction`. `SIGKILL` and `SIGSTOP` cannot be caught. User-space handlers are invoked by pushing a miniature save frame onto the user stack and redirecting `rip`; `sigreturn_trampoline` restores the saved context via `sys_sigreturn`.

**ELF loader.** Validates the ELF header, checks `e_type` (ET_EXEC / ET_DYN) and `e_machine` (EM_X86_64), parses PT_LOAD segments, checks all address and size arithmetic for overflow, maps each segment with correct PTE flags, zeroes BSS, sets up a user stack, and creates a fully initialised `Task`.

---

### v6 (fixed): Correctness Passes

Three classes of real kernel bugs were identified and corrected before any new features were added.

**`notify_zombie` double-free.** The original implementation called `free_process_pml4(t.mm.pml4_phys)` directly inside `notify_zombie`. However, `t.mm` is an `Arc<AddressSpace>` whose `Drop` implementation also calls `free_process_pml4`. Any thread sharing the address space via `CLONE_VM` would trigger a double-free when the `Arc` later dropped. Fixed: the direct call was removed entirely. `AddressSpace::Drop` is now the sole owner of page table teardown.

**Pipe blocking model.** `PipeReadEnd::read` originally spun for up to 100,000 iterations, then called `hlt`. This burned an entire CPU while waiting for data. Replaced with proper task suspension: the reader registers a `Weak<TaskInner>` waker reference in `PipeInner`, sets its state to `Blocked`, and calls `schedule()`. The write end calls `wake_one()` after producing data. The `Drop` impls on both ends call `wake_all()` to unblock any lingering waiters.

**Signal delivery guard for kernel-mode frames.** `deliver_signal` would push a signal frame onto the user stack even when the task was mid-syscall (CS CPL = 0). This corrupted the kernel stack frame in progress. A guard `if (*frame).cs & 3 != 3 { return; }` was added to both `handle_pending_signals` and `deliver_signal`. Signals arriving while in kernel mode are left in `pending` and delivered at the next user-mode return.

---

### v7: Hardening

**Private PML4 per kernel thread.** `spawn_kthread` calls `new_kthread_pml4()`, which allocates a fresh zeroed PML4 and copies only the upper-half kernel entries (indices 256–511). The lower half is empty — a stray kernel pointer cannot reach any user-space mapping. The `AddressSpace::Drop` only frees lower-half structures (indices 0–255), so the shared kernel mappings are not freed per-thread.

**Guard pages and stack canaries.** `alloc_kstack()` allocates `KSTACK_SIZE + 1 × FRAME_SIZE` total. The bottom page is the guard page (left unmapped). A per-task canary `STACK_CANARY ^ prng_next()` is written at the base of the live region at spawn time and stored in `TaskInner::stack_canary`. `schedule()` calls `check_stack_canary()` on the outgoing task before switching. A mismatch triggers an immediate halt with a diagnostic message.

**Write-protected kernel `.text`.** `protect_kernel_text()` walks the boot page tables over `[__text_start, __text_end)` and clears `PTE_W` on every PTE. Called once in `_start` before `irq_enable()`. The linker script exports `__text_start` and `__text_end`. After this call, any write to kernel code (from a bug or exploit) faults with `#PF` error code bit 0 set (protection violation, not a missing mapping).

**`WaitQueue` — proper blocking primitive.** `WaitQueue` replaces ad-hoc `Weak<TaskInner>` waker slots. It holds a `Spinlock<VecDeque<Weak<TaskInner>>>`. `wq.wait(guard)` atomically releases the passed-in spinlock guard (ensuring no lost wakeup) and blocks the task. `wq.wake_one()` and `wq.wake_all()` drain the queue and re-enqueue valid waiters. Used by both pipe ends and by `sys_waitpid`.

**Syscall argument validation.** `SyscallArgs` captures raw register values at the top of every syscall handler. Typed validators are called before touching any kernel state: `va_user` / `va_user_buf` (address range and alignment), `fd_valid` (< MAX_FDS), `sig_valid_kill` / `sig_valid_action` (range and non-maskable exclusions), `mmap_len` (non-zero, rounded up), `prot_valid` (no unknown bits), `clone_flags_valid` (CLONE_VM required, no unknown bits). Failures return named errno constants.

---

### v8: IST, Phased Scheduler, Preemption Control

**Full 7-slot IST assignment.** Every exception that can arrive with a corrupted or exhausted stack gets a dedicated 8 KiB IST stack stored inline in `PercpuData`. The full assignment:

| IST slot | Exceptions | Stack field | Rationale |
|----------|-----------|-------------|-----------|
| 1 | #DB | `db_stack` | Debug traps can occur during interrupt delivery; need clean stack |
| 2 | #BP (int3) | `bp_stack` | DPL 3 — user can fire; IST prevents confusion with kernel handler |
| 3 | NMI, #BR | `nmi_stack` | NMI is the classic non-maskable case; arrives at any time |
| 4 | #DF, #MC | `df_stack` | Both are fatal non-recoverable; share because they cannot nest |
| 5 | #TS, #NP, #SS | `ss_stack` | Segment selector faults; share because they are non-reentrant |
| 6 | #GP, #AC | `gp_stack` | Protection and alignment violations |
| 7 | #PF | `pf_stack` | Own slot: stack overflow appears as clean #PF rather than silent corruption |

**Phased scheduler.** `schedule()` is decomposed into clearly named single-responsibility functions: `sched_pick_next` (dequeue Ready, reap Zombies), `sched_steal` (migrate from busiest sibling), `sched_idle_wait` (brief IRQ-enabled pause for IPI delivery), `perform_context_switch` (re-enqueue prev, call `switch_context`), and `perform_first_run` (bootstrap cold-start). `schedule()` itself is the coordinator: it checks `preempt_count`, clears `need_resched`, runs the pick/steal/idle loop, checks the canary, swaps CR3, and dispatches to the appropriate switch function.

**Preemption disable/enable.** `PercpuData` gains `preempt_count: AtomicI32`. `preempt_disable()` increments it; `preempt_enable()` decrements it and lazily calls `schedule()` if the count reaches zero and `need_resched` is set. `Spinlock::lock` calls `preempt_disable()` before acquiring; `SpinlockGuard::drop` calls `preempt_enable()` after releasing. This means any spinlock critical section automatically suppresses preemption — the same semantics as Linux's `spin_lock`. `schedule()` returns early if `preempt_count() > 0`, making it safe to call unconditionally from any context.

---

## Subsystem Reference

### Memory Management

| Component | Key functions | Notes |
|-----------|--------------|-------|
| Frame allocator | `alloc_frame`, `free_frame`, `alloc_zeroed_frame` | Bump + free list; per-frame `AtomicU32` refcount |
| CoW sharing | `cow_share_frame` | Increments refcount on shared pages; #PF handler does the actual copy |
| Slab caches | `SLAB_16`–`SLAB_2048` | 8 sizes; each cache backed by 4 KiB frames |
| Large allocator | `large_alloc`, `large_dealloc` | Page-granular bump + best-fit free list |
| Address space | `AddressSpace` | `Arc`-managed; `Drop` recursively frees PML4 |
| VMA tracking | `VmaRegion` in `BTreeMap<usize, VmaRegion>` | Demand-zero on page-fault miss |
| Page table ops | `pte_for_vaddr`, `map_pages`, `free_process_pml4` | 4-level walk; allocates intermediate tables on demand |
| TLB coherence | `targeted_tlb_shootdown` | Async IPI + ring drain; blocking completion |
| Text protection | `protect_kernel_text` | Clears PTE_W on `[__text_start, __text_end)` at boot |

### Scheduling

| Function | Single responsibility |
|----------|----------------------|
| `sched_pick_next(cpu)` | Dequeue next Ready task from local queue; reap Zombies |
| `sched_steal(cpu)` | Find busiest sibling; migrate up to half its tasks |
| `sched_idle_wait()` | Re-enable IRQs briefly; spin × 4; re-disable |
| `perform_context_switch(prev, next)` | Re-enqueue prev if preempted; call `switch_context` |
| `perform_first_run(next)` | Cold-start: load context directly; no return |
| `switch_context(save, load)` | Naked ASM: callee-saved save/restore + ret |
| `enqueue_task(task)` | Place on least-loaded CPU's queue |
| `resched(cpu_idx)` | Set `need_resched`; IPI if remote |
| `preempt_disable()` / `preempt_enable()` | Increment / decrement `preempt_count`; lazy schedule on enable |
| `schedule()` | Orchestrate all phases; guard on `preempt_count` |

### IST / Exception Stack Assignment

| Vector | Name | IST | Stack | Handler |
|--------|------|-----|-------|---------|
| 0 | #DE | — | irq_stack | kill task |
| 1 | #DB | 1 | db_stack | log, continue |
| 2 | NMI | 3 | nmi_stack | log |
| 3 | #BP | 2 | bp_stack | log |
| 5 | #BR | 3 | nmi_stack | kill task |
| 6 | #UD | — | irq_stack | kill task |
| 8 | #DF | 4 | df_stack | panic |
| 10 | #TS | 5 | ss_stack | kill task |
| 11 | #NP | 5 | ss_stack | kill task |
| 12 | #SS | 5 | ss_stack | kill task |
| 13 | #GP | 6 | gp_stack | kill task |
| 14 | #PF | 7 | pf_stack | CoW / demand-zero / SIGSEGV |
| 17 | #AC | 6 | gp_stack | kill task |
| 18 | #MC | 4 | df_stack | panic |
| 32 | Timer | — | irq_stack | tick, signals, preempt |
| 0x42 | TLB flush IPI | — | irq_stack | drain ring |
| 0x43 | Resched IPI | — | irq_stack | set need_resched |

### Syscall Table

| Number | Name | Key validation |
|--------|------|---------------|
| 0 | `read` | `fd_valid`, `va_user`, length cap |
| 1 | `write` | `fd_valid`, `va_user`, length cap |
| 2 | `open` | `va_user` (path pointer), null-terminator scan |
| 3 | `close` | `fd_valid`, fd open check |
| 9 | `mmap` | `mmap_len`, `prot_valid`, hint alignment |
| 11 | `munmap` | page alignment, length, address in user range |
| 13 | `sigaction` | `sig_valid_action`, `va_user_buf` for both pointers |
| 15 | `sigreturn` | `va_user_buf`, restored `rip < KERNEL_VMA` |
| 22 | `pipe` | `va_user_buf(8)` for fds array |
| 33 | `dup2` | `fd_valid` × 2, fd open check |
| 56 | `clone` | `clone_flags_valid`, `va_user(child_rsp)` if non-zero |
| 57 | `fork` | — |
| 60 | `exit` | — |
| 62 | `kill` | `sig_valid_kill`, pid non-zero |
| 247 | `waitpid` | pid non-zero, pid != self |
| 500 | `yield` | — |

---

## What Still Needs to Be Done

### Near-Term (High Priority)

**`sys_clone` without `CLONE_VM`.** Currently returns `EINVAL` if `CLONE_VM` is not set. Creating a new process via `clone` (without shared address space) is therefore impossible. The CoW walk from `sys_fork` should be extracted into a shared `fork_address_space(parent) -> Option<u64>` helper and called from both.

**`CLONE_SIGHAND` as true Arc-sharing.** Currently copies the signal handler map at `clone` time. A handler installed by one thread after creation is not seen by siblings. `SignalInfo::handlers` needs to become `Arc<Spinlock<BTreeMap<u32, SigAction>>>` shared across the thread group.

**`waitpid(-1, ...)`.** The current implementation requires an exact child PID. POSIX `waitpid(-1, ...)` (wait for any child) is not implemented. This requires a per-process `Arc<WaitQueue>` for child-exit events, shared among all children.

**Guard page enforcement.** Guard pages exist in the heap allocation but the PML4 does not actively forbid access to them — `pte_for_vaddr` with `alloc=true` would silently create a mapping. The guard page VA range must be explicitly marked not-present in the kthread's PML4 at spawn time.

**`sys_execve`.** `create_task_from_elf` is implemented but there is no `execve` syscall. A full implementation must: replace the current address space in-place, load the new ELF, reset signal handlers to defaults, close `FD_CLOEXEC` file descriptors, and lay out `argv`/`envp` on the new user stack.

**`sys_brk`.** Programs expecting glibc-style heap growth via `brk` will fail. A per-process `brk` pointer and a simple `sys_brk` handler would allow `malloc` implementations that use the heap break rather than `mmap`.

**Standard errno values.** Sentinel values are currently `u64::MAX - N`. Standard Linux programs expect negative `i64` errno values (e.g. `EINVAL = -22`). All errno constants and return-value encoding must match the Linux ABI.

**Lost-wakeup race in `WaitQueue::wait`.** Between `push_back(downgrade)` and `set_state(Blocked)`, a producer on another CPU could call `wake_one()` and miss the not-yet-blocked waiter. This requires either a generation counter checked under the data lock, or restructuring `wait` to set `Blocked` under the same lock that guards the data predicate.

---

### Medium-Term (Infrastructure)

**Tickless / one-shot LAPIC timer.** The current periodic ~100 Hz mode wastes ticks on idle CPUs. Needed: TSC calibration against HPET or PM timer, one-shot LAPIC mode, and a sorted timer wheel to support accurate `nanosleep` and `poll` timeouts.

**`sys_poll` / `sys_select` / `sys_epoll`.** Without I/O multiplexing primitives, multi-connection servers cannot be written. The `WaitQueue` abstraction already provides the needed blocking; a `PollTable` that collects wait-queue registrations and wakes the poll caller when any fires is the missing piece.

**VirtIO block driver.** Enables persistent storage. A MMIO or PCI VirtIO block device driver that enqueues descriptor chains and handles completions via interrupt is the prerequisite for everything that follows.

**Ext2 or FAT filesystem.** Even a read-only ext2 driver mounted over the VirtIO block device would allow loading programs from disk rather than embedding them in the kernel binary.

**Dynamic linker support.** The ELF loader accepts `ET_DYN` but never invokes `ld.so`. Supporting dynamic linking requires either loading the dynamic linker as a user-space executable and setting up `AT_*` auxiliary vectors, or implementing basic RELA/PLT fixup in the kernel loader.

**`sys_mprotect`.** Changing VMA permissions after mapping is required for JIT compilers, `dlopen`, and any code that generates executable pages at runtime.

**File-backed `mmap`.** Currently only anonymous (demand-zero) mappings are supported. File-backed mappings are needed for `dlopen`, shared libraries, and memory-mapped I/O.

**Coalescing large allocator.** `large_dealloc` pushes freed regions to a free list without merging adjacent blocks. Long-running workloads will fragment the large-allocation heap. A boundary-tag allocator or a buddy system would fix this.

**Network stack.** A VirtIO-net NIC driver, ARP/IP/UDP/TCP, and socket syscalls (`socket`, `bind`, `connect`, `send`, `recv`, `accept`). The `WaitQueue` primitive already handles blocking I/O; the socket layer just needs to hook into it.

**CPU affinity syscall.** `sched_setaffinity` / `sched_getaffinity`. The `cpu_affinity` field in `TaskInner` is present but only set heuristically. Exposing it via syscall allows pinning threads to specific CPUs.

**Larger physical address space.** Parse the BIOS/UEFI memory map instead of hard-coding `PHYS_MEM_LIMIT = 512 MiB`. Extend the boot PD to cover more than 256 × 2 MiB entries. Support NUMA topology (multiple ACPI SRAT entries).

---

### Long-Term (Full OS)

**UEFI boot.** Replace the raw `_start` entry with a UEFI application that uses Boot Services to obtain a memory map, sets up the framebuffer, loads the kernel image to the correct virtual address, and exits Boot Services before jumping to the kernel.

**`SYSCALL`/`SYSRET` fast path.** `int 0x80` costs a full privilege-level change through the IDT. The `SYSCALL` instruction skips the IDT lookup and uses `IA32_LSTAR`/`IA32_STAR` MSRs. Implementing it requires a separate entry stub that handles the different ABI (no automatic stack switch; `rcx` holds the return address).

**KASLR.** Randomise the kernel's virtual base at boot using the TSC or a boot-time entropy source (`RDRAND` if available). Requires relocatable kernel image (PIE) and a boot stub that fixes up absolute addresses.

**KPTI (Kernel Page Table Isolation).** Maintain two page table sets per process: a minimal "user" set containing only the kernel trampoline needed for `SYSCALL` entry, and a full "kernel" set used during kernel execution. Mitigates Meltdown and related speculative execution attacks.

**Userspace `pthreads` library.** A minimal threading library on top of `clone(CLONE_VM | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD)` implementing `pthread_create`, `pthread_join`, `pthread_mutex_t`, and `pthread_cond_t` using `sys_futex`-like primitives.

**Full POSIX process model.** `getpid`, `getppid`, `getuid`, `setuid`, process groups, sessions, `setsid`, `setpgid`, `tcsetpgrp`. These are required for a POSIX-conformant shell and for `init`-style process supervision.

**Loadable kernel modules.** A simple ELF-based module loader that maps code into the kernel address space, resolves symbol references against the running kernel's symbol table, and calls the module's `init`/`exit` functions.

**Crash dump.** A second minimal crash kernel (kexec-style) that runs after a panic, dumps the crashed kernel's memory to a block device, and reboots.

---

## Potential Uses

### Education and Research

CosmicWave is purpose-built for learning. The source is compact enough to read across a weekend but rich enough to teach every major kernel concept at production fidelity.

**OS courses.** Each version of CosmicWave maps to a standard OS course module: v6 covers threads, virtual memory, and IPC; v7 covers kernel security primitives; v8 covers interrupt architecture and scheduling theory. The progression — introduce, identify bug, fix, harden, extend — mirrors the real development cycle of production kernels and teaches students not just the concepts but the failure modes.

**Systems programming courses.** The `unsafe` blocks in CosmicWave are a catalogue of the exact operations that require memory-safety opt-outs: MMIO access, page table manipulation, user-memory copying, naked interrupt stubs. Every one is bracketed with the invariant that must hold across it. This makes the Rust `unsafe` model concrete and motivates it more effectively than abstract language documentation.

**Security research.** The versioned structure is a case study in kernel hardening. Each version adds a class of defence against a known attack pattern: v6-fixed addresses double-free and information leakage; v7 adds ASLR-relevant mitigations (stack canaries, private PML4s, write-protected text); v8 closes IST-based stack-pivot attacks. CosmicWave is small enough to be a tractable target for academic fuzzing research (kAFL, Nyx), symbolic execution, and kernel eBPF-style safety proofs.

**Architecture exploration.** Want to understand why Linux's scheduler has five distinct phases for `__schedule()`? Or why Linux bothers with `preempt_count` in spinlocks? CosmicWave provides a minimally complex implementation of both, making the rationale clear without the distraction of production complexity.

### Kernel Development Prototyping

When designing a new kernel subsystem, implementing it first in CosmicWave has dramatically lower friction than prototyping in Linux or FreeBSD.

The entire relevant codebase is in one file. There are no Kconfig dependencies, no `include/linux/` header chains, no build system beyond a single shell script. Boot-to-test-result in QEMU takes under three seconds. The Rust type system rejects data races at compile time, so a prototype that compiles is already free of the most common concurrency bugs.

Subsystems well-suited to prototyping here: wait queue designs, CoW fork semantics, pipe ring buffer layouts, TLB shootdown protocols, IST stack assignments, syscall ABI conventions, and VFS abstraction layers.

### Embedded and Bare-Metal Rust Reference

CosmicWave demonstrates patterns directly applicable to any bare-metal Rust project that needs to interact with x86-64 hardware:

- IRQ-save/restore with RFLAGS using inline assembly
- Per-CPU data accessed via GS segment base (`IA32_GS_BASE` MSR)
- LAPIC MMIO via `ptr::read/write_volatile` without device drivers
- Naked functions for interrupt entry stubs with precise ABI control
- `#[global_allocator]` implementation with no OS heap support
- `AtomicU32` / `AtomicUsize` refcounting with correct memory ordering
- ACPI table parsing without external crates
- ELF loading directly from byte slices

These patterns appear repeatedly in firmware, hypervisors, embedded real-time systems, and TEE (Trusted Execution Environment) runtimes.

### Hypervisor Guest Testing

CosmicWave's independence from external dependencies and its predictable behaviour make it a clean guest for hypervisor development and validation.

**LAPIC emulation.** CosmicWave exercises the LAPIC timer (periodic mode, ICR, DCR), IPI delivery (fixed delivery, specific destination), and EOI writes. A hypervisor implementing APIC virtualisation can use CosmicWave to stress-test its emulation with a known-correct workload.

**SMP bringup.** The real-mode AP trampoline, INIT-SIPI-SIPI sequencing, and AP-side GDT/TSS/IDT setup exercise the vCPU startup and inter-processor reset paths in the hypervisor.

**TLB coherence.** The shootdown IPI sequence (write PTE → invalidate local → send IPI → wait for ring drain) generates a specific pattern of VM exits and INVLPG execution that can expose bugs in EPT/shadow page table coherence handling.

**Page fault injection.** CoW fork generates a predictable stream of write faults that can test a VMM's nested page fault handling, memory ballooning, and dirty page tracking.

### Foundation for a Purpose-Specific OS

For teams building a minimal production system — a unikernel, a game console kernel, a TEE runtime, a secure enclave supervisor — CosmicWave provides a working starting point with the following properties:

- **Linux-compatible syscall numbers** (read=0, write=1, exit=60, fork=57, etc.) so that programs compiled for Linux musl will call the right handlers.
- **Working ELF loader** that handles both `ET_EXEC` and `ET_DYN`, including per-segment permission mapping.
- **Working VFS abstraction** that can be extended with real filesystem backends by implementing `Vnode`.
- **SMP infrastructure** scaling to 16 cores, tested with real QEMU multi-vCPU emulation.
- **Memory allocator** that works without any external heap runtime.
- **No mandatory subsystems**: there is no device model, no driver framework, no scheduler policy that cannot be replaced. The codebase is structured to be stripped down rather than built up.

---

## Project Layout

```
cosmicwave/
├── README.md                  ← This file
├── Cargo.toml                 ← Rust package manifest (no dependencies)
├── x86_64-cosmicwave.json     ← Custom target spec (no_std, no redzone, soft-float)
├── build.sh                   ← Build + QEMU launch in one script
└── src/
    ├── linker.ld              ← Section layout; exports __text_start/end, __ex_table_start/end
    └── main.rs                ← Entire kernel (~2,800 lines)
```

The single-file approach is intentional. Subsystem boundaries are marked with section dividers (`// ═══…`) rather than module boundaries. The entire call graph is visible without navigating between files.

---

## Building and Running

### Requirements

| Tool | Minimum version | Purpose |
|------|----------------|---------|
| `rustup` | any | Nightly toolchain management |
| Rust nightly | ≥ 2024-01 | `naked_functions`, `asm_const`, `asm_sym`, `alloc_error_handler` |
| `lld` | ≥ 14 | `rust-lld` for the custom target |
| `qemu-system-x86_64` | ≥ 7.0 | x86-64 emulation with APIC and SMP |

### Install nightly and components

```bash
rustup toolchain install nightly
rustup component add rust-src --toolchain nightly
```

### Build flags

```toml
# Cargo.toml
[profile.release]
panic    = "abort"   # No unwinding runtime needed
lto      = true      # Whole-program inlining
opt-level = "s"      # Optimise for size
```

```json
// x86_64-cosmicwave.json
"disable-redzone": true          // ISRs cannot use the red zone
"features": "-mmx,-sse,+soft-float"   // No SSE save/restore in kernel
"relocation-model": "static"     // No PIE (KASLR deferred to future work)
```

### QEMU flags explained

```bash
-smp 4               # 4 vCPUs — exercises work-stealing and IPI paths
-m 512M              # Must match PHYS_MEM_LIMIT constant
-cpu qemu64,+apic,+smep,+smap   # Expose SMEP/SMAP to test hardware enforcement
-serial stdio        # All kernel output to terminal
-display none        # No graphical window
-no-reboot           # Stay up after triple-fault for post-mortem
```

---

## Design Philosophy

**One file, one read.** The kernel is in a single source file so that a reader never has to context-switch between files to understand a subsystem. The cost is length; the benefit is that the entire call graph and all data layout decisions are visible at once.

**Document the invariant, not the mechanism.** Comments explain *why* a particular ordering, data structure, or instruction sequence is required — not *what* the code does (the code makes that clear). Where a known real-world kernel bug motivated a design choice, that bug is described.

**`unsafe` at the boundary, safe above it.** All hardware access, user-memory copies, and raw pointer arithmetic are in `unsafe`. The safe abstractions above them (`WaitQueue`, `Spinlock`, `AddressSpace`) enforce the invariants that make the `unsafe` correct. This is what `no_std` Rust enables that C cannot: machine-checked safety boundaries with no runtime overhead.

**Fix before extend.** The versioned structure reflects a deliberate discipline: identify and correct defects in the current version before adding features. v6 fixed three correctness bugs before v7 added any new capabilities. This prevents the latent-defect accumulation that characterises most long-lived OS codebases.

**Correctness over performance.** The slab allocator has no per-CPU caches. The scheduler does not implement CFS or EEVDF. The TLB shootdown spins until all CPUs acknowledge. These are knowingly simpler than production implementations. Where a production system would diverge, the comment says so and explains the trade-off.

---

## Known Limitations

| Area | Current state |
|------|-------------|
| Physical memory limit | Hard-coded 512 MiB; no memory map parsing |
| Filesystem | RamFs only; no persistence; no block device |
| Networking | Not implemented |
| Syscall ABI | `int 0x80`; no `syscall`/`sysret` fast path |
| `clone` without `CLONE_VM` | Returns EINVAL; process-create via clone not supported |
| `CLONE_SIGHAND` | Copies handlers at clone time; not truly shared |
| `waitpid` | Exact PID only; no wait-for-any-child |
| Large allocator | No coalescing; long-running heap will fragment |
| Timer | Fixed ~100 Hz periodic; no `nanosleep` / `timerfd` |
| Address space | 512 MiB physical; no KASLR; no KPTI |
| Guard pages | Allocated but PML4 not explicitly marked not-present |
| Errno values | Non-standard sentinel values; not Linux ABI compatible |
| Lost-wakeup race | `WaitQueue::wait` has a narrow TOCTOU window |
| Security | No speculative execution mitigations |
| Boot | Raw `_start`; no GRUB multiboot or UEFI support |
| Debugging | Serial only; no GDB stub, no KGDB, no crash dump |

---

## Version History

| Version | Summary |
|---------|---------|
| v1–v5 | Paging (4-level, 2 MiB huge, NX, SMEP/SMAP); frame allocator with CoW refcounts; slab + large allocator; IRQ-safe spinlock; SMP bringup (ACPI MADT, INIT-SIPI-SIPI); TLB shootdown ring; full IDT; exception table; safe user accessors (SMAP bracket + inline fixup) |
| v6 | Per-CPU run queues; work-stealing scheduler; LAPIC user-mode preemption; IPI reschedule; CoW fork; `clone(CLONE_VM/FILES/SIGHAND/THREAD)`; VFS (`Vnode` trait, `FileTable`, `RamFs`); pipes; basic signals; ELF loader |
| v6-fixed | Fix `notify_zombie` double-free; replace pipe spin-wait with blocking `Weak` wakeup; signal delivery guard for kernel-mode frames |
| v7 | Private PML4 per kernel thread; guard pages + PRNG stack canaries; write-protect kernel `.text` after boot; `WaitQueue` blocking primitive; syscall argument validation with typed validators and errno constants |
| v8 | Full 7-slot IST assignment (8 KiB per slot, all critical exceptions IST-switched); phased scheduler decomposition (`sched_pick_next`, `sched_steal`, `sched_idle_wait`, `perform_context_switch`, `perform_first_run`); `preempt_count` disable/enable with lazy schedule; spinlock preemption management |

---

## License

MIT License

Copyright (c) 2025 CosmicWave Contributors

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
