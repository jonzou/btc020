# Overview of Bitcoin v0.2.0

This document provides a comprehensive overview of Bitcoin v0.2.0, an early implementation of the Bitcoin cryptocurrency system developed by Satoshi Nakamoto.

## Purpose and Scope

Bitcoin v0.2.0 represents a peer-to-peer electronic cash system that enables direct online payments between parties without requiring a trusted third party.

This overview covers the high-level architecture, major subsystems, and their relationships within the Bitcoin v0.2.0 codebase. For detailed information about specific subsystems, see the following specialized pages:

*   Core Blockchain Systems
*   Cryptography and Encoding
*   Networking
*   Database and Persistence
*   Market System
*   User Interface
*   Build System

## System Architecture

Bitcoin v0.2.0 implements a layered architecture with distinct separation of concerns across six major subsystems.

### High-Level System Components

The `headers.h` file provides a glimpse into the high-level components:

```cpp
// Copyright (c) 2009 Satoshi Nakamoto
// Distributed under the MIT/X11 software license, see the accompanying
// file license.txt or http://www.opensource.org/licenses/mit-license.php.

#ifdef _MSC_VER
#pragma warning(disable:4786)
#pragma warning(disable:4804)
#pragma warning(disable:4805)
#pragma warning(disable:4717)
#endif
#ifdef _WIN32_WINNT
#undef _WIN32_WINNT
#endif
#define _WIN32_WINNT 0x0400
#ifdef _WIN32_IE
#undef _WIN32_IE
#endif
#define _WIN32_IE 0x0400
#define WIN32_LEAN_AND_MEAN 1
#define __STDC_LIMIT_MACROS // to enable UINT64_MAX from stdint.h
#include <wx/wx.h>
#include <wx/clipbrd.h>
#include <wx/snglinst.h>
#include <wx/taskbar.h>
#include <wx/stdpaths.h>
#include <wx/utils.h>
#include <openssl/ecdsa.h>
#include <openssl/evp.h>
#include <openssl/rand.h>
#include <openssl/sha.h>
#include <openssl/ripemd.h>
// ... (other includes)
#include "strlcpy.h"
#include "serialize.h"
#include "uint256.h"
#include "util.h"
#include "key.h"
#include "bignum.h"
#include "base58.h"
#include "script.h"
#include "db.h"
#include "net.h"
#include "irc.h"
#include "main.h"
#include "market.h"
#include "uibase.h"
#include "ui.h"

#include "xpm/addressbook16.xpm"
#include "xpm/addressbook20.xpm"
#include "xpm/bitcoin16.xpm"
#include "xpm/bitcoin20.xpm"
#include "xpm/bitcoin32.xpm"
#include "xpm/bitcoin48.xpm"
#include "xpm/check.xpm"
#include "xpm/send16.xpm"
#include "xpm/send16noshadow.xpm"
#include "xpm/send20.xpm"
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L87-L101)*

This section of `headers.h` (specifically lines 87-101 in the original file, though the snippet above is more extensive for context) includes the main header files for the various components of the system. This demonstrates the modular design, with clear separation for utilities (`util.h`), cryptography (`key.h`, `bignum.h`), data handling (`serialize.h`, `uint256.h`, `db.h`), networking (`net.h`, `irc.h`), core logic (`main.h`), the market system (`market.h`), and user interface (`ui.h`, `uibase.h`).

### Core Data Flow

The core data flow revolves around the interaction of these components, orchestrated by `main.cpp` (which includes `main.h`).

```cpp
#include "strlcpy.h"
#include "serialize.h"
#include "uint256.h"
#include "util.h"
#include "key.h"
#include "bignum.h"
#include "base58.h"
#include "script.h"
#include "db.h"
#include "net.h"
#include "irc.h"
#include "main.h"
#include "market.h"
#include "uibase.h"
#include "ui.h"
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L95-L101)*

Lines 95-101 of `headers.h` (shown above) specifically highlight the inclusion of `net.h` (networking), `irc.h` (IRC for node discovery), `main.h` (core application logic), `market.h` (market system), and `ui.h` (user interface). This indicates how data would flow from network events or UI interactions into the core logic, potentially interacting with the database and market systems.

## Major Subsystems

### Core Blockchain Engine

The blockchain engine, primarily managed by `main.cpp`, is responsible for:
*   **Block Processing**: Validating blocks and managing the blockchain. (main.cpp)
*   **Transaction Validation**: Verifying transactions. (main.cpp)
*   **Script Engine**: Executing smart contracts. (`script.h`)
*   **Core Data Structures**: Defining `CBlock`, `CTransaction`, `CBlockIndex`. (`main.h`)

### Cryptographic Foundation

This subsystem underpins Bitcoin's security, implementing ECDSA signatures, address generation, and hashing.

```cpp
#include <openssl/ecdsa.h>
#include <openssl/evp.h>
#include <openssl/rand.h>
#include <openssl/sha.h>
#include <openssl/ripemd.h>
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L27-L31)*

These lines from `headers.h` show the inclusion of OpenSSL libraries for essential cryptographic operations like ECDSA, SHA256, and RIPEMD-160.

```cpp
#include "key.h"
#include "bignum.h"
#include "base58.h"
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L91-L93)*

Further, `key.h` would contain implementations for key management (public, private keys), `bignum.h` for large number arithmetic necessary in cryptography, and `base58.h` for the encoding scheme used in Bitcoin addresses.

### Networking and Discovery

Handles P2P communication and node discovery via IRC.
*   **P2P Protocol**: Node communication (`net.cpp`, `net.h`).
*   **IRC Discovery**: Peer discovery (`irc.cpp`, `irc.h`).
*   **Message Handling**: Protocol messages (`net.cpp`).
*   **Address Management**: Peer addresses (integrates with the database).

### Database Persistence

Bitcoin v0.2.0 uses Berkeley DB for storing:
*   `wallet.dat`: Private keys, transaction history.
*   `blkindex.dat`: Blockchain index, block metadata.
*   `addr.dat`: Known peer addresses.
*   `reviews.dat`: Market review data.

The use of Berkeley DB is implied by `#include "db.h"` in `headers.h`, which would contain the interface to the database operations.

### Market System

A unique feature allowing users to advertise products and exchange reviews. This was later removed from Bitcoin. (`market.h`, `market.cpp`)

### User Interface

A wxWidgets-based GUI for wallet management, transactions, and market interaction. (`ui.h`, `uibase.h`, and various `.cpp` files implementing UI elements).

## Dependencies and Build Environment

Key external libraries include:
*   **wxWidgets**: GUI framework.
*   **OpenSSL**: Cryptographic functions.
*   **Berkeley DB**: Database engine.
*   **Boost**: C++ utilities.

The inclusion of these dependencies is evident in `headers.h`:
```cpp
// ... (wxWidgets includes)
#include <wx/wx.h>
#include <wx/clipbrd.h>
#include <wx/snglinst.h>
#include <wx/taskbar.h>
#include <wx/stdpaths.h>
#include <wx/utils.h>

// ... (OpenSSL includes)
#include <openssl/ecdsa.h>
#include <openssl/evp.h>
#include <openssl/rand.h>
#include <openssl/sha.h>
#include <openssl/ripemd.h>

// ... (Boost includes)
#include <boost/foreach.hpp>
#include <boost/lexical_cast.hpp>
// ... (other boost includes)
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L21-L83)*

The file `db.h` (not shown here but included by `headers.h`) would provide the interface for Berkeley DB.

## Version Information

This is Bitcoin version 0.2.0. Enhancements from v0.1.5 include UI improvements like minimize to tray, startup on boot, and an NSIS installer.

From `changelog.txt`:
```
Changes after 0.1.5:
--------------------
+ Options dialog layout changed - added the UI options panel
+ Minimize to tray feature
+ Startup on system boot feature
+ Ask before closing
+ NSIS installer
```
*Referenced from [changelog.txt](https://github.com/jonzou/btc020/blob/ebbdade7/changelog.txt#L1-L7)*

The system is distributed under the MIT/X11 software license.

From `license.txt`:
```
Copyright (c) 2009 Satoshi Nakamoto

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
```
*Referenced from [license.txt](https://github.com/jonzou/btc020/blob/ebbdade7/license.txt)*

The first lines of `headers.h` also reference the license:
```cpp
// Copyright (c) 2009 Satoshi Nakamoto
// Distributed under the MIT/X11 software license, see the accompanying
// file license.txt or http://www.opensource.org/licenses/mit-license.php.
```
*Referenced from [headers.h](https://github.com/jonzou/btc020/blob/ebbdade7/headers.h#L1-L2)*

This provides open-source access to the cryptocurrency implementation.
