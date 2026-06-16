# WinUAE patch — Vortex AT-Once Plus (286 board) emulation

This patch adds emulation of the **Vortex AT-Once Plus**, a 1991 Intel 80286
emulator board that plugs into the Amiga 500 68000 socket, to **WinUAE**.

It is the practical result of the *Dueottosei* reverse-engineering project: the
board's Xilinx XC2018 LCA (bridge/chipset) was decoded down to the CLB level and
re-expressed as formally verified RTL, the x86 BIOS and the Amiga 68k driver were
disassembled, and the resulting model was implemented on top of WinUAE's existing
PCem-based x86 core. 

## Baseline

The patch is generated against **tonioni/WinUAE `master`**, commit
`f08ee5e` ("Fix AMD LANCE chip ID byteswap"). It applies cleanly to that tree
with `git apply` (verified on a clean checkout).

## What the board is, in one paragraph

The AT-Once Plus carries an 80C286-16, an optional 80C287, 512 KB of local DRAM
and a single Xilinx XC2018 LCA. There is **no PC support chipset at all**: every
`IN`/`OUT` the 286 executes (except the 287 range 0xF8–0xFF) freezes the CPU, and
the Amiga-side 68k driver emulates the requested device in software, then resumes
the 286. The LCA is the bus bridge: it translates addresses 286→Amiga, does the
byte-swap (big-endian/little-endian) in copper, generates the I/O traps, and
exposes two "magic" registers in CIA space through which the 68k driver drives
everything (reset, A20, MEMMODE, interrupt delivery, trap readback/resume).

## Files in the patch

New files:

| File | Purpose |
|---|---|
| `atonce.cpp` | The board model: magic ports, 286↔Amiga address translation (ported from the verified RTL `atplus2_core.v`), I/O trap semantics, execution scheduling, A20 gate, video-refresh bridge, HDD bypass. |
| `include/atonce.h` | Public hooks (`atonce_init`, `atonce_cia_access`, `atonce_intr_vector`). |

Modified files (minimal hooks, all guarded by `#ifdef WITH_X86` / `#ifdef UAE`):

| File | Change |
|---|---|
| `cia.cpp` | Call `atonce_cia_access()` first in the CIA byte/word get/put paths: the board sits in the 68000 socket, so it decodes the magic ports before Gary. |
| `x86.cpp` | When no PC bridgeboard owns the x86 side (`bridges[0]==NULL`), route `portin/portout` (8/16/32-bit) to `atonce_x86_io()` (the 286 I/O trap). |
| `pcem/pic.cpp` | `picinterrupt()` returns the LCA-supplied vector (`atonce_intr_vector`) — there is no 8259 on the board. |
| `expansion.cpp` | Register the board in the expansion list as **"ATonce Plus" (Vortex)**: non-autoconfig, no BIOS ROM file (the x86 BIOS is loaded from the floppy `.dsg` into Amiga chip RAM by the 68k driver). |
| `include/rommgr.h` | Allocate `ROMTYPE_ATONCE`. |
| `od-win32/winuae_msvc15/winuae_msvc.vcxproj` | Add `atonce.cpp` to the MSVC project. |

The board reuses the PCem x86 core already present for the A1060/A2088/A2286/A2386
bridgeboards and is therefore **mutually exclusive** with those boards (PCem is a
single instance).

## How to apply

From the root of a clean WinUAE checkout at the baseline commit:

```sh
git checkout f08ee5e
git apply /path/to/atonceplus.patch
```

Then build as usual (the MSVC project already lists `atonce.cpp` after the patch).
On non-MSVC build systems, add `atonce.cpp` to the source list manually.

## How to use

1. Enable the **"ATonce Plus"** board in WinUAE's expansion settings (it appears
   among the x86 bridgeboards; remember it is mutually exclusive with them).
2. Provide the original AT-Once install floppy/floppies (in `Disk/` in the project
   repo): the 68k driver `AT-Emulator/atonce` configures the board and boots the
   286. No PC BIOS ROM file is needed — the x86 BIOS lives in the floppy `.dsg`.
3. A PC hard disk can be served from a WinUAE hardfile that contains a PC
   partition (MBR); the model auto-detects it and bypasses the 68k disk path.

The reference software is driver **v2.20** (the model based on it also covers
v3.00: the hardware/protocol contract is byte-identical between the two

## Status and known limitations

Working: driver boot, BIOS POST (`OUT 0xBB` beacon), DOS prompt, keyboard, text
video, hard disk (hardfile bypass), extended memory / himem.

- **Performance**: driver v3.00 runs slower than v2.20 (a heavier, data-dependent
  device-tick iteration in v3.00); under investigation.
- **Extended memory**: the >1 MB window is currently backed by a private flat
  buffer (`ATONCE_ISOLATE_EXT`) so himem.sys/HMA code fetch works; the "real"
  MEMMODE mapping onto Amiga fast/slow RAM (4/2/1 → 0x080000/0x200000/0xC00000)
  is decode-validated but not yet exercised at runtime (the `#else` path exists).
- **287/FPU**: `FPU_NONE`; the optional 287 + error path is not wired yet.
- **Windows 3.1**: triple-fault loop tied to the 286 protected→real-mode switch
  via reset (`LIDT null; INT 3` → shutdown → CMOS warm path), currently handled
  as a cold reset.

## Provenance / correctness notes

- The 286→Amiga address translation (`atonce_rebuild_map`) is computed directly
  from the CLB equations of `atplus2_core.v`, the cleaned RTL that was **proven
  equivalent** to the golden LCA netlist with yosys, and the regenerated bitstream
  was **validated on real hardware**.
- The sub-1 MB exec-pointer window table (`atonce_win_map`) uses the LCA hardware
  values for banks A0000/C0000 (chip 0x20000), not the `.dsg` software-translation
  aliases (0x60000/0x00000); both banks are unused, so this is a fidelity choice
  (the emulator models the hardware). 
