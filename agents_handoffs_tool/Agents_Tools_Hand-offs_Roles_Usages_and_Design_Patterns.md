| Concept | Role & Function | Key Characteristics |
| :--- | :--- | :--- |
| **Agent** | The core "brain" of the application; an LLM instance equipped with instructions, tools, and the ability to reason and act. | Defined by `name` and `instructions`; can use tools, hand off tasks, and produce structured outputs. |
| **Tool** | Extends an agent's capabilities, allowing it to interact with external systems or perform specific functions. | Can be a Python function (decorated with `@function_tool`), an external API, or another agent. |
| **Handoff** | A mechanism for one agent to delegate an entire task and conversation context to another, more specialized agent. | Enables the creation of multi-agent, collaborative workflows; the new agent takes over the interaction. |
| **Runner** | Manages the execution of an agent, handling the agent loop, retries, and streaming responses. | The conductor of the orchestra; you use `Runner.run()` or `Runner.run_sync()` to execute an agent. |
| **Design Pattern: Agent-as-Tools** | A centralized pattern where a manager agent retains control of the conversation and calls specialized agents as needed. | The manager agent consults specialists and synthesizes their results into a single, cohesive response for the user. |
| **Design Pattern: Handoffs** | A decentralized pattern where an agent transfers full control of the conversation to a specialist agent in a relay-like fashion. | Ideal for sequential, domain-specific workflows where one specialist at a time is sufficient. |

### 🧩 A Closer Look at the Core Components

Here is more detailed information on these core components and how they are used.

- **The Agent**: At a minimum, an agent requires a `name` and `instructions` (its system prompt). However, its true power comes from customization. You can equip it with a list of `tools`, define a list of `handoffs` to other agents, specify an `output_type` (like a Pydantic model) for structured outputs, and even provide a `context` object for dependency injection.
- **Tools in Practice**: Tools are how agents perform actions. Any Python function can be turned into a tool using the `@function_tool` decorator, with the SDK automatically handling input/output schema validation. For example, you can easily create a tool to get the weather or connect to external services like GitHub via MCP (Model Context Protocol) servers.
    ```python
    from agents import function_tool

    @function_tool
    def get_weather(city: str) -> str:
        # You could call a real API here
        return f"The weather in {city} is sunny."
    ```
- **The Agent Loop**: When you trigger an agent with `Runner.run()`, it starts a sophisticated loop:
    1.  The LLM is called with the agent's instructions and message history.
    2.  The LLM returns a response, which may contain tool calls, a handoff, or a final output.
    3.  If there's a **final output**, the loop ends.
    4.  If there's a **handoff**, the active agent is switched, and the loop restarts.
    5.  If there are **tool calls**, they are executed, and the results are fed back into the history before the loop continues.

### 🤝 Multi-Agent Design Patterns Deep Dive

The two primary patterns for orchestrating multiple agents serve different purposes.

- **Agent-as-Tools (Centralized Manager)**: In this pattern, a primary "manager" agent is the single point of contact for the user. When it encounters a task outside its expertise, it invokes a specialized agent as if it were a tool. The manager receives the specialist's output and is responsible for synthesizing it into a final, cohesive response for the user.
    - **Best for**: Dynamic queries that require consulting multiple specialists and combining their results, or when you want a single agent to maintain global context and ensure a consistent user experience.
    - **Considerations**: The manager agent can become a bottleneck, and there is overhead for simple queries that only require one specialist.

- **Handoffs (Decentralized Relay)**: This pattern is like a relay race. One agent, upon determining that another is better suited for the task, can "hand off" the entire conversation. The new agent takes over with the full context and directly interacts with the user from that point on. A common use case is a triage agent that routes user requests to the correct specialist.
    - **Best for**: Structured, sequential workflows where tasks can be cleanly segmented and handled by different specialists one at a time.
    - **Considerations**: Handoffs are sequential and can be slower; debugging a chain of handoffs can be complex, and it's harder to synthesize information from multiple agents.

### 💡 Key Design Principles and Advanced Usage

To build effective agents, keep these principles and capabilities in mind.

- **Python-First Philosophy**: The SDK uses Python's native features—like loops, conditionals, and function calls—for orchestration, avoiding the need to learn a new domain-specific language (DSL). This makes it powerful for expressing complex relationships with a minimal learning curve.
- **Structured Outputs**: Instead of just plain text, you can define an `output_type` for an agent using a Pydantic model. This forces the agent to produce validated, structured data, making its output predictable and easy to consume by other parts of your application.
    ```python
    from pydantic import BaseModel
    from agents import Agent

    class Summary(BaseModel):
        key_points: list[str]
        sentiment: str

    agent = Agent(name="Summarizer", instructions="...", output_type=Summary)
    ```
- **Guardrails for Safety and Reliability**: Guardrails are validation functions that run in parallel to your agent. **Input guardrails** check and can moderate user input before the agent processes it, while **output guardrails** validate the agent's final output before it's returned to the user. This is crucial for building safe, production-ready applications.
