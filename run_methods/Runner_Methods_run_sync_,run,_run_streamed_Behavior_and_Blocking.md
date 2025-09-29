The table below summarizes the core characteristics of these methods for quick comparison:

| Feature | `run_sync` | `run` | `run_streamed` |
| :--- | :--- | :--- | :--- |
| **Execution Type** | Synchronous (blocking)  | Asynchronous (non-blocking)  | Asynchronous (non-blocking)  |
| **Return Type** | `RunResult`  | `RunResult`  | `RunResultStreaming`  |
| **Primary Use Case** | Simple scripts, CLI tools, sequential tasks  | Web servers (FastAPI), async applications, Jupyter notebooks  | Real-time chat interfaces, live terminal UIs, progress updates  |
| **Key Differentiator** | Blocks program execution until the agent run is complete  | Uses `await`; allows other tasks to run concurrently  | Returns a stream of events as they occur  |

### 🔍 A Closer Look at Each Method

Here is a more detailed breakdown of each runner method's operation and usage.

- **The `run_sync` Method: Synchronous and Blocking**
    - **Behavior**: The `Runner.run_sync()` method is the synchronous counterpart to `run`. When called, it **blocks the execution of the main program thread** until the agent has finished processing and produces a final output or an exception is raised . This makes its behavior straightforward to reason about but unsuitable for applications that need to handle multiple operations concurrently.
    - **Internal Workings**: It's important to note that this method is a wrapper around the async `run()` method. The official documentation cautions that it "will not work if there's already an event loop (e.g. inside an async function, or in a Jupyter notebook or async context like FastAPI)" . In these environments, you must use the async `run()` method instead .
    - **Usage Example**:
        ```python
        from agents import Agent, Runner

        agent = Agent(name="Assistant", instructions="You are a helpful assistant.")
        result = Runner.run_sync(agent, "What is the capital of France?")
        print(result.final_output)
        ```

- **The `run` Method: Asynchronous and Non-Blocking**
    - **Behavior**: The `Runner.run()` method is asynchronous. It allows your program to continue processing other tasks while waiting for the agent's response, making it much more efficient for I/O-bound applications . This is the recommended method for most real-world applications, especially those involving network calls or handling multiple user requests .
    - **Usage**: Because it is an asynchronous method, you must use the `await` keyword when calling it, and it must be inside an async function .
    - **Usage Example**:
        ```python
        import asyncio
        from agents import Agent, Runner

        async def main():
            agent = Agent(name="Assistant", instructions="You are a helpful assistant.")
            result = await Runner.run(agent, "What is the capital of France?")
            print(result.final_output)

        asyncio.run(main())
        ```

- **The `run_streamed` Method: Asynchronous and Real-Time Streaming**
    - **Behavior**: The `Runner.run_streamed()` method provides real-time, incremental updates during the agent's execution . It returns a `RunResultStreaming` object immediately, which provides an asynchronous stream of events . This allows you to see text chunks, tool calls, and other events as they happen, which is ideal for creating responsive user interfaces .
    - **Event Types**: You can iterate over the event stream and filter for different types of events, such as `ResponseTextDeltaEvent` for text chunks or `run_item_stream_event` for higher-level actions like tool calls and handoffs .
    - **Usage Example**:
        ```python
        import asyncio
        from agents import Agent, Runner

        async def main():
            agent = Agent(name="Assistant", instructions="Tell me a story.")
            result = Runner.run_streamed(agent, "Once upon a time...")

            async for event in result.stream_events():
                # Check for and process text delta events
                if event.type == "raw_response_event":
                    # Print text chunks as they are generated
                    print(event.data.delta, end="", flush=True)

        asyncio.run(main())
        ```

### ⚙️ Shared Parameters and Configuration

All three `run` methods share a common set of parameters that control the agent's execution .

- **`max_turns`**: This parameter defines the maximum number of turns (AI invocations) allowed in a single run and serves as a crucial safeguard against infinite loops. The default value is 5 . If exceeded, a `MaxTurnsExceeded` exception is raised .
- **`run_config`**: An optional `RunConfig` object that lets you override global settings for the entire agent run . Key configurations include:
    - `model`: Override the LLM model for all agents in the run .
    - `model_settings`: Globally set parameters like `temperature` or `top_p` .
    - `input_guardrails` / `output_guardrails`: Apply global validation checks .
    - `tracing_disabled`: Disable built-in tracing for the run .
    - `workflow_name`: Label the run for easier tracking in observability dashboards .
- **`session`**: Pass a `Session` object (e.g., `SQLiteSession`) for automatic conversation history management across multiple turns, eliminating the need for manual state handling .

### ⚠️ Potential Exceptions

Be prepared to handle these common exceptions that can occur during an agent run :
- `MaxTurnsExceeded`: Raised when the agent run exceeds the `max_turns` limit .
- `ModelBehaviorError`: Occurs if the LLM produces unexpected or invalid outputs, like malformed JSON for tool calls .
- `InputGuardrailTripwireTriggered` / `OutputGuardrailTripwireTriggered`: Raised when a configured input or output guardrail check fails, halting the run .

