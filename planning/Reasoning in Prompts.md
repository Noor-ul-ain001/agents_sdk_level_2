
# Reasoning in Prompts: Advanced Techniques for AI Agents

This guide covers sophisticated reasoning techniques and prompt engineering strategies to enhance AI agent capabilities in the OpenAI Agents SDK.

## 🎯 Core Reasoning Patterns

### Structured Reasoning Approaches
| Pattern | Purpose | Key Characteristics |
| :--- | :--- | :--- |
| **Chain of Thought (CoT)** | Step-by-step reasoning for complex problems | Explicit intermediate steps, logical progression |
| **Tree of Thought (ToT)** | Multiple reasoning paths with evaluation | Parallel reasoning branches, quality assessment |
| **Graph of Thought (GoT)** | Complex reasoning with interconnected steps | Networked reasoning, cyclical dependencies |
| **Self-Critique** | Error detection and correction | Reflection, verification, iterative improvement |

## 🛠️ Implementation Patterns

### 1. Chain of Thought Reasoning

```python
from agents import Agent
from pydantic import BaseModel
from typing import List, Optional

class ReasoningStep(BaseModel):
    step_number: int
    thought: str
    evidence: Optional[str] = None
    confidence: float = 0.0

class ChainOfThoughtResponse(BaseModel):
    question: str
    reasoning_steps: List[ReasoningStep]
    final_answer: str
    confidence: float
    assumptions: List[str]

def create_cot_agent() -> Agent:
    """Create an agent with explicit chain-of-thought reasoning."""
    
    cot_instructions = """
    You are an analytical reasoning agent. For every problem, you MUST:
    
    1. **Understand the Problem**
       - Restate the problem in your own words
       - Identify key components and constraints
       - Note any implicit assumptions
    
    2. **Step-by-Step Reasoning**
       - Break down the problem into logical steps
       - For each step:
         * Clearly state what you're calculating or reasoning about
         * Show your work or thought process
         * Note any evidence or data used
         * Assign a confidence level (0.0 to 1.0)
    
    3. **Synthesize Solution**
       - Combine step results into final answer
       - Verify the solution makes sense
       - Identify potential limitations
    
    Example reasoning format:
    Step 1: [Thought process with calculation]
    Step 2: [Next logical step]
    ...
    Final Answer: [Synthesized solution]
    
    Always structure your response using the provided output format.
    """
    
    return Agent(
        name="Chain-of-Thought Reasoner",
        instructions=cot_instructions,
        output_type=ChainOfThoughtResponse
    )
```

### 2. Multi-Path Reasoning with Tree of Thought

```python
from enum import Enum
from typing import Dict, Any

class ReasoningPath(BaseModel):
    path_id: str
    approach: str
    steps: List[str]
    strengths: List[str]
    weaknesses: List[str]
    confidence: float

class MultiPathResponse(BaseModel):
    problem: str
    considered_paths: List[ReasoningPath]
    selected_path: str
    reasoning: str
    final_answer: str

class ToTAgent:
    def __init__(self):
        self.thinker = Agent(
            name="Tree-of-Thought Reasoner",
            instructions="""
            You explore multiple reasoning paths before settling on a solution.
            
            Process:
            1. **Generate Alternatives**: Propose 3-5 different approaches
            2. **Evaluate Each Path**:
               - Strengths of this approach
               - Weaknesses and risks
               - Required assumptions
               - Confidence level
            
            3. **Compare and Select**:
               - Which path seems most promising?
               - Why eliminate other paths?
               - Any hybrid approach possible?
            
            4. **Execute Selected Path**:
               - Detailed reasoning using chosen method
               - Final verified answer
            
            Always consider:
            - Simplicity vs comprehensiveness trade-offs
            - Data availability and quality
            - Time and resource constraints
            """,
            output_type=MultiPathResponse
        )
    
    async def solve_complex_problem(self, problem: str) -> MultiPathResponse:
        """Solve complex problems using multi-path reasoning."""
        prompt = f"""
        Analyze this complex problem using tree-of-thought reasoning:
        
        PROBLEM: {problem}
        
        Consider:
        - Mathematical approaches
        - Heuristic methods  
        - Analogical reasoning
        - First principles thinking
        - Divide and conquer strategies
        """
        
        result = await Runner.run(self.thinker, prompt)
        return result.final_output
```

### 3. Self-Correction and Reflection

```python
class ReflectionStep(BaseModel):
    initial_solution: str
    identified_issues: List[str]
    corrections: List[str]
    improved_solution: str
    learning_insights: List[str]

class SelfCorrectingAgent:
    def __init__(self):
        self.primary_solver = Agent(
            name="Primary Problem Solver",
            instructions="Solve problems directly and efficiently."
        )
        
        self.critic = Agent(
            name="Solution Critic",
            instructions="""
            You are a critical reviewer of solutions. Your job is to:
            
            1. **Identify Potential Issues**:
               - Logical fallacies or gaps
               - Mathematical errors
               - Unsupported assumptions
               - Better alternative approaches
            
            2. **Suggest Improvements**:
               - More efficient methods
               - Additional verification steps
               - Clearer explanations
               - Edge case considerations
            
            3. **Provide Constructive Feedback**
            """
        )
        
        self.reflector = Agent(
            name="Learning Reflector",
            instructions="Extract insights and learning from the correction process.",
            output_type=ReflectionStep
        )
    
    async def solve_with_reflection(self, problem: str) -> ReflectionStep:
        """Solve problems with self-correction and reflection."""
        
        # Initial solution
        initial_result = await Runner.run(
            self.primary_solver, 
            f"Solve: {problem}"
        )
        initial_solution = initial_result.final_output
        
        # Critical review
        critique_result = await Runner.run(
            self.critic,
            f"""
            Problem: {problem}
            Proposed Solution: {initial_solution}
            
            Critically review this solution and identify:
            1. Potential errors or issues
            2. Missing considerations  
            3. Better approaches
            4. Verification steps needed
            """
        )
        
        # Generate improved solution
        improved_result = await Runner.run(
            self.primary_solver,
            f"""
            Original Problem: {problem}
            Initial Solution: {initial_solution}
            Critique: {critique_result.final_output}
            
            Generate an improved solution addressing the critique.
            """
        )
        
        # Reflection and learning
        reflection_result = await Runner.run(
            self.reflector,
            f"""
            Problem: {problem}
            Initial: {initial_solution}
            Critique: {critique_result.final_output}  
            Improved: {improved_result.final_output}
            
            Extract learning insights from this process.
            """
        )
        
        return reflection_result.final_output
```

## 🔧 Advanced Reasoning Techniques

### 4. Analogical Reasoning

```python
class Analogy(BaseModel):
    source_domain: str
    target_domain: str
    mapping: Dict[str, str]
    strengths: List[str]
    limitations: List[str]
    insights: List[str]

class AnalogicalReasoner:
    def __init__(self):
        self.agent = Agent(
            name="Analogical Reasoner",
            instructions="""
            You solve problems using analogical reasoning:
            
            1. **Find Analogies**:
               - Identify similar problems in different domains
               - Look for structural similarities
               - Consider known solutions to analogous problems
            
            2. **Map Relationships**:
               - How do elements correspond?
               - What transfers directly?
               - What needs adaptation?
            
            3. **Generate Insights**:
               - What can we learn from the analogy?
               - What limitations does it have?
               - How can it guide our solution?
            
            Examples of powerful analogies:
            - Electrical circuits ↔ water flow
            - Biological evolution ↔ cultural change  
            - Computer memory ↔ human memory
            - Market economics ↔ ecosystem dynamics
            """,
            output_type=Analogy
        )
    
    async def reason_by_analogy(self, problem: str, domain: str) -> Analogy:
        """Apply analogical reasoning from a specific domain."""
        prompt = f"""
        Problem: {problem}
        Source Domain for Analogy: {domain}
        
        Create an analogy to help solve this problem:
        """
        
        result = await Runner.run(self.agent, prompt)
        return result.final_output
```

### 5. First Principles Reasoning

```python
class FirstPrinciplesBreakdown(BaseModel):
    fundamental_truths: List[str]
    assumptions_challenged: List[str]
    reconstruction_steps: List[str]
    novel_insights: List[str]

class FirstPrinciplesReasoner:
    def __init__(self):
        self.agent = Agent(
            name="First Principles Reasoner",
            instructions="""
            You break down problems to fundamental truths and rebuild from there.
            
            Process:
            1. **Identify and Challenge Assumptions**:
               - What are we taking for granted?
               - Why do we believe these things?
               - What evidence supports them?
            
            2. **Establish Fundamental Truths**:
               - What do we know for certain?
               - What are the basic building blocks?
               - What are the immutable laws or constraints?
            
            3. **Reconstruct from Basics**:
               - Build up logically from first principles
               - Create new solutions unconstrained by convention
               - Verify each step logically
            
            4. **Compare with Conventional Wisdom**:
               - How does this approach differ?
               - What advantages does it offer?
               - What limitations does it overcome?
            """,
            output_type=FirstPrinciplesBreakdown
        )
    
    async def first_principles_analysis(self, problem: str) -> FirstPrinciplesBreakdown:
        """Analyze a problem using first principles reasoning."""
        prompt = f"""
        Analyze this problem using first principles reasoning:
        
        PROBLEM: {problem}
        
        Guide your analysis:
        1. What are the common assumptions in this domain that might be wrong?
        2. What are the fundamental physical/mathematical/logical truths?
        3. How would you solve this if starting from scratch?
        4. What novel approaches emerge?
        """
        
        result = await Runner.run(self.agent, prompt)
        return result.final_output
```

## 🚀 Complex Problem-Solving Frameworks

### 6. Multi-Agent Reasoning System

```python
from typing import List, Dict
import asyncio

class ReasoningOrchestrator:
    def __init__(self):
        self.specialists = {
            'analytical': Agent(
                name="Analytical Specialist",
                instructions="Focus on logical, mathematical, and systematic reasoning"
            ),
            'creative': Agent(
                name="Creative Specialist", 
                instructions="Focus on novel, out-of-the-box thinking and analogies"
            ),
            'practical': Agent(
                name="Practical Specialist",
                instructions="Focus on feasibility, resources, and implementation"
            ),
            'critical': Agent(
                name="Critical Specialist",
                instructions="Focus on identifying flaws, risks, and limitations"
            )
        }
        
        self.synthesizer = Agent(
            name="Solution Synthesizer",
            instructions="Integrate multiple perspectives into coherent solutions"
        )
    
    async def orchestrated_reasoning(self, problem: str) -> Dict[str, str]:
        """Solve problems using multiple reasoning specialists."""
        
        # Get perspectives from all specialists
        specialist_tasks = []
        for specialty, agent in self.specialists.items():
            task = Runner.run(
                agent,
                f"""
                Problem: {problem}
                
                Analyze this from your specialized perspective:
                - {specialty.capitalize()} viewpoint
                - Key insights from your domain
                - Recommendations or solutions
                - Important caveats
                """
            )
            specialist_tasks.append((specialty, task))
        
        # Wait for all specialists
        specialist_results = {}
        for specialty, task in specialist_tasks:
            result = await task
            specialist_results[specialty] = result.final_output
        
        # Synthesize perspectives
        synthesis_prompt = f"""
        Problem: {problem}
        
        Specialist Perspectives:
        {chr(10).join(f'{spec}: {persp}' for spec, persp in specialist_results.items())}
        
        Synthesize these perspectives into:
        1. Integrated solution approach
        2. Key insights from each perspective
        3. Balanced recommendation
        4. Implementation considerations
        """
        
        synthesis_result = await Runner.run(self.synthesizer, synthesis_prompt)
        
        return {
            'specialist_views': specialist_results,
            'synthesized_solution': synthesis_result.final_output
        }
```

### 7. Meta-Reasoning and Strategy Selection

```python
class ReasoningStrategy(BaseModel):
    strategy_name: str
    description: str
    when_to_use: List[str]
    when_to_avoid: List[str]
    implementation_steps: List[str]

class MetaReasoner:
    def __init__(self):
        self.strategy_advisor = Agent(
            name="Reasoning Strategy Advisor",
            instructions="""
            You recommend optimal reasoning strategies for different problem types.
            
            Available strategies:
            - Chain of Thought: For logical, sequential problems
            - Tree of Thought: For problems with multiple valid approaches  
            - First Principles: For innovative solutions and assumption-breaking
            - Analogical: For transferring solutions across domains
            - Divide and Conquer: For complex, decomposable problems
            - Working Backwards: For goal-oriented problems
            - Hypothesis Testing: For scientific or investigative problems
            
            For each problem, recommend:
            1. Primary strategy and why
            2. Alternative strategies and when they'd be better
            3. Step-by-step implementation guide
            """,
            output_type=ReasoningStrategy
        )
    
    async def select_reasoning_strategy(self, problem: str, context: Dict = None) -> ReasoningStrategy:
        """Select the optimal reasoning strategy for a problem."""
        prompt = f"""
        Problem: {problem}
        Context: {context or 'No additional context'}
        
        Recommend the best reasoning strategy and explain:
        1. Why this strategy fits the problem
        2. Step-by-step implementation
        3. What to watch out for
        4. Alternative strategies to consider
        """
        
        result = await Runner.run(self.strategy_advisor, prompt)
        return result.final_output
```

## 📊 Evaluation and Improvement

### 8. Reasoning Quality Assessment

```python
class ReasoningQualityMetrics(BaseModel):
    clarity: float  # 0-1 scale
    logical_soundness: float
    completeness: float  
    creativity: float
    practicality: float
    issues_identified: List[str]
    improvement_suggestions: List[str]

class ReasoningEvaluator:
    def __init__(self):
        self.evaluator = Agent(
            name="Reasoning Quality Evaluator",
            instructions="""
            You evaluate the quality of reasoning processes.
            
            Evaluation criteria:
            1. **Clarity**: Are steps well-explained and easy to follow?
            2. **Logical Soundness**: Does reasoning follow logically?
            3. **Completeness**: Are all aspects considered?
            4. **Creativity**: Are novel insights or approaches used?
            5. **Practicality**: Is the solution feasible and actionable?
            
            Provide specific, constructive feedback for improvement.
            """,
            output_type=ReasoningQualityMetrics
        )
    
    async def evaluate_reasoning(self, problem: str, reasoning: str) -> ReasoningQualityMetrics:
        """Evaluate the quality of a reasoning process."""
        prompt = f"""
        Problem: {problem}
        Reasoning Process: {reasoning}
        
        Evaluate this reasoning on:
        - Clarity and explicitness
        - Logical coherence
        - Completeness of analysis
        - Creativity of approach
        - Practical feasibility
        
        Provide specific improvement suggestions.
        """
        
        result = await Runner.run(self.evaluator, prompt)
        return result.final_output
```

## 💡 Best Practices

### 9. Prompt Engineering for Better Reasoning

```python
class ReasoningPromptTemplates:
    """Templates for eliciting better reasoning from agents."""
    
    @staticmethod
    def cot_template(problem: str) -> str:
        return f"""
        Let's think step by step about this problem:
        
        PROBLEM: {problem}
        
        Please structure your reasoning as:
        
        Step 1: Understand and restate the problem
        Step 2: Identify key elements and constraints  
        Step 3: Break down into subproblems
        Step 4: Solve each subproblem
        Step 5: Combine solutions
        Step 6: Verify and reflect
        
        Show your work clearly at each step.
        """
    
    @staticmethod
    def self_correction_template(initial_solution: str) -> str:
        return f"""
        Let's critically examine this solution:
        
        {initial_solution}
        
        Questions to consider:
        1. Are there any logical gaps or jumps?
        2. What assumptions were made? Are they valid?
        3. Could there be alternative approaches?
        4. What edge cases weren't considered?
        5. How could this solution be improved?
        
        Provide a revised solution addressing these points.
        """
    
    @staticmethod
    def multi_perspective_template(problem: str) -> str:
        return f"""
        Analyze this problem from multiple perspectives:
        
        PROBLEM: {problem}
        
        Consider:
        - Mathematical/logical perspective
        - Practical/implementation perspective  
        - Creative/innovative perspective
        - Risk/critical perspective
        
        For each perspective:
        1. What insights does it provide?
        2. What solutions does it suggest?
        3. What limitations does it reveal?
        
        Then synthesize the best overall approach.
        """
```

### 10. Domain-Specific Reasoning Patterns

```python
class DomainSpecificReasoning:
    """Reasoning patterns tailored to specific domains."""
    
    @staticmethod
    def scientific_reasoning(problem: str) -> str:
        return f"""
        Scientific Reasoning Approach:
        
        Problem: {problem}
        
        Follow the scientific method:
        1. Observation: What phenomena are we explaining?
        2. Hypothesis: What's our proposed explanation?
        3. Prediction: What would we expect if hypothesis is true?
        4. Experiment: How could we test this?
        5. Analysis: What do results tell us?
        6. Conclusion: What can we conclude?
        """
    
    @staticmethod
    def business_reasoning(problem: str) -> str:
        return f"""
        Business Decision Framework:
        
        Problem: {problem}
        
        Analyze using:
        - SWOT Analysis (Strengths, Weaknesses, Opportunities, Threats)
        - Cost-Benefit Analysis
        - Risk Assessment
        - Stakeholder Impact
        - Strategic Alignment
        - Implementation Feasibility
        
        Provide reasoned recommendations.
        """
    
    @staticmethod
    def ethical_reasoning(dilemma: str) -> str:
        return f"""
        Ethical Reasoning Process:
        
        Dilemma: {dilemma}
        
        Consider:
        1. Identify all stakeholders and their interests
        2. Apply multiple ethical frameworks:
           - Utilitarian: Greatest good for greatest number
           - Deontological: Following moral rules/duties  
           - Virtue Ethics: Acting with good character
           - Rights-based: Respecting individual rights
        3. Identify conflicts between frameworks
        4. Propose resolution balancing competing values
        """
```
