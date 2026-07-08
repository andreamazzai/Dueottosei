# Amiga Dueottosei

Amiga Dueottosei (Italian for "two-eight-six") is a clone of the [Vortex ATonce Plus](https://amiga.resource.cx/exp/atonceplus), a PC AT Emulator board for the Amiga 500.

![alt text](img/duottosei_compled.jpg "Dueottosei")

## Main features of Dueottosei

* Fully compatible 16 MHz PC/AT emulator
* CMOS 80C286-16 CPU chip
* 512 KB onboard RAM
* Socket for optional 80C287-12 math coprocessor
* Fully AT compatible ROM/BIOS
* Full 640 KB base memory as minimum standard configuration
* Ability to address additional FAST RAM as either Extended or Expanded Memory
* Runs unrestricted in '286 Protected Mode
* Supports Microsoft Windows 3.0 and Windows 3.1
* Emulates EGA and VGA monochrome graphics, CGA 16-color graphics, Hercules, Olivetti and Toshiba 3100 display standards
* Runs as a concurrent process within the AmigaDOS operating system
* Reads and writes MS/DOS file system on any standard Amiga floppy drive
* Full support for MS/DOS partitions on SCSI and IDE controllers (not all 100% compatible)
* Recognizes the Amiga mouse as a Microsoft Serial mouse (on COM 1 or COM 2)
* Recognizes the Amiga's Parallel port as LPT1
* Recognizes the Amiga's Serial port as either COM2 or COM1 (depending on mouse setting)
* Emulates only the PC/AT alert beep through Amiga audio hardware
* Recognizes and uses the Amiga's Real-Time clock
* Compatible with all versions of MS/DOS from 3.2 through 5.0

## Bill of materials

| Qty | Description        | Designator                         | Digikey Part         | Note                                                      |
|:---:|--------------------|------------------------------------|----------------------|-----------------------------------------------------------|
| 10  | Resistor 10K       | R11-R17, R41, R49, R50             | ERJ-8ENF1002V        |                                                           |
|  9  | Resistor 22R       | R31-R39                            | ERJ-8GEYJ220V        |                                                           |
|  5  | Resistor 4.7K      | R42-R44, R47, R48                  | ERJ-8GEYJ472V        |                                                           |
|  1  | Resistor 2.2K      | R110                               | ERJ-8GEYJ222V        |                                                           |
|  2  | Resistor 1K        | R45, R46                           | ERJ-8GEYJ102V        |                                                           |
|  3  | Resistor 100R      | R311-R313                          | ERJ-8ENF1000V        |                                                           |
| 27  | Cap. 100nF         | C12-C16, C21-C29, C31-C37, C41-C46 | 12065C104KAT2A       |                                                           |
|  2  | Cap. 100uF         | C10, C11                           | T495X107K025ATE150   |                                                           |
|  1  | 74LS08             | U15                                | SN74LS08D            |                                                           |
|  3  | 74HCT374           | U21, U26, U29                      | MC74HCT374ADWG       |                                                           |
|  2  | GAL16V8            | U22, U44                           | ATF16V8BQL-15PU      | TL866-like programmer needed                              |
|  2  | 74F245             | U24, U25                           | SN74F245DW           |                                                           |
|  1  | 74F86              | U28                                | SN74F86D             |                                                           |
|  2  | 74F157             | U35, U36                           | SN74F157AD           |                                                           |
|  1  | 74F153             | U37                                | SN74F153D            |                                                           |
|  1  | 74F125             | U42                                | SN74F125D            |                                                           |
|  1  | 74F00              | U43                                | SN74F00D             |                                                           |
|  1  | 74F260             | U45                                | SN74F260D            |                                                           |
|  1  | 32Mhz Oscil.       | Q1                                 | MXO45-3C-32M000000   |                                                           |
|  1  | PLCC 84 Socket     | U41                                | 8484-11B1-RK-TP      |                                                           |
|  1  | PLCC 68 Socket     | U13                                | 8468-11B1-RK-TP      |                                                           |
|  6  | DIP 20 Socket      | U22, U44, U31-U34                  | 4820-3004-CP         | Optional, but useful for testing                          |
|  1  | DIP 40 Socket      | U14                                | 4840-6000-CP         |                                                           |
|  1  | DIP 64 Socket      | J1                                 | 110-99-964-41-001000 |                                                           |
|  2  | SIP 32 Socket      | J2                                 | D01-9973242          | Qty 4 if you use socket in U44                            |
|  2  | SIP 32 male Socket | J1                                 | D01-9923246          |                                                           |
|  1  | N80C286-16         | U13                                |                      | eBay or [utsurce](https://www.utsource.net)               |
|  2  | 80C287 12Mhz       | U14                                |                      | Optional from eBay or [utsurce](https://www.utsource.net) |
|  1  | XC2018-70          | U41                                |                      | eBay or [utsurce](https://www.utsource.net)               |
|  4  | TMS44C256-80N      | U31-U34                            |                      | eBay or [utsurce](https://www.utsource.net)               |

## Building notes

* Check the [steps needed](img/BuildStep) to build the board.
* I bought five XC2018 before finding a good one running for more than 10 minutes without hanging or crashing. Buy a recent production one like the one pictured above.
* If you want to install the math coprocessor, you'll need a 287 at 12Mhz like the XL model from Intel or compatible.
* You will need a programmer like TL866+ to cook the two GAL's with the [jed files](GAL).
* The A500 keyboard [touches](img/keyboard.jpg) the PLCC socket and remains a few millimeters higher, but the 500 case closes without problems.
* You could add a plastic support to hold the card on the opposite side of the 68000 socket.
* Software is available on [Amiga Hardware Database](https://amiga.resource.cx/exp/atonceplus).
  * German User Manual on [Internet Archive](https://archive.org/details/ATonce-Amiga_1991_Vortex_Computersysteme_DE).
  * English User Manual on [Internet Archive](https://archive.org/details/vortex-atonce-plus-manual-en).
* Dueottosei is affected by electromagnetic interference (EMI) coming from Agnus on the Amiga 500 board. A [shield](img/BuildStep/step4.jpg) to be placed between the two boards is essential; you can use this RF EMI shielding tape (part no. 3M11331-ND) or similar. You can also use simple aluminum foil wrapped in electrical tape.

Inside the repository, you can find the Gerber files for the production of the [Dueottosei PCB](kicad/dueottosei/gerber_dueottosei.zip), and also the faithful replica of the [Vortex ATonce Plus PCB](kicad/atonceplus/Gerber.zip) (useful for original board restoration), as well as the complete KiCad projects.

![alt text](img/dueottosei_pbc.PNG "Dueottosei PCB")

## Compatibility

If you decide to build this board, you must know that it has limited compatibility: it only works with Kickstart 1.3 or 2.x, and only on the most recent 500 PCB versions. Read these reviews first to get an idea of how it works:

* [Amiga News - May 1992](http://obligement.free.fr/articles/atonce_plus.php)
* [Todd Lowe - Blob Shop Programmers](misc/review.txt)

Watch a short [demonstration video](https://www.youtube.com/watch?v=Pk9XQjsHf10) on YouTube.
Download the [CF disk image](misc/dueottosei.rar) used in the demo.

Based on my testing, the following is the best setup, working every time without crashing after a few hours of running Windows 3.1:

* Amiga 500+ board rev 8a Cpu 68010 with 2 MB chip Ram
* Kickstart and Workbench 2.05
* [AlfaPower Plus](https://amiga.resource.cx/exp/alfapowerplus) with 8 MB Fast (4 MB dedicated as MS-DOS extended memory) and 500 MB IDE HDD
* External floppy disk DF1 configured as A:
* 80 MB partition non formatted with Amiga filesystem as C:
* 150 Watt PSU

The [socket adapter for Gary](img/GaryAdapter/socket.jpg) could solve some problems; actually, in my tests it didn't help.

## About Xilinx XC2018

The Xilinx XC2018 is a first-generation FPGA produced by Xilinx without permanent internal memory; in practice, it must be programmed — in this case, by the Amiga — every time it is started. You can read the details in [this article](https://www.righto.com/2020/09/reverse-engineering-first-fpga-chip.html) written by Ken Shirriff a few years ago.

Inside the floppy disks Vortex v2.20/v3.00 and GVP PC286, I found some configuration bitstreams:

* [one](xc2018/v3.00/atonce.bin.rbt) appears twice in atonce.bin/pc286.bin (offsets 0x9300-0x8A44 and 0x9C00-0x9344), apparently unused in any setup and identical across all bin file versions.
* there are two more in atplus.dsg/pc286.dsg: the [first one](xc2018/v3.00/atplus1.rbt) (offset 0x1400-0x0B44) could be for the AT-Once Classic (non-Plus) board, while the [second one](xc2018/v3.00/atplus2.rbt) (offset 0x2400-0x1B44) is the one for the AT-Once Plus. This last one changes across all the different versions of the dsg file.

Thanks to [this tool](https://github.com/na103/xc2018), originally created by Ken Shirriff and modified by me for the 100-tile XC2018, I rebuilt all three LCAs found inside the v3.00 floppy disk. The new bitstream generated from the LCA file is 99% identical to the original stream extracted from the dsg file, except for a small difference: in the original bitstream, CLB blocks configured as Base F, with output (X or Y) not used in any equation and not connected to any net, for some strange reason (maybe an early makebit bug) have the X or Y bit output on G active.

Anyway, I [patched](xc2018/v3.00/patchdsg.py) the [dsg](xc2018/v3.00/atplus_patch.dsg) with the newly generated bitstream, and it works just as well as the original, so I'm fairly confident it's a correct LCA. The [notie](xc2018/notie) folder contains the LCA file with only the essential logic, cleaned of all the nets and CLBs generated by makebits with the tie option. From the notie LCA, a Verilog description was created.

If you found this my work useful, please consider buying me a cup of coffee if you want:

<a href='https://ko-fi.com/na103' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://storage.ko-fi.com/cdn/cup-border.png' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>

## License

This work is licensed under a Creative Commons Attribution 4.0 International License. See [https://creativecommons.org/licenses/by/4.0/](https://creativecommons.org/licenses/by/4.0/).
