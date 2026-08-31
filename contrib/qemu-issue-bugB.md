# WHPX: guest receives "Unexpected VP exit code 4" (UnrecoverableException) during earliest kernel boot — VM stopped

## Summary

An x86_64 guest that boots fine under KVM and HVF (same machine model, same
firmware, same disk image) dies during its earliest kernel initialization when
accelerated with WHPX on a Windows 11 host. QEMU logs
`WHPX: Unexpected VP exit code 4` and pauses the VM
(`vm_stop(RUN_STATE_PAUSED)`), leaving no information about the underlying
guest exception.

## Environment

- Host: Windows 11 (24H2 + April 2025 optional updates), AMD Ryzen 7 9700X
  (16 threads), 32 GB RAM, NVIDIA RTX 5060
- "Windows Hypervisor Platform" optional feature: Enabled
- QEMU: 11.0.50 (development build from the reims-vgpu fork of QEMU,
  base commit e17ddb98, plus upstream whpx patches 7c847523 "fix xsaves
  enablement in legacy probing path", 37ce6d8d "work around Hyper-V FP state
  oddities", 7b051691 "synchronise PAT too", 3bfcd8ef "enable fast hypercall
  output", plus a local CPUID experiment described below)
- Guest: x86_64 OS (macOS 13.7.8) booted via OVMF + OpenCore
- CPU model: `-cpu Skylake-Client,-hle,-rtm,vendor=GenuineIntel,+invtsc,
  vmware-cpuid-freq=on,+ssse3,+sse4.2,+popcnt,+avx,+avx2,+aes,+xsave,+xsaveopt,check`
- Command line (abridged): `qemu-system-x86_64.exe -accel whpx -m 16G
  -machine q35 -smp 16 ...` (full details in evidence file)

## Reproduction

1. Boot the guest with `-accel whpx`. Firmware and boot loader complete
   normally; the guest kernel takes over.
2. Within seconds the guest hits the failure; QEMU prints
   `WHPX: Unexpected VP exit code 4` and the VM shows STOPPED.
3. The same guest on the same host with the Linux build of the same QEMU tree
   under KVM boots to a working desktop; under TCG it also proceeds past this
   point (slowly). The failure is specific to the WHPX backend.

## Observed behavior

- `WHPX: Unexpected VP exit code 4` (WHvRunVpExitReasonUnrecoverableException),
  then QEMU pauses the VM; the window title shows QEMU (Stopped).
- QMP register dump taken after the stop (two independent boots; only
  ASLR-derived addresses differ — the fault is deterministic):
  - `CR2=0x000000000000009e` (page fault on a near-NULL address, both boots)
  - `CR4=0x00000020` (PCIDE not yet enabled)
  - `EFER=0x0000000000000d00` (NXE not yet enabled)
  - boot-stage IDT/GDT bases, IF=0
  - Full dumps in evidence/registers.md
- Guest framebuffer at the moment of the stop contains only the boot logo
  (~1.5 % non-black pixels, zero white pixels): the guest died before its
  first console output.
- CPUID observations: `WHvGetVirtualProcessorCpuidOutput` hides
  `tsc-deadline`, `arat`, and `pcid` from the guest on this AMD host.
  Forcing `tsc-deadline`+`arat` on in `whpx_get_supported_cpuid`
  (mirroring the existing unconditional X2APIC/HT bits) removes the QEMU
  "host doesn't support requested feature" warnings but does not change the
  crash signature.
- `-accel whpx,ssd=off` does not change the crash.
- Applying the four upstream whpx patches listed above (including the xsaves
  legacy-probing fix, which removed the xsavec warning) does not change the
  crash.

## Expected behavior

Either the guest boots (as it does under KVM), or QEMU reports *which* guest
exception the hypervisor refused to deliver. A bare exit-code line plus a
pause is not actionable: the UnrecoverableException case in
`whpx-all.c` should dump the available VP context (instruction pointer,
exception type if exposed by WHvGetVirtualProcessorState, etc.) before
stopping, and ideally inject the exception into the guest instead of pausing.

## Additional context

- Same guest + same QEMU tree on Linux: KVM boots to desktop, TCG proceeds.
- The CPUID feature filtering behavior described above may be independently
  worth review: bits Hyper-V hides on AMD hosts are not distinguishable from
  bits the hypervisor genuinely cannot provide, and QEMU's own LAPIC emulation
  handles TSC-deadline regardless.
