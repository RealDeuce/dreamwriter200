# Hardware Notes

These notes start from the current MAME `nakajies.cpp` driver and first ROM
inspection. They are not yet board-confirmed.

## MAME Configuration

`drwrt200` uses the `nakajies250` machine configuration:

| Device | MAME model |
| --- | --- |
| CPU | NEC V20 at `X301 / 2`. |
| RAM | 256 KiB internal RAM in the current driver. |
| Display | 480x128 LCD, rendered as 80 columns x 16 rows of 6x8 cells. |
| Font | `GFXDECODE_ENTRY("bios", 0xdbbeb, nakajies_charlayout, 0, 1)`. |
| Storage | PC Card slot plus N82077AA floppy controller; comment says actual chip is Intel N82877SL. |
| Serial | NEC uPD71051-compatible USART via MAME `I8251`. |
| RTC/NVRAM | Ricoh `RP5C01` plus battery-backed internal RAM. |
| Printer | Centronics port. |
| Power input | NMI-style power input, matching T400/T450 rather than the older IRQ input. |

## Keyboard

MAME uses the same 10-row active-high keyboard matrix family as the other
Nakajima/DreamWriter machines. The T400 notes found raw row cache at
`6D06..6D0F`; T200 needs confirmation, but the startup code and diagnostic
strings strongly suggest the same firmware family.

Synthetic debugger function-key vectors in the MAME input map are:

```text
F1 -> irq FF
F2 -> irq FE
F3 -> irq FD
F4 -> irq FC
F5 -> irq FB
F6 -> irq FA
F7 -> irq F9
F8 -> irq F8
```

## LCD / Framebuffer

MAME models the LCD scanout base through port `0x00`:

```text
lcd_base = value << 9
```

T200 early startup writes:

```asm
C000:004A  mov al,40
C000:004C  out 00,al
```

That selects a scanout base of `0x8000` in internal RAM under the current MAME
model. MAME renders 480 visible pixels per row with a 64-byte row stride.

## Ports Seen So Far

The shared MAME I/O map includes:

| Port | Evidence |
| ---: | --- |
| `0x00` | LCD scanout base select. T200 boot writes `0x40`. |
| `0x10..0x17` | Eight 128 KiB bank select registers. T200 reset/startup writes `0x10`, `0x16`, and `0x17` early. |
| `0x20` | Startup writes `0x20`; likely control/status latch as in related models. |
| `0x30` | USART/control latch in MAME. |
| `0x40` | Centronics data latch. Startup writes `0xFF`. |
| `0x50..0x53` | Buzzer/timer counter area in MAME. |
| `0x60` | IRQ/source control latch in MAME. |
| `0x61` | Keyboard row-scan sequencer control candidate. |
| `0x70` | Power/reset transition control candidate. |
| `0x90` | IRQ active/source clear register in MAME. Startup writes `0xFF`. |
| `0xA0` | Shared status input for card, battery, and printer state. |
| `0xB0` | Keyboard row input. |
| `0xC0..0xC1` | USART data/status/control. |
| `0xD0..0xDF` | RTC register block in MAME. T200 startup writes `0xF8` to `0xDD` and `0xF0` to `0xDE`. |
| `0xE0..0xE1`, `0xEC..0xEF` | Card/control-looking fixed I/O area in related models. |

Run:

```sh
tools/rom2.py io-scan --summary
```

after tightening `docs/rom-regions.tsv` code ranges to reduce false positives.

## Floppy Controller

Unlike T400, the T200 MAME configuration adds an FDC:

```cpp
N82077AA(config, m_fdc, 24_MHz_XTAL);  // Actually Intel N82877SL
FLOPPY_CONNECTOR(config, m_floppy, "35hd", FLOPPY_35_HD, true, ...);
```

The visible T200 strings include FDD-specific file workflow messages:

```text
No disk is in the FDD
Disk is not formatted or not compatible
Card or FD is write-protected
(Built-in, Card, DreamLink, or FDD)
```

MAME maps the FDC into the T200 I/O map at `0xE0..0xEF`. The ROM's main
direct FDC cluster is `0xC5900..0xC6A2A`, with the densest port traffic in the
`0xC6315..0xC6A20` service routines:

| Port | ROM use observed |
| ---: | --- |
| `0xE2` | Controller output/control register. The ROM shadows this byte at RAM `0x6FF6`; `0xC6961` sets bit `0x10` for motor-on and `0xC697E` clears it. |
| `0xE4` | Main status register polling before FIFO reads/writes. |
| `0xE5` | Data FIFO for command bytes, result bytes, and 512-byte sector payloads. |
| `0xE7` | Control/config register written during controller reset/config. |
| `0xE0..0xE1`, `0xEC..0xEF` | Touched by the diagnostic/probe helpers at `0xC8C00` and `0xC8C0B`; not yet tied to normal file I/O. |

Important first-pass routine labels:

| Offset | Role |
| ---: | --- |
| `0xC66FD` | Reset/configure the FDC, including DOR/control writes and controller command setup. |
| `0xC6961` / `0xC697E` | Motor on/off using the `0xE2` shadow at `0x6FF6`. |
| `0xC69A1` | Send one command/parameter byte through the FIFO after status polling. |
| `0xC69B3` / `0xC69E5` | Receive one result/data byte from the FIFO after status polling. |
| `0xC677C` | Read one 512-byte sector into RAM buffer `0x6300`. |
| `0xC67E0` | Write one 512-byte sector from RAM buffer `0x6300`. |
| `0xC6840` | Build the FDC read/write parameter sequence: drive/head, cylinder, sector, size, EOT/GPL/DTL-style bytes. |
| `0xC68EE` | Read the command result phase, storing FDC status bytes around RAM `0x6FF0..0x6FF2`. |
| `0xC5AFA` | Scan directory entries in 0x20-byte records. |
| `0xC5CFF` / `0xC5D00` | File read path over the sector/cache helpers. |
| `0xC5DEE` | File write/export path, including optional text conversion before sector writes. |
| `0xC5F11` | Close/finalize write path, including EOF marker handling and directory/FAT-style updates. |
| `0xC6048` | Delete path; marks a directory entry with `0xE5`. |
| `0xC60AF` | Rename/update path; copies an 11-byte name into a directory entry. |

The floppy file code looks FAT-like: it uses 512-byte sectors, 0x20-byte
directory entries, deleted-entry marker `0xE5`, and an 11-byte 8.3-style name
buffer at RAM `0x6F33`. Current working RAM landmarks include cylinder
`0x6F00`, sector `0x6F02`, selected head/drive `0x6FEF`, FDC result/status
bytes near `0x6FF0`, and error/status storage at `0x15B5`.

The firmware reaches the FDD file routines through its DOS-like `INT 21h`
dispatcher at `0xC73D2`. The storage target enum observed there is `0x08`
built-in, `0x09` PC Card, `0x0A` DreamLink, and `0x0B` FDD; FDD operations use
active-target RAM `0x6F51` and handle-target cache entries near `0x6FE3`.

See [`floppy.md`](floppy.md) for the focused FDD routine map and next traces.
