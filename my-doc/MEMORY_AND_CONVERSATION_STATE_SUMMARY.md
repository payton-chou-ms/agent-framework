# Memory & Conversation State - Code Summary

This document provides a comprehensive summary of the code implementation related to the Memory & Conversation State architecture shown in the diagram.

## Architecture Overview

The Microsoft Agent Framework provides two types of memory to maintain conversation context:

1. **Short-term Memory (Session-scoped)**: Immediate conversation context managed through `AgentThread`
2. **Long-term Memory (Persistent)**: User preferences, facts, and historical data managed through `ContextProvider` integrations (e.g., Mem0, Redis)

## Core Components

### 1. AgentThread - Short-term Memory

**Location**: `python/packages/core/agent_framework/_threads.py`

The `AgentThread` class represents a conversation thread that maintains session-scoped conversation history across turns within a single session.

#### Key Classes

```python
class AgentThread:
    """
    The Agent thread class represents both locally managed and service-managed threads.
    Maintains conversation state and message history for an agent interaction.
    """
    def __init__(
        self,
        *,
        service_thread_id: str | None = None,
        message_store: ChatMessageStoreProtocol | None = None,
        context_provider: AggregateContextProvider | None = None,
    ):
        # Either service_thread_id OR message_store may be set, but not both
        self._service_thread_id = service_thread_id
        self._message_store = message_store
        self.context_provider = context_provider
```

#### Storage Options

The framework supports two storage mechanisms for threads:

1. **Service-Managed Threads** (`service_thread_id`):
   - Thread managed by the AI service provider (e.g., OpenAI Assistants API)
   - Thread ID stored locally, messages stored remotely
   - Example: `thread = agent.get_new_thread(service_thread_id="thread_abc123")`

2. **Local Message Store** (`ChatMessageStore`):
   - Messages stored locally in-memory or custom storage
   - Full control over message persistence
   - Example: `agent = ChatAgent(chat_message_store_factory=ChatMessageStore)`

#### Code Example: Thread Management

```python
# From: python/samples/getting_started/threads/suspend_resume_thread.py

# Create a thread and have a conversation
thread = agent.get_new_thread()
await agent.run("Tell me a joke about a pirate.", thread=thread)

# Serialize thread state for persistence
serialized_thread = await thread.serialize()
# Save to database, file, etc.

# Later: Deserialize and resume
resumed_thread = await agent.deserialize_thread(serialized_thread)
await agent.run("Tell the same joke in pirate voice", thread=resumed_thread)
```

#### Key Methods

```python
class AgentThread:
    async def on_new_messages(self, new_messages: ChatMessage | Sequence[ChatMessage]):
        """Invoked when new messages are added to the conversation"""
        
    async def serialize(self) -> dict[str, Any]:
        """Serializes the thread state for persistence"""
        
    @classmethod
    async def deserialize(cls, serialized_thread_state, message_store=None):
        """Deserializes state from a dictionary into a new AgentThread"""
```

---

### 2. ChatMessageStore - Message Persistence

**Location**: `python/packages/core/agent_framework/_threads.py`

```python
class ChatMessageStore:
    """
    In-memory implementation of ChatMessageStoreProtocol.
    Stores messages in a list with support for serialization.
    """
    def __init__(self, messages: Sequence[ChatMessage] | None = None):
        self.messages = list(messages) if messages else []
    
    async def add_messages(self, messages: Sequence[ChatMessage]):
        """Add messages to the store"""
        
    async def list_messages(self) -> list[ChatMessage]:
        """Get all messages in chronological order"""
        
    async def serialize(self) -> dict[str, Any]:
        """Serialize for persistence"""
        
    @classmethod
    async def deserialize(cls, serialized_store_state):
        """Create store from serialized state"""
```

#### Custom Storage Example

```python
# From: python/samples/getting_started/threads/custom_chat_message_store_thread.py

class CustomChatMessageStore:
    """Custom implementation for database/file storage"""
    def __init__(self):
        self._messages = []
    
    async def list_messages(self) -> list[ChatMessage]:
        return self._messages
    
    async def add_messages(self, messages: Sequence[ChatMessage]):
        self._messages.extend(messages)
        # Could save to database here
```

---

### 3. ContextProvider - Long-term Memory

**Location**: `python/packages/core/agent_framework/_memory.py`

The `ContextProvider` abstract base class enables long-term memory capabilities by providing additional context to the AI model before each invocation.

```python
class ContextProvider(ABC):
    """
    Base class for context providers.
    Used to enhance AI's context management with long-term memory.
    """
    
    async def thread_created(self, thread_id: str | None):
        """Called when a new thread is created - load relevant long-term data"""
        
    @abstractmethod
    async def invoking(self, messages, **kwargs) -> Context:
        """
        Called before model invocation.
        Returns Context with instructions, messages, and tools.
        """
        
    async def invoked(self, request_messages, response_messages, **kwargs):
        """Called after model response - update long-term memory"""
```

#### Context Object

```python
class Context:
    """Contains context provided to the AI model by a ContextProvider"""
    def __init__(
        self,
        instructions: str | None = None,
        messages: Sequence[ChatMessage] | None = None,
        tools: Sequence[ToolProtocol] | None = None,
    ):
        self.instructions = instructions  # Additional instructions
        self.messages = messages or []    # Additional messages
        self.tools = tools or []          # Additional tools
```

---

### 4. AggregateContextProvider - Multiple Memory Sources

**Location**: `python/packages/core/agent_framework/_memory.py`

```python
class AggregateContextProvider(ContextProvider):
    """
    Combines multiple context providers.
    Aggregates responses from all providers before returning.
    """
    def __init__(self, context_providers):
        self.providers = context_providers
    
    async def invoking(self, messages, **kwargs) -> Context:
        # Gather context from all providers
        contexts = await asyncio.gather(
            *[provider.invoking(messages) for provider in self.providers]
        )
        # Combine all instructions, messages, and tools
        return Context(
            instructions=combined_instructions,
            messages=combined_messages,
            tools=combined_tools
        )
```

---

## Implementation Examples

### Example 1: Mem0 Provider - Long-term Memory

**Location**: `python/samples/getting_started/context_providers/mem0/mem0_threads.py`

Mem0 provides persistent memory across conversation sessions with different scoping strategies:

```python
# Global Scope: Memories shared across all threads
agent = AzureAIAgentClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        thread_id=global_thread_id,
        scope_to_per_operation_thread_id=False  # Share across threads
    )
)

# Store preference
await agent.run("Remember that I prefer technical responses")

# New thread - but memory persists
new_thread = agent.get_new_thread()
await agent.run("What are my preferences?", thread=new_thread)
# Agent remembers the preference from previous thread
```

```python
# Thread Scope: Memories isolated per thread
agent = AzureAIAgentClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=True  # Isolate per thread
    )
)

thread = agent.get_new_thread()
await agent.run("I'm working on a Python project", thread=thread)
# Memory only accessible within this specific thread
```

### Example 2: Redis Provider - Searchable Memory

**Location**: `python/samples/getting_started/context_providers/redis/redis_threads.py`

Redis provider enables persistent, searchable memory with full-text search:

```python
from agent_framework.redis import RedisProvider

redis_provider = RedisProvider(
    user_id="user123",
    redis_url="redis://localhost:6379"
)

agent = ChatAgent(
    client=chat_client,
    context_providers=redis_provider
)

# Memories are stored in Redis and can be searched
await agent.run("I prefer dark mode interfaces")
await agent.run("My favorite color is blue")

# Later sessions can retrieve these memories
await agent.run("What do you know about my preferences?")
```

---

## Memory Flow Diagram Mapping

Based on the image provided, here's how the code maps to the architectural concepts:

### Thread (Travel Planning)
- **Code**: `AgentThread` class
- **Purpose**: Maintains conversation context within a session
- **Storage**: `ChatMessageStore` or service-managed

### Run 1 & Run 2
- **Code**: `agent.run()` method calls
- **Purpose**: Each invocation of the agent
- **Flow**:
  1. ContextProvider.invoking() - Load long-term memory
  2. Agent processes with short-term context (thread messages)
  3. ContextProvider.invoked() - Update long-term memory

### Memory (Right side)
- **Code**: `ContextProvider` implementations (Mem0, Redis)
- **Purpose**: Long-term persistent storage
- **Examples**: 
  - "Trip to New York" stored via Mem0Provider
  - "Daily meal allowance" stored via RedisProvider

### Short-term Memory
- **Code**: `ChatMessageStore` within `AgentThread`
- **Scope**: Session-scoped, immediate context
- **Content**: User and agent messages in current conversation

### Long-term Memory
- **Code**: `ContextProvider` integrations (Mem0, Redis, custom)
- **Scope**: Persistent across sessions and threads
- **Content**: User preferences, facts, historical data

---

## Code Architecture Summary

```
┌─────────────────────────────────────────────────────────────┐
│                        Agent                                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           AgentThread (Short-term Memory)            │  │
│  │  - ChatMessageStore (conversation messages)          │  │
│  │  - AggregateContextProvider (long-term access)       │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           │ agent.run()                      │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         1. ContextProvider.invoking()                │  │
│  │            (Load long-term memories)                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │    2. Agent processes with combined context:         │  │
│  │       - Short-term: thread messages                  │  │
│  │       - Long-term: provider instructions/messages    │  │
│  └──────────────────────────────────────────────────────┘  │
│                           │                                  │
│                           ▼                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         3. ContextProvider.invoked()                 │  │
│  │            (Update long-term memories)               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘

External Storage:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Mem0 API      │  │  Redis Server   │  │  Custom DB      │
│ (long-term)     │  │ (long-term)     │  │ (long-term)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

---

## Key Files Reference

### Core Implementation
- `python/packages/core/agent_framework/_threads.py` - AgentThread, ChatMessageStore
- `python/packages/core/agent_framework/_memory.py` - ContextProvider, Context
- `python/packages/core/agent_framework/exceptions.py` - AgentThreadException

### Context Provider Implementations
- `python/packages/mem0/agent_framework/mem0/_provider.py` - Mem0Provider
- `python/packages/redis/agent_framework/redis/_provider.py` - RedisProvider

### Examples
- `python/samples/getting_started/threads/suspend_resume_thread.py` - Thread serialization
- `python/samples/getting_started/threads/custom_chat_message_store_thread.py` - Custom storage
- `python/samples/getting_started/context_providers/mem0/mem0_threads.py` - Long-term memory scoping
- `python/samples/getting_started/context_providers/redis/redis_threads.py` - Searchable memory

---

## Summary

The Microsoft Agent Framework provides a comprehensive memory architecture:

1. **AgentThread** manages short-term, session-scoped conversation history
2. **ChatMessageStore** provides pluggable storage for thread messages
3. **ContextProvider** enables long-term, persistent memory across sessions
4. **AggregateContextProvider** combines multiple memory sources

This architecture allows agents to:
- Maintain immediate conversation context within a session (short-term)
- Remember user preferences and facts across sessions (long-term)
- Combine multiple memory sources for rich context
- Serialize and resume conversations seamlessly
- Scale from simple in-memory to production databases (Redis, Mem0, custom)

The diagram in the issue illustrates this two-tier memory system where:
- **Thread** contains the conversation flow (short-term)
- **Memory** contains persistent facts (long-term)
- **Runs** represent agent invocations that combine both memory types
