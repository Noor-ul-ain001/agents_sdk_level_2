# Output Type Parsing and Handling in OpenAI Agents SDK

The OpenAI Agents SDK provides robust mechanisms for handling structured outputs through Pydantic models, including comprehensive error handling for invalid JSON and Pydantic validation errors.

## 🎯 Core Concepts

### Output Type System Overview
| Component | Purpose | Error Handling |
| :--- | :--- | :--- |
| **Pydantic Models** | Define structured output schemas for agent responses | Automatic validation with detailed error messages |
| **JSON Parsing** | Convert LLM text responses to structured data | Graceful handling of malformed JSON with retry logic |
| **Validation Pipeline** | Multi-stage parsing and validation process | Context-aware error recovery and user feedback |

## 🛠️ Implementation Patterns

### Basic Pydantic Model Definition
```python
from pydantic import BaseModel, Field, validator
from typing import List, Optional
from datetime import datetime
import json

class ResearchSummary(BaseModel):
    """Structured output for research agent."""
    topic: str = Field(description="The main research topic")
    key_findings: List[str] = Field(description="List of key discoveries")
    confidence_score: float = Field(ge=0.0, le=1.0, description="Confidence in findings")
    sources: List[str] = Field(default_factory=list, description="Data sources used")
    timestamp: datetime = Field(default_factory=datetime.now)
    
    @validator('key_findings')
    def validate_findings(cls, v):
        if len(v) < 1:
            raise ValueError('At least one key finding is required')
        if len(v) > 10:
            raise ValueError('Maximum 10 key findings allowed')
        return v
    
    @validator('confidence_score')
    def validate_confidence(cls, v):
        if v < 0.5:
            # Could trigger different handling for low-confidence responses
            print("Warning: Low confidence score detected")
        return v

# Agent with structured output
research_agent = Agent(
    name="Research Analyst",
    instructions="Analyze topics and provide structured research summaries.",
    output_type=ResearchSummary
)
```

### Advanced Error Handling Patterns
```python
from agents import Agent, Runner
from pydantic import ValidationError
import json
from typing import Type, TypeVar

T = TypeVar('T', bound=BaseModel)

class OutputParsingError(Exception):
    """Custom exception for output parsing failures."""
    def __init__(self, original_error: Exception, raw_output: str, model_type: Type[BaseModel]):
        self.original_error = original_error
        self.raw_output = raw_output
        self.model_type = model_type
        super().__init__(f"Failed to parse output as {model_type.__name__}: {original_error}")

def safe_parse_output(raw_output: str, model_type: Type[T]) -> T:
    """
    Safely parse raw LLM output into Pydantic model with comprehensive error handling.
    
    Args:
        raw_output: Raw text output from LLM
        model_type: Pydantic model class for parsing
        
    Returns:
        Parsed model instance
        
    Raises:
        OutputParsingError: If parsing fails
    """
    try:
        # Attempt to extract JSON from markdown code blocks
        cleaned_output = extract_json_from_markdown(raw_output)
        
        # Parse JSON
        if cleaned_output:
            json_data = json.loads(cleaned_output)
        else:
            # Fallback: try to parse entire output as JSON
            json_data = json.loads(raw_output)
        
        # Validate against Pydantic model
        return model_type(**json_data)
        
    except json.JSONDecodeError as e:
        raise OutputParsingError(e, raw_output, model_type)
    except ValidationError as e:
        raise OutputParsingError(e, raw_output, model_type)
    except Exception as e:
        raise OutputParsingError(e, raw_output, model_type)

def extract_json_from_markdown(text: str) -> Optional[str]:
    """Extract JSON from markdown code blocks."""
    import re
    
    # Match JSON in ```json ``` or ``` ``` blocks
    json_pattern = r'```(?:json)?\s*(.*?)\s*```'
    matches = re.findall(json_pattern, text, re.DOTALL)
    
    if matches:
        return matches[0].strip()
    
    # Try to find JSON object or array pattern
    json_obj_pattern = r'\{.*\}'
    json_arr_pattern = r'\[.*\]'
    
    for pattern in [json_obj_pattern, json_arr_pattern]:
        match = re.search(pattern, text, re.DOTALL)
        if match:
            return match.group()
    
    return None
```

## 🔧 Retry Mechanisms and Recovery

### Intelligent Retry with Error Feedback
```python
from typing import Optional, Callable
import asyncio

class OutputParserWithRetry:
    """Enhanced output parser with automatic retry logic."""
    
    def __init__(
        self,
        max_retries: int = 3,
        retry_delay: float = 1.0,
        exponential_backoff: bool = True
    ):
        self.max_retries = max_retries
        self.retry_delay = retry_delay
        self.exponential_backoff = exponential_backoff
    
    async def parse_with_retry(
        self,
        agent: Agent,
        input_text: str,
        context: Optional[dict] = None,
        error_callback: Optional[Callable] = None
    ) -> BaseModel:
        """
        Run agent with retry logic for output parsing failures.
        """
        last_error = None
        
        for attempt in range(self.max_retries):
            try:
                result = await Runner.run(agent, input_text, context=context)
                
                # Attempt to parse the output
                if hasattr(agent, 'output_type') and agent.output_type:
                    parsed_output = safe_parse_output(
                        result.final_output, 
                        agent.output_type
                    )
                    return parsed_output
                else:
                    return result.final_output
                    
            except OutputParsingError as e:
                last_error = e
                
                # Call error callback if provided
                if error_callback:
                    error_callback(e, attempt)
                
                # Calculate delay with exponential backoff
                delay = self.retry_delay
                if self.exponential_backoff:
                    delay *= (2 ** attempt)
                
                print(f"Parsing failed on attempt {attempt + 1}. Retrying in {delay}s...")
                await asyncio.sleep(delay)
                
                # Enhance context with error information for retry
                enhanced_context = context or {}
                enhanced_context['previous_errors'] = enhanced_context.get('previous_errors', [])
                enhanced_context['previous_errors'].append(str(e))
        
        # All retries failed
        raise OutputParsingError(
            last_error.original_error if last_error else Exception("Max retries exceeded"),
            last_error.raw_output if last_error else "",
            agent.output_type
        )

# Usage example
async def run_research_with_retry(topic: str) -> ResearchSummary:
    parser = OutputParserWithRetry(max_retries=3)
    
    def error_handler(error: OutputParsingError, attempt: int):
        print(f"Attempt {attempt} failed: {error}")
        # Could log to monitoring system here
    
    return await parser.parse_with_retry(
        research_agent,
        f"Research topic: {topic}",
        error_callback=error_handler
    )
```

## ⚡ Advanced Validation Strategies

### Multi-Stage Validation Pipeline
```python
from pydantic import BaseModel, validator
import re

class ValidationPipeline:
    """Multi-stage validation for robust output handling."""
    
    @staticmethod
    def pre_validate_json(raw_text: str) -> dict:
        """Stage 1: Basic JSON validation and cleaning."""
        try:
            # Remove common LLM artifacts
            cleaned = raw_text.strip()
            cleaned = re.sub(r'^```json\s*', '', cleaned)
            cleaned = re.sub(r'\s*```$', '', cleaned)
            
            return json.loads(cleaned)
        except json.JSONDecodeError as e:
            raise OutputParsingError(e, raw_text, dict)
    
    @staticmethod
    def validate_required_fields(json_data: dict, required_fields: list) -> dict:
        """Stage 2: Check for required fields."""
        missing_fields = [field for field in required_fields if field not in json_data]
        if missing_fields:
            raise OutputParsingError(
                ValueError(f"Missing required fields: {missing_fields}"),
                json.dumps(json_data),
                dict
            )
        return json_data
    
    @staticmethod
    def validate_field_types(json_data: dict, field_specs: dict) -> dict:
        """Stage 3: Basic type validation."""
        for field, expected_type in field_specs.items():
            if field in json_data and not isinstance(json_data[field], expected_type):
                raise OutputParsingError(
                    ValueError(f"Field {field} should be {expected_type}"),
                    json.dumps(json_data),
                    dict
                )
        return json_data

def robust_output_parsing(raw_output: str, model_type: Type[BaseModel]) -> BaseModel:
    """
    Comprehensive output parsing with multi-stage validation.
    """
    # Stage 1: JSON parsing
    json_data = ValidationPipeline.pre_validate_json(raw_output)
    
    # Stage 2: Get required fields from Pydantic model
    required_fields = [
        name for name, field in model_type.__fields__.items() 
        if field.required
    ]
    json_data = ValidationPipeline.validate_required_fields(json_data, required_fields)
    
    # Stage 3: Pydantic model validation
    try:
        return model_type(**json_data)
    except ValidationError as e:
        raise OutputParsingError(e, raw_output, model_type)
```

## 🚀 Production-Ready Error Handling

### Comprehensive Error Recovery System
```python
from enum import Enum
from typing import Any, Dict, Optional

class ParserErrorType(Enum):
    INVALID_JSON = "invalid_json"
    MISSING_FIELDS = "missing_fields"
    TYPE_MISMATCH = "type_mismatch"
    VALIDATION_ERROR = "validation_error"
    UNKNOWN = "unknown"

class OutputParser:
    """Production-ready output parser with detailed error classification."""
    
    def __init__(self, enable_fallback: bool = True):
        self.enable_fallback = enable_fallback
        self.error_stats = {error_type: 0 for error_type in ParserErrorType}
    
    def classify_error(self, error: Exception) -> ParserErrorType:
        """Classify parsing errors for appropriate handling."""
        if isinstance(error, json.JSONDecodeError):
            return ParserErrorType.INVALID_JSON
        elif isinstance(error, ValidationError):
            error_str = str(error)
            if "field required" in error_str:
                return ParserErrorType.MISSING_FIELDS
            elif "type" in error_str and "error" in error_str:
                return ParserErrorType.TYPE_MISMATCH
            else:
                return ParserErrorType.VALIDATION_ERROR
        else:
            return ParserErrorType.UNKNOWN
    
    def create_fallback_response(self, model_type: Type[BaseModel], original_error: Exception) -> BaseModel:
        """Create fallback response when parsing fails completely."""
        if not self.enable_fallback:
            raise original_error
        
        # Create minimal valid instance with error information
        fallback_data = {}
        
        for field_name, field in model_type.__fields__.items():
            if field.default is not None:
                fallback_data[field_name] = field.default
            elif field.default_factory is not None:
                fallback_data[field_name] = field.default_factory()
            else:
                # Use type-appropriate empty values
                if field.type_ == str:
                    fallback_data[field_name] = f"Error: {original_error}"
                elif field.type_ == list:
                    fallback_data[field_name] = []
                elif field.type_ == dict:
                    fallback_data[field_name] = {}
                elif field.type_ == int or field.type_ == float:
                    fallback_data[field_name] = 0
                else:
                    fallback_data[field_name] = None
        
        return model_type(**fallback_data)
    
    def parse_output(
        self, 
        raw_output: str, 
        model_type: Type[BaseModel]
    ) -> BaseModel:
        """
        Parse output with comprehensive error handling and fallback.
        """
        try:
            return safe_parse_output(raw_output, model_type)
            
        except OutputParsingError as e:
            # Classify and track error
            error_type = self.classify_error(e.original_error)
            self.error_stats[error_type] += 1
            
            # Log detailed error information
            self._log_parsing_error(e, error_type)
            
            # Create fallback response if enabled
            return self.create_fallback_response(model_type, e)
    
    def _log_parsing_error(self, error: OutputParsingError, error_type: ParserErrorType):
        """Log parsing errors for monitoring and improvement."""
        error_info = {
            'error_type': error_type.value,
            'model_type': error.model_type.__name__,
            'raw_output_preview': error.raw_output[:200] + '...' if len(error.raw_output) > 200 else error.raw_output,
            'original_error': str(error.original_error),
            'timestamp': datetime.now().isoformat()
        }
        
        print(f"Output parsing error: {error_info}")
        # In production, send to your logging/monitoring system
        # logger.warning("Output parsing failed", extra=error_info)

# Usage in production
production_parser = OutputParser(enable_fallback=True)

try:
    result = await Runner.run(research_agent, "Analyze quantum computing trends")
    structured_output = production_parser.parse_output(
        result.final_output, 
        ResearchSummary
    )
    
    # Check if we got a fallback response
    if "Error:" in str(structured_output.topic):
        print("Warning: Using fallback response due to parsing errors")
        # Implement additional fallback logic
        
except Exception as e:
    print(f"Critical error: {e}")
    # Implement critical error handling
```

## 📊 Monitoring and Improvement

### Error Analytics and Model Improvement
```python
class ParserAnalytics:
    """Track parsing performance and identify improvement opportunities."""
    
    def __init__(self):
        self.parsing_attempts = 0
        self.successful_parses = 0
        self.error_breakdown = {}
        self.common_failures = []
    
    def record_attempt(self, success: bool, error_type: Optional[ParserErrorType] = None):
        """Record parsing attempt results."""
        self.parsing_attempts += 1
        
        if success:
            self.successful_parses += 1
        else:
            self.error_breakdown[error_type] = self.error_breakdown.get(error_type, 0) + 1
    
    def get_success_rate(self) -> float:
        """Calculate parsing success rate."""
        if self.parsing_attempts == 0:
            return 0.0
        return self.successful_parses / self.parsing_attempts
    
    def generate_report(self) -> dict:
        """Generate analytics report for model improvement."""
        return {
            'success_rate': self.get_success_rate(),
            'total_attempts': self.parsing_attempts,
            'error_breakdown': self.error_breakdown,
            'recommendations': self._generate_recommendations()
        }
    
    def _generate_recommendations(self) -> list:
        """Generate improvement recommendations based on error patterns."""
        recommendations = []
        
        if self.error_breakdown.get(ParserErrorType.INVALID_JSON, 0) > 10:
            recommendations.append("Consider adding JSON formatting instructions to agent prompts")
        
        if self.error_breakdown.get(ParserErrorType.MISSING_FIELDS, 0) > 5:
            recommendations.append("Review required fields and consider making optional fields optional")
        
        return recommendations

# Integrate analytics with parser
analytics = ParserAnalytics()
parser = OutputParser()
parser.analytics = analytics  # Add analytics capability to parser
```
