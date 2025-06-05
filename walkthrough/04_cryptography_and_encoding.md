# Cryptography and Encoding in Bitcoin v0.2.0

This document covers the cryptographic foundations of Bitcoin v0.2.0, including key management (ECDSA via `CKey`), address encoding (Base58Check), mathematical primitives for large numbers (`CBigNum`), and hashing utilities (`uint256`, `sha.h`). These components are essential for the security and functioning of the Bitcoin system.

## Core Cryptographic Architecture

The cryptographic subsystem in Bitcoin v0.2.0 is built upon:
1.  **Key Management (`key.h`)**: Handles Elliptic Curve Digital Signature Algorithm (ECDSA) private and public keys.
2.  **Address Encoding (`base58.h`)**: Manages the conversion of public key hashes into human-readable Bitcoin addresses using Base58Check encoding.
3.  **Mathematical Foundations (`bignum.h`, `uint256.h`)**: Provides arbitrary-precision arithmetic for cryptographic calculations and custom integer types for hashes.
4.  **Hashing Utilities (`sha.h`, `util.h`)**: Implements cryptographic hash functions like SHA-256 and RIPEMD-160.

## Key Management (`CKey` class)
*Relevant file: [key.h](https://github.com/jonzou/btc020/blob/ebbdade7/key.h)*

The `CKey` class is central to managing cryptographic keys. It wraps OpenSSL's `EC_KEY` functionality to provide ECDSA operations using the `secp256k1` elliptic curve.

### `CKey` Class Structure
*Defined in [key.h Lines 43-169](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L43-L169)*
```cpp
class CKey
{
protected:
    EC_KEY* pkey; // Pointer to OpenSSL's EC_KEY structure
    bool fSet;    // Flag indicating if the key material is set

public:
    CKey()
    {
        pkey = EC_KEY_new_by_curve_name(NID_secp256k1); // Initialize with secp256k1
        if (pkey == NULL)
            throw key_error("CKey::CKey() : EC_KEY_new_by_curve_name failed");
        fSet = false;
    }

    // ... Copy constructor, assignment operator, destructor ...

    bool IsNull() const { return !fSet; }
    void MakeNewKey(); // Generates a new public/private key pair

    bool SetPrivKey(const CPrivKey& vchPrivKey); // Sets private key
    CPrivKey GetPrivKey() const; // Retrieves private key

    bool SetPubKey(const vector<unsigned char>& vchPubKey); // Sets public key
    vector<unsigned char> GetPubKey() const; // Retrieves public key

    bool Sign(uint256 hash, vector<unsigned char>& vchSig); // Signs a hash
    bool Verify(uint256 hash, const vector<unsigned char>& vchSig); // Verifies a signature

    // Static utility methods for signing and verification
    static bool Sign(const CPrivKey& vchPrivKey, uint256 hash, vector<unsigned char>& vchSig);
    static bool Verify(const vector<unsigned char>& vchPubKey, uint256 hash, const vector<unsigned char>& vchSig);
};
```
*   The constructor initializes an `EC_KEY` object for the `secp256k1` curve.
    *Referenced from [key.h Line 52](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L52)*
*   `fSet` tracks if the key object holds a valid key.
    *Referenced from [key.h Line 47](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L47)*

### Key Operations
*   **Generation (`MakeNewKey`)**: Uses `EC_KEY_generate_key()` to create a new random key pair.
    *Referenced from [key.h Lines 84-89](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L84-L89)*
*   **Private Key Handling**:
    *   `SetPrivKey()`: Imports a DER-encoded private key using `d2i_ECPrivateKey()`.
    *   `GetPrivKey()`: Exports a DER-encoded private key using `i2d_ECPrivateKey()`.
    *   `CPrivKey` is a `typedef vector<unsigned char, secure_allocator<unsigned char>>` ensuring private key data is securely zeroed upon deallocation.
        *Referenced from [key.h Line 39](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L39)*
*   **Public Key Handling**:
    *   `SetPubKey()`: Imports a public key (typically in compressed or uncompressed format) using `o2i_ECPublicKey()`.
    *   `GetPubKey()`: Exports a public key using `i2o_ECPublicKey()`.
*   **Signing and Verification**:
    *   `Sign(uint256 hash, ...)`: Creates an ECDSA signature for a given hash using `ECDSA_sign()`.
        *Referenced from [key.h Lines 133-141](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L133-L141)*
    *   `Verify(uint256 hash, ...)`: Verifies an ECDSA signature using `ECDSA_verify()`.
        *Referenced from [key.h Lines 143-151](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L143-L151)*

### Error Handling
A `key_error` exception class (derived from `std::runtime_error`) is used for error reporting.
*Defined in [key.h Lines 31-35](https://github.com/jonzou/btc020/blob/ebbdade7/key.h#L31-L35)*

## Address Encoding (`base58.h`)
*Relevant file: [base58.h](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h)*

Bitcoin addresses are human-readable representations of public key hashes, using Base58Check encoding.

### Base58 Encoding
Base58 encoding uses an alphabet of 58 characters, excluding visually ambiguous ones (0, O, I, l).
*Alphabet defined in [base58.h Line 16](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L16)*
```cpp
static const char* pszBase58 = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";
```
*   `EncodeBase58()`: Converts a byte vector to a Base58 string. It treats the input bytes as a large big-endian number, repeatedly divides by 58, and maps remainders to `pszBase58` characters. Leading zero bytes are preserved as '1's.
    *Referenced from [base58.h Lines 19-55](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L19-L55)*
*   `DecodeBase58()`: Converts a Base58 string back to a byte vector.
    *Referenced from [base58.h Lines 62-105](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L62-L105)*

### Base58Check Encoding
This adds a 4-byte checksum to detect errors.
*   `EncodeBase58Check()`:
    1.  Takes a byte vector (`vchIn`).
    2.  Appends the first 4 bytes of `Hash(vchIn.begin(), vchIn.end())` (double SHA-256) as a checksum.
        *Referenced from [base58.h Lines 121-122](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L121-L122)*
    3.  Encodes the result using `EncodeBase58()`.
*   `DecodeBase58Check()`:
    1.  Decodes the Base58 string.
    2.  Separates the payload and checksum.
    3.  Recalculates the checksum of the payload and verifies it.
        *Referenced from [base58.h Lines 135-141](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L135-L141)*

### Bitcoin Address Generation
A Bitcoin address is a Base58Check encoding of:
1.  A 1-byte version prefix (`ADDRESSVERSION = 0` for main network).
    *Referenced from [base58.h Line 155](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L155)*
2.  A 20-byte RIPEMD-160 hash of the SHA-256 hash of a public key (Hash160).

Key functions:
*   `Hash160ToAddress(uint160 hash160)`: Prepends version byte to `hash160` and Base58Check encodes it.
    *Referenced from [base58.h Lines 157-163](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L157-L163)*
*   `PubKeyToAddress(const vector<unsigned char>& vchPubKey)`: Calculates `Hash160(vchPubKey)` then calls `Hash160ToAddress()`.
    *Referenced from [base58.h Lines 198-201](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L198-L201)*
*   `AddressToHash160(const char* psz, uint160& hash160Ret)`: Decodes and validates a Bitcoin address, returning the version and Hash160.
    *Referenced from [base58.h Lines 165-182](https://github.com/jonzou/btc020/blob/ebbdade7/base58.h#L165-L182)*

## Mathematical Foundations

### Arbitrary-Precision Arithmetic (`CBigNum` class)
*Relevant file: [bignum.h](https://github.com/jonzou/btc020/blob/ebbdade7/bignum.h)*

The `CBigNum` class wraps OpenSSL's `BIGNUM` library to handle large integers needed for cryptography.
It inherits from `BIGNUM` directly.
*Defined in [bignum.h Line 49](https://github.com/jonzou/btc020/blob/ebbdade7/bignum.h#L49)*
```cpp
class CBigNum : public BIGNUM
{
public:
    CBigNum() { BN_init(this); }
    // ... Constructors for various types (integers, strings, uint256) ...
    // ... Destructor calls BN_clear_free(this) ...

    // Setters/getters for various formats
    void setulong(unsigned long n);
    unsigned long getulong() const;
    // ... setint64, setuint64, setuint256, getuint256 ...
    void setvch(const std::vector<unsigned char>& vch); // From byte vector (MPI format)
    std::vector<unsigned char> getvch() const;         // To byte vector (MPI format)
    void SetHex(const std::string& str);               // From hex string

    // Compact format for difficulty representation in block headers
    CBigNum& SetCompact(unsigned int nCompact);
    unsigned int GetCompact() const;

    // Serialization
    unsigned int GetSerializeSize(...) const;
    template<typename Stream> void Serialize(Stream& s, ...) const;
    template<typename Stream> void Unserialize(Stream& s, ...);

    // Operator overloads for arithmetic (+, -, *, /, %, <<, >>, ++, --)
    // and comparison (==, !=, <, >, <=, >=)
};
```
*   **`CAutoBN_CTX`**: A helper class to manage `BN_CTX*` (OpenSSL's context for temporary variables in bignum operations) ensuring it's freed automatically.
    *Defined in [bignum.h Lines 21-45](https://github.com/jonzou/btc020/blob/ebbdade7/bignum.h#L21-L45)*
*   **Error Handling**: `bignum_error` (derived from `std::runtime_error`) is thrown on OpenSSL errors.
    *Defined in [bignum.h Lines 13-17](https://github.com/jonzou/btc020/blob/ebbdade7/bignum.h#L13-L17)*

### 256-bit Unsigned Integers (`uint256`, `base_uint`)
*Relevant file: [uint256.h](https://github.com/jonzou/btc020/blob/ebbdade7/uint256.h)*

The `base_uint<BITS>` template class provides a custom implementation for large fixed-size unsigned integers. `uint256` and `uint160` are instantiations of this template.
```cpp
template<unsigned int BITS>
class base_uint
{
protected:
    enum { WIDTH = BITS / 32 };
    unsigned int pn[WIDTH]; // Stores data as an array of 32-bit unsigned integers
public:
    // ... Constructors, assignment operators ...
    // ... Bitwise operators (^, &, |, ~, <<, >>) ...
    // ... Arithmetic operators (+, -, ++, --) ...
    // ... Comparison operators (<, <=, >, >=, ==, !=) ...

    std::string GetHex() const;
    void SetHex(const std::string& str);
    // ... begin(), end(), size() ...
    // ... Serialization methods ...
};

class uint160 : public base_uint160 { /* ... specific constructors ... */ };
class uint256 : public base_uint256 { /* ... specific constructors ... */ };
```
These classes are used for representing transaction hashes, block hashes, Merkle roots, and public key hashes (uint160). They offer basic arithmetic and bitwise operations, hex conversion, and serialization.

## Hashing Utilities
*Relevant files: `sha.h` (provides SHA1 and SHA256 from CryptoPP), `util.h` (provides `Hash` and `Hash160`)*

Bitcoin v0.2.0 uses SHA-256 and RIPEMD-160.
*   **SHA-256**: The `CryptoPP::SHA256` class from `sha.h` is used.
    *Declared in [sha.h Lines 80-86](https://github.com/jonzou/btc020/blob/ebbdade7/sha.h#L80-L86)*
*   **`Hash()` (Double SHA-256)**: Typically, Bitcoin uses "Hash256", which is SHA-256 applied twice. This is implemented in `util.h` (not directly part of this subtask's file list, but crucial).
    ```cpp
    // In util.h (conceptual)
    // template<typename T1>
    // inline uint256 Hash(const T1 pbegin, const T1 pend)
    // {
    //     static unsigned char pblank[1];
    //     uint256 hash1;
    //     SHA256((pbegin == pend ? pblank : (unsigned char*)&pbegin[0]), (pend - pbegin) * sizeof(pbegin[0]), (unsigned char*)&hash1);
    //     uint256 hash2;
    //     SHA256((unsigned char*)&hash1, sizeof(hash1), (unsigned char*)&hash2);
    //     return hash2;
    // }
    ```
*   **`Hash160()` (SHA-256 then RIPEMD-160)**: Used for generating Bitcoin addresses. Public keys are first hashed with SHA-256, and the result is then hashed with RIPEMD-160. This is also typically found in `util.h`.
    ```cpp
    // In util.h (conceptual)
    // template<typename T1>
    // inline uint160 Hash160(const T1 pbegin, const T1 pend)
    // {
    //     static unsigned char pblank[1];
    //     uint256 hash1;
    //     SHA256((pbegin == pend ? pblank : (unsigned char*)&pbegin[0]), (pend - pbegin) * sizeof(pbegin[0]), (unsigned char*)&hash1);
    //     uint160 hash2;
    //     RIPEMD160((unsigned char*)&hash1, sizeof(hash1), (unsigned char*)&hash2);
    //     return hash2;
    // }
    ```
    The actual `RIPEMD160` function would be linked from OpenSSL, as seen in `headers.h` including `<openssl/ripemd.h>`.

This combination of ECDSA for signatures, Base58Check for human-readable addresses, robust large number arithmetic, and standard hashing functions forms the cryptographic backbone of Bitcoin v0.2.0.
