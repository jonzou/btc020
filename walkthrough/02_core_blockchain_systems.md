# Core Blockchain Systems in Bitcoin v0.2.0

This document covers the fundamental blockchain processing mechanisms in Bitcoin v0.2.0, including block and transaction validation, chain management, and mining operations. These systems form the heart of the Bitcoin protocol implementation.

## Core Data Structures

The blockchain system is built around several fundamental data structures that represent blocks, transactions, and chain state. These are primarily defined in `main.h`.

*For information about how these structures are processed and validated, see [Block and Transaction Processing](#block-and-transaction-processing). For details about the database layer that persists these structures, see `05_database_and_persistence.md`.*

### Overview of Data Structures

Core data structures can be categorized into:
*   **Transaction Structures**: `CTransaction`, `CTxIn`, `CTxOut`, `CMerkleTx`, `CWalletTx`
*   **Block Structures**: `CBlock`, `CBlockIndex`, `CDiskBlockIndex`
*   **Location and Indexing Structures**: `COutPoint`, `CInPoint`, `CDiskTxPos`, `CTxIndex`, `CBlockLocator`

### Transaction Data Structures

#### Transaction Hierarchy
The transaction structures form an inheritance hierarchy:
*   `CTransaction`: Basic network transaction.
*   `CMerkleTx`: Extends `CTransaction` with block linkage (Merkle branch).
*   `CWalletTx`: Extends `CMerkleTx` with wallet-specific data.

#### Core Transaction Structure (`CTransaction`)
Represents the basic transaction broadcasted on the network.
*Referenced from [main.h Lines 362-616](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L362-L616)*
```cpp
class CTransaction
{
public:
    int nVersion;
    vector<CTxIn> vin;
    vector<CTxOut> vout;
    unsigned int nLockTime;

    CTransaction()
    {
        SetNull();
    }

    IMPLEMENT_SERIALIZE
    (
        READWRITE(this->nVersion);
        nVersion = this->nVersion;
        READWRITE(vin);
        READWRITE(vout);
        READWRITE(nLockTime);
    )
    // ... other methods like GetHash(), IsCoinBase(), GetValueOut(), GetMinFee()
};
```
**Fields**:
*   `nVersion`: Transaction format version.
*   `vin`: Vector of `CTxIn` (transaction inputs).
*   `vout`: Vector of `CTxOut` (transaction outputs).
*   `nLockTime`: Time-based or block-based lock.

#### Transaction Inputs (`CTxIn`) and Outputs (`CTxOut`)
*Referenced from [main.h Lines 198-270 (CTxIn)](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L198-L270) and [main.h Lines 279-353 (CTxOut)](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L279-L353)*

`CTxIn` represents a transaction input:
```cpp
class CTxIn
{
public:
    COutPoint prevout;
    CScript scriptSig;
    unsigned int nSequence;
    // ... constructors and methods
};
```
*   `prevout`: `COutPoint` referencing a previous transaction's output.
*   `scriptSig`: Signature script to satisfy the referenced output.
*   `nSequence`: Sequence number (used for replacement or time-locking).

`CTxOut` represents a transaction output:
```cpp
class CTxOut
{
public:
    int64 nValue;
    CScript scriptPubKey;
    // ... constructors and methods
};
```
*   `nValue`: Value in satoshis.
*   `scriptPubKey`: Public key script that must be satisfied to spend this output.

#### Enhanced Transaction Types (`CMerkleTx`, `CWalletTx`)
`CMerkleTx` extends `CTransaction` by adding blockchain linkage information.
*Referenced from [main.h Lines 625-687](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L625-L687)*
```cpp
class CMerkleTx : public CTransaction
{
public:
    uint256 hashBlock;
    vector<uint256> vMerkleBranch;
    int nIndex; // Position of transaction within block

    // memory only
    mutable bool fMerkleVerified;
    // ... other members and methods like SetMerkleBranch(), GetDepthInMainChain()
};
```

`CWalletTx` extends `CMerkleTx` with data specific to the user's wallet.
*Referenced from [main.h Lines 697-768](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L697-L768)*
```cpp
class CWalletTx : public CMerkleTx
{
public:
    vector<CMerkleTx> vtxPrev; // Supporting transactions
    map<string, string> mapValue; // Key-value metadata (labels, comments)
    vector<pair<string, string> > vOrderForm; // Order form data for payments
    unsigned int fTimeReceivedIsTxTime;
    unsigned int nTimeReceived;  // Local receipt timestamp
    char fFromMe; // Whether transaction originates from this wallet
    char fSpent;
    // ... other members and methods
};
```

### Block Data Structures

#### Block Structure (`CBlock`)
Represents a complete block in the blockchain.
*Referenced from [main.h Lines 845-1042](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L845-L1042)*
```cpp
class CBlock
{
public:
    // header
    int nVersion;
    uint256 hashPrevBlock;
    uint256 hashMerkleRoot;
    unsigned int nTime;
    unsigned int nBits;
    unsigned int nNonce;

    // network and disk
    vector<CTransaction> vtx;

    // memory only
    mutable vector<uint256> vMerkleTree;

    CBlock()
    {
        SetNull();
    }
    // ... methods like GetHash(), BuildMerkleTree(), ReadFromDisk(), WriteToDisk()
};
```
**Header Fields**:
*   `nVersion`: Block format version.
*   `hashPrevBlock`: Hash of the previous block in the chain.
*   `hashMerkleRoot`: Root of the Merkle tree of transactions in this block.
*   `nTime`: Block timestamp.
*   `nBits`: Proof-of-work difficulty target.
*   `nNonce`: Nonce used in proof-of-work.
**Data Fields**:
*   `vtx`: Vector of `CTransaction` objects included in this block.

#### Block Indexing (`CBlockIndex`, `CDiskBlockIndex`)
`CBlockIndex` maintains the blockchain structure in memory, forming a tree.
*Referenced from [main.h Lines 1057-1174](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L1057-L1174)*
```cpp
class CBlockIndex
{
public:
    const uint256* phashBlock; // Pointer to block hash (memory address of mapBlockIndex key)
    CBlockIndex* pprev;    // Previous block in chain
    CBlockIndex* pnext;    // Next block in main chain (longest chain)
    unsigned int nFile;    // File number containing block data on disk
    unsigned int nBlockPos;// Position of block data within the file
    int nHeight;      // Block height in chain

    // block header (duplicated for quick access)
    int nVersion;
    uint256 hashMerkleRoot;
    unsigned int nTime;
    unsigned int nBits;
    unsigned int nNonce;
    // ... constructors and methods like GetBlockHash(), IsInMainChain(), GetMedianTimePast()
};
```
`CDiskBlockIndex` is a serializable version of `CBlockIndex` for disk storage, using hashes instead of pointers for `pprev` and `pnext`.
*Referenced from [main.h Lines 1181-1246](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L1181-L1246)*
```cpp
class CDiskBlockIndex : public CBlockIndex
{
public:
    uint256 hashPrev;
    uint256 hashNext;
    // ... constructors and serialization
};
```

### Location and Reference Structures
These structures help locate and reference parts of blocks and transactions.

*   `COutPoint` ([main.h Lines 152-188](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L152-L188)): Uniquely identifies a transaction output using `hash` (of the transaction) and `n` (index of the output).
    ```cpp
    class COutPoint
    {
    public:
        uint256 hash;
        unsigned int n;
        // ... constructors, methods, serialization
    };
    ```
*   `CInPoint` ([main.h Lines 137-147](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L137-L147)): References a transaction input using `ptx` (pointer to `CTransaction`) and `n` (index of input).
*   `CDiskTxPos` ([main.h Lines 85-132](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L85-L132)): Specifies disk location of a transaction (`nFile`, `nBlockPos`, `nTxPos`).
*   `CTxIndex` ([main.h Lines 778-828](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L778-L828)): Tracks transaction locations and spending status (`pos`, `vSpent`).
*   `CBlockLocator` ([main.h Lines 1260-1366](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L1260-L1366)): Efficiently describes a position in the blockchain using a vector of block hashes at exponentially increasing intervals (`vHave`).

### Constants and Global State
Defined in `main.h` and `main.cpp`.

**System Constants** from `main.h`:
*Referenced from [main.h Lines 17-22](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L17-L22)*
```cpp
static const unsigned int MAX_SIZE = 0x02000000; // Max serialized block/tx size
static const int64 COIN = 100000000;         // Satoshis per bitcoin
static const int64 CENT = 1000000;           // Satoshis per cent
static const int COINBASE_MATURITY = 100;    // Blocks before coinbase can be spent
static const CBigNum bnProofOfWorkLimit(~uint256(0) >> 32); // Max proof-of-work target
```

**Global Blockchain State Variables** (defined in `main.cpp`, declared in `main.h`):
*Referenced from [main.cpp Lines 16-38](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L16-L38) and [main.h Lines 29-38](https://github.com/jonzou/btc020/blob/ebbdade7/main.h#L29-L38)*
```cpp
// In main.cpp
map<uint256, CBlockIndex*> mapBlockIndex; // Index of all known blocks by hash
const uint256 hashGenesisBlock("0x000000000019d6689c085ae165831e934ff763ae46a2a6c172b3f1b60a8ce26f");
CBlockIndex* pindexGenesisBlock = NULL;   // Pointer to genesis block index
int nBestHeight = -1;                     // Height of the current best chain
uint256 hashBestChain = 0;                // Hash of the current best chain tip
CBlockIndex* pindexBest = NULL;           // Pointer to the best (longest) chain tip

map<uint256, CTransaction> mapTransactions; // Memory pool of unconfirmed transactions (cs_mapTransactions)
map<uint256, CWalletTx> mapWallet;          // Local wallet transactions (cs_mapWallet)
// ... and others like mapOrphanBlocks, mapOrphanTransactions
```
These variables are protected by critical sections (e.g., `cs_main`, `cs_mapTransactions`, `cs_mapWallet`) for thread safety.

## Block and Transaction Processing

This section covers the core mechanisms for validating and processing blocks and transactions, primarily handled in `main.cpp`.

### Global State Management (Processing Perspective)
Key global data structures used in processing:
*   `mapTransactions`: Memory pool of unconfirmed transactions.
    *Referenced from [main.cpp Line 18](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L18)*
*   `mapBlockIndex`: Index of all known blocks.
    *Referenced from [main.cpp Line 23](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L23)*
*   `mapOrphanBlocks`: Orphaned blocks awaiting parent blocks.
    *Referenced from [main.cpp Line 30](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L30)*
*   `mapOrphanTransactions`: Orphaned transactions awaiting input transactions.
    *Referenced from [main.cpp Line 33](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L33)*
*   `pindexBest`: Pointer to the tip of the best (longest) chain.
    *Referenced from [main.cpp Line 28](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L28)*

### Transaction Processing Pipeline

#### Transaction Acceptance (`AcceptTransaction`)
The `AcceptTransaction()` function in `main.cpp` is central to validating new transactions.
*Referenced from [main.cpp Lines 442-518](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L442-L518)*
```cpp
bool CTransaction::AcceptTransaction(CTxDB& txdb, bool fCheckInputs, bool* pfMissingInputs)
{
    // ... (initial checks: coinbase, CheckTransaction, nLockTime)

    // Do we already have it? (mapTransactions, txdb)
    // ...

    // Check for conflicts with in-memory transactions (mapNextTx)
    // ...

    // Check against previous transactions (ConnectInputs)
    // ... (calls ConnectInputs)

    // Store transaction in memory (AddToMemoryPool)
    // ...

    // printf("AcceptTransaction(): accepted %s\n", hash.ToString().substr(0,6).c_str());
    return true;
}
```
**Steps**:
1.  **Basic Validation**: Ensures not coinbase, calls `CheckTransaction()`, validates `nLockTime`.
2.  **Duplicate Detection**: Checks `mapTransactions` and the database (`txdb.ContainsTx(hash)`).
3.  **Input Conflict Resolution**: Checks `mapNextTx` for spending conflicts; allows replacement by newer versions.
4.  **Input Validation**: Calls `ConnectInputs()` to verify all inputs, calculate fees, and check signatures.
5.  **Memory Pool Integration**: If valid, adds to `mapTransactions` via `AddToMemoryPool()`.

#### Input Connection and Validation (`ConnectInputs`)
The `ConnectInputs()` function validates that a transaction's inputs are legitimate and spendable.
*Referenced from [main.cpp Lines 812-910](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L812-L910)*
```cpp
bool CTransaction::ConnectInputs(CTxDB& txdb, map<uint256, CTxIndex>& mapTestPool, CDiskTxPos posThisTx, int nHeight, int64& nFees, bool fBlock, bool fMiner, int64 nMinFee)
{
    if (!IsCoinBase())
    {
        // ... (loop through vin)
        //      Read TxIndex for prevout (from mapTestPool or txdb)
        //      Read prev transaction (txPrev) (from mapTransactions or disk via txindex.pos)
        //      Maturity check for coinbase inputs
        //      VerifySignature(txPrev, *this, i)
        //      Check for double spends (txindex.vSpent[prevout.n].IsNull())
        //      Mark outpoints as spent (txindex.vSpent[prevout.n] = posThisTx)
        //      Update txdb or mapTestPool
        //      Accumulate nValueIn
        // ...
        // Tally transaction fees (nValueIn - GetValueOut())
    }
    // Add transaction to disk index (txdb.AddTxIndex) or test pool (mapTestPool)
    return true;
}
```

#### Orphan Transaction Management
Transactions arriving before their inputs are stored as orphans.
*   `AddOrphanTx()`: Stores orphan tx and indexes by required input hashes.
    *Referenced from [main.cpp Lines 184-194](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L184-L194)*
*   `EraseOrphanTx()`: Removes processed orphans.
    *Referenced from [main.cpp Lines 196-216](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L196-L216)*
When new transactions or blocks arrive, the system checks `mapOrphanTransactionsByPrev` to find and process dependent orphans. (e.g. in `ProcessMessage` "tx" handler, [main.cpp Lines 2009-2034](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L2009-L2034))

### Block Processing Pipeline
The `ProcessBlock` function in `main.cpp` orchestrates block validation and acceptance.
*Referenced from [main.cpp Lines 1292-1350](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1292-L1350)*
```cpp
bool ProcessBlock(CNode* pfrom, CBlock* pblock)
{
    // Check for duplicate (mapBlockIndex, mapOrphanBlocks)
    // ...

    // Preliminary checks (pblock->CheckBlock())
    // ...

    // If prev block missing, add to orphans (mapOrphanBlocks, mapOrphanBlocksByPrev)
    // ...

    // Store to disk and add to block index (pblock->AcceptBlock())
    // ... (this calls AddToBlockIndex, which may call ConnectBlock or Reorganize)

    // Recursively process any orphan blocks that depended on this one
    // ...
    return true;
}
```

#### Block Validation (`CheckBlock`, `AcceptBlock`)
`CheckBlock()` performs context-independent validation:
*Referenced from [main.cpp Lines 1202-1238](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1202-L1238)*
*   Size limits, timestamp validation.
*   Coinbase transaction rules (first, only one).
*   `CheckTransaction()` for all contained transactions.
*   Proof-of-work verification (`GetHash() <= CBigNum().SetCompact(nBits).getuint256()`).
*   Merkle root validation (`hashMerkleRoot == BuildMerkleTree()`).

`AcceptBlock()` performs context-dependent validation and integration:
*Referenced from [main.cpp Lines 1240-1290](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1240-L1290)*
*   Checks against previous block (timestamp, `pindexPrev`).
*   Transaction finality checks.
*   Proof-of-work difficulty adjustment (`nBits == GetNextWorkRequired(pindexPrev)`).
*   Writes block to disk (`WriteToDisk()`).
*   Calls `AddToBlockIndex()` to integrate into the block index tree.

#### Chain Reorganization (`Reorganize`)
If a new block results in a longer valid chain, `Reorganize()` switches to it.
*Referenced from [main.cpp Lines 1014-1109](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1014-L1109)*
**Process**:
1.  Find the fork point between the current best chain and the new longer chain.
2.  Disconnect blocks on the shorter (current best) branch back to the fork point (`DisconnectBlock()`). Transactions from these blocks are queued for re-adding to the memory pool.
3.  Connect blocks on the new longer branch from the fork point (`ConnectBlock()`). Transactions from these blocks are removed from the memory pool.
4.  Update `pindexBest` and `hashBestChain`.
5.  Database transactions (`CTxDB::TxnBegin()`, `TxnCommit()`, `TxnAbort()`) ensure atomicity.

#### Block Connection/Disconnection (`ConnectBlock`, `DisconnectBlock`)
`ConnectBlock()` integrates a block into the active chain:
*Referenced from [main.cpp Lines 977-1010](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L977-L1010)*
1.  Calls `ConnectInputs()` for each transaction to validate and mark spent outputs.
2.  Validates coinbase reward.
3.  Updates block index chain pointers (`pprev->pnext = pindex`).
4.  Adds relevant transactions to the wallet (`AddToWalletIfMine()`).

`DisconnectBlock()` reverses these operations:
*Referenced from [main.cpp Lines 958-975](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L958-L975)*
1.  Calls `DisconnectInputs()` for each transaction.
2.  Updates block index pointers (`pindex->pprev->pnext = NULL`).

### Mining System (`BitcoinMiner`)
Creates new blocks via proof-of-work.
*Referenced from [main.cpp Lines 2403-2605](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L2403-L2605)*
**Process in `BitcoinMiner()` loop**:
1.  **Transaction Collection**: Iterates `mapTransactions` to select valid transactions. `ConnectInputs()` is used with `fMiner=true` to validate and calculate fees, using a temporary `mapTestPool`.
2.  **Coinbase Transaction**: Creates a new coinbase transaction, paying the block reward + fees to a new key. `bnExtraNonce` is incremented for each attempt.
3.  **Block Assembly**: Constructs block header (`pblock->nVersion`, `pblock->hashPrevBlock`, `pblock->hashMerkleRoot`, `pblock->nTime`, `pblock->nBits`).
4.  **Proof-of-Work**: Repeatedly hashes the block header while incrementing `pblock->nNonce` until `block.GetHash() <= target`.
    *   Optimized SHA256 hashing is performed by `BlockSHA256()`.
5.  **Block Submission**: If a solution is found, calls `ProcessBlock()` to integrate the new block.

**Key mining-related functions**:
*   `GetNextWorkRequired()`: Calculates difficulty adjustment.
    *Referenced from [main.cpp Lines 725-769](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L725-L769)*
*   `GetBlockValue()`: Calculates block reward (subsidy + fees).
    *Referenced from [main.cpp Lines 715-723](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L715-L723)*

### Network Integration
Block and transaction processing is tightly coupled with network messages handled in `ProcessMessage()`.
*Referenced from [main.cpp Lines 1759-2178](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1759-L2178)*

**Transaction Messages**:
*   `"tx"`: Receives a transaction. Calls `AcceptTransaction()`, handles orphans.
    *Referenced from [main.cpp Lines 1991-2041](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1991-L2041)*
*   `"getdata"` (for TX): Serves transaction data from `mapTransactions` or relay cache.

**Block Messages**:
*   `"block"`: Receives a block. Calls `ProcessBlock()`.
    *Referenced from [main.cpp Lines 2062-2077](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L2062-L2077)*
*   `"getblocks"`: Responds with block inventory using `CBlockLocator`.
*   `"inv"`: Tracks inventory and requests missing items via `AskFor()`.

### Error Handling and Edge Cases
*   Orphan limits (transactions, blocks) to prevent memory exhaustion.
*   Reorganization failures can trigger chain rollbacks.
*   Database consistency is maintained using `CTxDB` transactions.
    *   e.g. `Reorganize` failure handling [main.cpp Lines 1066-1078](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L1066-L1078)
*   Mining interruptions (shutdown, new block found by others).
    *   e.g. `BitcoinMiner` checks [main.cpp Lines 2574-2600](https://github.com/jonzou/btc020/blob/ebbdade7/main.cpp#L2574-L2600)
