# ScanTailor Advanced – fork with extended image import

This is a fork of [ScanTailor Advanced](https://github.com/ScanTailor-Advanced/scantailor-advanced),
an interactive post-processing tool for scanned pages.

**For the features and how to use the program, please see the
[documentation of the original project](https://github.com/ScanTailor-Advanced/scantailor-advanced#readme).**
Everything described there applies to this fork as well.

The changes in this fork were developed with the help of Claude (Anthropic).

## Changes in this fork

### Image import
* **TIFF reading reworked**
  * Tiled TIFF files can be opened (previously, bi-level, grayscale and palette images failed).
  * Floating point, signed integer and 32-bit images are supported.
  * Colors of palette images in big-endian ("Motorola") TIFF files are correct now.
  * Fixed undefined behaviour when reading 2 and 4 bit images.
  * TIFF files using a compression the program can't decode are rejected right away when importing,
    naming the compression.
* **JPEG 2000 import** (`.jp2 .j2k .j2c .jpc .jpf .jpx .jph .jhc`) through OpenJPEG:
  fast import (only the file header is read), thumbnails decoded at reduced resolution,
  multi-threaded decoding, huge images decoded strip by strip to limit memory use.
* **Faster thumbnails for JPEG** files, decoded directly at reduced size.

### Error reporting
* When images can't be loaded or output files can't be written, the reason is shown –
  collected in a single, non-modal message instead of one message per file.
* Output TIFF files are checked to have been written completely (e.g. on a full disk);
  a failed write no longer leaves a damaged file behind.

### Robustness
* Fixed a use-after-free when finishing a lasso zone, thread-safety issues with the
  application settings and the default parameter profiles, and possible crashes
  with missing resolution (DPI) information.
* Auto-save is also triggered by changes to the page list and to the current page.
* Code cleanups based on the compiler's static code analysis.

### Build and tests
* CMake verifies which compression schemes libtiff supports (LZMA etc. are required).
* Optional static code analysis: `-DENABLE_CODE_ANALYSIS=ON` (MSVC: `/W4 /analyze`).
* New tests for the image readers and writer; CI now fails on failing tests and also builds on Windows.

## Building

### Windows (Visual Studio + vcpkg)

Install the dependencies (the quotes are needed in PowerShell):

```
vcpkg install qtbase qtsvg qttools libjpeg-turbo libpng "tiff[core,jpeg,zip,lzma,zstd,webp,lerc,libdeflate,tools]" openjpeg zlib boost-test boost-foreach boost-intrusive boost-multi-index boost-lambda
```

Then, in a "Native Tools Command Prompt for VS x64", from a `build` directory inside the source directory:

```
cmake -G "NMake Makefiles JOM" -DCMAKE_BUILD_TYPE=Release -DCMAKE_TOOLCHAIN_FILE="%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake" ..
jom
ctest -C Release --output-on-failure
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

* `-DTIFF_REQUIRED_CODECS=...` / `-DTIFF_RECOMMENDED_CODECS=...` – the TIFF compression schemes libtiff
  has to / should support (defaults: `LZW;PACKBITS;CCITT;JPEG;OJPEG;DEFLATE;LZMA` / `ZSTD;WEBP;LERC`).
  `-DSKIP_TIFF_CODEC_CHECK=ON` skips the check.
* `-DENABLE_CODE_ANALYSIS=ON` – extra warnings and static code analysis. Slow; best used in a separate build directory.
* `-DBUILD_TESTS=OFF` – don't build the unit tests.

## License

GNU GPLv3, see [LICENSE](LICENSE). This is a modified version of ScanTailor Advanced;
the modifications are documented above and in the commit history.
