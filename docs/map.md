# ROM Map

Working target: `drwrt200.bin`

```text
Size:   1048576 bytes
SHA256: b48c123edaa7cc5271947c20dc0641c62bc444052aa3a90c66570ede9b6b0fd7
```

## Address Model

MAME loads the T200 BIOS at region offset `0x000000..0x0FFFFF`. For the
standalone image, file offsets and 20-bit physical addresses are currently the
same:

```text
physical = file offset
file offset = physical
```

Examples:

```text
C000:0000 -> physical 0xC0000 -> file 0xC0000
C000:8AB2 -> physical 0xC8AB2 -> file 0xC8AB2
FB64:0000 -> physical 0xFB640 -> file 0xFB640
FFFF:0000 -> physical 0xFFFF0 -> file 0xFFFF0
```

Real-mode segment aliases still matter. The helper prints canonical
segment:offset addresses:

```sh
tools/rom2.py addr c000:8ab2
tools/rom2.py addr fb64:0000
```

## Banking Model

MAME maps the 1 MiB V20 address space as eight 128 KiB banks:

| CPU range | Bank | Select port |
| --- | ---: | ---: |
| `0x00000..0x1FFFF` | 0 | `0x10` |
| `0x20000..0x3FFFF` | 1 | `0x11` |
| `0x40000..0x5FFFF` | 2 | `0x12` |
| `0x60000..0x7FFFF` | 3 | `0x13` |
| `0x80000..0x9FFFF` | 4 | `0x14` |
| `0xA0000..0xBFFFF` | 5 | `0x15` |
| `0xC0000..0xDFFFF` | 6 | `0x16` |
| `0xE0000..0xFFFFF` | 7 | `0x17` |

For ROM banking, MAME uses `((value & 0x0F) ^ 0x0F)` as the 128 KiB ROM bank
entry, with the 1 MiB image mirrored into entries `0x08..0x0F`. Values with bit
`0x10` set select internal RAM or PC Card space depending on bit `0x08` and card
state. The T200 configuration currently has 256 KiB internal RAM and enables the
bit-3 RAM behavior inherited from `nakajies220_256`.

Use:

```sh
tools/rom2.py bank 0x17 0x00 --cpu 0xffff0
tools/rom2.py bank 0x16 0x01 --cpu 0xc0000
```

## Coarse Layout

This is a first-pass map. Confirmed landmarks were checked directly against
`drwrt200.bin`, but most ranges are broad and should not be treated as audited
function/resource boundaries until recursive control-flow and resource passes
prove tighter limits.

| File range | Physical range | First-pass notes |
| --- | --- | --- |
| `0x00000..0x17708` | `0x00000..0x17708` | Probable 8086 spelling/grammar runtime code before linguistic tables and suffix records. |
| `0x17708..0x17810` | `0x17708..0x17810` | Probable high-byte character ordering/translation tables immediately before visible suffix records. |
| `0x17810..0x186FE` | `0x17810..0x186FE` | Probable suffix/morphology records with visible possessive, `-ing`, `-ally`, `-ation`, and `-ability` style fragments. |
| `0x186FE..0x1884C` | `0x186FE..0x1884C` | Compact visible word-fragment prelude before binary/index table bytes resume. |
| `0x1884C..0x194A6` | `0x1884C..0x194A6` | Probable linguistic index/pointer tables between visible fragment streams, including dense lookup bytes and 16-bit offset-style entries. |
| `0x194A6..0x1B800` | `0x194A6..0x1B800` | Dense compact word-fragment stream visible in low ROM string scans. |
| `0x1B800..0x1B8DE` | `0x1B800..0x1B8DE` | Probable small binary pointer table before phonetic/substitution data. |
| `0x1B8DE..0x1C0EE` | `0x1B8DE..0x1C0EE` | Probable phonetic and substitution text tables with compact vowel/consonant pattern fragments. |
| `0x1C0EE..0x202E6` | `0x1C0EE..0x202E6` | Probable spelling exception and rule data with visible words such as `although`, `doughnut`, and `highlight`. |
| `0x202E6..0x20790` | `0x202E6..0x20790` | Probable character order, allowed-character, and letter-set tables. |
| `0x20790..0x20C40` | `0x20790..0x20C40` | Probable numeric/index tables preceding a small fragment string table. |
| `0x20C40..0x20E36` | `0x20C40..0x20E36` | Probable small late string and fragment tables. |
| `0x20E36..0x20F78` | `0x20E36..0x20F78` | Probable compact binary linguistic rule tables before zero padding. |
| `0x20F78..0x2127C` | `0x20F78..0x2127C` | Confirmed mostly zero-filled padding between linguistic rule tables and late bitmask tables. |
| `0x2127C..0x21366` | `0x2127C..0x21366` | Probable late small tables and bitmasks before the next packed-data block. |
| `0x21366..0x22000` | `0x21366..0x22000` | Confirmed all-`0x3F` padding after low linguistic tables, ending on the next `0x2000`-aligned boundary. |
| `0x22000..0x5AF00` | `0x22000..0x5AF00` | Probable aligned stream of `0x80`-byte packed dictionary records; records mix dense payload bytes with zero gaps and compact trailing indexes. |
| `0x5AF00..0x60000` | `0x5AF00..0x60000` | Confirmed long `0x3F` padding run. |
| `0x60000..0x73760` | `0x60000..0x73760` | Probable `6000`-segment CIET/InfoSoft thesaurus engine code with far calls into shared firmware helpers. |
| `0x73760..0x73955` | `0x73760..0x73955` | Probable CIET resource/index tables immediately after the thesaurus engine code. |
| `0x73955..0x73A42` | `0x73955..0x73A42` | Confirmed CIET English-US metadata, copyright, part-of-speech, and relation strings. |
| `0x73A42..0x73F48` | `0x73A42..0x73F48` | Probable compact thesaurus word-fragment list before offset tables. |
| `0x73F48..0x74600` | `0x73F48..0x74600` | Probable monotonic 32-bit offset table for thesaurus resources. |
| `0x74600..0x74870` | `0x74600..0x74870` | Probable compact suffix/rule records with visible endings such as `'s`, `ed`, `ing`, `er`, `ies`, `ate`, and `ful`. |
| `0x74870..0x74948` | `0x74870..0x74948` | Probable high-byte character ordering/translation table before thesaurus offset entries. |
| `0x74948..0x74C04` | `0x74948..0x74C04` | Probable 16-bit offset-style table preceding the packed thesaurus entry stream. |
| `0x74C04..0x76E8E` | `0x74C04..0x76E8E` | Probable dense packed thesaurus entry stream ending at the next character/letter-set table block. |
| `0x76E8E..0x7726D` | `0x76E8E..0x7726D` | Probable letter-frequency, allowed-character, and compact character-set tables. |
| `0x7726D..0x77458` | `0x7726D..0x77458` | Probable small null-separated compact fragment strings. |
| `0x77458..0x77870` | `0x77458..0x77870` | Probable rule and bitmask-looking tables following compact fragment strings. |
| `0x77870..0x77A02` | `0x77870..0x77A02` | Probable high-bit character maps and 32-bit bitmask tables before markup tags. |
| `0x77A02..0x77AE0` | `0x77A02..0x77AE0` | Confirmed null-terminated thesaurus markup/tag strings such as `<DS>`, `<SY>`, `<HY>`, `<TRADE>`, and `<{sz}>`. |
| `0x77AE0..0x78098` | `0x77AE0..0x78098` | Probable markup/index lookup tables ending immediately before the long `0x3F` padding run. |
| `0x78098..0x80000` | `0x78098..0x80000` | Confirmed long `0x3F` padding run. |
| `0x80000..0xA82CB` | `0x80000..0xA82CB` | Probable dense packed payload with no stable ASCII strings; bounded by the `0x80000` bank start and repeated trailer at `0xA82CB`. |
| `0xA82CB..0xA8400` | `0xA82CB..0xA8400` | Probable repeated five-byte trailer pattern `71 5C 57 15 C5`, ending with a final zero byte before padding. |
| `0xA8400..0xAD000` | `0xA8400..0xAD000` | Confirmed long `0x3F` padding run. |
| `0xAD000..0xAF346` | `0xAD000..0xAF346` | Probable banked Typin' Time runtime code; near offsets resolve as `A000:Dxxx`. |
| `0xAF346..0xB1AA0` | `0xAF346..0xB1AA0` | Confirmed Typin' Time practice/test text records beginning with `A1` and ending after `PGPUNCT`. |
| `0xB1AA0..0xB2A44` | `0xB1AA0..0xB2A44` | Typin' Time pointer/index tables into the `AF34` text pool plus option strings. |
| `0xB2A44..0xB3036` | `0xB2A44..0xB3036` | Typin' Time display-script UI strings and option/menu text, including version/copyright. |
| `0xB3036..0xB3976` | `0xB3036..0xB3976` | Built-in database application strings and error text. |
| `0xB3976..0xB4000` | `0xB3976..0xB4000` | Built-in spreadsheet application strings and error text. |
| `0xB4000..0xBE0CF` | `0xB4000..0xBE0CF` | Probable built-in database/spreadsheet/application runtime code area. |
| `0xBE0CF..0xC0000` | `0xBE0CF..0xC0000` | Confirmed all-`0xFF` padding before the normal `C000` startup window. |
| `0xC0000..0xC13D0` | `0xC0000..0xC13D0` | Probable startup, IRQ stubs, low-level hardware setup, and early dispatch routines before the internal-error display records. |
| `0xC13D0..0xC1451` | `0xC13D0..0xC1451` | Confirmed display-script records for the internal software error and `INT` diagnostic prompts. |
| `0xC1451..0xC287C` | `0xC1451..0xC287C` | Probable debug/monitor trap helpers plus file/device service code; short trap strings are inline in the executable flow. |
| `0xC287C..0xC28B8` | `0xC287C..0xC28B8` | Confirmed display-script resource for `Error accessing Device. Press any key to cancel.` |
| `0xC28B8..0xC2FD8` | `0xC28B8..0xC2FD8` | Probable file/device helper routines and DOS-like `INT 21h` service wrappers before character remap tables. |
| `0xC2FD8..0xC3078` | `0xC2FD8..0xC3078` | Probable character remap/translation tables referenced by lookup code immediately before and after the table. |
| `0xC3078..0xC3554` | `0xC3078..0xC3554` | Probable keyboard/display control code leading into bitmap and keyboard-layout tables. |
| `0xC3554..0xC3834` | `0xC3554..0xC3834` | Probable fixed-width symbol bitmaps used by keyboard/display control routines. |
| `0xC3834..0xC3AA2` | `0xC3834..0xC3AA2` | Probable keyboard layout, shifted character, and keycode mapping tables. |
| `0xC3AA2..0xC3E20` | `0xC3AA2..0xC3E20` | Probable keyboard input decode and modifier-state code using the preceding keymap tables. |
| `0xC3E20..0xC3E70` | `0xC3E20..0xC3E70` | Probable small keycode translation table used by nearby lookup routines. |
| `0xC3E70..0xC3F66` | `0xC3E70..0xC3F66` | Probable low-level input, storage, macro/card, and terminal helper code before the storage target services. |
| `0xC3F66..0xC3F74` | `0xC3F66..0xC3F74` | Confirmed far-call wrapper that saves registers, calls the FDD probe entry at `0xC5900`, and returns far. |
| `0xC3F74..0xC52BD` | `0xC3F74..0xC52BD` | Confirmed DOS-like `INT 21h` storage target services. The target enum uses `0x08` built-in, `0x09` PC Card, `0x0A` DreamLink, and `0x0B` FDD. |
| `0xC52BD..0xC5900` | `0xC52BD..0xC5900` | Probable path parser, filename buffer builder, handle-target cache, and built-in/card helpers before the floppy service cluster. |
| `0xC5900..0xC6A2A` | `0xC5900..0xC6A2A` | Confirmed FDD filesystem/controller code with inline `VMACRO2 INC` and `NTS 2000` markers. This cluster uses N82077AA-compatible ports `0xE2`, `0xE4`, `0xE5`, and `0xE7`, handles 512-byte sector transfers through a buffer at RAM `0x6300`, and scans 0x20-byte directory entries. |
| `0xC6A2A..0xC733A` | `0xC6A2A..0xC733A` | Probable low-level helper code after the floppy service cluster and before the `INT 21h` dispatch tables. |
| `0xC733A..0xC73D2` | `0xC733A..0xC73D2` | Confirmed `INT 21h` byte-index table at `0xC733A` and target pointer table at `0xC739A`. |
| `0xC73D2..0xC7650` | `0xC73D2..0xC7650` | Confirmed DOS-like `INT 21h` dispatcher installed via `C000:0006`; it records `AH` at RAM `0x6F0B`, dispatches selected functions, and returns with `iret`. |
| `0xC7650..0xC8A28` | `0xC7650..0xC8A28` | Probable low-level helper code after the `INT 21h` wrappers and before the terminal-mode display record. |
| `0xC8A28..0xC8A58` | `0xC8A28..0xC8A58` | Confirmed terminal-mode display record. |
| `0xC8A58..0xC8AA6` | `0xC8A58..0xC8AA6` | Confirmed terminal-mode state helpers around RAM byte `0x1106`. |
| `0xC8AA6..0xC8BB0` | `0xC8AA6..0xC8BB0` | Confirmed diagnostic display record and embedded command strings. |
| `0xC8BB0..0xC8C70` | `0xC8BB0..0xC8C70` | Confirmed diagnostic display/call wrappers plus fixed I/O probes at ports `0xE0`/`0xE1` and `0xEC..0xEF`. |
| `0xC8C70..0xD0000` | `0xC8C70..0xD0000` | Probable high C000-relative firmware routines after diagnostic wrappers, with small inline dispatch/bitmask tables. |
| `0xD0000..0xD1A6F` | `0xD0000..0xD1A6F` | Probable `D000`-segment document-list/runtime helper code leading up to the document-list template prefix. |
| `0xD1A6F..0xD1A80` | `0xD1A6F..0xD1A80` | Probable small binary prefix immediately before the document-list display resource. |
| `0xD1A80..0xD1ACF` | `0xD1A80..0xD1ACF` | Confirmed `D000`-segment document-list screen template containing `LIST OF DOC.`. |
| `0xD1ACF..0xD6FA3` | `0xD1ACF..0xD6FA3` | Probable mixed `D000`-segment editor/printing/file helper code and inline resource islands, including merge/address strings, printer escapes, and thesaurus labels. |
| `0xD6FA3..0xD7183` | `0xD6FA3..0xD7183` | Probable `D000`-segment high-byte editor status/control tables before visible status-line labels. |
| `0xD7183..0xD7268` | `0xD7183..0xD7268` | Probable `D000`-segment editor status labels and address-book field resources, including `OFF`/`CHA`/`LIN`/`CAPS`, `CODE`/`PRNT`/`FULL`, `WAIT`, `NAME`, `TEL`, `FAX`, and `ADRS`. |
| `0xD7268..0xD789B` | `0xD7268..0xD789B` | Probable `D000`-segment editor/address-book helper code ending immediately before the startup banner resource. |
| `0xD789B..0xDBBEB` | `0xD789B..0xDBBEB` | `D000`-segment startup banner, first menu resources, word-processor UI strings, file/storage UI, spell/thesaurus/grammar UI, DreamLink/FDD strings. |
| `0xDBBEB..0xDC000` | `0xDBBEB..0xDC000` | MAME-declared 8x8 font stream for T200. Printable ASCII starts at file `0xDBBEB` with code `0x20`. |
| `0xDC000..0xE04C2` | `0xDC000..0xE04C2` | Probable additional fixed-width glyph/bitmap streams after the MAME-declared font; repeated font-like blocks continue up to the `E04C` code entry. |
| `0xE04C2..0xF1FE0` | `0xE04C2..0xF1FE0` | Probable `E04C`-segment application runtime code. |
| `0xF1FE0..0xF4FF0` | `0xF1FE0..0xF4FF0` | Probable `F1FE`-segment file-server/application code. |
| `0xF4FF0..0xF5000` | `0xF4FF0..0xF5000` | Confirmed small `FILE SERVER` display resource. |
| `0xF5000..0xF538E` | `0xF5000..0xF538E` | Copy route and built-in/card/disk selector resources. |
| `0xF538E..0xF5722` | `0xF538E..0xF5722` | `EROMCARD.X`, applications, system setup, editor preferences, and ROM-card errors. |
| `0xF5722..0xF65F8` | `0xF5722..0xF65F8` | Word-processor menu, printer/RS-232 setup, symbol allocation, copy direction, DreamLink, FDD, password, and filename-change resources. |
| `0xF65F8..0xF7D00` | `0xF65F8..0xF7D00` | Organizer menu, calculator errors, calendar/scheduler/world-clock/address-book display-script resources. |
| `0xF7D00..0xF7E42` | `0xF7D00..0xF7E42` | Probable address-book/world-clock index, character-order, and selector tables after address-book UI resources. |
| `0xF7E42..0xFAF30` | `0xF7E42..0xFAF30` | Confirmed `ADDRESS.ODB` / `ORGAN[ADDRESS]` records and world-city/country/timezone database ending after `ZURICH` / `Switzerland`. |
| `0xFAF30..0xFB220` | `0xFAF30..0xFB220` | Probable large bitmap-like digit glyphs and zero-filled spacing before alarm resource prefix records. |
| `0xFB220..0xFB640` | `0xFB220..0xFB640` | Schedule/daily alarm display-script resources immediately before the reset trampoline. |
| `0xFB640..0xFB64E` | `0xFB640..0xFB64E` | Reset trampoline. |
| `0xFB64E..0xFFFF0` | `0xFB64E..0xFFFF0` | Confirmed all-`0xFF` padding before the reset vector. |
| `0xFFFF0..0x100000` | `0xFFFF0..0x100000` | CPU reset vector and trailing bytes. |

## Reset Chain

The CPU reset vector is at physical/file `0xFFFF0`:

```asm
FFFF:0000  cli
FFFF:0001  jmp far FB64:0000
```

The reset trampoline at physical/file `0xFB640`:

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
```

Early startup writes bank/LCD/control ports before setting stack and checking
warm retained state:

```asm
C000:002A  mov al,17
C000:002C  out 10,al
C000:002E  mov al,01
C000:0030  out 16,al
C000:0032  mov al,00
C000:0034  out 17,al
C000:004A  mov al,40
C000:004C  out 00,al
```
