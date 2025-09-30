
Here is a summary of the common errors and their solutions:

| Error Category | Specific Error/Symptom | Root Cause | Recommended Solution |
| :--- | :--- | :--- | :--- |
| **Schema Compatibility** | `'additionalProperties' is required to be... false` | Using `Dict` type or a field with a `dict` type without explicit schema rules. | Manually set `"additionalProperties": False` in the JSON schema. |
| **Schema Compatibility** | `'oneOf' is not permitted` | Using a Pydantic model with a `List` of a `Union` type, which generates an unsupported `oneOf` in the schema. | Avoid using `List[Union[...]]` in your `output_type` models; this may require a model redesign. |
| **Schema Compatibility** | `'default' is not permitted` | Defining a field with a default value (e.g., `my_field: str = "default"`) in your Pydantic model. | Remove default values from fields in models used for `output_type`. |
| **Output Truncation** | `Invalid JSON: EOF while parsing...` at ~6000 chars | Large outputs (e.g., from `WebSearchTool`) cause the model's response to be cut off mid-generation. | Set `max_tokens` in `ModelSettings` to a high value (e.g., 128,000) or instruct the model to keep output concise. |
| **SDK Version** | `'MockValSer' object cannot be converted...` | Version mismatch; the `openai-agents` 0.1.0 package was built against Pydantic 2.10.x but a newer version (e.g., 2.11.7) is installed. | Ensure your environment uses a compatible version of Pydantic. |


- **Handling Complex Schemas**: For Pydantic models that use advanced features like unions with discriminators or nested models, the automatically generated schema is often incompatible. The only robust workaround currently is to avoid these features in your `output_type` models. For scenarios requiring complex, validated outputs, you might need to fall back to prompt-based JSON generation and manual validation outside the `output_type` mechanism.

- **Using a Schema Sanitizer**: You can use a battle-tested sanitizer to automatically transform any Pydantic model into an OpenAI-compatible schema. This tool handles the conversion of optional fields, removal of numeric constraints, and fixing of `additionalProperties`. This allows you to keep your existing Pydantic models without extensive modification.

- **Managing Large Outputs and Truncation**: If you are using tools like `WebSearchTool` alongside structured outputs and encounter JSON truncation, try setting `max_tokens` in the `ModelSettings` to a high value (e.g., 128,000). Alternatively, you can add instructions in your prompt asking the model to keep the output below a specific character count, such as 5500 characters.


