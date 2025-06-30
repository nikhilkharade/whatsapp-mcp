# WhatsApp MCP Server - Documentation Index

This repository contains comprehensive documentation for the WhatsApp MCP Server project. The documentation is organized into several specialized documents covering different aspects of the system.

## 📋 Documentation Overview

### 🏗️ [Architecture Documentation](./ARCHITECTURE.md)
**Complete system architecture covering HLD, LLD, and implementation details**

**Contents:**
- High Level Design (HLD) with system architecture diagrams
- Low Level Design (LLD) with component specifications
- Database schema and relationships
- API documentation for all MCP tools
- Security considerations and implementation
- Deployment architecture and system requirements

**Best for:** Understanding the overall system design, component interactions, and architectural decisions.

### 🔄 [Flow Diagrams](./FLOW_DIAGRAMS.md)
**Detailed process flows and interaction diagrams**

**Contents:**
- System initialization and authentication flows
- Message sending and retrieval processes
- WhatsApp event processing and data synchronization
- Error handling and recovery mechanisms
- State management and transaction flows

**Best for:** Understanding how data flows through the system and how different components interact during various operations.

### ⚙️ [Technical Specification](./TECHNICAL_SPECIFICATION.md)
**Detailed implementation guidelines and technical requirements**

**Contents:**
- System and hardware requirements
- Technology stack and dependencies
- Data models and API specifications
- Performance requirements and optimization strategies
- Security implementation details
- Configuration management

**Best for:** Implementation details, performance tuning, and technical configuration.

## 🚀 Quick Start Guide

### For Developers
1. Start with [Architecture Documentation](./ARCHITECTURE.md) to understand the system design
2. Review [Flow Diagrams](./FLOW_DIAGRAMS.md) to understand operational processes
3. Consult [Technical Specification](./TECHNICAL_SPECIFICATION.md) for implementation details

### For System Architects
1. Focus on the HLD section in [Architecture Documentation](./ARCHITECTURE.md)
2. Review security considerations and deployment architecture
3. Examine flow diagrams for system integration patterns

### For DevOps/Deployment
1. Check system requirements in [Technical Specification](./TECHNICAL_SPECIFICATION.md)
2. Review deployment architecture in [Architecture Documentation](./ARCHITECTURE.md)
3. Understand configuration management and monitoring requirements

## 📊 System at a Glance

### Core Components
```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   AI Assistant  │    │  MCP Server     │    │ WhatsApp Bridge │
│   (Claude)      │◄──►│  (Python)       │◄──►│    (Go)         │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Key Features
- **🔐 Secure Local Processing**: All data stays on your local machine
- **🤖 AI Integration**: Seamless integration with Claude and other AI assistants
- **📱 Real-time Messaging**: Bidirectional WhatsApp communication
- **🔍 Advanced Search**: Powerful message and contact search capabilities
- **📊 Message Context**: Retrieve conversation context and history

### Technology Stack
- **Backend**: Go 1.24+ (WhatsApp Bridge)
- **API Layer**: Python 3.11+ (MCP Server)
- **Database**: SQLite 3.x
- **Protocol**: MCP (Model Context Protocol)
- **WhatsApp API**: whatsmeow library

## 🔧 Available MCP Tools

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `search_contacts` | Find contacts by name/phone | `query` |
| `list_messages` | Retrieve messages with filters | `date_range`, `sender`, `chat_jid`, `query` |
| `list_chats` | Get available chats | `query`, `sort_by` |
| `get_chat` | Get specific chat details | `chat_jid` |
| `send_message` | Send WhatsApp message | `recipient`, `message` |
| `get_message_context` | Get context around message | `message_id`, `before`, `after` |

## 📁 Project Structure

```
whatsapp-mcp/
├── whatsapp-bridge/          # Go WhatsApp integration
│   ├── main.go              # Bridge entry point
│   ├── go.mod               # Go dependencies
│   └── store/               # SQLite databases
│       ├── messages.db      # Message history
│       └── whatsapp.db     # Session data
├── whatsapp-mcp-server/     # Python MCP server
│   ├── main.py             # MCP entry point
│   ├── whatsapp.py         # Business logic
│   └── pyproject.toml      # Python dependencies
└── docs/                   # Documentation
    ├── ARCHITECTURE.md
    ├── FLOW_DIAGRAMS.md
    └── TECHNICAL_SPECIFICATION.md
```

## 🗃️ Database Schema Summary

### messages.db
- **chats**: Chat metadata (JID, name, last activity)
- **messages**: Message content (ID, chat, sender, content, timestamp)

### whatsapp.db (managed by whatsmeow)
- **Device session**: Authentication and encryption keys
- **Contacts**: Contact information and metadata
- **Chat settings**: Preferences and configuration

## 🔒 Security Highlights

- **Local Storage Only**: All messages stored locally in SQLite
- **No Cloud Dependencies**: No external services required
- **Encrypted Sessions**: WhatsApp sessions encrypted by whatsmeow
- **Input Validation**: Comprehensive input sanitization
- **File Permissions**: Secure database file access

## 📈 Performance Characteristics

### Throughput
- Message retrieval: 100+ messages/sec
- Contact search: 50+ queries/sec  
- Database operations: 200+ queries/sec

### Latency
- MCP tool calls: < 100ms typical
- Database queries: < 50ms typical
- WhatsApp API calls: < 1s typical

## 🚨 Key Considerations

### Setup Requirements
1. **WhatsApp Mobile**: Required for QR code authentication
2. **Go with CGO**: Enabled for SQLite support (especially on Windows)
3. **Python 3.11+**: For MCP server compatibility
4. **UV Package Manager**: For Python dependency management

### Operational Notes
- Sessions expire after ~20 days and require re-authentication
- Initial history sync may take several minutes for large message histories
- Bridge must run continuously to receive real-time messages
- MCP server starts on-demand when Claude/Cursor connects

## 📚 Additional Resources

### External Documentation
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/)
- [whatsmeow Library](https://github.com/tulir/whatsmeow)
- [Claude Desktop Configuration](https://docs.anthropic.com/claude/desktop)
- [SQLite Documentation](https://sqlite.org/docs.html)

### Development Tools
- [Go Development](https://golang.org/doc/)
- [Python FastMCP](https://github.com/anthropics/mcp)
- [SQLite Browser](https://sqlitebrowser.org/)

---

## 📞 Support

For technical issues or questions:
1. Check the relevant documentation section above
2. Review the troubleshooting section in [Architecture Documentation](./ARCHITECTURE.md)
3. Examine error handling patterns in [Flow Diagrams](./FLOW_DIAGRAMS.md)
4. Consult performance optimization in [Technical Specification](./TECHNICAL_SPECIFICATION.md)

This documentation provides a complete technical reference for understanding, implementing, and maintaining the WhatsApp MCP Server system. 