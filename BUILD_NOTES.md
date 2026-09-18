# UniCAD — Personal Build Notes

This file contains observations and fixes discovered while building UniCAD from source on Arch Linux.

These notes are based on an actual successful build and may be useful if the project fails to build or launch after a fresh setup.

---

## 1. Use the system Python

The repository may contain a `.python-version` that points to a pyenv Python version.

For the Arch Linux setup tested here, the build works with the **system Python**:

```bash
/usr/bin/python
```

Check:

```bash
/usr/bin/python --version
```

If using pyenv, check:

```bash
which python
python --version
```

If this points to a pyenv installation, do not use it for the UniCAD CMake configuration.

Configure CMake explicitly:

```bash
cmake -S . -B build \
  -DPython3_EXECUTABLE=/usr/bin/python \
  -DPython_EXECUTABLE=/usr/bin/python \
  -DPYTHON_EXECUTABLE=/usr/bin/python
```

### Why?

Arch's `PySide6` and `Shiboken6` packages are built for the system Python.

Using a different Python version can result in errors where Shiboken6 is built against one Python version while CMake detects another.

---

## 2. Do not install Pivy with pip

On this Arch setup, Pivy is provided by the system package:

```bash
python-pivy
```

Install it with:

```bash
sudo pacman -S python-pivy
```

Verify:

```bash
/usr/bin/python -c "import pivy; print(pivy.__version__)"
```

Installing Pivy into the pyenv Python with `pip` is not necessary and may fail because of Coin3D/SWIG compatibility.

---

## 3. Coin3D version detection

The Arch system used:

```text
Coin3D 4.0.10
Pivy 0.6.11
```

UniCAD's CMake detection originally had a problem with multi-digit Coin3D micro versions.

The regular expression in:

```text
cmake/SetupCoin3D.cmake
```

was matching only one digit of the version.

For example, `4.0.10` could incorrectly be interpreted as `4.0.1`.

The version matching expressions were changed from:

```text
([0-9?])
```

to:

```text
([0-9]+)
```

This allows versions such as:

```text
4.0.10
```

to be detected correctly.

---

## 4. VTK and fast_float

CMake initially failed because VTK referenced:

```text
FastFloat::fast_float
```

without the corresponding package being available.

Installing Arch's `fast_float` package fixed this:

```bash
sudo pacman -S fast_float
```

---

## 5. PCL package

The Arch package manager did not provide a package named:

```text
pcl
```

on the tested system.

Do not add `pcl` blindly to the dependency list.

The tested build succeeded without installing it.

---

## 6. TBB package name

Use:

```text
onetbb
```

rather than the old:

```text
tbb
```

package name.

Install:

```bash
sudo pacman -S onetbb
```

---

## 7. Build parallelism

The project can use a lot of RAM during compilation.

A 16-core system successfully built UniCAD with:

```bash
cmake --build build --parallel 16
```

If compilation runs out of memory, reduce the number of jobs:

```bash
cmake --build build --parallel 8
```

or:

```bash
cmake --build build --parallel 4
```

There is normally no need to delete the `build` directory after an out-of-memory failure.

---

## 8. SALOME/SMESH warnings

The build produces many warnings from:

```text
src/3rdParty/salomesmesh
```

including warnings related to:

* overloaded virtual functions
* deprecated OpenCASCADE APIs
* `Poly_Triangulation::Triangles()`
* enum arithmetic

These warnings are not necessarily build failures.

For example, the build successfully reached:

```text
Built target StdMeshers
```

despite the warnings.

If the build reports:

```text
make: *** [Makefile:146: all] Błąd 2
```

the important information is the **first actual compiler/linker error above it**.

---

## 9. If the build fails in parallel

If the output is difficult to follow because many compilation jobs are running simultaneously, rerun with one job:

```bash
cmake --build build --parallel 1
```

This makes the first actual error much easier to identify.

---

## 10. Boost intrusive_ptr / Coin3D issue

The Arch Boost version caused a compilation problem involving Coin3D's `SoBase` and Boost `intrusive_ptr`.

The compiler reported errors such as:

```text
error: ‘intrusive_ptr_release’ was not declared in this scope
error: ‘intrusive_ptr_add_ref’ was not declared in this scope
```

The project already contained `CoinPtr` usage, but the required `SoBase` overload declarations were missing.

The following declarations were added to:

```text
src/Gui/ViewProvider.h
```

```cpp
class SoBase;

void intrusive_ptr_add_ref(SoBase* p);
void intrusive_ptr_release(SoBase* p);
```

The implementations were added to:

```text
src/Gui/ViewProvider.cpp
```

before the `Gui` namespace:

```cpp
void intrusive_ptr_add_ref(SoBase* p)
{
    p->ref();
}

void intrusive_ptr_release(SoBase* p)
{
    p->unref();
}
```

This resolved the linker errors and allowed the complete project to build.

---

## 11. PartDesign startup crash

After successfully compiling, UniCAD initially crashed during startup with:

```text
Base::BaseClass::initSubclass(...)
assertion `!parentType.isBad()` failed
```

The crash occurred while initializing classes such as:

```text
PartDesign::UnifiedSweep
PartDesign::UnifiedLoft
```

The cause was initialization order.

A derived class must not be initialized before its parent class.

### Core PartDesign initialization

The relevant order in:

```text
src/Mod/PartDesign/App/AppPartDesign.cpp
```

must be:

```cpp
PartDesign::UnifiedRevolve::init();
PartDesign::Pipe::init();
PartDesign::UnifiedSweep::init();
PartDesign::Loft::init();
PartDesign::UnifiedLoft::init();
```

In particular:

```text
Pipe → UnifiedSweep
Loft → UnifiedLoft
```

Each parent must only be initialized once.

There must not be a second:

```cpp
PartDesign::Loft::init();
```

later in the same initialization sequence.

---

## 12. PartDesign GUI initialization

The same parent-before-child principle applies to the GUI classes.

The relevant order in:

```text
src/Mod/PartDesign/Gui/AppPartDesignGui.cpp
```

is:

```cpp
PartDesignGui::ViewProviderUnifiedRevolve::init();
PartDesignGui::ViewProviderPipe::init();
PartDesignGui::ViewProviderUnifiedSweep::init();
PartDesignGui::ViewProviderLoft::init();
PartDesignGui::ViewProviderUnifiedLoft::init();
```

Again:

```text
ViewProviderPipe → ViewProviderUnifiedSweep
ViewProviderLoft  → ViewProviderUnifiedLoft
```

Do not initialize the derived class before its parent.

---

## 13. Debugging startup crashes

If UniCAD builds successfully but crashes immediately when launched, use GDB to identify the exact class being initialized:

```bash
gdb -q -batch \
  -ex run \
  -ex 'bt 20' \
  ./build/bin/FreeCAD
```

For a shorter output:

```bash
gdb -q -batch \
  -ex run \
  -ex 'bt 20' \
  ./build/bin/FreeCAD 2>&1 | grep -E 'assert|::init\(\)|PyInit'
```

The important information is the first failing:

```text
ClassName::init()
```

Do not randomly reorder initialization functions. Check the class inheritance and `PROPERTY_SOURCE` relationship first.

---

## 14. Build version vs installed version

During development, UniCAD can be launched directly from the build directory:

```bash
./build/bin/FreeCAD
```

After installation:

```bash
sudo cmake --install build
```

the installed application can be launched with:

```bash
FreeCAD
```

When debugging source changes, make sure you know which copy you are running.

Check the installed executable:

```bash
which FreeCAD
```

---

## 15. Rebuilding after changes

Normally:

```bash
cmake --build build --parallel 16
```

Then reinstall:

```bash
sudo cmake --install build
```

Then:

```bash
FreeCAD
```

A full CMake reconfiguration is not normally necessary after ordinary source changes.

---

## 16. Current known-good workflow

The workflow that successfully produced a working installed UniCAD was:

```bash
sudo pacman -S --needed \
  base-devel git cmake ninja swig gcc clang ccache \
  coin python-pivy opencascade eigen boost fmt freetype2 \
  vtk hdf5 xerces-c yaml-cpp pugixml onetbb libspnav \
  graphviz qt6-base qt6-declarative qt6-tools qt6-svg \
  pyside6 shiboken6 pybind11 fast_float \
  python python-numpy python-matplotlib python-ply \
  python-yaml python-lxml
```

Configure:

```bash
cmake -S . -B build \
  -DPython3_EXECUTABLE=/usr/bin/python \
  -DPython_EXECUTABLE=/usr/bin/python \
  -DPYTHON_EXECUTABLE=/usr/bin/python
```

Build:

```bash
cmake --build build --parallel 16
```

Test:

```bash
./build/bin/FreeCAD
```

Install:

```bash
sudo cmake --install build
```

Run installed version:

```bash
FreeCAD
```

---

## 17. Important distinction

The main build guide should describe the **normal supported build procedure**.

This file documents problems encountered during one real Arch Linux build and the fixes that were required.

Some of these fixes may become unnecessary in future UniCAD versions, Arch package updates, or upstream changes.
