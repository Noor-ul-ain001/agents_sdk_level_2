# Planning Reminders System for AI Agents

A comprehensive guide to implementing planning and reminder systems within AI agent workflows using the OpenAI Agents SDK.

## 🎯 Overview

Planning reminders help agents maintain context, track goals, and execute multi-step processes reliably. This system enables agents to plan, remember, and follow through on complex tasks.

## 🏗️ Architecture Components

### Core Planning Elements
| Component | Purpose | Implementation |
| :--- | :--- | :--- |
| **Task Planner** | Breaks down high-level goals into actionable steps | Specialized planning agent |
| **Reminder Store** | Persistent storage for tasks and deadlines | Database, in-memory cache, or file system |
| **Scheduler** | Executes tasks at appropriate times | Background worker or timed triggers |
| **Progress Tracker** | Monitors task completion and updates status | Session storage or external state |

## 🛠️ Implementation Patterns

### 1. Basic Planning Agent

```python
from agents import Agent, Runner
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime, timedelta
import asyncio

class TaskStep(BaseModel):
    description: str
    estimated_duration: int  # in minutes
    dependencies: List[str]  # step IDs
    status: str = "pending"  # pending, in_progress, completed

class ProjectPlan(BaseModel):
    goal: str
    steps: List[TaskStep]
    deadline: Optional[datetime]
    priority: str = "medium"

class PlanningAgent:
    def __init__(self):
        self.planner = Agent(
            name="Project Planner",
            instructions="""Break down complex goals into actionable steps.
            Consider dependencies, estimated time, and priority.
            Create realistic timelines and identify critical paths.""",
            output_type=ProjectPlan
        )
    
    async def create_plan(self, goal: str, context: dict = None) -> ProjectPlan:
        prompt = f"""
        Create a detailed project plan for: {goal}
        
        Consider:
        - Break down into 3-8 actionable steps
        - Estimate realistic time for each step (in minutes)
        - Identify dependencies between steps
        - Suggest overall priority
        
        Context: {context}
        """
        
        result = await Runner.run(self.planner, prompt)
        return result.final_output
```

### 2. Reminder System with Persistent Storage

```python
import sqlite3
from dataclasses import dataclass
from typing import List, Optional
import json

@dataclass
class Reminder:
    id: str
    agent_id: str
    message: str
    trigger_time: datetime
    context: dict
    status: str = "pending"
    created_at: datetime = None
    
    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()

class ReminderManager:
    def __init__(self, db_path: str = "reminders.db"):
        self.db_path = db_path
        self._init_db()
    
    def _init_db(self):
        """Initialize SQLite database for reminders."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS reminders (
                id TEXT PRIMARY KEY,
                agent_id TEXT,
                message TEXT,
                trigger_time TIMESTAMP,
                context TEXT,
                status TEXT,
                created_at TIMESTAMP
            )
        ''')
        conn.commit()
        conn.close()
    
    def add_reminder(
        self,
        agent_id: str,
        message: str,
        trigger_delta: timedelta,
        context: dict = None
    ) -> str:
        """Add a new reminder."""
        reminder_id = f"rem_{datetime.now().timestamp()}"
        trigger_time = datetime.now() + trigger_delta
        
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT INTO reminders 
            (id, agent_id, message, trigger_time, context, status, created_at)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (
            reminder_id,
            agent_id,
            message,
            trigger_time.isoformat(),
            json.dumps(context or {}),
            "pending",
            datetime.now().isoformat()
        ))
        
        conn.commit()
        conn.close()
        return reminder_id
    
    def get_due_reminders(self) -> List[Reminder]:
        """Get all reminders that are due."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute('''
            SELECT * FROM reminders 
            WHERE trigger_time <= ? AND status = 'pending'
            ORDER BY trigger_time ASC
        ''', (datetime.now().isoformat(),))
        
        reminders = []
        for row in cursor.fetchall():
            reminders.append(Reminder(
                id=row[0],
                agent_id=row[1],
                message=row[2],
                trigger_time=datetime.fromisoformat(row[3]),
                context=json.loads(row[4]),
                status=row[5],
                created_at=datetime.fromisoformat(row[6])
            ))
        
        conn.close()
        return reminders
    
    def mark_completed(self, reminder_id: str):
        """Mark a reminder as completed."""
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        cursor.execute(
            "UPDATE reminders SET status = 'completed' WHERE id = ?",
            (reminder_id,)
        )
        conn.commit()
        conn.close()
```

### 3. Goal-Oriented Agent with Planning

```python
from agents import Agent, Runner, function_tool
from typing import Dict, List

class GoalOrientedAgent:
    def __init__(self):
        self.reminder_manager = ReminderManager()
        self.active_goals: Dict[str, ProjectPlan] = {}
        
        self.goal_agent = Agent(
            name="Goal Manager",
            instructions="""Manage long-term goals and break them into plans.
            Track progress and set appropriate reminders.""",
            tools=[self.add_reminder_tool, self.update_plan_tool]
        )
    
    @function_tool
    def add_reminder_tool(self, message: str, minutes_from_now: int) -> str:
        """Add a reminder for the future."""
        reminder_id = self.reminder_manager.add_reminder(
            agent_id="goal_agent",
            message=message,
            trigger_delta=timedelta(minutes=minutes_from_now)
        )
        return f"Reminder set: {message} (ID: {reminder_id})"
    
    @function_tool
    def update_plan_tool(self, goal: str, completed_step: str) -> str:
        """Update plan progress when a step is completed."""
        if goal in self.active_goals:
            plan = self.active_goals[goal]
            for step in plan.steps:
                if step.description == completed_step:
                    step.status = "completed"
                    return f"Step completed: {completed_step}"
        return "Step not found in current plan"
    
    async def set_goal(self, goal: str, deadline: datetime = None):
        """Set a new goal and create execution plan."""
        planner = PlanningAgent()
        plan = await planner.create_plan(goal)
        
        if deadline:
            plan.deadline = deadline
        
        self.active_goals[goal] = plan
        
        # Set initial reminders
        for step in plan.steps:
            self.reminder_manager.add_reminder(
                agent_id="goal_agent",
                message=f"Check progress on: {step.description}",
                trigger_delta=timedelta(hours=2)
            )
        
        return plan
    
    async def check_reminders(self):
        """Check and process due reminders."""
        due_reminders = self.reminder_manager.get_due_reminders()
        
        for reminder in due_reminders:
            # Process the reminder
            await self.process_reminder(reminder)
            # Mark as completed
            self.reminder_manager.mark_completed(reminder.id)
    
    async def process_reminder(self, reminder: Reminder):
        """Process a single reminder."""
        print(f"🔔 REMINDER: {reminder.message}")
        
        # Could trigger specific agent actions based on reminder content
        if "check progress" in reminder.message.lower():
            await self.review_progress()
    
    async def review_progress(self):
        """Review progress on all active goals."""
        for goal, plan in self.active_goals.items():
            completed = sum(1 for step in plan.steps if step.status == "completed")
            total = len(plan.steps)
            progress = (completed / total) * 100
            
            print(f"📊 Goal: {goal} - {progress:.1f}% complete")
            
            if progress < 50:
                # Set follow-up reminder for struggling goals
                self.reminder_manager.add_reminder(
                    agent_id="goal_agent",
                    message=f"Follow up on slow progress: {goal}",
                    trigger_delta=timedelta(hours=4)
                )
```

## 🔄 Advanced Planning Patterns

### 4. Multi-Agent Planning System

```python
from enum import Enum
import asyncio

class Priority(Enum):
    LOW = "low"
    MEDIUM = "medium" 
    HIGH = "high"
    CRITICAL = "critical"

class MultiAgentPlanner:
    def __init__(self):
        self.coordinator = Agent(
            name="Planning Coordinator",
            instructions="Coordinate between specialized planning agents"
        )
        
        self.technical_planner = Agent(
            name="Technical Planner",
            instructions="Plan technical implementation details"
        )
        
        self.timeline_planner = Agent(
            name="Timeline Planner", 
            instructions="Create realistic timelines and schedules"
        )
        
        self.reminder_manager = ReminderManager()
    
    async def create_comprehensive_plan(self, project_brief: str) -> dict:
        """Create a comprehensive plan using multiple specialized agents."""
        
        # Parallel planning with different specialists
        technical_task = asyncio.create_task(
            self._create_technical_plan(project_brief)
        )
        timeline_task = asyncio.create_task(
            self._create_timeline(project_brief)
        )
        
        technical_plan, timeline = await asyncio.gather(
            technical_task, timeline_task
        )
        
        # Integrate plans
        integrated_plan = await self._integrate_plans(
            technical_plan, timeline, project_brief
        )
        
        # Set up monitoring reminders
        await self._setup_plan_monitoring(integrated_plan)
        
        return integrated_plan
    
    async def _create_technical_plan(self, brief: str):
        result = await Runner.run(
            self.technical_planner,
            f"Create technical implementation plan for: {brief}"
        )
        return result.final_output
    
    async def _create_timeline(self, brief: str):
        result = await Runner.run(
            self.timeline_planner,
            f"Create timeline estimate for: {brief}"
        )
        return result.final_output
    
    async def _integrate_plans(self, technical_plan, timeline, brief):
        result = await Runner.run(
            self.coordinator,
            f"Integrate technical plan and timeline for: {brief}\n\n"
            f"Technical: {technical_plan}\nTimeline: {timeline}"
        )
        return result.final_output
    
    async def _setup_plan_monitoring(self, plan: dict):
        """Set up reminders to monitor plan execution."""
        # Set milestone reminders
        if 'milestones' in plan:
            for milestone in plan['milestones']:
                self.reminder_manager.add_reminder(
                    agent_id="coordinator",
                    message=f"Check milestone: {milestone['name']}",
                    trigger_delta=timedelta(days=1)  # Check daily
                )
```

### 5. Adaptive Planning with Context Awareness

```python
class AdaptivePlanner:
    def __init__(self):
        self.context_history = []
        self.adaptation_rules = {
            "behind_schedule": self._handle_schedule_slip,
            "blocked": self._handle_blockers,
            "ahead_of_schedule": self._handle_ahead_schedule
        }
    
    async def adapt_plan(self, current_plan: ProjectPlan, context: dict) -> ProjectPlan:
        """Adapt plan based on current context and progress."""
        
        # Analyze situation
        situation = await self._analyze_situation(current_plan, context)
        
        # Apply adaptation rules
        if situation in self.adaptation_rules:
            adapted_plan = await self.adaptation_rules[situation](
                current_plan, context
            )
            return adapted_plan
        
        return current_plan
    
    async def _analyze_situation(self, plan: ProjectPlan, context: dict) -> str:
        """Analyze current situation to determine needed adaptations."""
        completed_steps = sum(1 for step in plan.steps if step.status == "completed")
        total_steps = len(plan.steps)
        
        if context.get('blockers'):
            return "blocked"
        
        progress_ratio = completed_steps / total_steps if total_steps > 0 else 0
        expected_progress = self._calculate_expected_progress(plan)
        
        if progress_ratio < expected_progress - 0.2:  # 20% behind
            return "behind_schedule"
        elif progress_ratio > expected_progress + 0.3:  # 30% ahead
            return "ahead_of_schedule"
        
        return "on_track"
    
    async def _handle_schedule_slip(self, plan: ProjectPlan, context: dict) -> ProjectPlan:
        """Handle plans that are behind schedule."""
        # Prioritize critical path
        for step in plan.steps:
            if len(step.dependencies) > 2:  # Many dependencies = likely critical
                step.priority = "high"
        
        # Add recovery reminder
        self.reminder_manager.add_reminder(
            agent_id="adaptive_planner",
            message="Review recovery plan for delayed project",
            trigger_delta=timedelta(hours=12)
        )
        
        return plan
    
    async def _handle_blockers(self, plan: ProjectPlan, context: dict) -> ProjectPlan:
        """Handle blocked plans."""
        blockers = context.get('blockers', [])
        
        # Reprioritize to work on unblocked tasks
        for step in plan.steps:
            if any(blocker in step.description for blocker in blockers):
                step.status = "blocked"
            else:
                step.priority = "high"  # Focus on unblocked work
        
        return plan
```

## ⚡ Real-time Monitoring System

### 6. Background Reminder Processor

```python
import threading
import time

class BackgroundReminderService:
    def __init__(self, planning_system: GoalOrientedAgent):
        self.planning_system = planning_system
        self.is_running = False
        self.thread = None
    
    def start(self):
        """Start the background reminder service."""
        self.is_running = True
        self.thread = threading.Thread(target=self._run, daemon=True)
        self.thread.start()
        print("🔔 Reminder service started")
    
    def stop(self):
        """Stop the background reminder service."""
        self.is_running = False
        if self.thread:
            self.thread.join()
        print("🔔 Reminder service stopped")
    
    def _run(self):
        """Main loop for processing reminders."""
        while self.is_running:
            try:
                # Process due reminders
                asyncio.run(self.planning_system.check_reminders())
                
                # Wait before next check
                time.sleep(60)  # Check every minute
                
            except Exception as e:
                print(f"Error in reminder service: {e}")
                time.sleep(5)  # Brief pause on error

# Usage example
async def main():
    # Create planning system
    planning_system = GoalOrientedAgent()
    
    # Set a goal with plan
    plan = await planning_system.set_goal(
        "Build customer feedback analysis dashboard",
        deadline=datetime.now() + timedelta(days=7)
    )
    
    # Start background reminder service
    reminder_service = BackgroundReminderService(planning_system)
    reminder_service.start()
    
    try:
        # Keep main thread alive
        while True:
            await asyncio.sleep(1)
    except KeyboardInterrupt:
        reminder_service.stop()

# Run the system
# asyncio.run(main())
```

## 🚀 Best Practices

### 1. Reminder Granularity
```python
# ✅ Good: Specific, actionable reminders
specific_reminder = "Review PR #123 for authentication module"

# ❌ Avoid: Vague reminders  
vague_reminder = "Check project progress"
```

### 2. Context Preservation
```python
def create_context_aware_reminder(task: str, relevant_data: dict):
    """Create reminders with sufficient context."""
    return {
        "message": task,
        "context": {
            "project": relevant_data.get("project"),
            "last_state": relevant_data.get("state"),
            "blockers": relevant_data.get("blockers", []),
            "resources": relevant_data.get("resources", [])
        }
    }
```

### 3. Failure Recovery
```python
async def robust_reminder_processing(reminder: Reminder, max_retries: int = 3):
    """Process reminders with retry logic."""
    for attempt in range(max_retries):
        try:
            await process_reminder(reminder)
            break
        except Exception as e:
            if attempt == max_retries - 1:
                logger.error(f"Failed to process reminder {reminder.id}: {e}")
                # Escalate or create follow-up reminder
                create_followup_reminder(reminder, f"Processing failed: {e}")
```

