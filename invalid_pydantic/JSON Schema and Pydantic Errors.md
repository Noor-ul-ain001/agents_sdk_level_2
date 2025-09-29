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

## 🛠️ Solutions and Workarounds

### 1. Fixing AdditionalProperties Errors

**Problem**: Using `Dict` types generates schemas that violate OpenAI API requirements.

```python
from pydantic import BaseModel
from typing import Dict, Any
from agents import Agent

# ❌ Problematic: Dict type without schema control
class ProblematicOutput(BaseModel):
    metadata: Dict[str, Any]  # This will cause additionalProperties errors
    data: dict

# ✅ Solution 1: Use explicit models instead of Dict
class ExplicitMetadata(BaseModel):
    source: str
    timestamp: str
    version: str

class FixedOutput(BaseModel):
    metadata: ExplicitMetadata  # Explicit structure
    data: dict  # Still problematic, but less so

# ✅ Solution 2: Manual schema modification for tools
from agents import FunctionTool
import json

def create_tool_with_fixed_schema():
    # Generate schema from model
    schema = FixedOutput.model_json_schema()
    
    # Manually fix additionalProperties in the schema
    if 'properties' in schema:
        for prop_name, prop_schema in schema['properties'].items():
            if 'type' in prop_schema and prop_schema['type'] == 'object':
                prop_schema['additionalProperties'] = False
    
    schema['additionalProperties'] = False
    
    return schema

# ✅ Solution 3: Use strict mode in function_tool
from agents import function_tool

@function_tool(strict_mode=True)  # Enforces strict JSON schema
def safe_tool(input_param: str) -> str:
    return f"Processed: {input_param}"
```

### 2. Handling Union Type Restrictions

**Problem**: `Union` types and complex generics generate unsupported `oneOf` schemas.

```python
from typing import Union, List, Optional
from pydantic import BaseModel, Field

# ❌ Problematic: Union types in lists
class ProblematicModel(BaseModel):
    items: List[Union[str, int]]  # Generates oneOf in schema
    value: Union[str, int]

# ✅ Solution 1: Flatten to common denominator
class SafeModel(BaseModel):
    items: List[str]  # Convert all to string representation
    value: str
    
    @classmethod
    def from_complex(cls, complex_data):
        # Handle conversion in code, not in schema
        return cls(
            items=[str(item) for item in complex_data['items']],
            value=str(complex_data['value'])
        )

# ✅ Solution 2: Use discriminators for typed unions
from typing import Literal
from pydantic import Field

class StringItem(BaseModel):
    type: Literal["string"] = "string"
    value: str

class NumberItem(BaseModel):
    type: Literal["number"] = "number" 
    value: float

class SafeUnionModel(BaseModel):
    items: List[Union[StringItem, NumberItem]] = Field(discriminator="type")
```

### 3. Managing Default Value Issues

**Problem**: Default values in Pydantic models violate OpenAI schema constraints.

```python
from pydantic import BaseModel, Field
from typing import List

# ❌ Problematic: Default values in output models
class ProblematicOutput(BaseModel):
    name: str = "default_name"  # Causes 'default not permitted'
    tags: List[str] = Field(default_factory=list)
    count: int = 0

# ✅ Solution 1: Remove defaults from output models
class CleanOutput(BaseModel):
    name: str
    tags: List[str]
    count: int

# ✅ Solution 2: Use factory functions instead
def create_default_output() -> CleanOutput:
    return CleanOutput(
        name="default_name",
        tags=[],
        count=0
    )

# ✅ Solution 3: Handle defaults in agent instructions
default_agent = Agent(
    name="Default Handler",
    instructions="""Always provide values for all fields. 
    Use 'unknown' for name if not specified, 
    empty list for tags, and 0 for count.""",
    output_type=CleanOutput
)
```

## 🔧 Advanced Schema Management

### Schema Sanitization Utility

```python
import json
from typing import Type, Dict, Any
from pydantic import BaseModel

class SchemaSanitizer:
    """Sanitizes Pydantic JSON schemas for OpenAI compatibility."""
    
    @classmethod
    def sanitize_schema(cls, model: Type[BaseModel]) -> Dict[str, Any]:
        """Convert Pydantic model to OpenAI-compatible JSON schema."""
        schema = model.model_json_schema()
        
        # Remove problematic fields
        schema = cls._remove_defaults(schema)
        schema = cls._fix_additional_properties(schema)
        schema = cls._simplify_unions(schema)
        
        return schema
    
    @staticmethod
    def _remove_defaults(schema: Dict[str, Any]) -> Dict[str, Any]:
        """Remove default values from schema properties."""
        if 'properties' in schema:
            for prop in schema['properties'].values():
                prop.pop('default', None)
                # Handle nested objects
                if 'properties' in prop:
                    SchemaSanitizer._remove_defaults(prop)
        return schema
    
    @staticmethod
    def _fix_additional_properties(schema: Dict[str, Any]) -> Dict[str, Any]:
        """Set additionalProperties to false where needed."""
        schema['additionalProperties'] = False
        
        if 'properties' in schema:
            for prop_name, prop_schema in schema['properties'].items():
                if (isinstance(prop_schema, dict) and 
                    prop_schema.get('type') == 'object'):
                    prop_schema['additionalProperties'] = False
                    
                # Handle arrays of objects
                if (isinstance(prop_schema, dict) and 
                    prop_schema.get('type') == 'array' and
                    'items' in prop_schema and
                    isinstance(prop_schema['items'], dict) and
                    prop_schema['items'].get('type') == 'object'):
                    prop_schema['items']['additionalProperties'] = False
        
        return schema
    
    @staticmethod
    def _simplify_unions(schema: Dict[str, Any]) -> Dict[str, Any]:
        """Convert union types to more compatible formats."""
        def _process_schema(s):
            if 'oneOf' in s:
                # For unions, take the first option or convert to string
                if all(isinstance(item, dict) and 'type' in item for item in s['oneOf']):
                    # If all options are simple types, use string
                    s['type'] = 'string'
                    del s['oneOf']
                else:
                    # For complex unions, use the first schema
                    first_option = s['oneOf'][0]
                    s.update(first_option)
                    del s['oneOf']
            return s
        
        return SchemaSanitizer._traverse_schema(schema, _process_schema)
    
    @staticmethod
    def _traverse_schema(schema: Dict[str, Any], func: callable) -> Dict[str, Any]:
        """Recursively apply function to schema."""
        schema = func(schema)
        
        if 'properties' in schema:
            for key, value in schema['properties'].items():
                if isinstance(value, dict):
                    schema['properties'][key] = SchemaSanitizer._traverse_schema(value, func)
        
        if 'items' in schema and isinstance(schema['items'], dict):
            schema['items'] = SchemaSanitizer._traverse_schema(schema['items'], func)
            
        return schema

# Usage example
sanitized_schema = SchemaSanitizer.sanitize_schema(YourPydanticModel)
```

### Custom Tool with Manual Schema Control

```python
from agents import FunctionTool
from pydantic import BaseModel, create_model
import inspect

def create_safe_tool(func: callable, model: Type[BaseModel] = None):
    """Create a tool with OpenAI-compatible schema."""
    
    # Get function signature
    sig = inspect.signature(func)
    
    # Create simplified model if needed
    if model is None:
        fields = {}
        for param_name, param in sig.parameters.items():
            if param_name != 'self':
                # Convert complex types to string for simplicity
                fields[param_name] = (str, ...)
        
        model = create_model(f"{func.__name__}Input", **fields)
    
    # Sanitize the schema
    sanitized_schema = SchemaSanitizer.sanitize_schema(model)
    
    # Create the tool
    tool = FunctionTool(
        name=func.__name__,
        description=func.__doc__ or f"Tool for {func.__name__}",
        params_json_schema=sanitized_schema,
        on_invoke_tool=lambda params: func(**params)
    )
    
    return tool

# Usage
def complex_processing(data: dict, options: list) -> str:
    """Process complex data with options."""
    return f"Processed {len(data)} items with {len(options)} options"

# This will create a tool with a safe schema
safe_tool = create_safe_tool(complex_processing)
```

## 🚀 Production Best Practices

### 1. Schema Testing and Validation

```python
import pytest
from pydantic import ValidationError
from agents import Runner

def validate_schema_compatibility(model: Type[BaseModel]):
    """Test that a Pydantic model produces OpenAI-compatible schema."""
    schema = SchemaSanitizer.sanitize_schema(model)
    
    # Check for prohibited features
    assert 'default' not in str(schema), "Schema contains default values"
    assert 'oneOf' not in str(schema), "Schema contains oneOf (union types)"
    
    # Verify additionalProperties is properly set
    def check_additional_properties(s):
        if isinstance(s, dict):
            if s.get('type') == 'object' and 'properties' in s:
                assert s.get('additionalProperties') is False, \
                    "Object properties must have additionalProperties: false"
            # Check nested structures
            for value in s.values():
                if isinstance(value, (dict, list)):
                    check_additional_properties(value)
        elif isinstance(s, list):
            for item in s:
                check_additional_properties(item)
    
    check_additional_properties(schema)
    return True

# Test your models
def test_output_models():
    assert validate_schema_compatibility(CleanOutput)
    assert validate_schema_compatibility(SafeModel)
```

### 2. Error Recovery Patterns

```python
from typing import Type, Optional
import json

class SchemaErrorHandler:
    """Handle schema-related errors gracefully."""
    
    @staticmethod
    def handle_parsing_error(
        raw_output: str, 
        model: Type[BaseModel],
        max_attempts: int = 2
    ) -> Optional[BaseModel]:
        """Attempt to recover from schema parsing errors."""
        
        for attempt in range(max_attempts):
            try:
                # Try direct parsing first
                if raw_output.strip().startswith('{'):
                    data = json.loads(raw_output)
                    return model(**data)
                
                # Try to extract JSON from text
                json_match = re.search(r'\{.*\}', raw_output, re.DOTALL)
                if json_match:
                    data = json.loads(json_match.group())
                    return model(**data)
                    
            except (json.JSONDecodeError, ValidationError) as e:
                if attempt == max_attempts - 1:
                    # Last attempt failed, return None or raise
                    return None
                # Log and continue to next attempt
                print(f"Parsing attempt {attempt + 1} failed: {e}")
        
        return None
    
    @staticmethod
    def create_fallback_model(model: Type[BaseModel], error_info: str):
        """Create a fallback instance when parsing fails."""
        # Create minimal valid instance
        fields = {}
        for field_name, field in model.__fields__.items():
            if hasattr(field, 'default') and field.default is not None:
                fields[field_name] = field.default
            else:
                # Provide type-appropriate empty values
                if field.type_ == str:
                    fields[field_name] = f"Error: {error_info}"
                elif field.type_ == list:
                    fields[field_name] = []
                elif field.type_ == dict:
                    fields[field_name] = {}
                else:
                    fields[field_name] = None
        
        return model(**fields)
```

### 3. Monitoring and Analytics

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Dict, Any

@dataclass
class SchemaErrorMetrics:
    """Track schema-related errors for monitoring."""
    error_type: str
    model_name: str
    timestamp: datetime
    error_message: str
    raw_output_preview: str
    
    @classmethod
    def record_error(cls, error: Exception, model: Type[BaseModel], raw_output: str):
        """Record schema error for analytics."""
        error_metrics = cls(
            error_type=type(error).__name__,
            model_name=model.__name__,
            timestamp=datetime.now(),
            error_message=str(error),
            raw_output_preview=raw_output[:200] + '...' if len(raw_output) > 200 else raw_output
        )
        
        # Send to your monitoring system
        # monitoring_system.record(error_metrics)
        
        return error_metrics

# Usage in agent execution
try:
    result = await Runner.run(agent, input_text)
    output = safe_parse_output(result.final_output, OutputModel)
except (ValidationError, json.JSONDecodeError) as e:
    metrics = SchemaErrorMetrics.record_error(e, OutputModel, result.final_output)
    # Implement fallback logic
    output = SchemaErrorHandler.create_fallback_model(OutputModel, str(e))
```
