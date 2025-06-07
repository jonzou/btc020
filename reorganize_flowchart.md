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
