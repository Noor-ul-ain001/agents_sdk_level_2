# OpenAI Agents SDK 

## 🎯 Design Philosophy & Core Principles

The OpenAI Agents SDK is built on two fundamental principles:

- **Minimal yet Powerful**: Enough features to be worth using, but few enough primitives to make it quick to learn
- **Flexible by Design**: Works great out of the box, but allows you to customize exactly what happens

### Python-First Approach

The SDK is **Python-first**, using built-in language features to orchestrate and chain agents rather than forcing developers to learn new abstractions. This design makes it powerful for expressing complex relationships with a minimal learning curve.

## 🧱 Core Architectural Components

### Agent

The core building block of the SDK. An Agent is an LLM equipped with instructions, tools, and the ability to hand off tasks.

```python
from agents import Agent

# Basic agent creation
agent = Agent(
    name="Research Assistant",
    instructions="You are a helpful research assistant that finds and summarizes information.",
    model="gpt-4o"
)
```

**Required Properties:**
- `name`: Identifier for the agent
- `instructions`: System prompt that defines the agent's behavior

**Optional Properties:**
- `tools`: List of tools the agent can use
- `handoffs`: List of agents this agent can hand off to
- `model`: Which LLM to use
- `output_type`: For structured outputs

### Tool

How agents extend their capabilities beyond text generation. Any Python function can be easily converted into a tool.

```python
from agents import function_tool
import requests

@function_tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    # Implementation here
    return f"Weather in {city}: Sunny, 72°F"

# Use the tool in an agent
weather_agent = Agent(
    name="Weather Agent",
    instructions="Get weather information for cities.",
    tools=[get_weather]
)
```

The SDK automatically generates and validates tool input/output schemas using Pydantic.

### Runner

Manages the execution of an agent and handles the agent loop.

```python
from agents import Runner

# Async execution
result = await Runner.run(agent, "What's the weather in Tokyo?")

# Sync execution
result = Runner.run_sync(agent, "What's the weather in Tokyo?")

print(result.final_output)
```

### Handoff

A mechanism that allows one agent to delegate a task and full conversation context to another, more specialized agent.

```python
from agents import Agent

# Create specialized agents
spanish_agent = Agent(
    name="Spanish Agent",
    instructions="You only speak Spanish.",
)

english_agent = Agent(
    name="English Agent", 
    instructions="You only speak English.",
)

# Triage agent that can hand off to specialists
triage_agent = Agent(
    name="Triage Agent",
    instructions="Handoff to the appropriate agent based on the language of the request.",
    handoffs=[spanish_agent, english_agent],
)
```

### Guardrail

Functions that run checks in parallel to your agents:

- **Input Guardrails**: Validate user input before the agent processes it
- **Output Guardrails**: Validate the agent's output before it's returned

```python
from agents import Agent, Runner, InputGuardrail, OutputGuardrail

def content_filter(input_text: str) -> None:
    """Input guardrail to filter inappropriate content."""
    if "inappropriate_word" in input_text:
        raise ValueError("Content violates policy")

def fact_check(output_text: str) -> None:
    """Output guardrail to verify facts."""
    if "false_claim" in output_text:
        raise ValueError("Output contains unverified claims")

agent = Agent(
    name="Safe Agent",
    instructions="Provide helpful and accurate information.",
)
```

### Session

Manages conversation history automatically across multiple agent runs.

```python
from agents import Agent, Runner, SQLiteSession

# Using built-in SQLite session
session = SQLiteSession("conversations.db")

agent = Agent(
    name="Session Agent",
    instructions="You remember conversation history.",
)

result = Runner.run_sync(agent, "What did we talk about earlier?", session=session)
```

## 🔄 The Agent Loop

When you execute an agent with `Runner.run()`, it initiates the following loop:

### Step-by-Step Process

1. **LLM Call**: The LLM is called with the agent's instructions, model settings, and message history
2. **Response Processing**: The LLM returns a response, which may include:
   - Tool calls
   - Handoff requests  
   - Final output
3. **Final Output Check**: If the response has a final output, it is returned and the loop ends
4. **Handoff Processing**: If the response has a handoff, the agent is switched and the loop restarts
5. **Tool Execution**: If there are tool calls, they are processed and results are added to history

### Structured vs Unstructured Outputs

```python
from pydantic import BaseModel
from agents import Agent

class Summary(BaseModel):
    key_points: list[str]
    overall_sentiment: str

# Agent with structured output
structured_agent = Agent(
    name="Summary Agent",
    instructions="Summarize text and provide structured output.",
    output_type=Summary
)

# Agent with unstructured output (default)
unstructured_agent = Agent(
    name="Chat Agent", 
    instructions="Chat with the user.",
    # No output_type = plain text
)
```

## 🤝 Multi-Agent System Design Patterns

### 👑 Manager Pattern (Agents as Tools)

A central manager/orchestrator invokes specialized sub-agents as tools and retains control of the conversation.

```python
from agents import Agent, function_tool

@function_tool
def call_research_agent(query: str) -> str:
    """Tool that calls a research agent."""
    research_agent = Agent(
        name="Research Specialist",
        instructions="Conduct detailed research on topics.",
    )
    result = Runner.run_sync(research_agent, query)
    return result.final_output

@function_tool  
def call_analysis_agent(data: str) -> str:
    """Tool that calls an analysis agent."""
    analysis_agent = Agent(
        name="Analysis Specialist",
        instructions="Analyze data and provide insights.",
    )
    result = Runner.run_sync(analysis_agent, data)
    return result.final_output

# Manager agent that uses specialist agents as tools
manager_agent = Agent(
    name="Project Manager",
    instructions="Coordinate research and analysis tasks.",
    tools=[call_research_agent, call_analysis_agent]
)
```

### 🔄 Handoffs Pattern

Peer agents hand off control to specialized agents, which take over the entire conversation.

```python
from agents import Agent, Runner

# Specialist agents
customer_support_agent = Agent(
    name="Customer Support",
    instructions="Handle general customer inquiries and basic support.",
)

technical_support_agent = Agent(
    name="Technical Support", 
    instructions="Handle complex technical issues and bug reports.",
)

billing_agent = Agent(
    name="Billing Department",
    instructions="Handle billing, payments, and account issues.",
)

# Frontline agent that triages and hands off
frontline_agent = Agent(
    name="Frontline Support",
    instructions="""Greet customers and determine their needs.
    
Available handoffs:
- Technical Support: For technical issues, bugs, or complex problems
- Billing Department: For payment issues, refunds, or account billing questions
- Customer Support: For general questions and basic assistance

Hand off to the most appropriate specialist based on the customer's issue.""",
    handoffs=[technical_support_agent, billing_agent, customer_support_agent]
)

# Usage
result = Runner.run_sync(
    frontline_agent, 
    "I'm having trouble with my payment method and also need technical help with the app."
)
```

## ⚙️ Operational Features

### Context

Agents support dependency injection through a generic `context` type that is available to every agent, tool, and handoff during a run.

```python
from typing import Any
from agents import Agent, Runner, function_tool

class ApplicationContext:
    def __init__(self):
        self.database = "database_connection"
        self.api_key = "secret_key"
        self.config = {"setting": "value"}

@function_tool
def database_query(ctx: Any, query: str) -> str:
    """Tool that uses application context."""
    # ctx is automatically injected with ApplicationContext
    db = ctx.database
    # Execute query using db connection
    return "Query results"

context = ApplicationContext()
agent = Agent(
    name="Context-Aware Agent",
    instructions="Use tools that require application context.",
    tools=[database_query]
)

result = Runner.run_sync(agent, "Run a database query", context=context)
```

### Structured Outputs

Produce validated structured data from agent interactions.

```python
from pydantic import BaseModel
from typing import List
from agents import Agent, Runner

class MeetingNotes(BaseModel):
    attendees: List[str]
    action_items: List[str]
    key_decisions: List[str]
    next_meeting: str

structured_agent = Agent(
    name="Meeting Processor",
    instructions="Extract structured information from meeting notes.",
    output_type=MeetingNotes
)

meeting_text = """
Attendees: Alice, Bob, Charlie
We decided to launch the project next month.
Action items: Alice will prepare documentation, Bob will setup servers.
Next meeting: 2024-01-15
"""

result = Runner.run_sync(structured_agent, meeting_text)
meeting_data = result.final_output  # This is a MeetingNotes instance

print(meeting_data.attendees)  # ['Alice', 'Bob', 'Charlie']
print(meeting_data.action_items)  # ['Alice will prepare documentation', 'Bob will setup servers']
```

### Lifecycle Hooks

Observe and intercept the agent lifecycle for logging, monitoring, or pre-processing.

```python
from agents import AgentHooks, RunEvent, ToolCallEvent

class LoggingHooks(AgentHooks):
    async def on_run_start(self, event: RunEvent):
        print(f"🚀 Starting agent run: {event.agent.name}")
        print(f"📝 Input: {event.input}")
    
    async def on_tool_call(self, event: ToolCallEvent):
        print(f"🛠️ Tool called: {event.tool_name}")
        print(f"📦 Input: {event.tool_input}")
    
    async def on_run_end(self, event: RunEvent):
        print(f"✅ Agent run completed: {event.agent.name}")
        print(f"📤 Output: {event.result.final_output}")

agent = Agent(name="Logged Agent", instructions="An agent with lifecycle logging.")
hooks = LoggingHooks()

result = Runner.run_sync(agent, "Hello world", hooks=hooks)
```

### Tracing & Observability

Built-in tracing lets you visualize, debug, and monitor your agent workflows.

```python
from agents import Agent, Runner, trace

# Trace a specific run
with trace("Research Workflow") as trace_id:
    result = Runner.run_sync(
        research_agent, 
        "Research renewable energy trends",
        metadata={"user_id": "123", "workflow": "research"}
    )
    
print(f"Trace ID: {trace_id}")
print(f"View trace at: https://platform.openai.com/traces/{trace_id}")
```

## 🔧 Advanced Configuration

### Tool Use Behavior

Control how tool outputs are handled with the `tool_use_behavior` parameter.

```python
from agents import Agent, ModelSettings

# Default behavior - run LLM again with tool results
default_agent = Agent(
    name="Default Agent",
    instructions="Use tools as needed.",
    model_settings=ModelSettings(
        tool_use_behavior="run_llm_again"  # Default
    )
)

# Stop after first tool and use its output as final result
tool_first_agent = Agent(
    name="Tool-First Agent", 
    instructions="Use a tool and return its result directly.",
    model_settings=ModelSettings(
        tool_use_behavior="stop_on_first_tool"
    )
)
```

### Forcing Tool Use

Require agents to use specific tools.

```python
from agents import Agent, ModelSettings

@function_tool
def required_calculation(x: int, y: int) -> int:
    """A calculation that must be used."""
    return x * y

forced_tool_agent = Agent(
    name="Forced Tool Agent",
    instructions="You MUST use the calculation tool.",
    tools=[required_calculation],
    model_settings=ModelSettings(
        tool_choice="required"  # Force any tool usage
        # tool_choice="required_calculation"  # Force specific tool
    )
)
```

### Provider Agnostic

While optimized for OpenAI, the SDK works with any LLM supporting the Chat Completions format.

```python
from agents import Agent, ModelSettings

# Using Anthropic Claude
claude_agent = Agent(
    name="Claude Agent",
    instructions="You are helpful assistant powered by Claude.",
    model_settings=ModelSettings(
        model="claude-3-sonnet-20240229",
        provider="anthropic"  # Via LiteLLM
    )
)

# Using Google Gemini
gemini_agent = Agent(
    name="Gemini Agent",
    instructions="You are helpful assistant powered by Gemini.",
    model_settings=ModelSettings(
        model="gemini/gemini-pro",
        provider="google"  # Via LiteLLM
    )
)
```

## 🚀 Installation and Setup

```bash
# Install the core SDK
pip install openai-agents

# For additional providers via LiteLLM
pip install 'openai-agents[litellm]'

# For SQLite session support
pip install 'openai-agents[sqlite]'

# For development with all extras
pip install 'openai-agents[all]'
```

### Environment Setup

```bash
# Required: OpenAI API key
export OPENAI_API_KEY="your-api-key-here"

# Optional: For other providers
export ANTHROPIC_API_KEY="your-anthropic-key"
export GOOGLE_API_KEY="your-google-key"
```

## 💡 Example: Complete Multi-Agent System

```python
from agents import Agent, Runner, function_tool
from pydantic import BaseModel
from typing import List

# Data models
class ResearchReport(BaseModel):
    topic: str
    summary: str
    sources: List[str]
    confidence_score: float

# Tools
@function_tool
def web_search(query: str) -> str:
    """Search the web for information."""
    # Implementation using requests/beautifulsoup
    return f"Search results for: {query}"

@function_tool
def analyze_sentiment(text: str) -> str:
    """Analyze sentiment of text."""
    # Implementation
    return "Positive sentiment"

# Specialist agents
research_agent = Agent(
    name="Researcher",
    instructions="Conduct thorough research on topics using web search.",
    tools=[web_search],
    output_type=ResearchReport
)

analysis_agent = Agent(
    name="Analyst",
    instructions="Analyze research findings and provide insights.",
    tools=[analyze_sentiment]
)

# Manager agent
manager_agent = Agent(
    name="Research Manager",
    instructions="Coordinate research and analysis workflows.",
    handoffs=[research_agent, analysis_agent]
)

# Run the system
result = Runner.run_sync(
    manager_agent,
    "Research the impact of AI on climate change and analyze the sentiment of findings."
)

print("Final result:", result.final_output)
```

