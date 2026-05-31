# Floppy Support Notes

The DreamWriter T200 is the first mapped DreamWriter target in this repo with
an integrated floppy controller. MAME configures `drwrt200` with an
N82077AA-compatible controller at I/O `0xE0..0xEF`; the driver comment says the
actual part is Intel N82877SL.

## Controller Ports

The ROM's direct FDC code is concentrated in `0xC5900..0xC6A2A`. The register
use matches the normal PC-compatible 82077 layout relative to base `0xE0`.

| Port | Likely register | ROM evidence |
| ---: | --- | --- |
| `0xE2` | Digital output/control register | Written from a shadow byte at RAM `0x6FF6`. `0xC6961` sets bit `0x10` for motor-on; `0xC697E` clears it. |
| `0xE4` | Main status register | Polled before command, result, and sector payload transfers. |
| `0xE5` | Data FIFO | Used for command bytes, parameters, result bytes, and 512-byte sector payloads. |
| `0xE7` | Configuration/control register | Written during reset/configuration at `0xC66FD`. |
| `0xE0..0xE1`, `0xEC..0xEF` | Diagnostic/probe area | Touched by diagnostic helpers at `0xC8C00` and `0xC8C0B`; not yet tied to normal file I/O. |

Run:

```sh
tools/rom2.py io-scan --types code,monitor-code --limit 0
```

to reproduce the direct-port hits after `docs/rom-regions.tsv` has the FDD
cluster split out.

## Core Routines

The wrapper area includes two useful inline landmarks: `VMACRO2 INC` at
`0xC5936` and `NTS 2000` at `0xC59A9`.

| Offset | Current label |
| ---: | --- |
| `0xC66FD` | Reset/configure the controller; writes `0xE2`/`0xE7` and sends setup commands through the FIFO. |
| `0xC6961` | Motor on; sets bit `0x10` in the `0x6FF6` DOR/control shadow and writes `0xE2`. |
| `0xC697E` | Motor off; clears bit `0x10` in the `0x6FF6` shadow and writes `0xE2`. |
| `0xC69A1` | Send one command or parameter byte from `AH` through FIFO `0xE5` after polling `0xE4`. |
| `0xC69B3` / `0xC69E5` | Receive one byte from FIFO `0xE5` after polling `0xE4`. |
| `0xC677C` | Read one sector into RAM buffer `0x6300`. |
| `0xC67E0` | Write one sector from RAM buffer `0x6300`. |
| `0xC6840` | Build the read/write command parameter sequence: drive/head, cylinder, head, sector, size, EOT/GPL/DTL-style bytes. |
| `0xC68EE` | Read the FDC result phase; status bytes are stored near RAM `0x6FF0..0x6FF2`. |
| `0xC5AFA` | Scan directory entries in 0x20-byte records. |
| `0xC5CFF` / `0xC5D00` | File read path; reads sectors through `0xC677C` and copies from buffer `0x6300` to the caller buffer. |
| `0xC5DEE` | File write/export path; copies caller bytes into buffer `0x6300`, with optional text conversion controlled by RAM `0x132E`. |
| `0xC5F11` | Close/finalize write path; handles `0x1A` EOF and directory/allocation updates. |
| `0xC6048` | Delete path; marks the directory entry first byte with `0xE5`. |
| `0xC60AF` | Rename/update path; copies an 11-byte filename into a directory entry. |

## INT 21h Bridge

The floppy routines are reached through the firmware's DOS-like `INT 21h`
layer, not only through direct callers around the controller code. Startup at
`0xC10E1` installs interrupt vector `0x21` to `C000:0006`; that stub jumps to
the real handler at `0xC73D2`.

The handler saves the caller state, records `AH` at RAM `0x6F0B`, clears
status byte `0x15B5`, and dispatches through a byte index table at `0xC733A`
and a target table at `0xC739A`. It returns with `iret` and propagates the
handler carry flag back into the caller's saved FLAGS.

Storage targets now have a useful enum:

| Value | Target | Evidence |
| ---: | --- | --- |
| `0x08` | Built-in storage | Selects work segment `0x1800` and the built-in scan path. |
| `0x09` | PC Card | Selects work segment `0x4000` and card helpers. |
| `0x0A` | DreamLink | Selects the shared external-file path and DreamLink helpers. |
| `0x0B` | FDD | Selects work segment `0x0530`, calls the floppy mount/scan routines, and branches to FDD file operations. |

Key storage state lives in low RAM:

| RAM address | Current meaning |
| ---: | --- |
| `0x133D` | Current selected storage target/drive. |
| `0x6F4F` | Work segment selected for the active target. |
| `0x6F51` | Active storage target for the current file operation. |
| `0x6FE3..` | Handle-to-target cache; DreamLink/FDD handles store directly by handle, while built-in/card handles use an offset. |

Useful `INT 21h` storage entries seen so far:

| AH | Handler path | FDD branch evidence |
| ---: | --- | --- |
| `0x0E` | `0xC749D` -> `0xC3FAD` | Selects target from `DL`; `DL=0x0B` calls `0xC59FA` and `0xC5AFA`. |
| `0x19` | `0xC74A1` -> `0xC3F86` | Reads the current target/drive. |
| `0x1A` | `0xC74A5` -> `0xC3F90` | Sets the DTA/work pointer used by later file calls. |
| `0x3C` | `0xC75B2` -> `0xC40C8` | Create/open variant; FDD branch calls the no-op success helper at `0xC6111` and returns through `0xC5212`. |
| `0x3D` | `0xC75B6` -> `0xC42B0` | Create/truncate/open variant; FDD branch calls `0xC5C3E`. |
| `0x3E` | `0xC75BA` -> `0xC4381` | Close path reaches `0xC56CD`; the FDD branch calls `0xC5F11` to finalize metadata. |
| `0x3F` | `0xC75BE` -> `0xC49B6` | Read path calls `0xC5CFF` / `0xC5D00` for FDD. |
| `0x40` | `0xC75C2` -> `0xC4B14` | Write path calls `0xC5DEE` for FDD. |
| `0x41` | `0xC75C6` -> `0xC4FCD` | Delete/unlink path calls `0xC6048` for FDD. |
| `0x42` | `0xC75CA` -> `0xC4E0C` | Seek-style handle path using origin in `AL` and offset in `CX:DX`. |
| `0x43` | `0xC75CE` -> `0xC5058` | Attribute get/set path; FDD/DreamLink bypass the built-in/card media probe before shared directory-entry handling. |
| `0x44` | `0xC75D2` | IOCTL-style dispatcher with several `AX=0x44xx` subfunctions. |

The path parser at `0xC52BD` initializes `0x6F51` from current target `0x133D`
and updates it from drive prefixes by storing the low nibble of the parsed
drive character. For FDD paths it keeps building the normal 11-byte filename at
`0x6F33`, but has special character handling around `.` and `K`.

There is also a private service entry: `AH=0xFF`, `BL=0xA5`, `DL=0x0B` enters
`0xC43AB` and runs the FDD availability/init path through `0xC59FA`, `0xC5610`,
and `0xC6114`. A far-call wrapper at `0xC3F66` saves registers, calls the
FDD probe entry at `0xC5900`, and returns far.

## Disk Format Clues

The file code looks FAT-like, but the exact on-disk variant still needs a
sector-level trace:

| Observation | Evidence |
| --- | --- |
| 512-byte sectors | `0xC677C` and `0xC67E0` transfer `0x200` bytes through FIFO `0xE5` using RAM buffer `0x6300`. |
| 0x20-byte directory records | `0xC5AFA` scans entries in 32-byte steps and treats `0x00` and `0xE5` as free/deleted entry markers. |
| 8.3-style names | The rename/create paths copy an 11-byte name from RAM `0x6F33` into directory entries. |
| FAT/allocation-like chain handling | The read/write/delete paths call shared helpers around `0xC653D`, `0xC65E0`, and `0xC6660` while walking or updating file data. |
| Text EOF marker | The write path handles `0x1A`, consistent with text-file export behavior. |

## RAM Work Area

Known FDD-related RAM fields:

| RAM address | Current meaning |
| ---: | --- |
| `0x15B5` | Error/status code reported by storage routines. |
| `0x133D` | Current selected storage target/drive. |
| `0x6300` | 512-byte sector buffer. |
| `0x6F00` | Current cylinder/track. |
| `0x6F02` | Current sector number. |
| `0x6F0B` | Last `INT 21h` function number (`AH`) observed by the dispatcher. |
| `0x6F33` | 11-byte filename buffer. |
| `0x6F4F` | Work segment selected for the active storage target. |
| `0x6F51` | Active storage target for the current file operation. |
| `0x6FE3..` | Handle-to-target cache. |
| `0x6FEF` | Selected head/drive byte used in FDC command parameters. |
| `0x6FF0..0x6FF2` | FDC result/status bytes. |
| `0x6FF6` | Shadow for the controller output/control byte written to port `0xE2`. |

## UI And Route Strings

Visible FDD-specific resources include:

| Offset | String/use |
| ---: | --- |
| `0xD7BFA` | `Dreamlink or the Floppy Drive`. |
| `0xD84B7` / `0xD851E` | `No disk is in the FDD`. |
| `0xD8563` | Disk format or compatibility error. |
| `0xD8627` | `Card or FD is write-protected`. |
| `0xD88A2` | Built-in/card/FDD directory switching text. |
| `0xD8AF2` / `0xD8C7B` | Directory selector mentioning Built-in, Card, DreamLink, and FDD. |
| `0xD9CD8` | `FDD-M`. |
| `0xD9D3D` | `Only FDD is available`. |
| `0xF5606` | `SAVE ONLY TEXT ON FDD:  {OFF} {ON}`. |
| `0xF5DA2..0xF5E43` | Copy directions between Built-in, Card, DreamLink, and FDD. |
| `0xF6006` / `0xF601C` | FDD availability messages. |

## Next Traces

The main FDC routine boundaries are now known. The next useful pass is to trace
callers and data flow outward:

| Question | Next trace |
| --- | --- |
| How do UI route choices become `DL`/target values? | Trace route resources around `0xF5000..0xF65F8` back through the file-server code at `0xF1FE0..0xF4FF0` and document-list resources around `0xD84B7..0xD8C7B`, then into the `INT 21h` target enum. |
| What exactly distinguishes the `AH=0x3C` and `AH=0x3D` create/open variants? | Continue from `0xC40C8` and `0xC42B0`, especially their FDD branches through `0xC6111` and `0xC5C3E`, and compare caller setup for `0x6F32` / attributes. |
| What is the full work structure at `0x6F00..0x7007`? | Follow read, write, delete, and rename paths and label every field touched by the FDD service routines. |
| How do storage error codes become strings? | Trace writes to `0x15B5` into the common error display path and map codes such as `0x21..0x23`, `0x44..0x4A`, `0x61`, and `0x62`. |
| What does `SAVE ONLY TEXT ON FDD` change? | Follow the preference behind `0xF5606` into the file write/export path and compare normal document saves with text-only floppy saves. |
