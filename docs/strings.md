# String Landmarks

Use:

```sh
tools/rom2.py strings --start 0xc0000 --end 0x100000 -n 12
```

## Firmware And Diagnostics

| File offset | Physical | Notes |
| ---: | ---: | --- |
| `0xC13EA` | `0xC13EA` | Internal software error/reset message. |
| `0xC1793` | `0xC1793` | Trap set/removed status text. |
| `0xC26AD` | `0xC26AD` | Hex digit table. |
| `0xC2888` | `0xC2888` | Device-access error prompt. |
| `0xC3A76` | `0xC3A76` | Keyboard/calculator-layout-looking character table. |
| `0xC8A34` | `0xC8A34` | Terminal mode prompt. |
| `0xC8AB2` | `0xC8AB2` | Diagnostic banner: `Diagnostic 30CAB192       98May11`. |
| `0xC8AF1` | `0xC8AF1` | Diagnostic `Mxxxx:yyyy` memory dump help. |
| `0xC8B50` | `0xC8B50` | Diagnostic I/O dump/out command help. |
| `0xC8B86` | `0xC8B86` | Diagnostic card/COM/spell command help. |

## Built-In Applications

| File offset | Physical | Notes |
| ---: | ---: | --- |
| `0xB2A44` | `0xB2A44` | Typin' Time title. |
| `0xB2B35` | `0xB2B35` | `Version 1.0 for DreamWriter`. |
| `0xB2B57` | `0xB2B57` | Almena Keyboard Training Systems copyright line. |
| `0xB3036` | `0xB3036` | Database file marker/string area begins. |
| `0xB3240` | `0xB3240` | Database main menu title. |
| `0xB3976` | `0xB3976` | Spreadsheet file marker/string area begins. |
| `0xB3BC2` | `0xB3BC2` | Spreadsheet main menu title. |

## Main Word Processor And Storage UI

| File offset | Physical | Notes |
| ---: | ---: | --- |
| `0xD1AA8` | `0xD1AA8` | `LIST OF DOC.` document-list text. |
| `0xD3A97` | `0xD3A97` | `:MERGE.FIL` filename. |
| `0xD3C51` | `0xD3C51` | `H:ADDRESS.ODB` filename. |
| `0xD6D49` | `0xD6D49` | `Synonyms for ` thesaurus UI string. |
| `0xD789B` | `0xD789B` | `INITIALIZING`. |
| `0xD78B3` | `0xD78B3` | Word processor startup banner. |
| `0xD791D` | `0xD791D` | CorrectSpell startup banner. |
| `0xD79E4` | `0xD79E4` | Concise International Electronic Thesaurus banner. |
| `0xD7AB3` | `0xD7AB3` | `ORGANIZER MENU`. |
| `0xD7AE7` | `0xD7AE7` | `WORD PROCESSOR MENU?`. |
| `0xD7BFA` | `0xD7BFA` | `Dreamlink or the Floppy Drive`; T200-specific storage wording. |
| `0xD84B7` | `0xD84B7` | `No disk is in the FDD`. |
| `0xD8563` | `0xD8563` | Disk format/compatibility error. |
| `0xD8627` | `0xD8627` | Card or floppy write-protect error. |
| `0xD8AF2` | `0xD8AF2` | Directory selector includes `Built-in, Card, DreamLink, or FDD`. |
| `0xD8D60` | `0xD8D60` | Spell-check run screen. |
| `0xD9002` | `0xD9002` | Work-memory-full copy/move prompt. |
| `0xD9CD8` | `0xD9CD8` | `FDD-M` label in the DreamLink/FDD storage UI cluster. |
| `0xD9D3D` | `0xD9D3D` | `Only FDD is available`. |

## ROM Card, DreamLink, And Organizer

| File offset | Physical | Notes |
| ---: | ---: | --- |
| `0xD9CC5` | `0xD9CC5` | DreamLink UI cluster. |
| `0xF5606` | `0xF5606` | `SAVE ONLY TEXT ON FDD:  {OFF} {ON}` setup preference. |
| `0xF5676` | `0xF5676` | `Can not open EROMCARD.X`. |
| `0xF5D3E` | `0xF5D3E` | DreamLink direction/status string. |
| `0xF5DA2` | `0xF5DA2` | Copy direction resources begin for Built-in/FDD, FDD/Built-in, Card/FDD, FDD/Card, DreamLink/FDD, and FDD/DreamLink. |
| `0xF5F8E` | `0xF5F8E` | `Only DreamLink is available`. |
| `0xF6006` | `0xF6006` | `Only FDD is available`. |
| `0xF601C` | `0xF601C` | `FDD is not available`. |
| `0xF677E` | `0xF677E` | Calendar display-script text area. |
| `0xF6AF2` | `0xF6AF2` | Weekly scheduler display-script text area. |
| `0xF6DAA` | `0xF6DAA` | World Clock display-script text area. |

## Fonts And Bitmaps

| File offset | Physical | Notes |
| ---: | ---: | --- |
| `0xD7AFA` | `0xD7AFA` | 36x34 1bpp rounded button bitmap referenced by nearby `FF 42` records. |
| `0xD7BA4` | `0xD7BA4` | 24x7 first-menu small bitmap/text resource. |
| `0xD7BB9` | `0xD7BB9` | 24x7 first-menu small bitmap/text resource. |
| `0xDBBEB` | `0xDBBEB` | MAME-declared 8x8 T200 font, printable ASCII first code `0x20`. |
