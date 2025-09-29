Pydantic plays a fundamental role in structuring agent outputs and validating data. The framework uses it for type validation and schema generation, making agents more reliable and their outputs easier to integrate with other code.


| Feature | Role & Function | Key Configuration |
| :--- | :--- | :--- |
| **Structured Outputs** | Forces the agent to produce a validated, structured data type instead of plain text. | `output_type` parameter on the Agent. |
| **Output Schema** | Automatically generates and validates the JSON schema for the structured output. | Handled by `AgentOutputSchema`; strict mode is recommended. |
| **Input Guardrails** | Validates user input before the main agent processes it, allowing you to break early if checks fail. | `input_guardrails` parameter on the Agent. |
| **Output Guardrails** | Validates the agent's final output before it is returned to the user. | `output_guardrails` parameter (inferred from guardrails documentation). |
| **Function Tools** | Automatically generates the JSON schema for tool inputs from Python function signatures using Pydantic. | `@function_tool` decorator. |

### 🛠️ Implementing Structured Outputs and Validation

To build robust agents, you can leverage the SDK's features for input and output control.

- **Structured Outputs with Pydantic**: By defining a Pydantic model and setting it as the agent's `output_type`, you force the LLM to produce a validated, structured JSON object instead of plain text. This is crucial for making the agent's output a predictable data structure that can be easily used by downstream applications.

    The SDK internally uses an `AgentOutputSchema` object to manage the JSON schema of the output type and validate the JSON produced by the LLM. It is highly recommended to use **strict JSON schema** mode, as it constrains the schema features to those that guarantee valid JSON, increasing the likelihood of correct generation by the model.

- **Input and Output Guardrails**: Guardrails are a powerful mechanism for running validation checks in parallel to your agent's main operation.
    - **Input Guardrails** are functions that validate or screen the user's input before the agent even begins processing it. If a check fails (the "tripwire is triggered"), the agent run can be halted early.
    - **Output Guardrails** similarly validate the agent's final output before it is returned to the user.

### ⚙️ Configuration and Default Behaviors

The SDK is designed to work out-of-the-box but allows for extensive customization.

- **Default Text Output**: If you do not specify an `output_type` for an agent, it will default to producing plain text (`str`). In this mode, the agent loop considers the first LLM response that does not contain any tool calls or handoffs as the final output.

- **Tool Argument Validation**: When you use the `@function_tool` decorator, the SDK automatically creates a Pydantic model for the tool's arguments. The descriptions for the tool and its arguments are automatically extracted from the function's docstring, supporting popular formats like Sphinx and NumPy.

- **Configuring SDK-Wide Defaults**: You can configure the SDK's default behavior regarding API keys, clients, and tracing.
    - The SDK looks for the `OPENAI_API_KEY` environment variable by default, but you can also set it programmatically using `set_default_openai_key()`.
    - **Tracing** is enabled by default, providing visibility into your agent's workflow. You can disable it with `set_tracing_disabled()`.
    - For security, you can set environment variables to prevent sensitive data (like model inputs/outputs or tool data) from being logged.

