# Build System in Bitcoin v0.2.0

The build system for Bitcoin v0.2.0 was designed to support compilation across different platforms, primarily Windows (using MinGW GCC and Microsoft Visual C++) and Unix-like systems (using GCC). It involved managing several external dependencies and using platform-specific makefiles.

*Relevant files:*
*   `build-msw.txt`: Instructions for building on Windows.
*   `build-unix.txt`: Instructions for building on Unix-like systems.
*   `makefile`: Used for MinGW GCC builds on Windows.
*   `makefile.unix`: Used for GCC builds on Unix-like systems.
*   `makefile.vc`: Used for Microsoft Visual C++ builds on Windows.
*   `setup.nsi`: NSIS (Nullsoft Scriptable Install System) script for creating the Windows installer.

## Supported Platforms and Compilers

Bitcoin v0.2.0 officially supported the following build environments:

*   **Windows**:
    *   MinGW GCC version 3.4.5 (using `makefile`)
    *   Microsoft Visual C++ 6.0 SP6 (using `makefile.vc`)
*   **Unix/Linux**:
    *   GCC version 4.3.3 (using `makefile.unix`)

*References: [build-msw.txt Lines 14-17](https://github.com/jonzou/btc020/blob/ebbdade7/build-msw.txt#L14-L17), [build-unix.txt Line 32](https://github.com/jonzou/btc020/blob/ebbdade7/build-unix.txt#L32)*

## External Dependencies

The project relied on several external libraries:

1.  **wxWidgets**: For the graphical user interface.
    *   Version: 2.8.9
    *   License: LGPL 2.1 with exceptions.
    *   Build: Typically required manual compilation from source, especially on Unix to ensure correct static linking.
        *Unix build: [build-unix.txt Lines 49-58](https://github.com/jonzou/btc020/blob/ebbdade7/build-unix.txt#L49-L58)*
2.  **OpenSSL**: For cryptographic functions (ECDSA, SHA256, RIPEMD160).
    *   Version: 0.9.8k
    *   License: BSD with an advertising clause.
    *   Customization: `build-msw.txt` details a "no-everything" build of OpenSSL to exclude unused encryption routines (RC2, RC4, DES, AES, etc.) as Bitcoin only uses its hashing and ECDSA features.
        *Windows OpenSSL build: [build-msw.txt Lines 55-83](https://github.com/jonzou/btc020/blob/ebbdade7/build-msw.txt#L55-L83)*
3.  **Berkeley DB**: For data storage (wallet, blockchain index, addresses).
    *   Version: 4.7.25.NC
    *   License: BSD with a requirement that linked software must be open source.
    *   Build: Required specific configuration for C++ bindings.
        *Windows BDB build: [build-msw.txt Lines 86-91](https://github.com/jonzou/btc020/blob/ebbdade7/build-msw.txt#L86-L91)*
4.  **Boost**: For various C++ utilities.
    *   Version: 1.34.1 (Windows), 1.40.0 (Unix)
    *   License: MIT-like.

*Dependency paths (default): `\wxwidgets`, `\openssl`, `\db`, `\boost`.*
*References: [build-msw.txt Lines 20-42](https://github.com/jonzou/btc020/blob/ebbdade7/build-msw.txt#L20-L42), [build-unix.txt Lines 26-36](https://github.com/jonzou/btc020/blob/ebbdade7/build-unix.txt#L26-L36)*

## Build Configuration System

All makefiles support `debug` and `release` build configurations via the `BUILD` variable.
*   `BUILD=debug` (default):
    *   Enables debug symbols (`-g` for GCC, `/Zi` for MSVC).
    *   Defines `__WXDEBUG__` for wxWidgets debug mode.
    *   Typically uses lower optimization levels (`-O0` for GCC).
*   `BUILD=release`:
    *   Optimized build without debug symbols.
*   The variable `D` is used to suffix library names (e.g., `wxmsw28d_core.lib` for debug, `wxmsw28_core.lib` for release).

*References:*
*   MinGW: [makefile Lines 6-14](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L6-L14)
*   Unix: [makefile.unix Lines 6-14](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.unix#L6-L14)
*   MSVC: [makefile.vc Lines 6-12](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L6-L12)

## Makefile Architecture

Three main makefiles cater to the different build environments: `makefile` (MinGW), `makefile.unix`, and `makefile.vc`.

### Shared Elements
*   **`HEADERS` variable**: Defines a common list of core header files used across all makefiles.
    ```makefile
    HEADERS=headers.h util.h main.h serialize.h uint256.h key.h bignum.h script.h db.h base58.h
    ```
    *References: [makefile Line 27](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L27), [makefile.unix Line 38](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.unix#L38), [makefile.vc Line 25](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L25)*
*   **Object files**: Object files are typically placed in an `obj/` subdirectory.
*   **Compilation flow**: Generally involves compiling individual `.cpp` files into object files and then linking them into the final executable (`bitcoin.exe` or `bitcoin`).

### Compilation Process Highlights
*   **Precompiled Headers**: GCC makefiles (`makefile`, `makefile.unix`) support generating `headers.h.gch` to speed up compilation.
    *e.g., [makefile Lines 34-35](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L34-L35)*
    MSVC uses `/YX` and `/Fpobj/headers.pch` for its precompiled header mechanism.
    *e.g., [makefile.vc Line 26](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L26)*
*   **`sha.cpp` Optimization**: Compiled with higher optimization (`-O3` for GCC, `/O2` for MSVC) due to its performance sensitivity.
    *References: [makefile Line 62](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L62), [makefile.unix Line 73](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.unix#L73), [makefile.vc Line 60](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L60)*
*   **Windows Resources (`ui.rc`)**: Compiled using `windres` (MinGW) or `rc` (MSVC) for Windows builds to include icons and other resources.
    *MinGW: [makefile Lines 67-68](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L67-L68)*
    *MSVC: [makefile.vc Lines 62-63](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L62-L63)*
*   **Executable Naming**: `bitcoin.exe` on Windows, `bitcoin` on Unix.

### Library Linking
Linking strategies vary by platform:

*   **MinGW (`makefile`)**:
    *   `LIBPATHS`: Specifies paths to `/db/build_unix`, `/openssl/out`, `/wxwidgets/lib/gcc_lib`.
    *   `LIBS`: Links dynamically against `db_cxx`, `eay32` (OpenSSL), various wxWidgets libraries (e.g., `wxmsw28d_core`), and Windows system libraries (kernel32, user32, ws2_32, etc.).
    *Reference: [makefile Lines 18-26](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L18-L26)*

*   **Unix (`makefile.unix`)**:
    *   `LIBPATHS`: Specifies paths like `/usr/lib`, `/usr/local/lib`.
    *   `LIBS`: Uses `-Wl,-Bstatic` to prefer static linking for Boost (`boost_system`, `boost_filesystem`), Berkeley DB (`db_cxx`), and wxWidgets (`wx_gtk2d-2.8`). Uses `-Wl,-Bdynamic` for OpenSSL (`crypto`) and system libraries like GTK.
    *Reference: [makefile.unix Lines 27-34](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.unix#L27-L34)*

*   **Visual C++ (`makefile.vc`)**:
    *   `LIBPATHS`: Specifies paths to `/db/build_windows/$(BUILD)`, `/openssl/out`, `/wxwidgets/lib/vc_lib`.
    *   `LIBS`: Links against `libdb47s$(D).lib`, `libeay32.lib`, various wxWidgets `.lib` files, and Windows system libraries.
    *Reference: [makefile.vc Lines 18-24](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L18-L24)*

### Build Targets
*   **`all`**: Default target, builds the main executable.
*   **`clean`**: Removes intermediate object files (`obj/*`), precompiled headers, and MSVC-specific debug files (`*.ilk`, `*.pdb`).
    *MinGW clean: [makefile Lines 79-81](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L79-L81)*
    *Unix clean: [makefile.unix Lines 87-89](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.unix#L87-L89)*
    *MSVC clean: [makefile.vc Lines 74-77](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L74-L77)*
*   Windows makefiles include a step to `kill` the running `bitcoin.exe` before linking, to handle file locking issues.
    *MinGW: [makefile Line 76](https://github.com/jonzou/btc020/blob/ebbdade7/makefile#L76)*
    *MSVC: [makefile.vc Line 71](https://github.com/jonzou/btc020/blob/ebbdade7/makefile.vc#L71)*

## Windows Installer (`setup.nsi`)
*Relevant file: [setup.nsi](https://github.com/jonzou/btc020/blob/ebbdade7/setup.nsi)*

An NSIS script is provided to create a Windows installer (`bitcoin-0.2.0-win32-setup.exe`).
*   **General Info**: Defines product name, version, company, URL.
    *Lines 4-8: `Name Bitcoin`, `!define VERSION 0.2.0`, etc.*
*   **MUI (Modern User Interface)**: Uses MUI2 for the installer appearance.
    *Icons: `src\rc\bitcoin.ico`, `uninstall.ico`.*
    *Start Menu: Creates a "Bitcoin" group with shortcuts to the application and uninstaller.*
    *Finish Page: Option to run Bitcoin after installation.*
*   **Files Installed**:
    *   `bitcoin.exe`
    *   `libeay32.dll` (OpenSSL)
    *   `mingwm10.dll` (MinGW runtime, if built with MinGW)
    *   `license.txt`
    *   `readme.txt`
    *   The entire `src\` directory and its contents.
    *Section "-Main": [setup.nsi Lines 39-48](https://github.com/jonzou/btc020/blob/ebbdade7/setup.nsi#L39-L48)*
*   **Registry Entries**:
    *   Creates an uninstaller entry in "Add/Remove Programs".
    *   Stores installation path.
    *Section "-post": [setup.nsi Lines 50-67](https://github.com/jonzou/btc020/blob/ebbdade7/setup.nsi#L50-L67)*
*   **Uninstaller**: Removes installed files, Start Menu shortcuts, and registry entries.

## UI Development Notes
The `build-unix.txt` file mentions that the UI layout was designed using wxFormBuilder, with the project file `uiproject.fbp`. This tool generated `uibase.cpp` and `uibase.h`, which define the base classes for UI elements. These generated files were not meant to be edited manually.
*Reference: [build-unix.txt Lines 41-43](https://github.com/jonzou/btc020/blob/ebbdade7/build-unix.txt#L41-L43)*

This multi-platform build system, while common for its time, highlights the complexities of managing different compilers, libraries, and operating system conventions. The inclusion of detailed build text files was essential for developers to replicate the build environment.
