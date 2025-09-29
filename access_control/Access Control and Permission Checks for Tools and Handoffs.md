# Access Control for AI Agents - Simple Guide

## What is Access Control?
**Access control** decides which agents can use which tools and see which data. It's like giving different keys to different people in a company.

## Basic Permission Types

### 1. **Tool Permissions**
```python
# Example: Who can use what tools
permissions = {
    "research_agent": ["web_search", "read_files"],
    "finance_agent": ["database_query", "calculator"],
    "admin_agent": ["all_tools"]  # Special access
}
```

### 2. **Data Access Levels**
```
Public Data → Internal Data → Confidential Data → Secret Data
    ↓             ↓               ↓               ↓
 All Agents   Most Agents    Few Agents      One Agent
```

## Simple Implementation

### Basic Permission Check
```python
def can_use_tool(agent_name, tool_name):
    # Define who can use what
    permissions = {
        "researcher": ["search", "read_docs"],
        "writer": ["write_docs", "edit_files"],
        "analyst": ["read_docs", "analyze_data"]
    }
    
    # Check if agent has permission
    if tool_name in permissions.get(agent_name, []):
        return True
    else:
        return False

# Usage
if can_use_tool("researcher", "search"):
    result = do_search()
else:
    result = "Sorry, you can't use this tool"
```

### Data Handoff Control
```python
def can_receive_data(sender, receiver, data_type):
    # Define allowed data sharing
    sharing_rules = {
        "researcher": ["analyst", "writer"],  # Who researcher can talk to
        "analyst": ["writer"],
        "writer": []  # Writer doesn't send data to others
    }
    
    return receiver in sharing_rules.get(sender, [])
```

## Common Patterns

### 1. **Role-Based Access**
```python
user_roles = {
    "alice": "admin",      # Can do everything
    "bob": "researcher",   # Can research but not edit
    "charlie": "viewer"    # Can only read
}
```

### 2. **Level-Based Access**
```
Level 1: Read public data
Level 2: Read internal data  
Level 3: Read confidential data
Level 4: Read secret data + Edit data
```

## Simple Examples

### Example 1: Research Team
```
Researcher Agent → Can: search web, read documents
                  Cannot: delete files, access finances

Analyst Agent → Can: read research, analyze data
               Cannot: edit source documents

Writer Agent → Can: write reports, read analysis
              Cannot: change raw data
```

### Example 2: Customer Service
```
Tier 1 Agent → Can: view basic customer info, create tickets
Tier 2 Agent → Can: access order history, process returns
Manager Agent → Can: everything, including override actions
```

## Key Rules to Remember

1. **Principle of Least Privilege**: Give agents only the access they absolutely need
2. **Separation of Duties**: No single agent should do everything
3. **Audit Trail**: Log who did what and when

## Why It Matters
✅ **Security** - Prevents unauthorized actions  
✅ **Safety** - Avoids costly mistakes  
✅ **Compliance** - Meets legal requirements  
✅ **Reliability** - Makes system more predictable  

## Simple Checklist
- [ ] List all your agents
- [ ] List all available tools
- [ ] Decide who can use what
- [ ] Test permissions work correctly
- [ ] Log access attempts

