# ScanTailor Advanced – fork with reworked oblique correction and extended image import

This is a fork of [ScanTailor Advanced](https://github.com/ScanTailor-Advanced/scantailor-advanced),
an interactive post-processing tool for scanned pages.

**For the features and how to use the program, please see the
[documentation of the original project](https://github.com/ScanTailor-Advanced/scantailor-advanced#readme).**
Everything described there applies to this fork as well.

The changes in this fork were developed with the help of Claude (Anthropic).

## Changes in this fork

### Deskew: oblique correction reworked

* The oblique (shear) correction is switched the same way as the deskew rotation now: two wide
  **Auto / Manual buttons** instead of a check box, both in the page options panel and in the
  default parameters dialog, in a consistent layout (heading, buttons, angle).
* **Fixed: the deskew "Auto" button lost its highlight.** All four mode buttons (deskew and
  oblique) were auto-exclusive inside one group box, and Qt groups such buttons per parent
  widget, so choosing an oblique mode silently unchecked the deskew mode. Each pair now has
  its own button group.
* Switching oblique to *Manual* resets an automatically found shear angle to 0, so the shear
  isn't left applied with automatic correction turned off.
* "Apply oblique automatically" acts as a master switch: stored per-page parameters can no
  longer re-enable oblique correction while it is off.
* The oblique finder reports *no* angle when the image has too little structure to tell an
  angle from noise, instead of shearing the page by an arbitrary amount.
* The "apply to other pages" dialog no longer accepts a selection that would apply neither
  the deskew nor the oblique angle.
* The project and profile XML format is unchanged, so existing projects and profiles keep working.

### Image import

* **TIFF reading reworked**
  * Tiled TIFF files can be opened (previously, bi-level, grayscale and palette images failed).
  * Floating point, signed integer and 32-bit images are supported.
  * Colors of palette images in big-endian ("Motorola") TIFF files are correct now.
  * Fixed undefined behaviour when reading 2 and 4 bit images.
  * Read errors no longer yield images with uninitialised memory in the unread rows.
  * TIFF files using a compression the program can't decode are rejected right away when
    importing, naming the compression.
* **JPEG 2000 import** (`.jp2 .j2k .j2c .jpc .jpf .jpx .jph .jhc`) through OpenJPEG:
  fast import (only the file header is read), thumbnails decoded at reduced resolution,
  multi-threaded decoding, huge images decoded strip by strip to limit memory use.
* **Faster thumbnails for JPEG** files, decoded directly at reduced size.
* Fixed a leak and undefined behaviour in the JPEG metadata reader (libjpeg's error handling
  jumped past the cleanup of C++ objects) and a buffer bug in the PNG reader on partial reads.

### Error reporting

* When images can't be loaded or output files can't be written, the reason is shown –
  collected in a single, non-modal message instead of one message per file.
* Output TIFF files are checked to have been written completely (e.g. on a full disk);
  a failed write no longer leaves a damaged file behind.
* Malformed values in project files (e.g. a missing binarisation threshold) no longer turn
  into settings that black out a page.

### Correctness and robustness

* Fixed several defects that produced wrong results or crashes: a grayscale measurement that
  read the wrong image, a division by zero in Wolf binarisation on blank pages, integer
  overflows in pixel arithmetic (TIFF buffers, binarisation, distance transform) on large
  images, missing guards in `BinaryImage`, a wrong assertion and a lost search direction in
  the arc length mapper, an unsigned wraparound in the page split gap scan, and divisions by
  zero in the content finder and with missing resolution (DPI) information.
* Fixed a use-after-free when finishing a lasso zone, and thread-safety issues with the
  application settings, the default parameter profiles and the deviation statistics used for
  sorting thumbnails.
* Natural file name sorting now also compares the separators, so names differing only in
  those (`img-2.tif` vs. `img_1.tif`) sort by the whole name rather than falling back to a
  plain lexicographic compare. **This can change page order in projects with inconsistent
  file naming.**
* The thumbnail list keeps its position when re-sorting moves the current page elsewhere.
* Auto-save is also triggered by changes to the page list and to the current page.
* Code cleanups based on the compiler's static code analysis.

### Build and tests

* CMake verifies which compression schemes libtiff supports (LZMA etc. are required).
* Optional static code analysis: `-DENABLE_CODE_ANALYSIS=ON` (MSVC: `/W4 /analyze`).
* New tests for the image readers and the TIFF writer; CI now fails on failing tests and also
  builds on Windows.
* The `update_translations` target no longer refers to `Qt6::lupdate` by name, which broke the
  Qt 5 fallback build. A review of the Qt 6 port found no other problem: the code already
  guards every API removed in Qt 6.

## Building

### Windows, step by step

These steps assume a machine with nothing installed yet. Neither JOM nor vcpkg comes with an
installer: they are simply unpacked into a folder of your choice. The examples below use
`C:\Dev` as that folder – any path works, as long as you use it consistently and it contains
no spaces or non-English characters.

**1. Get the source code** into `C:\Dev\scantailor-advanced`, either with
[Git for Windows](https://git-scm.com/download/win):

```
cd /d C:\Dev
git clone https://github.com/2ndmax/scantailor-advanced-with-claude.git scantailor-advanced
```

or by downloading the ZIP of this repository ("Code" → "Download ZIP") and unpacking it there.

**2. Install Visual Studio Community** (the free edition):
<https://visualstudio.microsoft.com/vs/community/>
In the installer pick the workload **"Desktop development with C++"**. Of its optional
components, only these are needed:

* MSVC build tools (x64)
* C++ CMake tools for Windows
* Windows 11 SDK

**3. Download JOM** (a parallel `nmake` replacement): <https://wiki.qt.io/Jom>
Unpack it to `C:\Dev\jom`, so that `C:\Dev\jom\jom.exe` exists.

**4. Download vcpkg** as a ZIP: <https://github.com/microsoft/vcpkg>
Unpack it to `C:\Dev\vcpkg`.

**5. Build the libraries with vcpkg.** Open the **"Native Tools Command Prompt for VS x64"**
(from the start menu, inside the Visual Studio folder) and run:

```
cd /d C:\Dev\vcpkg
bootstrap-vcpkg.bat
vcpkg install --recurse qtbase qtsvg qttools libjpeg-turbo libpng "tiff[core,jpeg,zip,lzma,zstd,webp,lerc,libdeflate,tools]" openjpeg zlib boost-test boost-foreach boost-intrusive boost-multi-index boost-lambda
```

This compiles Qt and everything else from source and takes a while – plan for an hour or more
and several gigabytes of disk space.

**6. Set the environment variables.** Open **PowerShell as administrator** and run (adjust the
two paths if you used a different folder):

```powershell
$vcpkgRoot = "C:\Dev\vcpkg"
$jomRoot   = "C:\Dev\jom"
[System.Environment]::SetEnvironmentVariable("VCPKG_ROOT", $vcpkgRoot, "Machine")
[System.Environment]::SetEnvironmentVariable("JOM_ROOT",   $jomRoot,   "Machine")
$currentPath = [System.Environment]::GetEnvironmentVariable("PATH", "Machine")
[System.Environment]::SetEnvironmentVariable("PATH", "$currentPath;$vcpkgRoot;$jomRoot", "Machine")
```

Then **restart the computer**, so every program sees the new variables.

**7. Configure and build.** In the "Native Tools Command Prompt for VS x64":

```
cd /d C:\Dev\vcpkg
vcpkg integrate install

cd /d C:\Dev\scantailor-advanced
mkdir build
cd build
cmake -G "NMake Makefiles JOM" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE="%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake" ..
jom -j 10
```

`-j 10` is the number of parallel compiler processes; use roughly the number of processor cores.
The finished `scantailor-advanced.exe` and all needed DLLs end up in the `build` directory.

**8. Run the tests** (optional), in the same `build` directory:

```
ctest -C Release --output-on-failure
```

When configuring again later, e.g. after changing build options, add `--fresh` to the `cmake`
call to discard the cached configuration:

```
cmake --fresh -G "NMake Makefiles JOM" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE="%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake" ..
```

### Linux (Debian / Ubuntu)

```
sudo apt install build-essential cmake qtbase5-dev libqt5svg5-dev qttools5-dev \
  libboost-test-dev libboost-dev libjpeg-dev libpng-dev libtiff-dev zlib1g-dev libopenjp2-7-dev
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

### Build options

* `-DTIFF_REQUIRED_CODECS=...` / `-DTIFF_RECOMMENDED_CODECS=...` – the TIFF compression schemes
  libtiff has to / should support (defaults: `LZW;PACKBITS;CCITT;JPEG;OJPEG;DEFLATE;LZMA` /
  `ZSTD;WEBP;LERC`). `-DSKIP_TIFF_CODEC_CHECK=ON` skips the check.
* `-DENABLE_CODE_ANALYSIS=ON` – extra warnings and static code analysis. Slow; best used in a
  separate build directory.
* `-DBUILD_TESTS=OFF` – don't build the unit tests.

## License

GNU GPLv3, see [LICENSE](LICENSE). This is a modified version of ScanTailor Advanced;
the modifications are documented above and in the commit history.
