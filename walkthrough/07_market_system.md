# Market System in Bitcoin v0.2.0

Bitcoin v0.2.0 included a unique, decentralized marketplace system. This feature, later removed, allowed users to advertise products, leave reviews for each other, and participate in a novel reputation system based on "atoms." This document explores the data structures and operations of this market system.

*Relevant files: [market.h](https://github.com/jonzou/btc020/blob/ebbdade7/market.h), [market.cpp](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp)*

## Market System Architecture

The market system is primarily built around three data structures:
*   **`CProduct`**: Represents product advertisements.
*   **`CUser`**: Manages user profiles and their reputation "atoms."
*   **`CReview`**: Represents user reviews, linking users and influencing atom flow.

These structures interact to enable product listings, review posting, and a reputation mechanism via atom propagation.

## Core Market Data Structures

### `CUser` Class
*Defined in [market.h Lines 25-64](https://github.com/jonzou/btc020/blob/ebbdade7/market.h#L25-L64)*
The `CUser` class tracks a user's reputation using "atoms" and their connections to other users.

```cpp
class CUser
{
public:
    vector<unsigned short> vAtomsIn;  // Atoms received and absorbed
    vector<unsigned short> vAtomsNew; // New atoms received, pending processing
    vector<unsigned short> vAtomsOut; // Atoms to be propagated to linked users
    vector<uint256> vLinksOut;     // Hashes of users this user has reviewed

    CUser();

    IMPLEMENT_SERIALIZE
    (
        if (!(nType & SER_GETHASH)) // Version not included in hash
            READWRITE(nVersion);
        READWRITE(vAtomsIn);
        READWRITE(vAtomsNew);
        READWRITE(vAtomsOut);
        READWRITE(vLinksOut);
    )

    void SetNull(); // Clears all vectors
    uint256 GetHash() const; // Calculates hash of the user object

    int GetAtomCount() const; // Total atoms (vAtomsIn + vAtomsNew)
    void AddAtom(unsigned short nAtom, bool fOrigin); // Adds an atom and processes flow
};
```
*   **Atom Management**: Atoms represent user reputation. `vAtomsIn` are confirmed atoms, `vAtomsNew` are recently received and yet to be fully processed, and `vAtomsOut` are atoms that this user will propagate to others they've linked to (by reviewing).
*   **`AddAtom()` Method**: Handles the logic for adding new atoms. If an atom is an "origin" atom (e.g., the "zero atom" or one created by the user themselves), it's added to `vAtomsIn` and `vAtomsOut`. Otherwise, it's added to `vAtomsNew`. When `vAtomsNew` reaches `nFlowthroughRate` or `vAtomsOut` is empty, an atom from `vAtomsNew` is selected to flow to `vAtomsOut`, and `vAtomsNew` is merged into `vAtomsIn`.
    *Referenced from [market.cpp Lines 109-141](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L109-L141)*
*   `nFlowthroughRate`: A constant determining how many new atoms trigger propagation (value is 2).
    *Defined in [market.h Line 9](https://github.com/jonzou/btc020/blob/ebbdade7/market.h#L9)*

### `CReview` Class
*Defined in [market.h Lines 72-112](https://github.com/jonzou/btc020/blob/ebbdade7/market.h#L72-L112)*
Represents a review written by one user about another. Reviews are cryptographically signed.

```cpp
class CReview
{
public:
    int nVersion;
    uint256 hashTo; // Hash of the user being reviewed
    map<string, string> mapValue; // Key-value store for review content (e.g., "rating", "comment")
    vector<unsigned char> vchPubKeyFrom; // Public key of the reviewer
    vector<unsigned char> vchSig; // Signature of the review

    // memory only
    unsigned int nTime;  // Time review was processed locally
    int nAtoms;          // Atom count of the reviewer at the time of review (memory only)

    CReview();

    IMPLEMENT_SERIALIZE(...) // Conditional serialization for hashing and disk
    // nVersion, vchSig excluded for GetSigHash()
    // hashTo excluded for SER_DISK (presumably stored via key in CReviewDB)

    uint256 GetHash() const;
    uint256 GetSigHash() const; // Hash of review data for signature verification
    uint256 GetUserHash() const; // Hash of vchPubKeyFrom to identify the reviewer

    bool AcceptReview(); // Validates and processes the review
};
```
*   **Authentication**: `GetSigHash()` provides the hash that is signed. `CKey::Verify()` is used in `AcceptReview()` to check the signature.
*   **`AcceptReview()` Method**:
    1.  Verifies the signature.
    2.  Adds the review to the recipient's list of reviews (persisted in `CReviewDB`).
    3.  Adds `hashTo` (the reviewed user) to the reviewer's `vLinksOut` in their `CUser` object.
    4.  Triggers atom propagation from the reviewer to the reviewed user using `AddAtomsAndPropagate()`.
    *Referenced from [market.cpp Lines 197-231](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L197-L231)*

### `CProduct` Class
*Defined in [market.h Lines 120-171](https://github.com/jonzou/btc020/blob/ebbdade7/market.h#L120-L171)*
Represents a product advertisement in the marketplace.

```cpp
class CProduct
{
public:
    int nVersion;
    CAddress addr; // Network address of the seller
    map<string, string> mapValue; // Basic product info (e.g., "name", "price")
    map<string, string> mapDetails; // Detailed product description
    vector<pair<string, string> > vOrderForm; // Template for ordering
    unsigned int nSequence; // Sequence number for updates
    vector<unsigned char> vchPubKeyFrom; // Seller's public key
    vector<unsigned char> vchSig; // Signature of the product data

    // disk only
    int nAtoms; // Seller's atom count at time of listing (from CUser)

    // memory only
    set<unsigned int> setSources; // IPs from which this product was received

    CProduct();

    IMPLEMENT_SERIALIZE(...) // Conditional serialization
    // mapDetails, vOrderForm, nSequence, vchSig excluded for GetHash() (summary hash)
    // vchSig excluded for GetSigHash()
    // nAtoms only for SER_DISK

    uint256 GetHash() const; // Hash of summary product data
    uint256 GetSigHash() const; // Hash for signature verification (includes more fields than GetHash)
    uint256 GetUserHash() const; // Hash of vchPubKeyFrom (seller's user hash)

    bool CheckSignature();
    bool CheckProduct(); // Validates signature and product summary status
};
```
*   **Product Updates**: `nSequence` allows sellers to update their product listings. Higher sequence numbers replace older ones with the same base hash.
*   **`CheckProduct()` Method**:
    1.  Calls `CheckSignature()` (which uses `CKey::Verify()`).
    2.  Ensures it's a "summary" product (empty `mapDetails` and `vOrderForm` for initial broadcast).
    3.  Retrieves the seller's atom count from `CReviewDB` and stores it in `nAtoms`.
    *Referenced from [market.cpp Lines 237-264](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L237-L264)*

### Global Data Structures for Market
*Defined in [market.h Lines 180-183](https://github.com/jonzou/btc020/blob/ebbdade7/market.h#L180-L183) and [market.cpp Lines 21-27](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L21-L27)*
*   `mapMyProducts`: Stores products created by the local user.
*   `mapProducts`: Global map of all known valid product advertisements, indexed by their hash.
*   `cs_mapProducts`: A `CCriticalSection` to protect `mapProducts` from concurrent access.

## Market Operations

### Product Advertisement System
*   **`AdvertInsert(const CProduct& product)`**:
    1.  Acquires `cs_mapProducts` lock.
    2.  Inserts the product into `mapProducts` or finds the existing one.
    3.  If the new product has a higher `nSequence` than an existing one with the same hash, it replaces the older one.
    *Referenced from [market.cpp Lines 29-56](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L29-L56)*
*   **`AdvertErase(const CProduct& product)`**: Removes a product from `mapProducts`.
    *Referenced from [market.cpp Lines 58-64](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L58-L64)*

### Atom Propagation System
This system is the core of the market's reputation mechanism. Atoms flow from user to user when reviews are created.

*   **`AddAtomsAndPropagate(uint256 hashUserStart, const vector<unsigned short>& vAtoms, bool fOrigin)`**:
    1.  This function implements a breadth-first-like propagation of atoms.
    2.  It starts with `hashUserStart` and the initial `vAtoms`.
    3.  It iteratively processes users:
        *   Reads the current user's `CUser` data from `CReviewDB`.
        *   Calls `user.AddAtom()` for each received atom. `fOrigin` is true only for the initial call, ensuring these atoms are directly added to `vAtomsOut` of `hashUserStart`.
        *   If `user.AddAtom()` results in new atoms being added to `user.vAtomsOut`, these new outgoing atoms are collected.
        *   For each user linked by `user.vLinksOut`, these new outgoing atoms are added to a list for the next propagation round.
        *   Writes the updated `CUser` back to the database.
    4.  The process repeats until no more atoms are propagated.
    *Referenced from [market.cpp Lines 143-190](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L143-L190)*
*   **`Union(T& v1, T& v2)` Template Function**: A utility to merge two sorted vectors (`v1` and `v2`), placing the sorted union into `v1` and returning the count of new elements added to `v1`. Used in `CUser::AddAtom` for managing `vAtomsIn` and `vAtomsNew`.
    *Referenced from [market.cpp Lines 84-107](https://github.com/jonzou/btc020/blob/ebbdade7/market.cpp#L84-L107)*

The market system, with its product listings, user reviews, and atom-based reputation, was an ambitious early feature of Bitcoin. While complex and ultimately removed, it demonstrates early explorations into decentralized applications beyond simple currency transfer.
