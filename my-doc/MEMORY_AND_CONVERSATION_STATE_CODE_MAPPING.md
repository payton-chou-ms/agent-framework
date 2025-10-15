# Memory & Conversation State - Code Mapping

This document provides a visual mapping between the architectural diagram and the actual code implementation.

## Diagram to Code Mapping

### Diagram Component: "Thread (Travel Planning)"

**Maps to**: `AgentThread` class in `_threads.py`

```python
# Location: python/packages/core/agent_framework/_threads.py:292

class AgentThread:
    """
    The Agent thread class represents the conversation thread shown in the diagram.
    It maintains conversation state and message history for an agent interaction.
    """
    def __init__(
        self,
        *,
        service_thread_id: str | None = None,
        message_store: ChatMessageStoreProtocol | None = None,
        context_provider: AggregateContextProvider | None = None,
    ):
        self._service_thread_id = service_thread_id
        self._message_store = message_store
        self.context_provider = context_provider
```

**Usage Example**:
```python
# Create a thread (like "Travel Planning" in the diagram)
thread = agent.get_new_thread()
```

---

### Diagram Component: "User's message" & "Agent's message"

**Maps to**: `ChatMessage` objects stored in `ChatMessageStore`

```python
# Location: python/packages/core/agent_framework/_threads.py:185

class ChatMessageStore:
    """Stores the conversation messages shown in the diagram"""
    def __init__(self, messages: Sequence[ChatMessage] | None = None):
        self.messages = list(messages) if messages else []
    
    async def add_messages(self, messages: Sequence[ChatMessage]) -> None:
        """Add user and agent messages to the conversation"""
        self.messages.extend(messages)
    
    async def list_messages(self) -> list[ChatMessage]:
        """Get all messages in chronological order (oldest first)"""
        return self.messages
```

**Conversation Flow**:
```python
# User's message: "I need to book a hotel in New York for 2 stays."
await agent.run("I need to book a hotel in New York for 2 stays.", thread=thread)

# Agent's message: "Here are some suggestions:"
# (Stored automatically in thread's message_store)

# User's message: "What's the daily meal allowance for the business trip?"
await agent.run("What's the daily meal allowance for the business trip?", thread=thread)

# All messages are retained in thread.message_store.messages
```

---

### Diagram Component: "Run 1" & "Run 2"

**Maps to**: `agent.run()` method calls with context provider invocations

```python
# Each "Run" in the diagram corresponds to one agent.run() call

# Run 1: Use Tripadvisor API to search the nearest hotel
result1 = await agent.run(
    "I need to book a hotel in New York for 2 stays.", 
    thread=thread
)
# Behind the scenes:
# 1. context_provider.invoking() - Load "Trip to New York" from memory
# 2. Agent processes with thread context + long-term memory
# 3. context_provider.invoked() - Update memory if needed

# Run 2: Use Microsoft SharePoint to query the company travel policy
result2 = await agent.run(
    "What's the daily meal allowance for the business trip?", 
    thread=thread
)
# Behind the scenes:
# 1. context_provider.invoking() - Load relevant memories
# 2. Agent processes with accumulated thread context + long-term memory
# 3. context_provider.invoked() - Store "Daily meal allowance" to memory
```

---

### Diagram Component: "Memory" (Right side)

**Maps to**: `ContextProvider` implementations

```python
# Location: python/packages/core/agent_framework/_memory.py:74

class ContextProvider(ABC):
    """
    Base class for long-term memory providers.
    The "Memory" box in the diagram represents persistent storage
    accessed through ContextProvider implementations.
    """
    
    async def thread_created(self, thread_id: str | None) -> None:
        """
        Called when thread is created.
        Load relevant long-term memories (e.g., "Trip to New York")
        """
        pass
    
    @abstractmethod
    async def invoking(self, messages, **kwargs) -> Context:
        """
        Called before each Run (Run 1, Run 2, etc.).
        Returns additional context from long-term memory.
        """
        pass
    
    async def invoked(
        self, 
        request_messages, 
        response_messages, 
        **kwargs
    ) -> None:
        """
        Called after each Run.
        Updates long-term memory with new information
        (e.g., store "Daily meal allowance").
        """
        pass
```

**Concrete Implementations**:

1. **Mem0Provider** (for "Trip to New York", "Daily meal allowance"):
```python
# Location: python/packages/mem0/agent_framework/mem0/_provider.py

agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=False  # Global memory
    )
)

# Run 1 stores: "Trip to New York"
# Run 2 stores: "Daily meal allowance"
# Both are persisted in Mem0 and retrieved in future runs
```

2. **RedisProvider** (alternative long-term storage):
```python
# Location: python/packages/redis/agent_framework/redis/_provider.py

agent = ChatAgent(
    client=chat_client,
    context_providers=RedisProvider(
        user_id="user123",
        redis_url="redis://localhost:6379"
    )
)
```

---

### Diagram Component: Memory Arrows (⬇️)

**Maps to**: Data flow in ContextProvider lifecycle

```python
# Arrow from Run 1 to Memory ("Trip to New York")
# This happens in ContextProvider.invoked()

class Mem0Provider(ContextProvider):
    async def invoked(
        self, 
        request_messages, 
        response_messages, 
        **kwargs
    ):
        """
        After Run 1 completes, this method:
        1. Extracts information: "Trip to New York"
        2. Stores it in Mem0 API
        3. Makes it available for future runs
        """
        await self._client.add_memories(
            messages=[{"role": "user", "content": "Trip to New York"}],
            user_id=self._user_id
        )

# Arrow from Memory to Run 2 (retrieving "Daily meal allowance")
# This happens in ContextProvider.invoking()

class Mem0Provider(ContextProvider):
    async def invoking(self, messages, **kwargs) -> Context:
        """
        Before Run 2 starts, this method:
        1. Queries Mem0 for relevant memories
        2. Retrieves: "Trip to New York"
        3. Adds it to the context for the agent
        """
        memories = await self._client.search_memories(
            query="travel, trip",
            user_id=self._user_id
        )
        
        # Format memories as additional instructions
        instructions = "## Memories\n"
        for memory in memories:
            instructions += f"- {memory['text']}\n"
        
        return Context(instructions=instructions)
```

---

## Complete Code Flow

Here's how the diagram translates to actual code execution:

### Setup Phase

```python
# Step 1: Create agent with long-term memory provider
from agent_framework.mem0 import Mem0Provider

mem0_provider = Mem0Provider(user_id="user123")

agent = ChatAgent(
    client=chat_client,
    name="TravelAssistant",
    context_providers=mem0_provider
)

# Step 2: Create thread (represents "Thread: Travel Planning")
thread = agent.get_new_thread()
```

### Run 1 Execution

```python
# User's message: "I need to book a hotel in New York for 2 stays."
result1 = await agent.run(
    "I need to book a hotel in New York for 2 stays.",
    thread=thread
)

# Behind the scenes:
# 1. thread.on_new_messages() - Add user message to short-term memory
# 2. mem0_provider.invoking() - Load long-term memories (empty on first run)
# 3. Agent processes:
#    - Short-term: ["User: I need to book a hotel..."]
#    - Long-term: []
# 4. Agent response: "Here are some suggestions:"
# 5. thread.on_new_messages() - Add agent message to short-term memory
# 6. mem0_provider.invoked() - Store "Trip to New York" in long-term memory

# Thread now contains:
# - User's message
# - Agent's message
# Memory now contains:
# - "Trip to New York"
```

### Run 2 Execution

```python
# User's message: "What's the daily meal allowance for the business trip?"
result2 = await agent.run(
    "What's the daily meal allowance for the business trip?",
    thread=thread
)

# Behind the scenes:
# 1. thread.on_new_messages() - Add user message to short-term memory
# 2. mem0_provider.invoking() - Load long-term memories:
#    → Retrieves: "Trip to New York"
# 3. Agent processes:
#    - Short-term: [
#        "User: I need to book a hotel...",
#        "Agent: Here are some suggestions...",
#        "User: What's the daily meal allowance..."
#      ]
#    - Long-term: ["Trip to New York"]
# 4. Agent response: "The daily allowance for your business trip is $75..."
# 5. thread.on_new_messages() - Add agent message to short-term memory
# 6. mem0_provider.invoked() - Store "Daily meal allowance" in long-term memory

# Thread now contains:
# - User's message (Run 1)
# - Agent's message (Run 1)
# - User's message (Run 2)
# - Agent's message (Run 2)
# Memory now contains:
# - "Trip to New York"
# - "Daily meal allowance"
```

---

## Code File Locations

### Core Components

| Diagram Component | Code Location | Key Class/Function |
|-------------------|---------------|-------------------|
| Thread | `python/packages/core/agent_framework/_threads.py` | `AgentThread` (line 292) |
| User's/Agent's messages | `python/packages/core/agent_framework/_threads.py` | `ChatMessageStore` (line 185) |
| Memory (base) | `python/packages/core/agent_framework/_memory.py` | `ContextProvider` (line 74) |
| Memory (Mem0) | `python/packages/mem0/agent_framework/mem0/_provider.py` | `Mem0Provider` |
| Memory (Redis) | `python/packages/redis/agent_framework/redis/_provider.py` | `RedisProvider` |
| Context aggregation | `python/packages/core/agent_framework/_memory.py` | `AggregateContextProvider` (line 189) |

### Example Implementations

| Example | Location | Description |
|---------|----------|-------------|
| Thread suspend/resume | `python/samples/getting_started/threads/suspend_resume_thread.py` | Shows thread serialization (short-term persistence) |
| Custom message store | `python/samples/getting_started/threads/custom_chat_message_store_thread.py` | Custom storage for thread messages |
| Mem0 global scope | `python/samples/getting_started/context_providers/mem0/mem0_threads.py` | Long-term memory shared across threads |
| Mem0 thread scope | `python/samples/getting_started/context_providers/mem0/mem0_threads.py` | Long-term memory isolated per thread |
| Redis storage | `python/samples/getting_started/context_providers/redis/redis_threads.py` | Searchable long-term memory |

---

## Short-term vs Long-term Memory

### Short-term Memory (Session-scoped, Immediate Context)

**Characteristics**:
- Stored in `AgentThread.message_store`
- Contains conversation messages for current session
- Cleared when thread ends (unless serialized)
- Used for maintaining conversation flow

**Code**:
```python
# Location: python/packages/core/agent_framework/_threads.py:401

class AgentThread:
    async def on_new_messages(self, new_messages: ChatMessage | Sequence[ChatMessage]):
        """Stores messages in short-term memory (this session only)"""
        if self._message_store is None:
            self._message_store = ChatMessageStore()
        
        if isinstance(new_messages, ChatMessage):
            new_messages = [new_messages]
        
        await self._message_store.add_messages(new_messages)
```

### Long-term Memory (Persistent, User Facts)

**Characteristics**:
- Stored via `ContextProvider` (Mem0, Redis, etc.)
- Contains user preferences, facts, historical data
- Persists across sessions and threads
- Used for personalization and context continuity

**Code**:
```python
# Location: python/packages/core/agent_framework/_memory.py:119-138

class ContextProvider(ABC):
    async def invoked(
        self,
        request_messages: ChatMessage | Sequence[ChatMessage],
        response_messages: ChatMessage | Sequence[ChatMessage] | None = None,
        **kwargs
    ):
        """
        Called after agent response.
        Implementations use this to store information in long-term memory.
        
        Example: After Run 1, Mem0Provider extracts "Trip to New York"
        and stores it in Mem0 API for future retrieval.
        """
        pass
```

---

## Summary Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DIAGRAM → CODE MAPPING                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Thread (Travel Planning)    →    AgentThread                       │
│  - User's message            →    ChatMessage in ChatMessageStore   │
│  - Agent's message           →    ChatMessage in ChatMessageStore   │
│                                                                      │
│  Run 1                       →    await agent.run(...)              │
│  - Step 1: Use API           →    ContextProvider.invoking()        │
│  - Step 2: Create message    →    Agent processing                  │
│                                                                      │
│  Run 2                       →    await agent.run(...)              │
│  - Step 1: Use SharePoint    →    ContextProvider.invoking()        │
│  - Step 2: Create message    →    Agent processing                  │
│                                                                      │
│  Memory                      →    ContextProvider implementation    │
│  - Trip to New York          →    Mem0Provider / RedisProvider      │
│  - Daily meal allowance      →    Stored via invoked() method       │
│                                                                      │
│  Memory Arrows (⬇️)          →    Data flow:                        │
│  - To Memory                 →    ContextProvider.invoked()         │
│  - From Memory               →    ContextProvider.invoking()        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

1. **Thread = AgentThread**: The conversation container with short-term message history
2. **Messages = ChatMessageStore**: The list of user and agent messages within the thread
3. **Run = agent.run()**: Each execution that combines short-term + long-term context
4. **Memory = ContextProvider**: Long-term storage accessed via invoking() and updated via invoked()
5. **Short-term**: Conversation messages in current session (AgentThread)
6. **Long-term**: User facts persisted across sessions (ContextProvider implementations)

The architecture enables agents to:
- Remember the conversation flow within a session (short-term)
- Remember user facts and preferences across sessions (long-term)
- Combine both types of memory for intelligent, context-aware responses
