# Metal API emulator: upstream 69a57dd adaptation

This is the portable, text-only evidence summary for the local upstream
adaptation. The detailed local manifest remains outside this repository because
it includes machine-specific paths and deployment details.

## Identity

- Reims upstream base: `69a57dd69a6958e946c03b73e02db331f330f435`.
- Reims adaptation: `3f19c66c7af392d4b588430a07119142c5cea8bd`.
- Facade adapter: `metal-api-emulator@9c934cbf8a6a58724ca73bf4582ab6596c676349`.
- metal2vulkan dependency pin: `9e0e99a41dc3cb8bb7e288b531f1698a79fd4b1c`.
- Windows release executable SHA-256 (source and deployed copy):
  `b782b32e3d1586ae2a775c8990b6aa398a2c5b7596d4025d58e7dc39fc594968`.
- This text-only evidence summary is published in
  `Hi-Jiajun/hackintosh-research@2d5b3bb`; the emulator and reims source
  worktrees remain local.

## Verified

- Facade workspace tests: 20 passed.
- Reims upstream-69 all-targets Vulkan `cargo check`: passed.
- Focused reims tests: exact-thread 3, device-limits 3, explicit completion 1,
  typed validation 2; all passed.
- Linux/Lavapipe and Windows/RTX 5060 smoke passed for standalone and reims
  executors across textual, raw AIR, wrapped AIR and indexed boundary cases.

## Boundary

- The latest reims lib clippy run is blocked by two pre-existing upstream
  findings: `runtime/exec/mod.rs:3887` (`too_many_arguments`) and
  `backend/vulkan/engine/mod.rs:5725` (redundant closure). Neither is introduced
  or suppressed by the adaptation.
- No complete reims suite, VM/guest/display boot, or canonical production Metal
  rail integration was run.
- This is an off-VM compute provider seam smoke, not a conformance or release
  gate. No binary, AIR, MTLB, SPIR-V, VM image, log or credential is included.
