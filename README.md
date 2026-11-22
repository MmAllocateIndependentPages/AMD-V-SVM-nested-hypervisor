# svmb

windows kernel driver that flips your cpu into AMD SVM mode when it loads and uses that to mess with memory and syscalls. started as me poking at svm, turned into a sandbox i kept bolting things onto.

heads up this will bsod you. run it in a vm you don't care about.

## what it does

virtualizes every core the second the driver loads. identity-mapped NPT with 2M pages, mmio holes handled. then a pile of toys on top.

NPT hook thing that flips pages between execute and data so you can intercept kernel functions. tried it on NtCreateFile with a stolen prefix detour, worked first try which i did not expect.

CR3 stuff is three things. one, deny-writes thread object pages of watched processes so every schedule-in traps and you get a direction bit out of it. two, per core NCr3 so each core sees its own view of a process while that process is running on it. three, CR3 read spoof with an XOR key but that one's bare metal only, nested virt just swallows the intercept.

sysaudit is the one i had the most fun writing. X-denies the entry pages of MmCopyVirtualMemory, NtCreateFile, NtOpenProcess, NtReadVirtualMemory, NtWriteVirtualMemory. something calls one, NPF on the entry fetch, you grab caller pid and image and args, reopen the page, let the original instruction run. not one byte of guest memory gets touched. PatchGuard sees nothing because there's nothing to see.

on top of that a 1 second window rate table per (caller, target) with edge triggered alerts. trip a threshold and it can suspend the offender (reversibly) or kill it. off by default behind registry knobs because you really don't want this killing random processes at 2am while you're debugging something unrelated. ask me how i know.

mem_crypt is the weird one. dual frame split view. the target process reads and writes plaintext through its own NPT. the physical frame underneath is XOR keystream ciphertext. anyone else walks physical memory, anyone else's view of that page is ciphertext. the seal is two phases with the target frozen across the transition so you never catch half a seal. unload puts the plaintext back without needing to stash a key.

lock free breadcrumb ring for crash stuff. live IOCTL reads it, there's also an offline decoder that works on suspended RAM dumps from vmware. probably the single most useful piece of code in here just because of how many times it told me where things died.

svmbctl is the usermode cli. info, start, stop, storm, cr3 commands, exitprof, dbg, sysaudit, kdump, crumbs.

## what works where

vmware with nested virt on, that's all i've tested. haven't touched real AMD silicon yet. offsets hardcoded to windows 10 1903 (18362). test signing only, no microsoft signature. the IOCTLs don't check anything so anyone who can open the device can drive all of it. fine for a research box, obviously not something you'd ship.

## building

open svmb.sln in VS with the WDK installed, build x64. on the test machine `bcdedit /set testsigning on` and reboot.

the `build/` folder has a bunch of bat files i wrote for shoving the driver into a VM and pulling crash dumps back out. they're hardcoded to my paths, treat them as hints.

## docs

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) how it fits together
- [docs/MODULE_GUIDE.md](docs/MODULE_GUIDE.md) module system
- [tests/CRASH_DEBUG_LOG.md](tests/CRASH_DEBUG_LOG.md) round by round debug history, more fun to read than most of the code tbh
- [AGENTS.md](AGENTS.md) rules for when i had AI helping with bringup

## license

MIT

## last thing

like i said, run it in a vm you don't care about.
