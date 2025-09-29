# Agent Cloning, Copying, and Replication in OpenAI Agents SDK

This guide covers various techniques for creating agent copies, including cloning, shallow copying, and advanced replication patterns for building scalable multi-agent systems.

## 🎯 Core Concepts

### Copy Types Overview
| Method | Depth | Use Case | Performance Impact |
| :--- | :--- | :--- | :--- |
| **Agent.clone()** | Deep copy with overrides | Creating modified variants of existing agents | Moderate (creates new objects) |
| **copy.copy()** | Shallow copy | Quick duplicates sharing some references | Fast (minimal object creation) |
| **copy.deepcopy()** | Deep copy | Complete independent copies | Slow (full object tree copy) |
| **Template Pattern** | Structural replication | Creating agent factories and prototypes | Fast after initial setup |

## 🛠️ Implementation Patterns

### Basic Agent Cloning with Overrides
```python
from agents import Agent
from copy import copy, deepcopy

# Create base agent
base_agent = Agent(
    name="Base Assistant",
    instructions="You are a helpful assistant.",
    model="gpt-4"
)

# Method 1: Using built-in clone method (recommended)
support_agent = base_agent.clone(
    name="Support Specialist",
    instructions="You are a technical support specialist. Focus on troubleshooting and problem-solving."
)

sales_agent = base_agent.clone(
    name="Sales Agent",
    instructions="You are a sales representative. Focus on product features and benefits."
)

# Method 2: Manual shallow copy
shallow_copy_agent = copy(base_agent)
shallow_copy_agent.name = "Shallow Copy Agent"

# Method 3: Manual deep copy
deep_copy_agent = deepcopy(base_agent)
deep_copy_agent.name = "Deep Copy Agent"
```

### Advanced Clone with Tool Preservation
```python
from agents import Agent, function_tool
from typing import List

@function_tool
def search_knowledge_base(query: str) -> str:
    """Search company knowledge base."""
    return f"Results for: {query}"

@function_tool
def create_support_ticket(issue: str) -> str:
    """Create a new support ticket."""
    return f"Ticket created for: {issue}"

# Base agent with shared tools
base_support_agent = Agent(
    name="Base Support",
    instructions="Help users with their issues.",
    tools=[search_knowledge_base, create_support_ticket],
    model="gpt-4"
)

# Specialized clones with preserved tools
tier1_agent = base_support_agent.clone(
    name="Tier 1 Support",
    instructions="Handle basic customer inquiries and create tickets for complex issues."
)

tier2_agent = base_support_agent.clone(
    name="Tier 2 Support",
    instructions="Handle complex technical issues. Use advanced troubleshooting techniques."
)

engineering_agent = base_support_agent.clone(
    name="Engineering Support",
    instructions="Handle escalated technical issues requiring deep system knowledge.",
    # Add specialized tools for engineering
    tools=base_support_agent.tools + [advanced_diagnostics_tool]
)
```

## 🔧 Advanced Replication Patterns

### Agent Factory with Dynamic Configuration
```python
from agents import Agent
from typing import Dict, Any, Optional
import copy

class AgentFactory:
    """Factory for creating agent variants with consistent configurations."""
    
    def __init__(self, base_agent: Agent):
        self.base_agent = base_agent
        self._cache = {}
    
    def create_variant(
        self,
        name: str,
        instructions: str,
        tools: Optional[list] = None,
        model: Optional[str] = None,
        cache_key: Optional[str] = None
    ) -> Agent:
        """Create an agent variant with optional caching."""
        
        # Use cache if available
        if cache_key and cache_key in self._cache:
            return self._cache[cache_key]
        
        # Create variant using clone
        variant = self.base_agent.clone(
            name=name,
            instructions=instructions
        )
        
        # Apply optional overrides
        if tools is not None:
            variant.tools = tools
        
        if model is not None:
            variant.model = model
        
        # Cache if requested
        if cache_key:
            self._cache[cache_key] = variant
        
        return variant
    
    def create_role_based_agents(self, role_configs: Dict[str, Dict]) -> Dict[str, Agent]:
        """Create multiple agents based on role configurations."""
        agents = {}
        
        for role_name, config in role_configs.items():
            agents[role_name] = self.create_variant(
                name=config['name'],
                instructions=config['instructions'],
                tools=config.get('tools'),
                model=config.get('model'),
                cache_key=role_name
            )
        
        return agents

# Usage example
base_agent = Agent(
    name="Base Corporate Agent",
    instructions="You are a professional corporate assistant.",
    model="gpt-4"
)

factory = AgentFactory(base_agent)

role_configs = {
    "hr": {
        "name": "HR Assistant",
        "instructions": "Handle HR inquiries, policies, and employee questions.",
        "tools": [hr_tools]
    },
    "it": {
        "name": "IT Support",
        "instructions": "Provide technical support for IT issues.",
        "tools": [it_tools],
        "model": "gpt-4"  # Use specific model for IT
    },
    "finance": {
        "name": "Finance Assistant", 
        "instructions": "Help with financial queries and reporting.",
        "tools": [finance_tools]
    }
}

department_agents = factory.create_role_based_agents(role_configs)
```

### Stateful Agent Replication with Context
```python
from agents import Agent, RunContextWrapper
from copy import deepcopy
from typing import Any, List
import threading

class StatefulAgentReplicator:
    """Manages stateful agent replication with thread-safe operations."""
    
    def __init__(self, template_agent: Agent):
        self.template_agent = template_agent
        self._replicas = {}
        self._lock = threading.RLock()
    
    def create_replica(
        self, 
        replica_id: str,
        session_data: Dict[str, Any],
        custom_instructions: Optional[str] = None
    ) -> Agent:
        """Create a stateful agent replica with session context."""
        
        with self._lock:
            if replica_id in self._replicas:
                return self._replicas[replica_id]
            
            # Create replica with session context
            replica = self.template_agent.clone(
                name=f"{self.template_agent.name}-{replica_id}",
                instructions=custom_instructions or self.template_agent.instructions
            )
            
            # Store session context
            replica.session_data = session_data
            self._replicas[replica_id] = replica
            
            return replica
    
    def get_replica(self, replica_id: str) -> Optional[Agent]:
        """Retrieve an existing replica."""
        with self._lock:
            return self._replicas.get(replica_id)
    
    def update_replica_instructions(
        self, 
        replica_id: str, 
        new_instructions: str
    ) -> bool:
        """Update instructions for a specific replica."""
        with self._lock:
            if replica_id in self._replicas:
                self._replicas[replica_id].instructions = new_instructions
                return True
            return False
    
    def list_replicas(self) -> List[str]:
        """List all replica IDs."""
        with self._lock:
            return list(self._replicas.keys())
    
    def cleanup_replica(self, replica_id: str) -> bool:
        """Remove a replica from management."""
        with self._lock:
            if replica_id in self._replicas:
                del self._replicas[replica_id]
                return True
            return False

# Usage example
template_agent = Agent(
    name="User Session Agent",
    instructions="Maintain conversation context and provide personalized assistance.",
    model="gpt-4"
)

replicator = StatefulAgentReplicator(template_agent)

# Create user-specific replicas
user1_agent = replicator.create_replica(
    replica_id="user_123",
    session_data={"user_id": "123", "preferences": {"language": "en"}},
    custom_instructions="This user prefers technical details and code examples."
)

user2_agent = replicator.create_replica(
    replica_id="user_456", 
    session_data={"user_id": "456", "preferences": {"language": "es"}},
    custom_instructions="This user prefers high-level explanations in Spanish."
)
```

## ⚡ Performance-Optimized Patterns

### Flyweight Agent Pattern
```python
from agents import Agent
from typing import Dict, Any
import weakref

class FlyweightAgentFactory:
    """Implements flyweight pattern for memory-efficient agent replication."""
    
    def __init__(self):
        self._shared_agents = {}
        self._agent_cache = weakref.WeakValueDictionary()
    
    def get_shared_agent(self, agent_type: str, config: Dict[str, Any]) -> Agent:
        """Get or create shared agent instance."""
        
        cache_key = f"{agent_type}:{hash(frozenset(config.items()))}"
        
        if cache_key in self._agent_cache:
            return self._agent_cache[cache_key]
        
        # Create new shared agent
        if agent_type == "support":
            agent = self._create_support_agent(config)
        elif agent_type == "sales":
            agent = self._create_sales_agent(config)
        else:
            agent = self._create_generic_agent(config)
        
        self._agent_cache[cache_key] = agent
        return agent
    
    def _create_support_agent(self, config: Dict[str, Any]) -> Agent:
        """Create shared support agent."""
        return Agent(
            name=config.get('name', 'Support Agent'),
            instructions=config.get('instructions', 'Provide customer support.'),
            tools=config.get('tools', []),
            model=config.get('model', 'gpt-4')
        )
    
    def _create_sales_agent(self, config: Dict[str, Any]) -> Agent:
        """Create shared sales agent."""
        return Agent(
            name=config.get('name', 'Sales Agent'),
            instructions=config.get('instructions', 'Assist with sales inquiries.'),
            tools=config.get('tools', []),
            model=config.get('model', 'gpt-4')
        )
    
    def _create_generic_agent(self, config: Dict[str, Any]) -> Agent:
        """Create generic shared agent."""
        return Agent(
            name=config.get('name', 'Generic Agent'),
            instructions=config.get('instructions', 'Provide assistance.'),
            tools=config.get('tools', []),
            model=config.get('model', 'gpt-4')
        )

# Usage
factory = FlyweightAgentFactory()

# These will return the same instance if config is identical
agent1 = factory.get_shared_agent("support", {"name": "Support", "model": "gpt-4"})
agent2 = factory.get_shared_agent("support", {"name": "Support", "model": "gpt-4"})

print(agent1 is agent2)  # True - same object in memory
```

## 🚀 Best Practices

### 1. Memory Management
```python
import gc
from agents import Agent

class AgentPool:
    """Manages agent instances with memory optimization."""
    
    def __init__(self, max_size: int = 100):
        self.max_size = max_size
        self._pool = {}
        self._access_count = {}
    
    def get_agent(self, agent_id: str, creator_func) -> Agent:
        """Get agent from pool or create new one."""
        if agent_id in self._pool:
            self._access_count[agent_id] += 1
            return self._pool[agent_id]
        
        # Create new agent
        agent = creator_func()
        self._pool[agent_id] = agent
        self._access_count[agent_id] = 1
        
        # Cleanup if pool is too large
        if len(self._pool) > self.max_size:
            self._cleanup_least_used()
        
        return agent
    
    def _cleanup_least_used(self):
        """Remove least frequently used agents."""
        if not self._access_count:
            return
        
        # Find least used agent
        min_id = min(self._access_count.items(), key=lambda x: x[1])[0]
        
        # Remove from pool
        if min_id in self._pool:
            del self._pool[min_id]
            del self._access_count[min_id]
        
        # Force garbage collection
        gc.collect()
```

### 2. Configuration-Driven Replication
```python
from dataclasses import dataclass
from typing import List, Optional
import yaml

@dataclass
class AgentConfig:
    name: str
    instructions: str
    model: str = "gpt-4"
    tools: Optional[List[str]] = None

class ConfigDrivenFactory:
    """Creates agents from configuration files."""
    
    def __init__(self, config_path: str):
        self.config_path = config_path
        self._load_configs()
    
    def _load_configs(self):
        """Load agent configurations from YAML."""
        with open(self.config_path, 'r') as f:
            self.configs = yaml.safe_load(f)
    
    def create_agent_from_config(self, config_name: str, **overrides) -> Agent:
        """Create agent from named configuration."""
        if config_name not in self.configs:
            raise ValueError(f"Unknown config: {config_name}")
        
        config_data = self.configs[config_name].copy()
        config_data.update(overrides)
        
        return Agent(
            name=config_data['name'],
            instructions=config_data['instructions'],
            model=config_data.get('model', 'gpt-4'),
            tools=self._resolve_tools(config_data.get('tools', []))
        )
    
    def _resolve_tools(self, tool_names: List[str]) -> List:
        """Resolve tool names to actual tool objects."""
        # Implementation depends on your tool registry
        return [self._tool_registry[name] for name in tool_names]

# YAML configuration example
"""
support_agent:
  name: "Customer Support"
  instructions: "Help customers with their issues"
  model: "gpt-4"
  tools: ["knowledge_base", "ticket_system"]

sales_agent:
  name: "Sales Representative" 
  instructions: "Assist with product inquiries and sales"
  model: "gpt-4"
  tools: ["product_catalog", "crm_system"]
"""
```

