# SPE Python 3 Port

This repository is an unofficial port of **SPE (Stani's Python Editor) 0.8.4.h** to modern Python and wxPython.

Original SPE author: Stani / www.stani.be

## Port goals

- Python 3.12
- wxPython Phoenix
- Windows
- Linux

macOS support is currently out of scope.

The objective is to preserve the original SPE user interface, architecture and behaviour as closely as practical while restoring compatibility with modern Python and wxPython.

## Current status

The Windows port is under active development.

Core functionality already tested includes:

- application startup and clean shutdown
- editing and syntax highlighting
- open/save/reload
- encoding-aware Python source loading
- execution and Output window
- integrated Python Shell
- preferences and workspace persistence
- source browser and navigation
- autocomplete and call tips
- documentation
- UML display/export
- Find/Replace
- Import into Shell, including package-relative imports

Linux remains a target platform but has not yet been runtime-tested on the current port.

See `SPE_PORT_CHANGELOG.md` for detailed implementation and test notes.

## Origin

This project is based on SPE 0.8.4.h, originally written for Python 2 and wxPython Classic.

The repository also contains third-party components that were bundled with the original SPE distribution. These retain their original copyright and license notices.

See `THIRD_PARTY_NOTICES.md`.

## License

The main SPE application is treated as licensed under the GNU General Public License, version 2 or (at your option) any later version (`GPL-2.0-or-later`), consistent with SPE's original About documentation and historical distribution metadata.

Some bundled components use different compatible licenses and retain their own notices.

The `sm/` support library contains its own original licensing notice and should not be assumed to be covered by the SPE GPL notice.

The complete GNU GPL version 2 license text is provided in `LICENSE`. The original SPE notices permitting use under version 2 or any later version remain authoritative for the `GPL-2.0-or-later` grant.

## Port modifications

Python 3 / wxPython Phoenix compatibility changes in this repository are modifications to the original sources. Original copyright and attribution notices have intentionally been retained.

This is not an official continuation endorsed by the original SPE author.
