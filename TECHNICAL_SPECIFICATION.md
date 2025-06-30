# WhatsApp MCP Server - Technical Specification

## Table of Contents
1. [Overview](#overview)
2. [System Requirements](#system-requirements)
3. [Technology Stack](#technology-stack)
4. [Component Specifications](#component-specifications)
5. [Data Models](#data-models)
6. [API Specifications](#api-specifications)
7. [Database Design](#database-design)
8. [Protocol Implementation](#protocol-implementation)
9. [Error Handling](#error-handling)
10. [Performance Specifications](#performance-specifications)
11. [Security Implementation](#security-implementation)
12. [Configuration Management](#configuration-management)

## Overview

The WhatsApp MCP Server is a bi-component system that bridges AI assistants with WhatsApp through the Model Context Protocol (MCP). It consists of a Go-based WhatsApp bridge and a Python-based MCP server, providing secure, local access to WhatsApp messaging capabilities.

### Architecture Principles
- **Separation of Concerns**: Clear separation between WhatsApp integration and MCP protocol handling
- **Local-First**: All data processing occurs locally without external cloud dependencies
- **Real-time Processing**: Event-driven architecture for immediate message handling
- **Fault Tolerance**: Graceful degradation and recovery mechanisms

## System Requirements

### Hardware Requirements
| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 1 core, 1 GHz | 2 cores, 2.4 GHz |
| RAM | 512 MB | 1 GB |
| Storage | 1 GB free space | 5 GB free space |
| Network | Broadband internet | Stable broadband |

### Software Requirements
| Component | Version | Purpose |
|-----------|---------|---------|
| Go | 1.24+ | WhatsApp bridge implementation |
| Python | 3.11+ | MCP server implementation |
| SQLite | 3.x | Data persistence |
| UV | Latest | Python package management |
| CGO | Enabled | SQLite integration |

### Platform Support
- **Primary**: macOS, Linux
- **Secondary**: Windows (with additional CGO setup)
- **Mobile**: WhatsApp mobile app for authentication

## Technology Stack

### Go Bridge (`whatsapp-bridge/`)
```go
// Core Dependencies
go 1.24.1

require (
    github.com/mattn/go-sqlite3 v1.14.24        // SQLite driver
    go.mau.fi/whatsmeow v0.0.0-20250318233852   // WhatsApp Web API
)

// Supporting Libraries
filippo.io/edwards25519 v1.1.0                 // Cryptography
github.com/gorilla/websocket v1.5.0            // WebSocket support
github.com/mdp/qrterminal v1.0.1               // QR code display
github.com/rs/zerolog v1.33.0                  // Structured logging
google.golang.org/protobuf v1.36.5             // Protocol buffers
```

### Python MCP Server (`whatsapp-mcp-server/`)
```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.28.1",        # HTTP client for Go bridge communication
    "mcp[cli]>=1.6.0",      # Model Context Protocol framework
    "requests>=2.32.3",     # REST API client
]
```

## Component Specifications

### 1. WhatsApp Bridge (Go)

#### Core Components
```go
// main.go - Entry Point
func main() {
    // Initialize components in order
    messageStore := NewMessageStore()
    client := whatsmeow.NewClient(deviceStore, logger)
    startRESTServer(client, 8080)
    
    // Event loop
    for evt := range events {
        handleEvent(evt)
    }
}

// MessageStore - Database Interface
type MessageStore struct {
    db *sql.DB
}

// Message - Core Data Structure
type Message struct {
    Time     time.Time
    Sender   string
    Content  string
    IsFromMe bool
}
```

#### REST API Server
```go
// HTTP Server Configuration
type Server struct {
    Port    int           // Default: 8080
    Timeout time.Duration // Default: 30s
    Client  *whatsmeow.Client
}

// Endpoints
POST /api/send
Content-Type: application/json
Request: SendMessageRequest
Response: SendMessageResponse
```

#### Event Handlers
```go
// Event Handler Registry
type EventHandler interface {
    Handle(evt interface{}) error
}

// Supported Events
- *events.Message      // Incoming messages
- *events.HistorySync  // Message history synchronization
- *events.Connected    // Connection established
- *events.LoggedOut    // Session expired
```

### 2. MCP Server (Python)

#### FastMCP Framework Integration
```python
# main.py - MCP Server
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("whatsapp")

# Tool Registration Pattern
@mcp.tool()
def tool_name(param: Type) -> ReturnType:
    """Tool description."""
    return implementation()
```

#### Business Logic Layer
```python
# whatsapp.py - Core Logic
class WhatsAppInterface:
    def __init__(self):
        self.db_path = MESSAGES_DB_PATH
        self.api_base = WHATSAPP_API_BASE_URL
    
    def execute_query(self, query: str, params: tuple) -> List[Dict]:
        """Execute SQLite query with parameters."""
        
    def call_api(self, endpoint: str, data: Dict) -> Dict:
        """Call Go bridge REST API."""
```

## Data Models

### Core Data Structures

#### Message Model
```python
@dataclass
class Message:
    timestamp: datetime      # Message timestamp (UTC)
    sender: str             # Sender JID or phone number
    content: str            # Text content of message
    is_from_me: bool       # True if sent by current user
    chat_jid: str          # Chat identifier (JID)
    id: str                # Unique message identifier
    chat_name: Optional[str] = None  # Display name of chat
```

#### Chat Model
```python
@dataclass
class Chat:
    jid: str                           # WhatsApp JID
    name: Optional[str]                # Chat/contact name
    last_message_time: Optional[datetime]  # Last activity
    last_message: Optional[str] = None     # Last message content
    last_sender: Optional[str] = None      # Last message sender
    last_is_from_me: Optional[bool] = None # Last message direction
    
    @property
    def is_group(self) -> bool:
        """Determine if chat is a group."""
        return self.jid.endswith("@g.us")
```

#### Contact Model
```python
@dataclass
class Contact:
    phone_number: str       # E.164 format phone number
    name: Optional[str]     # Contact display name
    jid: str               # WhatsApp JID
```

#### Message Context Model
```python
@dataclass
class MessageContext:
    message: Message           # Target message
    before: List[Message]      # Messages before target
    after: List[Message]       # Messages after target
```

### Go Data Structures

#### Database Models
```go
// MessageStore represents the SQLite database interface
type MessageStore struct {
    db *sql.DB
}

// Message represents a WhatsApp message
type Message struct {
    Time     time.Time  `json:"time"`
    Sender   string     `json:"sender"`
    Content  string     `json:"content"`
    IsFromMe bool       `json:"is_from_me"`
}

// SendMessageRequest represents API request format
type SendMessageRequest struct {
    Recipient string `json:"recipient"`
    Message   string `json:"message"`
}

// SendMessageResponse represents API response format
type SendMessageResponse struct {
    Success bool   `json:"success"`
    Message string `json:"message"`
}
```

## API Specifications

### MCP Tools API

#### 1. search_contacts
```python
def search_contacts(query: str) -> List[Dict[str, Any]]
```
**Implementation Details:**
- Performs LIKE queries on chat names and message senders
- Extracts phone numbers from JID formats
- Deduplicates results across multiple data sources
- Returns contacts sorted by interaction frequency

**Query Logic:**
```sql
SELECT DISTINCT 
    CASE 
        WHEN sender LIKE '%@s.whatsapp.net' 
        THEN SUBSTR(sender, 1, INSTR(sender, '@') - 1)
        ELSE sender 
    END as phone_number,
    c.name,
    m.chat_jid as jid
FROM messages m 
LEFT JOIN chats c ON m.chat_jid = c.jid 
WHERE (c.name LIKE ? OR sender LIKE ?)
```

#### 2. list_messages
```python
def list_messages(
    date_range: Optional[Tuple[datetime, datetime]] = None,
    sender_phone_number: Optional[str] = None,
    chat_jid: Optional[str] = None,
    query: Optional[str] = None,
    limit: int = 20,
    page: int = 0,
    include_context: bool = True,
    context_before: int = 1,
    context_after: int = 1
) -> List[Dict[str, Any]]
```

**Implementation Details:**
- Supports multiple filter combinations
- Implements offset-based pagination
- Optional context retrieval for conversation flow
- Full-text search on message content

**Query Construction:**
```python
def build_query(filters: Dict) -> Tuple[str, List]:
    base_query = """
    SELECT m.*, c.name as chat_name 
    FROM messages m 
    LEFT JOIN chats c ON m.chat_jid = c.jid 
    WHERE 1=1
    """
    
    conditions = []
    params = []
    
    if filters.get('date_range'):
        conditions.append("m.timestamp BETWEEN ? AND ?")
        params.extend(filters['date_range'])
    
    # Additional filter logic...
```

#### 3. send_message
```python
def send_message(recipient: str, message: str) -> Dict[str, Any]
```

**Implementation Details:**
- Validates recipient format (phone number or JID)
- Makes HTTP POST to Go bridge
- Handles both individual and group messaging
- Returns structured success/error response

**Recipient Format Handling:**
```python
def normalize_recipient(recipient: str) -> str:
    """Normalize recipient to proper JID format."""
    if '@' in recipient:
        return recipient  # Already a JID
    else:
        return f"{recipient}@s.whatsapp.net"  # Phone number
```

### REST API (Go Bridge)

#### POST /api/send
```http
POST /api/send HTTP/1.1
Content-Type: application/json
Host: localhost:8080

{
    "recipient": "1234567890",
    "message": "Hello World"
}
```

**Response Format:**
```json
{
    "success": true,
    "message": "Message sent to 1234567890"
}
```

**Error Responses:**
```json
{
    "success": false,
    "message": "Error: recipient not found"
}
```

## Database Design

### Schema Definition

#### messages.db
```sql
-- Chat metadata table
CREATE TABLE IF NOT EXISTS chats (
    jid TEXT PRIMARY KEY,                -- WhatsApp JID
    name TEXT,                          -- Chat/contact name  
    last_message_time TIMESTAMP         -- Last activity timestamp
);

-- Message content table
CREATE TABLE IF NOT EXISTS messages (
    id TEXT,                           -- Message ID from WhatsApp
    chat_jid TEXT,                     -- Foreign key to chats.jid
    sender TEXT,                       -- Sender phone/JID
    content TEXT,                      -- Message text content
    timestamp TIMESTAMP,               -- Message timestamp (UTC)
    is_from_me BOOLEAN,               -- Direction indicator
    PRIMARY KEY (id, chat_jid),       -- Composite primary key
    FOREIGN KEY (chat_jid) REFERENCES chats(jid)
);

-- Performance indexes
CREATE INDEX IF NOT EXISTS idx_messages_timestamp ON messages(timestamp);
CREATE INDEX IF NOT EXISTS idx_messages_chat_jid ON messages(chat_jid);
CREATE INDEX IF NOT EXISTS idx_messages_content ON messages(content);
CREATE INDEX IF NOT EXISTS idx_chats_last_message_time ON chats(last_message_time);
```

### Database Operations

#### Connection Management
```go
// Go side - Write operations
func NewMessageStore() (*MessageStore, error) {
    db, err := sql.Open("sqlite3", "file:store/messages.db?_foreign_keys=on")
    if err != nil {
        return nil, err
    }
    
    // Enable WAL mode for better concurrency
    _, err = db.Exec("PRAGMA journal_mode=WAL")
    return &MessageStore{db: db}, err
}
```

```python
# Python side - Read operations
def get_connection() -> sqlite3.Connection:
    conn = sqlite3.connect(MESSAGES_DB_PATH, timeout=30.0)
    conn.row_factory = sqlite3.Row  # Enable column access by name
    return conn
```

#### Transaction Handling
```go
// Transactional message storage
func (store *MessageStore) StoreMessage(msg MessageData) error {
    tx, err := store.db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    _, err = tx.Exec(
        "INSERT OR REPLACE INTO messages (id, chat_jid, sender, content, timestamp, is_from_me) VALUES (?, ?, ?, ?, ?, ?)",
        msg.ID, msg.ChatJID, msg.Sender, msg.Content, msg.Timestamp, msg.IsFromMe,
    )
    if err != nil {
        return err
    }
    
    return tx.Commit()
}
```

## Protocol Implementation

### MCP Protocol Compliance

#### Tool Definition
```python
from mcp.types import Tool

def get_tool_definition() -> Tool:
    return Tool(
        name="search_contacts",
        description="Search WhatsApp contacts by name or phone number",
        inputSchema={
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search term for contact name or phone"
                }
            },
            "required": ["query"]
        }
    )
```

#### Message Handling
```python
# MCP Request Processing
async def handle_request(request: Request) -> Response:
    if request.method == "tools/call":
        tool_name = request.params.name
        arguments = request.params.arguments
        
        result = await execute_tool(tool_name, arguments)
        
        return Response(
            id=request.id,
            result={"content": [{"type": "text", "text": result}]}
        )
```

### WhatsApp Protocol Integration

#### whatsmeow Library Usage
```go
// Client initialization
client := whatsmeow.NewClient(deviceStore, logger)

// Event handler registration
client.AddEventHandler(func(evt interface{}) {
    switch v := evt.(type) {
    case *events.Message:
        handleMessage(v)
    case *events.HistorySync:
        handleHistorySync(v)
    }
})

// Message sending
_, err := client.SendMessage(context.Background(), recipientJID, &waProto.Message{
    Conversation: proto.String(messageText),
})
```

## Error Handling

### Error Categories

#### 1. Database Errors
```python
class DatabaseError(Exception):
    """Raised when SQLite operations fail."""
    pass

def handle_database_error(func):
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except sqlite3.Error as e:
            logger.error(f"Database error: {e}")
            raise DatabaseError(f"Database operation failed: {e}")
    return wrapper
```

#### 2. WhatsApp API Errors
```go
// Error handling in Go bridge
func sendWhatsAppMessage(client *whatsmeow.Client, recipient, message string) (bool, string) {
    recipientJID, err := parseRecipient(recipient)
    if err != nil {
        return false, fmt.Sprintf("Invalid recipient: %v", err)
    }
    
    _, err = client.SendMessage(context.Background(), recipientJID, messageProto)
    if err != nil {
        logger.Warnf("Failed to send message: %v", err)
        return false, fmt.Sprintf("Send failed: %v", err)
    }
    
    return true, "Message sent successfully"
}
```

#### 3. Network Errors
```python
def call_whatsapp_api(endpoint: str, data: Dict) -> Tuple[bool, str]:
    try:
        response = requests.post(f"{WHATSAPP_API_BASE_URL}{endpoint}", 
                               json=data, timeout=30)
        response.raise_for_status()
        return True, response.json()
    
    except requests.ConnectionError:
        return False, "WhatsApp bridge not reachable"
    except requests.Timeout:
        return False, "Request timeout"
    except requests.HTTPError as e:
        return False, f"HTTP error: {e.response.status_code}"
```

### Recovery Mechanisms

#### Connection Recovery
```go
// Auto-reconnection logic
func maintainConnection(client *whatsmeow.Client) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for range ticker.C {
        if !client.IsConnected() {
            logger.Warn("Connection lost, attempting to reconnect...")
            err := client.Connect()
            if err != nil {
                logger.Errorf("Reconnection failed: %v", err)
            }
        }
    }
}
```

## Performance Specifications

### Throughput Requirements
| Operation | Target | Maximum |
|-----------|--------|---------|
| Message retrieval | 100 messages/sec | 500 messages/sec |
| Message sending | 10 messages/sec | 20 messages/sec |
| Contact search | 50 queries/sec | 100 queries/sec |
| Database queries | 200 queries/sec | 1000 queries/sec |

### Latency Requirements
| Operation | Target | Maximum |
|-----------|--------|---------|
| MCP tool call | < 100ms | < 500ms |
| Database query | < 50ms | < 200ms |
| WhatsApp API call | < 1s | < 5s |
| Message sync | < 10s | < 30s |

### Optimization Strategies

#### Database Optimization
```sql
-- Query optimization with proper indexing
EXPLAIN QUERY PLAN 
SELECT * FROM messages 
WHERE chat_jid = ? 
AND timestamp BETWEEN ? AND ? 
ORDER BY timestamp DESC 
LIMIT 20;

-- Result: Uses index idx_messages_chat_jid and idx_messages_timestamp
```

#### Memory Management
```go
// Connection pooling for SQLite
func NewMessageStore() (*MessageStore, error) {
    db, err := sql.Open("sqlite3", connectionString)
    if err != nil {
        return nil, err
    }
    
    // Configure connection pool
    db.SetMaxOpenConns(1)      // SQLite works best with single connection
    db.SetMaxIdleConns(1)
    db.SetConnMaxLifetime(time.Hour)
    
    return &MessageStore{db: db}, nil
}
```

## Security Implementation

### Data Protection
```go
// File permissions for database files
func createSecureFile(path string) error {
    file, err := os.OpenFile(path, os.O_CREATE|os.O_WRONLY, 0600)
    if err != nil {
        return err
    }
    defer file.Close()
    return nil
}
```

### Session Management
```go
// Secure session storage (handled by whatsmeow)
type DeviceStore struct {
    container *sqlstore.Container
}

// Session encryption is handled internally by the library
// Keys are stored encrypted in whatsapp.db
```

### Input Validation
```python
def validate_phone_number(phone: str) -> bool:
    """Validate phone number format."""
    import re
    pattern = r'^\d{10,15}$'  # 10-15 digits
    return bool(re.match(pattern, phone))

def sanitize_query(query: str) -> str:
    """Sanitize search query to prevent SQL injection."""
    return query.replace("'", "''").replace(";", "")
```

## Configuration Management

### Environment Configuration
```python
# Configuration constants
MESSAGES_DB_PATH = os.path.join(
    os.path.dirname(os.path.abspath(__file__)), 
    '..', 'whatsapp-bridge', 'store', 'messages.db'
)

WHATSAPP_API_BASE_URL = "http://localhost:8080/api"

# Configurable timeouts
DEFAULT_TIMEOUT = 30  # seconds
MAX_MESSAGE_LENGTH = 4096  # characters
DEFAULT_PAGE_SIZE = 20  # messages per page
```

### Go Configuration
```go
// Configuration constants
const (
    DefaultPort = 8080
    DefaultTimeout = 30 * time.Second
    MaxMessageSize = 4096
    DatabasePath = "store/messages.db"
    SessionPath = "store/whatsapp.db"
)
```

### Runtime Configuration
```json
{
  "bridge": {
    "port": 8080,
    "timeout": "30s",
    "log_level": "INFO"
  },
  "mcp": {
    "max_results": 100,
    "default_page_size": 20,
    "enable_context": true
  }
}
```

This technical specification provides the detailed implementation guidelines and requirements for building and maintaining the WhatsApp MCP Server system. 