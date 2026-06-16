# How the Vortex AT-Once Plus works

*A description of how the board works. The first chapter is a plain, easy-to-read
overview of the essentials; the following chapters go into the details (hardware,
BIOS, driver), and the last chapter covers the WinUAE emulation. 

Reference configuration: **atplus2** (the LCA bitstream used at runtime).

---

## 1. In a nutshell (the plain overview)

The **AT-Once Plus** (Vortex, 1991) is a board that turns an **Amiga 500** into a
**286 PC**. It plugs into the Amiga's **68000** socket: the original CPU is not
thrown away, it is re-seated in a socket provided on the board itself. From that
point on, **two processors** share the Amiga's single bus: the usual 68000 and an
**Intel 80286 at 16 MHz** (with an optional **80287** math coprocessor).

The key point, and what makes it clever, is this: **the board contains no PC
chipset at all**. No disk controller, no graphics card, no PIC (*Programmable
Interrupt Controller*, the 8259 that on a PC collects and dispatches the hardware
IRQs to the CPU), no 8253 timer, no keyboard controller. None of the "support"
hardware a real PC has is present. In its place there is **a single programmable
chip** — a **Xilinx XC2018 (LCA)** FPGA — acting as a **bridge** between the two
worlds, plus 512 KB of RAM dedicated to the 286 and a little glue logic (two GALs,
latches and transceivers).

So how does the 286 "see" a disk, a screen, a keyboard? Through a very elegant
trick:

> **Every time the 286 executes an I/O instruction (`IN`/`OUT`) to a PC port, the
> LCA freezes it mid-instruction and hands over to the 68000, which emulates the
> requested device in software and then lets the 286 resume.**

In effect, **the Amiga's 68000 acts as the PC's chipset**: it emulates the disk
controller (using the Amiga's real floppy/hard disk), draws the PC's screen into
the Amiga's graphics memory, handles the keyboard, the PIC, the timer, and so on.
The 286 runs the actual x86 code (DOS, the applications) and believes it is on a
normal PC.

The three hardware/software pieces that create this illusion are:

1. **The LCA (XC2018)** — the bridge hardware: it translates addresses, swaps
   endianness, generates the I/O "traps", and manages reset, A20, interrupts and
   bus arbitration.
2. **The x86 BIOS** (`atonce.bin`) — runs on the 286: it is an ordinary AT BIOS,
   but instead of talking to a real chipset, every one of its services ends in an
   `OUT` that triggers the trap toward the Amiga.
3. **The Amiga 68k driver** (`atonce`) — runs on the 68000: it configures the LCA,
   starts the 286, then spins in a loop serving all of the 286's traps by
   emulating the PC devices.

The three parts communicate through **two "magic ports"** in memory (in the
Amiga's CIA space) and through a block of **shared RAM**. The rest of this
document covers each piece in detail.

---

## 2. Hardware: the board and the LCA

### 2.1 What is on the board

The board plugs into the 68000 socket (J1) and carries the original 68000 in its
own socket (J2). The 286 and the 68000 share one address/data bus through latches
and transceivers; the **LCA is the bus bridge, the sequencer and the PC-chipset
emulator**.

| Ref | Part | Role |
|---|---|---|
| U13 | N80C286-16 | The PC CPU, clocked 32 MHz on the CLK pin (16 MHz bus rate) |
| U14 | 80C287 | Socket for the optional FPU |
| U41 | **XC2018 (LCA)** | The programmable bridge chip (the heart of the board) |
| U21, U29 | 74HCT374 | Latch the 286 addresses A0–A15 onto the Zorro bus at cycle start |
| U24, U25 | 74F245 | Data transceivers, **wired byte-swapped**: the big-endian (68k) ↔ little-endian (286) conversion is done **physically in copper** |
| U26 | 74HCT374 | Port-address readback register for the I/O traps |
| U22 | GAL16V8 | Decodes the **two "magic" addresses** of the 68k, generates the config clock (CCLK) and the runtime strobes |
| U44 | GAL16V8 | 287 chip-select (ports 0xF8–0xFF) and 68000 bus arbitration |
| U31–U34 | TMS44C256 | **512 KB of onboard DRAM** (the 286's conventional RAM) |
| U35–U37 | 74F157/153 | Row/column MUX and bank selection for the DRAM |
| U42 | 74F125 | Tri-state drivers toward the 68000 socket (AS/UDS/LDS/DTACK) |
| U43, U45, U28, U15 | glue logic | Memory decodes (<512K / ≥1M), A20 gate, clocks |

### 2.2 The central point: no chipset, only the LCA

It is worth repeating, because it is the key to the whole project: **there is no PC
support hardware**. Every I/O cycle of the 286 — except those toward the 287
(ports 0xF8–0xFF, handled via GAL U44) — **is never completed by hardware**: the
LCA withholds the `READY` signal and the 286 **stalls inside the instruction**
until the 68000 serves it.

Everything this document says about the LCA does not come from a datasheet — the
XC2018 is an FPGA, its behaviour *is* the bitstream loaded into it. The
reconstruction of how it works is the result of the Dueottosei project's reverse
engineering: **extraction and decode of the bitstream** of the `atplus2`
configuration from the `atplus.dsg` file, **CLB-level mapping** of the programmed
logic (85 CLBs out of 100), and a rewrite into a "golden" **Verilog netlist**
(`xc2018/verilog/dueottosei/`) later cleaned into a **readable RTL**
(`xc2018/rtl/atplus2_core.v`) *formally proven equivalent* to the netlist with
yosys. The reconstructed bitstream was finally **regenerated and validated on real
hardware** (the board behaves like the original). The equations and the address
map in the following sections derive from this verified netlist, not from guesses.

### 2.3 Clocks and reset

The LCA has four internal clock domains:

- **32 MHz** (`P56`): the 286-side state machines (READY, DRAM controller, 287,
  sync).
- **7.14 MHz** (`P27`, the Amiga 68k clock): the 68k-side bus-cycle FSM and the
  refresh prescaler.
- **GCLK**: captures address/status at every 286 cycle start.
- **~112 kHz** (ripple chain from P27/64): the **refresh timer** of the onboard
  DRAM.

Reset is explicit and software-controlled: after configuration, the **286 is held
in reset** until the Amiga driver writes to magic port A to release it. Until the
driver activates it, the board stays inert (the LCA's **DONE** signal must be
asserted for the Amiga to continue booting).

### 2.4 The two magic ports

GAL **U22** decodes two addresses in the Amiga's CIA space (qualified by `/VPA`),
using in hardware **only the bits A6, A7, A15** (the remaining address bits are
don't-care, so the base is file-reconfigurable). The v2.20 driver uses:

- **PORT_A = 0xBFF041** — **byte** access (odd address → `/LDS`)
- **PORT_B = 0xBFF080** — **word** access

Because of the byte-swapped transceivers, the 68k data word maps onto the 286 data
pins with the bytes swapped (`Z_D0..D7 → 6_D8..D15` and vice versa): endianness is
resolved in hardware.

**PORT_A write** (mode register, 68k low byte):

| Bit | Function |
|---|---|
| D0 | Config DIN (each access pulses CCLK); at runtime = 286 reset pulse |
| D1 | 287 reset / enable **SETUP mode** (to load MEMMODE) |
| D2 | **A20 gate** (opens/closes the A20 line as on a PC) |
| D3 | 287 error/busy acknowledge |
| D4 | **DONE//PROG** control (reload the LCA configuration) |

**PORT_A read** (status byte):

- bit1 = **I/O trap pending** (the 286 is frozen inside an `IN`/`OUT`)
- bit0 = before the stop: **287 error**; after the stop: **direction** of the trap
  (1 = IN, 0 = OUT)

**PORT_B write** = **resume the 286**:

- if bit0 = 1, the high byte is an **interrupt vector**: it is loaded and `INTR` is
  asserted to the 286 (auto-cleared on INTA).
- in SETUP mode, each write **shifts one bit** into the MEMMODE register (see §2.5).

**PORT_B read** = **stop the 286** + readback of the port address of the last
trapped I/O cycle (10 bits + A0, byte-swapped).

### 2.5 MEMMODE: where the extended memory lands

The 286 has its onboard DRAM (512 KB) as low conventional memory, but the
**extended memory (>1 MB)** is mapped onto Amiga RAM, and *where* exactly is
programmable. The choice is a shift register inside the LCA, loaded in SETUP mode
by shifting bits in from PORT_B. For atplus2, **3 bits** are shifted (a 4-stage
register AH←BH←CH←DH):

| MEMMODE | Extended memory lands at | Amiga RAM type |
|---|---|---|
| 4 | 0x080000 | Agnus second 512K (1MB) |
| 2 | 0x200000 | Zorro fast |
| 1 | 0xC00000 | slow RAM |

### 2.6 The 286 → Amiga address translation

The LCA translates the 286 addresses A16–A23 into Amiga addresses. The sub-1MB map
is **fixed** (independent of MEMMODE):

| 286 range | Maps to (Amiga) | Note |
|---|---|---|
| 00000–7FFFF | **onboard DRAM** | never reaches the Amiga bus |
| 80000–9FFFF | chip 040000–05FFFF | upper conventional memory |
| A0000–AFFFF | chip 020000–02FFFF | unused alias |
| B0000–BFFFF | chip 010000–01FFFF | CGA/MDA video window |
| C0000–CFFFF | chip 020000–02FFFF | unused alias |
| D0000–DFFFF | chip 010000–01FFFF | hardware alias of B0000 |
| E0000–EFFFF | chip 020000–02FFFF | |
| F0000–FFFFF | chip 030000–03FFFF | **BIOS + shared mailboxes** |

In addition, the 286's onboard DRAM (its 512 KB) is exposed **to the 68000** in a
fixed window **0x900000–0x97FFFF**: this is how the Amiga driver reads/writes the
286's conventional memory (a comparator in the LCA cuts the strobes toward the
motherboard and serves the cycle from the onboard DRAM). This window is independent
of MEMMODE.

### 2.7 Bus arbitration and interrupts

*(Section to be deepened — only the facts verified in hardware/netlist here.)*

- The LCA requests the bus from the real 68000 through GAL U44 (`M_/BR`,
  `M_/BGACK`), merging the external Zorro DMA requests into the same signal: Amiga
  DMA stays active while the 286 runs.
- There is **no 8259 on the board**: an interrupt is delivered to the 286 by
  writing to PORT_B (bit0=1, vector in the high byte). When the 286 answers with
  the INTA cycle, the LCA puts that vector on the data bus and clears the `INTR`
  request.
- An Amiga **level-3** IRQ is decoded by the LCA and puts the 286 in **HOLD**
  (decode verified in the netlist; the exact scheduling policy is to be confirmed).

---

## 3. The x86 BIOS (`atonce.bin`)

`atonce.bin` is the **286-side BIOS**, loaded at **F000:2000**. It is an
IBM-compatible AT BIOS (it has the standard entries at the canonical offsets:
INT 10 at F000:F065, INT 13 at F000:EC59, etc.), but with one fundamental
difference: **it does not talk to a real chipset**.

### 3.1 The trap protocol (286 side)

Every PC service the Amiga has to emulate funnels into a stub of this exact shape:

```asm
pusha                     ; push all registers onto the stack
push  ds
push  es
mov   %ss, %cs:0x1402     ; publish SS:SP so the Amiga can find the register frame
mov   %sp, %cs:0x1400
out   %al, $0xBx          ; -> the LCA withholds READY, the 286 freezes HERE
jmp   .+2                 ; (pipeline flush after resume)
pop   es
pop   ds
popa                      ; registers possibly MODIFIED by the Amiga
iret
```

The mechanism is: the `OUT` is the trap; the LCA stalls the 286 and releases the
bus to the 68000; the Amiga driver reads the port number from PORT_B, reads SS:SP
from F000:1402/1400 to locate the `pusha` frame in shared RAM, performs the service
**editing the saved registers in place**, and resumes the 286, which returns the
results via `popa/iret`.

> Important consequence for emulation: **parameter passing happens entirely through
> shared RAM**. The only information carried by the `OUT` itself is the port number.

### 3.2 The shared RAM (F000 segment)

The **286's F000 segment = Amiga chip RAM 0x30000–0x3FFFF**. The part
F000:0000–1FFF is **shared scratch RAM** (mailboxes): the BIOS reads/writes
variables there and the Amiga writes there from the other side. Some known
locations:

| 286 address | Use |
|---|---|
| F000:1400 (word) | SP saved at trap time |
| F000:1402 (word) | SS saved at trap time |
| F000:140C, F000:1432 | status bytes returned by a private service (AL/AH) |
| F000:1470/1472/1474 (word) | values written **big-endian by the Amiga**, byte-swapped by the 286 with `xchg al,ah` before use |
| F000:15A0, 15B5, 15C4–15C6, 15E0 | video/mode state flags |
| F000:15E4 (10 bytes) | parameter block exchanged via `rep movsb` (private service AH=5) |
| F000:17B4, 17B8–17BD | drive flags/bitmap (32-bit installed-drive mask at 17B8) |
| F000:03B8, 03BF | configuration flags (written by the Amiga setup) |
| F000:+<port#> | **port shadow byte**: for several ports the last value lives here |

### 3.3 The Vortex service ports (0xB0–0xBF)

Besides the standard PC ports (PIC, PIT, 8042…, all emulated by the Amiga), the
BIOS uses a family of private ports **0xB0–0xBF** for high-level services (disk,
video, query, delay…). When the 286 does an `OUT` to one of these ports, it traps
toward the 68000 and triggers the corresponding handler in the driver. Complete map
(286 side from the BIOS, 68k side from the driver):

| Port | Service | Note |
|---|---|---|
| 0xB0 | hard disk services (INT 13 backend) | hard disk |
| 0xB1 | video render/update | screen update |
| 0xB2 | private service (BIOS AH=6) | |
| 0xB3 | no-op: plain resume | |
| 0xB4 | **fatal halt**: the Amiga runs `reset` and reboots | "the 286 gives up" |
| 0xB5 | private service (BIOS AH=7) | |
| 0xB6 | mode/param service via the `pusha` frame (BIOS AH=8) | |
| 0xB7 | timed delay (display off/on around the wait) | |
| 0xB8 | private service (BIOS AH=2) | |
| 0xB9 | disk control (BIOS AH=1) | HD presence query |
| 0xBA | video state-change notification | |
| 0xBB | **POST beacon**: first instruction of the POST → (re)enter the service loop | "BIOS alive" |
| 0xBC | private service (BIOS AH=13) | |
| 0xBD | private service (BIOS AH=10) | |
| 0xBE | set video mode | |
| 0xBF | `rts` — ignored | |

For the emulator the central point is: **the core does not need to know the
individual ports**. It only has to reproduce the *trap semantics* (freeze the 286
on any I/O outside 0xF8–0xFF and deliver the port number to the 68k protocol).
Everything else is implemented between the BIOS and the driver.

---

## 4. The Amiga 68k driver (`atonce`)

The `atonce` driver (an AmigaOS hunk executable) is the third piece: it runs on the
68000, configures the LCA, starts the 286 and serves all the traps. It is the real
"PC chipset", in software.

### 4.1 LCA configuration (PORT_A bit-banging)

The driver loads the LCA bitstream by writing it one bit at a time to PORT_A:
**each byte written to PORT_A = one configuration bit** (D0 = DIN, the write itself
pulses CCLK). Before reconfiguring, it pulses PORT_A bit4 (DONE//PROG) to return the
LCA to the unconfigured state. The bitstream is inside `atplus.dsg`: byte 0x14 of
the selected record chooses which of the two bitstreams to load (**byte 0x14 = 1 →
atplus2**).

### 4.2 The runtime loop (the heart of the emulation)

For each iteration of the main loop:

1. Run the **device ticks** (disk, video, keyboard, mouse, serial, sound…).
2. Disable interrupts, switch to a private stack.
3. If the emulated PIC has a pending unmasked IRQ, build `d2 = (vector << 8) | 1`;
   otherwise `d2 = 0`.
4. **Resume / IRQ delivery**:
   ```
   move.b $1404(a5), (a4)   ; PORT_A <- mode byte (A20 gate, etc.)
   move.w d2, $3f(a4)       ; PORT_B write: resume the 286 (+ optional INTR)
   ```
5. **Poll**:
   ```
   move.b (a4), d3          ; PORT_A status, pre-stop
   move.w $3f(a4), d0       ; PORT_B read -> STOPS the 286 and latches the trap
   move.b (a4), d1          ; PORT_A status, post-stop
   ```
6. Status decode: pre-stop bit0 → raise **IRQ 13** (287 error); pre-stop bit1 →
   trap pending, go to dispatch; post-stop bit0 → direction (IN/OUT).
7. **Dispatch**: fast-path for video status reads (IN 0x3DA/0x3BA: toggle the
   retrace bits in the shadow and resume immediately); otherwise
   `handler = table[port >> 4]` from two tables (OUT at +0x1D74, IN at +0x1E74).
8. The handler locates the 286's `pusha` frame (reads SS/SP, translates to an Amiga
   address), **edits it in place**; `popa/iret` in the BIOS stub completes the
   service.

### 4.3 Booting the 286

After LCA config + MEMMODE, the driver resumes the 286 out of reset and polls until
the trapped port number is **0xBB** (the "BIOS alive" beacon, the first instruction
of the POST), with a timeout that retries by pulsing PORT_A bit0 (286 reset).

### 4.4 The driver-side address translation and the `.dsg` records

The driver translates x86 → Amiga addresses with a **software table**
(`shared+0x1D14`), which is exactly the head of the `atplus.dsg` records (Amiga
64 KB bank numbers indexed by x86 bank). The records of group **'$' (16–21) are the
atplus2 configurations**; they carry the same map resolved in hardware in §2.6, and
the MEMMODE value selects only where the extended block lands.

### 4.5 The emulated I/O port map

This is where you concretely see how the 68000 acts as the PC chipset. The driver
keeps two handler tables (one for `OUT`s, one for `IN`s), indexed by `port >> 4`.
Each port range is emulated in software, in many cases backed by a real Amiga
device (floppy, serial, parallel):

| Port range | Emulated PC device | Amiga backend |
|---|---|---|
| 0x20–0x2F | PIC #1 (8259) | software (state in shared RAM) |
| 0x40–0x4F | PIT 8253 (timer) | software |
| 0x60–0x6F | 8042 (keyboard controller, incl. A20 via cmd 0xD1) | software + the LCA A20 gate |
| 0x70–0x7F | RTC/CMOS | software |
| 0xA0–0xAF | PIC #2 / NMI mask | software |
| 0xB0–0xBF | **Vortex private ports** (see §3.3) | high-level services |
| 0xF0–0xFF | 287 control (F0 = busy-ack → PORT_A bit3, F1 = reset → bit1) | hardware (GAL U44) |
| 0x200–0x20F | game port | software |
| 0x278 / 0x378 | parallel port | `parallel.device` |
| 0x2F8 | COM2 (serial) | Amiga serial hardware |
| 0x3B0–0x3BF | MDA/Hercules CRTC | video render |
| 0x3D0–0x3DF | CGA CRTC | video render |
| 0x3F0–0x3FF | floppy controller | `trackdisk` |
| everything else | default handler | shadow byte + resume |

---

## 5. v2.20 / v3.00 differences

The LCA bitstreams of the v2.20 and v3.00 software versions are **logically
identical** (they differ by 10 bytes of ASCII header). The **hardware/protocol
contract is unchanged**: all LCA-facing routines are byte-identical. v3.00 (07/1992)
adds only software features (CLI options /M /E /B /N, config choice from cfg,
PAL/NTSC via GfxBase, video replay at boot, a 287 guard on `OUT 0xF0`). The runtime
`.dsg` records of v3.00 are tagged '&' (0x26) instead of '$' (0x24), but the
content is identical and the tag is never read by the driver (selection is
positional). **A v2.20 model therefore covers v3.00 as well.**

---

## 6. The WinUAE emulation

This section describes how the board was modelled in the WinUAE core (main file
`atonce.cpp`, plus a few minimal patches to the surrounding files). It is the part
of most practical interest for anyone who wants to understand *how* this board is
emulated, and it collects all the important implementation choices, including the
ones still open.

### 6.1 Starting point: the PCem core

WinUAE already includes an x86 core derived from **PCem** (v16), used for the
Amiga's official PC bridgeboards (A1060/A2088/A2286/A2386). We reuse it for the
AT-Once's 286, building the LCA model around it: magic ports, memory translation,
trap semantics and execution scheduling. Since PCem is single-instance, the AT-Once
is **mutually exclusive** with those bridgeboards.

The core is brought up to the **bare minimum** (`atonce_init`): a 286 at 16 MHz
(`cpu_set_turbo(1)`, otherwise PCem would cap the 286 at 8 MHz and DOS would run at
half speed), no FPU (`FPU_NONE`, the 287 path is not wired yet), and **no PC support
chips**: PIT, DMA, keyboard controller, RTC are **not initialized**, because they do
not exist in the hardware — the 68k driver emulates them through the traps. The only
PCem piece kept active is the `pic`, because the CPU core calls `picinterrupt()`
(patched to deliver `atonce_intr_vector`).

### 6.2 The execution model and throughput

This is the central design choice, and the most delicate. The Amiga's 68000 runs at
its fixed clock (~7 MHz). The 286 **only advances inside** `atonce_run(xb, budget)`,
called mainly from the PORT_B resume (`atonce_portb_write`). Inside it, `exec386` is
an ordinary **synchronous** C call: the 286's cycles are effectively "free" on the
Amiga's time, and the 286 advances until:

- it executes an `IN`/`OUT` that triggers a trap (`atonce_trap` forces the slice to
  end by setting `cycles` to a hugely negative value), or
- it exhausts the cycle budget (`ATONCE_RUN_CYCLES = 65536`).

Because the driver, right after each resume, **stops** the 286 with a PORT_B read,
every iteration of its loop is a *resume → inspect → stop* cycle. As a consequence,
the per-resume budget is **the throughput unit**: the number of resumes per second
(i.e. how fast the 68k spins its loop) times the budget determines the 286's
effective frequency. With too small a budget (e.g. 2000) the 286 ran at 0.2–0.85 MHz
and DOS was very slow; with 65536 it reaches ~16 MHz effective without blocking the
Amiga thread for too long (a trap ends the burst early anyway).

One PCem technicality has to be handled by hand: `exec386` paces itself against the
timer subsystem (`cycle_period = timer_target - tsc + 1`). Since **we register no
PCem timer** (the PIT lives behind the trap), without intervention `timer_target`
would lag behind and `exec386` would never execute an instruction. `atonce_run`
therefore keeps `timer_target` a budget ahead of `tsc` on each burst.

A second execution path is `atonce_hsync` (called on every hsync): it keeps a 286
going that was in a long computation and did not reach a trap within the resume
slice. It stays idle while the 286 is `frozen`, so it never runs in parallel with
the resume slice.

> **Performance note (open):** all else being equal, the **v3.00 driver is slower
> than v2.20** (with v2.20 typing is fluid and himem loads in a flash). The cause was
> localized to a heavier, runtime data-dependent device-tick iteration in v3.00 (not
> visible in the static disassembly). To be investigated with a v3.00 trace.

### 6.3 The magic ports

They are modelled in `atonce_porta_write` / `atonce_portb_write` and the respective
reads, reached from `atonce_cia_access` (called by `cia.cpp` for accesses in CIA
space). The decode replicates GAL U22 — only A6/A7/A15, default base 0xBFF000:

- `PORT_A` ⇔ `(addr & 0x80C0) == 0x8040`
- `PORT_B` ⇔ `(addr & 0x80C0) == 0x8080`

The bits reproduce exactly the table in §2.4: a PORT_A write drives the A20 gate
(D2, via `atonce_rebuild_map` + `atonce_apply_a20`), SETUP mode (D1), 287 ack (D3),
and a 0→1 edge of D0 at runtime = an **explicit 286 reset** (`resetx86()`, used by
the driver's boot-retry). A PORT_A read composes the status (bit1 = trap pending,
bit0 = 287 error before the stop / direction after the stop). A PORT_B write does
the resume and, if bit0=1, delivers the interrupt (vector in the high byte →
`atonce_intr_vector`, `pic_intpending=1`); in SETUP mode it shifts one MEMMODE bit
(D12) into AH→BH→CH→DH. A PORT_B read stops the 286 and returns the trapped port in
the byte-swapped format the driver expects (`(port&0xFF)<<8 | (port>>8)&3`).

### 6.4 The memory translation

Below 1 MB the 286 executes code directly (BIOS at F000, upper conventional
memory), and PCem's code fetch (`getpccache`) needs a **real pointer**, not
functions. For this reason the sub-1MB windows (`atonce_win_map` +
`atonce_map_windows`) expose the **Amiga chip RAM** directly through an exec
pointer: the U24/U25 byte-swap is 1:1 at byte level, so a direct pointer matches the
handlers. The F000 window maps chip RAM 0x30000 (so `FFFF0` → chip `0x3FFF0`, the
286 reset vector).

For accesses that instead go through the functions (`atonce_mem_readb/writeb` etc.),
and for memory ≥1 MB, the translation comes from the `bank_amiga[]` table computed by
`atonce_rebuild_map()` **directly from the CLB equations** of `atplus2_core.v` (the
RTL proven equivalent to the golden netlist). It is therefore the hardware-true map,
rebuilt whenever MEMMODE or the A20 gate changes. A `-1` value in the table means
"onboard DRAM" (x86 0–512K), served from the local buffer instead of chip RAM.

> **Fidelity note (banks A/C):** `atonce_win_map` uses the LCA hardware values for
> banks A0000/C0000 (chip 0x20000), matching `atonce_rebuild_map` and §2.6 of this
> document. The `.dsg` software translation table the 68k driver shifts in says
> 0x60000 (A) / 0x00000 (C), but those are software-only aliases; both banks are
> unused (no EGA/VGA), so this is purely a fidelity choice — the emulator models the
> hardware, so it uses the LCA value.

### 6.5 The A20 gate

On the AT-Once there is no keyboard controller: the **A20 gate is the LCA's A20
line** (PORT_A D2). In WinUAE this drives PCem's `rammask` (`atonce_apply_a20`:
`mem_a20_alt` + `mem_a20_recalc`), so the 286 wraps 0x100000→0 when A20 is gated off
— exactly what himem.sys's A20-control test checks (otherwise it reports "Unable to
control A20 line!"). We keep `mem_a20_key` cleared because the gate is entirely ours
(PCem would default it to always-on).

### 6.6 The 0x900000 DRAM window and OS coexistence

The onboard DRAM is exposed to the 68k in the fixed window **0x900000–0x9FFFFF**
(`atonce_dram_bank`, mapped with `map_banks`). Important point: while the board is
**not configured** (DONE not asserted), the window must stay **inert** and let any
underlying Z2 fast RAM show through (`atonce_dram_under` looks up `fastmem_bank[0]`
at runtime), so the OS can use that RAM during boot. Without this care, the OS would
allocate at 0x900000 during early boot and the accesses would hit the 286's DRAM →
corruption → crash. This mirrors the real hardware (board inert from power-on until
the driver activates it).

### 6.7 Isolated extended memory (`ATONCE_ISOLATE_EXT`)

Current functional configuration: the 286's >1 MB extended memory is backed by a
**private flat buffer** (`emem`, ~15 MB) mapped with a real exec pointer (so
`getpccache` works when himem.sys relocates/executes in the HMA, above 1 MB), **not**
by shared Amiga RAM. This is the memory model with which DOS/himem work today.

> **Caveat:** the "real" mapping of >1 MB extended memory onto Amiga RAM (MEMMODE
> 4/2/1 → 0x080000/0x200000/0xC00000) is **decode-validated** but **not yet exercised
> at runtime** in WinUAE (the isolated `emem` buffer is used). It should be flagged as
> "modelled per decode, runtime-validation pending". The alternative path (`#else`)
> with dynamic translation onto Amiga RAM already exists in the code, to be enabled
> when this point is to be closed.

### 6.8 The trap semantics and the I/O shadow

Every 286 I/O access outside 0xF8–0xFF enters `atonce_x86_io` → `atonce_trap`, which
freezes the 286 and ends the slice **after** the I/O instruction. The data travels
through the `pusha` frame in shared RAM anyway, so this is equivalent to the
hardware's freeze-inside-OUT for BIOS service traps. Status bit1 stays set until
PORT_B is read; the next PORT_B write un-freezes the 286.

A detail that is faithful to the hardware: on an `OUT`, the written data is
**latched into the F000 I/O shadow** (`atonce_io_shadow`, chip 0x30000+port), so the
driver reads it back (CRTC index from F000:03D4, data from 03D5, etc.). On an `IN`,
we return the value the driver pre-staged in the same shadow and then trap, so the
driver can update the device state (e.g. clear the 8042 data-ready). The data
returned to a trapped `IN` is "junk" (0xFF) exactly as on real hardware: the BIOS
stub overwrites AX in the `pusha` frame after the service.

### 6.9 The video refresh (idle render)

A pragmatic but necessary point. The 68k driver re-draws the Amiga screen only on
real 286 video activity (a CRTC scroll, or the bottom-row `OUT 0xBA`). At the DOS
prompt, or when text lands on a row that does not trigger a render, the screen would
stay "behind" (boot prompt, pre-scroll "A>", typed characters invisible). So, when
needed, we **fabricate a synthetic CRTC trap** (`atonce_fire_refresh` fakes a write
to CRTC register 12), which makes the driver perform a full-screen re-render.

That re-render runs on the 68k via `graphics.library` and is heavy: forcing it every
frame (50 Hz) saturated the 68k and made everything sluggish. The fix is to do it
**only when the 286's text page has actually changed** (a cheap FNV hash once per
frame, `atonce_text_hash`), plus a slow ~2 Hz heartbeat for the blinking cursor. The
injection happens only at a safe point (`atonce_refresh_safe_port`): never inside a
disk/video/service sequence, which would derail the boot.

### 6.10 The hard-disk bypass

For the hard disk (drive ≥ 0x80) WinUAE can **bypass the 68k driver** and serve INT
13h directly from its own hardfile (`atonce_disk_bypass`, on ports 0xB0/0xB9).
`atonce_hdd_resolve` searches the WinUAE hardfiles for a disk with a PC partition
(it walks the RDB, finds a partition that starts with a `55 AA` MBR, derives the CHS
geometry from a partition-table entry). Once the disk is found, the INT 13h functions
(read/write sectors, get geometry, etc.) read/write the sectors from the hardfile
and **write the result directly into the 286's `pusha` frame** (`atonce_disk_result`:
AH=status, AL=count, CF in the flags). If no PC disk is configured, it falls back to
the 68k driver.

### 6.11 What an emulator needs, in summary

1. Two registers in CIA space, decoded on A15/A7/A6 (PORT_A byte, PORT_B word) with
   the semantics of §2.4.
2. 286 memory accesses translated per the atplus2 hardware map (§2.6) with the
   byte-swap (286 even byte ↔ 68k D8–15), and sub-1MB windows with real exec pointers
   onto chip RAM so the 286 can *execute* the BIOS.
3. Trap semantics: every I/O outside 0xF8–0xFF freezes the 286 (latching the data in
   the F000 shadow on OUT); the next PORT_B write releases it.
4. An A20 gate that masks bit 20 of the 286 (wrap 0x100000→0).
5. Amiga level-3 IRQs put the 286 in HOLD (relevant for timing only).
6. The 286 starts held in reset; the first resume after configuration releases it.
7. The 0x900000 DRAM window toward the 68k, inert until the board is configured.

### 6.12 Status and open points

- **Working**: driver boot, BIOS POST (`OUT 0xBB` beacon), DOS prompt, keyboard,
  text video, disk (hardfile bypass), extended memory/himem (isolated buffer).
- **Open**: v3.00 performance (§6.2); extended memory onto real Amiga RAM instead of
  the isolated buffer (§6.7); the 287/FPU path; booting Windows 3.1 (a triple-fault
  loop tied to the 286 technique of returning to real mode via reset — `LIDT null;
  INT 3` → shutdown → CMOS warm path, currently handled as a cold reset).

---
