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
