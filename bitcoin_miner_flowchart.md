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
