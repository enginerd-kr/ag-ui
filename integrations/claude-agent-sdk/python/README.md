# ag-ui-claude-agent-sdk

Implementation of the AG-UI protocol for the Anthropic Claude Agent SDK (Python).

## Installation

```bash
pip install -e .
```

## Usage

The adapter manages the SDK lifecycle internally — just call `adapter.run(input_data)`:

```python
from ag_ui_claude_sdk import ClaudeAgentAdapter, add_claude_fastapi_endpoint

adapter = ClaudeAgentAdapter(name="my_agent", options={"model": "claude-haiku-4-5"})
add_claude_fastapi_endpoint(app=app, adapter=adapter, path="/my_agent")
```

## Keeping Shared State Fresh Across Turns

Threads reuse one long-lived `SessionWorker` per `thread_id`, so `options["system_prompt"]`
(and the "Current Shared State" block the adapter appends to it) is only built once, on a
thread's first turn — later turns reuse that same worker without rebuilding it. If your use
case updates shared state after turn 1 and needs the model to see the latest value every turn
(e.g. a human-in-the-loop approval flag, a collaboratively-edited list), pass
`state_context_builder`:

```python
adapter = ClaudeAgentAdapter(
    name="my_agent",
    options={"model": "claude-haiku-4-5"},
    state_context_builder=lambda input_data, prompt: (
        f"Current shared state: {input_data.state}\n\n{prompt}"
    ),
)
```

Unlike `system_prompt`, this runs on **every** `run()` call regardless of whether the thread's
worker is being created or reused, so it stays accurate turn after turn. It only affects the
prompt text sent to the model for that call — `RunAgentInput.messages` and everything echoed
back to the AG-UI client (`MESSAGES_SNAPSHOT`, etc.) are untouched.

## Features

- **Full lifecycle management** - Handles client pooling, message extraction, and event translation internally
- **Interrupt support** - Call `adapter.interrupt()` to stop a running query
- **Dynamic frontend tools** - Client-provided tools automatically added as MCP server with auto-granted permissions
- **Frontend tool halting** - Streams pause after frontend tool calls for client-side execution (human-in-the-loop)
- **Streaming tool arguments** - Real-time TOOL_CALL_ARGS emission as JSON arguments stream in
- **Bidirectional state sync** - Shared state management via ag_ui_update_state tool
- **Context injection** - Context and state injected into prompts for agent awareness
- **Event cleanup** - Hanging events (tool calls, reasoning blocks) automatically closed on stream end
- **Custom tools via MCP** - Define custom tools using Claude SDK's @tool decorator
- **Forwarded props** - Per-run option overrides with security whitelist

## Examples

The integration includes 5 example agents:

| Route | Description | Features |
|-------|-------------|----------|
| `/agentic_chat` | Basic conversational assistant | Simple chat |
| `/backend_tool_rendering` | Weather tool (backend MCP) | Backend tool execution, tool rendering |
| `/shared_state` | Recipe collaboration | Bidirectional state sync, ag_ui_update_state |
| `/human_in_the_loop` | Task planning with approval | Frontend tools, step tracking, approval workflow |
| `/tool_based_generative_ui` | Frontend tool rendering | Dynamic frontend tools, generative UI |

## Running the Examples

```bash
# Install dependencies
cd integrations/claude-agent-sdk/python
pip install -e .

# Start server (port 8019)
cd examples
ANTHROPIC_API_KEY=sk-ant-xxx python server.py

# Start Dojo (in another terminal)
cd apps/dojo
pnpm dev
```

Visit **http://localhost:3000** and select **"Claude Agent SDK (Python)"**

## Session Persistence

Claude SDK maintains conversation state in the `.claude/` directory. For production deployments:

- **Development**: Sessions persist locally in `.claude/{session_id}/`
- **Production**: Mount `.claude/` as a persistent volume in your container
- **Resumption**: Pass `resume=<session_id>` via the options dict or `forwarded_props`

See [Claude SDK Hosting Guide](https://platform.claude.com/docs/en/agent-sdk/hosting) for deployment patterns.

## Links

- [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/python)
- [AG-UI Documentation](https://docs.ag-ui.com/)
- [AG-UI State Management](https://docs.ag-ui.com/concepts/state)
