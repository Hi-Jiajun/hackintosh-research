# Crash register evidence (two boots, after "WHPX: Unexpected VP exit code 4")

Both dumps taken via QMP `human-monitor-command info registers` while the VM
was paused by the UnrecoverableException handler. VM status: paused, 16 vCPUs.

## Boot 1 (stock tree + 4 upstream whpx patches)

- RIP=ffffff800d10a380  CR2=000000000000009e  CR3=00000000118ae000
- CR4=00000020  EFER=0000000000000d00  RFL=00000086 (IF=0)
- IDT=fffff6af00083000  GDT=fffff6af00039000
- RSI=000000000000009e  (== CR2)
- GS.base=ffffff800dcb6940

## Boot 2 (plus local CPUID tsc-deadline/arat experiment)

- RIP=ffffff801650a380  CR2=000000000000009e  CR3=000000001acae000
- CR4=00000020  EFER=0000000000000d00
- (identical CR2; RIP/CR3 differ only by ASLR/allocator offsets)

## Interpretation

Deterministic page fault on address 0x9e during the guest kernel's earliest
boot phase (boot IDT active, no PCIDE, no NXE, interrupts disabled), before
the first console character is drawn. Guest framebuffer dump confirms only the
boot logo is present (1.47 % non-black pixels, 0 % white).
