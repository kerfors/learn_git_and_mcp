# MCP Learning Reference Notebook

## Table of Contents
1. [Introduction to MCP (Model Context Protocol)](#introduction)
2. [Setting Up MCP in Jupyter](#setup)
3. [Basic MCP Server Operations](#basic-operations)
4. [Common MCP Servers](#common-servers)
5. [Best Practices](#best-practices)
6. [Troubleshooting Guide](#troubleshooting)
7. [Example Projects](#examples)
8. [References](#references)

## 1. Introduction to MCP (Model Context Protocol) <a name="introduction"></a>

MCP (Model Context Protocol) allows AI agents to connect to external tools and services, extending their capabilities beyond their base knowledge. Think of MCP servers as "plugins" that enable the AI to interact with real-world data and services.

### Key Concepts:
- **MCP Server**: A service that provides tools/resources to the AI agent
- **MCPServerStdio**: Used to start MCP servers via command line
- **Agent**: The Pydantic AI model that connects to MCP servers
- **Tools**: Functions or resources provided by MCP servers

## 2. Setting Up MCP in Jupyter <a name="setup"></a>

### Basic Setup Pattern
```python
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPServerStdio
import asyncio
import os

# API Key setup
API_KEY = os.environ.get("ANTHROPIC_API_KEY")
os.environ["ANTHROPIC_API_KEY"] = API_KEY

# For Jupyter environments - use await instead of asyncio.run()
# BAD:  asyncio.run(main())  
# GOOD: await main()

# Remember to update deprecated code:
# OLD: result.data
# NEW: result.output
```

### Import Statement Reference
```python
# Core imports for MCP
from pydantic_ai import Agent
from pydantic_ai.mcp import MCPServerStdio

# Async support
import asyncio

# Time and datetime utilities
from datetime import datetime
import time

# File operations
import os
from pathlib import Path
import json

# Process management
import subprocess
import sys
```

## 3. Basic MCP Server Operations <a name="basic-operations"></a>

### Creating a Single MCP Server
```python
# Basic server setup
server = MCPServerStdio(
    command='npx',
    args=['-y', 'package-name'],
    env=os.environ
)

# Create agent with server
agent = Agent('model-name', mcp_servers=[server])

# Use the agent with context manager
async with agent.run_mcp_servers():
    result = await agent.run("Your prompt here")
    print(result.output)
```

### Testing MCP Server Connection
```python
async def test_mcp_server(server_config):
    """Test if an MCP server can start successfully"""
    try:
        agent = Agent('claude-3-7-sonnet-20250219', mcp_servers=[server_config])
        async with agent.run_mcp_servers():
            result = await agent.run("Simple test prompt")
            return True, result.output
    except Exception as e:
        return False, str(e)

# Example usage
airbnb_server = MCPServerStdio(
    command='npx',
    args=['-y', '@openbnb/mcp-server-airbnb'],
    env=os.environ
)

success, message = await test_mcp_server(airbnb_server)
print(f"Server test {'passed' if success else 'failed'}: {message}")
```

### Timeout Handling
```python
async def run_with_timeout(agent, prompt, timeout=30):
    """Run agent with timeout to prevent hanging"""
    try:
        async def run_agent():
            async with agent.run_mcp_servers():
                return await agent.run(prompt)
        
        return await asyncio.wait_for(run_agent(), timeout=timeout)
    except asyncio.TimeoutError:
        return f"Operation timed out after {timeout} seconds"
```

## 4. Common MCP Servers <a name="common-servers"></a>

### Airbnb Server
```python
# Airbnb MCP Server for accommodation search
airbnb_server = MCPServerStdio(
    command='npx',
    args=['-y', '@openbnb/mcp-server-airbnb', '--ignore-robots-txt'],
    env=os.environ
)

# Example usage
agent = Agent(
    'claude-3-7-sonnet-20250219',
    system_prompt="You help find accommodations.",
    mcp_servers=[airbnb_server]
)

# Search for accommodations
prompt = "Find 3 Airbnbs in Paris for 2 people, check-in June 1, checkout June 5"
async with agent.run_mcp_servers():
    result = await agent.run(prompt)
```

### Filesystem Server
```python
# Filesystem MCP Server for file operations
filesystem_server = MCPServerStdio(
    command='npx',
    args=['-y', '@modelcontextprotocol/server-filesystem', './'],  # Directory to access
    env=os.environ
)

# Example file operations
agent = Agent('claude-3-7-sonnet-20250219', mcp_servers=[filesystem_server])

async with agent.run_mcp_servers():
    # Read files
    result = await agent.run("Read the content of README.md")
    
    # Write files
    result = await agent.run("Create a file named summary.txt with the content 'Project Summary'")
    
    # List directory
    result = await agent.run("List all files in the current directory")
```

### Google Maps Server
```python
# Google Maps MCP Server (sometimes unstable)
maps_server = MCPServerStdio(
    command='npx',
    args=['-y', '@modelcontextprotocol/server-google-maps'],
    env=os.environ
)

# Warning: This server can sometimes hang during startup
# Always test individually first
```

### Custom Tools
```python
# Add custom tools alongside MCP servers
async def get_current_time():
    return datetime.now().strftime('%Y-%m-%d %H:%M:%S')

# Combine with MCP servers
agent = Agent(
    'claude-3-7-sonnet-20250219',
    system_prompt="You're an assistant with time awareness.",
    mcp_servers=[filesystem_server],
    tools=[get_current_time]
)
```

## 5. Best Practices <a name="best-practices"></a>

### Progressive Server Testing
```python
# Test servers one at a time
async def test_all_servers():
    servers = {
        'filesystem': MCPServerStdio(command='npx', args=['-y', '@modelcontextprotocol/server-filesystem', './'], env=os.environ),
        'airbnb': MCPServerStdio(command='npx', args=['-y', '@openbnb/mcp-server-airbnb'], env=os.environ),
        'maps': MCPServerStdio(command='npx', args=['-y', '@modelcontextprotocol/server-google-maps'], env=os.environ)
    }
    
    results = {}
    for name, server in servers.items():
        print(f"Testing {name} server...")
        success, message = await test_mcp_server(server)
        results[name] = success
        if not success:
            print(f"{name} failed: {message}")
    
    return results

# Use only working servers
test_results = await test_all_servers()
working_servers = [server for name, server in servers.items() if test_results[name]]
```

### Error Handling Pattern
```python
async def robust_mcp_operation(agent, prompt):
    """Robust pattern for MCP operations"""
    try:
        # Short timeout for initial connection
        async with asyncio.timeout(10):
            async with agent.run_mcp_servers():
                # Longer timeout for actual operation
                async with asyncio.timeout(60):
                    return await agent.run(prompt)
    except asyncio.TimeoutError:
        return "Operation timed out - server may need more time to start"
    except Exception as e:
        return f"Error: {str(e)}"
```

### Fallback Strategies
```python
async def process_with_fallback(full_agent, simple_agent, prompt):
    """Try with all features, fall back to simpler version"""
    try:
        # Try full featured version
        result = await robust_mcp_operation(full_agent, prompt)
        if "timed out" not in result and "Error" not in result:
            return result
    except:
        pass
    
    # Fallback to simpler version
    print("Falling back to simpler configuration...")
    return await robust_mcp_operation(simple_agent, prompt)
```

## 6. Troubleshooting Guide <a name="troubleshooting"></a>

### Common Issues and Solutions

| Problem | Likely Cause | Solution |
|---------|--------------|----------|
| Script hangs at "Starting MCP servers..." | NPX package download delay or server startup issue | Add timeout, test servers individually |
| `RuntimeError: asyncio.run() cannot be called from a running event loop` | Using `asyncio.run()` in Jupyter | Use `await` directly |
| `result.data is deprecated` | Using old API | Switch to `result.output` |
| "NPX not found" | Node.js not installed | Install Node.js and npm |
| Server starts but no response | API key issues | Verify API key environment variable |
| Google Maps server hangs | Known instability | Test individually, have fallback |

### Debugging Template
```python
# Comprehensive debugging setup
async def debug_mcp_setup():
    # Check prerequisites
    print("Checking prerequisites...")
    print(f"API Key set: {bool(os.environ.get('ANTHROPIC_API_KEY'))}")
    
    # Test basic agent
    try:
        basic_agent = Agent('claude-3-7-sonnet-20250219')
        result = await basic_agent.run("Test")
        print("Basic agent: ✓ Working")
    except Exception as e:
        print(f"Basic agent: ✗ Failed ({e})")
    
    # Test NPX
    try:
        subprocess.run(['npx', '--version'], check=True, capture_output=True)
        print("NPX: ✓ Available")
    except:
        print("NPX: ✗ Not found - install Node.js")
    
    # Test individual servers
    servers_to_test = {
        'filesystem': MCPServerStdio(command='npx', args=['-y', '@modelcontextprotocol/server-filesystem', './'], env=os.environ),
        'airbnb': MCPServerStdio(command='npx', args=['-y', '@openbnb/mcp-server-airbnb'], env=os.environ)
    }
    
    for name, server in servers_to_test.items():
        print(f"\nTesting {name} server...")
        success, msg = await test_mcp_server(server)
        print(f"{name}: {'✓' if success else '✗'} {msg}")

await debug_mcp_setup()
```

## 7. Example Projects <a name="examples"></a>

### Travel Assistant
```python
async def travel_assistant_example():
    """Complete travel planning assistant"""
    
    # Server setup
    airbnb_server = MCPServerStdio(
        command='npx',
        args=['-y', '@openbnb/mcp-server-airbnb', '--ignore-robots-txt'],
        env=os.environ
    )
    
    filesystem_server = MCPServerStdio(
        command='npx',
        args=['-y', '@modelcontextprotocol/server-filesystem', './'],
        env=os.environ
    )
    
    # Custom time tool
    async def get_time():
        return datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    
    # Create agent
    agent = Agent(
        'claude-3-7-sonnet-20250219',
        system_prompt="""You're a travel agent who helps find accommodations. 
        You can search Airbnb, save results to files, and provide current time.""",
        mcp_servers=[airbnb_server, filesystem_server],
        tools=[get_time]
    )
    
    # Example interaction
    async with agent.run_mcp_servers():
        # Search for accommodations
        result = await agent.run("""
        Find 3 Airbnbs near Eiffel Tower in Paris:
        - Dates: December 20-25
        - 2 adults
        - Save results to paris_accommodations.md in markdown table format
        """)
        
        print(result.output)
        
        # Verify file creation
        if os.path.exists("paris_accommodations.md"):
            with open("paris_accommodations.md", "r") as f:
                print("\nSaved results:")
                print(f.read())

# Run the example
await travel_assistant_example()
```

### Data Analysis Assistant
```python
async def data_analysis_example():
    """Analyze data files using MCP"""
    
    filesystem_server = MCPServerStdio(
        command='npx',
        args=['-y', '@modelcontextprotocol/server-filesystem', './'],
        env=os.environ
    )
    
    agent = Agent(
        'claude-3-7-sonnet-20250219',
        system_prompt="""You're a data analyst. You can read CSV files, 
        analyze data, and create summary reports.""",
        mcp_servers=[filesystem_server]
    )
    
    async with agent.run_mcp_servers():
        # Read and analyze data
        result = await agent.run("""
        Read the file data.csv and provide:
        1. Basic statistics (count, mean, etc.)
        2. Any interesting patterns
        3. Save summary to data_summary.md
        """)
        
        print(result.output)

# Run the example
await data_analysis_example()
```

## 8. References <a name="references"></a>

### Official Documentation
- [Pydantic AI Documentation](https://docs.pydantic.dev/latest/)
- [Anthropic API Documentation](https://docs.anthropic.com/)
- [Model Context Protocol Specification](https://spec.modelcontextprotocol.io/)

### Available MCP Servers
- `@openbnb/mcp-server-airbnb` - Airbnb search
- `@modelcontextprotocol/server-filesystem` - File operations
- `@modelcontextprotocol/server-google-maps` - Location services
- `@modelcontextprotocol/server-sqlite` - Database access
- `@modelcontextprotocol/server-github` - GitHub integration

### Useful Commands
```bash
# Check NPX version
npx --version

# Clear NPX cache if needed
npx clear-npx-cache

# Install specific MCP server manually
npx @openbnb/mcp-server-airbnb

# Check Node.js installation
node --version
npm --version
```

### Advanced Tips
1. **Server Versioning**: Pin server versions for stability
   ```python
   args=['-y', '@openbnb/mcp-server-airbnb@1.0.0']
   ```

2. **Environment Variables**: Use .env files for API keys
   ```python
   from dotenv import load_dotenv
   load_dotenv()
   ```

3. **Logging**: Add comprehensive logging for debugging
   ```python
   import logging
   logging.basicConfig(level=logging.INFO)
   ```

4. **Rate Limiting**: Implement delays between requests
   ```python
   await asyncio.sleep(1)  # 1 second delay
   ```

## Happy MCP Learning! 🚀

Remember: Always test servers individually before combining them, and have fallback strategies for unreliable servers.