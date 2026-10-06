# SPE 0.8.4.h Port Analysis

The port appears feasible without changing SPE’s overall architecture. The main obstacle is that startup loads much of the editor before showing the window, including a bundled custom notebook, every standard tab, and an empty document with its sidebar, UML view, and documentation view.

No files were modified during the analysis. I inspected the source and parsed it in memory with Python 3.12.2, without importing or launching SPE. Of **316 Python files** outside `venv`, **101 fail Python 3 syntax parsing**; 75 of those are under `plugins`. Passing this scan does not establish runtime compatibility. The active interpreter has no `wx` package, and the repository’s virtual environment contains only pip support, so the minimum startup scope below is a static assessment.

**1. Startup beginning at `SPE.py`**

The main path is:

```text
SPE.py
  ├─ Windows pythonw stdout handling
  ├─ wxversion / wx availability check
  ├─ info.py
  │    └─ sm.osx → sm/__init__.py → sm/python.py
  ├─ sm.wxp.smdi
  │    ├─ sm/wxp/__init__.py → wx.py shell/crust/filling
  │    ├─ singleApp.py
  │    └─ NotebookCtrl.py
  ├─ Menu.py → wxgMenu.py
  ├─ Parent.py ↔ Child.py
  ├─ optional Blender and Crypto imports
  ├─ command-line options and defaults.cfg
  ├─ user preferences and shortcut script
  └─ smdi.App(...)
       └─ wx.App initialization → OnInit()
            └─ selected ParentFrame
                 ├─ Parent.Panel
                 ├─ menu, toolbar, status bar
                 └─ Parent.Panel.__finish__()
                      ├─ icons and all standard tabs
                      ├─ preferences/workspace restoration
                      └─ restored documents or an empty Child.Panel
                           ├─ sidebar and checker panel
                           ├─ StyledTextCtrl editor
                           ├─ UML canvas
                           └─ documentation panel
            └─ Show / SetTopWindow → MainLoop
```

The application construction is in [SPE.py](SPE.py), with frame selection and `OnInit()` in [smdi.py](sm/wxp/smdi.py) at line 1297.

With default preferences:

- Windows selects `MdiSashTabsParentFrame` and `MdiSashTabsChildFrame`.
- Linux and macOS select the split interface.
- Windows/macOS use the bundled Andrea notebook implementation; Linux selects the native notebook wrapper, but **still imports the bundled notebook module**.
- Single-instance mode defaults to disabled, but `singleApp.py` is imported regardless.
- Blender’s tab is omitted when Blender is unavailable.

[Parent.py](Parent.py) at line 146 discovers and imports tabs dynamically. Its workspace restoration creates an empty document if none are open.

The package name `_spe` is significant: many modules explicitly import `_spe.info`, `_spe.help`, and `_spe.tabs`. `info.py` inserts both the application directory and its parent into `sys.path`. Preserve this arrangement initially, while fixing Python 2’s implicit relative imports.

**2. Python 2 constructs incompatible with Python 3.12**

| Issue | Concrete examples | Migration needed |
|---|---|---|
| Print statements | `SPE.py`, `Parent.py`, `Menu.py`, `sm/spy.py` | Convert to function calls; preserve output spacing where relevant |
| Old exception syntax | `info.py`, `Child.py`, `view/documentation.py`, style editor | `except Exception as message` |
| Tuple-unpacking parameters | `NotebookCtrl.py`: `MakeGray((r,g,b), ...)`; `tabs/Recent.py`: unpacking lambda | Unpack inside the function |
| Assignment to reserved constants | `sm/wxp/__init__.py`, `sm/wxp/stc.py` assign `True` and `False` | Remove obsolete compatibility blocks |
| Implicit relative imports | `sm/__init__.py`: `from python import *`; `smdi.py`: `import singleApp`, `import NotebookCtrl` | Explicit package-relative imports |
| Removed built-ins | `execfile`, `apply`, `xrange` | Targeted replacements retaining execution namespaces |
| Removed dictionary methods | `has_key` throughout core and notebook | Membership tests |
| Removed type names | `types.UnicodeType`, `ListType`, `ClassType`, etc. | Appropriate Python 3 types/checks |
| Removed integer limit name | `Child.py`: `sys.maxint` | `sys.maxsize`, checking intended wx list usage |
| Old execution syntax | `sm/scriptutils.py`: `exec codeObject in mainDict` | `exec(codeObject, mainDict)`; update profiling strings too |
| Changed collection behavior | Notebook image processing indexes/mutates `map` results | Explicit sequences or direct byte processing |
| Changed division | Status-bar centering and notebook geometry | Integer division where pixel coordinates require integers |
| Removed `string` functions | `string.zfill`, `lower`, `split`, `join` | String methods |
| Changed text/bytes behavior | Editor loading/saving, compressed notebook images, process output | Decode/encode at boundaries |

Important module changes include:

- `ConfigParser` → `configparser`; replace `readfp()` with `read_file()`.
- `thread` → `_thread`.
- `SimpleXMLRPCServer` → `xmlrpc.server`.
- `xmlrpclib` → `xmlrpc.client`.
- `cPickle` → `pickle`.
- `cStringIO` → `io.StringIO` **for text**, `io.BytesIO` **for image/binary data**.
- `cgi.escape` in `tabs/Output.py` → `html.escape`, preserving the original quoting behavior.
- `inspect.getargspec` / `formatargspec` in editor call tips → a carefully adapted `inspect.signature` implementation.
- Python 3.12 removes `imp` and `distutils`, both used in optional legacy features. [Python 3.12 changes](https://docs.python.org/3.12/whatsnew/3.12.html)

The biggest nonmechanical issue is [Child.py](Child.py) at line 804: it imports and calls the removed `compiler` package. Its realtime syntax-checking path needs a small replacement using `compile()` or `ast.parse()`, preserving error reporting. The bundled `pychecker2` relies much more extensively on the old compiler AST and needs separate treatment.

Encoding deserves focused verification: existing code assumes Python 2 strings in several places. Scintilla positions also need care when indexing Python strings containing non-ASCII text.

**3. wxPython Classic incompatibilities**

| Existing use | Location | Phoenix direction |
|---|---|---|
| `wxversion.ensureMinimal()` | `SPE.py` | Remove Classic version selection; validate the installed Phoenix version |
| `wx.animate.GIFAnimationCtrl`, `GetPlayer()` | `Menu.py` | Adapt the existing throbber to `wx.adv.AnimationCtrl` |
| `wx.SashLayoutWindow`, `wx.LayoutAlgorithm`, sash constants/events | `smdi.py`, `Parent.py`, `Menu.py` | Use `wx.adv` equivalents |
| `wx.PyControl` | `NotebookCtrl.py` | Use `wx.Control` with the existing subclass behavior |
| `wx.SystemSettings_GetColour/GetFont/GetMetric` | `NotebookCtrl.py` | `wx.SystemSettings.GetColour/GetFont/GetMetric` |
| Image factory functions | Notebook and image helpers | Constructor equivalents such as `wx.Bitmap(image)` and `wx.Image(stream)` |
| Default Python encoding getters/setters | `info.py`, `Child.py`, `Parent.py` | Explicit text conversion at file boundaries |
| `GetPyData` / `SetPyData` | `SmFilling`, realtime tree wrapper, editor | `GetItemData` / `SetItemData`, preserving wrapper behavior |
| Old toolbar/menu/list overload names | `Menu.py`, notebook, tabs, checker wrapper | Current overloads; verify any retained compatibility aliases |
| Old file-dialog flags | Core and helpers | `wx.FD_*` equivalents |
| Old `wxPython.*` namespace | Shortcut help, images, ActiveX helper | Current modules where the feature is still supported |

The official migration guide confirms the namespace, Unicode, image-buffer, tree-data, and version-selection changes. [Phoenix migration guide](https://docs.wxpython.org/MigrationGuide.html)

Several distinctions prevent unnecessary changes:

- **`wx.PyCommandEvent` remains supported.** The notebook’s custom event class should not be replaced merely because of its name.
- `wx.NewId()` is deprecated, rather than an immediate startup blocker.
- `wx.gizmos` has had a transitional compatibility module; verify the target release rather than assuming every import fails. Its replacement implementations live under `wx.lib.gizmos`.
- `PythonSashSTC` references `DynamicSashWindow`, but the actual `PythonSTC` alias currently selects `PythonBaseSTC`. The unused class must still be importable.
- `wx.lib.evtmgr` and `wx.lib.ogl` remain documented Phoenix components; they can preserve SPE’s event and UML architecture. [Event manager](https://docs.wxpython.org/wx.lib.evtmgr.html), [OGL](https://docs.wxpython.org/wx.lib.ogl.html)

Additional runtime checks are needed for `tabs/Shell.py`’s unguarded `SetUseAntiAliasing()`, legacy keyword arguments, and custom notebook painting.

There is also an existing global override: `smdi.App` replaces `wx.Bitmap` with SPE’s image-path resolver. Preserve the resource behavior, but check its interaction with Phoenix library code and bitmap constructors. This is a compatibility hotspot.

**4. SPE code versus bundled third-party code**

| Category | Files/directories |
|---|---|
| SPE application | `SPE.py`, `info.py`, `Parent.py`, `Child.py`, `Menu.py`, `help.py`, `tabs/`, `sidebar/`, `view/`, shortcuts and application dialogs |
| Generated SPE UI source | `wxgMenu.py`, `wxgChild.py`, generated portions of other modules; associated `.wxg` files |
| Stani’s reusable support library | Most of `sm/`, including the SDI/MDI framework, realtime controls, utilities, and UML integration |
| Embedded third-party notebook | `sm/wxp/NotebookCtrl.py`, credited to Andrea Gavana and Julianne Sharer |
| Adapted third-party editor code | `sm/wxp/stc.py`, derived from Robin Dunn’s STC example |
| Adapted dialog code | `dialogs/stcStyleEditor.py`, with Boa/Riaan Booysen/Vlad/Stani attribution |
| Bundled external applications/libraries | `plugins/wxGlade/`, `plugins/XRCed/`, `plugins/winpdb/`, `plugins/pychecker/`, `plugins/pychecker2/`, `plugins/kiki/` |
| SPE plugin integration | `plugins/Pycheck.py`, `plugins/spe_winpdb.py`, `plugins/spe_xrced.py` |
| Local environment | `venv/`; exclude from migration source |

Thus, neither `sm/` nor `plugins/` is a uniform ownership boundary. The repository also explicitly notes a local modification to PyChecker2’s import checks.

**5–6. External and obsolete dependencies**

| Dependency | Role | Assessment |
|---|---|---|
| wxPython Phoenix | Required GUI, STC, shell, HTML, OGL | Main external runtime dependency |
| `wxversion` | Classic installation selection | Removed in Phoenix |
| `compiler` | Editor syntax checking; PyChecker2 | Removed Python 2 standard-library API |
| `win32api` / pywin32 | Optional Windows short-path handling | Guarded import; unnecessary for first startup |
| `Crypto.Cipher.DES` / PyCrypto | Optional debugger encryption probe | PyCrypto is obsolete; compatibility requires more than restoring its import |
| Legacy `Blender` module | Embedded Blender integration | Current Blender uses `bpy`; not a drop-in migration |
| PIL `Image` | Optional `sm/wxp/pil.py` | Modern Pillow uses `PIL.Image`; outside startup |
| Bundled Winpdb/rpdb2 | Debugging | This historical copy needs Python/runtime migration |
| Bundled PyChecker/PyChecker2 | Source analysis | Old bytecode/compiler assumptions incompatible with 3.12 |
| Bundled wxGlade/XRCed/Kiki | Optional tools | Historical copies require separate Python/wx migration |
| Browsers, terminals, file managers | External feature commands | Availability and command syntax are platform-dependent |
| ActiveX/IE and auxiliary `htmlCss` imports | Optional support-library paths | Outside normal startup; availability not established |

PyCrypto’s own site identifies it as unmaintained and obsolete. PyCryptodome offers substantial API compatibility, but installing it alone will not port SPE’s debugger. [PyCrypto status](https://www.pycrypto.org/), [PyCryptodome compatibility](https://www.pycryptodome.org/src/vs_pycrypto)

Modern Blender integration would be a separate feature project because its API is based on `bpy`. [Blender API](https://docs.blender.org/api/current/)

The bundled tools are **present but incompatible**, not missing dependencies. They do not all need to work before the main window can start.

**7. Minimum startup change scope**

An exact smallest set cannot be proven without running against the chosen Phoenix build. For the existing default startup, retaining all standard tabs and the empty document, the following **25 files have identified syntax, import, or compatibility work**:

| Group | Files |
|---|---|
| Entry and application | `SPE.py`, `info.py`, `Parent.py`, `Child.py`, `Menu.py`, `wxgMenu.py` |
| Stani support | `sm/__init__.py`, `sm/python.py`, `sm/osx.py`, `sm/scriptutils.py`, `sm/spy.py`, `sm/uml.py` |
| wx support/framework | `sm/wxp/__init__.py`, `sm/wxp/smdi.py`, `sm/wxp/singleApp.py`, `sm/wxp/NotebookCtrl.py`, `sm/wxp/stc.py`, `sm/wxp/realtime.py` |
| Eagerly imported components | `dialogs/stcStyleEditor.py`, `view/documentation.py`, `plugins/Pycheck.py` |
| Tabs with confirmed Python barriers | `tabs/Browser.py`, `tabs/Find.py`, `tabs/Output.py`, `tabs/Recent.py` |

Additionally, **`tabs/Shell.py` is a likely startup edit**, depending on the selected Phoenix STC API.

The other startup tabs and `sidebar/Browser.py` need runtime verification, but inspection did not establish that every one requires changes.

For the first window milestone:

- Port the checker **panel/wrapper**, while deferring execution of incompatible checker engines.
- Replace the editor’s `compiler` import and checking operation locally.
- Keep optional debugger, designer, regex-tool, and Blender implementation changes outside the startup patch.
- Leave `defaults.cfg`, skins, icons, documentation assets, and `.wxg` files unchanged unless verification exposes a specific problem.
- `wxgChild.py` is not the editor panel used by this startup path.

**8. Incremental migration plan**

1. **Establish a reproducible target.** Choose a Phoenix release supporting Python 3.12 and the target Windows architecture. Use a clean preferences/workspace directory for later launch tests.

2. **Make the startup modules parse.** Apply only targeted syntax and import corrections to the identified files. Preserve class structure, method names, tab order, and frame selection.

3. **Resolve import-time dependencies.** Remove Classic version selection, correct standard-library imports, replace the editor’s `compiler` dependency, and adapt class definitions and default arguments evaluated during import.

4. **Port frame construction.** Update sash/layout APIs, notebook internals, toolbar calls, animation control, resource loading, and integer geometry. Retain Windows MDI and the original split alternatives.

5. **Reach the full original startup milestone.** Require the main window, all standard tabs, and an empty document with sidebar/UML/documentation panels to appear and remain responsive through timer callbacks.

6. **Verify basic editor behavior.** Open/save ASCII and non-ASCII files, preserve encoding and line endings, test undo, shell execution, workspace restoration, source navigation, and clean shutdown.

7. **Restore optional features individually.** Address preferences/help dialogs, source checkers, debugger, wxGlade/XRCed/Kiki, single-instance operation, and finally Blender integration.

The first implementation should be a focused compatibility patch around this startup path. There is no demonstrated need to replace SPE’s frame framework, notebook behavior, shell, or UI layout.
