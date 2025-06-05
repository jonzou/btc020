# Bitcoin Script Engine

This document details the structure, execution, and role of the scripting language used in this Bitcoin client, based on `script.h` and `script.cpp`.

## Script Opcodes and Structure

### Structure (`CScript`)

A script in Bitcoin, represented by the `CScript` class, is a sequence of operations (opcodes) and data pushes. It's essentially a `std::vector<unsigned char>`.

```cpp
// From script.h
class CScript : public vector<unsigned char>
{
    // ... constructors and helper methods ...

public:
    CScript& operator<<(opcodetype opcode); // Pushes an opcode
    CScript& operator<<(const CBigNum& b);  // Pushes a CBigNum (serialized to vch)
    CScript& operator<<(const vector<unsigned char>& b); // Pushes data
    // ... other overloads for pushing various types ...

    bool GetOp(iterator& pc, opcodetype& opcodeRet, vector<unsigned char>& vchRet); // Parses an opcode and data from the script
    // ...
};
```

Scripts are created by "pushing" opcodes or data onto them. For example, `CScript() << OP_DUP << OP_HASH160 << vchPubKeyHash << OP_EQUALVERIFY << OP_CHECKSIG;` creates a standard Pay-to-PubKeyHash scriptPubKey.

### Opcodes (`opcodetype`)

Opcodes are defined in the `opcodetype` enum in `script.h`. They can be categorized as follows:

**1. Push Value Opcodes:**
   - `OP_0` (`OP_FALSE`): Pushes an empty vector (representing false or zero).
   - `OP_PUSHDATA1`, `OP_PUSHDATA2`, `OP_PUSHDATA4`: Opcodes that specify the length of the data to be pushed next. Data itself up to 75 bytes can be pushed by an opcode representing its length (0x01 to 0x4b).
   - `OP_1NEGATE`: Pushes -1.
   - `OP_1` (`OP_TRUE`) to `OP_16`: Push the number 1 through 16 onto the stack.

**2. Control Flow Opcodes:**
   - `OP_NOP`: No operation.
   - `OP_VER`: Pushes the client version. (Legacy, not used in modern Bitcoin)
   - `OP_IF`, `OP_NOTIF`: Executes the following statements if the top stack value is true (or false for `OP_NOTIF`).
   - `OP_VERIF`, `OP_VERNOTIF`: Executes if client version matches/doesn't match top stack value. (Legacy)
   - `OP_ELSE`: Executes if the preceding `OP_IF` or `OP_NOTIF` was not executed.
   - `OP_ENDIF`: Ends an `OP_IF`/`OP_ELSE` block.
   - `OP_VERIFY`: Marks transaction as invalid if top stack value is false. If true, the value is popped.
   - `OP_RETURN`: Marks transaction as invalid.

**3. Stack Operations:**
   - `OP_TOALTSTACK`: Moves the top item from the main stack to the alternate stack.
   - `OP_FROMALTSTACK`: Moves the top item from the alternate stack to the main stack.
   - `OP_2DROP`, `OP_DROP`: Removes top one or two items.
   - `OP_2DUP`, `OP_3DUP`, `OP_DUP`: Duplicates top item(s).
   - `OP_IFDUP`: Duplicates the top stack item if it's not zero.
   - `OP_NIP`: Removes the second-to-top stack item.
   - `OP_OVER`: Copies the second-to-top stack item to the top.
   - `OP_PICK`, `OP_ROLL`: Copies (`PICK`) or moves (`ROLL`) an item from a specific depth to the top.
   - `OP_ROT`: Rotates the top three items.
   - `OP_SWAP`: Swaps the top two items.
   - `OP_TUCK`: Copies the top item and inserts it before the second-to-top item.
   - `OP_DEPTH`: Pushes the number of items on the stack.

**4. Splice Operations (String/Byte Manipulation):**
   - `OP_CAT`: Concatenates the top two items. (Disabled in modern Bitcoin)
   - `OP_SUBSTR`: Returns a substring of the top item. (Disabled)
   - `OP_LEFT`, `OP_RIGHT`: Returns a prefix/suffix of the top item. (Disabled)
   - `OP_SIZE`: Pushes the size of the top item without consuming it.

**5. Bitwise Logic:**
   - `OP_INVERT`: Flips all bits of the top item. (Disabled)
   - `OP_AND`, `OP_OR`, `OP_XOR`: Bitwise operations on the top two items. (Disabled)
   - `OP_EQUAL`: Pushes true if the top two items are equal, false otherwise.
   - `OP_EQUALVERIFY`: Same as `OP_EQUAL`, but then executes `OP_VERIFY`.

**6. Numeric Operations:**
   - `OP_1ADD`, `OP_1SUB`: Adds/subtracts 1 from the top item.
   - `OP_2MUL`, `OP_2DIV`: Multiplies/divides the top item by 2. (Disabled)
   - `OP_NEGATE`: Negates the top item.
   - `OP_ABS`: Absolute value of the top item.
   - `OP_NOT`: Pushes 1 if the input is 0, 0 otherwise.
   - `OP_0NOTEQUAL`: Pushes 1 if the input is not 0, 0 otherwise.
   - `OP_ADD`, `OP_SUB`, `OP_MUL`, `OP_DIV`, `OP_MOD`: Basic arithmetic on top two numbers. (`OP_MUL`, `OP_DIV`, `OP_MOD` are disabled)
   - `OP_LSHIFT`, `OP_RSHIFT`: Bitwise shifts. (Disabled)
   - `OP_BOOLAND`, `OP_BOOLOR`: Logical AND/OR on top two numbers (interpreted as booleans).
   - `OP_NUMEQUAL`, `OP_NUMNOTEQUAL`: Numeric equality/inequality.
   - `OP_NUMEQUALVERIFY`: Same as `OP_NUMEQUAL` then `OP_VERIFY`.
   - `OP_LESSTHAN`, `OP_GREATERTHAN`, `OP_LESSTHANOREQUAL`, `OP_GREATERTHANOREQUAL`: Numeric comparisons.
   - `OP_MIN`, `OP_MAX`: Returns the minimum or maximum of the top two items.
   - `OP_WITHIN`: Checks if a number is within a given range.

**7. Cryptographic Operations:**
   - `OP_RIPEMD160`, `OP_SHA1`, `OP_SHA256`: Hashes the top item with the specified algorithm.
   - `OP_HASH160`: Hashes the top item: `RIPEMD160(SHA256(item))`.
        ```cpp
        // Example usage in EvalScript
        else if (opcode == OP_HASH160)
        {
            uint160 hash160 = Hash160(vch); // vch is stacktop(-1)
            memcpy(&vchHash[0], &hash160, sizeof(hash160));
        }
        ```
   - `OP_HASH256`: Hashes the top item twice with SHA256: `SHA256(SHA256(item))`.
   - `OP_CODESEPARATOR`: Marks a point in the script for signature checking. All script code before this point is ignored in signature verification for that `OP_CHECKSIG` operation.
   - `OP_CHECKSIG`, `OP_CHECKSIGVERIFY`: Verifies a signature against a public key.
        - Takes a signature and a public key from the stack.
        - The script from the last `OP_CODESEPARATOR` to the end of the script (with the signature itself removed) is used as the "message" being signed.
        - `CheckSig` function is called, which involves `SignatureHash` to prepare the actual data to be verified.
        - Pushes true if valid, false otherwise. `OP_CHECKSIGVERIFY` also runs `OP_VERIFY`.
        ```cpp
        // Example usage in EvalScript for OP_CHECKSIG
        case OP_CHECKSIG:
        case OP_CHECKSIGVERIFY:
        {
            // (sig pubkey -- bool)
            // ... stack checks ...
            valtype& vchSig    = stacktop(-2);
            valtype& vchPubKey = stacktop(-1);
            CScript scriptCode(pbegincodehash, pend); // pbegincodehash is from last OP_CODESEPARATOR
            scriptCode.FindAndDelete(CScript(vchSig)); // Remove signature from scriptCode
            bool fSuccess = CheckSig(vchSig, vchPubKey, scriptCode, txTo, nIn, nHashType);
            // ... push fSuccess, handle OP_CHECKSIGVERIFY ...
        }
        break;
        ```
   - `OP_CHECKMULTISIG`, `OP_CHECKMULTISIGVERIFY`: Verifies multiple signatures against multiple public keys (m-of-n).

**8. Template Matching Parameters (Pseudo-Opcodes for Solver):**
   - `OP_PUBKEY`: Represents a public key in script templates for the `Solver`.
   - `OP_PUBKEYHASH`: Represents a public key hash in script templates.

Many opcodes were disabled over time for security reasons (e.g., `OP_CAT`, `OP_SUBSTR`, `OP_LSHIFT`). This codebase reflects an earlier version where more might be active.

## Script Execution

The `EvalScript` function in `script.cpp` is the heart of the script engine. It evaluates a given script using a stack-based execution model.

```cpp
// From script.cpp
bool EvalScript(const CScript& script, const CTransaction& txTo, unsigned int nIn, int nHashType,
                vector<vector<unsigned char> >* pvStackRet)
{
    CAutoBN_CTX pctx; // For CBigNum operations
    CScript::const_iterator pc = script.begin();
    CScript::const_iterator pend = script.end();
    CScript::const_iterator pbegincodehash = script.begin(); // For OP_CHECKSIG
    vector<bool> vfExec; // For conditional execution (OP_IF/OP_NOTIF)
    vector<valtype> stack;    // Main stack
    vector<valtype> altstack; // Alternate stack
    // ...

    while (pc < pend)
    {
        bool fExec = !count(vfExec.begin(), vfExec.end(), false); // True if not inside a false branch of OP_IF

        opcodetype opcode;
        valtype vchPushValue;
        if (!script.GetOp(pc, opcode, vchPushValue)) // Get next opcode and potential data
            return false;

        if (fExec && opcode <= OP_PUSHDATA4) // If executing and it's a data push
            stack.push_back(vchPushValue);
        else if (fExec || (OP_IF <= opcode && opcode <= OP_ENDIF)) // Always process conditionals
        switch (opcode)
        {
            // ... cases for each opcode ...
            // Example: OP_DUP
            case OP_DUP:
            {
                // (x -- x x)
                if (stack.size() < 1)
                    return false;
                valtype vch = stacktop(-1); // stacktop(-1) gets top item
                stack.push_back(vch);
            }
            break;

            // Example: OP_ADD
            case OP_ADD:
            {
                // (x1 x2 -- out)
                if (stack.size() < 2)
                    return false;
                CBigNum bn1(stacktop(-2));
                CBigNum bn2(stacktop(-1));
                CBigNum bn = bn1 + bn2;
                stack.pop_back();
                stack.pop_back();
                stack.push_back(bn.getvch());
            }
            break;

            // Example: OP_IF
            case OP_IF:
            // ...
            {
                bool fValue = false;
                if (fExec) { /* get value from stack */ }
                vfExec.push_back(fValue); // Push execution state for this conditional
            }
            break;

            // Example: OP_ENDIF
            case OP_ENDIF:
            {
                if (vfExec.empty()) return false;
                vfExec.pop_back(); // Pop execution state
            }
            break;

            default:
                return false; // Invalid opcode
        }
    }

    if (pvStackRet)
        *pvStackRet = stack; // Optionally return the final stack state

    return (stack.empty() ? false : CastToBool(stack.back())); // Script succeeds if top stack item is true
}
```

**Parameters:**
*   `script`: The `CScript` to be evaluated.
*   `txTo`: The transaction whose input is being verified (used for `OP_CHECKSIG`).
*   `nIn`: The index of the input in `txTo` that is being verified.
*   `nHashType`: The signature hash type, affecting what parts of `txTo` are signed in `OP_CHECKSIG`.
*   `pvStackRet` (optional): If provided, the final state of the stack is returned here.

**Execution Process:**
1.  **Initialization**:
    *   `pc`: Program counter, an iterator to the current position in the script.
    *   `pbegincodehash`: Points to the start of the part of the script to be hashed for `OP_CHECKSIG` (updated by `OP_CODESEPARATOR`).
    *   `vfExec`: A stack of booleans to handle nested `OP_IF`/`OP_NOTIF`/`OP_ELSE`/`OP_ENDIF` blocks. `fExec` is true if the current instruction should be executed.
    *   `stack`: The main data stack.
    *   `altstack`: An alternate stack for temporary storage.
2.  **Main Loop**: Iterates through the script opcode by opcode (`script.GetOp`).
3.  **Conditional Execution**: If `fExec` is false (due to an `OP_IF` condition not being met), most opcodes are skipped. Conditional opcodes (`OP_IF` to `OP_ENDIF`) are always processed to maintain the `vfExec` stack correctly.
4.  **Opcode Handling**:
    *   **Data Pushes**: If the opcode is a data push instruction (e.g., `OP_PUSHDATA1` or a direct push like `OP_1`), the data (`vchPushValue`) is pushed onto the `stack`. Small integers (`OP_1` to `OP_16`, `OP_1NEGATE`) are converted to their byte representation and pushed.
    *   **Other Opcodes**: A `switch` statement handles each opcode:
        *   Stack manipulation opcodes (e.g., `OP_DUP`, `OP_SWAP`, `OP_DROP`) modify the `stack` directly.
        *   Arithmetic/logic opcodes (e.g., `OP_ADD`, `OP_EQUAL`) typically pop one or more values from the stack, perform an operation, and push the result back. `CBigNum` is used for numeric operations.
        *   Cryptographic opcodes (e.g., `OP_HASH160`, `OP_CHECKSIG`) perform their respective crypto operations. `OP_CHECKSIG` involves complex logic using `SignatureHash` and `CheckSig`.
        *   Control flow opcodes (`OP_IF`, `OP_VERIFY`, etc.) modify `vfExec` or terminate execution based on stack values.
5.  **Termination**:
    *   Execution stops if `pc` reaches `pend` (end of script), an invalid opcode is encountered, an operation fails due to insufficient stack items, or an `OP_RETURN` is executed.
    *   An `OP_VERIFY`, `OP_EQUALVERIFY`, `OP_CHECKSIGVERIFY`, or `OP_NUMEQUALVERIFY` failing also terminates execution by setting `pc = pend`.
6.  **Result**: The script evaluation is successful if, after execution, the stack is not empty and the top item on the stack, when cast to a boolean, is true. `CastToBool` considers a `CBigNum` value of zero (or an empty vector) as false, and anything else as true.

### Mermaid Flowchart for `EvalScript` (Simplified)

```mermaid
graph TD
    A[Start EvalScript(script, txTo, nIn, ...)] --> B[Initialize pc, stacks, vfExec];
    B --> C{pc < script.end()?};
    C -- No --> D{Stack empty?};
    D -- Yes --> E[Return False];
    D -- No --> F[result = CastToBool(stack.back())];
    F --> G[Return result];

    C -- Yes --> H[fExec = !any_false_in_vfExec];
    H --> I[GetOp(pc, opcode, vchPushValue)];
    I -- Failure to GetOp --> Fail[Return False];

    I -- Success --> J{fExec AND opcode is PUSHDATA?};
    J -- Yes --> K[stack.push(vchPushValue)];
    K --> C;

    J -- No --> L{fExec OR opcode is IF/NOTIF/ELSE/ENDIF?};
    L -- No (Opcode skipped) --> C;

    L -- Yes --> M[Switch (opcode)];
    M -- OP_DUP --> N[Check stack size (>=1)];
    N -- OK --> O[val = stacktop(-1); stack.push(val)];
    O --> C;
    N -- Fail --> Fail;

    M -- OP_CHECKSIG --> P[Check stack size (>=2)];
    P -- OK --> Q[vchSig = stacktop(-2); vchPubKey = stacktop(-1)];
    Q --> R[scriptCode = sub-script from pbegincodehash];
    R --> S[scriptCode.FindAndDelete(vchSig)];
    S --> T[fSuccess = CheckSig(vchSig, vchPubKey, scriptCode, ...)];
    T --> U[stack.pop; stack.pop; stack.push(fSuccess)];
    U --> C;
    P -- Fail --> Fail;

    M -- OP_IF --> V[fValue = false; IF fExec: fValue = CastToBool(stacktop(-1)); stack.pop];
    V --> W[vfExec.push(fValue)];
    W --> C;

    M -- OP_ENDIF --> X[vfExec.pop()];
    X --> C;

    M -- Other Opcodes --> Y[Execute Opcode Logic];
    Y -- Error (e.g. stack underflow) --> Fail;
    Y -- Success --> C;

    M -- Invalid Opcode --> Fail;
```

## Standard Script Types and Solving

Bitcoin transactions typically use a few "standard" script forms. The `Solver` function in `script.cpp` attempts to identify these forms in a `scriptPubKey` and determine what data (signatures, public keys) is needed to satisfy it.

**Standard Script Types (Inferred from `Solver` templates):**

1.  **Pay-to-PublicKey (P2PK):**
    *   `scriptPubKey`: `<pubKey> OP_CHECKSIG`
    *   `scriptSig` needed: `<signature>`
2.  **Pay-to-PublicKeyHash (P2PKH):**
    *   `scriptPubKey`: `OP_DUP OP_HASH160 <pubKeyHash> OP_EQUALVERIFY OP_CHECKSIG`
    *   `scriptSig` needed: `<signature> <pubKey>`

**`Solver` Function:**

There are two overloaded `Solver` functions:
1.  `Solver(const CScript& scriptPubKey, vector<pair<opcodetype, valtype> >& vSolutionRet)`: This function analyzes a `scriptPubKey` against a list of predefined templates.
    ```cpp
    // From script.cpp
    bool Solver(const CScript& scriptPubKey, vector<pair<opcodetype, valtype> >& vSolutionRet)
    {
        // Templates
        static vector<CScript> vTemplates;
        if (vTemplates.empty())
        {
            // Standard tx, sender provides pubkey, receiver adds signature
            vTemplates.push_back(CScript() << OP_PUBKEY << OP_CHECKSIG);

            // Short account number tx, sender provides hash of pubkey, receiver provides signature and pubkey
            vTemplates.push_back(CScript() << OP_DUP << OP_HASH160 << OP_PUBKEYHASH << OP_EQUALVERIFY << OP_CHECKSIG);
        }

        const CScript& script1 = scriptPubKey; // The script to solve
        foreach(const CScript& script2, vTemplates) // Iterate through known templates
        {
            vSolutionRet.clear();
            opcodetype opcode1, opcode2;
            vector<unsigned char> vch1, vch2;

            CScript::const_iterator pc1 = script1.begin();
            CScript::const_iterator pc2 = script2.begin();
            loop // Compare scriptPubKey with the template part by part
            {
                bool f1 = script1.GetOp(pc1, opcode1, vch1); // Op from scriptPubKey
                bool f2 = script2.GetOp(pc2, opcode2, vch2); // Op from template
                if (!f1 && !f2) // Both scripts ended, match found
                {
                    reverse(vSolutionRet.begin(), vSolutionRet.end()); // Items were added in reverse
                    return true;
                }
                else if (f1 != f2) break; // One ended, other didn't - no match
                else if (opcode2 == OP_PUBKEY) // Template expects a pubkey
                {
                    if (vch1.size() <= sizeof(uint256)) break; // Basic sanity check on pubkey data
                    vSolutionRet.push_back(make_pair(opcode2, vch1)); // Store pubkey data needed
                }
                else if (opcode2 == OP_PUBKEYHASH) // Template expects a pubkey hash
                {
                    if (vch1.size() != sizeof(uint160)) break; // Check size of hash
                    vSolutionRet.push_back(make_pair(opcode2, vch1)); // Store pubkey hash data needed
                }
                else if (opcode1 != opcode2) break; // Opcodes don't match
            }
        }
        vSolutionRet.clear();
        return false; // No template matched
    }
    ```
    *   It compares the input `scriptPubKey` against a static list of template scripts (`vTemplates`).
    *   If a template matches, it extracts the data parts (like public keys or public key hashes) from the `scriptPubKey` that correspond to `OP_PUBKEY` or `OP_PUBKEYHASH` placeholders in the template.
    *   These extracted parts are returned in `vSolutionRet` as pairs of `(opcodetype, data_value)`. For example, for a P2PKH script, `vSolutionRet` would contain one item: `(OP_PUBKEYHASH, <actual_pubKeyHash_from_scriptPubKey>)`.

2.  `Solver(const CScript& scriptPubKey, uint256 hash, int nHashType, CScript& scriptSigRet)`: This function uses the first `Solver` to understand `scriptPubKey`, then tries to create the corresponding `scriptSig` using available keys in the wallet.
    ```cpp
    // From script.cpp
    bool Solver(const CScript& scriptPubKey, uint256 hash, int nHashType, CScript& scriptSigRet)
    {
        scriptSigRet.clear();
        vector<pair<opcodetype, valtype> > vSolution;
        if (!Solver(scriptPubKey, vSolution)) // Call the first Solver
            return false;

        CRITICAL_BLOCK(cs_mapKeys) // Access wallet keys
        {
            foreach(PAIRTYPE(opcodetype, valtype)& item, vSolution)
            {
                if (item.first == OP_PUBKEY) // Need to provide a signature for this pubkey
                {
                    const valtype& vchPubKey = item.second;
                    if (!mapKeys.count(vchPubKey)) return false; // Check if private key is in wallet
                    if (hash != 0) // hash is 0 if just checking if mine, non-zero if actually signing
                    {
                        vector<unsigned char> vchSig;
                        if (!CKey::Sign(mapKeys[vchPubKey], hash, vchSig)) return false; // Sign the hash
                        vchSig.push_back((unsigned char)nHashType); // Append hash type
                        scriptSigRet << vchSig; // Push signature onto scriptSig
                    }
                }
                else if (item.first == OP_PUBKEYHASH) // Need to provide signature and the corresponding pubkey
                {
                    map<uint160, valtype>::iterator mi = mapPubKeys.find(uint160(item.second));
                    if (mi == mapPubKeys.end()) return false; // Check if pubkey for hash is known
                    const vector<unsigned char>& vchPubKey = (*mi).second;
                    if (!mapKeys.count(vchPubKey)) return false; // Check if private key for pubkey is in wallet
                    if (hash != 0)
                    {
                        vector<unsigned char> vchSig;
                        if (!CKey::Sign(mapKeys[vchPubKey], hash, vchSig)) return false;
                        vchSig.push_back((unsigned char)nHashType);
                        scriptSigRet << vchSig << vchPubKey; // Push signature then pubkey
                    }
                }
            }
        }
        return true;
    }
    ```
    *   It calls the first `Solver` to get the required data elements from `scriptPubKey`.
    *   It then iterates through these required elements:
        *   If `OP_PUBKEY` was identified, it means the `scriptPubKey` contains a public key. The `Solver` looks for the corresponding private key in `mapKeys`. If found and `hash` is non-zero (meaning we need to actually sign), it signs the `hash` and pushes the signature (with `nHashType` appended) onto `scriptSigRet`.
        *   If `OP_PUBKEYHASH` was identified, it means `scriptPubKey` contains a hash of a public key. The `Solver` looks up the actual public key in `mapPubKeys` using the hash. Then, it finds the private key for that public key in `mapKeys`. If found and `hash` is non-zero, it signs, and pushes both the signature and the full public key onto `scriptSigRet`.
    *   If `hash` is 0, the function essentially just checks if the keys needed to solve the `scriptPubKey` are present in the wallet, which is what `IsMine` uses.

### Mermaid Flowchart for `Solver(scriptPubKey, vSolutionRet)`

```mermaid
graph TD
    A[Start Solver(scriptPubKey, vSolutionRet)] --> B[Initialize static vTemplates (P2PK, P2PKH)];
    B --> C{For each template in vTemplates};
    C -- Next Template --> D[vSolutionRet.clear()];
    D --> E[pc1 = scriptPubKey.begin(), pc2 = template.begin()];
    E --> F{Loop: Compare scriptPubKey and template};
    F --> G[GetOp(pc1, opcode1, vch1)];
    F --> H[GetOp(pc2, opcode2, vch2)];
    H --> I{Both GetOp succeeded?};
    I -- No (One or both ended) --> J{Both ended simultaneously (f1=false, f2=false)?};
    J -- Yes (Match!) --> K[Reverse vSolutionRet];
    K --> L[Return True];
    J -- No (Mismatch length) --> M[Break from Loop (try next template)];

    I -- Yes --> N{opcode2 == OP_PUBKEY?};
    N -- Yes --> O[vch1 (data from scriptPubKey) is a pubkey. Store (OP_PUBKEY, vch1) in vSolutionRet];
    O --> F;
    N -- No --> P{opcode2 == OP_PUBKEYHASH?};
    P -- Yes --> Q[vch1 is a pubkey hash. Store (OP_PUBKEYHASH, vch1) in vSolutionRet];
    Q --> F;
    P -- No --> R{opcode1 == opcode2? (Opcodes match?)};
    R -- Yes --> F;
    R -- No (Opcodes differ) --> M;

    M --> C;
    C -- No More Templates --> S[vSolutionRet.clear()];
    S --> T[Return False];
```

## Interaction with Transactions

Scripts are fundamental to transaction validity. Each transaction input (`CTxIn`) contains a `scriptSig` (unlocking script/witness), and each transaction output (`CTxOut`) contains a `scriptPubKey` (locking script/encumbrance).

**Transaction Verification (`CTransaction::ConnectInputs` and `VerifySignature`):**
When a transaction is being connected to the blockchain (e.g., in `CTransaction::ConnectInputs` called from `CBlock::ConnectBlock` or `CTransaction::AcceptTransaction`):

1.  For each input in the transaction being verified (`txTo`):
    *   The previous output being spent (`txFrom.vout[txin.prevout.n]`) is retrieved. This output contains the `scriptPubKey`.
    *   The current input (`txin`) provides the `scriptSig`.
2.  The `VerifySignature` function (from `script.cpp`) is called:
    ```cpp
    // From script.cpp
    bool VerifySignature(const CTransaction& txFrom, const CTransaction& txTo, unsigned int nIn, int nHashType)
    {
        assert(nIn < txTo.vin.size());
        const CTxIn& txin = txTo.vin[nIn];
        if (txin.prevout.n >= txFrom.vout.size())
            return false;
        const CTxOut& txout = txFrom.vout[txin.prevout.n];

        if (txin.prevout.hash != txFrom.GetHash()) // Ensure input links to correct previous tx
            return false;

        // The core evaluation: scriptSig is concatenated with scriptPubKey, separated by OP_CODESEPARATOR (implicitly)
        // In this older codebase, it seems OP_CODESEPARATOR is explicitly added here.
        return EvalScript(txin.scriptSig + CScript(OP_CODESEPARATOR) + txout.scriptPubKey, txTo, nIn, nHashType);
    }
    ```
    *   It first performs basic checks (e.g., input index in range, `prevout.hash` matches `txFrom`).
    *   The crucial step is `EvalScript(txin.scriptSig + CScript(OP_CODESEPARATOR) + txout.scriptPubKey, txTo, nIn, nHashType)`.
        *   The `scriptSig` from the input and the `scriptPubKey` from the output being spent are concatenated (with an `OP_CODESEPARATOR` in between, though modern Bitcoin might handle this slightly differently by just running `scriptSig` then using its resulting stack as input to `scriptPubKey`).
        *   `EvalScript` is then called on this combined script.
        *   The `txTo` (the current transaction) and `nIn` (the current input index) are passed to `EvalScript` because `OP_CHECKSIG` operations need this context to correctly generate the hash of the transaction data that was actually signed.
3.  If `EvalScript` returns `true`, the input is considered valid for that specific `scriptPubKey`. If it returns `false`, the input (and thus the transaction) is invalid.

**`CTransaction::CheckTransaction` (from `main.cpp`):**
This function performs basic, context-free checks on a transaction. It does *not* execute scripts.
```cpp
// From main.cpp
bool CTransaction::CheckTransaction() const
{
    // Basic checks that don't depend on any context
    if (vin.empty() || vout.empty())
        return error("CTransaction::CheckTransaction() : vin or vout empty");

    // Check for negative values
    foreach(const CTxOut& txout, vout)
        if (txout.nValue < 0)
            return error("CTransaction::CheckTransaction() : txout.nValue negative");

    if (IsCoinBase())
    {
        if (vin[0].scriptSig.size() < 2 || vin[0].scriptSig.size() > 100)
            return error("CTransaction::CheckTransaction() : coinbase script size");
    }
    else
    {
        foreach(const CTxIn& txin, vin)
            if (txin.prevout.IsNull()) // Non-coinbase inputs must point to a previous output
                return error("CTransaction::CheckTransaction() : prevout is null");
    }
    return true;
}
```
The actual script execution for signature verification happens deeper, in `CTransaction::ConnectInputs` via `VerifySignature`, which is called when a transaction is being added to a block or accepted into the memory pool if full validation is performed.

The script system ensures that only the entity capable of providing the correct data and signatures (as required by `scriptPubKey`) can spend the funds associated with an output.
