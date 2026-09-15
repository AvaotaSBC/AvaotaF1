# Firmware for Avaota F1

## RustSBI Image

[RustSBI](https://github.com/rustsbi/rustsbi) is a Rust implementation of the
RISC-V Supervisor Binary Interface (SBI). It runs below the operating system
and provides services such as timers, inter-processor interrupts and system
reset. This firmware replaces the factory OpenSBI in the Avaota F1 boot chain
while retaining the factory U-Boot and Linux system.

[`rustsbi-0.4.1-uboot-toc1.bin`](rustsbi-0.4.1-uboot-toc1.bin) is a **256 KiB TOC1 boot package**,
including the RustSBI loader and its DTB, RustSBI, and the unchanged factory
U-Boot. It is the exact boot package used in the experiment, renamed for
distribution. It is neither a standalone SBI executable nor a full flash image
or Phoenix image. The standalone RustSBI executable inside it is 176,128 bytes.

| Property | Value |
|---|---|
| Board | Avaota F1, Allwinner V821, SPI NOR |
| File size | 262,144 bytes (`0x40000`) |
| NOR write offset | `0xc000` |
| End of write window (exclusive) | `0x4c000`, immediately before GPT |
| Build | One hart, 16 KiB stack, F1 driver selection, RV32IMAFDC + XAndesPerf, V disabled |
| Optimization | `opt-level=s`, LTO, one codegen unit |
| Logging | `log_level = "WARN"`, `log/release_max_level_warn` |

The factory TOC1 occupies `0xc000..0x3c000`. This package also uses the factory
image's erased gap at `0x3c000..0x4c000`. BOOT0 follows TOC1's valid length;
the factory updater's original `0x30000` slot limit cannot accommodate this
package. Do not write it at address zero or use that updater's old slot size.

## Flash with rfel or xfel

Use [rfel](https://github.com/rustsbi/allwinner-hal/tree/main/rfel) or a
[V821-capable xfel](https://github.com/xboot/xfel/blob/main/chips/v821.c).
The experiment used rfel; the xfel commands below follow its
[SPI NOR command interface](https://github.com/xboot/xfel/blob/main/main.c)
and have not been exercised in this experiment.

1. Disconnect other FEL boards. Hold the F1's FEL button while connecting its
   USB data port and powering it on, then release the button.
2. Run the identification commands below. Confirm that the chip is V821 and
   the detected storage is SPI NOR. Record its SID to distinguish this board
   from other connected devices.
3. Back up the first `0x50000` bytes and the entire `0x40000` TOC1 write window.
   Confirm the factory layout described above before proceeding. Keep these
   backups outside the repository.
4. Write the package, read it back and compare the hashes. Only after they
   match, power off and on normally **without holding FEL**.

Run from this `Firmware` directory. Choose one tool:

### rfel

```sh
rfel version
rfel sid
rfel spinor
rfel spinor read 0x0 0x50000 boot-region-backup.bin
rfel spinor read 0xc000 0x40000 toc1-backup.bin
rfel spinor write 0xc000 rustsbi-0.4.1-uboot-toc1.bin
rfel spinor read 0xc000 0x40000 toc1-readback.bin
```

When multiple devices must remain connected, use rfel's `--device <SELECTOR>`
option on every command to select the same V821 explicitly.

### xfel

```sh
xfel version
xfel sid
xfel spinor
xfel spinor read 0x0 0x50000 boot-region-backup.bin
xfel spinor read 0xc000 0x40000 toc1-backup.bin
xfel spinor write 0xc000 rustsbi-0.4.1-uboot-toc1.bin
xfel spinor read 0xc000 0x40000 toc1-readback.bin
```

Compare the readback with the distributed image on Linux:

```sh
sha256sum -c SHA256SUMS
cmp rustsbi-0.4.1-uboot-toc1.bin toc1-readback.bin
```

Or on Windows PowerShell, compare both hashes with each other and SHA256SUMS:

```powershell
Get-FileHash rustsbi-0.4.1-uboot-toc1.bin, toc1-readback.bin -Algorithm SHA256
Get-Content SHA256SUMS
```

UART0 at **115200 8N1** shows the boot log. Linux should report
`SBI implementation ID=0x4` for RustSBI (factory OpenSBI reports `0x1`).

To restore the previous firmware, enter FEL again, write `toc1-backup.bin`
to `0xc000` with the same tool, read back `0x40000` bytes and verify them against
the backup. Then power-cycle normally. No whole-chip erase is needed.

## Detailed experiment results

Measured with the factory Linux 5.4.220, U-Boot, Linux DTB,
root filesystem and boot arguments. RustSBI supplies its own loader DTB.
Only the TOC1 window was changed, and every flash operation was read back and
verified. Both firmware versions completed ten warm boots.

| Metric | OpenSBI mean | RustSBI mean | Change | OpenSBI SD | RustSBI SD |
|---|---:|---:|---:|---:|---:|
| Linux to `/init` (ms) | 1670.807300 | 1665.077500 | -0.34% | 1.436064 | 0.686141 |
| BOOT0 to Linux, host (ms) | 1285.700000 | 1162.500000 | -9.58% | 7.349225 | 8.181958 |
| BOOT0 to shell, host (ms) | 8907.600000 | 8746.900000 | -1.80% | 4.718757 | 9.631545 |
| `rdtime a0` (µs) | 0.525038 | 0.264952 | -49.54% | 0.000270 | 0.000288 |
| `rdtimeh a0` (µs) | 0.528543 | 0.268196 | -49.26% | 0.000079 | 0.000400 |
| `rdtime s2` (µs) | 0.524614 | 0.360180 | -31.34% | 0.000179 | 0.000115 |

SD is the sample standard deviation; a negative change means less elapsed time.

For each instruction on each boot, the same user-mode probe ran five batches
of 100,000 reads. The table averages the per-boot batch medians. For example,
`rdtime a0` reads the time counter into the caller-saved register `a0`, while
`rdtime s2` writes it into the callee-saved register `s2`; these exercise different
register-restoration paths. On this RV32 board, `rdtimeh` reads the upper word.
The firmware contained no handler timing probes.
