# Networking in Bitcoin

This document outlines the peer-to-peer (P2P) networking and initial node discovery mechanisms as implemented in this version of the Bitcoin client.

## P2P Protocol and Connection Management

The Bitcoin network is a peer-to-peer network where nodes communicate directly with each other. The core logic for this is found in `net.cpp` and `net.h`.

### Node Connection

Nodes are represented by the `CNode` class. Connections are typically established through TCP sockets.

**1. Initiating a Connection (`ConnectNode` and `OpenNetworkConnection`):**
   - `OpenNetworkConnection(const CAddress& addrConnect)`: This function is a higher-level wrapper that initiates an outbound connection.
     - It checks if already connected to the IP or if it's the local host.
     - Calls `ConnectNode(addrConnect)` to establish the actual connection.
     - If successful, it marks the node as a `fNetworkNode` (a full network node).
     - It then sends its own address (`addrLocalHost`) to the newly connected peer using a "addr" message and requests the peer's known addresses using a "getaddr" message.
     - It also pushes its local subscriptions to the new node.

   ```cpp
   // From net.cpp
   bool OpenNetworkConnection(const CAddress& addrConnect)
   {
       if (fShutdown)
           return false;
       if (addrConnect.ip == addrLocalHost.ip || !addrConnect.IsIPv4() || FindNode(addrConnect.ip))
           return false;

       vnThreadsRunning[1]--; // Assuming this is for thread management
       CNode* pnode = ConnectNode(addrConnect);
       vnThreadsRunning[1]++;
       if (fShutdown)
           return false;
       if (!pnode)
           return false;
       pnode->fNetworkNode = true;

       if (addrLocalHost.IsRoutable() && !fUseProxy)
       {
           vector<CAddress> vAddrToSend;
           vAddrToSend.push_back(addrLocalHost);
           pnode->PushMessage("addr", vAddrToSend);
       }

       pnode->PushMessage("getaddr");
       pnode->fGetAddr = true;
       // ... push subscriptions ...
       return true;
   }
   ```

   - `ConnectNode(CAddress addrConnect, int64 nTimeout)`:
     - Checks if already connected to the IP. If so, returns the existing `CNode` object.
     - Otherwise, it calls `ConnectSocket` to create a new socket connection.
     - If `ConnectSocket` is successful, a new `CNode` object is created, added to the global `vNodes` vector, and its connection time is recorded.
     - The socket is set to non-blocking mode.

   ```cpp
   // From net.cpp
   CNode* ConnectNode(CAddress addrConnect, int64 nTimeout)
   {
       if (addrConnect.ip == addrLocalHost.ip)
           return NULL;

       CNode* pnode = FindNode(addrConnect.ip);
       if (pnode)
       {
           // ... AddRef logic ...
           return pnode;
       }

       // ... logging ...
       CRITICAL_BLOCK(cs_mapAddresses)
           mapAddresses[addrConnect.GetKey()].nLastTry = GetAdjustedTime();

       SOCKET hSocket;
       if (ConnectSocket(addrConnect, hSocket))
       {
           // ... logging ...
           // Set to nonblocking (platform-specific)
           // ...
           CNode* pnode = new CNode(hSocket, addrConnect, false);
           // ... AddRef logic ...
           CRITICAL_BLOCK(cs_vNodes)
               vNodes.push_back(pnode);
           pnode->nTimeConnected = GetTime();
           return pnode;
       }
       return NULL;
   }
   ```

   - `ConnectSocket(const CAddress& addrConnect, SOCKET& hSocketRet)`: This function performs the low-level socket connection.
     - It creates a socket (`socket(AF_INET, SOCK_STREAM, IPPROTO_TCP)`).
     - Handles optional proxy connections (SOCKS4).
     - Calls `connect()` to establish the TCP connection.

**2. Accepting Incoming Connections:**
   - `BindListenPort(string& strError)`: Sets up a listening socket on `DEFAULT_PORT` (8333).
     - Initializes Winsock if on Windows.
     - Creates a socket, sets options like `SO_REUSEADDR` and non-blocking mode.
     - Binds the socket to `INADDR_ANY` (all local IPs).
     - Listens for incoming connections with `listen()`.
   - `ThreadSocketHandler2(void* parg)`: This is the main loop for handling network sockets.
     - It uses `select()` to monitor readable/writable sockets.
     - If the listening socket (`hListenSocket`) has an incoming connection (FD_ISSET), it calls `accept()` to get the new socket.
     - A new `CNode` object is created for the accepted connection with `fInbound = true`.

   ```cpp
   // Inside ThreadSocketHandler2 in net.cpp
   if (FD_ISSET(hListenSocket, &fdsetRecv))
   {
       // ... accept connection ...
       SOCKET hSocket = accept(hListenSocket, (struct sockaddr*)&sockaddr, &len);
       CAddress addr(sockaddr);
       if (hSocket != INVALID_SOCKET)
       {
           CNode* pnode = new CNode(hSocket, addr, true); // fInboundIn = true
           pnode->AddRef();
           CRITICAL_BLOCK(cs_vNodes)
               vNodes.push_back(pnode);
       }
   }
   ```

**3. Node Initialization (`CNode` Constructor):**
   - When a `CNode` object is created (either for an outbound or inbound connection), its constructor initializes various members.
   - Crucially, it sends a "version" message to the peer. This message contains the node's version, services, current time, and addresses.

   ```cpp
   // From CNode constructor in net.h
   CNode(SOCKET hSocketIn, CAddress addrIn, bool fInboundIn=false)
   {
       // ... other initializations ...
       fInbound = fInboundIn;
       // ...
       // Push a version message
       int64 nTime = (fInbound ? GetAdjustedTime() : GetTime());
       CAddress addrYou = (fUseProxy ? CAddress("0.0.0.0") : addr);
       CAddress addrMe = (fUseProxy ? CAddress("0.0.0.0") : addrLocalHost);
       RAND_bytes((unsigned char*)&nLocalHostNonce, sizeof(nLocalHostNonce));
       PushMessage("version", VERSION, nLocalServices, nTime, addrYou, addrMe, nLocalHostNonce, string(pszSubVer));
   }
   ```

### Message Handling

**1. Message Structure (`CMessageHeader`):**
   - Messages are prefixed with a `CMessageHeader` defined in `net.h`.
   - It includes:
     - `pchMessageStart`: A 4-byte magic string (`0xf9, 0xbe, 0xb4, 0xd9`) to identify the start of a message.
     - `pchCommand`: A 12-byte null-padded string for the message type (e.g., "version", "addr", "inv", "getdata", "block", "tx").
     - `nMessageSize`: The size of the message payload.

**2. Receiving Messages (`ThreadSocketHandler2` and `ProcessMessages`):**
   - `ThreadSocketHandler2` uses `select()` to detect when a node's socket has data.
   - It reads data using `recv()` into the node's `vRecv` buffer.
   - `ProcessMessages(CNode* pfrom)` (called from `ThreadMessageHandler2`) is responsible for parsing `pfrom->vRecv`.
     - It scans for `pchMessageStart`.
     - Reads the `CMessageHeader`.
     - If the full message payload (`hdr.nMessageSize`) is available in `vRecv`:
       - Copies the message payload into a separate `CDataStream vMsg`.
       - Calls `ProcessMessage(pfrom, strCommand, vMsg)` to handle the specific message type.

   ```cpp
   // Simplified logic from ProcessMessages in main.cpp (similar structure expected in net.cpp context)
   // Note: The actual ProcessMessages is in main.cpp, but net.cpp's ThreadMessageHandler2 calls it.
   // The following is illustrative of the general flow.
   loop
   {
       // Scan for message start in pfrom->vRecv
       // ... pstart = search(...) ...
       // Read header (hdr) from pfrom->vRecv
       // ... pfrom->vRecv >> hdr; ...
       // If full message available:
       // ... CDataStream vMsg(pfrom->vRecv.begin(), pfrom->vRecv.begin() + nMessageSize, ...); ...
       // ... pfrom->vRecv.ignore(nMessageSize); ...
       // Process message
       // ... fRet = ProcessMessage(pfrom, strCommand, vMsg); ... // This calls the main message handler
   }
   ```

**3. Processing Specific Messages (`ProcessMessage` in `main.cpp`):**
   - This large switch-like function (though implemented with if-else if) in `main.cpp` handles commands like "version", "addr", "inv", "getdata", "getblocks", "tx", "block", etc.
   - **"version"**: Exchanges version information, sets node properties (`pfrom->nVersion`, `pfrom->fClient`), and may trigger `getblocks`.
   - **"addr"**: Receives a list of peer addresses, adds them to `mapAddresses`.
   - **"inv"**: Receives an inventory of hashes (transactions or blocks). If the items are new, they are requested using `pfrom->AskFor(inv)` (which adds to `mapAskFor` to be sent via "getdata").
   - **"getdata"**: Fulfills requests for specific inventory items by sending "tx" or "block" messages.
   - **"tx"**: Receives a transaction. If valid (`tx.AcceptTransaction()`), it's relayed and potentially added to the wallet.
   - **"block"**: Receives a block. If valid (`ProcessBlock()`), it's processed and added to the chain.

**4. Sending Messages (`CNode::PushMessage` and `SendMessages`):**
   - `CNode::PushMessage(const char* pszCommand, ...)`: A family of template functions to construct and queue messages.
     - `BeginMessage(pszCommand)`: Initializes the message in `vSend` buffer with a header.
     - `vSend << ...`: Serializes arguments into the send buffer.
     - `EndMessage()`: Calculates payload size, patches it into the header, and finalizes the message in `vSend`.
   - `SendMessages(CNode* pto)` (called from `ThreadMessageHandler2`):
     - Handles sending data from `pto->vSend` buffer over the socket.
     - Also manages sending keep-alive pings, address broadcasts, "inv" messages from `vInventoryToSend`, and "getdata" messages from `mapAskFor`.

   ```cpp
   // From CNode::PushMessage in net.h
   template<typename T1>
   void PushMessage(const char* pszCommand, const T1& a1)
   {
       try
       {
           BeginMessage(pszCommand);
           vSend << a1;
           EndMessage();
       }
       catch (...)
       {
           AbortMessage();
           throw;
       }
   }

   // Inside ThreadSocketHandler2 in net.cpp (for actual sending)
   if (FD_ISSET(pnode->hSocket, &fdsetSend))
   {
       TRY_CRITICAL_BLOCK(pnode->cs_vSend)
       {
           CDataStream& vSend = pnode->vSend;
           if (!vSend.empty())
           {
               int nBytes = send(pnode->hSocket, &vSend[0], vSend.size(), MSG_NOSIGNAL | MSG_DONTWAIT);
               // ... handle nBytes sent or errors ...
           }
       }
   }
   ```

### Connection Management

- **`ThreadSocketHandler2`**:
  - Regularly checks for nodes to disconnect (if `fDisconnect` is true, or refcount is zero and buffers are empty).
  - Moves disconnected nodes to `vNodesDisconnected` and eventually deletes them when refs are zero.
  - Performs inactivity checks (e.g., no messages in the first 60 seconds, or long periods without sends/receives).
- **`ThreadOpenConnections2`**:
  - Tries to maintain a certain number of connections (`nMaxConnections`).
  - Selects addresses from `mapAddresses` to connect to, prioritizing based on last seen time, last try time, and other heuristics.
  - Avoids connecting to already connected IPs.
- **Address Management (`mapAddresses`, `cs_mapAddresses`):**
  - Stores known peer addresses with timestamps and service information.
  - `AddAddress` adds new addresses or updates existing ones.
  - `AddressCurrentlyConnected` updates the `nTime` for an address when it's part of an active connection.

### Simplified Flowchart: Connection Establishment & Message Processing

```mermaid
graph TD
    subgraph Node A (Initiator)
        A1[Start OpenNetworkConnection] --> A2[ConnectNode];
        A2 --> A3[ConnectSocket];
        A3 -- Success --> A4[Create CNode object];
        A4 --> A5[Push "version" message];
        A5 --> A6[Listen for messages in ThreadMessageHandler];
    end

    subgraph Node B (Receiver)
        B1[StartNode: BindListenPort] --> B2[Listen for incoming connections in ThreadSocketHandler];
        B2 -- Connection Detected --> B3[accept()];
        B3 -- Success --> B4[Create CNode object (inbound)];
        B4 --> B5[Wait for "version" message];
    end

    A6 --> MSG_LOOP_A{Receive/Send Loop};
    MSG_LOOP_A -- Data on Socket --> PA[ProcessMessages from Node B];
    PA --> PB[Handle specific command e.g., "version", "inv"];
    PB -- Need to Send Reply/New Msg --> PC[PushMessage to Node B's vSend];
    PC --> MSG_LOOP_A;

    B5 --> MSG_LOOP_B{Receive/Send Loop};
    MSG_LOOP_B -- Data on Socket --> PAA[ProcessMessages from Node A];
    PAA --> PBB[Handle specific command e.g., "version"];
    PBB -- Got "version" --> PBC[Send own "version", "verack" etc.];
    PBC --> PBD[PushMessage to Node A's vSend];
    PBD --> MSG_LOOP_B;

    subgraph Message Exchange
        direction LR
        A_to_B["version (A->B)"];
        B_to_A["version (B->A)"];
        A_to_B --> B_to_A;
        B_to_A --> Further_Comms["addr, inv, getdata, etc."];
    end

    A5 --> A_to_B;
    B4 --> B5;
```

## Node Discovery via IRC

Before more robust DNS seeds and hardcoded seeds became common, early versions of Bitcoin used IRC (Internet Relay Chat) as a mechanism for nodes to discover each other. The logic for this is in `irc.cpp` and `irc.h`.

### IRC Discovery Mechanism

1.  **Connection**:
    *   The `ThreadIRCSeed` function attempts to connect to an IRC server (e.g., `chat.freenode.net` on port 6667).
    *   It uses `ConnectSocket` for the TCP connection.
    *   It performs a basic IRC handshake: sends `NICK` and `USER` commands. The nickname chosen is either a Base58Check encoded version of the node's own address (if routable and not using a proxy) or a random name like "x" followed by random numbers.

    ```cpp
    // From irc.cpp ThreadIRCSeed
    CAddress addrConnect("216.155.130.130:6667"); // Default freenode
    // ... DNS lookup for chat.freenode.net ...
    SOCKET hSocket;
    if (!ConnectSocket(addrConnect, hSocket)) { /* ... handle error ... */ }

    // ... RecvUntil to wait for IRC server messages ...

    string strMyName;
    if (addrLocalHost.IsRoutable() && !fUseProxy && !fNameInUse)
        strMyName = EncodeAddress(addrLocalHost);
    else
        strMyName = strprintf("x%u", GetRand(1000000000));

    Send(hSocket, strprintf("NICK %s\r", strMyName.c_str()).c_str());
    Send(hSocket, strprintf("USER %s 8 * : %s\r", strMyName.c_str(), strMyName.c_str()).c_str());
    ```

2.  **Joining Channel and Getting Users**:
    *   After successful registration, the node joins a specific channel (e.g., "#bitcoin").
    *   It then sends a `WHO #bitcoin` command to list users in the channel.

    ```cpp
    // From irc.cpp ThreadIRCSeed
    Send(hSocket, "JOIN #bitcoin\r");
    Send(hSocket, "WHO #bitcoin\r");
    ```

3.  **Decoding Addresses**:
    *   The client parses messages from the IRC server:
        *   `352` (WHO reply): Extracts usernames from WHO replies.
        *   `JOIN`: Extracts the username of users joining the channel.
    *   If a username starts with 'u', it's assumed to be an encoded Bitcoin address.
    *   `DecodeAddress(pszName, addr)` is called to decode the Base58Check string back into an IP address and port.
        *   `EncodeAddress` and `DecodeAddress` use a simple scheme: the character 'u' followed by Base58Check encoding of a struct `ircaddr { int ip; short port; }`.

    ```cpp
    // From irc.cpp ThreadIRCSeed, inside the message processing loop
    if (pszName[0] == 'u')
    {
        CAddress addr;
        if (DecodeAddress(pszName, addr))
        {
            // ... AddAddress to local address manager ...
            nGotIRCAddresses++;
        }
    }

    // From irc.cpp
    string EncodeAddress(const CAddress& addr)
    {
        struct ircaddr tmp;
        tmp.ip    = addr.ip;
        tmp.port  = addr.port;
        vector<unsigned char> vch(UBEGIN(tmp), UEND(tmp));
        return string("u") + EncodeBase58Check(vch);
    }

    bool DecodeAddress(string str, CAddress& addr)
    {
        vector<unsigned char> vch;
        if (!DecodeBase58Check(str.substr(1), vch)) // Skip the 'u'
            return false;
        // ... copy to ircaddr and create CAddress ...
        return true;
    }
    ```

4.  **Address Management**:
    *   Successfully decoded addresses are added to the node's local address manager (`mapAddresses`) via `AddAddress`. This makes them candidates for future P2P connections.
    *   The variable `nGotIRCAddresses` tracks how many addresses were obtained via IRC.

5.  **Error Handling and Retry**:
    *   The `ThreadIRCSeed` function has retry logic with increasing wait times (`nErrorWait`, `nRetryWait`) if connections or IRC commands fail.
    *   It handles cases like "name already in use" (`fNameInUse`).
    *   For TOR users, it might only try once due to IRC networks often blocking TOR.

This IRC mechanism was a simple way for early nodes to bootstrap and find peers. It was largely superseded by DNS seeds, hardcoded seed nodes, and improved peer-to-peer address gossiping.

### Mermaid Flowchart for IRC Discovery Process

```mermaid
graph TD
    A[Start ThreadIRCSeed] --> B{Connect to IRC Server (e.g., freenode)};
    B -- Success --> C[Send NICK and USER commands];
    C -- IRC Welcome (e.g., 004) --> D[JOIN #bitcoin channel];
    C -- Name in Use (433) --> BFail[Handle Nick Collision, Wait, Retry];
    B -- Failure --> BFail;
    D --> E[Send WHO #bitcoin];
    E --> F[Loop: Receive IRC Lines];
    F -- PING received --> G[Respond with PONG];
    G --> F;
    F -- WHO Reply (352) or JOIN received --> H{Parse Nickname};
    H -- Nickname starts with 'u' --> I{DecodeAddress(Nickname)};
    I -- Success --> J[AddAddress(decoded_addr)];
    J --> F;
    I -- Failure (Invalid Address Format) --> F;
    H -- Nickname not 'u' prefixed --> F;
    F -- Socket Closed/Error --> K[Wait and Retry Connection];
    F -- Shutdown Signal --> L[End];
    BFail -- Wait --> A;
    K --> A;
```
