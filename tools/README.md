# ROM Tool Reference

`rom2.py` contains small inspection helpers for the DreamWriter T200 ROM image.
By default it reads `drwrt200.bin` from the repository root and expects the
1 MiB image with SHA-256:

```text
b48c123edaa7cc5271947c20dc0641c62bc444052aa3a90c66570ede9b6b0fd7
```

Use `--rom PATH` before the subcommand to inspect a different file:

```sh
tools/rom2.py --rom drwrt200.bin verify
```

The script assumes the T200 ROM is loaded at physical `0x00000`, matching the
current MAME model. File offsets and physical addresses are therefore the same.

## Number And Address Syntax

Integer arguments accept `0x` prefixes and `$` prefixes for hexadecimal values.
Bare values containing `a..f` are parsed as hexadecimal. Bare values made only
of digits use Python's normal integer rules, so use `0xc0000` or `$c0000` when
you mean hexadecimal `c0000`.

The `addr` command accepts:

| Form | Meaning |
| --- | --- |
| `file:0xc8ab2` | ROM file offset. |
| `phys:0xc8ab2` | 20-bit physical address. |
| `c000:8ab2` | Real-mode segment:offset address. |
| `fb64:0000` | Reset trampoline segment:offset address. |
| `0xc8ab2` | Bare value, equivalent to file/physical because load offset is zero. |

## Commands

### `verify`

Checks the ROM size and SHA-256 digest.

```sh
tools/rom2.py verify
```

### `addr`

Converts between ROM file offsets, physical addresses, and canonical
segment:offset form.

```sh
tools/rom2.py addr file:0xc8ab2
tools/rom2.py addr phys:0xffff0
tools/rom2.py addr fb64:0000
```

### `bank`

Describes how a MAME bank-select port/value pair maps a 128 KiB CPU window.

```sh
tools/rom2.py bank 0x17 0x00 --cpu 0xffff0
tools/rom2.py bank 0x16 0x01 --cpu 0xc0000
```

### `strings`

Lists printable ASCII runs.

```sh
tools/rom2.py strings --start 0xc0000 --end 0x100000 -n 12
```

### `glyphs`

Renders fixed-height 1bpp glyphs as `#` and `.` text. The default font is the
MAME-declared T200 8x8 font at file `0xDBBEB`, with first character code
`0x20`.

```sh
tools/rom2.py glyphs --text 'A0!' --columns 6
tools/rom2.py glyphs --base 0xdbbeb --first-code 0x20 --count 16 --columns 6
```

### `decode_lcd_text.py`

Decodes a `drwrt200` LCD snapshot PNG into text using the ROM's 6x8 text cells.
The decoder is strict: every 6x8 cell must match a ROM glyph unless cursor or
inverse matching is enabled.

```sh
python3 tools/decode_lcd_text.py ../mame/snap/drwrt200/0000.png
```

### `bitmap`

Renders fixed-size 1bpp bitmap blocks.

```sh
tools/rom2.py bitmap --base 0xd7afa --row-bytes 5 --height 34 --columns 36
```

### `bitmap-records`

Scans for plausible `FF 42` source-backed bitmap records.

```sh
tools/rom2.py bitmap-records --start 0xd7800 --end 0xf7000 --require-position --commands
```

### `position-ops`

Finds `FF 40` position records followed by another `FF xx` resource opcode.

```sh
tools/rom2.py position-ops --start 0xf6600 --end 0xfb640 --limit 40
```

### `xrefs`

Runs `ndisasm` over a file-offset range and extracts direct branch/call targets.
This is an inventory helper, not a recursive disassembler.

```sh
tools/rom2.py xrefs --start 0xc0000 --end 0xc2000
tools/rom2.py xrefs --start 0xc0000 --end 0xc9000 --format markdown --limit 20
```

### `regions`

Lists `docs/rom-regions.tsv`.

```sh
tools/rom2.py regions
tools/rom2.py regions --format markdown
```

### `io-scan`

Disassembles mapped code regions and scans for x86 `in`/`out` instructions.
The current region map is broad, so expect false positives until code ranges
are tightened.

```sh
tools/rom2.py io-scan --summary
tools/rom2.py io-scan --limit 80
```
