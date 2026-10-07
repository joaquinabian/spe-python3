# Third-party and historical licensing notices

SPE 0.8.4.h bundled several external projects and adapted source files. Their original copyright and licensing notices remain applicable.

This document is an index. The notices in the individual source files and bundled license files are authoritative.

## SPE application

Original author: Stani / www.stani.be

The main SPE application is documented by the original `doc/about.htm` as GNU GPL version 2 or later.

There is historical inconsistency in the source metadata: `info.py` also contains an LGPL 2.1-or-later notice. That original notice has been retained unchanged.

## sm/

`sm/` is Stani's reusable support library.

Its original `sm/__init__.py` explicitly states that `sm.*` is not released under the GPL and permits use and adaptation subject to attribution to the author and website.

That notice is non-standard and is retained unchanged. This repository does not attempt to relicense `sm/`.

## NotebookCtrl

File:

`sm/wxp/NotebookCtrl.py`

Authors include Andrea Gavana and Julianne Sharer.

The file states that NotebookCtrl is distributed under the wxPython License.

## StyledTextCtrl adaptation

File:

`sm/wxp/stc.py`

Original code is credited to Robin Dunn / Total Control Software.

License stated in the source: wxWindows License.

## STC Style Editor

File:

`dialogs/stcStyleEditor.py`

Authors/adapters include Riaan Booysen, Vlad and Stani.

License stated in the source: wxWindows License.

## XRCed

Directory:

`plugins/XRCed/`

Copyright: Roman Rolinsky.

License: BSD-style license.

The complete original license is retained in:

`plugins/XRCed/license.txt`

## wxGlade

Directory:

`plugins/wxGlade/`

Copyright: Alberto Griggio and contributors.

License: MIT License.

The complete original license is retained in:

`plugins/wxGlade/license.txt`

## Kiki

Directory:

`plugins/kiki/`

Copyright: Project 5.

License: GNU GPL version 2 or later.

The original license notice is present in `plugins/kiki/kiki.py`.

## WinPdb / rpdb2

Directory:

`plugins/winpdb/`

Copyright: Nir Aides.

License: GNU GPL version 2 or later.

The original license notice is included in the bundled source.

## PyChecker

Directory:

`plugins/pychecker/`

Copyright notices in the bundled source include MetaSlash Inc. and later portions credited to Google Inc.

The original source notices must be retained. Historical PyChecker distributions identify the project as BSD-licensed.

## PyChecker2

Directory:

`plugins/pychecker2/`

This historical code includes material with multiple origins.

In particular:

`plugins/pychecker2/symbols.py`

states that substantial portions originate from the Python 2.2 distribution and are covered by the corresponding Python license.

Original notices must be retained.

## General rule

Nothing in this document replaces or overrides licensing notices contained in individual files or bundled license files. Those notices should remain intact when redistributing this repository.
