# Bitcoin v0.2.0 Script Engine

The Bitcoin script engine is a crucial component responsible for validating the conditions under which a transaction output can be spent. It uses a stack-based execution language similar to Forth. This document details the structure, opcodes, and evaluation process of this engine, primarily defined in `script.h` and implemented in `script.cpp`.

## Script Overview

A script in Bitcoin is a sequence of instructions (opcodes) and data pushes. There are two main types of scripts involved in a transaction:
1.  **`scriptPubKey` (locking script)**: Found in a transaction output (CTxOut), it specifies the conditions that must be met to spend the output.
2.  **`scriptSig` (unlocking script)**: Found in a transaction input (CTxIn), it provides the data and signatures required to satisfy the `scriptPubKey` of the output it references.

To validate a transaction input, the `scriptSig` is concatenated with the `scriptPubKey` of the referenced output, and the resulting script is executed. If the script evaluates to true, the transaction is considered valid.

## Core Components

### `CScript` Class
*Defined in [script.h Lines 327-517](https://github.com/jonzou/btc020/blob/ebbdade7/script.h#L327-L517)*

The `CScript` class, derived from `std::vector<unsigned char>`, represents a script. It provides methods to build scripts by pushing opcodes and data onto the byte vector.

```cpp
class CScript : public vector<unsigned char>
{
protected:
    CScript& push_int64(int64 n); // Helper for pushing small integers
    CScript& push_uint64(uint64 n); // Helper for pushing small unsigned integers

public:
    CScript() { }
    // ... Constructors ...

    // Appending opcodes and data
    CScript& operator<<(char b);
    CScript& operator<<(short b);
    // ... other integer types ...
    CScript& operator<<(opcodetype opcode);
    CScript& operator<<(const uint160& b); // For pubkey hashes
    CScript& operator<<(const uint256& b); // For other hashes
    CScript& operator<<(const CBigNum& b);
    CScript& operator<<(const vector<unsigned char>& b); // For pushing arbitrary data

    // Methods for parsing and utility
    bool GetOp(iterator& pc, opcodetype& opcodeRet, vector<unsigned char>& vchRet);
    bool GetOp(const_iterator& pc, opcodetype& opcodeRet, vector<unsigned char>& vchRet) const;
    void FindAndDelete(const CScript& b); // Removes all occurrences of script b from this script
    string ToString() const; // Human-readable representation
    void PrintHex() const;   // Hex representation
};
```
The `operator<<` overloads are used to easily construct scripts. For example, `CScript() << OP_DUP << OP_HASH160 << pubKeyHash << OP_EQUALVERIFY << OP_CHECKSIG;`.

Data pushing opcodes (`OP_PUSHDATA1`, `OP_PUSHDATA2`, `OP_PUSHDATA4`) are handled internally when `operator<<(const vector<unsigned char>& b)` is called, based on the size of `b`. Small integers (-1 to 16) are pushed using dedicated opcodes like `OP_1`, `OP_2`, ..., `OP_16`, `OP_1NEGATE`.

### Opcodes (`opcodetype`)
*Defined in [script.h Lines 13-145](https://github.com/jonzou/btc020/blob/ebbdade7/script.h#L13-L145)*

`opcodetype` is an enumeration listing all available operations in the script language.

```cpp
enum opcodetype
{
    // push value
    OP_0=0, OP_FALSE=OP_0,
    OP_PUSHDATA1=76, OP_PUSHDATA2, OP_PUSHDATA4,
    OP_1NEGATE, OP_RESERVED,
    OP_1, OP_TRUE=OP_1, OP_2, /* ... */ OP_16,

    // control
    OP_NOP, OP_VER, OP_IF, OP_NOTIF, OP_VERIF, OP_VERNOTIF, OP_ELSE,
    OP_ENDIF, OP_VERIFY, OP_RETURN,

    // stack ops
    OP_TOALTSTACK, OP_FROMALTSTACK, OP_2DROP, OP_2DUP, OP_3DUP, OP_2OVER,
    OP_2ROT, OP_2SWAP, OP_IFDUP, OP_DEPTH, OP_DROP, OP_DUP, OP_NIP,
    OP_OVER, OP_PICK, OP_ROLL, OP_ROT, OP_SWAP, OP_TUCK,

    // splice ops
    OP_CAT, OP_SUBSTR, OP_LEFT, OP_RIGHT, OP_SIZE,

    // bit logic
    OP_INVERT, OP_AND, OP_OR, OP_XOR, OP_EQUAL, OP_EQUALVERIFY,
    OP_RESERVED1, OP_RESERVED2,

    // numeric
    OP_1ADD, OP_1SUB, /* ... */ OP_NEGATE, OP_ABS, OP_NOT, OP_0NOTEQUAL,
    OP_ADD, OP_SUB, /* ... */ OP_MIN, OP_MAX, OP_WITHIN,

    // crypto
    OP_RIPEMD160, OP_SHA1, OP_SHA256, OP_HASH160, OP_HASH256,
    OP_CODESEPARATOR, OP_CHECKSIG, OP_CHECKSIGVERIFY,
    OP_CHECKMULTISIG, OP_CHECKMULTISIGVERIFY,

    // template matching params (not actual opcodes on the wire)
    OP_PUBKEY, OP_PUBKEYHASH,

    OP_INVALIDOPCODE = 0xFFFF,
};
```
Each opcode performs a specific operation on the script's execution stack. The `GetOpName()` function provides string representations for these opcodes.

## Script Evaluation (`EvalScript`)
*Defined in [script.cpp Lines 17-541](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L17-L541)*

The `EvalScript` function is the heart of the script engine. It processes a script, opcode by opcode, manipulating a stack of byte vectors (`valtype`).

```cpp
bool EvalScript(const CScript& script, const CTransaction& txTo, unsigned int nIn, int nHashType,
                vector<vector<unsigned char> >* pvStackRet)
{
    CAutoBN_CTX pctx; // For OpenSSL big number operations
    CScript::const_iterator pc = script.begin();
    CScript::const_iterator pend = script.end();
    CScript::const_iterator pbegincodehash = script.begin(); // For OP_CODESEPARATOR
    vector<bool> vfExec; // For conditional execution (OP_IF, OP_NOTIF, etc.)
    vector<valtype> stack;
    vector<valtype> altstack; // Alternate stack

    while (pc < pend)
    {
        bool fExec = !count(vfExec.begin(), vfExec.end(), false); // Is current branch active?

        opcodetype opcode;
        valtype vchPushValue;
        if (!script.GetOp(pc, opcode, vchPushValue)) // Parse next opcode and any pushed data
            return false;

        if (fExec && opcode <= OP_PUSHDATA4) // If active and data push, push to stack
            stack.push_back(vchPushValue);
        else if (fExec || (OP_IF <= opcode && opcode <= OP_ENDIF)) // Process opcodes
        switch (opcode)
        {
            // ... cases for all opcodes ...
        }
    }

    if (pvStackRet) // Optionally return the final stack state
        *pvStackRet = stack;
    // Script succeeds if stack is not empty and top item is true (non-zero)
    return (stack.empty() ? false : CastToBool(stack.back()));
}
```

**Key aspects of `EvalScript`**:
*   **Stack Machine**: Operates on a main stack (`stack`) and an alternate stack (`altstack`).
*   **Conditional Execution**: `OP_IF`, `OP_NOTIF`, `OP_ELSE`, `OP_ENDIF` allow parts of the script to be skipped based on boolean conditions. The `vfExec` vector tracks the execution status of nested conditionals.
*   **OP_VERIFY**: If the top stack value is false, execution is terminated immediately, and the script fails. If true, the value is popped.
*   **OP_RETURN**: Terminates execution immediately, script fails.
*   **Arithmetic and Logic**: Opcodes for basic arithmetic (`OP_ADD`, `OP_SUB`), bitwise logic (`OP_AND`, `OP_OR`), and comparisons (`OP_EQUAL`, `OP_NUMEQUAL`).
*   **Crypto Operations**: Hashing (`OP_SHA256`, `OP_HASH160`) and signature checking (`OP_CHECKSIG`, `OP_CHECKMULTISIG`).
*   **Final State**: For a script to be valid, after all opcodes are executed, the stack must not be empty, and the top item on the stack must evaluate to true (non-zero). `CastToBool` handles this conversion.

### Signature Checking

#### `SignatureHash`
*Defined in [script.cpp Lines 553-620](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L553-L620)*

Before a signature is checked, a hash of the transaction (or parts of it) is created. This is what is actually signed. The `SignatureHash` function constructs this hash.

```cpp
uint256 SignatureHash(CScript scriptCode, const CTransaction& txTo, unsigned int nIn, int nHashType)
{
    if (nIn >= txTo.vin.size()) /* ... error ... */

    CTransaction txTmp(txTo);

    // Remove OP_CODESEPARATOR from scriptCode
    scriptCode.FindAndDelete(CScript(OP_CODESEPARATOR));

    // Blank out other inputs' scriptSigs
    for (int i = 0; i < txTmp.vin.size(); i++)
        txTmp.vin[i].scriptSig = CScript();
    txTmp.vin[nIn].scriptSig = scriptCode; // Set the scriptSig of the input being signed to the scriptCode

    // Handle SIGHASH types
    if ((nHashType & 0x1f) == SIGHASH_NONE) { /* modify txTmp for SIGHASH_NONE */ }
    else if ((nHashType & 0x1f) == SIGHASH_SINGLE) { /* modify txTmp for SIGHASH_SINGLE */ }

    if (nHashType & SIGHASH_ANYONECANPAY) { /* modify txTmp for SIGHASH_ANYONECANPAY */ }

    // Serialize and hash
    CDataStream ss(SER_GETHASH);
    ss << txTmp << nHashType; // The hash type itself is also part of the data being hashed
    return Hash(ss.begin(), ss.end());
}
```
*   `scriptCode`: Typically a subset of the `scriptPubKey` (from the last `OP_CODESEPARATOR`).
*   `txTo`: The transaction containing the input to be signed.
*   `nIn`: The index of the input being signed.
*   `nHashType`: A flag determining which parts of the transaction are included in the hash (e.g., `SIGHASH_ALL`, `SIGHASH_NONE`, `SIGHASH_SINGLE`, `SIGHASH_ANYONECANPAY`). This allows for different signature behaviors.

#### `CheckSig`
*Defined in [script.cpp Lines 623-643](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L623-L643)*

This function is called by the `OP_CHECKSIG` and `OP_CHECKSIGVERIFY` opcodes. It verifies a given signature against a public key and the transaction data.

```cpp
bool CheckSig(vector<unsigned char> vchSig, vector<unsigned char> vchPubKey, CScript scriptCode,
              const CTransaction& txTo, unsigned int nIn, int nHashType)
{
    CKey key;
    if (!key.SetPubKey(vchPubKey)) // Parse public key
        return false;

    // Extract hash type from the end of the signature
    if (vchSig.empty()) return false;
    if (nHashType == 0) nHashType = vchSig.back(); // If not specified, use the one from sig
    else if (nHashType != vchSig.back()) return false; // Must match if specified
    vchSig.pop_back(); // Remove hash type byte

    uint256 sighash = SignatureHash(scriptCode, txTo, nIn, nHashType);
    if (key.Verify(sighash, vchSig)) // ECDSA verification
        return true;

    return false;
}
```

#### `OP_CHECKSIG` and `OP_CHECKSIGVERIFY`
During `EvalScript`:
1.  Pops signature (`vchSig`) and public key (`vchPubKey`) from the stack.
2.  Constructs `scriptCode` (part of the original `scriptPubKey` from the last `OP_CODESEPARATOR`).
3.  Calls `CheckSig(vchSig, vchPubKey, scriptCode, txTo, nIn, nHashType)`.
4.  Pushes true or false onto the stack.
5.  `OP_CHECKSIGVERIFY` additionally runs `OP_VERIFY` on the result.

#### `OP_CHECKMULTISIG` and `OP_CHECKMULTISIGVERIFY`
These opcodes handle multi-signature P2SH-like schemes.
*   **Format**: `... <sig1> ... <sigM> M <pubkey1> ... <pubkeyN> N OP_CHECKMULTISIG`
*   Pops `N` (number of public keys), then `N` public keys.
*   Pops `M` (number of required signatures), then `M` signatures.
*   Compares signatures against public keys sequentially. Due to a bug, an extra unused value is popped from the stack initially (not shown in the simplified format above, but handled in `EvalScript` by the `i` counter logic).
*   `scriptCode` is determined similarly, removing all pushed signatures from it.
*   The process involves iterating through signatures and public keys, calling `CheckSig` for valid pairs until `M` successful signature checks are achieved or possibilities are exhausted.

### Script Solving and Creation

#### `Solver`
*Defined in [script.cpp Lines 657-714](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L657-L714)*

The `Solver` function attempts to identify a standard script template from a `scriptPubKey` and extract the necessary parameters (like public keys or public key hashes). This version of Bitcoin recognizes two main templates:
1.  Pay-to-Pubkey: `OP_PUBKEY OP_CHECKSIG`
2.  Pay-to-PubkeyHash: `OP_DUP OP_HASH160 OP_PUBKEYHASH OP_EQUALVERIFY OP_CHECKSIG`

```cpp
bool Solver(const CScript& scriptPubKey, vector<pair<opcodetype, valtype> >& vSolutionRet)
{
    // Templates
    static vector<CScript> vTemplates;
    if (vTemplates.empty())
    {
        vTemplates.push_back(CScript() << OP_PUBKEY << OP_CHECKSIG);
        vTemplates.push_back(CScript() << OP_DUP << OP_HASH160 << OP_PUBKEYHASH << OP_EQUALVERIFY << OP_CHECKSIG);
    }

    // Scan templates
    // ... compares scriptPubKey against vTemplates ...
    // If a match is found, extracts OP_PUBKEY or OP_PUBKEYHASH and their data into vSolutionRet
}
```
If `Solver` recognizes the `scriptPubKey`, `vSolutionRet` will contain pairs like `(OP_PUBKEY, <actual_pubkey_bytes>)` or `(OP_PUBKEYHASH, <actual_hash_bytes>)`.

#### `SignSignature`
*Defined in [script.cpp Lines 802-826](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L802-L826)*

This function is used by the wallet to create a `scriptSig` for spending an output.
1.  Takes the `txFrom` (transaction containing the output to spend) and `txTo` (the spending transaction).
2.  Determines the `scriptPubKey` from `txFrom.vout[txin.prevout.n]`.
3.  Calculates the `SignatureHash`.
4.  Calls the other `Solver` overload (see below) to generate the `scriptSig` based on the `scriptPubKey` and the calculated hash.

```cpp
bool SignSignature(const CTransaction& txFrom, CTransaction& txTo, unsigned int nIn, int nHashType, CScript scriptPrereq)
{
    // ...
    const CTxOut& txout = txFrom.vout[txin.prevout.n];
    uint256 hash = SignatureHash(scriptPrereq + txout.scriptPubKey, txTo, nIn, nHashType);

    if (!Solver(txout.scriptPubKey, hash, nHashType, txin.scriptSig)) // This calls the Solver that produces a scriptSig
        return false;

    txin.scriptSig = scriptPrereq + txin.scriptSig;
    // ... (optional test solution) ...
    return true;
}
```

The `Solver` overload used by `SignSignature`:
*Defined in [script.cpp Lines 717-765](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L717-L765)*
This version of `Solver` takes a `scriptPubKey`, a `hash` (from `SignatureHash`), and `nHashType`, and produces `scriptSigRet`.
1.  It first calls the template-matching `Solver` to understand the `scriptPubKey`.
2.  If recognized, it retrieves the required keys from the wallet (`mapKeys`, `mapPubKeys`).
3.  For `OP_PUBKEY` template, it signs the `hash` with the private key corresponding to the public key.
4.  For `OP_PUBKEYHASH` template, it signs the `hash` and includes the actual public key in the `scriptSigRet` along with the signature.
5.  The signature produced by `CKey::Sign` is then appended with the `nHashType`.

#### `VerifySignature`
*Defined in [script.cpp Lines 829-842](https://github.com/jonzou/btc020/blob/ebbdade7/script.cpp#L829-L842)*

This function is a wrapper that combines `scriptSig` and `scriptPubKey` and then calls `EvalScript` to verify them. This is the primary entry point for validating a transaction input's signature claim.
```cpp
bool VerifySignature(const CTransaction& txFrom, const CTransaction& txTo, unsigned int nIn, int nHashType)
{
    assert(nIn < txTo.vin.size());
    const CTxIn& txin = txTo.vin[nIn];
    if (txin.prevout.n >= txFrom.vout.size()) return false;
    const CTxOut& txout = txFrom.vout[txin.prevout.n];

    if (txin.prevout.hash != txFrom.GetHash()) return false;

    // The scriptSig from the input and the scriptPubKey from the output being spent
    // are concatenated with an OP_CODESEPARATOR in between.
    return EvalScript(txin.scriptSig + CScript(OP_CODESEPARATOR) + txout.scriptPubKey, txTo, nIn, nHashType);
}
```

## Standard Script Forms

Bitcoin v0.2.0 primarily supports two standard transaction types through its `Solver` and signing logic:

1.  **Pay-to-Pubkey (P2PK)**:
    *   `scriptPubKey`: `<pubKey> OP_CHECKSIG`
    *   `scriptSig`: `<signature>`
2.  **Pay-to-PubkeyHash (P2PKH)**:
    *   `scriptPubKey`: `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
    *   `scriptSig`: `<signature> <pubKey>`

The `EvalScript` function is general and can execute any valid script, but the wallet's ability to create and sign transactions is limited to these standard forms.

This concludes the overview of the script engine in Bitcoin v0.2.0. It's a flexible system for defining spending conditions, with `EvalScript` as the core execution logic and various helper functions for signature creation and verification.
