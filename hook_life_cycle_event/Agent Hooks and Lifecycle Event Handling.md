The OpenAI Agents SDK provides a comprehensive hook system that allows you to monitor and inject custom logic into your AI agent's execution process. You can use two types of hooks: **RunHooks** for global, run-wide observation and **AgentHooks** for monitoring specific agents.

The table below compares these two hook systems for a clearer understanding:

| Feature | RunHooks | AgentHooks |
| :--- | :--- | :--- |
| **Scope** | Global - the entire agent run, including all agents and handoffs | Local - a single, specific agent only |
| **Attachment Point** | Passed to the `Runner.run()` method | Set on an individual agent's `.hooks` property |
| **Primary Use Case** | Global logging, analytics, auditing, and debugging across a multi-agent workflow | Agent-specific behavior, logging, or logic for a single component of the system |

### 🔧 A Closer Look at Lifecycle Events

Both `RunHooks` and `AgentHooks` allow you to intercept the same core lifecycle events. Here are the key events you can track, along with the specific methods to override:

-   **Agent Events**: Monitor the start and end of an agent's execution.
    -   `on_agent_start` / `on_start`: Called just before an agent begins processing.
    -   `on_agent_end` / `on_end`: Called when the agent produces its final output.
-   **LLM Events**: Track the direct interactions with the Large Language Model.
    -   `on_llm_start`: Called immediately before the agent issues an LLM call, providing access to the system prompt and input items.
    -   `on_llm_end`: Called right after the LLM response is received.
-   **Tool Events**: Observe when an agent uses a tool (an external function).
    -   `on_tool_start`: Called as a tool begins invocation.
    -   `on_tool_end`: Called after a tool finishes, providing access to the tool's result.
-   **Handoff Events**: Detect when one agent delegates work to another.
    -   `on_handoff`: Called when a handoff occurs from one agent to another. In `AgentHooks`, the method signature focuses on the agent being handed *to* and the source agent.

### 💻 How to Implement and Use Hooks

Implementing hooks involves creating a class that inherits from `RunHooksBase` or `AgentHooksBase` and overriding the methods for the events you care about. All hook methods are asynchronous (`async`).

Here is a basic example of implementing global `RunHooks` for logging:

```python
from agents import RunHooks, Runner

class MyRunHooks(RunHooks):
    async def on_agent_start(self, ctx, agent):
        print(f"[RUN] Agent {agent.name} started.")

    async def on_tool_start(self, ctx, agent, tool):
        print(f"[RUN] Agent {agent.name} started tool: {tool.name}")

    async def on_handoff(self, ctx, from_agent, to_agent):
        print(f"[RUN] Handoff from {from_agent.name} to {to_agent.name}")

# Using the hooks in a run
my_hooks = MyRunHooks()
result = await Runner.run(agent, input="Hello", hooks=my_hooks)
```

For **AgentHooks**, you would define a similar class but attach it directly to a specific agent:

```python
from agents import Agent, AgentHooks

class MyAgentHooks(AgentHooks):
    async def on_start(self, ctx, agent):
        print(f"[AGENT] {agent.name} is starting")

# Attaching hooks to a specific agent
agent = Agent(name="MyAgent", instructions="...")
agent.hooks = MyAgentHooks()
```

### 🚀 Real-World Use Cases for Hooks

Hooks are incredibly powerful for moving agents from prototypes to production-ready applications.

-   **Observability and Monitoring**: Log key events, track performance metrics, and monitor token usage by accessing data like `context.usage` within your hooks.
-   **Custom Analytics and Auditing**: Record detailed traces of agent decisions, tool usage, and handoff paths for later analysis.
-   **Dynamic Behavior Injection**: Pre-process data before a tool runs or modify the agent's context based on intermediate results.
-   **Debugging Complex Workflows**: Use the detailed event stream to understand exactly how a multi-agent request was processed and identify failures or bottlenecks.

