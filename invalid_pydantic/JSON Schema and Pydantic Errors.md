# JSON Schema and Pydantic Errors in OpenAI Agents SDK

This guide covers common JSON Schema and Pydantic errors encountered when using the OpenAI Agents SDK, along with comprehensive solutions and best practices.

## 🎯 Common Error Patterns

### Schema Validation Errors
| Error Type | Root Cause | Typical Error Message |
| :--- | :--- | :--- |
| **Additional Properties** | Using `Dict` types without explicit schema rules | `'additionalProperties' is required to be supplied and to be false` |
| **Union Types** | Using `List[Union[...]]` in Pydantic models | `'oneOf' is not permitted within the items definition` |
| **Default Values** | Fields with default values in output models | `'default' is not permitted within a property definition` |
| **Schema Complexity** | Nested models with advanced Pydantic features | Various schema validation failures |
