# Context Providers Examples

## Summary

This folder demonstrates how to use Context Providers with the Agent Framework to give agents persistent memory capabilities. Context Providers allow agents to store and retrieve conversational context, user preferences, and tool outputs across multiple interactions. The examples showcase three different memory backends: a simple custom implementation, Mem0 (a managed memory service), and Redis (a powerful in-memory data store with vector search capabilities).

Context Providers enable agents to:
- Remember user preferences and details across conversations
- Recall past interactions and tool outputs
- Maintain continuity across different threads and sessions
- Provide personalized responses based on accumulated knowledge

## Examples

### Simple Context Provider

**File**: `simple_context_provider.py`

**Key Features**:
- Custom implementation of the ContextProvider interface
- Demonstrates how to extract and store user information (name, age)
- Uses structured output to parse user details from conversations
- Shows the basic lifecycle of context providers (invoked after each agent call)
- Minimal implementation suitable for learning and prototyping

**Use Case**: Understanding the fundamentals of context providers and creating custom memory solutions tailored to specific needs.

---

### Mem0 Integration

Mem0 is a managed memory service that provides intelligent context storage with minimal configuration.

#### Basic Usage

**File**: `mem0/mem0_basic.py`

**Key Features**:
- Integration with Mem0's cloud-based memory service
- Automatic memory association using user_id
- Demonstrates memory persistence across multiple queries
- Shows how agents learn user preferences (company code, report format)
- Tool integration with memory (retrieve_company_report function)

**Use Case**: Quick setup of persistent memory for production agents without managing infrastructure.

#### Open Source Local Mem0

**File**: `mem0/mem0_oss.py`

**Key Features**:
- Uses local Mem0 OSS client (AsyncMemory) instead of cloud service
- Self-hosted memory storage for privacy-sensitive applications
- OpenAI integration for embeddings (configurable to other LLM providers)
- Same memory capabilities as cloud version but running locally

**Use Case**: Self-hosted deployments where data must remain on-premises or when you need full control over the memory infrastructure.

#### Thread Scoping with Mem0

**File**: `mem0/mem0_threads.py`

**Key Features**:
- **Global thread scope**: Share memories across all threads and operations
- **Per-operation thread scope**: Isolate memories to individual threads
- Demonstrates scope_to_per_operation_thread_id configuration
- Shows how to create and manage separate conversational threads
- Memory isolation strategies for multi-user or multi-session scenarios

**Use Case**: Applications requiring fine-grained control over memory visibility, such as multi-tenant systems or conversational agents with distinct session boundaries.

---

### Redis Integration

Redis provides a powerful, self-hosted memory solution with advanced features like full-text search and vector similarity search.

#### Basic Usage

**File**: `redis/redis_basics.py`

**Key Features**:
- Three progressive scenarios: standalone provider, agent integration, tool memory
- Full-text and hybrid vector search for retrieving relevant context
- RediSearch integration for fast querying
- Tool output persistence (search_flights function example)
- Demonstrates how tools' structured outputs are captured in memory

**Use Case**: Applications requiring fast, scalable memory with advanced search capabilities, especially when tool outputs need to be searchable.

#### Conversation Storage

**File**: `redis/redis_conversation.py`

**Key Features**:
- Uses RedisChatMessageStore for persistent conversation history
- Integration with RedisProvider for dual functionality (messages + context)
- Hybrid search with OpenAI embeddings for semantic retrieval
- Demonstrates agent.get_chat_history() usage
- Shows how to configure Redis vectorizer and embedding cache

**Use Case**: Building chat applications that need both conversational history and intelligent context retrieval, such as customer support bots or knowledge assistants.

#### Thread Scoping with Redis

**File**: `redis/redis_threads.py`

**Key Features**:
- **Global thread scope**: Fixed thread_id shares memories across all operations
- **Per-operation thread scope**: Binds provider to a single thread lifetime
- **Multi-agent memory isolation**: Different agent_id values for separate personas
- Demonstrates how to scope memory visibility at different levels
- Shows best practices for thread and agent isolation

**Use Case**: Complex multi-agent systems where different agents need isolated memories, or applications with multiple concurrent user sessions requiring memory separation.

## Key Concepts

### What is a Context Provider?

A Context Provider is a component that intercepts agent interactions and manages contextual information. It:
- Implements the `ContextProvider` interface
- Gets called after each agent invocation via the `invoked()` method
- Can read request/response messages and store relevant information
- Provides context back to agents during subsequent interactions

### Memory Association

Context providers typically associate memories with identifiers:
- **user_id**: Link memories to specific users
- **agent_id**: Link memories to specific agent instances
- **thread_id**: Link memories to conversational threads
- **application_id**: Link memories to applications (for multi-tenant scenarios)

### Memory Scoping

Different scoping strategies control memory visibility:
- **Global scope**: All threads see the same memories
- **Per-thread scope**: Each thread has isolated memories
- **Per-agent scope**: Each agent maintains separate memories

## Installation

### Basic Requirements

```bash
pip install agent-framework --pre
```

### Mem0 Integration

```bash
pip install mem0ai
# Set MEM0_API_KEY environment variable for cloud service
# Or use local OSS version (requires OpenAI API key)
```

### Redis Integration

```bash
pip install "agent-framework-redis"
# Requires Redis Stack (with RediSearch module)
# Optionally set OPENAI_API_KEY for vector embeddings
```

## Environment Variables

### Azure Authentication

```bash
# For Azure AI agents, authenticate via Azure CLI
az login
# Or set these variables:
# AZURE_OPENAI_ENDPOINT
# AZURE_OPENAI_DEPLOYMENT_NAME
# AZURE_OPENAI_API_KEY
```

### Mem0 Configuration

```bash
# Cloud service
MEM0_API_KEY=your_mem0_api_key

# Local OSS (requires LLM for embeddings)
OPENAI_API_KEY=your_openai_api_key
```

### Redis Configuration

```bash
# Redis connection
REDIS_URL=redis://localhost:6379  # Default

# For vector search
OPENAI_API_KEY=your_openai_api_key  # If using OpenAI embeddings
OPENAI_CHAT_MODEL_ID=gpt-4o-mini    # Optional, for chat examples
```

## Running the Examples

1. Navigate to the context_providers folder:
   ```bash
   cd python/samples/getting_started/context_providers
   ```

2. Set up required environment variables (see above)

3. Run any example:
   ```bash
   python simple_context_provider.py
   python mem0/mem0_basic.py
   python redis/redis_basics.py
   ```

## Choosing the Right Provider

| Provider | Best For | Complexity | Hosting |
|----------|----------|------------|---------|
| **Simple Custom** | Learning, prototypes, custom logic | Low | N/A |
| **Mem0 Cloud** | Quick production setup, managed service | Low | Cloud |
| **Mem0 OSS** | Self-hosted, privacy-sensitive apps | Medium | Self-hosted |
| **Redis** | High-performance, advanced search, large scale | High | Self-hosted |

## Common Patterns

### Extract and Store User Info

```python
class UserInfoMemory(ContextProvider):
    async def invoked(self, request_messages, response_messages, **kwargs):
        # Extract structured data from conversations
        user_info = await self._chat_client.get_response(
            messages=request_messages,
            chat_options=ChatOptions(response_format=UserInfo)
        )
```

### Provide Context to Agents

```python
# Mem0 automatically retrieves relevant memories
agent = client.create_agent(
    name="Assistant",
    context_providers=Mem0Provider(user_id=user_id)
)
```

### Scope Memory to Threads

```python
# Global scope (shared across threads)
provider = Mem0Provider(
    user_id=user_id,
    thread_id=global_thread_id,
    scope_to_per_operation_thread_id=False
)

# Per-operation scope (isolated threads)
provider = Mem0Provider(
    user_id=user_id,
    scope_to_per_operation_thread_id=True
)
```

## References

- [Mem0 Documentation](https://docs.mem0.ai/)
- [Redis Documentation](https://redis.io/docs/)
- [RediSearch](https://redis.io/docs/stack/search/)
- [Agent Framework Core Documentation](../../../README.md)
