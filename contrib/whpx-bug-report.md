## WHPX page-table shadow is never refreshed after a delivered #PF — reproducible macOS boot stall + HYPERVISOR_ERROR 0x20001 on VP recreation

### Summary

Two reproducible defects in the Windows Hypervisor Platform (WHPX) API, both found while
running an unmodified x86_64 macOS guest (xnu-8796.141.3.713.2~2) under QEMU (11.0.50) on
Windows 11 (AMD Ryzen 7 9700X):

1. **Shadow page-table staleness**: after a page fault is delivered to the guest and the guest
   installs the missing mapping (writes the PDPTE in its page table), the hypervisor keeps
   faulting on the same address forever. Reading the guest's physical page table from the
   host side (via QEMU's RAM view) shows the mapping IS present, yet WHvRunVirtualProcessor
   still exits with WHvRunVpExitReasonException / PageFault on that address. The guest's
   fault handler sees the mapping and returns success, the instruction retries, and the
   hypervisor faults again — an infinite loop that stalls boot for 15-20 minutes (the guest
   occasionally gets past it only by luck).

2. **HYPERVISOR_ERROR 0x20001 on VP recreation**: destroying and recreating the virtual
   processor (WHvDeleteVirtualProcessor + WHvCreateVirtualProcessor on the same VpIndex, with
   all registers saved and restored) reliably crashes the whole host with bugcheck
   0x00020001 HYPERVISOR_ERROR, parameters (0x11, 0x3578c0, 0x1005, 0xffffe70000c04dc0).
   This happened twice (minidumps 090326-10984-01.dmp / 090326-9484-01.dmp).

### What does NOT refresh the shadow (all tried, none worked)

- Writing CR3 back with the same value via WHvSetVirtualProcessorRegisters (optimized away).
- Writing CR3 with a toggled PCD bit and back (a real value change — still no refresh).
- Host-side repair of the guest page table (the mapping is already there; the shadow just
  never sees it).
- A periodic host-side CR3 rewrite timer (same-value writes are ignored).

### Repro steps

1. Windows 11 + QEMU built with WHPX support, partition created with
   WHvPartitionPropertyCodeExceptionExitBitmap including WHvX64ExceptionTypePageFault.
2. Boot any x86_64 macOS (13.x) guest. During early boot (IOKit kext loading,
   OSKext::addKextsFromKextCollection copies kext data into freshly allocated kalloc pages),
   the first write to each new page faults; the guest installs the mapping; and the same
   address keeps faulting in a loop.
3. Evidence captured per fault: host-side walk of the guest page table (CR3 from
   WHvGetVirtualProcessorRegisters) shows the PDPTE present (e.g. 0x1f56c6027, a valid 2MB
   page), while the hypervisor still reports #PF with CR2 = that address.

### Impact

- Any guest that faults-and-maps pages during boot (macOS, and likely any OS) can stall for
  tens of minutes under WHPX, or loop indefinitely.
- The only obvious workaround (recreating the VP) crashes the host, so there is currently no
  safe way for a WHPX consumer to recover from the stale shadow.

Reported from a QEMU/WHPX investigation; full diagnostic history is tracked in QEMU GitLab
work item 4385 (https://gitlab.com/qemu-project/qemu/-/work_items/4385).

