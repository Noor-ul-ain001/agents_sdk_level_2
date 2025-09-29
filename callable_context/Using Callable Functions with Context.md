# Dynamic Instructions with Context in OpenAI Agents SDK

Dynamic instructions allow you to programmatically generate agent instructions at runtime using context, enabling highly adaptive and personalized agent behavior.

## 🎯 Core Concept

Instead of providing static instructions when creating an agent, you can pass a **callable function** that generates instructions dynamically based on the current execution context. This function receives the agent and context as parameters and returns the final instruction string.

## 🛠️ Basic Implementation

### Function Signature
```python
def dynamic_instruction_function(agent: Agent, context: Any) -> str:
    # Generate instructions based on context
    return "Generated instructions string"
```

### Simple Example
```python
from agents import Agent, Runner

def time_based_instructions(agent, context):
    import datetime
    hour = datetime.datetime.now().hour
    
    if hour < 12:
        return "You are a morning assistant. Be energetic and concise."
    elif hour < 18:
        return "You are an afternoon assistant. Be detailed and helpful."
    else:
        return "You are an evening assistant. Be relaxed and thorough."

# Create agent with dynamic instructions
agent = Agent(
    name="Context-Aware Assistant",
    instructions=time_based_instructions  # Pass the function, don't call it
)

result = Runner.run_sync(agent, "What's the weather like?")
```

## 🔧 Advanced Context Usage

### Custom Context Objects
```python
from typing import Any
from agents import Agent, Runner, RunContextWrapper

class UserSession:
    def __init__(self, user_id: str, preferences: dict, conversation_history: list):
        self.user_id = user_id
        self.preferences = preferences
        self.history = conversation_history

def personalized_instructions(agent, context: RunContextWrapper[UserSession]):
    user_data = context.context  # Access your custom context
    
    instructions = f"""
    You are assisting user {user_data.user_id}.
    
    User Preferences:
    - Language: {user_data.preferences.get('language', 'English')}
    - Tone: {user_data.preferences.get('tone', 'professional')}
    - Expertise: {user_data.preferences.get('expertise', 'general')}
    
    Recent conversation history: {len(user_data.history)} messages
    
    Tailor your responses according to these preferences.
    """
    
    return instructions

# Create context with user data
user_context = UserSession(
    user_id="user_123",
    preferences={"language": "English", "tone": "friendly", "expertise": "technical"},
    conversation_history=["Hello", "Hi there!"]
)

# Create agent
agent = Agent(
    name="Personalized Assistant",
    instructions=personalized_instructions
)

# Run with context
result = Runner.run_sync(agent, "Help me with my project", context=user_context)
```

### Real-time Data Integration
```python
import requests
from agents import Agent, Runner

def weather_aware_instructions(agent, context):
    # Fetch real-time data
    try:
        response = requests.get("https://api.weather.com/current", timeout=5)
        weather_data = response.json()
        temperature = weather_data.get('temperature', 'unknown')
        conditions = weather_data.get('conditions', 'unknown')
    except:
        temperature = "unknown"
        conditions = "unknown"
    
    return f"""
    You are a helpful assistant. 
    
    Current weather conditions: {conditions}, Temperature: {temperature}
    
    Consider the weather when giving advice about outdoor activities or travel.
    Be mindful that people might be affected by current conditions.
    """

agent = Agent(
    name="Weather-Aware Assistant",
    instructions=weather_aware_instructions
)
```

## 🏗️ Multi-Agent Systems with Dynamic Handoffs

### Context-Powered Handoff Instructions
```python
from agents import Agent, Runner

class SupportContext:
    def __init__(self, issue_type: str, urgency: str, customer_tier: str):
        self.issue_type = issue_type
        self.urgency = urgency
        self.customer_tier = customer_tier

def technical_support_instructions(agent, context: RunContextWrapper[SupportContext]):
    ctx = context.context
    
    priority_map = {"high": "immediately", "medium": "promptly", "low": "when convenient"}
    
    return f"""
    You are a {ctx.issue_type} technical support specialist.
    
    Customer Tier: {ctx.customer_tier}
    Issue Urgency: {ctx.urgency} - respond {priority_map.get(ctx.urgency, 'promptly')}
    
    Provide detailed, technical assistance for {ctx.issue_type} issues.
    { "Offer premium support options." if ctx.customer_tier == "premium" else "Focus on essential solutions." }
    """

def billing_support_instructions(agent, context: RunContextWrapper[SupportContext]):
    ctx = context.context
    
    return f"""
    You are a billing specialist.
    
    Customer Tier: {ctx.customer_tier}
    Issue Type: {ctx.issue_type}
    
    Handle billing inquiries with care and accuracy.
    { "Offer personalized payment plans." if ctx.customer_tier == "premium" else "Provide standard billing information." }
    """

def triage_instructions(agent, context):
    return """
    You are a support triage agent. Determine the type of support needed.
    
    Available handoffs:
    - Technical support: For software, hardware, or technical issues
    - Billing support: For payments, invoices, or account billing questions
    
    Analyze the user's issue and hand off to the appropriate specialist.
    """

# Create specialized agents with dynamic instructions
technical_agent = Agent(
    name="Technical Support",
    instructions=technical_support_instructions
)

billing_agent = Agent(
    name="Billing Support",
    instructions=billing_support_instructions
)

# Triage agent that hands off to specialists
triage_agent = Agent(
    name="Support Triage",
    instructions=triage_instructions,
    handoffs=[technical_agent, billing_agent]
)

# Usage with context
support_context = SupportContext(
    issue_type="software",
    urgency="high", 
    customer_tier="premium"
)

result = Runner.run_sync(
    triage_agent, 
    "My application keeps crashing and I need this fixed urgently!",
    context=support_context
)
```

## ⚡ Advanced Patterns

### Async Dynamic Instructions
```python
import asyncio
from agents import Agent, Runner

async def async_dynamic_instructions(agent, context):
    # Simulate async data fetching
    user_data = await fetch_user_data_async(context.user_id)
    system_status = await check_system_status_async()
    
    return f"""
    User: {user_data['name']}
    System Status: {system_status}
    
    Adjust your responses based on the current system status and user profile.
    """

async def main():
    agent = Agent(
        name="Async Context Agent",
        instructions=async_dynamic_instructions
    )
    
    result = await Runner.run(agent, "Hello", context=context_data)
    return result

# asyncio.run(main())
```

### Instruction Templates with Context Variables
```python
from string import Template

def template_based_instructions(agent, context):
    template = Template("""
    You are a $role assistant for $company.
    
    Current Focus: $focus_area
    Communication Style: $style
    
    $special_instructions
    """)
    
    return template.safe_substitute({
        'role': context.get('role', 'general'),
        'company': context.get('company', 'our company'),
        'focus_area': context.get('focus_area', 'general assistance'),
        'style': context.get('style', 'professional'),
        'special_instructions': context.get('special_instructions', '')
    })

# Usage
context = {
    'role': 'technical support',
    'company': 'TechCorp',
    'focus_area': 'software troubleshooting',
    'style': 'clear and patient',
    'special_instructions': 'Focus on step-by-step guidance for beginners.'
}

agent = Agent(
    name="Template-Based Agent",
    instructions=template_based_instructions
)
```

## 🚀 Best Practices

### 1. Error Handling in Dynamic Functions
```python
def robust_dynamic_instructions(agent, context):
    try:
        # Potentially failing operations
        dynamic_data = fetch_dynamic_data()
        return f"Instructions with {dynamic_data}"
    except Exception as e:
        # Fallback instructions
        return "Standard assistance instructions. Be helpful and professional."
```

### 2. Context Validation
```python
def validated_instructions(agent, context):
    if not hasattr(context, 'required_field'):
        return "Default instructions: Provide general assistance."
    
    # Use validated context
    return f"Custom instructions using {context.required_field}"
```

### 3. Performance Optimization
```python
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_dynamic_instructions(agent, context_key: str):
    # Cache based on context signature to avoid recomputation
    return f"Instructions for context {context_key}"

def wrapper_instructions(agent, context):
    context_key = f"{context.user_id}_{context.session_id}"
    return cached_dynamic_instructions(agent, context_key)
```

## 💡 Use Cases

- **Personalization**: Adapt instructions based on user preferences, history, or profile
- **A/B Testing**: Test different instruction sets dynamically
- **Feature Flags**: Enable/disable capabilities based on context
- **Multi-tenant Systems**: Different instructions for different organizations
- **Real-time Adaptation**: Adjust behavior based on external conditions (time, weather, system status)
- **Progressive Disclosure**: Reveal capabilities based on user expertise or permissions

Dynamic instructions with context provide powerful flexibility for building adaptive, context-aware AI applications that can respond intelligently to changing conditions and requirements.