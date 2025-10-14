# Tools Examples

## Summary

This folder contains examples demonstrating advanced tool usage patterns with the Agent Framework. Tools (also called functions) allow agents to interact with external systems, perform computations, or execute actions on behalf of users. These examples showcase critical patterns including approval workflows, error handling, dependency injection, and thread-based tool management.

Tools enable agents to:
- Execute external functions and interact with APIs
- Require user approval before performing sensitive operations
- Handle errors gracefully and recover from failures
- Maintain tool call history across conversation threads
- Inject dependencies at runtime for flexible tool configuration

## Examples

### AI Tool with Approval

**File**: `ai_tool_with_approval.py`

**Key Features**:
- **Approval Mode**: Uses `approval_mode="always_require"` to gate tool execution
- **Stateless Approval Handling**: Demonstrates approval workflow without threads
- **Manual Message Management**: Shows how to manually pass messages when not using threads
- **Multiple Tools**: Includes both approved (`get_weather_detail`) and unapproved (`get_weather`) tools
- **Approval Request Loop**: Iteratively handles approval requests until completion
- **User Input Requests**: Shows how to process and respond to user input requests

**Use Case**: Agents that need user consent before performing sensitive operations (e.g., making purchases, sending emails, or modifying data) without persistent thread storage.

**Key Pattern**:
```python
@ai_function(approval_mode="always_require")
def get_weather_detail(location: str) -> str:
    """Get detailed weather - requires user approval."""
    return f"Weather details for {location}..."

# Handle approvals manually
result = await agent.run(query)
while result.user_input_requests:
    # Collect approvals
    approval_response = request.create_response(approved=True)
    result = await agent.run([query, result, approval_response])
```

---

### AI Tool with Approval and Threads

**File**: `ai_tool_with_approval_and_threads.py`

**Key Features**:
- **Thread-Based Approval**: Uses threads to automatically manage conversation history
- **Simplified Approval Flow**: No need to manually track messages - thread handles it
- **Calendar Tool Example**: Demonstrates approval for calendar event creation
- **Approval Request Processing**: Shows clean pattern for handling approval requests
- **Thread State Management**: Thread automatically stores approval requests and responses

**Use Case**: Production applications where tool approvals need to be tracked across multiple turns in a conversation, such as assistants that schedule meetings, create tasks, or make reservations.

**Key Pattern**:
```python
@ai_function(approval_mode="always_require")
def add_to_calendar(event_name: str, date: str) -> str:
    """Add calendar event - requires approval."""
    return f"Added '{event_name}' to calendar on {date}"

# With threads, approval is simpler
thread = agent.get_new_thread()
result = await agent.run(query, thread=thread)

if result.user_input_requests:
    for request in result.user_input_requests:
        approval = request.create_response(approved=True)
        result = await agent.run(ChatMessage(contents=[approval]), thread=thread)
```

---

### Failing Tools

**File**: `failing_tools.py`

**Key Features**:
- **Exception Handling**: Demonstrates graceful handling of tool execution failures
- **Error Recovery**: Shows how agents can recover from failed tool calls
- **ZeroDivisionError Example**: Uses division by zero to trigger predictable failures
- **Continued Conversation**: Proves that conversation continues after tool failures
- **Error Context for LLM**: Agent receives exception details to make recovery decisions
- **Conversation Replay**: Shows how failed tool calls appear in message history

**Use Case**: Building robust agents that can handle tool failures without crashing, retry operations, or inform users about errors in a conversational manner (e.g., API timeouts, invalid inputs, external service failures).

**Key Pattern**:
```python
def safe_divide(a: int, b: int) -> str:
    """Divide numbers - can fail with ZeroDivisionError."""
    try:
        result = a / b
    except ZeroDivisionError as exc:
        print(f"Tool failed: {exc}")
        raise  # Agent receives the error and can recover

# Agent automatically handles the error
response = await agent.run("Divide 10 by 0", thread=thread)
# Agent responds with error explanation
# Conversation continues normally
```

---

### Tool with Injected Function

**File**: `tool_with_injected_func.py`

**Key Features**:
- **Dependency Injection**: Shows how to inject function implementations at runtime
- **Tool Serialization**: Demonstrates creating tools from dictionary definitions
- **Deferred Implementation**: Tool schema defined separately from implementation
- **Runtime Configuration**: Enables dynamic tool configuration without code changes
- **Flexible Tool Creation**: Useful for loading tools from configuration files or databases

**Use Case**: Applications that need to dynamically configure tools based on runtime conditions, load tool configurations from external sources, or support plugin-like tool systems where implementations are provided separately from schemas.

**Key Pattern**:
```python
# Tool definition (could come from config/database)
definition = {
    "type": "ai_function",
    "name": "add_numbers",
    "description": "Add two numbers together.",
    "input_model": { ... }
}

# Inject actual implementation at runtime
def func(a, b) -> int:
    return a + b

tool = AIFunction.from_dict(
    definition,
    dependencies={
        "ai_function": {
            "name:add_numbers": {"func": func}
        }
    }
)
```

**Note**: This feature is under active development and the API may evolve in future versions. Always refer to the latest documentation for updates.

---

## Key Concepts

### Tool Approval Modes

Tools can be configured with different approval modes:
- **No approval**: Tool executes immediately (default)
- **`always_require`**: User must explicitly approve each tool call
- **Conditional approval**: Custom logic determines when approval is needed

### Approval Workflow

1. Agent identifies need for tool call
2. Framework intercepts and creates `UserInputRequest`
3. Application prompts user for approval
4. User responds with `approval_response.create_response(approved=True/False)`
5. If approved, tool executes; if rejected, agent receives rejection message

### Thread vs. Stateless Approval

**With Threads** (Recommended):
- Thread automatically stores conversation history
- Simpler code - no manual message management
- Better for multi-turn conversations

**Without Threads** (Advanced):
- Must manually pass all previous messages
- More complex but more control
- Useful for stateless or serverless scenarios

### Error Handling Philosophy

The framework follows a "fail gracefully" approach:
- Tool exceptions are caught and reported to the agent
- Agent receives error details and can decide how to respond
- Conversation continues after failures
- Users receive natural language explanations

### Dependency Injection Benefits

- **Flexibility**: Change tool implementations without changing agent code
- **Testing**: Inject mock functions for unit testing
- **Configuration**: Load tool definitions from external sources
- **Plugin Systems**: Support third-party tool implementations

## Environment Variables

```bash
# For OpenAI examples
OPENAI_API_KEY=your_openai_api_key

# For Azure OpenAI examples
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT_NAME=your-deployment-name
AZURE_OPENAI_API_KEY=your-api-key  # Or use Azure CLI auth
```

## Running the Examples

1. **Set up environment variables** (see above)

2. **Navigate to the tools folder**:
   ```bash
   cd python/samples/getting_started/tools
   ```

3. **Run individual examples**:
   ```bash
   # Approval without threads
   python ai_tool_with_approval.py

   # Approval with threads (simpler pattern)
   python ai_tool_with_approval_and_threads.py

   # Error handling
   python failing_tools.py

   # Dependency injection
   python tool_with_injected_func.py
   ```

## Best Practices

### 1. Use Threads for Approval Workflows
Threads simplify approval handling by automatically managing conversation history:
```python
thread = agent.get_new_thread()
result = await agent.run(query, thread=thread)
# Thread tracks all messages automatically
```

### 2. Always Handle Tool Exceptions
Let the agent handle errors gracefully instead of crashing:
```python
def my_tool(param: str) -> str:
    try:
        return risky_operation(param)
    except Exception as exc:
        raise  # Let framework handle it
```

### 3. Provide Clear Approval Context
Give users enough information to make approval decisions:
```python
@ai_function(approval_mode="always_require")
def send_email(to: str, subject: str, body: str) -> str:
    """Send email - requires approval.
    
    Args:
        to: Recipient email address
        subject: Email subject line
        body: Email message body
    """
    # Clear parameters help user understand what they're approving
```

### 4. Test Tool Failures
Ensure your agent handles failures gracefully:
```python
# Test with invalid inputs
await agent.run("Divide 10 by 0")
# Verify agent provides helpful error message
```

### 5. Use Type Annotations
Annotated types provide better tool descriptions to the LLM:
```python
from typing import Annotated

def get_weather(
    location: Annotated[str, "City and state, e.g. 'San Francisco, CA'"]
) -> str:
    """Get weather for a location."""
```

### 6. Consider Security Implications
- Always require approval for destructive operations
- Validate tool inputs before execution
- Log tool executions for audit trails
- Implement rate limiting for external API calls

## Common Patterns

### Approval with Thread

```python
@ai_function(approval_mode="always_require")
def sensitive_operation(param: str) -> str:
    """Operation requiring approval."""
    return f"Executed with {param}"

thread = agent.get_new_thread()
result = await agent.run("Do something sensitive", thread=thread)

for request in result.user_input_requests:
    # Present to user, get approval
    approved = get_user_approval(request)
    
    response = request.create_response(approved=approved)
    result = await agent.run(
        ChatMessage(contents=[response]),
        thread=thread
    )
```

### Error Recovery

```python
def api_call(endpoint: str) -> str:
    """Call external API - may fail."""
    try:
        return call_external_api(endpoint)
    except requests.RequestException as exc:
        # Agent receives error and can retry or inform user
        raise

# Agent automatically handles failure
result = await agent.run("Call the user API")
# Agent: "I'm sorry, the API is currently unavailable..."
```

### Dynamic Tool Loading

```python
# Load tool definitions from config
tool_configs = load_from_database()

tools = []
for config in tool_configs:
    impl = get_implementation(config["name"])
    tool = AIFunction.from_dict(
        config,
        dependencies={
            "ai_function": {
                f"name:{config['name']}": {"func": impl}
            }
        }
    )
    tools.append(tool)

agent = client.create_agent(tools=tools)
```

## Troubleshooting

### Approval Not Working
- Verify `approval_mode="always_require"` is set
- Check that you're processing `user_input_requests`
- Ensure approval responses are correctly formatted

### Tool Exceptions Not Handled
- Make sure exceptions are raised (not caught and returned as strings)
- Verify agent has proper error handling instructions
- Check that tool is properly registered with agent

### Dependency Injection Fails
- Verify dependency dictionary structure matches the pattern
- Ensure tool name matches between definition and dependencies
- Check that function signature matches input model

### Messages Not Persisting
- Use threads for automatic message persistence
- If not using threads, manually pass all previous messages
- Verify thread is correctly initialized and passed to `run()`

## References

- [Thread Management](../threads/README.md)
- [Agent Framework Core Documentation](../../../README.md)
- [Middleware Examples](../middleware/README.md) - For more advanced tool interception patterns
