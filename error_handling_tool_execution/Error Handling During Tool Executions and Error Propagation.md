In the OpenAI Agents SDK, error handling during tool execution is managed through built-in mechanisms and custom functions, while error propagation poses a significant challenge in multi-agent systems where a single error can cascade and be amplified.

The table below summarizes the core error handling mechanisms available when using the `@function_tool` decorator:

| Mechanism | Description | Use Case |
| :--- | :--- | :--- |
| **`failure_error_function`** | A function you can pass to the decorator to provide a custom error response to the LLM if the tool call crashes. | Customizing the error message the agent sees, allowing it to understand and potentially recover from the failure. |
| **`default_tool_error_function`** | The default behavior if no custom function is provided; informs the LLM that a generic error occurred. | Basic error handling where the specific error details are not needed for the agent's reasoning. |
| **Explicit Error Raising** | Setting `failure_error_function=None` causes tool call errors to be re-raised as exceptions (e.g., `ModelBehaviorError`, `UserError`) for you to handle. | Implementing fine-grained, external error handling or failing the entire agent run on a tool error. |

### 🛠️ Implementing Error Handling in Tools

For **manually created `FunctionTool` objects**, you must handle all errors within the `on_invoke_tool` function. For the more common `@function_tool` decorator, you control the error handling strategy by passing different arguments.

Here is a practical example of implementing a custom error function:

```python
from agents import function_tool

def my_custom_error_function(ctx, agent, tool, error):
    # You can log the error here
    print(f"Tool {tool.name} failed with error: {str(error)}")
    # Return a helpful message for the LLM
    return f"The tool '{tool.name}' encountered an error. Please adjust your request and try again."

@function_tool(failure_error_function=my_custom_error_function)
def my_potentially_failing_tool(input_param: str) -> str:
    # Your tool logic that might fail
    result = some_risky_operation(input_param)
    return result
```

### 🚨 Common Errors and Propagation Risks

When building agentic systems, you will encounter several types of errors that can propagate through multiple agents.

- **SDK and API Exceptions**: The SDK defines specific exceptions like `ModelBehaviorError` (for unexpected model actions) and `UserError` (for SDK misuse). When using hosted tools, you might encounter intermittent server errors (like 500 errors) or schema validation errors such as `InvalidRequestError: Invalid tool output: expected object matching schema, got null`.

- **The Risk of Error Propagation**: In a multi-agent system, each agent has a probability of error. When agents work sequentially, the failure of one agent can corrupt the input for the next, leading to cascading failures and confident but incorrect outputs. These are often **coordination failures**, where functional agents fail to exchange information properly.

Error propagation risk increases with system complexity. In a system with many agents, even a low individual agent error rate can lead to frequent overall failures due to the number of interactions.

### 💡 Mitigation Strategies for Robust Systems

To build reliable agent systems, you should implement a combination of technical safeguards and architectural patterns.

- **Divide Labor Between LLMs and Code**: A key design principle is to use the LLM for creative reasoning and code for rule-based, deterministic tasks. This reduces the attack surface for hallucinations and errors. For instance, formatting a complex data structure into a strict syntactic format is better done in code than by an LLM.

- **Implement Cross-Agent Validation**: Introduce automated checks to verify information for logical and factual consistency as it passes between agents. This can include "circuit breaker" patterns that halt processing when a critical inconsistency is detected, preventing error propagation.

- **Use Structured Data Passing**: To avoid errors from natural language misunderstandings, use structured data channels (like the `sly_data` concept in one multi-agent accelerator) to pass critical information between agents. This keeps data in a machine-readable format that is less prone to misinterpretation by LLMs.

- **Adopt "Error-as-Value" Patterns**: Instead of relying solely on exceptions, you can structure your tools to return a tuple `(success: bool, result: Any)`. This forces the calling code to explicitly handle both success and failure states, making error processing a first-class concern.

