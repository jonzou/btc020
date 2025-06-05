# Database and Persistence in Bitcoin v0.2.0

Bitcoin v0.2.0 utilizes Berkeley DB for its data persistence layer. This system is responsible for storing and retrieving all critical data, including the blockchain, user wallets, peer network addresses, and market-related information. The database architecture is designed with a base abstraction class and several specialized classes for different data domains.

*Relevant files: [db.h](https://github.com/jonzou/btc020/blob/ebbdade7/db.h), [db.cpp](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp)*

## Database Architecture Overview

The persistence layer is built around Berkeley DB, using a hierarchical class structure. A base class `CDB` provides common functionality, and specialized classes inherit from it to manage specific types of data.

### Database Class Hierarchy
*   **`CDB`**: The base class providing core Berkeley DB interactions, including read/write operations, transaction management, and cursor support.
    *   *Defined in [db.h Lines 29-250](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L29-L250)*
*   **`CTxDB`**: Manages blockchain data (block index, transaction index).
    *   *Defined in [db.h Lines 259-282](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L259-L282)*
*   **`CWalletDB`**: Manages wallet data (keys, wallet transactions, address book, settings).
    *   *Defined in [db.h Lines 346-429](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L346-L429)*
*   **`CAddrDB`**: Stores known peer addresses for networking.
    *   *Defined in [db.h Lines 327-337](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L327-L337)*
*   **`CReviewDB`**: Stores user profiles and product reviews for the market system.
    *   *Defined in [db.h Lines 288-308](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L288-L308)*
*   **`CMarketDB`**: Stores other market-specific data.
    *   *Defined in [db.h Lines 314-321](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L314-L321)*

### Database File Organization
Each specialized database class typically corresponds to a specific `.dat` file in the Bitcoin data directory:
*   `blkindex.dat`: Blockchain index data, managed by `CTxDB`.
    *Reference from constructor in [db.h Line 262](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L262)*
*   `wallet.dat`: User wallet data, managed by `CWalletDB`.
    *Reference from constructor in [db.h Line 349](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L349)*
*   `addr.dat`: Peer network addresses, managed by `CAddrDB`.
    *Reference from constructor in [db.h Line 330](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L330)*
*   `reviews.dat`: Market system reviews, managed by `CReviewDB`.
    *Reference from constructor in [db.h Line 291](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L291)*
*   `market.dat`: Other market data, managed by `CMarketDB`.
    *Reference from constructor in [db.h Line 317](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L317)*

## Base Database Infrastructure (`CDB`)

The `CDB` class provides the foundational interface to Berkeley DB.

### `CDB` Class Structure
*Defined in [db.h Lines 29-250](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L29-L250)*
```cpp
class CDB
{
protected:
    Db* pdb; // Berkeley DB database handle
    string strFile; // Filename
    vector<DbTxn*> vTxn; // Stack for managing nested transactions
    bool fReadOnly;

    explicit CDB(const char* pszFile, const char* pszMode="r+");
    ~CDB() { Close(); }
public:
    void Close();

protected:
    // Template methods for CRUD operations
    template<typename K, typename T> bool Read(const K& key, T& value);
    template<typename K, typename T> bool Write(const K& key, const T& value, bool fOverwrite=true);
    template<typename K> bool Erase(const K& key);
    template<typename K> bool Exists(const K& key);

    Dbc* GetCursor(); // Get a database cursor
    int ReadAtCursor(Dbc* pcursor, CDataStream& ssKey, CDataStream& ssValue, unsigned int fFlags=DB_NEXT);

    DbTxn* GetTxn(); // Get current transaction context
public:
    bool TxnBegin();   // Start a new transaction
    bool TxnCommit();  // Commit the current transaction
    bool TxnAbort();   // Abort the current transaction

    bool ReadVersion(int& nVersion);
    bool WriteVersion(int nVersion);
};
```
*   **Constructor (`CDB::CDB`)**: Initializes the Berkeley DB environment (`dbenv`) if not already done, opens the specified database file, and increments its usage count in `mapFileUseCount`.
    *Referenced from [db.cpp Lines 43-127](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L43-L127)*
*   **`Close()`**: Aborts any pending transactions, decrements file usage count, and potentially allows `DBFlush` to close the underlying DB file if the count reaches zero.
    *Referenced from [db.cpp Lines 129-139](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L129-L139)*

### Berkeley DB Environment Setup
The global `DbEnv dbenv` object manages the Berkeley DB environment. It's initialized once when the first `CDB` object is constructed.
*Referenced from [db.cpp Lines 55-86](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L55-L86)*
Key configurations applied via `dbenv.set_...` methods:
*   Log directory: `<datadir>/database`
*   Log size: 10MB
*   Lock limits: 10,000 locks/objects
*   Error file: `<datadir>/db.log`
*   Flags: `DB_AUTO_COMMIT`, `DB_THREAD`, `DB_PRIVATE`, `DB_RECOVER` (for crash recovery).

### Data Operations (Templates)
`CDB` uses template methods for type-safe data operations. Serialization is handled via `CDataStream` with `SER_DISK` type.
*   `Read<K,T>(key, value)`: Reads a value associated with a key.
    *Referenced from [db.h Lines 53-73](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L53-L73)*
*   `Write<K,T>(key, value, fOverwrite)`: Writes a key-value pair.
    *Referenced from [db.h Lines 75-103](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L75-L103)*
*   `Erase<K>(key)`: Deletes a key-value pair.
*   `Exists<K>(key)`: Checks if a key exists.

### Transaction Management
`CDB` supports ACID transactions using Berkeley DB's transaction mechanism.
*   `TxnBegin()`: Starts a new transaction. If a transaction is already active, it creates a nested transaction. Pushes the `DbTxn*` onto `vTxn`.
*   `TxnCommit()`: Commits the topmost transaction from `vTxn`.
*   `TxnAbort()`: Aborts the topmost transaction from `vTxn`.
*Referenced from [db.h Lines 206-238](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L206-L238)*

## Specialized Database Classes

### `CTxDB` (Blockchain Data)
*Class definition: [db.h Lines 259-282](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L259-L282)*
*Implementation: [db.cpp Lines 203-428](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L203-L428)*
Handles `blkindex.dat`. Stores:
*   **Transaction Index**: Key: `("tx", tx_hash)`, Value: `CTxIndex` (disk position of tx, spent status of its outputs).
    *   `ReadTxIndex()`, `UpdateTxIndex()`, `AddTxIndex()`, `EraseTxIndex()`, `ContainsTx()`.
*   **Block Index**: Key: `("blockindex", block_hash)`, Value: `CDiskBlockIndex` (block metadata, pointers to prev/next blocks by hash).
    *   `WriteBlockIndex()`, `EraseBlockIndex()`.
*   **Best Chain Hash**: Key: `"hashBestChain"`, Value: `uint256` (hash of the tip of the best chain).
    *   `ReadHashBestChain()`, `WriteHashBestChain()`.
*   **`LoadBlockIndex()`**: Reads all `("blockindex", ...)` entries from the database to reconstruct `mapBlockIndex`, `pindexGenesisBlock`, and `pindexBest` in memory. This is crucial for initializing the node's view of the blockchain.
    *Referenced from [db.cpp Lines 360-428](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L360-L428)*

### `CWalletDB` (Wallet Data)
*Class definition: [db.h Lines 346-429](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L346-L429)*
*Implementation: [db.cpp Lines 537-700](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L537-L700)*
Handles `wallet.dat`. Stores:
*   **Private Keys**: Key: `("key", pubKeyBytes)`, Value: `CPrivKey`.
    *   `ReadKey()`, `WriteKey()`.
*   **Wallet Transactions**: Key: `("tx", tx_hash)`, Value: `CWalletTx` (transaction with wallet-specific metadata).
    *   `ReadTx()`, `WriteTx()`, `EraseTx()`.
*   **Address Book**: Key: `("name", address_string)`, Value: `string` (label for the address).
    *   `ReadName()`, `WriteName()`, `EraseName()`.
*   **Default Key**: Key: `"defaultkey"`, Value: `vector<unsigned char>` (public key).
    *   `ReadDefaultKey()`, `WriteDefaultKey()`.
*   **Settings**: Key: `("setting", setting_name_string)`, Value: varies (e.g., `fGenerateBitcoins`, `nTransactionFee`).
    *   `ReadSetting()`, `WriteSetting()`.
*   **Version**: Key: `"version"`, Value: `int` (wallet database version).
*   **`LoadWallet()`**: Reads all data from `wallet.dat` into memory structures like `mapKeys`, `mapWallet`, `mapAddressBook`, and global settings.
    *Referenced from [db.cpp Lines 537-670](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L537-L670)*

### `CAddrDB` (Peer Addresses)
*Class definition: [db.h Lines 327-337](https://github.com/jonzou/btc020/blob/ebbdade7/db.h#L327-L337)*
*Implementation: [db.cpp Lines 438-507](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L438-L507)*
Handles `addr.dat`. Stores known peer `CAddress` objects.
*   `WriteAddress(const CAddress& addr)`: Writes an address. Key: `("addr", addr.GetKey())`.
*   `LoadAddresses()`: Loads addresses from `addr.dat` and also from a text file `addr.txt` (if present) into `mapAddresses`.
    *Referenced from [db.cpp Lines 446-505](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L446-L505)*

### `CReviewDB` and `CMarketDB`
These handle data for the marketplace features, stored in `reviews.dat` and `market.dat` respectively. They follow similar patterns for reading and writing their specific data types.

## Database Lifecycle and Maintenance

### Initialization (`CDBInit`)
A global static instance `instance_of_cdbinit` of the `CDBInit` class ensures that the Berkeley DB environment `dbenv` is properly closed (`dbenv.close(0)`) upon program termination.
*Defined in [db.cpp Lines 25-40](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L25-L40)*

### Flushing (`DBFlush`, `ThreadFlushWalletDB`)
*   **`DBFlush(bool fShutdown)`**: This function is called to flush log data to the actual data files and manage log file archiving. If `fShutdown` is true, it attempts to remove log archives and close the `dbenv`. It iterates `mapFileUseCount`; if a file's reference count is zero, it's closed, checkpointed, and its log is reset.
    *Referenced from [db.cpp Lines 156-192](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L156-L192)*
*   **`ThreadFlushWalletDB(void* parg)`**: A dedicated thread that periodically flushes `wallet.dat`.
    *Referenced from [db.cpp Lines 702-760](https://github.com/jonzou/btc020/blob/ebbdade7/db.cpp#L702-L760)*
    *   It monitors `nWalletDBUpdated` (incremented by `CWalletDB::Write*` methods).
    *   If changes are detected and a 2-second delay has passed since the last update, and if no other part of the code is using `wallet.dat` (ref count is 0), it flushes the wallet.
    *   Flushing involves: `CloseDb("wallet.dat")`, `dbenv.txn_checkpoint()`, `dbenv.lsn_reset("wallet.dat", 0)`.

This database system provides a robust way to store Bitcoin's critical data, with support for transactions ensuring atomicity and a modular design for different data types.
