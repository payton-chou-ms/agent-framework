# Memory & Conversation State Documentation

This directory contains comprehensive documentation for the Memory & Conversation State architecture in the Microsoft Agent Framework.

## Overview

The Microsoft Agent Framework provides a two-tier memory architecture for building intelligent agents:

1. **Short-term Memory** (Session-scoped): Manages immediate conversation context within a session
2. **Long-term Memory** (Persistent): Stores user preferences, facts, and historical data across sessions

## Documentation Files

### 1. [MEMORY_AND_CONVERSATION_STATE_SUMMARY.md](./MEMORY_AND_CONVERSATION_STATE_SUMMARY.md)
**Comprehensive technical summary of the memory architecture**

This document provides:
- Detailed explanation of core components (`AgentThread`, `ChatMessageStore`, `ContextProvider`)
- Code examples from the repository
- Architecture diagrams
- Implementation details for Mem0 and Redis providers
- Complete code flow examples

**Use this when**: You need a deep technical understanding of how memory works in the framework.

---

### 2. [MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.md](./MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.md)
**Visual mapping between architectural diagram and actual code**

This document provides:
- Direct mapping from diagram components to code classes/methods
- Detailed code flow for each "Run" in the diagram
- File locations for all related code
- Complete execution lifecycle
- Comparison of short-term vs long-term memory

**Use this when**: You want to understand how the architectural diagram translates to actual code implementation.

---

### 3. [MEMORY_QUICK_REFERENCE.md](./MEMORY_QUICK_REFERENCE.md)
**Quick reference guide with common patterns and code snippets**

This document provides:
- Quick overview table of memory types
- Common usage patterns (4 main patterns)
- Copy-paste code snippets
- Lifecycle diagram
- FAQ section
- Use case examples (e-commerce, support, personal assistant)

**Use this when**: You want to quickly implement memory features in your application.

---

## Quick Start

### For Beginners: Start Here

1. Read the **Quick Reference Guide** first: [MEMORY_QUICK_REFERENCE.md](./MEMORY_QUICK_REFERENCE.md)
2. Try **Pattern 1** (Basic Thread) to understand short-term memory
3. Try **Pattern 2** (Thread + Long-term Memory) to add persistence

### For Intermediate Users

1. Review the **Code Mapping** document: [MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.md](./MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.md)
2. Study the complete code flow examples
3. Explore custom implementations (custom message store, custom context provider)

### For Advanced Users

1. Study the **Technical Summary**: [MEMORY_AND_CONVERSATION_STATE_SUMMARY.md](./MEMORY_AND_CONVERSATION_STATE_SUMMARY.md)
2. Review the core implementation files:
   - `python/packages/core/agent_framework/_threads.py`
   - `python/packages/core/agent_framework/_memory.py`
3. Build custom context providers for your specific use case

---

## Architecture Diagram Reference

The documentation is based on the following architectural diagram:

![Memory & Conversation State Architecture](https://github.com/user-attachments/assets/f62b33ee-4b3c-4bb5-b9ca-eaa1e35f1b0f)

**Key Concepts from the Diagram**:

- **Thread**: Represents a conversation with its message history (short-term memory)
- **Run 1, Run 2**: Individual agent invocations that combine short and long-term context
- **Memory**: Long-term persistent storage accessed through context providers
- **Arrows**: Data flow between runs and memory (storing and retrieving information)

---

## Code Locations

### Core Implementation Files

```
python/packages/core/agent_framework/
├── _threads.py                    # AgentThread, ChatMessageStore
├── _memory.py                     # ContextProvider, Context
└── exceptions.py                  # AgentThreadException

python/packages/mem0/agent_framework/mem0/
└── _provider.py                   # Mem0Provider (long-term memory)

python/packages/redis/agent_framework/redis/
└── _provider.py                   # RedisProvider (searchable memory)
```

### Example Files

```
python/samples/getting_started/
├── threads/
│   ├── suspend_resume_thread.py              # Thread serialization
│   ├── custom_chat_message_store_thread.py   # Custom storage
│   └── redis_chat_message_store_thread.py    # Redis-backed threads
└── context_providers/
    ├── mem0/
    │   ├── mem0_basic.py                     # Basic Mem0 usage
    │   └── mem0_threads.py                   # Memory scoping strategies
    └── redis/
        └── redis_threads.py                  # Redis memory examples
```

---

## Key Concepts

### Short-term Memory (Session-scoped)

**Implementation**: `AgentThread` + `ChatMessageStore`

**Characteristics**:
- Stores conversation messages for current session
- Cleared when thread ends (unless serialized)
- Used for maintaining conversation flow
- Fast, in-memory access

**Example**:
```python
thread = agent.get_new_thread()
await agent.run("What is Python?", thread=thread)
await agent.run("Show me an example", thread=thread)  # Remembers "Python"
```

---

### Long-term Memory (Persistent)

**Implementation**: `ContextProvider` (Mem0, Redis, custom)

**Characteristics**:
- Stores user preferences, facts, historical data
- Persists across sessions and threads
- Used for personalization and context continuity
- Integrates with external storage (Mem0 API, Redis, databases)

**Example**:
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(user_id="user123")
)

# Session 1
await agent.run("I prefer dark mode")

# Session 2 (different thread)
await agent.run("What are my preferences?")  # Remembers "dark mode"
```

---

## Common Patterns

### Pattern 1: Basic Thread (Short-term Only)
```python
thread = agent.get_new_thread()
await agent.run("Message 1", thread=thread)
await agent.run("Message 2", thread=thread)  # Has context from Message 1
```

### Pattern 2: Thread + Long-term Memory (Recommended)
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(user_id="user123")
)
thread = agent.get_new_thread()
await agent.run("I like Python", thread=thread)  # Stored in Mem0

# Later session
thread2 = agent.get_new_thread()
await agent.run("What do I like?", thread=thread2)  # Retrieves from Mem0
```

### Pattern 3: Multiple Memory Sources
```python
agent = ChatAgent(
    client=chat_client,
    context_providers=[
        Mem0Provider(user_id="user123"),
        RedisProvider(user_id="user123"),
    ]
)
```

---

## API Reference

### AgentThread

```python
class AgentThread:
    async def on_new_messages(messages: ChatMessage | Sequence[ChatMessage])
    async def serialize() -> dict[str, Any]
    
    @classmethod
    async def deserialize(serialized_thread_state, message_store=None) -> AgentThread
    
    @property
    def service_thread_id -> str | None
    
    @property
    def message_store -> ChatMessageStoreProtocol | None
```

### ContextProvider

```python
class ContextProvider(ABC):
    async def thread_created(thread_id: str | None) -> None
    async def invoking(messages, **kwargs) -> Context
    async def invoked(request_messages, response_messages, **kwargs) -> None
```

### ChatMessageStore

```python
class ChatMessageStore:
    async def add_messages(messages: Sequence[ChatMessage]) -> None
    async def list_messages() -> list[ChatMessage]
    async def serialize() -> dict[str, Any]
    
    @classmethod
    async def deserialize(serialized_store_state) -> ChatMessageStore
```

---

## FAQ

### Q: What's the difference between Thread and Memory?

**Thread** (`AgentThread`):
- Short-term conversation history
- Session-scoped
- Contains user and agent messages
- Temporary (unless serialized)

**Memory** (`ContextProvider`):
- Long-term facts and preferences
- Persistent across sessions
- Contains extracted knowledge
- Permanent

### Q: Which should I use: Mem0 or Redis?

**Mem0Provider**:
- ✅ Managed service (no infrastructure)
- ✅ Automatic memory extraction
- ✅ Easy to get started

**RedisProvider**:
- ✅ Self-hosted (full control)
- ✅ Full-text and vector search
- ✅ High performance

### Q: Can I combine both?

Yes! Use an array of context providers:
```python
context_providers=[Mem0Provider(...), RedisProvider(...)]
```

### Q: How do I persist threads?

```python
# Save
serialized = await thread.serialize()
db.save(serialized)

# Load
thread = await agent.deserialize_thread(db.load())
```

---

## Related Resources

### Microsoft Learn Documentation
- [Agent Framework Overview](https://learn.microsoft.com/agent-framework/overview/agent-framework-overview)
- [User Guide](https://learn.microsoft.com/agent-framework/user-guide/overview)

### Repository Links
- [Python Examples](../../python/samples/getting_started/)
- [.NET Examples](../../dotnet/samples/GettingStarted/)

### External Services
- [Mem0 Platform](https://mem0.ai/)
- [Redis](https://redis.io/)

---

## Feedback and Contributions

If you have questions, feedback, or suggestions for improving this documentation:
- File a [GitHub issue](https://github.com/microsoft/agent-framework/issues)
- Join the [Microsoft Azure AI Foundry Discord](https://discord.gg/b5zjErwbQM)

---

## Document History

- **2025-10-14**: Initial documentation created based on architectural diagram
  - Created comprehensive technical summary
  - Created diagram-to-code mapping
  - Created quick reference guide
  - All documents validated against repository code

---

## Summary

This documentation provides everything you need to understand and implement memory features in the Microsoft Agent Framework:

1. **Three comprehensive documents** covering different levels of detail
2. **Code examples** pulled directly from the repository
3. **Visual mappings** between architecture and code
4. **Common patterns** for quick implementation
5. **FAQ and troubleshooting** for common questions

Start with the Quick Reference Guide for immediate implementation, or dive deep into the Technical Summary for complete understanding.
