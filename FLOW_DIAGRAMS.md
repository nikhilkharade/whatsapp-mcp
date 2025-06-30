# WhatsApp MCP Server - Flow Diagrams

This document contains detailed flow diagrams for the WhatsApp MCP Server system, illustrating various processes and interactions.

## System Overview Diagram

```mermaid
graph TB
    subgraph "AI Assistant Layer"
        A[Claude Desktop/Cursor]
    end
    
    subgraph "MCP Server Layer"
        B[Python MCP Server]
        B1[Tool Registry]
        B2[Data Transformation]
        B3[HTTP Client]
    end
    
    subgraph "WhatsApp Bridge Layer"
        C[Go WhatsApp Bridge]
        C1[WhatsApp Client]
        C2[Event Handlers]
        C3[REST API Server]
    end
    
    subgraph "WhatsApp Services"
        D[WhatsApp Web API]
        D1[Message Events]
        D2[Authentication]
        D3[Message Sending]
    end
    
    subgraph "Data Layer"
        E[(messages.db)]
        F[(whatsapp.db)]
    end
    
    A <==> B
    B --> B1
    B --> B2
    B --> B3
    B3 <==> C3
    B2 <==> E
    
    C --> C1
    C --> C2
    C --> C3
    C1 <==> D
    C2 --> E
    C2 --> F
    
    D --> D1
    D --> D2
    D --> D3
```

## 1. System Initialization Flow

```mermaid
sequenceDiagram
    participant User
    participant Claude as Claude Desktop
    participant MCP as MCP Server
    participant Bridge as Go Bridge
    participant WA as WhatsApp API
    participant DB as SQLite DBs
    
    Note over User, DB: System Startup Process
    
    User->>Bridge: Start Go bridge (go run main.go)
    Bridge->>DB: Initialize SQLite connections
    Bridge->>Bridge: Setup WhatsApp client
    
    alt New Device (No Session)
        Bridge->>WA: Request QR code
        WA-->>Bridge: QR code data
        Bridge->>User: Display QR terminal
        User->>User: Scan QR with WhatsApp mobile
        User->>WA: Authentication via mobile app
        WA-->>Bridge: Session established
        Bridge->>DB: Store session data (whatsapp.db)
    else Existing Session
        Bridge->>DB: Load session data
        Bridge->>WA: Connect with stored session
        WA-->>Bridge: Connection established
    end
    
    Bridge->>Bridge: Start REST API server (port 8080)
    Bridge->>WA: Register event handlers
    
    Note over User, DB: MCP Server Startup
    
    User->>Claude: Start Claude Desktop
    Claude->>MCP: Initialize MCP server via stdio
    MCP->>MCP: Register WhatsApp tools
    MCP->>Bridge: Health check (optional)
    
    Note over User, DB: Ready State
    Claude-->>User: WhatsApp tools available
```

## 2. Message Retrieval Flow (Detailed)

```mermaid
sequenceDiagram
    participant Claude
    participant MCP as MCP Server
    participant DB as messages.db
    
    Note over Claude, DB: User Query Processing
    
    Claude->>MCP: list_messages(query="hello", limit=10, page=0)
    
    MCP->>MCP: Validate parameters
    MCP->>MCP: Build SQL query with filters
    
    Note over MCP: Query Construction
    MCP->>DB: SELECT m.*, c.name FROM messages m JOIN chats c...
    
    alt Messages Found
        DB-->>MCP: Message rows with chat info
        MCP->>MCP: Transform rows to Message objects
        MCP->>MCP: Apply pagination logic
        
        opt Include Context
            MCP->>DB: SELECT context messages (before/after)
            DB-->>MCP: Context message rows
            MCP->>MCP: Merge context with main results
        end
        
        MCP->>MCP: Format response data
        MCP-->>Claude: List[Message] with context
        
    else No Messages Found
        DB-->>MCP: Empty result set
        MCP-->>Claude: Empty list []
    end
    
    Note over Claude, DB: Error Handling
    alt Database Error
        DB-->>MCP: SQLite error
        MCP->>MCP: Log error details
        MCP-->>Claude: Error response
    end
```

## 3. Message Sending Flow (Detailed)

```mermaid
sequenceDiagram
    participant Claude
    participant MCP as MCP Server
    participant Bridge as Go Bridge
    participant WA as WhatsApp API
    participant DB as messages.db
    
    Note over Claude, DB: Send Message Process
    
    Claude->>MCP: send_message(recipient="1234567890", message="Hello")
    
    MCP->>MCP: Validate input parameters
    MCP->>MCP: Prepare HTTP request payload
    
    MCP->>Bridge: POST /api/send {"recipient": "...", "message": "..."}
    
    Bridge->>Bridge: Parse request body
    Bridge->>Bridge: Validate recipient format
    
    alt Valid Phone Number
        Bridge->>Bridge: Create JID from phone (user@s.whatsapp.net)
    else Valid JID Format
        Bridge->>Bridge: Parse existing JID
    else Invalid Format
        Bridge-->>MCP: HTTP 400 - Invalid recipient format
        MCP-->>Claude: {"success": false, "message": "Invalid format"}
    end
    
    Bridge->>WA: SendMessage(context, recipientJID, messageProto)
    
    alt Message Sent Successfully
        WA-->>Bridge: Success response
        Bridge->>DB: Store sent message in database
        Bridge-->>MCP: HTTP 200 {"success": true, "message": "sent"}
        MCP-->>Claude: Success response
        
    else Send Failed
        WA-->>Bridge: Error response
        Bridge-->>MCP: HTTP 500 {"success": false, "message": "error details"}
        MCP-->>Claude: Failure response
    end
    
    Note over Claude, DB: Real-time Event Processing
    WA->>Bridge: Message event (if successful)
    Bridge->>DB: Update message status/timestamp
```

## 4. WhatsApp Event Processing Flow

```mermaid
sequenceDiagram
    participant WA as WhatsApp API
    participant Bridge as Go Bridge
    participant DB as SQLite DBs
    participant MCP as MCP Server (if running)
    
    Note over WA, MCP: Real-time Event Handling
    
    WA->>Bridge: WebSocket event stream
    
    Bridge->>Bridge: Event type switch
    
    alt Message Event
        Bridge->>Bridge: Extract message content
        Bridge->>Bridge: Determine chat JID and sender
        Bridge->>Bridge: Get/generate chat name
        Bridge->>DB: INSERT/UPDATE chats table
        Bridge->>DB: INSERT INTO messages table
        Bridge->>Bridge: Log message received
        
    else History Sync Event
        Bridge->>Bridge: Process conversation batch
        
        loop For each conversation
            Bridge->>Bridge: Parse conversation metadata
            Bridge->>DB: UPSERT chat information
            
            loop For each message in conversation
                Bridge->>Bridge: Extract message data
                Bridge->>Bridge: Determine timestamp and sender
                Bridge->>DB: INSERT message (ignore duplicates)
            end
        end
        
        Bridge->>Bridge: Log sync completion
        
    else Connection Event
        Bridge->>Bridge: Log connection status
        Bridge->>WA: Request history sync (if first connection)
        
    else Logout Event
        Bridge->>Bridge: Log logout warning
        Bridge->>Bridge: Clear connection state
    end
    
    Note over WA, MCP: Database State Updated
```

## 5. Contact Search Flow

```mermaid
sequenceDiagram
    participant Claude
    participant MCP as MCP Server
    participant DB as messages.db
    
    Note over Claude, DB: Contact Search Process
    
    Claude->>MCP: search_contacts(query="john")
    
    MCP->>MCP: Sanitize search query
    MCP->>MCP: Build LIKE pattern (%john%)
    
    MCP->>DB: SELECT DISTINCT sender, chat_jid FROM messages WHERE...
    DB-->>MCP: Unique sender records
    
    MCP->>DB: SELECT jid, name FROM chats WHERE name LIKE...
    DB-->>MCP: Matching chat records
    
    MCP->>MCP: Merge and deduplicate results
    MCP->>MCP: Extract phone numbers from JIDs
    MCP->>MCP: Create Contact objects
    
    MCP-->>Claude: List[Contact] with phone, name, jid
```

## 6. Chat Management Flow

```mermaid
flowchart TD
    A[Claude requests chat info] --> B{Request type?}
    
    B -->|list_chats| C[Build chat list query]
    B -->|get_chat| D[Query specific chat]
    B -->|get_direct_chat_by_contact| E[Find chat by phone number]
    B -->|get_contact_chats| F[Find all chats for contact]
    
    C --> G[Apply filters and sorting]
    G --> H[Execute paginated query]
    H --> I[Include last message if requested]
    
    D --> J[Query chat by JID]
    J --> K[Get chat metadata]
    K --> I
    
    E --> L[Convert phone to JID pattern]
    L --> M[Find matching chat]
    M --> I
    
    F --> N[Query all chats containing JID]
    N --> O[Apply pagination]
    O --> I
    
    I --> P[Format response data]
    P --> Q[Return to Claude]
```

## 7. Database Transaction Flow

```mermaid
sequenceDiagram
    participant App as Application Layer
    participant Store as MessageStore
    participant DB as SQLite Database
    
    Note over App, DB: Message Storage Transaction
    
    App->>Store: StoreMessage(id, chatJID, sender, content, timestamp, isFromMe)
    
    Store->>DB: BEGIN TRANSACTION
    
    Store->>DB: INSERT OR REPLACE INTO messages (...)
    
    alt Insert Successful
        Store->>DB: COMMIT
        Store-->>App: nil (success)
        
    else Insert Failed
        DB-->>Store: SQLite error
        Store->>DB: ROLLBACK
        Store-->>App: error details
    end
    
    Note over App, DB: Chat Update Transaction
    
    App->>Store: StoreChat(jid, name, lastMessageTime)
    
    Store->>DB: BEGIN TRANSACTION
    Store->>DB: INSERT OR REPLACE INTO chats (...)
    
    alt Update Successful
        Store->>DB: COMMIT
        Store-->>App: nil (success)
    else Update Failed
        DB-->>Store: SQLite error
        Store->>DB: ROLLBACK
        Store-->>App: error details
    end
```

## 8. Error Handling Flow

```mermaid
flowchart TD
    A[Operation Started] --> B{Error Occurred?}
    
    B -->|No| C[Continue Normal Flow]
    B -->|Yes| D{Error Type?}
    
    D -->|Database Error| E[Log SQLite Error]
    D -->|Network Error| F[Log Connection Error]
    D -->|WhatsApp API Error| G[Log API Error]
    D -->|Validation Error| H[Log Input Error]
    
    E --> I[Check Database Connection]
    F --> J[Check Network Status]
    G --> K[Check WhatsApp Session]
    H --> L[Return Validation Message]
    
    I --> M{Can Recover?}
    J --> M
    K --> M
    
    M -->|Yes| N[Retry Operation]
    M -->|No| O[Return Error Response]
    
    L --> O
    N --> P{Retry Successful?}
    
    P -->|Yes| C
    P -->|No| Q[Log Retry Failure]
    Q --> O
    
    C --> R[Return Success Response]
    O --> S[Return Error Response]
```

## 9. Authentication State Flow

```mermaid
stateDiagram-v2
    [*] --> Disconnected
    
    Disconnected --> Connecting : Start Bridge
    Connecting --> QRRequired : No Saved Session
    Connecting --> Authenticating : Has Saved Session
    
    QRRequired --> QRDisplayed : Generate QR
    QRDisplayed --> QRScanned : User Scans QR
    QRScanned --> Authenticating : Mobile App Confirms
    
    Authenticating --> Connected : Auth Success
    Authenticating --> AuthFailed : Auth Failure
    
    Connected --> SyncingHistory : First Connection
    Connected --> Ready : Subsequent Connections
    
    SyncingHistory --> Ready : Sync Complete
    
    Ready --> MessageReceiving : Incoming Message
    Ready --> MessageSending : Outgoing Message
    
    MessageReceiving --> Ready : Message Processed
    MessageSending --> Ready : Message Sent
    
    Ready --> Disconnected : Connection Lost
    AuthFailed --> Disconnected : Reset Required
    
    Ready --> LoggedOut : Session Expired
    LoggedOut --> Disconnected : Clear Session
```

## 10. Data Synchronization Flow

```mermaid
sequenceDiagram
    participant WA as WhatsApp Server
    participant Bridge as Go Bridge
    participant DB as Local Database
    participant MCP as MCP Server
    
    Note over WA, MCP: Initial History Sync
    
    Bridge->>WA: Connect with authenticated session
    WA->>Bridge: Send HistorySync events
    
    loop For each HistorySync batch
        WA->>Bridge: HistorySync(conversations[])
        
        Bridge->>Bridge: Parse conversation data
        
        loop For each conversation
            Bridge->>DB: UPSERT chat metadata
            
            loop For each message
                Bridge->>Bridge: Extract message details
                Bridge->>DB: INSERT message (ON CONFLICT IGNORE)
            end
        end
        
        Bridge->>Bridge: Update sync progress
    end
    
    Note over WA, MCP: Real-time Sync
    
    WA->>Bridge: New message event
    Bridge->>DB: INSERT new message
    Bridge->>DB: UPDATE chat last_message_time
    
    Note over WA, MCP: MCP Query Handling
    
    MCP->>DB: Query message history
    DB-->>MCP: Return cached results
    
    Note over WA, MCP: Data Consistency
    
    Bridge->>Bridge: Periodic health check
    Bridge->>DB: VACUUM and ANALYZE (maintenance)
```

This comprehensive set of flow diagrams illustrates the complete operational behavior of the WhatsApp MCP Server system, from initialization through various operational scenarios and error conditions. 