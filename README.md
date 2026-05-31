# DreamWriter T200 ROM Notes

Working target: `drwrt200.bin`

```text
Size:   1048576 bytes
SHA256: b48c123edaa7cc5271947c20dc0641c62bc444052aa3a90c66570ede9b6b0fd7
```

This repo is for mapping and tooling around the NTS DreamWriter T200 ROM image.
It follows the same working pattern as `dreamwriter-rom-map`, but starts as a
first-pass T200 map, not a complete offset-by-offset port of the T400 notes.
Offsets called out as confirmed were checked against `drwrt200.bin`; rough and
probable ranges still need recursive code/resource tracing. Do not reuse T400
offsets unless they are explicitly labeled as comparative notes.

## MAME

Machine:

```sh
mame drwrt200
```

The local MAME driver currently marks `drwrt200` as `MACHINE_NOT_WORKING`.
It uses the `nakajies250` configuration: V20 CPU, 256 KiB RAM, PC Card support,
Intel N82877SL-compatible floppy controller, NMI power input, and a 480x128 LCD.
A snapshot of the relevant driver is kept at [`mame/nakajies.cpp`](mame/nakajies.cpp).

## ROM Mapping

The T200 ROM is a 1 MiB image loaded at MAME BIOS region offset `0x000000`, so
ROM file offsets and 20-bit physical addresses are initially identical:

```text
physical address 0x00000..0xFFFFF -> file offset 0x00000..0xFFFFF
```

Examples:

```text
C000:0000 physical C0000 -> file offset 0xC0000
FB64:0000 physical FB640 -> file offset 0xFB640
FFFF:0000 physical FFFF0 -> file offset 0xFFFF0
```

The reset vector at file `0xFFFF0` jumps to `FB64:0000`, which sets bank ports
`0x16` and `0x17`, then jumps to `C000:0000`.

Address, string, glyph, bitmap, direct branch/call, and I/O-scan helpers are in
[`tools/rom2.py`](tools/rom2.py); see [`tools/README.md`](tools/README.md).

## Documentation Index

| File | Description |
| --- | --- |
| [`docs/README.md`](docs/README.md) | Index for the topic-specific ROM notes. |
| [`docs/map.md`](docs/map.md) | Address model, coarse ROM layout, and reset chain. |
| [`docs/rom-regions.tsv`](docs/rom-regions.tsv) | Machine-readable first-pass region map. |
| [`docs/strings.md`](docs/strings.md) | Initial string/resource landmarks. |
| [`docs/hardware.md`](docs/hardware.md) | Hardware notes from MAME and first ROM inspection. |
| [`docs/open-questions.md`](docs/open-questions.md) | Working questions for the next mapping pass. |
| [`tools/README.md`](tools/README.md) | Command reference for `tools/rom2.py`. |

## Boot Path

The CPU reset vector is at physical/file `0xFFFF0`:

```asm
FFFF:0000  cli
FFFF:0001  jmp far FB64:0000
```

The reset trampoline at file `0xFB640`:

```asm
FB64:0000  cli
FB64:0001  mov al,01
FB64:0003  out 16,al
FB64:0005  mov al,00
FB64:0007  out 17,al
FB64:0009  jmp far C000:0000
```

Normal startup begins at `C000:0000`:

```asm
C000:0000  jmp C000:0029
C000:0029  cli
C000:002A  mov al,17
C000:002C  out 10,al
C000:002E  mov al,01
C000:0030  out 16,al
C000:0032  mov al,00
C000:0034  out 17,al
```

## Diagnostic Landmark

The diagnostic banner is currently visible at file/physical `0xC8AB2`:

```text
Diagnostic 30CAB192       98May11       K: Keyboard check
```

Nearby help strings include memory dump/set, single-step, generic I/O dump/out,
card attribute, COM, and clear/reset spell commands.
