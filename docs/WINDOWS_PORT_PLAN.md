# Windows port plan (NVIDIA-first)

Status: proposal, 2026-10-05. Nothing in this document has been built or tested on Windows.
It maps each Linux-only mechanism in bbport to the corresponding Windows mechanism that
shadPS4 (GPL-2.0-or-later, the same licence as bbport) already uses in production, so the
scope can be judged before any code is written.

Target: Windows 10 1803 or newer, x86-64, a Vulkan 1.3 GPU. NVIDIA RTX 20-series or newer
gets FSR 3.1 and should get FSR 4; FSR 4.1.1 stays Linux/Mesa-only (see §7).

## 1. Why this is not a recompile

bbport does not emulate a CPU. It loads the game's own x86-64 code into memory and runs it,
and a 5,500-line C runtime answers every request the game would normally make to the PS4
operating system. To do that convincingly on Linux it uses kernel features that Windows
does not have in the same form. Those features are concentrated in a small number of
files, and that is where all the real work is.

Everything else is already portable or nearly so:

| Part | Lines | Portability |
|---|---|---|
| Renderer (`gpu/shadps4`, `gpu/shim`) | ~105,000 | shadPS4 upstream builds on Windows. bbport's ~200 changes are algorithmic. |
| Window, gamepad, audio | — | SDL3, cross-platform. |
| Preparation scripts (`scripts/`) | ~2,000 Python | Pure Python, no Linux calls. |
| Game patches (`patches/`, `scripts/patches.py`) | — | Byte patches on the memory image. |
| Launcher (`launcher/`) | ~1,500 Python | GTK4/libadwaita. Not portable in practice; replace or drop (§8). |
| Runtime + loader (`src/`) | ~5,500 C | **The work.** Six files hold the Linux dependencies. |

## 2. The six hard problems, each mapped to shadPS4's Windows solution

Each entry says what the mechanism does in plain terms, how bbport does it on Linux, how
shadPS4 does it on Windows, and what has to change in bbport.

### 2.1 Thread-local storage and the GS register (hardest)

**What it is.** Every thread in the game keeps a small block of private data (its "thread
control block", TCB). The game finds that block through a CPU segment register. On the PS4
the register is FS and the game code contains instructions like `mov rax, fs:[0]`.

**bbport on Linux.** The host C library owns FS, but GS is free. So the offline linker
rewrites the segment prefix byte of every matching 9-byte `mov rax, fs:[0]` instruction to
GS (`scripts/link_libc.py:160`), and each thread points GS at its TCB with the Linux-only
`arch_prctl(ARCH_SET_GS)` call (`src/runtime_thread.c:61`).

**Windows.** GS belongs to the operating system: it points at the Thread Environment Block
(TEB) and cannot be redirected from user mode. FS cannot be set either. So no segment
register is available, and every rewritten instruction needs a different fix.

**shadPS4 on Windows** (`src/core/cpu_patches.cpp`, `src/core/tls.cpp`):

- Allocates one Windows TLS slot with `TlsAlloc()`; each thread stores its TCB pointer
  there with `TlsSetValue()`.
- Uses the Zydis disassembler to find `mov`, `cmp` and `xor` instructions whose source is
  `fs:[disp]` with no base or index register and `disp < sizeof(Tcb)` (`FilterTcbAccess`).
- Replaces each with a near jump (5 bytes) to a generated trampoline (xbyak) that reads
  the TCB pointer from the TEB directly: `gs:[0x1480 + slot*8]` for slots below 64, or via
  the expansion table at `gs:[0x1780]` otherwise (`RetrieveTcbPointer`), then performs the
  original operation and jumps back. Scratch registers are rax, or rbx if rax is the
  destination. The stack pointer is moved 128 bytes to skip the red zone before pushing.
- Patches ahead of time where it can (`TryPatchAot`) and otherwise lazily from the
  illegal-instruction exception handler (`TryPatchJit`).

**Change in bbport.**

- `scripts/link_libc.py`: on a `--windows` flag, do not rewrite FS to GS. Instead emit a
  list of the matched instruction offsets into the `BBPROBE` image header (a new
  `BBPROBE6` record, or a side table), so the loader does not need a disassembler at run
  time. The current filter matches only the exact 9-byte `mov rax, fs:[0]`; verify with
  Zydis over the whole image that no other `fs:` access exists in the eboot and the
  linked PS4 libc (shadPS4's wider filter exists because other games need it).
- `src/runtime_thread.c`: replace `set_gs()` with `TlsSetValue(tcb_slot, tcb)`.
- `src/probe.c` (Windows branch): at load, overwrite each listed instruction with a near
  jump into a trampoline page allocated with `VirtualAlloc(..., PAGE_EXECUTE_READWRITE)`.
  Port `RetrieveTcbPointer` as hand-written machine code bytes (bbport has no xbyak
  dependency in `src/`; the sequence is about 30 bytes and fixed, so a byte template with
  the slot index patched in is enough).
- The 9-byte instruction leaves 4 bytes of padding after the 5-byte jump; fill with `nop`.

Risk: any `fs:` access the filter misses crashes immediately with an access violation at
a recognisable address, so misses are easy to find but each costs an iteration.

### 2.2 Memory aliasing: the same physical memory at several addresses

**What it is.** The PS4 lets a game map one block of physical memory at more than one
virtual address, and the GPU reads the same memory the CPU writes. The runtime must honour
`sceKernelMapDirectMemory`, `sceKernelMprotect`, `sceKernelBatchMap` and friends.

**bbport on Linux** (`src/runtime_memory.c`). One anonymous memory file from
`memfd_create` holds all "direct" and "flexible" memory (5 GiB + flexible span). Every guest
mapping is `mmap(addr, MAP_SHARED | MAP_FIXED, pool_fd, phys)`; protection changes are
`mprotect`; the whole pool is also mapped once contiguously for the GPU side.

**shadPS4 on Windows** (`src/core/address_space.cpp`):

- `CreateFileMapping2(INVALID_HANDLE_VALUE, ..., PAGE_EXECUTE_READWRITE, SEC_COMMIT, size)`
  creates the backing memory object (the analogue of the memfd).
- `VirtualAlloc2(..., MEM_RESERVE | MEM_RESERVE_PLACEHOLDER)` reserves the guest address
  ranges up front so nothing else lands there.
- `MapViewOfFile3(backing, ..., addr, offset, size, MEM_REPLACE_PLACEHOLDER, prot)` maps a
  piece of the backing object at a fixed address; this is the `MAP_FIXED` equivalent and
  the only way to alias on Windows.
- Unmapping is `UnmapViewOfFile2(..., MEM_PRESERVE_PLACEHOLDER)` so the range stays
  reserved, then `VirtualFreeEx(..., MEM_COALESCE_PLACEHOLDERS)` to merge.
- Protection changes are `VirtualProtectEx`.
- Address ranges: `SYSTEM_MANAGED 0x400000–0x7FFFFBFFF`, `SYSTEM_RESERVED
  0x7FFFFC000–0xFFFFFFFFF`, `USER_MIN 0x1000000000`. Windows 11 22H2 and earlier are
  detected with `RtlGetVersion` because `VirtualAlloc2` is slow there with very large
  reservations, and the user range is capped at `0x10000000000` (1 TiB).

**Change in bbport.** Rewrite `runtime_memory.c`'s host layer (roughly the 150 lines
around `pool()`, `place_mapping`, `kernel_mprotect`, `runtime_low_map`) behind a small
internal interface: `host_pool_create`, `host_map_fixed`, `host_unmap`,
`host_protect`, `host_reserve_low`. Keep the guest-visible logic (overlap checks,
budgets, hooks to the GPU) unchanged. One important difference: on Linux an `mmap` over an
existing mapping replaces it; on Windows the range must be a placeholder first, so
`host_map_fixed` must unmap-preserving-placeholder before mapping. Placeholders cannot be
split arbitrarily without `VirtualFreeEx(MEM_PRESERVE_PLACEHOLDER)` on the sub-range, so
mirror shadPS4's split-then-map sequence exactly.

### 2.3 Addresses below 1 TiB

**What it is.** The game packs pointers into 40 bits, so every pointer the runtime hands
to it (thread handles, TLS blocks, stacks, mutex objects) must be below 1 TiB.

**bbport on Linux.** `mallopt(M_ARENA_MAX, 1)` and a large `M_MMAP_THRESHOLD` keep glibc's
heap in the low `brk` region (`src/probe.c:273`); stacks come from `runtime_low_map`,
which tries `MAP_FIXED_NOREPLACE` at increasing low addresses.

**Windows.** The default heap and thread stacks land wherever the loader puts them,
usually low on a fresh process but not guaranteed, and ASLR applies.

**shadPS4 on Windows.** Reserves the whole guest range up front (§2.2) and allocates its
own objects from that reservation; it does not rely on the system heap for guest-visible
objects.

**Change in bbport.** Reserve one low region (for example 0x10000000–0x7FFFFFFFF) with
`VirtualAlloc2(MEM_RESERVE_PLACEHOLDER)` at startup, and carve guest-visible host objects
(`aligned_alloc` in `attach()`, `calloc` for `GuestThread`, mutex/sema/rwlock objects,
guest stacks) from a simple bump or free-list allocator over that region. Audit: grep
`src/` for every allocation whose pointer is returned to guest code. This also removes the
glibc-specific `mallopt` calls. Build the executable with `/DYNAMICBASE:NO` or `-Wl,--disable-dynamicbase`
so the image itself stays low.

### 2.4 GPU write tracking

**What it is.** The renderer must learn when the game writes to memory the GPU has cached
(textures, vertex buffers). bbport also denies reads on pages being read back from the GPU.

**bbport on Linux** (`gpu/shadps4/video_core/page_manager.cpp:275`, `UffdImpl`).
Write-protects pages with `userfaultfd` (`UFFDIO_WRITEPROTECT`) and reads faults from a
file descriptor on a service thread; readback denial uses `mprotect` plus the SIGSEGV
handler. The project measured this as faster than page protection because `mprotect` takes
the kernel's address-space lock and triggers TLB shootdowns on every call.

**shadPS4 on Windows** (`SignalImpl` in the same file, still present in bbport's vendored
copy at line 413). Protects pages through `AddressSpace::Protect` →
`VirtualProtectEx`, and receives the fault through the vectored exception handler (§2.5),
which calls `RegisterAccessViolationHandler` callbacks. Windows has no `userfaultfd`
equivalent. (`GetWriteWatch` exists but only on `MEM_WRITE_WATCH` private allocations, not
on file-mapped views, so it cannot serve aliased memory.)

**Change in bbport.** Compile out `UffdImpl` on Windows and select `SignalImpl`. Implement
`AddressSpace::Protect` in `gpu/shim/bbgpu.cpp:133` with `VirtualProtectEx` (it is
`mprotect` today). The 256 KiB "unprotect window" optimisation that bbport documents as
+22% FPS applies to either backend and should be kept. Expect lower performance than Linux
in load-heavy scenes; measure before optimising.

### 2.5 Signals → vectored exception handling

**What it is.** The loader catches crashes to print diagnostics, lets the GPU library
handle access faults (§2.4), recovers from speculative guest-memory reads with
`sigsetjmp`/`siglongjmp`, dumps every thread's stack on `SIGUSR2`, and runs a startup
watchdog on `SIGALRM`.

**bbport on Linux.** `sigaction` for SEGV/ILL/BUS/USR2/ALRM in `src/probe.c:320`;
`ucontext_t` register access in `probe.c` and `gpu/shadps4/common/signal_context.cpp`.

**shadPS4 on Windows** (`src/core/signals.cpp`). `AddVectoredExceptionHandler(0, handler)`;
maps `EXCEPTION_ACCESS_VIOLATION` → SIGSEGV path, `EXCEPTION_ILLEGAL_INSTRUCTION` → SIGILL
path; reads and modifies registers via `pExp->ContextRecord` (`Rip`, `Rsp`, ...); returns
`EXCEPTION_CONTINUE_EXECUTION` when handled, else `EXCEPTION_CONTINUE_SEARCH`.
`signal_context.cpp` already has `_WIN32` branches that read `CONTEXT` fields.

**Change in bbport.**

- `probe.c`: one vectored handler replacing `fault()`. The fault-recover path becomes:
  set `ContextRecord->Rip` to a recovery stub and `Rax` to a flag, return
  `EXCEPTION_CONTINUE_EXECUTION`; the stub `longjmp`s. (Plain `setjmp`/`longjmp` is fine on
  Windows; there is no signal mask to restore.)
- Thread dump: replace `tgkill(SIGUSR2)` with `SuspendThread` + `GetThreadContext` +
  `ResumeThread` over a list of thread handles the runtime already keeps (`threads` in
  `runtime_thread.c`).
- Watchdog: `CreateTimerQueueTimer` or a plain thread with `Sleep`.
- `gpu/shim/core/signals.h` dispatch is already context-agnostic through
  `Common::GetRip`; keep it.

### 2.6 Threads and synchronisation primitives

**What it is.** The runtime implements PS4 threads, mutexes, semaphores, rwlocks and
condition variables on top of pthreads (`runtime_thread.c`, `runtime_mutex.c`,
`runtime_sema.c`, `runtime_rwlock.c`, about 980 lines). Timed waits use
`clock_gettime(CLOCK_MONOTONIC)`.

**Windows.** Two options: (a) build against a pthreads layer (MinGW-w64's winpthreads, or
llvm-mingw) and keep the code; (b) rewrite on `SRWLOCK`, `CONDITION_VARIABLE`,
`CreateSemaphore`, `WaitOnAddress` as shadPS4 does in `src/core/libraries/kernel/threads/`.
Option (a) is far less work and is what the recommended toolchain (§3) provides; the only
pthread calls not in winpthreads are `pthread_setname_np` (use `SetThreadDescription`) and
`pthread_getcpuclockid` (`QueryThreadCycleTime`). Recursive and timed mutex variants exist
in winpthreads. `sched_yield` → `SwitchToThread`; `clock_gettime` exists in winpthreads.

### 2.7 Files, saves, services, time (small)

`runtime_file.c`, `runtime_savedata.c`, `runtime_services.c`, `runtime_rtc.c`,
`runtime_kernel.c` use POSIX `open`/`stat`/`readdir`/`mkdir`/`clock_gettime`/`getdents`.
MinGW provides all of these (with `_mkdir` semantics differences for permissions, which are
ignored anyway). Case-insensitive file lookups for mods (`scripts/mods.py`, `runtime_file.c`)
become simpler on NTFS. Restart-on-settings-change (`runtime_restart`, `execlp bash run.sh`)
becomes `CreateProcess` on the launcher executable, or is dropped in favour of the live
resolution path.

## 3. Toolchain decision

Use **clang (llvm-mingw) or GCC (MSYS2 MinGW-w64), not MSVC.** Reasons:

- `src/` uses `__attribute__((sysv_abi))` on every guest-callable function (`ABI` in
  `runtime.h:12`). The game code is System V ABI; Windows is Microsoft x64 ABI. GCC and
  clang both honour `sysv_abi` on Windows targets so the runtime can be called directly from
  guest code. MSVC has no equivalent and would need hand-written thunks for ~300 functions.
- `probe.c` has GNU inline assembly (`enter_on_stack`).
- The renderer uses GNU extensions in places and shadPS4's own Windows CI uses clang-cl /
  MSYS2.
- Dependencies (SDL3, fmt, Boost, magic_enum, robin-map, xxhash, VMA, glslang, SPIRV-Cross,
  Zydis, miniz, ffmpeg) are all in MSYS2's `mingw-w64-clang-x86_64-*` repository.

Build system: convert `build.sh` into a top-level `CMakeLists.txt` that includes `gpu/`
and builds `bb-probe.exe` plus `bbgpu.dll`. On Windows the GPU library should either be
static or export a C ABI only (it already does: `gpu/bbgpu.h`), since C++ symbols across a
DLL boundary with different runtimes are fragile. Drop `-rpath`; put the DLL next to the
executable. LTO/PGO stay available in clang.

## 4. Phased plan

Each phase ends with a check that can be observed, so no phase claims progress it cannot
show. Effort assumes one experienced systems programmer familiar with Win32 memory APIs.

| Phase | Deliverable | Observable check | Estimate |
|---|---|---|---|
| 0. Toolchain | `gpu/` builds as `bbgpu.dll` under llvm-mingw; `tools/gpu_capabilities.c` and `src/vulkan_smoke.c` run | `bb-probe.exe --vulkan-only` opens a window and clears it on the NVIDIA card | 1–2 weeks |
| 1. Runtime compiles | `src/` builds with the `host_*` memory interface (§2.2) stubbed, signals (§2.5) and TLS (§2.1) on Windows paths; existing C tests (`tests/test_runtime.c`, `test_sema.c`, `test_content.c`, `test_file_mods.c`) pass | Test binaries exit 0 on Windows | 1–2 weeks |
| 2. Memory | `host_*` implemented on `CreateFileMapping2`/`MapViewOfFile3`; low-region allocator (§2.3) | A new unit test maps the same backing offset at two addresses, writes through one, reads through the other, protects, unmaps, remaps | 2–3 weeks |
| 3. CPU-only boot | `link_libc.py --windows` emits the TLS patch table; loader applies trampolines; `bb-probe.exe boot.bin --cpu-only` runs the game's initialisation | Log reaches the same "Entering original x86-64 code" and first `sceKernel*` calls as the Linux `--cpu-only` run; `--strict-imports` lists no missing symbols | 2–4 weeks (TLS is the unknown) |
| 4. GPU boot | `SignalImpl` tracking, `VirtualProtectEx`, fault dispatch through the vectored handler | Title screen renders; `BB_FRAME_STATS=1` prints frame times | 2–3 weeks |
| 5. Play | Audio, pad, saves verified; FSR 3.1 then FSR 4 on RTX | Hunter's Dream loads a save; 30 minutes without a crash (the project's existing soak criterion, `tools/soak.sh`) | 2–4 weeks |
| 6. Packaging | Minimal launcher (§8), zip or installer, shader-cache and `user/` paths under `%LOCALAPPDATA%` | Fresh Windows machine runs from the zip | 1–2 weeks |

Total: roughly 3 to 5 months of focused work, with phase 3 carrying most of the
uncertainty. Phases 0, 1 and 2 can be done without the game dump and are safe to start
first.

## 5. What stays Linux-only

- FSR 4.1.1 (needs `VK_VALVE_shader_mixed_float_dot_product`, Mesa only, and the asset
  build runs AMD's DLL under vkd3d-proton).
- `userfaultfd` write tracking and its measured advantage.
- The GTK4 launcher, AppImage, Nix packaging, MangoHud integration.
- `tools/fsr4cap` capture pipeline (Proton-based). The produced assets are portable.

## 6. Risks, in order

1. **TLS instruction coverage.** If the game or linked libc reaches TLS through any
   instruction the filter does not match, it faults. Mitigation: run Zydis over the whole
   image offline and list every `fs:` operand before writing any runtime code.
2. **Placeholder semantics.** `MapViewOfFile3` is strict about placeholder boundaries;
   batch-map calls that split and merge ranges are the most likely source of subtle bugs.
   Mitigation: port shadPS4's sequence verbatim and unit-test it (phase 2 check).
3. **Performance of signal-based tracking** in streaming-heavy scenes. Mitigation: keep
   the unprotect window; consider coarser tracking granularity if needed. Accept that
   Windows may trail Linux here.
4. **NVIDIA driver behaviour** under the two-stage draw pipeline and the Vulkan recording
   thread. Nothing in the code is AMD-specific, but all tuning was on RADV. Mitigation:
   Phase 0 and 4 on real NVIDIA hardware, `BB_DRAW_PIPE=0` as a fallback toggle.
5. **Windows version.** `MapViewOfFile3`/`VirtualAlloc2` need Windows 10 1803+; large
   placeholder reservations are slow before Windows 11 23H2 (shadPS4 caps the user range
   for that reason). State the minimum clearly.

## 7. NVIDIA-specific notes

- FSR 3.1: no extra features; expected to work (one Linux user report on a GTX 1060).
- FSR 4 v07: needs `shaderFloat16`, `shaderInt8`, `shaderIntegerDotProduct`,
  `computeDerivativeGroupLinear`, extended storage image formats. Turing and newer expose
  all of these. The one reported attempt gave a black window on Linux/GTX 1060 (Pascal,
  which lacks some of them, so that report is not evidence against RTX). Test on RTX
  during phase 5; if it fails, capture one frame with `BB_CAPTURE_TRIGGER` and compare
  pass outputs against the AMD capture.
- FSR 4.1.1: unavailable; the menu already greys it out when the extension is missing.
- Live resolution `auto` rule excludes pre-Turing NVIDIA (`tools/gpu_capabilities.c:58`);
  keep that rule.
- Roadmap item "DLSS for NVIDIA users" becomes much more reachable on Windows, since the
  DLSS SDK is Windows-native; it would slot in where FSR 3.1 is called with the same motion
  vectors and jitter. Out of scope for the port itself.

## 8. Launcher

Do not port GTK4. Options, cheapest first: (a) `bbport.ini` plus a batch file for the first
build; (b) a Dear ImGui launcher reusing the renderer's existing ImGui integration (one
window, the same settings the in-game menu already exposes); (c) a small Win32 or Qt tool
later. (b) keeps everything in one codebase and one language.

## 9. Open questions to settle before phase 3

- Exact count and shape of `fs:` accesses in the prepared image (`link_libc.py` prints
  `fs->gs patched=N`; record N and verify with a full Zydis scan).
- Whether any guest-visible host object is currently allocated with plain `malloc`
  outside `attach()`/`new_thread()` (audit for §2.3).
- Whether the GPU library should be a DLL or linked statically into `bb-probe.exe`
  (static avoids the DLL boundary and `$ORIGIN` issues entirely).
