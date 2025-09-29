In the OpenAI Agents SDK, when one agent hands off a task to another, the entire conversation history is passed along by default, allowing the new agent to understand the full context. You can also use the `Sessions` feature for automatic history management or manually manage the history for finer control.

The table below summarizes the core concepts for managing conversation history during handoffs:

| Feature | Description | Key Characteristics |
| :--- | :--- | :--- |
| **Default Handoff Behavior** | The target agent receives the complete conversation history, including past user messages, assistant responses, and tool calls. | Maintains full context automatically; can sometimes cause confusion if the new agent fixates on old tool calls. |
| **Input Filters** | Functions that modify the history passed to the next agent, useful for cleaning up the context. | Can remove tool calls from history or apply other custom logic using `input_filter` parameter in `handoff()`. |
| **Sessions** | A built-in SDK feature that automatically manages conversation history across multiple agent runs. | Eliminates manual history handling; integrates seamlessly with handoffs; supports various backends like SQLite and the OpenAI Conversations API. |
| **Manual History Management** | Using `result.to_input_list()` to get the conversation history and pass it explicitly in the next run. | Offers maximum control; requires more code to manage the history list manually. |

### 🔧 How to Manage History in Handoffs

For more granular control, you can customize what history is passed during a handoff using an **input filter**. This is especially useful for cleaning up the context. For instance, you might want to remove all previous tool calls from the history before handing off to a new agent to prevent confusion. The SDK provides common filter patterns in `agents.extensions.handoff_filters`.

Here is a conceptual example of how you might apply a filter:

```python
from agents import handoff
from agents.extensions import handoff_filters

# Create a handoff that automatically removes tool calls from the history
custom_handoff = handoff(
    agent=faq_agent,
    input_filter=handoff_filters.remove_tools # This is a pre-built filter
)
```

### 💡 Recommendations and Best Practices

- **Use Sessions for Simplicity**: For most applications, especially those involving multi-turn conversations, using the built-in `Sessions` is the most robust and straightforward method. It automatically handles the storage and retrieval of conversation history for you.
- **Guide the LLM with Prompts**: The SDK provides a recommended prompt prefix that helps agents understand how and when to use handoffs. You can access it via `agents.extensions.handoff_prompt.RECOMMENDED_PROMPT_PREFIX` and include it in your agent's instructions.
- **Consider Filters for Complex Workflows**: If you find that agents are getting distracted by irrelevant parts of the conversation history after a handoff, implementing a custom `input_filter` is a powerful way to curate the context.
- **Manage Token Limits**: Be mindful that passing very long conversation histories can consume many tokens. While not covered in detail in the provided results, this is a general best practice for working with LLMs.

