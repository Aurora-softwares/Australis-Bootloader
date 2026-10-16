# Bootloader completion criteria

This checklist was audited against `program.hy`, the self-hosted UEFI image
emitter, the documented `KernelBootInfo` ABI, and the AHCI/NVMe QEMU boot
tests.

The bootloader stops at the kernel handoff. The kernel owns recovery through
KMon and starts `/System/Core/System.exec` as PID 1 on normal boot; neither
service startup nor the userland shell belongs in the bootloader. See
`../README.md` and `../userland/system/README.md` for that path.

Status key: `🟢` implemented and exercised, `🟡` partially complete, and
`🔴` not yet implemented or verified.

## Stage 1 - Minimum Viable Bootloader

| | Requirement | Evidence |
| --- | --- | --- |
| 🟢 | UEFI support | `BOOTX64.EFI` boots under x86-64 OVMF from optical and disk images. |
| 🟢 | Kernel discovery | Opens `\EFI\AUSTRALIS\KERNEL.BIN` through the boot volume's Simple File System protocol. |
| 🟢 | Kernel loading | Reads the complete image into `EfiLoaderCode` memory and rejects short reads. |
| 🟢 | Memory allocation | Allocates the image, a 64 KiB kernel stack, boot information, page tables, heap pages, and reserved DMA pages. |
| 🟢 | Kernel entry point | Validates the AUKR v1 header, exact code length, and entry offset `16`. |
| 🟢 | Boot information | Passes the versioned `KernelBootInfo` pointer to `KernelMain.Run` in `RDI`. |
| 🟢 | UEFI handover | Captures the final map and retries `ExitBootServices` with a fresh map key when required. |
| 🟢 | Execution transfer | Switches to the dedicated stack and calls the position-independent raw kernel entry. |
| 🟡 | Error handling | Firmware load, allocation, file, image-validation, memory-map, and GOP failures reach visible error paths. Missing and malformed kernel-image QEMU checks remain to be added. |
| 🟢 | Debug output | `D` enables detailed boot diagnostics; firmware-stage failures are always reported over the UEFI console. |

**Stage 1 handoff result: complete on the positive path.** The handoff reaches
the kernel and then the userland shell on both AHCI and NVMe QEMU
configurations. Negative image and firmware-failure injection remain open.

## Stage 2 - Feature Complete Bootloader

| | Requirement | Current evidence or remaining work | Priority |
| --- | --- | --- | --- |
| 🟢 | UEFI boot | Emits a PE32+ EFI application with the Microsoft x64 UEFI entry ABI. | Essential |
| 🟢 | Kernel loading | Validates AUKR magic, version, size, code length, and entry offset before handoff. | Essential |
| 🟢 | Memory map | Preserves map address, byte length, descriptor size, and descriptor version. | Essential |
| 🟢 | Memory allocation | Uses firmware loader allocations before handoff, then only conventional-memory descriptors afterwards. | Essential |
| 🟢 | Video initialization | Captures GOP geometry and pixel format before handoff and renders directly afterwards. | Essential |
| 🟢 | Boot information | Uses the `AUBI` record at ABI version 7 with a documented field layout. | Essential |
| 🔴 | ACPI discovery | Pass the ACPI RSDP location when available. | Recommended |
| 🔴 | Kernel parameters | Support configurable kernel arguments. | Recommended |
| 🔴 | Boot configuration | Read boot settings from a configuration file. | Recommended |
| 🟡 | Debug mode | `D` selects detailed AUKR, GOP, and handoff diagnostics and passes the selected mode to the kernel; automated mode checks are still needed. | Essential |
| 🟢 | Silent mode | The default with no key is silent; `S` also selects it explicitly. It suppresses optional bootloader diagnostics and early framebuffer status lines while retaining errors and the interactive console. | Essential |
| 🟡 | Normal mode | `N` restores early framebuffer status lines; automated mode checks are still needed. | Essential |
| 🟡 | Error handling | Missing and malformed AUKR images and unavailable GOP have specific firmware error paths; negative-image QEMU checks are still needed. | Essential |
| 🟢 | UEFI exit | Retries final map capture and exits with the current map key. | Essential |
| 🟢 | Kernel handover | The documented contract passes `KernelBootInfo` in `RDI` on a dedicated stack. | Essential |
| 🟢 | Boot device support | Hybrid ISO and disk boot paths pass under OVMF with the supported AHCI and NVMe profiles. Physical USB and internal-drive validation is a production gate. | Essential |
| 🔴 | Multiple architectures | Support ARM64 or additional architectures. | Future |
| 🔴 | Secure Boot | Support signed bootloader binaries and a trusted boot chain. | Future |
| 🟢 | Recovery support | Pressing `R` passes Rescue mode to the kernel and enters the `rescue>` monitor before controller, GPT, HyFS, USB, or AUEX startup. | Optional |

**Stage 2 essential result: validation in progress.**
`make test-rescue-boot` exercises `R` with an EFI-only disk and no HyFS
partition. The normal serial boot tests cover default boot on AHCI and NVMe.
Automated `D`, `N`, `S`, malformed-image, and unavailable-GOP checks remain.
ACPI discovery, kernel parameters, file-based configuration, and physical-media
validation remain recommended or production work.

## Stage 3 - Production Ready

Before considering the bootloader finished for a stable Australis release:

All rows are release requirements. Critical items protect boot correctness or
memory safety; High items provide the remaining validation and resilience.

| | Requirement | Evidence | Priority |
| --- | --- | --- | --- |
| 🟡 | Boot reliability across supported UEFI systems | AHCI and NVMe QEMU boots pass; multiple physical UEFI systems remain untested. | Critical |
| 🔴 | Physical cold boot and restart consistency | Repeated cold boots and restarts have not been verified on physical hardware. | High |
| 🟡 | Kernel image validation and integrity | Size, AUKR header, version, code length, exact read, and entry offset are checked. AUKR v1 has no payload checksum, so altered code with a valid header is not detected before execution. | Critical |
| 🟡 | Graceful firmware and hardware failure handling | Missing GOP and invalid kernels are covered; allocation and final handoff failure injection remains incomplete. | Critical |
| 🟢 | Independence from UEFI boot services after handoff | The kernel uses the captured memory map and GOP information after `ExitBootServices`; QEMU handoff boots pass. | Critical |
| 🟢 | Versioned boot compatibility | The loader validates AUKR v1 and passes the documented `KernelBootInfo` ABI v7 record. | High |
| 🟡 | Automated loading and handoff tests | Positive AHCI/NVMe paths pass. Dedicated negative-image and failed-`ExitBootServices` injection targets remain to be added. | High |
| 🔴 | No critical boot or memory corruption defects on supported hardware | The physical hardware matrix has not been tested sufficiently to make this claim. | Critical |
| 🔴 | Handoff validation | Validate the handoff on physical firmware and controller hardware. | Essential |
| 🔴 | qemu handoff injection | Inject post-load handoff failures in QEMU. | Recommended |

**Stage 3 result: not complete.** The remaining release gates are failure
injection, an integrity-bearing kernel image format, and validation on physical
UEFI machines. File-based boot configuration and ACPI discovery remain
recommended Stage 2 work.

## Completion definition

The bootloader is complete for a stable Australis release when it can reliably
discover, validate, load, and execute an Australis kernel on supported hardware,
provide all required boot information, handle failures gracefully, and leave
the firmware environment cleanly.
