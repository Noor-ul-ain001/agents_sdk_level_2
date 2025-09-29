In the OpenAI Agents SDK, context management is the architectural practice of strategically providing the right information to the right part of your agent system at the right time. It is fundamentally split into two categories: local context for your code and LLM context for the model.

The table below outlines the core concepts and methods for providing context to the LLM:

| Concept | Description | Primary Methods in OpenAI Agents SDK |
| :--- | :--- | :--- |
| **Local Context** | A Python object for data, dependencies, and helpers accessible by your tools and hooks. Not sent to the LLM. | Passed via `Runner.run(..., context=your_object)` and accessed via `RunContextWrapper[T].context`. |
| **LLM Context** | The conversation history and instructions the model uses to generate a response. Managed to avoid exceeding token limits. | 1. Agent `instructions` (system prompt).<br>2. User `input`.<br>3. **Function Tools** for on-demand data.<br>4. **Retrieval/Web Search** tools.<br>5. **Session Memory** for automated history management. |

### 🛠️ Advanced Context Management with Sessions

For multi-turn conversations, the SDK provides a `Session` object to automate conversation history management, replacing the need for manual state handling. Effective use of sessions involves techniques to manage context length.

-   **Context Trimming**: This technique keeps only the most recent `N` turns of a conversation. A "turn" typically includes a user message and the subsequent assistant messages and tool calls/results up to the next user message.
    -   **Pros**: It's deterministic, simple, and adds no latency or cost from extra model calls. It preserves recent details perfectly, which is great for debugging.
    -   **Cons**: It can abruptly forget important long-range context, making the agent seem like it has "amnesia" for earlier parts of a long conversation.
    -   **Best for**: Short workflows or tool-heavy operations where recent steps are most important.

-   **Context Summarization**: This technique compresses older messages into a concise, structured summary that is injected into the conversation history.
    -   **Pros**: Retains long-range memory compactly, leading to a smoother user experience where the agent "remembers" earlier commitments.
    -   **Cons**: The summarization process can introduce distortion or bias, and each refresh adds latency and cost. It also requires careful logging for observability.
    -   **Best for**: Long, complex conversations like planning or analysis, where continuity and remembering early decisions are crucial.

### 💡 Strategic Importance and Best Practices

Thinking strategically about context is key to building effective agents.

-   **Why Context is a Strategic Advantage**: Robust context management transforms agent capabilities from generic to exceptional. It significantly enhances response accuracy and relevance, enables deeper reasoning, and provides a future-proof architecture where your organization's unique data becomes a durable asset, independent of the underlying AI model.
-   **General Best Practices**:
    -   **Use Local Context for Dependencies**: Inject objects like database connections, loggers, or user data into the run context so they are available to all tools and hooks without being sent to the LLM.
    -   **Keep LLM Context Focused**: Avoid stuffing the model's context window with irrelevant information. Use techniques like trimming, summarization, and on-demand tool calls to keep the context sharp and relevant.
    -   **Leverage Sessions for Multi-Turn Conversations**: Use the SDK's `Session` object to automatically manage conversation history, making it much easier to build coherent, long-running agents.
