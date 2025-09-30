# Dynamic Instructions with Context in OpenAI Agents SDK

Dynamic instructions allow you to programmatically generate agent instructions at runtime using context, enabling highly adaptive and personalized agent behavior.

## 🎯 Core Concept

Instead of providing static instructions when creating an agent, you can pass a **callable function** that generates instructions dynamically based on the current execution context. This function receives the agent and context as parameters and returns the final instruction string.

## 💡 Use Cases

- **Personalization**: Adapt instructions based on user preferences, history, or profile
- **A/B Testing**: Test different instruction sets dynamically
- **Feature Flags**: Enable/disable capabilities based on context
- **Multi-tenant Systems**: Different instructions for different organizations
- **Real-time Adaptation**: Adjust behavior based on external conditions (time, weather, system status)
- **Progressive Disclosure**: Reveal capabilities based on user expertise or permissions


Dynamic instructions with context provide powerful flexibility for building adaptive, context-aware AI applications that can respond intelligently to changing conditions and requirements.
