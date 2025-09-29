
### 🛡️ Understanding Guardrails

Guardrails act as programmable safety checks that run in parallel to your main agent. They are categorized based on when they act during the agent's processing cycle.

| Guardrail Type | Purpose | Execution Timing |
| :--- | :--- | :--- |
| **Input Guardrails** | Validate, filter, or screen the initial user input. | Run **before** the main agent processes the request. |
| **Output Guardrails** | Validate, filter, or screen the agent's final response. | Run **after** the main agent has generated its output. |

The primary goal of input guardrails is to **prevent misuse and save resources** by stopping off-topic or malicious queries before they reach a potentially expensive main agent. Output guardrails serve to **ensure safety and compliance**, for instance, by preventing the leakage of sensitive information like Personally Identifiable Information (PII) in the agent's responses.

### ⚙️ Implementing Guardrails

Here is how you can implement both LLM-based and rule-based guardrails in your application.

**1. Input Guardrail: Detecting Off-Topic Queries**

You can create an LLM-based guardrail to determine if a user's question is outside your agent's purpose. First, define a Pydantic model to structure the guardrail agent's output.

```python
from pydantic import BaseModel, Field

class TopicClassificationOutput(BaseModel):
    is_off_topic: bool = Field(
        description="True if the input is off-topic, False otherwise"
    )
    reasoning: str = Field(
        description="Brief explanation of the classification"
    )
```

Next, create a guardrail agent that uses a cost-effective model. This agent is given clear instructions on what constitutes on-topic and off-topic queries.

```python
from agents import Agent

topic_classification_agent = Agent(
    name="Topic Classification Agent",
    instructions=(
        "You are a topic classifier for a weather and air quality application. "
        "Your task is to determine if a user's question is on-topic... "
        "Mark as OFF-TOPIC only if the query is clearly unrelated..."
    ),
    output_type=TopicClassificationOutput,
    model="gpt-4o-mini"  # Use a fast, cost-effective model
)
```

Finally, wrap the agent in an async function decorated with `@input_guardrail`. This function runs the classification agent and triggers a tripwire if the input is off-topic.

```python
from agents import input_guardrail, GuardrailFunctionOutput, Runner

@input_guardrail
async def off_topic_guardrail(ctx, agent, input) -> GuardrailFunctionOutput:
    result = await Runner.run(topic_classification_agent, input, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output.reasoning,
        tripwire_triggered=result.final_output.is_off_topic  # Triggers if True
    )
```

**2. Input Guardrail: Rule-Based Injection Detection**

For predictable patterns like prompt injection attempts, a simpler rule-based guardrail is effective. This example checks for dangerous keywords without using an LLM.

```python
@input_guardrail
async def injection_detection_guardrail(ctx, agent, input) -> GuardrailFunctionOutput:
    injection_patterns = [
        "ignore previous instructions",
        "you are now a",
        "forget everything above",
        "developer mode",
        "override safety"
    ]
    tripwire_triggered = any(pattern in input.lower() for pattern in injection_patterns)
    
    return GuardrailFunctionOutput(
        output_info="Potential injection attempt detected" if tripwire_triggered else "",
        tripwire_triggered=tripwire_triggered
    )
```

**3. Implementing an Output Guardrail**

Output guardrails follow the same pattern but inspect the agent's final output. For example, you could create a guardrail to check if the agent's response contains PII. First, define the guardrail agent's output structure, then create the agent and the guardrail function.

```python
from agents import output_guardrail

class PiiDetectionOutput(BaseModel):
    contains_pii: bool
    reasoning: str

# Create the output guardrail agent
pii_detection_agent = Agent(
    name="PII Detection Agent",
    instructions="Check if the provided text contains Personally Identifiable Information (PII) such as emails, phone numbers, or government IDs.",
    output_type=PiiDetectionOutput
)

@output_guardrail
async def pii_guardrail(ctx, agent, output) -> GuardrailFunctionOutput:
    result = await Runner.run(pii_detection_agent, output, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output.reasoning,
        tripwire_triggered=result.final_output.contains_pii  # Triggers if True
    )
```

### 🚨 The Tripwire Mechanism

The core of the guardrail system is the **tripwire**. If any guardrail sets `tripwire_triggered=True`, the SDK immediately halts the agent's execution by raising a specific exception:
- An `InputGuardrailTripwireTriggered` exception for input guardrail violations.
- An `OutputGuardrailTripwireTriggered` exception for output guardrail violations.

This mechanism ensures that invalid inputs or unsafe outputs stop the process early, protecting your system and saving resources.

### 📜 Compliance in Multi-Agent Systems

In more complex, multi-agent systems, ensuring compliance involves additional challenges and requires a structured technical framework.

- **Distributed Accountability**: It can be difficult to trace how a decision was made across multiple autonomous agents. Implementing **tamper-evident logging** that captures the identity, actions, and full sequence of interactions for each agent is crucial for audits.
- **Cross-Agent Data Governance**: Data flowing between agents can lead to compliance issues (e.g., with GDPR or HIPAA). Implement **persistent data classification tags** that travel with the data and policy enforcement points between agents to verify handling permissions.
- **Inconsistent Security**: Different agents may have different security postures. A federated identity and access management system can enforce the **principle of least privilege** across all agent interactions.
- **Emergent Behavior**: Unpredictable behaviors can arise from complex agent interactions. **Automated continuous monitoring and chaos engineering** can help detect compliance drift and test system resilience under failure.

