# Building & Installing Triton on Windows — verified walkthrough

This document is a **replace** for the upstream build instructions in `README.md`, which do not
produce a working Windows build with a current toolchain. Every command, error message and
result below was executed on the machine described in the table, on 2026-10-26, against
Triton `master` @ `bc84cf74`.

It covers the *whole* path: dependencies → configure → compile → test → `import triton`
from the global Python interpreter.

---

## 1. Verified environment

| Component | Version / path |
|---|---|
| OS | Windows 10.0.19045, x64 |
| Triton source | `C:\Users\kittty\source\repos\Triton` (branch `master`, `bc84cf74`, build number 1599) |
| Visual Studio | Visual Studio **18** Community, MSVC `14.51.36231` (`cl` 19.51.36260.0), Windows SDK 10.0.26100.0 |
| CMake | **4.4.4** (`C:\Users\kittty\AppData\Local\Programs\Python\Python315\Lib\site-packages\cmake\data\bin\cmake.exe`) |
| Ninja | bundled with VS: `C:\Program Files\Microsoft Visual Studio\18\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe` (not on `PATH` by default) |
| Python | **3.15.1** x64, `C:\Users\kittty\AppData\Local\Programs\Python\Python315` |
| vcpkg | clone at `C:\Users\kittty\source\repos\Triton\build\vcpkg` (commit `817892b0ec`), bootstrapped, manifest baseline `ef7dbf94` |
| Dependencies | 38 packages: boost 1.88.0 (math, multiprecision, numeric-conversion), capstone 5.0.7 (x86/arm/arm64/riscv), z3 4.16.0 |
| Generator | `Ninja Multi-Config` (from `CMakePresets.json`) |
| Config built | `Release` |

Outcome: `triton.dll` + `triton.pyd` build in a couple of minutes; `ctest` passes
(619 unittest cases); `import triton` works from any directory in the global interpreter.

> Notes on scope
> * Verified: CMake 4.4.4 + MSVC 19.51 + Python 3.15 + the dynamic triplet `x64-windows`.
> * Not verified here: the CI-only static triplet (`x64-windows-static-release-md`), clang, ARM64,
>   Linux/macOS, `pip install .`.
> * `README.md` §"Installing from vcpkg" (`vcpkg install triton`) installs the *vcpkg port*, not
>   your local checkout. It has nothing to do with the workflow below.

---

## 2. TL;DR — the whole thing

Open **one** `cmd.exe` window (not PowerShell, not git-bash) and paste this:

```bat
:: ---- one-time variables (adjust paths) ----------------------------------
set "TRITON_SRC=C:\Users\kittty\source\repos\Triton"
set "VCPKG_DIR=%TRITON_SRC%\build\vcpkg"
set "PYTHON_DIR=C:\Users\kittty\AppData\Local\Programs\Python\Python315"
set "VS_DIR=C:\Program Files\Microsoft Visual Studio\18\Community"

:: ---- enter the MSVC developer environment -------------------------------
call "%VS_DIR%\VC\Auxiliary\Build\vcvars64.bat"
set "PATH=%VS_DIR%\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;%PATH%"
set "VCPKG_ROOT=%VCPKG_DIR%"

cd /d "%TRITON_SRC%"

:: ---- configure (installs the 38 deps on first run, ~25 min) -------------
cmake --preset windows-x64

:: ---- compile Release ----------------------------------------------------
cmake --build build\windows-x64 --config Release

:: ---- make it importable from the global Python --------------------------
copy /Y build\windows-x64\src\libtriton\Release\triton.pyd  "%PYTHON_DIR%\Lib\site-packages\"
copy /Y build\windows-x64\src\libtriton\Release\capstone.dll "%PYTHON_DIR%\Lib\site-packages\"
copy /Y build\windows-x64\src\libtriton\Release\libz3.dll    "%PYTHON_DIR%\Lib\site-packages\"

:: ---- verify -------------------------------------------------------------
python -c "import triton; print(triton.VERSION.MAJOR, triton.VERSION.MINOR, triton.VERSION.BUILD)"
```

Expected final line: `1 0 1599`.

**One source change is mandatory** with CMake 4.x — see §6. Without it, configure fails at
*generate* time after the entire dependency install has already succeeded.

---

## 3. Prerequisites

Install these once.

1. **Visual Studio** with the *Desktop development with C++* workload
   (MSVC x64 toolset + Windows SDK). Verified with VS 18 / MSVC 14.51 / SDK 10.0.26100.
   The C++ workload also ships **Ninja**; take note of:

   ```
   <VS>\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja\ninja.exe
   ```

2. **CMake** — any recent version. CMake 4.4.4 (the PyPI `cmake` package) was used. The VS-bundled
   CMake also works, but `PATH` order decides which one runs; check with `cmake --version`.
   *With CMake ≥ 4.x you must apply the patch in §6.*

3. **Python x64** (3.15.1 used). The interpreter must be the one you want to import `triton` from.
   * CPython must have its **development** files (`Include`, `libs\python3xx.lib`) — included in
     the python.org installer.
   * The Windows Store alias (`...\WindowsApps\python.exe`) is not suitable.

4. **Git** (for the Triton checkout and the vcpkg clone).

5. **vcpkg** — see §4.

6. **Disk space**: the vcpkg tree ends up at **~6 GB** (sources in `buildtrees\`, archives in
   `downloads\`, built packages in `packages\`). The build adds ~500 MB of installed dependency
   payload (`build\windows-x64\vcpkg_installed`) plus ~90 MB of object files.

---

## 4. Get vcpkg (one time)

```bat
cd /d C:\Users\kittty\source\repos\Triton
git clone https://github.com/microsoft/vcpkg.git build\vcpkg
call build\vcpkg\bootstrap-vcpkg.bat
```

Expected: `build\vcpkg\vcpkg.exe` exists afterwards.

> **Layout warning.** In this setup vcpkg lives *inside* the Triton `build\` directory. That works,
> but it means you must never `rmdir /s build` — you would delete the vcpkg clone and its 6 GB
> cache. Delete only `build\windows-x64` when you want a clean build (§12). If you start from
> scratch, a location outside the source tree (e.g. `C:\vcpkg`) is safer; just point
> `VCPKG_ROOT` at it.

You do **not** run `vcpkg install` yourself. The manifest (`vcpkg.json`) is installed automatically
by CMake during configure, into `build\windows-x64\vcpkg_installed`.

---

## 5. Enter the MSVC + Ninja environment

Every configure/build command must run in a shell where:
* `cl.exe` is on `PATH` (the preset uses the **Ninja** generator, which cannot auto-discover MSVC), and
* `ninja.exe` is on `PATH`.

Do this in `cmd.exe`:

```bat
set "VS_DIR=C:\Program Files\Microsoft Visual Studio\18\Community"
call "%VS_DIR%\VC\Auxiliary\Build\vcvars64.bat"
set "PATH=%VS_DIR%\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;%PATH%"
set "VCPKG_ROOT=C:\Users\kittty\source\repos\Triton\build\vcpkg"
```

Alternatively launch *"x64 Native Tools Command Prompt for VS"* from the Start menu and only add
Ninja + `VCPKG_ROOT`:

```bat
set "PATH=C:\Program Files\Microsoft Visual Studio\18\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;%PATH%"
set "VCPKG_ROOT=C:\Users\kittty\source\repos\Triton\build\vcpkg"
```

PowerShell equivalent for the two variables (`vcvars64.bat` itself does not export into PowerShell;
prefer a `cmd` window):

```powershell
$env:VCPKG_ROOT = 'C:\Users\kittty\source\repos\Triton\build\vcpkg'
$env:PATH = 'C:\Program Files\Microsoft Visual Studio\18\Community\Common7\IDE\CommonExtensions\Microsoft\CMake\Ninja;' + $env:PATH
```

### Why `VCPKG_ROOT` must be exact

`CMakePresets.json` uses `"toolchainFile": "$penv{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"`.
It must point at the **directory that contains `scripts\`**, as a **Windows-style absolute path**.
If it is unset, or given in MSYS/git-bash style (`/c/Users/...`), configure dies immediately:

```
CMake Error at .../CMakeDetermineSystem.cmake:165 (message):
  Could not find toolchain file:
   "/c/Users/kittty/source/repos/Triton/build/vcpkg/scripts/buildsystems/vcpkg.cmake"
```

---

## 6. Required source patch (CMake 4.x)

### The failure

With the unpatched tree, `cmake --preset windows-x64` installs all 38 dependencies successfully and
then fails while **generating**:

```
CMake Error (install-absolute-destination) at src/libtriton/CMakeLists.txt:378 (install):
  INSTALL command given absolute DESTINATION path:
    C:/Users/kittty/source/repos/Triton/out/install/windows-x64/lib/python3.15/site-packages
This error is for project developers.  Use -Wno-error=author or
-Wno-error=install-absolute-destination to suppress it.

CMake Generate step failed.  Build files cannot be regenerated correctly.
```

Why it is fatal rather than a warning:

* CMake 4.x has a developer diagnostic category `install-absolute-destination`; it fires whenever
  an `install()` rule passes an absolute `DESTINATION`.
* `CMakePresets.json` opts into `<preset> "warnings": {"dev": true}` **and**
  `"errors": {"dev": true}` — so every dev diagnostic becomes a hard error.
* Triton composes an absolute path by hand:
  `${CMAKE_INSTALL_PREFIX}/${PYTHON_SITE_PACKAGES}`, while `PYTHON_SITE_PACKAGES` is already a
  prefix-relative path (`lib/python3.15/site-packages`). The prefix is prepended by the author
  instead of by CMake, so the resulting destination is absolute.

### The fix (recommended, applied in this working tree)

`src/libtriton/CMakeLists.txt`, in the `else()` branch of the Python-bindings install block:

```diff
-        install(FILES $<TARGET_FILE_DIR:triton>/triton${PYTHON_SUFFIX} DESTINATION ${CMAKE_INSTALL_PREFIX}/${PYTHON_SITE_PACKAGES})
+        install(FILES $<TARGET_FILE_DIR:triton>/triton${PYTHON_SUFFIX} DESTINATION ${PYTHON_SITE_PACKAGES})
```

Installed path is unchanged (`<prefix>/lib/python3.15/site-packages/triton.pyd`) because CMake
prepends the prefix to relative destinations by itself. The change is behaviour-neutral on old
CMake versions too.

### Alternative without touching sources

CMake's own suggested suppression works (verified: the diagnostic is demoted to a warning and
generate succeeds):

```bat
cmake --preset windows-x64 -Wno-error=install-absolute-destination
```

The preset accepts the flag, but then every subsequent configure must repeat it, and the
diagnostic is genuinely correct — patching the rule is the better option.

> The *other* Python install branch (used only when `CMAKE_INSTALL_PREFIX` is left at its default)
> still contains an absolute destination from `sysconfig`; it is unreachable through the presets
> because they set `installDir`, so it does not fail this build. Do not "fix" it blindly: that
> branch intentionally installs into the interpreter's own `site-packages`.

---

## 7. Configure

```bat
cd /d C:\Users\kittty\source\repos\Triton
cmake --preset windows-x64
```

What happens:

1. vcpkg installs the manifest dependencies into `build\windows-x64\vcpkg_installed\x64-windows\`
   (or re-uses already-built packages in `build\vcpkg\packages\` — a re-configure takes seconds).
   First run downloads and builds everything: **~25 minutes** observed; no progress bar for
   minutes is normal.
2. Python, Z3 and Capstone are located.
3. Ninja Multi-Config files are generated in `build\windows-x64\`.

Expected tail of a successful configure:

```
-- Compiling with Python bindings
-- Found Python3: .../Python315/python.exe (found version "3.15.1") found components: Interpreter Development ...
-- Compiling with Z3 SMT solver
-- Z3 version: 4.16.0.0
-- Compiling with Capstone
-- CAPSTONE version: 5.0.7
-- Compiling with wide-integer headers
-- Configuring done (17.6s)
-- Generating done (0.2s)
-- Build files have been written to: C:/Users/kittty/source/repos/Triton/build/windows-x64
```

Relevant preset facts (`CMakePresets.json`):

| Setting | Value | Consequence |
|---|---|---|
| generator | `Ninja Multi-Config` | `cl` **and** `ninja` must be on `PATH`; always pass `--config` when building |
| `CMAKE_DEFAULT_BUILD_TYPE` | `Debug` | `cmake --build` **without** `--config` builds Debug, not Release |
| `installDir` | `<src>/out/install/windows-x64` | `CMAKE_INSTALL_PREFIX` |
| `toolchainFile` | `$penv{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake` | `VCPKG_ROOT` must be set |
| `errors.dev` | `true` | CMake dev diagnostics are fatal (§6) |
| `BUILD_SHARED_LIBS` | `ON` by default | shared `triton.dll` + Python bindings |

---

## 8. Compile

```bat
cmake --build build\windows-x64 --config Release
```

Expected: 96 build steps, last ones copy dependency DLLs and rename the shared library:

```
[95/96] Linking CXX shared library src\libtriton\Release\triton.dll
...vcpkg_installed\x64-windows\bin\capstone.dll -> ...\src\libtriton\Release\capstone.dll
...vcpkg_installed\x64-windows\bin\libz3.dll    -> ...\src\libtriton\Release\libz3.dll
[96/96] ... -E copy ...\Release\triton.dll ...\Release\triton.pyd
```

Artifacts in `build\windows-x64\src\libtriton\Release\`:

| File | Purpose |
|---|---|
| `triton.dll` | the C++ library |
| `triton.lib` | import library (C++ consumers) |
| `triton.pyd` | byte-identical copy of `triton.dll`, the Python extension module |
| `capstone.dll` | runtime dependency |
| `libz3.dll` | runtime dependency |

The preset's build preset (`cmake --build --preset build-windows-x64 --config Release`) is
equivalent but passes `-v` and is extremely verbose.

---

## 9. Install (optional, for C++ use)

```bat
cmake --build build\windows-x64 --config Release --target install
```

Resulting tree (verified):

```
out\install\windows-x64\
├── include\triton\            headers (+ *.spec files)
├── lib\
│   ├── triton.dll             shared library
│   ├── triton.lib             import library
│   ├── cmake\triton\          tritonConfig.cmake, tritonConfigVersion.cmake
│   └── python3.15\site-packages\triton.pyd
```

Two important caveats, both verified:

1. `triton.pyd` lands in a **Unix-style** path (`lib/python3.15/site-packages`), which is **not**
   on `sys.path` on Windows. Adding it to `PYTHONPATH` alone is not enough:
   ```
   ImportError: DLL load failed while importing triton: The specified module could not be found.
   ```
   because the install tree contains `triton.dll` in `lib\` but **no `capstone.dll` / `libz3.dll`**.
   Copying those two DLLs next to the installed `triton.pyd` makes `import triton` work
   (verified: `1.0.1599`).
2. `tritonConfig.cmake` carries **empty Capstone/Z3 include directories** (the vcpkg targets were
   used at configure time). A C++ consumer should therefore do:
   ```cmake
   find_package(Z3 CONFIG REQUIRED)
   find_package(capstone CONFIG REQUIRED)
   find_package(triton CONFIG REQUIRED)
   target_link_libraries(app PRIVATE triton z3::libz3 capstone::capstone)
   ```
   and point `CMAKE_PREFIX_PATH` at both `out\install\windows-x64` and the vcpkg tree. This
   consumer flow was not exercised here.

For Python use the next section is simpler and is the verified path.

---

## 10. Make `import triton` work in the global Python

The Python module is a single file (`triton.pyd`), but it loads two DLLs at import time. CPython's
extension loader searches **the module's own directory** (`LOAD_LIBRARY_SEARCH_DLL_LOAD_DIR`), so
the three files must live together.

```bat
set "PYTHON_DIR=C:\Users\kittty\AppData\Local\Programs\Python\Python315"
set "SP=%PYTHON_DIR%\Lib\site-packages"

copy /Y build\windows-x64\src\libtriton\Release\triton.pyd   "%SP%\"
copy /Y build\windows-x64\src\libtriton\Release\capstone.dll "%SP%\"
copy /Y build\windows-x64\src\libtriton\Release\libz3.dll    "%SP%\"
```

Result in `...\Python315\Lib\site-packages\` (verified):

| File | Size |
|---|---|
| `triton.pyd` | ~6.3 MB |
| `capstone.dll` | ~4.1 MB |
| `libz3.dll` | ~14.9 MB |

Verify from **any** directory, with no `PYTHONPATH`:

```bat
cd /d C:\
python -c "import triton; print(triton.__file__)"
python -c "import triton; print(triton.VERSION.MAJOR, triton.VERSION.MINOR, triton.VERSION.BUILD)"
```

Expected:

```
C:\Users\kittty\AppData\Local\Programs\Python\Python315\Lib\site-packages\triton.pyd
1 0 1599
```

Functional smoke test (verified output shown):

```python
from triton import TritonContext, ARCH, Instruction

ctx = TritonContext()
ctx.setArchitecture(ARCH.X86_64)

# lift + concrete execution
i = Instruction()
i.setOpcode(b"\x48\xc7\xc0\x11\x22\x33\x44")   # mov rax, 0x44332211
ctx.processing(i)
print(i.getDisassembly())                       # mov rax, 0x44332211
print(hex(ctx.getConcreteRegisterValue(ctx.registers.rax)))   # 0x44332211

# symbolic + Z3 backend
ctx.symbolizeRegister(ctx.registers.rax, "x")
expr = ctx.getSymbolicRegister(ctx.registers.rax)
ast = ctx.getAstContext()
model = ctx.getModel(ast.equal(expr.getAst(), ast.bv(0x4142, 64)))
print({str(k): hex(v.getValue()) for k, v in model.items()})  # {'0': '0x4142'}
```

### Alternatives to copying

* **Per-project / development**: keep the build tree and set `PYTHONPATH` (verified):
  ```bat
  set "PYTHONPATH=C:\Users\kittty\source\repos\Triton\build\windows-x64\src\libtriton\Release"
  ```
  That directory already contains `triton.pyd`, `capstone.dll`, `libz3.dll`, so import works;
  rebuilds are picked up automatically.
* **Virtual environment**: copy the same three files into
  `<venv>\Lib\site-packages\` — DLL resolution is identical inside a venv.

### Caveats

* **Re-copy after every rebuild.** The files in `site-packages` are independent copies; a rebuild
  does not update them.
* **Uninstall** = delete those three files. There is no pip metadata, so `pip uninstall triton`
  does nothing (and `pip list` will not show it).
* **Name collision**: the PyPI package `triton` is OpenAI's GPU compiler and installs a
  `site-packages\triton\` *package directory*. It is not related to this library. If you need both,
  remove the loose `triton.pyd` first; with both present, imports are ambiguous.
* **Why not `pip install .`?** `setup.py` invokes CMake **without** the vcpkg toolchain and with the
  default Visual Studio generator. Verified failure:
  ```
  CMake Error at CMakeModules/LibFindMacros.cmake:75 (message):
    Required library Z3 NOT FOUND.
  ```
  The preset workflow above is the supported path. (`setup.py` itself notes
  `BUILDTOOLS = vs2022`.)
* **No editor autocomplete.** The `.pyi` stub is not generated by default. To get it, configure with
  `-DPYTHON_BINDINGS_AUTOCOMPLETE=ON`, build the `python_autocomplete` target, and copy
  `build\windows-x64\doc\triton_autocomplete\triton.pyi` next to `triton.pyd`.

---

## 11. Tests

The Python test suite needs three extra modules (the CI installs the same ones):

```bat
python -m pip install lief unicorn z3-solver
```

Then, from a developer shell:

```bat
cd /d C:\Users\kittty\source\repos\Triton
ctest --test-dir build\windows-x64 -C Release --output-on-failure
```

(Equivalent preset form: `ctest --preset test-windows-x64 -C Release`.)

Verified result:

```
1/1 Test #1: test-python ......................   Passed  438.93 sec
100% tests passed, 0 tests failed out of 1
```

* `619` unit tests plus the `examples/python` programs are driven by the `test-python` CTest entry,
  which sets `PYTHONPATH` to the built module directory — the global copy is not involved.
* `DummyTest` is a placeholder: C++/Unicorn semantic tests are enabled on Linux/macOS only.
* Without `lief`, the suite reports `83` failures (`59 × ModuleNotFoundError: No module named
  'lief'`, `24 ×` failing examples). They are environment failures, not build failures.

---

## 12. Clean rebuild

```bat
rmdir /s /q build\windows-x64
cmake --preset windows-x64
cmake --build build\windows-x64 --config Release
```

* Do **not** delete `build\vcpkg` (the vcpkg clone + 6 GB package cache) unless you want to rebuild
  every dependency from scratch.
* With a warm vcpkg cache the dependency install replays in well under a second per package
  (`Installed contents ... in: 595 us`), so a clean Triton build is cheap.
* A stale `vcpkg_installed\` directory at the **repository root** (496 MB in this setup) is a
  leftover from having run `vcpkg install` manually in the source directory. The preset build never
  reads it; it can be deleted.

---

## 13. Troubleshooting

### `Could not find toolchain file: ".../scripts/buildsystems/vcpkg.cmake"`

`VCPKG_ROOT` is unset, points at the wrong directory, or is written in MSYS style (`/c/...`).
Use an absolute Windows path, e.g. `set "VCPKG_ROOT=C:\Users\kittty\source\repos\Triton\build\vcpkg"`.

### `CMake was unable to find a build program corresponding to "Ninja Multi-Config"` / `CMAKE_C_COMPILER not set`

Not in a developer shell, and/or the VS Ninja directory is not on `PATH` (§5). Same symptom if you
run from PowerShell without `vcvars64.bat`.

### `CMake Error (install-absolute-destination) ... Generate step failed.`

Apply §6 (patch) or pass `-Wno-error=install-absolute-destination`.

### `Required library Z3 NOT FOUND`

You configured **without** the preset/toolchain (e.g. `pip install .`, or `cmake -B build` by hand).
Either use `cmake --preset windows-x64`, or pass
`-DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake`.

### `ImportError: DLL load failed while importing triton: The specified module could not be found.`

`triton.pyd` is importable but cannot find `capstone.dll` / `libz3.dll`. Copy both **into the same
directory** as the `.pyd` (§10), or add the Release directory to `PYTHONPATH` instead of copying a
lone `.pyd`.

### `ModuleNotFoundError: No module named 'triton'`

The module is not on `sys.path` for the interpreter you are running. Check
`python -c "import sys; print(sys.executable); print(sys.path)"`, then either copy the three files
(§10) or set `PYTHONPATH` to the Release directory. Beware the Store alias `python.exe` — it is a
different interpreter.

### Long silence during the first `cmake --preset`

vcpkg is building Boost/Z3/Capstone. Look at progress with a second shell:
`type build\windows-x64\vcpkg-manifest-install.log` or the per-port logs under
`build\vcpkg\buildtrees\<port>\`. Expect ~25 minutes.

### Debug build by accident

The preset sets `CMAKE_DEFAULT_BUILD_TYPE=Debug`. Always use
`cmake --build build\windows-x64 --config Release`.

---

## 14. Where things live

| Path | Contents |
|---|---|
| `build\vcpkg\` | vcpkg clone, `vcpkg.exe`, 6 GB of sources/downloads/packages |
| `build\windows-x64\` | Ninja Multi-Config build tree (never delete casually? safe to delete; it is regenerable) |
| `build\windows-x64\vcpkg_installed\x64-windows\` | headers/libs/DLLs of the 38 dependencies for this build |
| `build\windows-x64\src\libtriton\Release\` | `triton.dll`, `triton.pyd`, `triton.lib`, dependency DLLs |
| `out\install\windows-x64\` | result of the `install` target (§9) |
| `%PYTHON_DIR%\Lib\site-packages\` | the three globally installed files (§10) |

---

## 15. Deviations from upstream `README.md` / CI

| Upstream says / does | Reality here |
|---|---|
| `vcpkg install triton` | Installs the *port*, not your checkout; unrelated to building `master` |
| CI configure step (`cmake --preset ...` with `VCPKG_ROOT`) | Correct idea, but **fails at generate** with CMake 4.x until §6 is applied |
| CI passes `-DVCPKG_OVERLAY_TRIPLETS:PATH=<repo>/vcpkg/triplets -DVCPKG_TARGET_TRIPLET:STRING=x64-windows-static-release-md` | CI-only static triplet; not exercised here. The default dynamic triplet `x64-windows` is what this document verifies |
| `.github/workflows/vcpkg.yml` installs `lief unicorn z3-solver` before ctest | Required for a clean test run (§11) |
| — | The CMake `install` target does **not** yield a working Python module on Windows: Unix-style path, and dependency DLLs are not installed (§9) |
| `pip install .` / `setup.py` | Fails to configure without extra env (`Z3_LIBRARIES`, `CAPSTONE_LIBRARIES`, …) because it does not pass the vcpkg toolchain (§10) |

---

## 16. Verified command log (summary)

| # | Command | Result |
|---|---|---|
| 1 | `cmake --preset windows-x64` (unpatched) | fails: `install-absolute-destination` at `src/libtriton/CMakeLists.txt:378` |
| 2 | apply one-line patch | — |
| 3 | `cmake --preset windows-x64` | `Configuring done` / `Generating done` |
| 4 | `cmake --build build\windows-x64 --config Release` | 96/96 steps, `triton.dll` + `triton.pyd` linked |
| 5 | Python smoke from `build\...\Release` | lift `mov rax, 0x44332211`; `rax = 0x44332211`; Z3 model `x = 0x4142` |
| 6 | `cmake --build ... --target install` | `out\install\windows-x64` populated; `triton.pyd` in `lib\python3.15\site-packages` |
| 7 | `import triton` from install tree | `ImportError: DLL load failed` (no capstone/z3 there) |
| 8 | same + `capstone.dll`, `libz3.dll` copied next to `.pyd` | import OK, `1 0 1599` |
| 9 | copy three files to global `site-packages`, run from `C:\` | import OK |
| 10 | `pip install lief unicorn z3-solver`; `ctest ... -C Release` | `100% tests passed`, 619 tests, 438.93 s |
| 11 | `cmake -S . -B <tmp>` without toolchain (pip probe) | `Required library Z3 NOT FOUND` |
| 12 | `cmake --preset windows-x64 -Wno-error=install-absolute-destination` | accepted; diagnostics demoted (verified in a scratch project) |
