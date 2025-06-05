# Cryptography and Encoding in Bitcoin

This document outlines some of the core cryptographic and encoding mechanisms used in this Bitcoin client, based on the provided header files.

## Key Management

Key management revolves around the `CKey` class defined in `key.h`. This class is responsible for generating, storing, and managing elliptic curve (specifically secp256k1) private and public keys.

### Key Generation

The primary method for generating a new key pair is `CKey::MakeNewKey()`.

```cpp
// From key.h
class CKey
{
protected:
    EC_KEY* pkey;
    bool fSet;

public:
    CKey()
    {
        pkey = EC_KEY_new_by_curve_name(NID_secp256k1);
        if (pkey == NULL)
            throw key_error("CKey::CKey() : EC_KEY_new_by_curve_name failed");
        fSet = false;
    }

    // ... other methods ...

    void MakeNewKey()
    {
        if (!EC_KEY_generate_key(pkey))
            throw key_error("CKey::MakeNewKey() : EC_KEY_generate_key failed");
        fSet = true;
    }

    // ... other methods ...
};
```

**Logic:**
1.  The `CKey` constructor initializes an `EC_KEY` object using the `secp256k1` curve (`EC_KEY_new_by_curve_name(NID_secp256k1)`). This prepares an object to hold an elliptic curve key.
2.  The `MakeNewKey()` method calls `EC_KEY_generate_key(pkey)`. This OpenSSL function generates a new private and corresponding public key pair using the curve specified during the `EC_KEY` object's initialization.
3.  If the key generation is successful, the `fSet` flag is set to `true`, indicating that the `CKey` object now holds a valid key pair.
4.  Error handling is present: if `EC_KEY_new_by_curve_name` or `EC_KEY_generate_key` fails, a `key_error` exception is thrown.

The private key can be retrieved using `GetPrivKey()` and the public key using `GetPubKey()`. These methods use OpenSSL's `i2d_ECPrivateKey` and `i2o_ECPublicKey` functions for serialization.

### Mermaid Flowchart for Key Generation (`CKey::MakeNewKey`)

```mermaid
graph TD
    A[Start CKey::MakeNewKey] --> B{pkey (EC_KEY object) initialized with secp256k1?};
    B -- Yes --> C[Call EC_KEY_generate_key(pkey)];
    B -- No (Constructor Failed) --> F[Throw key_error "EC_KEY_new_by_curve_name failed"];
    C -- Success --> D[Set fSet = true];
    C -- Failure --> E[Throw key_error "EC_KEY_generate_key failed"];
    D --> G[End - Key Pair Generated];
    E --> H[End];
    F --> H;
```

## Address Encoding

Bitcoin addresses are an encoded version of a hashed public key, using Base58Check encoding. The relevant functions are found in `base58.h`.

### Base58 Encoding Process

The function `EncodeBase58` (and its wrapper `EncodeBase58Check`) is central to this.

```cpp
// From base58.h

static const char* pszBase58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";

inline string EncodeBase58(const unsigned char* pbegin, const unsigned char* pend)
{
    CAutoBN_CTX pctx;
    CBigNum bn58 = 58;
    CBigNum bn0 = 0;

    // Convert big endian data to little endian
    // Extra zero at the end make sure bignum will interpret as a positive number
    vector<unsigned char> vchTmp(pend-pbegin+1, 0);
    reverse_copy(pbegin, pend, vchTmp.begin());

    // Convert little endian data to bignum
    CBigNum bn;
    bn.setvch(vchTmp);

    // Convert bignum to string
    string str;
    str.reserve((pend - pbegin) * 138 / 100 + 1);
    CBigNum dv;
    CBigNum rem;
    while (bn > bn0)
    {
        if (!BN_div(&dv, &rem, &bn, &bn58, pctx))
            throw bignum_error("EncodeBase58 : BN_div failed");
        bn = dv;
        unsigned int c = rem.getulong();
        str += pszBase58[c];
    }

    // Leading zeroes encoded as base58 zeros
    for (const unsigned char* p = pbegin; p < pend && *p == 0; p++)
        str += pszBase58[0];

    // Convert little endian string to big endian
    reverse(str.begin(), str.end());
    return str;
}

inline string EncodeBase58Check(const vector<unsigned char>& vchIn)
{
    // add 4-byte hash check to the end
    vector<unsigned char> vch(vchIn);
    uint256 hash = Hash(vch.begin(), vch.end()); // Assuming Hash is SHA256d
    vch.insert(vch.end(), (unsigned char*)&hash, (unsigned char*)&hash + 4);
    return EncodeBase58(vch);
}

// Example of use for addresses: PubKeyToAddress -> Hash160ToAddress -> EncodeBase58Check
inline string PubKeyToAddress(const vector<unsigned char>& vchPubKey)
{
    return Hash160ToAddress(Hash160(vchPubKey)); // Hash160 is RIPEMD160(SHA256(pubkey))
}

inline string Hash160ToAddress(uint160 hash160)
{
    // add 1-byte version number to the front
    vector<unsigned char> vch(1, ADDRESSVERSION); // ADDRESSVERSION = 0 for mainnet
    vch.insert(vch.end(), UBEGIN(hash160), UEND(hash160));
    return EncodeBase58Check(vch);
}
```

**Logic for `EncodeBase58`:**
1.  **Input**: A sequence of bytes (`pbegin` to `pend`).
2.  **BigNum Conversion**:
    *   The input bytes (assumed big-endian) are reversed into a temporary vector `vchTmp` and an extra zero byte is added at the end. This ensures the `CBigNum` interprets it as a positive number.
    *   `vchTmp` (now little-endian) is converted into a `CBigNum` object `bn`.
3.  **Base Conversion Loop**:
    *   While `bn` is greater than 0:
        *   Divide `bn` by 58 (`bn58`). The remainder is stored in `rem`, and the quotient in `dv`.
        *   The remainder (`rem.getulong()`) is used as an index into `pszBase58` to get the corresponding Base58 character. This character is appended to the result string `str`.
        *   `bn` is updated to the quotient (`dv`).
4.  **Leading Zeroes**: The original input data is scanned for leading zero bytes. For each leading zero, a Base58 '1' (which is `pszBase58[0]`) is appended to `str`. This is because leading zeros would be lost in the bignum conversion.
5.  **Reverse**: The resulting string `str` is reversed because the characters were generated in little-endian order (least significant digit first).

**Logic for `EncodeBase58Check` (used for addresses):**
1.  **Input**: A vector of bytes `vchIn` (e.g., version byte + RIPEMD160 hash).
2.  **Checksum**:
    *   A copy `vch` is made from `vchIn`.
    *   A checksum is calculated by hashing `vch` (typically `SHA256(SHA256(vch))`, though `Hash()` implementation details would be in `util.cpp` or similar) and taking the first 4 bytes of the resulting hash.
    *   These 4 checksum bytes are appended to `vch`.
3.  **Base58 Encode**: The extended `vch` (with checksum) is then passed to `EncodeBase58` to get the final Base58Check encoded string.

A Bitcoin address is formed by:
1.  Taking the public key.
2.  Hashing it: `RIPEMD160(SHA256(pubKey))` to get a `uint160` hash.
3.  Prepending a version byte (e.g., `0` for mainnet).
4.  Applying `EncodeBase58Check` to the version byte + hash160.

### Mermaid Flowchart for `EncodeBase58Check` (Simplified for Address Encoding)

```mermaid
graph TD
    A[Start EncodeBase58Check (for Address)] --> B[Input: Version Byte + RIPEMD160(SHA256(PubKey))];
    B --> C[vch = VersionByte + Hash160];
    C --> D[checksum = First4Bytes(SHA256(SHA256(vch)))];
    D --> E[vch_with_checksum = vch + checksum];
    E --> F[Call EncodeBase58(vch_with_checksum)];
    F --> G[Output: Base58 Encoded String (Address)];
    G --> H[End];

    subgraph EncodeBase58 Internal
        direction LR
        F1[Input: bytes_to_encode (vch_with_checksum)] --> F2[Convert to CBigNum bn];
        F2 --> F3{Loop while bn > 0};
        F3 -- Yes --> F4[remainder = bn % 58];
        F4 --> F5[char = pszBase58[remainder]];
        F5 --> F6[Append char to string_reversed];
        F6 --> F7[bn = bn / 58];
        F7 --> F3;
        F3 -- No --> F8[Prepend '1' for each leading zero in original bytes_to_encode];
        F8 --> F9[Reverse string_reversed];
        F9 --> F10[Output: Base58 String];
    end
```

## Mathematical Foundations

Several header files define structures and functions for essential mathematical operations in Bitcoin's cryptography.

### `bignum.h`

This file provides the `CBigNum` class, which is a wrapper around OpenSSL's `BIGNUM` library.
```cpp
// From bignum.h
class CBigNum : public BIGNUM
{
public:
    CBigNum()
    {
        BN_init(this);
    }
    // ... various constructors and methods ...

    void setvch(const std::vector<unsigned char>& vch)
    {
        // ... (implementation uses BN_mpi2bn) ...
    }

    std::vector<unsigned char> getvch() const
    {
        // ... (implementation uses BN_bn2mpi) ...
    }

    CBigNum& SetCompact(unsigned int nCompact); // For difficulty representation
    unsigned int GetCompact() const;            // For difficulty representation
};
```
**Purpose**:
*   **Arbitrary-Precision Arithmetic**: `CBigNum` allows for calculations involving very large integers, which are fundamental to elliptic curve cryptography (ECC) operations (like key generation and signing) and also for handling the 256-bit numbers used in difficulty targets and hashes.
*   **Serialization/Deserialization**: It provides methods to convert between byte vectors (`std::vector<unsigned char>`) and bignum representations (`setvch`, `getvch`). This is crucial for encoding/decoding numbers from network messages or disk.
*   **Difficulty Representation**: `SetCompact` and `GetCompact` are used to convert between the compact "nBits" representation of mining difficulty stored in block headers and the actual 256-bit target value.
*   The Base58 encoding process heavily relies on `CBigNum` for the conversion between byte sequences and a base-58 representation.

### `sha.h` / `sha.cpp`

These files provide implementations of SHA-1 and SHA-2 (SHA-224, SHA-256, SHA-384, SHA-512) hashing algorithms. The implementations are based on Crypto++ code.

```cpp
// From sha.h (illustrative)
namespace CryptoPP
{
class SHA256
{
public:
    typedef word32 HashWordType;
    static void InitState(word32 *state);
    static void Transform(word32 *digest, const word32 *data);
    // ...
};
} // namespace CryptoPP

// From sha.cpp (illustrative of Transform)
void SHA256::Transform(word32 *state, const word32 *data)
{
    // ... (Core SHA-256 compression function logic) ...
    // Involves rounds of operations using Ch, Maj, S0, S1, s0, s1 functions
    // and SHA256_K constants.
}
```
**Purpose**:
*   **Hashing**: Cryptographic hashing is a cornerstone of Bitcoin.
    *   **SHA-256**: Used extensively. For example:
        *   Transaction hashes are the double SHA-256 of the transaction data.
        *   Block hashes are the double SHA-256 of the block header.
        *   The Merkle tree in blocks uses SHA-256.
        *   Public keys are hashed with SHA-256 before RIPEMD-160 when generating addresses.
    *   **RIPEMD-160**: While not defined in `sha.h`, it's commonly used in conjunction with SHA-256 for address generation (`Hash160` often refers to `RIPEMD160(SHA256(data))`). The `sha.h` and `sha.cpp` files provide the SHA part of this.
*   **Proof-of-Work**: The mining process involves finding a nonce such that the SHA-256 hash of a block header is below a certain target.
*   **Data Integrity**: Hashes are used to ensure data integrity throughout the system.

The `sha.cpp` file contains the low-level implementation details of the SHA algorithms, including the initialization vectors (`InitState`) and the compression function (`Transform`) which processes data blocks.

### `uint256.h`

This file defines fixed-size unsigned integer types, specifically `uint160` and `uint256`, which are crucial for handling hash values and other large numbers that don't require arbitrary-precision arithmetic but need to be larger than standard integer types.

```cpp
// From uint256.h
template<unsigned int BITS>
class base_uint
{
protected:
    enum { WIDTH=BITS/32 };
    unsigned int pn[WIDTH];
public:
    // ... (various operators: !, ~, -, =, ^=, &=, |=, <<=, >>=, +=, -=, ++, --)
    // ... (comparison operators: <, <=, >, >=, ==, !=)
    // ... (SetHex, GetHex, ToString, Serialize, Unserialize)
};

class uint160 : public base_uint160 { /* ...constructors... */ };
class uint256 : public base_uint256 { /* ...constructors... */ };
```
**Purpose**:
*   **Hash Representation**: `uint256` is the standard type for representing SHA-256 hashes (e.g., block hashes, transaction hashes, Merkle roots). `uint160` is used for RIPEMD-160 hashes (e.g., the result of `Hash160(pubKey)` used in address generation).
*   **Efficiency**: For fixed-size numbers (160 or 256 bits), these classes are more efficient than `CBigNum` as they avoid the overhead of dynamic memory allocation and complex library calls for simpler arithmetic.
*   **Type Safety**: Provides distinct types for these common large numbers, improving code readability and safety.
*   **Serialization**: They include methods for serialization (`Serialize`, `Unserialize`) and string conversion (`GetHex`, `SetHex`, `ToString`), which are necessary for network communication and storage.
*   **Bitwise Operations**: They support a rich set of bitwise and arithmetic operations, essential for various cryptographic computations and manipulations of hash data.
