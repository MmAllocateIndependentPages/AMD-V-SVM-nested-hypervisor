# svmb

AMD-V (SVM) nested hypervisor research driver for Windows. Virtualizes every core at DriverEntry and uses the hypervisor position to implement NPT memory virtualization, page hooks, CR3 protection, syscall sensing, memory encryption, and crash forensics.

## Features

### Virtualization
- All-core SVM at DriverEntry with single-phase unload.
- NPT identity map across `[0, RamEnd)` with 2M RWX large pages, MMIO holes handled.
- Configurable TLB flush modes (`TlbMode=0` flush-all per VMRUN, `TlbMode=1` never flush).

### NPT Page Hooks
- Execute/data view transitions per 4K page.
- Stub + callback layers for managed hook registration.
- Stolen-prefix ASM detour; works on real kernel exports (e.g. `NtCreateFile`).

### CR3 Protection Suite
- **Sentinel sensing:** NPT W-deny on thread-object pages of watched processes; one trap per schedule write with direction bit.
- **Per-core process views:** publishes a per-process NCr3 on the core while the target is current (base for policy-driven hiding).
- **CR3 read spoofing:** XOR-key disguise. Bare-metal only, nested virt does not forward the intercept.
- **Region write protection:** `cr3 protect/unprotect/holdpage/regions` combining L1 guest-MM blocking with L2 NPT sensing; MDL-bypass probe; shadow-kick wiggle for deterministic single-exit sensing; dead-process sweep.

### sysaudit (syscall / exec sensing)
- X-denies entry pages of `MmCopyVirtualMemory`, `NtCreateFile`, `NtOpenProcess`, `NtReadVirtualMemory`, `NtWriteVirtualMemory`.
- SSDT + SSN anchored, multi-anchor validated.
- Entry-fetch NPF records caller pid, image, and arguments.
- Page is reopened and the original instruction re-executes natively.
- 250 ms timer re-arms deny.
- Zero guest bytes modified by construction (PatchGuard-invisible).

### Behavior Analytics
- 1 second window per-(caller, target) cross-process call rate tables.
- Edge-triggered alerts; thresholds set via registry knobs.
- Live RD / WR / OP tables.

### Alert Response
- Reversible suspend of offending processes.
- Deny-list.
- Kill-time live-EPROCESS image re-verify.
- Off by default (`BehaveResponse` knob).

### mem_crypt (memory-block encryption)
- Dual-frame split view: each protected block page maps to a private plaintext copy on the target's NPT view; the original frame holds XOR-keystream ciphertext.
- Target reads and writes plaintext; every other observer (physical walk, scanners, other processes) sees ciphertext.
- Two-phase seal gated by an exit-generation fence, target frozen across the transition.
- Key-free decrypt and unload-time restore.
- Per-core view sync via intercepted CPUID at block-access points (see `tests/mcrypt_target.c`).

### Crash Forensics
- Lock-free breadcrumb flight recorder.
- Live IOCTL reader + offline suspended-RAM decoder (`tools/decode_vmss_crumbs.py`).
- kdump helper.

### Module System
- Function-table API for pluggable modules.
- Shipped modules: debugger, cr3_monitor, cr3_seed, sysaudit, stealth, mem_read, mem_crypt, npt_hook_mgr, dbg_events, demo_cpuid.

### Control Tool (`svmbctl`)
- `info`, `start`, `stop`, `storm`
- `cr3 protect/unprotect/holdpage/regions/probe`
- `exitprof`
- `dbg` commands
- `sysaudit` config and readout
- `kdump`, `crumbs`

## Scope & Status
- Target: Windows 10 1903 (build 18362). Offsets are hardcoded to that build.
- Tested under VMware with nested virtualization enabled.
- Real AMD hardware not yet verified.
- No Microsoft signature; test-signing certificate only.
- IOCTLs have no permission model.

## Build
Open `svmb.sln` in Visual Studio with the WDK installed, build x64. Enable test-signing on the target (`bcdedit /set testsigning on`). The `build/` directory contains deployment and crash-pull scripts used during development.

## Documentation
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- [docs/MODULE_GUIDE.md](docs/MODULE_GUIDE.md)
- [docs/MEM_CRYPT.md](docs/MEM_CRYPT.md)
- [docs/HYPERVISOR_MEM_READ.md](docs/HYPERVISOR_MEM_READ.md)
- [docs/L2_BLOCKING_DESIGN.md](docs/L2_BLOCKING_DESIGN.md)
- [docs/SECURITY_AUDIT.md](docs/SECURITY_AUDIT.md)
- [tests/CRASH_DEBUG_LOG.md](tests/CRASH_DEBUG_LOG.md)

## License
MIT. See [LICENSE](LICENSE).
