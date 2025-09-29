# Sync vs Async Tool Integration in OpenAI Agents SDK

The OpenAI Agents SDK provides comprehensive support for both synchronous and asynchronous tool integration, allowing you to choose the right approach based on your application's needs and performance requirements.

## 🔄 Core Concepts

### Execution Models
| Aspect | Synchronous Tools | Asynchronous Tools |
| :--- | :--- | :--- |
| **Execution** | Blocking - runs in main thread | Non-blocking - runs in event loop |
| **Performance** | Suitable for CPU-bound operations | Ideal for I/O-bound operations |
| **Complexity** | Simpler to implement and debug | More complex but more scalable |
| **Use Cases** | Simple calculations, data processing | API calls, database queries, file I/O |

## 🛠️ Implementation Patterns

### Basic Synchronous Tools
```python
from agents import function_tool
import time

@function_tool
def calculate_metrics(data: dict) -> dict:
    """Calculate business metrics from data.
    
    Args:
        data: Dictionary containing sales and operational data
    """
    # Simulate CPU-intensive calculation
    total_revenue = sum(data.get('sales', []))
    average_order = total_revenue / len(data.get('sales', [1]))
    
    # This blocks the event loop
    time.sleep(2)  # Simulate processing time
    
    return {
        'total_revenue': total_revenue,
        'average_order_value': average_order,
        'processed_items': len(data.get('sales', []))
    }
```

### Basic Asynchronous Tools
```python
from agents import function_tool
import aiohttp
import asyncio
from typing import List

@function_tool
async def fetch_multiple_apis(urls: List[str]) -> dict:
    """Fetch data from multiple APIs concurrently.
    
    Args:
        urls: List of API endpoints to fetch data from
    """
    async with aiohttp.ClientSession() as session:
        tasks = []
        for url in urls:
            task = fetch_single_api(session, url)
            tasks.append(task)
        
        # Execute all API calls concurrently
        results = await asyncio.gather(*tasks, return_exceptions=True)
    
    return {
        'successful_responses': [r for r in results if not isinstance(r, Exception)],
        'errors': [str(r) for r in results if isinstance(r, Exception)]
    }

async def fetch_single_api(session: aiohttp.ClientSession, url: str) -> dict:
    """Fetch data from a single API endpoint."""
    try:
        async with session.get(url, timeout=30) as response:
            return await response.json()
    except Exception as e:
        return {"error": str(e), "url": url}
```

## ⚡ Advanced Integration Patterns

### Mixed Sync/Async Tool Sets
```python
from agents import Agent, function_tool
import sqlite3
import aiohttp
import asyncio

# Synchronous tool for local database operations
@function_tool
def query_local_database(query: str) -> list:
    """Query local SQLite database (synchronous)."""
    conn = sqlite3.connect('local_data.db')
    cursor = conn.cursor()
    
    try:
        cursor.execute(query)
        results = cursor.fetchall()
        return [dict(row) for row in results]
    finally:
        conn.close()

# Asynchronous tool for external API calls
@function_tool
async def fetch_external_data(api_endpoint: str) -> dict:
    """Fetch data from external API (asynchronous)."""
    async with aiohttp.ClientSession() as session:
        async with session.get(api_endpoint) as response:
            return await response.json()

# Create agent with mixed tool types
data_agent = Agent(
    name="Data Integration Agent",
    instructions="Use appropriate tools for different data sources",
    tools=[query_local_database, fetch_external_data]
)
```

### Context-Aware Async Tools
```python
from agents import function_tool, RunContextWrapper
from typing import Any
import aiohttp
import asyncio

@function_tool
async def context_aware_api_call(
    ctx: RunContextWrapper[Any], 
    endpoint: str
) -> dict:
    """Make API call with context-aware authentication.
    
    Args:
        ctx: Run context for accessing shared dependencies
        endpoint: API endpoint to call
    """
    # Access shared context (e.g., authentication tokens)
    auth_token = ctx.context.get('auth_token')
    rate_limiter = ctx.context.get('rate_limiter')
    
    # Wait for rate limiting if applicable
    if rate_limiter:
        await rate_limiter.acquire()
    
    headers = {'Authorization': f'Bearer {auth_token}'}
    
    async with aiohttp.ClientSession() as session:
        async with session.get(endpoint, headers=headers) as response:
            return {
                'status': response.status,
                'data': await response.json(),
                'endpoint': endpoint
            }
```

## 🚀 Performance Optimization

### Concurrent Async Tool Execution
```python
from agents import function_tool
import asyncio
from typing import List, Dict

@function_tool
async def parallel_data_processing(tasks: List[Dict]) -> List[Dict]:
    """Process multiple data tasks in parallel.
    
    Args:
        tasks: List of task specifications to process
    """
    # Create semaphore to limit concurrent operations
    semaphore = asyncio.Semaphore(5)  # Max 5 concurrent tasks
    
    async def process_single_task(task: Dict) -> Dict:
        async with semaphore:
            # Simulate async processing
            await asyncio.sleep(1)
            return {
                'task_id': task.get('id'),
                'processed': True,
                'result': f"Processed {task.get('name')}"
            }
    
    # Execute all tasks concurrently
    processed_tasks = await asyncio.gather(
        *[process_single_task(task) for task in tasks]
    )
    
    return processed_tasks
```

### Hybrid Sync/Async Wrapper
```python
from agents import function_tool
import asyncio
from concurrent.futures import ThreadPoolExecutor
import requests  # synchronous requests library

# Thread pool for running sync code in async context
executor = ThreadPoolExecutor(max_workers=10)

@function_tool
async def hybrid_api_call(url: str, use_async: bool = True) -> dict:
    """Make API call using either sync or async approach.
    
    Args:
        url: API endpoint to call
        use_async: Whether to use async implementation
    """
    if use_async:
        # Use native async implementation
        return await async_api_call(url)
    else:
        # Run synchronous code in thread pool
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(
            executor, 
            sync_api_call, 
            url
        )

async def async_api_call(url: str) -> dict:
    """Native async API call implementation."""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

def sync_api_call(url: str) -> dict:
    """Synchronous API call implementation."""
    response = requests.get(url, timeout=30)
    return response.json()
```

## ⚠️ Error Handling and Resilience

### Robust Async Tool with Retries
```python
from agents import function_tool
import aiohttp
import asyncio
from typing import Optional

@function_tool
async def resilient_api_call(
    url: str,
    max_retries: int = 3,
    timeout: int = 30
) -> dict:
    """Make API call with retry logic and error handling.
    
    Args:
        url: API endpoint to call
        max_retries: Maximum number of retry attempts
        timeout: Request timeout in seconds
    """
    last_exception = None
    
    for attempt in range(max_retries):
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(url, timeout=timeout) as response:
                    if response.status == 200:
                        return {
                            'success': True,
                            'data': await response.json(),
                            'attempts': attempt + 1
                        }
                    else:
                        last_exception = f"HTTP {response.status}"
                        
        except asyncio.TimeoutError:
            last_exception = "Request timeout"
        except Exception as e:
            last_exception = str(e)
        
        # Exponential backoff before retry
        if attempt < max_retries - 1:
            await asyncio.sleep(2 ** attempt)
    
    return {
        'success': False,
        'error': last_exception,
        'attempts': max_retries
    }
```

## 📊 Best Practices

### 1. Choose the Right Approach
- **Use Synchronous Tools For**:
  - CPU-intensive calculations
  - Simple data transformations
  - Operations with minimal I/O
  - Quick, deterministic functions

- **Use Asynchronous Tools For**:
  - Network requests (APIs, web scraping)
  - Database operations
  - File I/O operations
  - Any operation with waiting periods

### 2. Resource Management
```python
from agents import function_tool
import aiohttp
from typing import Any

@function_tool
async def managed_resource_tool(operation: str) -> Any:
    """Example of proper async resource management."""
    # Use context managers for proper resource cleanup
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.example.com/data') as response:
            data = await response.json()
            
            # Process data asynchronously
            processed_data = await process_data_async(data)
            
            return processed_data

async def process_data_async(data: dict) -> dict:
    """Async data processing function."""
    # Simulate async processing
    await asyncio.sleep(1)
    return {'processed': True, 'items': len(data)}
```

### 3. Performance Monitoring
```python
import time
import asyncio
from agents import function_tool
from contextlib import contextmanager

@contextmanager
def timer(operation_name: str):
    """Context manager to measure execution time."""
    start = time.time()
    try:
        yield
    finally:
        duration = time.time() - start
        print(f"{operation_name} took {duration:.2f} seconds")

@function_tool
async def monitored_async_tool() -> dict:
    """Async tool with performance monitoring."""
    with timer("Async API Call"):
        await asyncio.sleep(1)  # Simulate work
        return {'status': 'completed'}
```
