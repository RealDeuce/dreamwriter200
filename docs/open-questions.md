# Open Questions

## Addressing And Banking

| Question | Current evidence | Next step |
| --- | --- | --- |
| Are all normal T200 code paths file-offset-equals-physical after startup? | MAME loads the full 1 MiB ROM at `0x000000`; reset jumps to `FB64:0000`, then to `C000:0000`. | Trace bank writes after `C000:0029` and label which segments are active for far calls. |
| What does startup bank value `0x17` on port `0x10` expose? | `C000:002A` writes `0x17` before setting `0x16/0x17`. | Follow early reads/writes through the low CPU window and compare with retained RAM checks. |

## Diagnostics

| Question | Current evidence | Next step |
| --- | --- | --- |
| Which diagnostic exit keys map to user-visible labels? | The T200 diagnostic chord is confirmed as `F+J+SPACE`. The command parser exits on internal keycodes `0x0B`, `0x02`, and `0x03`, returning to its caller without reset. | Map those internal keycodes to the T200 key names so the ROM CARD/FDD test can name a specific exit key. |
| What exact wake/reset paths can enter the T200 diagnostic? | Normal startup calls `C000:0ABD` at `C000:01FC`, `C000:0230`, and `C000:0275`; that routes through `C000:1454` / `C000:1466`. The interrupt-side path at `C000:03FA` can also route to the same compare and diagnostic setup, but normal reset has interrupts disabled until after the `0000:1005 = 'H'` writes at `C000:00C0` and `C000:0127`. | Trace which hardware event reaches `C000:03FA`, then verify whether NMI or wake can reach it before retained-RAM startup state is rebuilt. |

## Storage

| Question | Current evidence | Next step |
| --- | --- | --- |
| How do built-in, PC Card, DreamLink, and FDD UI choices become file-service targets? | The `INT 21h` handler at `0xC73D2` dispatches storage calls through target values: `0x08` built-in, `0x09` PC Card, `0x0A` DreamLink, and `0x0B` FDD. The confirmed FDD service cluster is `0xC5900..0xC6A2A`, while copy/route resources sit around `0xD84B7..0xD8C7B` and `0xF5000..0xF65F8`. | Trace the selector resources back through the file-server and document-list handlers into `DL`, current target `0x133D`, and active target `0x6F51`. |
| What exactly distinguishes the `AH=0x3C` and `AH=0x3D` create/open variants? | Startup installs vector `0x21` to `C000:0006`, which jumps to dispatcher `0xC73D2`. The table maps `AH=0x3C` to `0xC40C8` and `AH=0x3D` to `0xC42B0`; both parse paths and return through the handle setup path, but their FDD branches differ. | Continue tracing `0xC40C8`, `0xC42B0`, FDD helper `0xC5C3E`, and caller setup for `0x6F32` / attributes. |
| What is the full RAM layout for the FDD work area? | The FDD routines use sector buffer `0x6300`, filename buffer `0x6F33`, cylinder `0x6F00`, sector `0x6F02`, selected head/drive `0x6FEF`, result bytes near `0x6FF0`, DOR shadow `0x6FF6`, active target `0x6F51`, handle-target cache near `0x6FE3`, and error/status at `0x15B5`. | Decode the `0x6F00..0x7007` structure field by field while following read, write, delete, and rename paths. |
| How do FDD error codes map to UI messages? | The FDD path writes codes such as `0x03`, `0x07`, `0x0A`, `0x10`, `0x21..0x23`, `0x2C`, `0x31`, `0x44..0x4A`, `0x61`, and `0x62` to RAM `0x15B5`; nearby resources include no-disk, format, compatibility, and write-protect messages. | Trace the common error display routine from `0x15B5` to the string tables. |
| What does `SAVE ONLY TEXT ON FDD` change? | The setup string appears at `0xF5606`, and FDD copy directions appear around `0xF5DA2..0xF601C`. | Follow the preference bit into the FDD write path and identify whether it changes document conversion, metadata, or file extension handling. |

## Applications

| Question | Current evidence | Next step |
| --- | --- | --- |
| How are Typin' Time, Database, and Spreadsheet launched? | UI/resource strings exist around `0xB2A44`, `0xB3036`, and `0xB3976`. | Find menu entries and far-call/load wrappers that enter these built-in apps. |
| Can the ROM CARD option load `EROMCARD.X` from FDD? | The ROM CARD menu path calls the loader at `E04C:2DDE`. It builds `([0000:1005] + 1):EROMCARD.X`, then `[0000:1005]:EROMCARD.X`, using the filename resource at `0xF538E`; startup initializes `0000:1005` to ASCII `H`, so the normal attempts are `I:EROMCARD.X` and `H:EROMCARD.X`. The byte is a default drive-letter seed whose low nibble matches the target enum, so `K:...` would map to FDD target `0x0B`; however, `AH=0x0E`/`AH=0x19` use `0x133D`, not `0000:1005`, and no normal ROM CARD path found so far writes `0000:1005` to `J` or builds `K:EROMCARD.X`. The diagnostic paths run after the startup `H` writes, and the diagnostic `S0000:1005,4A` command should make the later ROM CARD path try `K:EROMCARD.X`. | Build or mount a floppy image containing a valid root `EROMCARD.X`, run the documented diagnostic poke, and confirm whether the loader reaches the FDD open/read path. |
