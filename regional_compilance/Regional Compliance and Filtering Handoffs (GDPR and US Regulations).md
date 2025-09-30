In the OpenAI Agents SDK, **handoffs** allow an agent to transfer a task to another specialized agent. You can implement **regional compliance filtering** by using features like `input_filter` to control the data and context passed during these transfers to adhere to regulations like GDPR.

### 🔧 How Handoffs and Filtering Work

In the SDK, a handoff to an agent named "Billing Agent" is presented to the LLM as a tool called `transfer_to_billing_agent`. You customize this process using the `handoff()` function, with the `input_filter` parameter being key for compliance.

- **`input_filter`**: This is a function that lets you modify the conversation history and context passed to the next agent. For compliance, you can use it to **strip out sensitive information, personal data, or entire tool calls** from the history before the handoff is complete. The SDK provides common helper functions in its extensions library to make this easier.

- **Dynamic Control (`is_enabled`)**: You can dynamically enable or disable a handoff tool based on the runtime context. This allows you to programmatically prevent a handoff to an agent in a region where it should not operate due to compliance rules.

- **Prompt Engineering**: OpenAI recommends including instructions about handoffs in your agent's prompt. Using the provided `RECOMMENDED_PROMPT_PREFIX` can help the LLM understand when and how to use handoffs correctly.

### 💡 A Practical Framework for Compliance

Building compliant multi-agent systems involves more than just SDK features. Here is a checklist to guide your development process, synthesizing general best practices for compliance.

| Principle | Key Actions | Example Tools / Techniques |
| :--- | :--- | :--- |
| **Risk Classification & Governance** | Map AI components to risk levels; enforce stricter governance for high-risk areas (e.g., finance, HR). | Data classification with Microsoft Purview. |
| **Transparency & Auditability** | Maintain logs for all AI decisions and overrides; clearly label AI-generated outputs. | Telemetry with Azure Monitor; use of Responsible AI Dashboard. |
| **Data Governance & Security** | Apply sensitivity labels; enforce access controls; restrict agents to approved data sources. | Data Loss Prevention (DLP) policies; Azure Key Vault. |
| **Human Oversight** | Define roles for review and override; create automated escalation workflows. | Power Automate for escalation workflows; Omnichannel for live handoff. |
| **Technical Safeguards** | Use input/output guardrails to filter content and prevent PII leaks. | Amazon Bedrock Guardrails; Azure AI Content Safety. |

### ⚠️ Important Limitations and Considerations

- **No Built-In Regional Filters**: The OpenAI Agents SDK does not come with pre-built filters for GDPR or other region-specific regulations. You must **design, implement, and test these compliance filters yourself** based on your specific data and legal requirements.
- **Legal Compliance is Your Responsibility**: The SDK provides the technical tools, but ensuring your application fully complies with GDPR, US regulations, or the EU AI Act is your responsibility. This often requires a combination of technical controls, clear documentation, and defined processes.

