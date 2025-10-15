# Memory & Conversation State - Quick Reference Guide

This is a quick reference guide for understanding and using the Memory & Conversation State features in the Microsoft Agent Framework.

## Quick Overview

The framework provides **two types of memory**:

| Memory Type | Implementation | Scope | Use Case | Example |
|------------|----------------|-------|----------|---------|
| **Short-term** | `AgentThread` + `ChatMessageStore` | Session-scoped | Conversation flow | "Tell me about X" → "What else about X?" |
| **Long-term** | `ContextProvider` (Mem0, Redis) | Persistent | User preferences, facts | "I prefer dark mode" (remembered forever) |

## Key Classes & Locations

```
AgentThread               → python/packages/core/agent_framework/_threads.py:292
ChatMessageStore          → python/packages/core/agent_framework/_threads.py:185
ContextProvider           → python/packages/core/agent_framework/_memory.py:74
Mem0Provider             → python/packages/mem0/agent_framework/mem0/_provider.py
RedisProvider            → python/packages/redis/agent_framework/redis/_provider.py
AggregateContextProvider → python/packages/core/agent_framework/_memory.py:189
```

## Common Patterns

### Pattern 1: Basic Thread (Short-term Memory Only)

```python
from agent_framework.openai import OpenAIChatClient

# Create agent
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    instructions="You are a helpful assistant"
)

# Create thread
thread = agent.get_new_thread()

# Conversation (messages stored in thread)
await agent.run("What is Python?", thread=thread)
await agent.run("Show me an example", thread=thread)  # Remembers context

# Serialize for persistence
state = await thread.serialize()
# Save state to database...

# Later: Deserialize and resume
thread = await agent.deserialize_thread(state)
await agent.run("Can you explain more?", thread=thread)
```

**When to use**: 
- Simple conversations within a single session
- No need for cross-session memory
- Want full control over message history

---

### Pattern 2: Thread + Long-term Memory (Recommended)

```python
from agent_framework.openai import OpenAIChatClient
from agent_framework.mem0 import Mem0Provider

# Create agent with long-term memory
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    instructions="You are a helpful assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=False  # Share memories across threads
    )
)

# Session 1
thread1 = agent.get_new_thread()
await agent.run("I prefer dark mode interfaces", thread=thread1)
# → Stored in Mem0 long-term memory

# Session 2 (different thread, but memory persists)
thread2 = agent.get_new_thread()
await agent.run("What are my preferences?", thread=thread2)
# → Agent retrieves "dark mode preference" from Mem0
```

**When to use**:
- Multi-session conversations
- User preferences and personalization
- Cross-thread memory sharing
- Production applications

---

### Pattern 3: Per-Thread Long-term Memory

```python
from agent_framework.mem0 import Mem0Provider

# Create agent with thread-scoped memory
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    context_providers=Mem0Provider(
        user_id="user123",
        scope_to_per_operation_thread_id=True  # Isolate memories per thread
    )
)

# Work thread
work_thread = agent.get_new_thread()
await agent.run("I'm working on Project X", thread=work_thread)
# → Stored in Mem0 with work_thread.id

# Personal thread (different memories)
personal_thread = agent.get_new_thread()
await agent.run("I like hiking", thread=personal_thread)
# → Stored in Mem0 with personal_thread.id

# Memories are isolated between threads
```

**When to use**:
- Separate contexts (work vs personal)
- Multi-tenant applications
- Thread-specific persistence

---

### Pattern 4: Multiple Memory Sources

```python
from agent_framework.mem0 import Mem0Provider
from agent_framework.redis import RedisProvider

# Combine multiple context providers
agent = OpenAIChatClient().create_agent(
    name="Assistant",
    context_providers=[
        Mem0Provider(user_id="user123"),      # User preferences
        RedisProvider(user_id="user123"),     # Searchable facts
    ]
)

# Both providers are invoked before each run
# Both providers can store memories after each run
await agent.run("I prefer Python and like cats", thread=thread)
# → Stored in both Mem0 and Redis
```

**When to use**:
- Multiple memory backends
- Hybrid memory strategies
- Advanced production scenarios

---

## Code Snippets

### Save and Load Thread

```python
# Save thread to database
thread = agent.get_new_thread()
await agent.run("Tell me about cats", thread=thread)

serialized = await thread.serialize()
# serialized = {
#   "chat_message_store_state": {
#     "messages": [
#       {"role": "user", "content": "Tell me about cats"},
#       {"role": "assistant", "content": "Cats are..."}
#     ]
#   }
# }

# Save to database
db.save("thread_123", serialized)

# Later: Load from database
serialized = db.load("thread_123")
thread = await agent.deserialize_thread(serialized)
await agent.run("What else?", thread=thread)  # Context preserved
```

### Custom Message Store

```python
class DatabaseMessageStore:
    """Store messages in a database"""
    
    def __init__(self, db_connection, thread_id):
        self.db = db_connection
        self.thread_id = thread_id
    
    async def list_messages(self) -> list[ChatMessage]:
        # Load from database
        rows = await self.db.query(
            "SELECT * FROM messages WHERE thread_id = ?", 
            self.thread_id
        )
        return [ChatMessage.from_dict(row) for row in rows]
    
    async def add_messages(self, messages: Sequence[ChatMessage]):
        # Save to database
        for msg in messages:
            await self.db.insert(
                "INSERT INTO messages VALUES (?, ?, ?)",
                self.thread_id, msg.role, msg.content
            )
    
    async def serialize(self) -> dict:
        messages = await self.list_messages()
        return {"messages": [msg.to_dict() for msg in messages]}
    
    @classmethod
    async def deserialize(cls, serialized_state):
        # Restore from serialized state
        messages = [ChatMessage.from_dict(m) for m in serialized_state["messages"]]
        store = cls(db_connection=get_db(), thread_id="restored")
        await store.add_messages(messages)
        return store

# Use custom store
agent = ChatAgent(
    client=chat_client,
    chat_message_store_factory=DatabaseMessageStore
)
```

### Custom Context Provider

```python
from agent_framework import ContextProvider, Context

class CustomMemoryProvider(ContextProvider):
    """Load user preferences from a database"""
    
    def __init__(self, user_id: str):
        self.user_id = user_id
        self.db = get_database_connection()
    
    async def thread_created(self, thread_id: str | None):
        """Called when a new thread is created"""
        print(f"New thread created: {thread_id}")
    
    async def invoking(self, messages, **kwargs) -> Context:
        """Called before agent invocation - load memories"""
        # Query database for user preferences
        preferences = await self.db.query(
            "SELECT * FROM user_preferences WHERE user_id = ?",
            self.user_id
        )
        
        # Format as instructions
        instructions = "## User Preferences\n"
        for pref in preferences:
            instructions += f"- {pref['key']}: {pref['value']}\n"
        
        # Return context with instructions
        return Context(instructions=instructions)
    
    async def invoked(self, request_messages, response_messages, **kwargs):
        """Called after agent response - store new memories"""
        # Extract and store new preferences
        for msg in response_messages or []:
            if "remember" in msg.content.lower():
                # Extract and store preference
                await self.db.insert(
                    "INSERT INTO user_preferences VALUES (?, ?, ?)",
                    self.user_id, "extracted_key", "extracted_value"
                )

# Use custom provider
agent = ChatAgent(
    client=chat_client,
    context_providers=CustomMemoryProvider(user_id="user123")
)
```

---

## Lifecycle Diagram

```
User: "I like Python"
       │
       ▼
   agent.run(message, thread=thread)
       │
       ├─→ 1. thread.on_new_messages(user_message)
       │      └─→ Add to ChatMessageStore (short-term)
       │
       ├─→ 2. context_provider.invoking(messages)
       │      └─→ Load long-term memories from Mem0/Redis
       │          Returns: Context(instructions="User likes Python")
       │
       ├─→ 3. Agent processes with combined context:
       │      - Short-term: conversation history from thread
       │      - Long-term: instructions from context provider
       │
       ├─→ 4. Agent generates response
       │
       ├─→ 5. thread.on_new_messages(agent_message)
       │      └─→ Add to ChatMessageStore (short-term)
       │
       └─→ 6. context_provider.invoked(request, response)
              └─→ Store "likes Python" in Mem0/Redis (long-term)
```

---

## Common Questions

### Q: When should I use short-term vs long-term memory?

**Short-term** (`AgentThread`):
- ✅ Conversation flow within a session
- ✅ Context for immediate follow-up questions
- ✅ Temporary working memory
- ❌ Cross-session persistence
- ❌ User preferences

**Long-term** (`ContextProvider`):
- ✅ User preferences and facts
- ✅ Cross-session memory
- ✅ Personalization
- ✅ Historical data
- ❌ Conversation flow (use thread for this)

### Q: How do I choose between Mem0 and Redis?

**Mem0Provider**:
- ✅ Managed service (no infrastructure)
- ✅ Automatic memory extraction
- ✅ Easy to get started
- ✅ Self-improving memory
- ❌ Requires API key / external service

**RedisProvider**:
- ✅ Self-hosted (full control)
- ✅ Full-text search
- ✅ Vector search support
- ✅ High performance
- ❌ Requires Redis infrastructure

### Q: Can I use both service-managed and local threads?

No. An `AgentThread` can have either:
- `service_thread_id` (managed by AI service like OpenAI Assistants)
- `message_store` (managed locally)

But **not both**. Choose based on your needs:

**Service-managed** (`service_thread_id`):
```python
# Messages stored by OpenAI
agent = OpenAIChatClient().create_agent(...)
thread = agent.get_new_thread(service_thread_id="thread_abc123")
```

**Local** (`message_store`):
```python
# Messages stored locally
agent = ChatAgent(
    client=chat_client,
    chat_message_store_factory=ChatMessageStore
)
thread = agent.get_new_thread()
```

### Q: How do I migrate from session to persistent memory?

```python
# Step 1: Start with basic thread
thread = agent.get_new_thread()
await agent.run("I like Python", thread=thread)

# Step 2: Serialize thread
serialized = await thread.serialize()

# Step 3: Extract key information and store in long-term memory
# (This is typically done automatically by ContextProvider.invoked())
mem0_provider = Mem0Provider(user_id="user123")
await mem0_provider.invoked(
    request_messages=[ChatMessage(content="I like Python", role="user")],
    response_messages=[]
)

# Step 4: Future sessions automatically have access to long-term memory
new_agent = ChatAgent(
    client=chat_client,
    context_providers=mem0_provider
)
new_thread = new_agent.get_new_thread()
await new_agent.run("What do I like?", thread=new_thread)
# → Agent retrieves "likes Python" from Mem0
```

---

## Examples by Use Case

### E-commerce Assistant
```python
# Remember user preferences across sessions
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="customer_123",
        scope_to_per_operation_thread_id=False  # Global memory
    )
)

# Session 1: User shares preferences
thread1 = agent.get_new_thread()
await agent.run("I prefer size M and blue colors", thread=thread1)

# Session 2: Agent remembers
thread2 = agent.get_new_thread()
await agent.run("Show me shirts", thread=thread2)
# → Agent suggests size M, blue shirts
```

### Customer Support
```python
# Remember user issues and context per thread
agent = ChatAgent(
    client=chat_client,
    context_providers=Mem0Provider(
        user_id="customer_123",
        scope_to_per_operation_thread_id=True  # Per-thread memory
    )
)

# Ticket 1 thread
ticket1_thread = agent.get_new_thread()
await agent.run("My order is delayed", thread=ticket1_thread)

# Ticket 2 thread (different issue, different context)
ticket2_thread = agent.get_new_thread()
await agent.run("I need to return an item", thread=ticket2_thread)

# Memories are isolated between tickets
```

### Personal Assistant
```python
# Combine short-term conversation with long-term user profile
agent = ChatAgent(
    client=chat_client,
    context_providers=[
        Mem0Provider(user_id="user_123"),     # Personal preferences
        RedisProvider(user_id="user_123"),    # Calendar, contacts, etc.
    ]
)

thread = agent.get_new_thread()

# Agent has access to:
# - Short-term: Current conversation
# - Long-term (Mem0): User preferences
# - Long-term (Redis): User data (calendar, etc.)

await agent.run("Schedule a meeting tomorrow", thread=thread)
# → Agent knows user's preferences and calendar availability
```

---

## Related Files

### Documentation
- `/docs/MEMORY_AND_CONVERSATION_STATE_SUMMARY.md` - Comprehensive summary
- `/docs/MEMORY_AND_CONVERSATION_STATE_CODE_MAPPING.md` - Diagram-to-code mapping

### Examples
- `python/samples/getting_started/threads/suspend_resume_thread.py`
- `python/samples/getting_started/threads/custom_chat_message_store_thread.py`
- `python/samples/getting_started/context_providers/mem0/mem0_threads.py`
- `python/samples/getting_started/context_providers/redis/redis_threads.py`

### Core Code
- `python/packages/core/agent_framework/_threads.py`
- `python/packages/core/agent_framework/_memory.py`
- `python/packages/mem0/agent_framework/mem0/_provider.py`
- `python/packages/redis/agent_framework/redis/_provider.py`
