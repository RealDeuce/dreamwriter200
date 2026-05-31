# Open Questions

## Addressing And Banking

| Question | Current evidence | Next step |
| --- | --- | --- |
| Are all normal T200 code paths file-offset-equals-physical after startup? | MAME loads the full 1 MiB ROM at `0x000000`; reset jumps to `FB64:0000`, then to `C000:0000`. | Trace bank writes after `C000:0029` and label which segments are active for far calls. |
| What does startup bank value `0x17` on port `0x10` expose? | `C000:002A` writes `0x17` before setting `0x16/0x17`. | Follow early reads/writes through the low CPU window and compare with retained RAM checks. |

## Diagnostics

| Question | Current evidence | Next step |
| --- | --- | --- |
| Is the diagnostic chord still `F+J+SPACE`? | The diagnostic banner/help cluster exists at `0xC8AB2`, but the chord compare has not been mapped for T200. | Search for the row-cache compare routine and test in MAME with power/NMI wake. |
| Where is the T200 diagnostic entry routine? | T400 used a warm IRQ path and a C000 compare helper. | Use xrefs around `0xC8AB2` resources and direct calls around keyboard startup. |

## Storage

| Question | Current evidence | Next step |
| --- | --- | --- |
| How are built-in, PC Card, DreamLink, and FDD storage paths dispatched? | UI strings explicitly mention all four targets. MAME adds an FDC for T200. | Map the directory-selector resources around `0xD84B7..0xD8C7B` back to handlers. |
| Which FDC ports are touched by ROM code? | MAME maps an N82077AA-compatible controller in the T200 I/O map. | Tighten executable regions and run `tools/rom2.py io-scan` against code-only ranges. |

## Applications

| Question | Current evidence | Next step |
| --- | --- | --- |
| How are Typin' Time, Database, and Spreadsheet launched? | UI/resource strings exist around `0xB2A44`, `0xB3036`, and `0xB3976`. | Find menu entries and far-call/load wrappers that enter these built-in apps. |
| Is `EROMCARD.X` loading compatible with the T400 path? | `Can not open EROMCARD.X` appears at `0xF5676`. | Map nearby ROM-card strings and compare header validation with T400 `EROMCARD.X` notes. |
