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

The next pass should map the FDD I/O handlers and separate PC Card, built-in
store, DreamLink, and FDD path dispatch.
