# ROM Notes Index

These notes track the NTS DreamWriter T200 `drwrt200.bin` ROM image. They are a
first-pass map: confirmed landmarks were checked against the T200 ROM, while
rough/probable regions are placeholders for later tracing rather than audited
function boundaries.

Start here:

| File | Purpose |
| --- | --- |
| [`map.md`](map.md) | Address model, coarse ROM layout, and reset chain. |
| [`rom-regions.tsv`](rom-regions.tsv) | Machine-readable first-pass code/data/resource map. |
| [`strings.md`](strings.md) | Initial string/resource landmarks. |
| [`hardware.md`](hardware.md) | Hardware notes from MAME and first ROM inspection. |
| [`floppy.md`](floppy.md) | FDD controller, filesystem, RAM work area, and storage-route notes. |
| [`open-questions.md`](open-questions.md) | Working questions and next traces. |

Reusable helpers live in [`tools/rom2.py`](../tools/rom2.py) at the repository
root; see [`tools/README.md`](../tools/README.md) for the command reference.

The local MAME driver snapshot in [`mame/nakajies.cpp`](../mame/nakajies.cpp)
is supporting evidence for banking, IRQ, keyboard, LCD, PC Card, floppy, serial,
printer, RTC, and machine configuration notes.
