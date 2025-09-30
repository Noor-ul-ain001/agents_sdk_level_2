In the OpenAI Agents SDK, dynamic instructions allow you to programmatically generate an agent's system prompt at runtime, enabling you to create context-aware and highly adaptive agents.

### 🛠️ How Dynamic Instructions Work

Instead of providing a static string for the `instructions` parameter when creating an agent, you can provide a function. This function is executed just before the agent runs, and it must return the prompt string that will be used for that specific run.

The function can be synchronous or asynchronous (`async`), and it receives two key pieces of information that provide the context needed to build the dynamic prompt:

| Parameter | Description |
| :--- | :--- |
| **`agent`** | The agent object that is about to run. You can access its properties, such as `agent.name`. |
| **`context`** | A dependency-injection object (the `ctx` parameter you pass to `Runner.run()`). This is a "grab bag" for any dependencies or state, such as user data or conversation history. |

### 💡 Implementation Guide and Use Cases

Here is a basic template for using dynamic instructions:

```python
# Example of a dynamic instructions function
def my_dynamic_instructions(agent, context):
    # Your logic to generate instructions based on the agent and context
    dynamic_prompt = f"Custom instructions based on context: {context}"
    return dynamic_prompt

# Create an agent with dynamic instructions
my_agent = Agent(
    name="My Dynamic Agent",
    instructions=my_dynamic_instructions,  # Pass the function, don't call it
    model="gpt-4"
)
```

Dynamic instructions are powerful for several real-world applications:

- **Personalizing Agent Behavior**: You can tailor the agent's instructions based on user profile information stored in the context, such as user preferences, role, or past interactions.
- **Creating Context-Aware Handoffs**: When building a multi-agent system with handoffs, you can generate specialized instructions for the handoff agent on the fly, incorporating information from the main conversation context.
- **Adapting to Real-Time Data**: The instructions can incorporate real-time data fetched or computed within the function, allowing the agent to act on the most current information available.

