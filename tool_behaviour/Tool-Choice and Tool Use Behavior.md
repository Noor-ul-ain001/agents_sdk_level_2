In the OpenAI Agents SDK, you can precisely control if and how your agents use tools through the `tool_choice` and `tool_use_behavior` parameters. `tool_choice` determines whether an agent *must* use a tool, while `tool_use_behavior` controls what happens *after* a tool is called.

The table below summarizes the core configuration options:

| Parameter | Purpose | Key Values & Behaviors |
| :--- | :--- | :--- |
| **`tool_choice`** | Forces the agent's initial decision on tool usage. | - `"auto"` (Default): Model decides.<br>- `"required"`: Must use at least one tool.<br>- `"none"`: Forbidden from using tools.<br>- `"my_tool_name"`: Must call a specific tool. |
| **`tool_use_behavior`** | Controls the agent's workflow after making a tool call. | - `"run_llm_again"` (Default): Sends tool results back to the LLM for a final response.<br>- `"stop_on_first_tool"`: Uses the first tool's output as the agent's final response.<br>- `StopAtTools`: Stops execution if specified tools are used.<br>- Custom Function: For implementing completely custom logic. |
| **`reset_tool_choice`** | Prevents infinite loops by resetting `tool_choice` after a tool call. | - `True` (Default): Resets to `"auto"` after a tool call.<br>- `False`: Leaves the `tool_choice` setting enforced, which risks infinite loops. |

### 🛠️ Detailed Configuration Guide

Here's a closer look at how to apply these settings and their implications.

- **Configuring `tool_choice`**: This parameter is set within the `model_settings` of an agent. It's important to note that due to the default `reset_tool_choice=True`, a setting of `"required"` or a specific tool name is only enforced for the very first LLM call in an agent's turn. After a tool is used, the setting reverts to `"auto"` to prevent the model from being forced to call tools repeatedly with the tool results, which would cause an infinite loop.

- **Using `tool_use_behavior`**: This parameter offers flexible control over the agent's execution flow.
    - The `"run_llm_again"` strategy is the most common, allowing the agent to synthesize tool results into a polished, final answer.
    - `"stop_on_first_tool"` is useful when the tool's output is the final desired result, such as a raw data fetch.
    - For more advanced scenarios, you can pass a `StopAtTools` dictionary with a `stop_at_tool_names` list. The agent will halt execution if it calls any tool in that list, using that tool's output as the final result without further LLM processing.
    - For maximum control, you can provide a custom function that takes the run context and a list of tool results, then decides whether to stop or continue.

### 💡 Best Practices and Important Considerations

- **Preventing Infinite Loops**: The SDK is designed to prevent infinite loops by automatically resetting `tool_choice` to `"auto"` after a tool is called. While you can disable this with `reset_tool_choice=False`, it is not recommended as it can easily lead to situations where the model is forced to call a tool in every subsequent turn.
- **Tool Selection Performance**: Providing a large number of tools to an agent can sometimes confuse the model and degrade its performance in selecting the right one. It is a best practice to only provide the agent with tools that are relevant to its specific task.
- **Hosted Tools Behavior**: Note that the `tool_use_behavior` configuration primarily applies to `FunctionTools`. Built-in hosted tools like web search or file search are always processed by the LLM, meaning their results are always sent back to the model for interpretation.

