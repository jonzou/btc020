# Networking in Bitcoin v0.2.0

The networking layer in Bitcoin v0.2.0 is responsible for peer-to-peer (P2P) communication between nodes. This includes discovering other nodes, managing connections, and exchanging blockchain data (blocks and transactions) as well as market-related information. It uses a custom TCP-based protocol for direct P2P communication and an innovative IRC-based mechanism for initial peer discovery.

## Network Architecture Overview

The networking system is multi-threaded, with distinct responsibilities for different threads:
1.  **Socket Handler Thread (`ThreadSocketHandler2`)**: Manages all active socket I/O (reading, writing, accepting new connections) using `select()`.
2.  **Connections Thread (`ThreadOpenConnections2`)**: Handles outgoing connection establishment and peer selection.
3.  **Message Handler Thread (`ThreadMessageHandler2`)**: Processes received messages and sends queued messages.
4.  **IRC Seed Thread (`ThreadIRCSeed`)**: Connects to an IRC network to discover initial peer addresses.

*Relevant files: [net.h](https://github.com/jonzou/btc020/blob/ebbdade7/net.h), [net.cpp](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp), [irc.h](https://github.com/jonzou/btc020/blob/ebbdade7/irc.h), [irc.cpp](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp)*

## P2P Protocol and Connection Management (`net.h`, `net.cpp`)

### Core Data Structures

#### `CNode` Class
*Defined in [net.h Lines 479-915](https://github.com/jonzou/btc020/blob/ebbdade7/net.h#L479-L915)*
Represents a connection to a peer node.
```cpp
class CNode
{
public:
    // socket
    uint64 nServices;       // Services offered by this node
    SOCKET hSocket;         // The socket for this connection
    CDataStream vSend;      // Data buffer for sending
    CDataStream vRecv;      // Data buffer for receiving
    CCriticalSection cs_vSend;
    CCriticalSection cs_vRecv;
    int64 nLastSend;
    int64 nLastRecv;
    int64 nTimeConnected;
    CAddress addr;          // Address of the peer
    int nVersion;           // Peer's protocol version
    bool fClient;           // True if peer is a client (doesn't relay blocks)
    bool fInbound;          // True if this is an incoming connection
    bool fNetworkNode;      // True if this node is part of the P2P network
    bool fSuccessfullyConnected;
    bool fDisconnect;       // Flag to mark node for disconnection
protected:
    int nRefCount;
public:
    // ... other members for request tracking, inventory, subscriptions ...

    CNode(SOCKET hSocketIn, CAddress addrIn, bool fInboundIn=false);
    ~CNode();

    // ... methods for reference counting, message pushing, inventory management, subscriptions ...
    void PushMessage(const char* pszCommand, ...); // Variadic template for sending messages
    void AskFor(const CInv& inv); // Request data (block or tx)
    void PushGetBlocks(CBlockIndex* pindexBegin, uint256 hashEnd);
    void CloseSocketDisconnect();
};
```
Each `CNode` instance maintains send/receive buffers (`vSend`, `vRecv`), the peer's address (`addr`), protocol version, and various state flags.

#### `CMessageHeader`
*Defined in [net.h Lines 54-121](https://github.com/jonzou/btc020/blob/ebbdade7/net.h#L54-L121)*
All P2P messages start with a standard header.
```cpp
class CMessageHeader
{
public:
    enum { COMMAND_SIZE=12 };
    char pchMessageStart[sizeof(::pchMessageStart)]; // Magic bytes (0xf9beb4d9)
    char pchCommand[COMMAND_SIZE];                 // Command string (e.g., "version", "block")
    unsigned int nMessageSize;                     // Size of the payload

    CMessageHeader();
    CMessageHeader(const char* pszCommand, unsigned int nMessageSizeIn);
    // ... IsValid() method ...
};
```
The magic bytes `pchMessageStart` identify the network (mainnet in this case).

#### `CAddress`
*Defined in [net.h Lines 130-332](https://github.com/jonzou/btc020/blob/ebbdade7/net.h#L130-L332)*
Represents a network address, capable of storing IPv4 and (rudimentarily) IPv6 addresses.
```cpp
class CAddress
{
public:
    uint64 nServices;       // Services offered by the address
    unsigned char pchReserved[12]; // Reserved for IPv6, typically holds pchIPv4 mapping
    unsigned int ip;        // IPv4 address (network byte order)
    unsigned short port;    // Port (network byte order)

    // disk only
    unsigned int nTime;     // Last time this address was seen

    // memory only
    unsigned int nLastTry;  // Last time a connection attempt was made to this address

    CAddress();
    // ... constructors, methods like IsRoutable(), IsValid(), ToString() ...
};
```
`pchIPv4` ([net.h Line 126](https://github.com/jonzou/btc020/blob/ebbdade7/net.h#L126)) is used to map IPv4 addresses into the `pchReserved` field for IPv6 compatibility.

### Connection Lifecycle

1.  **Opening Connections (`OpenNetworkConnection`, `ConnectSocket`)**:
    *   `ThreadOpenConnections2` ([net.cpp Lines 837-965](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L837-L965)) selects addresses to connect to from `mapAddresses`.
    *   `OpenNetworkConnection` ([net.cpp Lines 968-1007](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L968-L1007)) initiates an outgoing connection.
    *   `ConnectSocket` ([net.cpp Lines 60-115](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L60-L115)) creates a TCP socket and connects. It includes SOCKS4 proxy support.
        *   Proxy usage is determined by `fUseProxy` and `addrProxy` ([net.cpp Lines 73-76](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L73-L76)).
    *   Once connected, a new `CNode` object is created and added to `vNodes`.
    *   Initial messages like "version" and "getaddr" are sent.

2.  **Accepting Connections (`ThreadSocketHandler2`)**:
    *   The main listening socket `hListenSocket` is checked by `select()`.
    *   If a new connection is pending, `accept()` is called.
    *   A new `CNode` object is created for the inbound connection.
    *Referenced from [net.cpp Lines 657-680](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L657-L680)*

3.  **Socket I/O (`ThreadSocketHandler2`)**:
    *   Uses `select()` to monitor sockets for readability and writability.
    *   **Receiving**: Data is read into `pnode->vRecv` buffer.
        *Referenced from [net.cpp Lines 703-738](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L703-L738)*
    *   **Sending**: Data from `pnode->vSend` buffer is sent.
        *Referenced from [net.cpp Lines 745-770](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L745-L770)*

4.  **Message Handling (`ThreadMessageHandler2`, `ProcessMessages`)**:
    *   `ProcessMessages` ([main.cpp Lines 1700-1781](https://github.com/jonzou/btc020/blob/main.cpp#L1700-L1781)) parses `CMessageHeader` from `pnode->vRecv`.
    *   The payload is then passed to `ProcessMessage` ([main.cpp Lines 1786-2220](https://github.com/jonzou/btc020/blob/main.cpp#L1786-L2220)) which handles specific commands (e.g., "version", "addr", "inv", "getdata", "block", "tx").

5.  **Disconnection**:
    *   Nodes can be marked for disconnection (`pnode->fDisconnect = true`) due to errors, inactivity, or shutdown.
    *   `ThreadSocketHandler2` cleans up disconnected nodes, closes sockets, and calls `pnode->Cleanup()`.
    *Referenced from [net.cpp Lines 554-598](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L554-L598)*

### Address Management
*   `mapAddresses`: Global map storing known peer addresses.
    *Referenced from [net.cpp Line 26](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L26)*
*   `AddAddress()`: Adds new addresses to `mapAddresses` and persists them to `addr.dat` via `CAddrDB`.
    *Referenced from [net.cpp Lines 226-267](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L226-L267)*
*   **External IP Detection**: `GetMyExternalIP()` attempts to find the node's external IP using web services like `www.ipaddressworld.com/ip.php` and `checkip.dyndns.org`.
    *Referenced from [net.cpp Lines 164-220](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L164-L220)*

### Inventory System
Bitcoin uses an inventory-based system to announce new transactions and blocks, avoiding redundant data transfer.
*   `CInv` class: Represents an inventory item (e.g., `MSG_TX`, `MSG_BLOCK`) with its type and hash.
    *Defined in [net.h Lines 340-425](https://github.com/jonzou/btc020/blob/ebbdade7/net.h#L340-L425)*
*   **Flow**:
    1.  A node announces new data by sending an "inv" message containing a list of `CInv` objects.
    2.  The receiving node checks if it already has this data (`AlreadyHave()` in `main.cpp`).
    3.  If not, it requests the data using a "getdata" message with the `CInv`.
    4.  The sending node responds with the actual "tx" or "block" message.
*   `mapRelay`, `vRelayExpiration`: Used to cache recently relayed messages to serve "getdata" requests.
    *Referenced from [net.cpp Lines 32-34](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L32-L34)*
*   `mapAlreadyAskedFor`: Tracks items already requested to avoid duplicates.
    *Referenced from [net.cpp Line 35](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L35)*

## Node Discovery via IRC (`irc.h`, `irc.cpp`)

Bitcoin v0.2.0 uses IRC as a bootstrap mechanism to find initial peers.

### IRC Discovery Process (`ThreadIRCSeed`)
*Implemented in [irc.cpp Lines 157-296](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp#L157-L296)*
1.  **Connect to IRC Server**: Connects to `chat.freenode.net:6667` (with fallback to a hardcoded IP).
2.  **Register Nickname**:
    *   If the node has a routable IP, it encodes its own `CAddress` (IP and port) into a Base58Check string prefixed with "u" (e.g., `u1LpyY...`). This is used as the IRC nickname.
        *   `EncodeAddress()`: [irc.cpp Lines 19-27](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp#L19-L27)
    *   If no routable IP or if the name is in use, a random name like "x123456789" is used.
3.  **Join Channel**: Joins the `#bitcoin` channel.
4.  **Listen for Peers**:
    *   Sends a `WHO #bitcoin` command to get a list of users in the channel.
    *   Monitors `JOIN` messages as new users enter.
5.  **Decode Addresses**:
    *   Parses usernames from `WHO` replies (RPL_WHOREPLY, code 352) and `JOIN` messages.
    *   If a username starts with "u", it attempts to decode it using `DecodeAddress()`.
        *   `DecodeAddress()`: [irc.cpp Lines 29-42](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp#L29-L42)
    *   Successfully decoded addresses are added to the node's address manager (`mapAddresses`) via `AddAddress()`.
6.  **Handle PING/PONG**: Responds to IRC PING messages to keep the connection alive.
    *Handled in `RecvLineIRC()`: [irc.cpp Lines 85-105](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp#L85-L105)*
7.  **Error Handling**: Implements exponential backoff for connection failures and retries for name conflicts. Special handling for TOR connections (tries only once due to IRC servers often blocking TOR).

### `ircaddr` Structure
*Defined in [irc.cpp Lines 12-16](https://github.com/jonzou/btc020/blob/ebbdade7/irc.cpp#L12-L16)*
A simple packed struct to hold IP and port for Base58Check encoding/decoding.
```cpp
#pragma pack(push, 1)
struct ircaddr
{
    int ip;
    short port;
};
#pragma pack(pop)
```

This IRC discovery mechanism was an early solution to the peer bootstrapping problem and was later replaced by DNS seeds and hardcoded seed nodes in subsequent Bitcoin versions.

## Network Startup and Shutdown
*   `StartNode()` ([net.cpp Lines 1167-1324](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L1167-L1324)): Initializes the local host address, determines external IP, and starts all networking threads (`ThreadIRCSeed`, `ThreadSocketHandler`, `ThreadOpenConnections`, `ThreadMessageHandler`).
*   `StopNode()` ([net.cpp Lines 1326-1347](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L1326-L1347)): Sets `fShutdown` flag and waits for threads to terminate.
*   `CNetCleanup` class ([net.cpp Lines 1349-1371](https://github.com/jonzou/btc020/blob/ebbdade7/net.cpp#L1349-L1371)): RAII class to ensure sockets are closed and Winsock (on Windows) is cleaned up at exit.
