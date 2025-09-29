
### 🛠️ What is the `@function_tool` Decorator?

The `@function_tool` decorator transforms a standard Python function into a tool that an AI agent can use. It automatically generates a JSON schema for the tool's parameters by inspecting the function's signature, type hints, and docstring. This schema tells the LLM what the tool does and how to call it correctly.

### ⚙️ How to Use the Decorator and Configure It

Using the decorator is straightforward. Here is a basic example of creating a tool:

```python
from agents import function_tool

@function_tool
def get_weather(city: str) -> str:
    """Get the current weather for a given city.

    Args:
        city: The name of the city to get the weather for.
    """
    return f"The weather in {city} is sunny."
```

You can then add this tool to an agent by including it in the agent's `tools` list.

The decorator also accepts several parameters for customization:

| Parameter | Description | Default Behavior |
| :--- | :--- | :--- |
| `name_override` | Overrides the tool's name. | Uses the function's name. |
| `description_override` | Overrides the tool's description. | Uses the function's docstring. |
| `use_docstring_info` | Controls whether docstring info is used. | `True` (enabled). |
| `strict_mode` | Uses strict JSON schema. | `True` (recommended). |
| `is_enabled` | Dynamically enables/disables the tool. | `True` (enabled). |

Here is an example of using some of these parameters:
```python
@function_tool(
    name_override="BudgetCalculator",
    description_override="Calculates travel expenses for a trip.",
    strict_mode=True
)
def complex_budget_calculator(destination: str, days: int, travel_style: str) -> float:
    # ... function logic ...
    return estimated_cost
```

For functions that require more complex input, you can use a `TypedDict` to define a structured input schema:
```python
from typing_extensions import TypedDict

class TripInfo(TypedDict):
    """Information about a trip."""
    destination: str
    days: int

@function_tool
def estimate_budget(trip_info: TripInfo) -> float:
    """Estimate the budget for a trip."""
    base_cost = 150
    cost_multipliers = {"Norway": 2.0, "Spain": 1.2}
    multiplier = cost_multipliers.get(trip_info["destination"], 1.5)
    return base_cost * trip_info["days"] * multiplier
```

### 🔍 How Schema Generation Works

The `function_schema()` function powering the decorator performs several key steps to generate the tool's schema:

1.  **Docstring Parsing**: If `use_docstring_info=True`, it extracts the function's description and parameter descriptions from its docstring. It auto-detects the docstring style but allows you to specify it with the `docstring_style` parameter.
2.  **Signature Inspection**: It uses Python's `inspect.signature` to get the function's parameters and their default values.
3.  **Type Hint Processing**: It processes type hints for parameters and can also extract descriptions from `Annotated` types.
4.  **Pydantic Model Creation**: It creates a Pydantic model where each function parameter becomes a field. This model is used for validation.
5.  **JSON Schema Generation**: Finally, it generates a strict JSON schema from the Pydantic model, which is sent to the LLM.

It is strongly recommended to keep **strict JSON schema mode enabled** (the default) as it constrains the schema to features that guarantee valid JSON, significantly increasing the likelihood of the LLM producing correct input.

### ⚡ Advanced Usage and Features

-   **Dynamic Tool Activation**: The `is_enabled` parameter can be a callable that dynamically decides whether the tool is active based on the run context or the agent's state.
-   **Input and Output Guardrails**: You can equip your tools with `tool_input_guardrails` and `tool_output_guardrails` to run validation checks before and after the tool is invoked.
-   **Handling Context**: A function can access the run context by including a `RunContextWrapper` or `ToolContext` parameter as its **first argument**. This provides access to the current session, agent, and other state.
