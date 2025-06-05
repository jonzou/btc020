# Database and Persistence in Bitcoin

This document outlines the database architecture and key persistence mechanisms used in this version of the Bitcoin client, primarily leveraging Berkeley DB.

## Database Architecture

The database design is centered around the `CDB` class (in `db.h` and `db.cpp`), which serves as a wrapper for Berkeley DB operations. Multiple database files are used to store different types of data.

**Key Characteristics:**

*   **Berkeley DB (BDB)**: The system uses Berkeley DB as its underlying database engine. BDB is an embedded database library providing key-value storage.
*   **Environment (`DbEnv dbenv`)**: A single Berkeley DB environment (`dbenv`) is initialized and shared for managing transactions, logging, and recovery across multiple database files. This is set up in the `CDB` constructor if not already initialized.
    ```cpp
    // From db.cpp - CDB Constructor (Environment Setup)
    CRITICAL_BLOCK(cs_db)
    {
        if (!fDbEnvInit)
        {
            // ... (setup data/log directories) ...
            dbenv.set_lg_dir(strLogDir.c_str());
            dbenv.set_lg_max(10000000);
            dbenv.set_lk_max_locks(10000);
            // ... (other settings) ...
            ret = dbenv.open(strDataDir.c_str(),
                             DB_CREATE     |
                             DB_INIT_LOCK  |
                             DB_INIT_LOG   |
                             DB_INIT_MPOOL |
                             DB_INIT_TXN   |
                             DB_THREAD     |
                             DB_PRIVATE    |
                             DB_RECOVER,
                             S_IRUSR | S_IWUSR);
            // ... (error handling) ...
            fDbEnvInit = true;
        }
        // ... (database file opening logic) ...
    }
    ```
*   **File-based Databases**: Different types of data are stored in separate `.dat` files, each managed as a distinct Berkeley DB database within the shared environment. Examples include:
    *   `blkindex.dat`: Stores the block index (`CTxDB`).
    *   `wallet.dat`: Stores wallet-related data like transactions, keys, and settings (`CWalletDB`).
    *   `addr.dat`: Stores known peer addresses (`CAddrDB`).
    *   `reviews.dat`: Appears to store user reviews (`CReviewDB`).
    *   `market.dat`: Purpose less clear from headers, likely market-related data (`CMarketDB`).
*   **Key-Value Storage**: Data is stored as key-value pairs. Keys are often composite, typically a string prefix indicating the type of data followed by a unique identifier (e.g., a hash).
    *   Example key for a transaction index: `make_pair(string("tx"), hash)`
    *   Example key for a block index: `make_pair(string("blockindex"), blockindex.GetBlockHash())`
    *   Example key for a wallet transaction: `make_pair(string("tx"), hash)`
*   **Serialization**: Custom serialization (`CDataStream`) is used to convert objects into byte streams for storage in BDB and to deserialize them upon retrieval. The `SER_DISK` type is used for database serialization.
*   **Transactions**: BDB transactions (`DbTxn`) are used to ensure atomicity for database operations. The `CDB` class provides `TxnBegin()`, `TxnCommit()`, and `TxnAbort()` methods.
*   **Caching/Use Counting**: `mapFileUseCount` and `mapDb` are used to manage open database handles and their usage counts, allowing for flushing and closing of unused database files.

**Data Stored:**

*   **Block Index (`blkindex.dat` via `CTxDB`)**:
    *   Stores `CDiskBlockIndex` objects, which contain block header information, pointers to previous/next blocks in the chain (by hash), file number and position on disk where the full block is stored, and block height.
    *   Key: `("blockindex", blockHash)`
    *   Also stores `hashBestChain` (the hash of the tip of the best known valid chain).
*   **Transaction Index (`blkindex.dat` via `CTxDB`)**:
    *   Stores `CTxIndex` objects, which contain the disk position (`CDiskTxPos`) of a transaction and information about which transactions spend its outputs.
    *   Key: `("tx", txHash)`
*   **Wallet (`wallet.dat` via `CWalletDB`)**:
    *   **Wallet Transactions (`CWalletTx`)**: Detailed information about transactions relevant to the user's wallet, including inputs, outputs, Merkle branch, time received, etc.
        *   Key: `("tx", txHash)`
    *   **Private/Public Keys**: Pairs of public keys and their corresponding encrypted private keys.
        *   Key: `("key", vchPubKey)`
    *   **Default Key**: The user's primary key.
        *   Key: `"defaultkey"`
    *   **Address Book (`mapAddressBook`)**: Mapping of Bitcoin addresses to names/labels.
        *   Key: `("name", strAddress)`
    *   **Settings**: Various application settings like transaction fees, display preferences, proxy settings.
        *   Key: `("setting", strKey)`
    *   **Version**: Database version.
        *   Key: `"version"`
*   **Peer Addresses (`addr.dat` via `CAddrDB`)**:
    *   Stores `CAddress` objects representing known network peers, including their IP, port, services, and last seen time.
    *   Key: `("addr", addr.GetKey())` (where `GetKey()` likely serializes IP and port).
*   **Reviews (`reviews.dat` via `CReviewDB`)**:
    *   Stores `CUser` objects and `vector<CReview>` associated with a hash.
    *   Keys: `("user", hash)` and `("reviews", hash)`.

## Database Implementation

The `CDB` class provides template methods `Read`, `Write`, and `Erase` for generic key-value operations, and `GetCursor`/`ReadAtCursor` for iterator-style access. Specialized classes like `CTxDB` and `CWalletDB` inherit from `CDB` and provide type-specific methods.

### Writing a Block Index

Full blocks themselves are written to flat files (e.g., `blk0001.dat`). The database (`blkindex.dat` managed by `CTxDB`) stores the *index* to these blocks. The relevant operation is `CTxDB::WriteBlockIndex`.

```cpp
// From db.h
class CTxDB : public CDB
{
public:
    // ...
    bool WriteBlockIndex(const CDiskBlockIndex& blockindex);
    // ...
};

// From db.cpp
bool CTxDB::WriteBlockIndex(const CDiskBlockIndex& blockindex)
{
    return Write(make_pair(string("blockindex"), blockindex.GetBlockHash()), blockindex);
}

// Generic Write method in CDB (db.h)
template<typename K, typename T>
bool Write(const K& key, const T& value, bool fOverwrite=true)
{
    if (!pdb)
        return false;
    if (fReadOnly)
        assert(("Write called on database in read-only mode", false));

    // Key
    CDataStream ssKey(SER_DISK);
    ssKey.reserve(1000);
    ssKey << key; // e.g., pair<string, uint256>("blockindex", blockHash)
    Dbt datKey(&ssKey[0], ssKey.size());

    // Value
    CDataStream ssValue(SER_DISK);
    ssValue.reserve(10000);
    ssValue << value; // The CDiskBlockIndex object
    Dbt datValue(&ssValue[0], ssValue.size());

    // Write
    int ret = pdb->put(GetTxn(), &datKey, &datValue, (fOverwrite ? 0 : DB_NOOVERWRITE));

    // Clear memory ...
    return (ret == 0);
}
```
**Process:**
1.  `CTxDB::WriteBlockIndex` is called with a `CDiskBlockIndex` object.
2.  It constructs a key: a pair consisting of the string "blockindex" and the block's hash (`blockindex.GetBlockHash()`).
3.  It calls the generic `CDB::Write` method.
4.  Inside `CDB::Write`:
    *   The key is serialized into a `CDataStream` (`ssKey`), then placed into a Berkeley DB `Dbt` (Database Thang) object `datKey`.
    *   The `CDiskBlockIndex` object (the value) is serialized into `ssValue`, then placed into `datValue`.
    *   `pdb->put(GetTxn(), &datKey, &datValue, ...)` is called to store the key-value pair in the BDB database. `GetTxn()` retrieves the current BDB transaction if one is active.

### Writing/Reading Wallet Transactions

Wallet transactions (`CWalletTx`) are managed by `CWalletDB` and stored in `wallet.dat`.

**Writing a Wallet Transaction (`CWalletDB::WriteTx`)**

```cpp
// From db.h
class CWalletDB : public CDB
{
public:
    // ...
    bool WriteTx(uint256 hash, const CWalletTx& wtx);
    // ...
};

// From db.cpp
bool CWalletDB::WriteTx(uint256 hash, const CWalletTx& wtx)
{
    nWalletDBUpdated++; // Global counter to track wallet updates
    return Write(make_pair(string("tx"), hash), wtx);
}
```
**Process:**
1.  `CWalletDB::WriteTx` is called with the transaction's hash and the `CWalletTx` object.
2.  `nWalletDBUpdated` is incremented. This flag is used by `ThreadFlushWalletDB` to determine when to flush the wallet database.
3.  It constructs a key: a pair consisting of the string "tx" and the transaction's hash.
4.  It calls the generic `CDB::Write` method (similar to `WriteBlockIndex`) to serialize the key and the `CWalletTx` object and store them in `wallet.dat`.

**Reading a Wallet Transaction (`CWalletDB::ReadTx`)**

```cpp
// From db.h
class CWalletDB : public CDB
{
public:
    bool ReadTx(uint256 hash, CWalletTx& wtx);
    // ...
};

// From db.cpp
bool CWalletDB::ReadTx(uint256 hash, CWalletTx& wtx)
{
    return Read(make_pair(string("tx"), hash), wtx);
}

// Generic Read method in CDB (db.h)
template<typename K, typename T>
bool Read(const K& key, T& value)
{
    if (!pdb)
        return false;

    // Key
    CDataStream ssKey(SER_DISK);
    ssKey.reserve(1000);
    ssKey << key; // e.g., pair<string, uint256>("tx", txHash)
    Dbt datKey(&ssKey[0], ssKey.size());

    // Read
    Dbt datValue;
    datValue.set_flags(DB_DBT_MALLOC); // BDB allocates memory for the value
    int ret = pdb->get(GetTxn(), &datKey, &datValue, 0);
    // ... (clear key memory) ...
    if (datValue.get_data() == NULL)
        return false;

    // Unserialize value
    CDataStream ssValue((char*)datValue.get_data(), (char*)datValue.get_data() + datValue.get_size(), SER_DISK);
    ssValue >> value; // Deserialize into the CWalletTx object

    // Clear and free memory
    // ... (clear and free datValue.get_data()) ...
    return (ret == 0);
}
```
**Process:**
1.  `CWalletDB::ReadTx` is called with the transaction hash and a reference to a `CWalletTx` object to be filled.
2.  It constructs the key: `make_pair(string("tx"), hash)`.
3.  It calls the generic `CDB::Read` method.
4.  Inside `CDB::Read`:
    *   The key is serialized into `ssKey` and then into `datKey`.
    *   `pdb->get(GetTxn(), &datKey, &datValue, 0)` retrieves the data associated with the key. BDB allocates memory for `datValue`.
    *   The retrieved byte stream from `datValue` is used to construct a `CDataStream` (`ssValue`).
    *   The `CWalletTx` object (`value`) is deserialized from `ssValue`.
    *   The memory allocated by BDB for `datValue` is freed.

### Flowchart: Writing a Block Index (`CTxDB::WriteBlockIndex`)

```mermaid
graph TD
    A[Start CTxDB::WriteBlockIndex(diskindex)] --> B[Create Key: pair("blockindex", diskindex.GetBlockHash())];
    B --> C[Call CDB::Write(key, diskindex)];
    C --> D{PDB (Db handle) valid?};
    D -- No --> E[Return false];
    D -- Yes --> F{Read-only mode?};
    F -- Yes --> G[Assert/Error];
    F -- No --> H[Serialize Key to CDataStream (ssKey)];
    H --> I[Create Dbt datKey from ssKey];
    I --> J[Serialize CDiskBlockIndex (value) to CDataStream (ssValue)];
    J --> K[Create Dbt datValue from ssValue];
    K --> L[Call pdb->put(GetTxn(), datKey, datValue, flags)];
    L -- Success (ret == 0) --> M[Clear key/value memory];
    M --> N[Return true];
    L -- Failure --> O[Clear key/value memory];
    O --> P[Return false];
    E --> Z[End];
    G --> Z;
    N --> Z;
    P --> Z;
```

### Flowchart: Writing a Wallet Transaction (`CWalletDB::WriteTx`)

```mermaid
graph TD
    A[Start CWalletDB::WriteTx(hash, wtx)] --> B[Increment nWalletDBUpdated];
    B --> C[Create Key: pair("tx", hash)];
    C --> D[Call CDB::Write(key, wtx)];
    D --> E{PDB (Db handle) valid?};
    E -- No --> F[Return false];
    E -- Yes --> G{Read-only mode?};
    G -- Yes --> H[Assert/Error];
    G -- No --> I[Serialize Key to CDataStream (ssKey)];
    I --> J[Create Dbt datKey from ssKey];
    J --> K[Serialize CWalletTx (value) to CDataStream (ssValue)];
    K --> L[Create Dbt datValue from ssValue];
    L --> M[Call pdb->put(GetTxn(), datKey, datValue, flags)];
    M -- Success (ret == 0) --> N[Clear key/value memory];
    N --> O[Return true];
    M -- Failure --> P[Clear key/value memory];
    P --> Q[Return false];
    F --> Z[End];
    H --> Z;
    O --> Z;
    Q --> Z;
```
