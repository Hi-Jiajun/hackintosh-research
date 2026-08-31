# Windows host port (WSL/Windows + WHPX + native Vulkan)

This series enables the x86 macOS / host-Vulkan pathway on Windows hosts
(QEMU on Windows + WHPX + a native Vulkan ICD such as the NVIDIA driver).

## What works (verified on this machine)

- QEMU fork builds on Windows (MSYS2 MINGW64): qemu-system-x86_64.exe with
  tcg + whpx accelerators and the reims-vgpu-pci device linked against the
  Vulkan staticlib.
- The Rust staticlib compiles for x86_64-pc-windows-gnu (backend-vulkan,
  host-window) after the lib.rs platform gate is widened.
- The device initializes on the NVIDIA native driver: vk_caps reports
  api=1.4, host_pointer_import=supported, swapchain 1920x1080, first frame
  presented.
- With the GOP option ROM attached (boot-windows.sh), the guest firmware
  installs the reims EFI GOP and the boot loader proceeds to kernel handoff
  under both TCG and WHPX.

## Patch list

- 0001 lib.rs: allow backend-vulkan on target_os = "windows" (the comment in
  the gate says a new port is a deliberate edit to this list — this is it).
- 0002 context.rs: add the VK_KHR_win32_surface platform arm next to the
  macOS/Linux arms.
- 0003 present.rs: Windows refuses a non-main-thread winit event loop unless
  the app opts in; add the EventLoopBuilderExtWindows::with_any_thread
  opt-in, mirroring the existing X11/Wayland arms.
- 0004 qemu-build.sh: the ninja target and QEMU_BIN path carry a .exe suffix
  on MINGW/MSYS hosts.
- 0005 (qemu submodule) meson.build: Windows link arm for the Rust staticlib
  (winit/ntdll/ws2_32 system libs; -lm/-ldl do not exist on mingw).
- 0006 (qemu submodule) reims-vgpu-pci.c: guard the packed-view mmap alias
  with #if !defined(_WIN32) and return -1 on Windows — scattered GVA mappings
  take the copying rails, which the Rust side already treats as the fallback.
- boot-windows.sh (new): Windows boot script — WHPX/TCG selection, the
  standard pcibridge + GOP romfile attach, QMP over tcp (no unix sockets on
  Windows), SSH forward 2222.

## Not in this series (known follow-ups)

- The early-boot console rail on Windows: host_window_capture reports
  window_capture_unsupported; the GOP framebuffer path works, the
  capture-based console feed does not.
- A separate QEMU whpx issue (WHPX: Unexpected VP exit code 4) blocks the
  guest kernel from completing boot under WHPX; evidence package is in
  contrib/qemu-issue-bugB.md. TCG boots past the boot loader.

## Test notes

- Build: MSYS2 MINGW64 with PYTHONUTF8=1 (non-ASCII Windows locales).
- Rust deps for windows-gnu resolved via cargo registry (TUNA sparse mirror
  used in China); metal2vulkan stayed a git dependency in the WSL tree.
- The WSL/Linux KVM pathway is unaffected: cargo check on Linux passes with
  the same changes.
