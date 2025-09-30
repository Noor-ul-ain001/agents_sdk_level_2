
| **Data Point** | **Description** |
| :--- | :--- |
| **Tool/Function Name** | The name of the tool that was invoked (e.g., `get_weather`) . |
| **Acting Agent** | The agent responsible for making the tool call . |
| **Timestamp** | The precise time the event occurred . |
| **Tool Input** | The specific arguments provided to the tool (e.g., `{"city": "Berlin"}`) . |
| **Tool Output** | The result returned by the tool function . |
| **LLM Calls & Token Usage** | Details of the LLM interaction that led to the tool call, including token counts for cost tracking . |
| **Event Sequence** | The order of operations, showing the tool call within the broader context of the agent's workflow . |

### 🛠️ Implementing Audit Logging with OpenAI Agents SDK

The OpenAI Agents SDK has a built-in tracing system that automatically generates these audit trails . To implement and enhance audit logging for your tool calls, you can follow these steps:

- **Leverage Built-in Tracing**: The SDK's tracing is enabled by default. When you use `Runner.run()`, it automatically creates a trace that records the entire agent run, including LLM calls, tool executions, and agent handoffs . You do not need to write additional code for this basic level of logging.
- **Integrate with Observability Platforms**: For more powerful analysis, visualization, and long-term storage, you can export these traces to specialized platforms like **Langfuse** . These platforms help you search through logs, visualize the chain of events, and monitor metrics like latency and cost in real-time .
- **Create Custom Spans**: For tracking specific business logic or external operations (like database saves or API calls), you can use the `custom_span()` function. This allows you to inject your own detailed steps into the agent's trace .
- **Manage Privacy and Control**: You can disable tracing globally via an environment variable (`OPENAI_AGENTS_DISABLE_TRACING=1`) or for a single run using the `RunConfig` . To protect sensitive information, you can set `trace_include_sensitive_data=False` in the run configuration, which will prevent the SDK from recording tool inputs and outputs .

### 🔒 Security, Compliance, and Best Practices

When implementing audit logs for production systems, especially in regulated industries, it's crucial to adhere to several best practices :

- **Ensure Log Immutability and Integrity**: Protect logs from tampering by using write-once-read-many (WORM) storage, encryption, or hash-chaining techniques to make any alterations evident . This is vital for the logs to serve as reliable evidence.
- **Implement Strict Access Controls**: Restrict access to audit logs to a small number of authorized personnel to prevent unauthorized viewing or modification .
- **Define and Enforce Retention Policies**: Retain logs for a period that meets both your organizational needs and regulatory requirements (e.g., 90 days is common, but some regulations require 6+ years) .
- **Use Centralized Log Management**: Aggregate logs from all components of your system into a central platform. This makes correlation, analysis, and monitoring much more efficient .
