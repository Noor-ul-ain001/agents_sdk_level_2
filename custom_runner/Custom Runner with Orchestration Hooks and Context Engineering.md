
### 🔧 Orchestration with Hooks

**Hooks** are lifecycle callbacks that let you inject custom code at key points during an agent's execution, making them ideal for orchestration tasks like logging, monitoring, and modifying behavior .

The SDK offers two primary hook systems:

| Hook System | Scope | Primary Use Case |
| :--- | :--- | :--- |
| **`RunHooks`**  | Global (entire run) | Logging, analytics, & behavior across all agents & tools in a workflow . |
| **`AgentHooks`**  | Local (single agent) | Monitoring or controlling behavior for a specific agent . |

You can implement hooks for events like `on_agent_start`, `on_agent_end`, `on_handoff`, `on_tool_start`, and `on_tool_end` . This allows you to track conversation time, log outcomes, add custom metrics, or pre-process inputs before tools run .

### 🧠 Context Engineering

The `context` is a powerful tool for dependency injection and state management throughout an agent run.

- **Purpose**: You can provide any Python object as context when starting a run with `Runner.run()`. This object is then passed to every agent, tool, handoff, and hook, serving as a shared grab bag for dependencies and state .
- **Usage**: The context is not passed to the LLM itself. It's designed for your code—in tool functions, callbacks, and hooks—to access common dependencies or maintain state across the execution of a run . For instance, you could use it to pass a database connection or a shared counter to all parts of your agent system.

### 💡 A Practical Blueprint for Custom Orchestration

While building a custom `Runner` isn't covered, you can achieve sophisticated orchestration by combining the existing `Runner` with hooks and context. Here’s a conceptual blueprint:

1.  **Design Your Agent Network**: Define a set of specialized agents that excel at specific tasks .
2.  **Implement `RunHooks` for Orchestration Logic**: Create a global `RunHooks` class. Key orchestration logic can be placed in the `on_handoff` method to make decisions about agent routing, or in `on_tool_end` to trigger new processes based on tool results .
3.  **Use Context for State and Dependencies**: Initialize a context object with shared resources (like API clients or data structures) and pass it to the `Runner.run()` method. Your hooks and tools can then read from and write to this context to share information and manage state cohesively .
4.  **Execute with the Standard Runner**: Kick off the entire workflow using the standard `Runner.run()` or `Runner.run_sync()`, passing in your starting agent, the initial input, your custom hooks, and the engineered context .

