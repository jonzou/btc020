# Core Blockchain System Flowcharts

This document contains Mermaid flowcharts illustrating various core processes within a Bitcoin-like blockchain system.

## AcceptTransaction Flowchart

```mermaid
graph TD
    A[Receive Transaction] --> B{Basic Validation};
    B -- Valid --> C{Check for Double Spending};
    B -- Invalid --> H[Reject Transaction];
    C -- No Double Spending --> D{Verify Signatures};
    C -- Double Spending Detected --> H;
    D -- Valid Signatures --> E{Check Transaction Fee};
    D -- Invalid Signatures --> H;
    E -- Sufficient Fee --> F{"Add to Mempool?"};
    E -- Insufficient Fee --> H;
    F -- Yes --> G[Add to Mempool];
    F -- No --> H;
```

## ConnectInputs Flowchart

```mermaid
graph TD
    A[Receive Transaction Inputs] --> B{For each Input};
    B -- Loop Start --> C{"Find UTXO?"};
    C -- UTXO Found --> D[Mark UTXO as Spent];
    C -- UTXO Not Found --> G["Error during connection?"];
    D --> E{"Create New UTXOs (Outputs)?"};
    E -- Yes --> F[Add New UTXOs to UTXO Set];
    E -- No --> B;
    F --> B;
    B -- Loop End --> H[Inputs Connected];
    G -- Yes --> I[Handle Error];
    G -- No --> H;
```

## ProcessBlock Flowchart

```mermaid
graph TD
    A[Receive New Block] --> B{"Block Syntactically Valid? (e.g., size, PoW)"};
    B -- No --> J[Block Invalid or Error];
    B -- Yes --> C{"Block Already Processed?"};
    C -- Yes --> J;
    C -- No --> D{For each Transaction in Block};
    D -- Loop Start --> E["Validate Transaction (similar to AcceptTransaction but within block context)"];
    E -- Invalid --> J;
    E -- Valid --> F["Connect Transaction Inputs (Call ConnectInputs logic)"];
    F -- Error --> J;
    F -- Connected --> D;
    D -- Loop End --> G{"All Transactions Valid and Connected?"};
    G -- No --> J;
    G -- Yes --> H[Update Block Index];
    H --> I{"Store Block?"};
    I -- Yes --> K[Block Processed Successfully];
    I -- No --> J;
```

## AcceptBlock Flowchart

```mermaid
graph TD
    A[Validated Block Received] --> B{"Check for Orphan?"};
    B -- Orphan --> G[Block Not Accepted (e.g., orphan, failed connection)];
    B -- Not Orphan --> C{"Connect to Main Chain?"};
    C -- Yes --> D[Add to Main Chain];
    C -- No --> E{"Is it a new longest chain?"};
    E -- Yes --> F["Trigger Reorganization (Call Reorganize logic)"];
    F --> D;
    E -- No --> H[Add to Side Chain / Store as Orphan];
    H --> G;
    D --> I[Update Chain State (e.g., difficulty, height)];
    I --> J{"Broadcast Block to Peers?"};
    J -- Yes --> K[Block Accepted];
    J -- No --> K;
```

## Reorganize Flowchart

```mermaid
graph TD
    A[Reorganization Triggered (New longer chain identified)] --> B["Identify Fork Point"];
    B --> C{For Blocks on Old Chain (after fork point)};
    C -- Loop Start --> D["Disconnect Block"];
    D --> E["Revert Transactions (add back to mempool or discard)"];
    E --> F[Update UTXO Set];
    F --> C;
    C -- Loop End --> G{For Blocks on New Chain (after fork point)};
    G -- Loop Start --> H["Connect Block (similar to AcceptBlock/ProcessBlock logic for these blocks)"];
    H -- Invalid Block --> M["Reorganization Failed? (e.g. invalid block found in new chain during process)"];
    H -- Valid Block --> I["Apply Transactions"];
    I --> J[Update UTXO Set];
    J --> G;
    G -- Loop End --> K["Switch to New Chain"];
    K --> L[Update Chain Head Pointer];
    L --> N{"Broadcast New Chain Tip?"};
    N -- Yes --> O[Reorganization Complete];
    N -- No --> O;
    M -- Yes --> P[Handle Reorganization Failure];
    M -- No --> K; // Should not happen if block is invalid
```

## BitcoinMiner Flowchart

```mermaid
graph TD
    A[Start Mining] --> B["Get Latest Block Template (Coinbase transaction, select transactions from mempool)"];
    B --> C["Assemble Block Header (Version, Prev. Block Hash, Merkle Root, Time, Bits, Nonce)"];
    C --> D{Loop: Vary Nonce / ExtraNonce};
    D --> E["Hash Block Header"];
    E --> F{"Is Hash < Target?"};
    F -- No --> G{"Receive New Block from Network?"};
    G -- No --> D;
    G -- Yes --> H["Validate New Block"];
    H -- Valid --> B; // Restart with new template
    H -- Invalid --> D; // Continue mining current attempt
    F -- Yes --> I["Block Found!"];
    I --> J["Broadcast Valid Block"];
    J --> B; // Go back to get new template for next block
    A --> K[Stop Mining]; // Allow for stopping condition
```
