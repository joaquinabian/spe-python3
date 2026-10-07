# SPE port implementation log

## First incremental startup pass

Target: Windows 11 x64 and Linux x86_64, Python 3.12, wxPython 4.3.1.
Windows is the first runtime platform. macOS functionality is out of scope.

Command rerun after each blocker fix:

```powershell
& .\venv\Scripts\python.exe -B SPE.py
```

The virtual environment reports wxPython `4.3.1 msw (phoenix) wxWidgets 3.3.3`.
Only encountered startup blockers were addressed. Syntax conversions were
limited to the failing construct in the failing module; no whole-project
conversion was performed. Deprecated APIs and warnings that did not stop
startup were left in place.

### Files changed and reasons

| File | Change and encountered blocker |
|---|---|
| `SPE.py` | Converted print statements to Python 3 calls, preserving the trailing-comma output with `end=' '`. Replaced the missing `ConfigParser` import with `configparser` under the existing alias. |
| `info.py` | Converted the import-error handler's exception and print syntax so the module parses. |
| `sm/__init__.py` | Converted its print statement and made the `python` sibling import explicitly relative. |
| `sm/python.py` | Converted its print statement so the shared support library parses. |
| `sm/osx.py` | Converted print statements, exception handlers, and backtick representations as each construct blocked parsing. This is shared OS/filesystem support despite its name. |
| `sm/wxp/__init__.py` | Removed pre-boolean-Python assignments to `True`/`False`; converted print syntax; changed the import-time `wx.OPEN` default to `wx.FD_OPEN`; retained revision attributes with an empty fallback because Phoenix's `crust.__revision__` is absent. |
| `sm/wxp/smdi.py` | Converted print syntax, including the macOS branch solely for parsing. Made the `singleApp` and `NotebookCtrl` sibling imports relative. Windows MDI and Linux split selection logic is unchanged. |
| `sm/wxp/singleApp.py` | Converted its print statement. Mapped missing `thread`, `SimpleXMLRPCServer`, and `xmlrpclib` imports to `_thread`, `xmlrpc.server`, and `xmlrpc.client`, retaining existing local names. Single-instance runtime behavior was not tested. |
| `sm/wxp/NotebookCtrl.py` | Replaced tuple-unpacking function parameters with unpacking inside `MakeGray`; replaced the binary `cStringIO` stream with `io.BytesIO`; mapped `cPickle` to `pickle`. Custom notebook behavior was not redesigned or runtime-tested. |
| `Menu.py` | Converted print syntax. Replaced missing `wx.animate.GIFAnimationCtrl` import with `wx.adv.AnimationCtrl` under the existing local name. The throbber constructor/player calls have not yet been reached and still need runtime adaptation. |
| `wxgMenu.py` | Converted generated event-handler print statements so the module parses. |
| `Parent.py` | Converted print and exception syntax. Mapped missing `ConfigParser` and `thread` imports to their Python 3 equivalents under existing aliases. |
| `sm/scriptutils.py` | Converted executable `exec` statements, exception handlers, and print statements so this eagerly imported helper parses. A temporary indentation error in the second `exec` edit was corrected on the following run. Profiling strings and other runtime-only legacy APIs were not changed. |
| `dialogs/stcStyleEditor.py` | Converted exception and backtick syntax; mapped the missing `ConfigParser` import to `configparser`. This dialog is imported eagerly by `Parent.py`; its UI was not tested or redesigned. |
| `Child.py` | Converted exception and print syntax so parsing progresses to the legacy `compiler` import. The compiler dependency has deliberately not been replaced pending review. |
| `SPE_PORT_CHANGELOG.md` | Added this concise implementation log and review checkpoint. |

### Current startup state

Startup prints the SPE banner and imports `info`, the shared `sm` and `sm.wxp`
packages, the SDI/MDI framework, single-instance helper, custom notebook, menu
modules, and style editor. Importing `Parent` reaches its import of `Child`.
The latest run exits with:

```text
File "Child.py", line 12
    import codecs, compiler, inspect, os, sys, re, shutil, thread, time, types
ModuleNotFoundError: No module named 'compiler'
```

No main window has appeared. `sm/wxp/smdi.App` has not been instantiated.
Linux and macOS were not runtime-tested. No optional plugin implementation
was changed. `SPE_PORT_ANALYSIS.md` is unchanged from the previous task.

### Review checkpoint: realtime syntax checking

`Child._idleCheck()` calls `compiler.parse(source)` at line 804. The returned
tree is not used. This check is enabled by the existing
`CheckSourceRealtime = compiler` preference and feeds status text, error
indicators, and the throbber.

The choices are:

- `ast.parse(source)`: retain a parsing-only check using Python 3's AST parser.
- `compile(source, filename, 'exec')`: also reject compiler context errors,
  which can change when and what the existing checker reports.

Recommendation: use `ast.parse()` as the narrow replacement, retaining the
existing `compiler` preference value and UI. Adapt this method's Python 2
string-type check and exception retention at the same time: Python 3 clears
an `except ... as e` target when the handler exits, but the existing code
uses that exception afterward to mark source errors and save `self.e`.

Implementation stopped before making this behavior choice. Additional
startup blockers are expected beyond it; this is not a claim that imports
or frame construction are otherwise complete.

## Second incremental startup pass: AST syntax checking

The previous review checkpoint was approved: replace only the editor's
`compiler.parse()` use with `ast.parse()`. Bundled compiler-dependent
PyChecker code remains unchanged.

### Files changed and reasons in this pass

| File | Change and encountered blocker |
|---|---|
| `Child.py` | Replaced the `compiler` import/check with `ast.parse()` without executing source. Preserved `CheckSourceRealtime == 'compiler'`. Retained the complete `SyntaxError` object outside its handler, including message, line, offset, and original error text. Preserved status/throbber/error-marker callbacks; refreshed `self.e` even for repeated diagnostics and cleared it on valid input. Removed the obsolete `types.StringTypes` check on an unused local text value. Kept generic unexpected-error reporting. Made recovery safe when `self.e` has not yet been set. Subsequent runs required `_thread` under the existing `thread` alias and `sys.maxsize` instead of `sys.maxint`. |
| `sm/spy.py` | Converted its print statement after `Child` reached this import. |
| `sm/wxp/stc.py` | Removed obsolete assignments to `True` and `False` that prevented parsing. No editor API modernization was performed. |
| `view/documentation.py` | Converted old exception syntax so the eagerly imported documentation panel parses. |
| `SPE.py` | Replaced `readfp()` with `read_file()` and a closed file context. Replaced reached dictionary `has_key()` checks for interface selection and shortcut translation. Replaced shortcut `execfile()` with compilation/execution of the existing bundled shortcut script in the same module globals, retaining its filename and source encoding support. This execution is unrelated to the parsing-only check of user source. |
| `sm/wxp/smdi.py` | Replaced reached `has_key()` checks. Imported `wx.adv` and moved sash-window classes, reached layout/sash constants, the sash event binder, and layout-algorithm calls to that namespace. Retained the Windows MDI and Linux split classes and selection paths. Unreached legacy sash status code remains for a later concrete blocker. |
| `sm/wxp/NotebookCtrl.py` | Replaced reached `SystemSettings_GetColour` and `SystemSettings_GetMetric` calls with class methods. Replaced the removed `SetBestSize()` calls with `SetInitialSize()` in the existing notebook sizing logic. Deprecated APIs that still run were left unchanged. |
| `Parent.py` | Replaced missing `EVT_COMMAND_FIND*` calls with `Bind(wx.EVT_FIND*)` for the existing handlers. Imported `wx.adv` and moved layout-algorithm calls there. Did not remove the encoding preference or change file-encoding behavior. |
| `Menu.py` | Adapted throbber initialization to construct Phoenix's animation control and load its existing GIF explicitly. Replaced missing player/background calls with the control's background colour set from the status bar. Moved layout-algorithm calls to `wx.adv`. Existing animation filenames and status-bar architecture are retained. |
| `tabs/Output.py` | Converted encountered print and exception syntax. Replaced missing `cgi.escape` with `html.escape`, explicitly retaining `quote=False` at both call sites. |
| `tabs/Find.py` | Converted print syntax. Removed the vertical alignment flag from the reached box-sizer item that also uses `wx.EXPAND`: the old layout already ignored that flag, but current wxWidgets rejects the combination. No global assertion suppression was added. |
| `tabs/Browser.py` | Converted print statements so the dynamically imported tab parses. |
| `tabs/Recent.py` | Replaced tuple-unpacking lambda syntax with equivalent basename lookup for the existing case-insensitive sort. |
| `SPE_PORT_CHANGELOG.md` | Recorded the approved parser change, validation, subsequent blockers, and the new review checkpoint. |

### Validation

Focused in-memory checks exercised the actual `Child._idleCheck` method with
recorded UI callbacks, without importing the incomplete application. They
passed for:

- Valid source containing a `raise` statement, demonstrating that it is
  parsed without execution.
- CRLF normalization and retained `SyntaxError.msg`, `lineno`, `offset`,
  and `text`; the existing error marker receives the retained coordinates.
- Repeated errors retaining the latest exception object.
- Invalid-to-valid recovery, error clearing, and stopping the throbber.
- Parsing-only acceptance of top-level `return`, which compilation would
  reject.
- `IndentationError` details, handled as a `SyntaxError` subclass.
- The unchanged `compiler` preference value in the code and `defaults.cfg`.

`SPE.py` was rerun after each blocker fix. Once wx application construction
began, runs used the existing `--debug` option to expose tracebacks otherwise
hidden by wx output redirection. Later command output displayed tracebacks
and the tail of the startup log to omit repeated nonblocking warnings.

All 22 source files changed across both passes parse under Python 3.12.
The final diff whitespace check passes. The final startup rerun confirms
the encoding-setter blocker below.

### Current startup state after this pass

The initial application imports and preference-file loading succeed.
`sm/wxp/smdi.App` enters `OnInit()` and constructs the Windows
`MdiSashTabsParentFrame`, its sash windows and custom notebook, menu,
toolbar, status bar, and all eleven standard tabs: Shell, Locals, Session,
Output, Find, Browser, Recent, Todo, Index, Notes, and Donate.

Startup then fails in `Parent.preferencesUpdate()` while applying the
encoding preference:

```text
File "Parent.py", line 1501, in preferencesUpdate
    wx.SetDefaultPyEncoding(INFO['encoding'])
AttributeError: module 'wx' has no attribute 'SetDefaultPyEncoding'
OnInit returned false, exiting...
```

The main SPE window has not appeared; workspace restoration and the empty
editor document have not been reached. No successful runtime verification
of the editor, notebook interaction, or animation transitions is claimed.
Linux and macOS were not runtime-tested. No bundled PyChecker or other
optional plugin implementation was changed.

### New review checkpoint: encoding preference and file boundaries

Classic SPE changes wx's global conversion codec in response to its
`Encoding` preference. Phoenix no longer supports changing that codec.
`Child.save()` and `Child.revert()` also temporarily change it around
`GetText()`/`SetText()`, and contain additional Python 2 text/bytes assumptions.

Recommendation for review: keep wx/editor text as Unicode, preserve the
existing `Encoding` preference and per-file encoding detection, and apply
those encodings explicitly at file decoding/encoding boundaries. Do not
restore a global wx codec or silently ignore the user's encoding preference.
The meaning of `<default>` and handling of coding declarations/BOMs should
remain explicit when implementing the boundary changes.

Implementation stopped at the first reached encoding setter before making
that behavior decision. The AST syntax-checking change is complete.

## Third incremental startup pass: application encoding preference

The encoding review checkpoint was approved with a narrow scope: retain
Unicode GUI text and SPE's existing encoding preference, remove wx global
codec setting, and change only necessary byte/text boundaries.

### Files changed and exact compatibility changes in this pass

| File | Changes and reasons |
|---|---|
| `Parent.py` | Removed `wx.SetDefaultPyEncoding()` calls from `preferencesUpdate()`. The existing `Encoding` configuration is still parsed and stored in `self.defaultEncoding` for document encoding selection. No replacement wx global codec was introduced. |
| `Child.py` | Removed wx encoding getter/setter calls around `GetText()` and `SetText()`. GUI text remains Unicode. `save()` now uses the returned Unicode string directly instead of the obsolete `types.UnicodeType`/decode branch, retaining encoding validation before overwrite and the existing `codecs.open(..., self.encoding)` file boundary. `revert()` still passes the Unicode text produced by its existing codec reader to the GUI. Updated the default-encoding comment. A focused save check exposed that joining the first two source lines without a separator could merge a declaration with the next line, e.g. produce `latin-1name`; changed that join to preserve the newline. No broader loading/BOM subsystem rewrite was performed. |
| `info.py` | Replaced the deprecated wx encoding getter with SPE metadata `'utf-8'`, which is the same value the installed Phoenix getter returned in previous runs. This removes dependency on wx encoding state without changing this target's existing `<default>` value. Explicit configured encodings and source declarations continue to take precedence. |
| `sm/wxp/NotebookCtrl.py` | Replaced the reached missing `wx.SystemSettings_GetFont()` API with `wx.SystemSettings.GetFont()`. Replaced `xrange()` with Python 3 `range()` after initial document page creation reached it. No notebook redesign was performed. |
| `plugins/Pycheck.py` | Changed only the startup-blocking process event binding from the legacy three-argument `wx.EVT_END_PROCESS(...)` call to `self.Bind(wx.EVT_END_PROCESS, self.OnProcessEnded)`. This is SPE's checker panel integration; no bundled PyChecker/compiler-dependent implementation was ported. |
| `SPE_PORT_CHANGELOG.md` | Recorded this pass, encoding validation, current startup state, and the bitmap review checkpoint. |

The source configuration and its encoding preference were not changed.
There are no wx global encoding getter/setter calls left in the core
encoding paths. No user source is executed by syntax checking.

### Boundary behavior and validation

Startup created an empty document; no actual source file loading or saving
boundary was reached before the new blocker. The existing reader was not
broadly rewritten, and file-loading/BOM compatibility is not claimed to be
complete.

Focused checks exercised the actual `Child.save()` and `getEncoding()`
methods, extracted in memory with GUI collaborators stubbed, writing only
temporary files inside the project and removing each afterward. Checks
passed for:

- A configured CP1252 file containing the euro sign, with CP1252 bytes and
  CRLF line endings preserved.
- A Latin-1 coding declaration taking precedence over configured CP1252,
  including the corrected first-two-line detection and existing Unix EOL
  normalization behavior.
- The existing Phoenix-target `<default>` value of UTF-8 for an undeclared
  document; this is not substituted for a configured or declared codec.
- An ASCII preference rejecting non-ASCII text before overwriting the
  original file.

This verifies the narrow save boundary and preference precedence, not a
complete editor/file I/O port. Remaining raw-byte assumptions in the
unreached source-loading path are deferred until that boundary is exercised.

All 23 source files changed across the three passes parse under Python 3.12.
The final diff whitespace check passes and temporary validation files were
removed.

### Resulting startup state

After each encountered blocker was fixed, `SPE.py --debug` was rerun with
the current virtual environment. Startup now applies preferences, reaches
workspace restoration, creates the initial Windows MDI child/document tab,
and constructs its sidebar, including the SPE checker panel and source
editor. It then attempts the UML canvas and fails in Phoenix OGL:

```text
File "wx/lib/ogl/canvas.py", line 85, in __init__
    self._buffer = wx.Bitmap(1, 1)
File "sm/wxp/smdi.py", line 205, in __call__
    path = os.path.join(self.path,os.path.basename(x))
TypeError: expected str, bytes or os.PathLike object, not int
OnInit returned false, exiting...
```

The main SPE window has not appeared. The default Windows MDI selection
and Linux split selection remain unchanged; Linux and macOS were not
runtime-tested. The final rerun still reaches the same bitmap blocker.

### New review checkpoint: global bitmap override

`smdi.App` globally replaces the native `wx.Bitmap` class with SPE's
filename resolver. `Menu.Tool` and `tabs.Browser.Panel` also assign that
resolver to `wx.Bitmap`. Phoenix OGL now calls the normal width/height
constructor to create a drawing buffer, which the filename-only resolver
cannot handle.

Possible approaches are to extend the global resolver to forward native
constructor overloads, or to keep the native wx class and route SPE skin
asset calls through the existing `app.bitmap` helper. A global callable
replacement can also affect class/type expectations in wx library code.

Recommendation for review: retain native `wx.Bitmap` and use the existing
SPE helper only for skin asset loading, narrowly adapting affected SPE
call sites. Keep the custom notebook, UML canvas, resources, and interface
architecture. No bitmap override or loading changes were made in this pass.


## Pass 4: native wx.Bitmap and main-window milestone (2026-10-06)

### Files changed and exact compatibility changes

- `sm/wxp/smdi.py`: removed the assignment to `wx.Bitmap` during App construction. `self.bitmap` retains the existing SPE Bitmap helper, selected imagePath, basename lookup, bitmap-type argument, and native constructor delegate. Without an imagePath it still uses native wx.Bitmap.
- `Menu.py`: removed the toolbar's global wx.Bitmap assignment; routed its 25 skin bitmap calls through app.bitmap. Passed parent.app.bitmap explicitly into the floating palette panel.
- `wxgMenu.py`: the palette accepts an optional bitmap keyword, removes it before wx.Panel construction, and uses that helper for its 17 asset calls. Its standalone default is native wx.Bitmap. Changed the import icon's backslash path to slash separators so the existing basename resolver works on Linux as well as Windows.
- `tabs/Browser.py`: replaced the global bitmap assignment with self.bitmap, initialized before generated control construction. The two folder buttons and Python source icon use this application helper.
- `tabs/Blender.py`: removed its global bitmap assignment and routed its single logo asset through self.bitmap. No other Blender functionality was ported or tested; this narrow change prevents the optional tab from reintroducing the override.
- `dialogs/winpdbDialog.py`: routed the SPE-owned dialog logo through its existing app reference and app.bitmap. No bundled debugger implementation was changed or tested.
- `sm/uml.py`: after the bitmap fix, the next SPE.py run failed at SetScrollbars because Python 3 division produced floats. Changed only maxWidth/20 and maxHeight/20 to integer division (//), preserving Python 2's integer scrollbar counts.
- `SPE_PORT_CHANGELOG.md`: recorded this pass and its validation/startup result.

No global wx.Bitmap assignments remain in the affected SPE sources. Phoenix and OGL retain native wx.Bitmap, including width/height and image constructor overloads. The existing selected skin resolver and resources remain in place. Full-path image calls that did not depend on skin lookup remain native. No wx.lib.ogl or other installed third-party files were changed. Windows MDI and Linux split interface selections remain unchanged; no macOS-specific work was performed.

### Incremental runs and resulting startup state

1. After removing the bitmap overrides and adapting SPE asset calls, SPE.py --debug progressed beyond OGL's wx.Bitmap(1, 1) buffer creation. The next blocker was sm/uml.py SetScrollbars: argument 3 had unexpected type float.
2. After the integer-division fix, SPE.py --debug completed enough startup to display the Windows MDI main window and initial unnamed document. Windows reported title `SPE 0.8.4.h - [unnamed]`, a nonzero window handle, and IsWindowVisible returned true. The process remains running.

The requested main-window milestone has been reached, so implementation stopped. Startup is not error-free: the event loop reports AttributeError in sm/wxp/NotebookCtrl.py OnPaint (line 4348), because BufferedPaintDC no longer has BeginDrawing. This is the next concrete compatibility issue for a later pass; it was not changed after the window milestone. Existing deprecation and syntax warnings also remain. Editor actions, optional plugins, Linux runtime, and macOS were not tested.

### Validation

All seven source files changed in this pass parse under Python 3.12. A targeted check with the actual installed wxPython verified native wx.Bitmap(1, 1), native wx.Bitmap(wx.Image(2, 2)), and the unchanged SPE helper loading toolbar, folder, and import assets from skins/default. The helper check also asserted wx.Bitmap retained its native class identity. A source search confirmed removal of the global bitmap assignments in the affected files. The final diff whitespace check passed. No persistent test files were added.


## Pass 5: notebook painting and runtime checks (2026-10-06)

### Files changed and compatibility changes

- `sm/wxp/NotebookCtrl.py`: removed OnPaint's obsolete dc.BeginDrawing() and matching dc.EndDrawing(). Retained wx.BufferedPaintDC, the existing background, tab skin, clipping/rendering helpers, insertion marks, and event flow. No GraphicsContext replacement or painting redesign.
- In the same file, subsequent runs exposed three concrete paint incompatibilities, fixed and rerun individually: changed the bitmap validity test from bmp.Ok() to bmp.IsOk(); converted x/y coordinates to int at the nonrotated DC.DrawText boundary (Phoenix rejects floats, while Classic accepted/coerced these pixel positions); restored Python 2 integer division with // in _CalcXRect's close-button offsets. Other rendering paths were left untouched.
- `sm/wxp/stc.py`: the running editor had reported a TypeError on Return because auto-indentation multiplied a string by indent/max(1,self.tabWidth), now a float in Python 3. Changed this one expression to integer division, preserving Python 2 indentation counts. No other editor behavior was refactored.
- `SPE_PORT_CHANGELOG.md`: documented fixes, checks, and the shutdown review checkpoint.

No third-party wxPython code was modified. Changes use shared Windows/Linux source and leave both interface paths intact. Linux was not runtime-tested and macOS remains out of scope.

### Verification and runtime state

Ran SPE.py --debug repeatedly through a temporary in-process harness using the current Python 3.12 / wxPython 4.3.1 environment. The harness scheduled checks in the real wx event loop; application output was filtered, while callback exceptions were collected. No persistent test implementation was added.

Final running-window checks passed with zero captured Python callback exceptions:

- Windows MDI main window appeared with its initial unnamed document.
- A captured window image was visually inspected: the custom document tab, its close button, notebook skin/background, native sidebar/source tabs, and standard bottom tabs rendered visibly.
- Reduced the main window width/height, refreshed it, restored its size, and processed paint events without a paint exception.
- Switched the standard notebook to Shell, Output, Browser, Recent, Todo, Index, and Notes, with separate event-loop intervals and repaint processing between selections. No paint exceptions occurred. These selections were programmatic, not manual mouse clicks.
- Sent Return through the actual source editor key-event handler with `if True:` and an indented `pass`. Result was `if True:\n    pass\r\n    `, retaining indentation without an exception. The temporary document content was cleared afterward.

Both changed source files parse under Python 3.12; final diff whitespace validation passes. Temporary harness and screenshot were removed after inspection. Existing deprecation warnings and duplicate image-handler notices remain.

### Stop for review: shutdown event delivery

The running-window tests now pass, but a subsequent normal frame.Close() reproduced shutdown exceptions:

- A callback delivered through wx.lib.evtmgr reached sm/wxp/smdi.py onFrameSize and attempted LayoutMDIFrame on a deleted MdiSashTabsParentFrame.
- App.OnExit returned None, which Phoenix rejects because it requires an integer result.

An earlier harness that exited MainLoop directly additionally saw a child activation callback access a deleted PythonBaseSTC; that observation is specific to that harness exit sequence and is not claimed as a normal-close reproduction. The final normal-close run reproduced the parent size callback and OnExit return issues above. The test process exited; no new SPE test window was left open.

SingleInstanceApp.OnExit in sm/wxp/singleApp.py currently calls wx.Yield before stopping/waiting for its argument-posting thread and has no explicit return. Fixing the return value is straightforward, but changing event draining and teardown ordering needs review to preserve close/veto/save behavior and single-instance thread shutdown. No shutdown implementation changes were made in this pass. Recommended next review: narrowly define shutdown ordering and event-manager deregistration so pending events cannot invoke destroyed frames, while retaining the existing thread shutdown and returning an integer from OnExit. Do not redesign the notebook or replace wx.lib.evtmgr broadly.


## Pass 6: shutdown compatibility only (2026-10-07)

Investigation found that Parent.onFrameClose called Framework.onFrameClose with destroy=0, bypassing Framework's event-manager deregistration. It deregistered child frames but destroyed the parent while its own size-event subscription remained active. Default SPE startup sets active=True and disables the single-instance server, so the observed default-close size callback cannot be attributed solely to the wx.Yield in the server shutdown branch.

Files changed:

- `sm/wxp/smdi.py`: added eventManager.DeregisterWindow(self) immediately before the parent Destroy(), after close approval and existing child deregistration. The save confirmation/cancellation path returns before this change; event flow for a live parent remains registered. This shared parent close path covers Windows MDI and Linux split interfaces.
- `sm/wxp/singleApp.py`: added return 0 at the end of OnExit, covering both active/non-server and server-owning branches. Preserved wx.Yield, argument-posting thread Stop(), and the existing wait loop; no thread lifecycle or event architecture redesign.
- `SPE_PORT_CHANGELOG.md`: recorded investigation, narrowly scoped fixes, and verification.

Verification used SPE.py --debug in the current Python 3.12 / wxPython 4.3.1 Windows environment with temporary in-memory event-loop harnesses:

1. Normal startup, visible MDI main window, resize/refresh, then frame.Close(): MainLoop returned 0 with no captured callback exceptions and no invalid OnExit return error.
2. Simulated a declined document-save confirmation via the document's confirmSave callback: frame.Close() left the main window visible. Restored the callback, resized/refreshed, and closed normally; no exceptions. This checks cancellation without interacting with a save dialog or writing document content.
3. Enabled SPE's existing single-instance path in memory without changing preferences, on a temporary available localhost port. The main window appeared and closed normally. The actual XML-RPC thread Stop request completed; OnExit returned integer 0 and IsRunning() was False. No captured callback exceptions.

Both changed source files parse under Python 3.12, and diff whitespace validation passes. No persistent harness files were created, and all verification app processes exited. Only these two shutdown issues were addressed. No third-party files or unrelated source were changed. Linux remains untested at runtime; macOS was not tested or modified.


## Pass 7: basic editor operations; output decoding review checkpoint (2026-10-07)

Worked incrementally in the running Windows MDI application using Python 3.12 / wxPython 4.3.1. Reran the affected operation and preceding checks after each concrete blocker. No UI or architectural refactoring, optional plugin port, third-party modification, or macOS work.

### Files changed and exact changes

- `sm/scriptutils.py`: the enabled CheckFileOnSave check reached RunTabNanny, which imported removed cStringIO and then accessed an unbound name. Replaced that import with io and its capture buffer with io.StringIO. Retained the standard-library tabnanny check and output capture/restoration.
- `Parent.py`: in openList only, replaced types.ListType/ types.TupleType with the built-in list/tuple, retaining exact-type comparisons. Disabled universal newline translation on the preliminary text read with newline="" so Child's existing CRLF detection receives original line endings. Retained the subsequent encoding-aware read and configured/declaration-based encoding selection; did not force a new encoding policy. In onFind, extracted the first element of each Phoenix FindText range tuple, retaining the existing search/wrap logic and return position.
- `sm/wxp/NotebookCtrl.py`: DeletePage now removes its timer from _timers with pop(nPage) before stopping/destroying it. Previously the destroyed timer stayed in the list, so repeated close/reopen/close accessed a deleted native timer. The timer and page lists now stay aligned.
- `sm/wxp/realtime.py`: only the reached TreeCtrl.AppendItem and TreeCtrl.SetItemImage has_key calls became membership checks. Other unreached methods remain unchanged.
- `tabs/Output.py`: restored integer scrollbar counts using // in AddText. Replaced removed Windows EXEC_NOHIDE with Phoenix EXEC_SHOW_CONSOLE to retain the existing visible-console execution intent. Linux's EXEC_MAKE_GROUP_LEADER branch remains unchanged. No process-output decoder was added yet.
- `SPE_PORT_CHANGELOG.md`: documented changes, verification, and outstanding decisions.

### Tests performed and results

The temporary harness ran SPE.py --debug and scheduled checks through the actual wx event loop. It used SPE's new/save/close/open paths, menu edit handlers, Find/Replace dialog data and handlers, and the normal run_with_arguments output path. Text entry was programmatic through StyledTextCtrl editing operations and Return key-event delivery, rather than physical keyboard typing. The temporary Python file and backup were inside the project root; they and the harness were removed afterward.

Passed before the output test:

1. Created a new document via Parent.new, inserted/edited a small Python example, and delivered Return to its actual editor key handler. Auto-indentation produced spaces as expected.
2. Colourised the document and verified the Python `if` keyword received STC_P_WORD. This checks basic lexer operation, not every syntax style.
3. Saved through Child.save with the normal CheckFileOnSave preference enabled. The TabNanny check no longer errors. Save was invoked with a supplied filename; native Save As file-picker interaction was not tested.
4. Closed/reopened via SPE's document close and Parent.openList paths. Repeated document closing passed after the timer-list fix. Native Open file-picker interaction was not tested.
5. Saved/reopened a UTF-8-declared source containing all requested characters: ? ? ? ? ? ?. Compared saved UTF-8 bytes and reopened Unicode text successfully. No new codec override was introduced. Other configured encodings, BOMs, and invalid-byte error behavior were not fully exercised in this pass.
6. The new document saved with LF. A separate all-CRLF source was reopened and saved; bytes matched exactly after disabling newline translation in the preliminary read. Mixed endings and CR-only sources were not tested. Existing assertEOL/normalization preferences remain.
7. Used the actual menu undo/redo handlers and asserted text restoration/reapplication.
8. Used the actual menu copy/paste/cut handlers with the Windows clipboard and verified the resulting text. Clipboard tests used ASCII text.
9. Created the Find/Replace dialog, supplied its FindReplaceData, and called SPE's actual find/replace handlers. ASCII match selection and replacement passed after adapting the FindText tuple result. Unicode search ranges, Replace All, and manual dialog-button interaction remain untested.
10. Saved `print("SPE_BASIC_RUN_OK")` and invoked Parent.run_with_arguments(confirm=False, beep=False). The current configured Python process launched after the Output compatibility fixes, but its stdout could not be displayed due to the pending byte/text issue below. This test does NOT pass yet.

All five changed sources parse under Python 3.12; final diff whitespace validation passes. Earlier editor checks completed with no captured callback exceptions. The final execution test produced the concrete callback exceptions below, and its process exited. Existing duplicate image-handler notices remain. Linux runtime remains untested; shared changes retain its architecture.

### Stop for review: redirected process output encoding

Phoenix process stream read() returned bytes. Output.OnIdle passed stdout bytes to html.escape, causing TypeError: a bytes-like object is required, not str. The traceback occurred while OnEndProcess attempted to drain remaining output, so the normal output completion/display path did not finish. Stderr also feeds AddText and needs a consistent byte/text boundary.

Decoding requires an explicit process-output encoding policy, independent of source-file encodings and GUI Unicode. Arbitrary locale-based decoding or silently treating every external program as UTF-8 could corrupt non-ASCII output. Incremental decoding is needed because stream reads may split multibyte sequences.

Recommended approach for review: define UTF-8 for SPE-launched Python stdout/stderr and use per-stream incremental decoders, flushing them on process completion. Preserve source encodings and Unicode GUI strings. Scope this to the Python run/output path, reviewing how to establish that encoding in the child environment without changing unrelated process behavior. No decoding or child-environment changes were made in this pass.

A second callback in that final run exposed Ctrl._deleteItem in sm/wxp/realtime.py still using self.items.has_key(item.id) when realtime sidebar updates removed an obsolete tree item after editing. This concrete compatibility blocker is recorded for the next incremental fix; it was not changed after the output-decoding review checkpoint. Consequently realtime sidebar refresh is not yet fully restored.


## Pass 8: incremental UTF-8 process output and sidebar deletion (2026-10-07)

### Files changed and compatibility changes

- `tabs/Output.py`: imports codecs and creates fresh, separate incremental UTF-8 decoders for stdout and stderr at each launch, with errors='replace'. Native wx process streams stay byte streams; only decoded text reaches HTML escaping/rendering. Empty decoder results (partial characters) are not rendered. OnEndProcess drains all readable remaining bytes using the existing stdout-first/stderr-second polling order, then finalizes each decoder with decode(b'', final=True) before normal process cleanup/completion feedback. Incomplete final byte sequences and invalid bytes display replacement characters rather than raising Unicode errors. Existing stdout spacing/escaping, stderr red/error formatting, Output tab activation, completion feedback, and process flags remain.
- In the same file, sets PYTHONIOENCODING=utf-8 during wx.Execute so the launched Python child inherits UTF-8 stdout/stderr, then restores the prior parent environment value (or removes the temporary key) in finally. SPE's global/application/file encoding preferences are unchanged. Investigation of the installed Phoenix 4.3.1 found wx.ExecuteEnv cannot be instantiated/subclassed and wx.Execute rejects a dictionary for env. The narrowly scoped inheritance/restore approach preserves the existing wx.Process/wx.Execute architecture. The parent environment is temporarily set during the launch call, not permanently changed; this is not an isolated per-child ExecuteEnv object. UTF-8 decoding is scoped to this existing Output/Python execution path, not arbitrary tools elsewhere.
- `sm/wxp/realtime.py`: after the output test passed, separately replaced only Ctrl._deleteItem's self.items.has_key(item.id) with item.id in self.items. No other unreached sidebar methods or bundled tools were ported.
- `SPE_PORT_CHANGELOG.md`: documented implementation, tests, and the remaining font-rendering review point.

### Tests and observed results

Used temporary scripts inside the project root and an in-memory wrapper around SPE.py --debug's real wx event loop, using the current Python 3.12 / wxPython 4.3.1 Windows environment. Actual scripts launched through Parent.run_with_arguments and wx.Process. No permanent test code or fixtures were added.

1. A script printed ASCII, ? ? ? ? ?, ? ? ?, several lines, and non-ASCII stderr. It reported sys.stdout.encoding and sys.stderr.encoding as utf-8. The first isolated run stopped the parent sidebar timer to avoid the previously known deletion blocker; final combined runs left the real timer running after the separate sidebar fix.
2. Forced actual wx stdout/stderr stream reads to one byte per read, in addition to a child writing UTF-8 bytes one at a time with delays. Collected decoded stdout and stderr independently and asserted exact accented/Greek strings on both streams. No split characters were corrupted. Bytes from stderr never completed a partial stdout character, or vice versa.
3. The script wrote invalid bytes directly with os.write to both streams, and ended stdout with an incomplete three-byte UTF-8 character and stderr with an incomplete two-byte character. Invalid sequences and final partial sequences produced U+FFFD on their respective streams without an exception. Completion feedback followed final decoder output.
4. Reran with normal stream reads. Output.ToText contained complete ASCII/Latin/Greek lines, stderr text, and Script terminated. A normal-size output screenshot was inspected: stderr remained red and the existing Output layout/formatting remained in use.
5. Checked parent environment restoration with PYTHONIOENCODING initially absent, and with an existing latin-1 value. The launched Python still printed UTF-8 successfully; the parent value was restored after wx.Execute. The source/file encoding preference was not changed.
6. Multiple successive launches completed and reset IsBusy correctly, including a simple print("SPE_BASIC_RUN_OK") script whose text appeared in the Output tab. Decoders are recreated per run rather than carrying pending bytes into another process.
7. After the separate sidebar change, inserted a temporary source variable and refreshed Explore, then restored the script and refreshed again, explicitly exercising obsolete-item removal. Realtime sidebar updates remained enabled during the final run and no has_key callback occurred.
8. Reran Unicode source save/close/reopen with CRLF, verifying the requested ? ? ? ? ? ? characters and CRLF persistence. Reran actual menu undo/redo and cut/copy/paste plus ASCII Find/Replace using its real dialog data/handlers. All assertions passed.

The final combined harness finished with zero captured callback exceptions and MainLoop returned 0. Closing encountered a normal save-confirmation dialog for the temporary test document; No was selected, and the test app exited. Temporary scripts, backup, harness, and screenshot were removed afterward. Both changed source files parse under Python 3.12 and diff whitespace validation passes. Linux runtime was not tested; Windows MDI and Linux split interface architecture is unchanged. macOS and third-party code were not modified.

### Current state and next review point

UTF-8 subprocess launch/output decoding and the pending sidebar deletion blocker are fixed. Each stream's byte order is retained; stdout is polled/rendered before stderr in each pass as before. There is no guarantee of total chronological ordering across two separate pipes. In the artificial one-byte-read test, stdout/stderr text can interleave in the combined HTML view; independent stream assertions verified correct decoded text. No line-buffering/reordering redesign was introduced.

The screenshot exposed a separate font-rendering limitation: some Greek/euro glyphs appear missing or as bars/blocks with the existing Output tab Courier font, despite Output.ToText containing the correct Unicode characters. This is not a decoding failure. Stopped without changing font selection/rendering or modernizing the UI. Review a narrowly scoped Unicode-capable monospace font/fallback while preserving the existing Output appearance and layout. No font change was made in this pass.
