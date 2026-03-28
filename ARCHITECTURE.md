# Hermes Agent - Architecture Documentation

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Core Components](#core-components)
4. [Agent Execution Flow](#agent-execution-flow)
5. [Tool System](#tool-system)
6. [Skills System](#skills-system)
7. [CLI Architecture](#cli-architecture)
8. [Gateway Architecture](#gateway-architecture)
9. [Method Usage Guide](#method-usage-guide)
10. [Extension Patterns](#extension-patterns)

---

## Overview

Hermes Agent is a self-improving AI agent system with a built-in learning loop. It features:

- **Multi-platform support**: CLI, Telegram, Discord, Slack, WhatsApp, Signal
- **Tool orchestration**: 40+ tools organized into toolsets
- **Skills system**: Agent-curated procedural memory and autonomous skill creation
- **Session persistence**: SQLite-based conversation storage with full-text search
- **Provider agnostic**: Works with any OpenAI-compatible API

### Key Design Principles

1. **Self-registering tools**: Tools automatically register themselves at import time
2. **Toolset grouping**: Tools are organized into logical groups that can be enabled/disabled
3. **Session persistence**: All conversations stored with full-text search capability
4. **Ephemeral injection**: System prompts never persisted to maintain prompt caching
5. **Provider abstraction**: Works with any OpenAI-compatible API

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Entry Points                             │
├─────────────────┬───────────────────────┬──────────────────────┤
│   CLI (cli.py)  │  Gateway (gateway/)   │  Batch (batch_runner.py) │
└────────┬────────┴───────────┬───────────┴──────────┬───────────┘
         │                    │                       │
         └────────────────────┼───────────────────────┘
                              │
         ┌────────────────────▼────────────────────┐
         │      AIAgent (run_agent.py)             │
         │  - Core conversation loop               │
         │  - Tool dispatch                        │
         │  - Session management                   │
         └────────┬────────────────────┬───────────┘
                  │                    │
      ┌───────────▼──────────┐    ┌───▼─────────────────┐
      │  Prompt Builder      │    │  Tool Registry      │
      │  (agent/prompt_*)    │    │  (tools/registry.py)│
      └──────────────────────┘    └───┬─────────────────┘
                                      │
                            ┌─────────▼──────────┐
                            │   Tool Handlers    │
                            │  (tools/*.py)      │
                            └────────────────────┘
```

### Component Layers

| Layer | Components | Responsibility |
|-------|------------|----------------|
| **Interface** | CLI, Gateway, Batch Runner | User interaction, message routing |
| **Core Agent** | AIAgent, Prompt Builder | Conversation loop, LLM orchestration |
| **Tool System** | Registry, Handlers, Toolsets | Tool discovery, dispatch, execution |
| **Persistence** | SessionDB, Memory | Session storage, full-text search |
| **Backends** | Terminal environments | Local, Docker, SSH, Modal, Daytona execution |

---

## Core Components

### AIAgent Class (`run_agent.py`)

The central orchestrator for all agent conversations.

**Key Methods:**

```python
class AIAgent:
    def __init__(
        self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        platform: str = None,
        session_id: str = None,
        **kwargs
    ):
        """Initialize the agent with configuration."""

    def chat(self, message: str) -> str:
        """Simple synchronous interface - returns final response string."""

    def run_conversation(
        self,
        user_message: str,
        system_message: str = None,
        conversation_history: list = None,
        task_id: str = None
    ) -> dict:
        """
        Full conversation interface - returns dict with:
        - final_response: str
        - messages: list
        - tool_calls: list
        """
```

**Usage Example:**

```python
from run_agent import AIAgent

# Initialize agent
agent = AIAgent(
    model="anthropic/claude-sonnet-4.5",
    enabled_toolsets=["terminal", "file", "web"],
    platform="cli"
)

# Simple chat
response = agent.chat("List files in current directory")
print(response)

# Full conversation with history
result = agent.run_conversation(
    user_message="Analyze this error",
    conversation_history=previous_messages
)
print(result['final_response'])
```

### Prompt Builder (`agent/prompt_builder.py`)

Assembles system prompts from multiple sources.

**Key Functions:**

```python
def build_system_prompt(
    agent,
    include_memory: bool = True,
    include_skills: bool = True,
    include_context_files: bool = True
) -> str:
    """Build complete system prompt from all sources."""

def build_skills_system_prompt(
    skills_dir: str,
    available_tools: set,
    available_toolsets: set
) -> str:
    """Build skills section with conditional activation."""
```

**Usage Pattern:**

The prompt builder automatically:
1. Loads base identity/personality
2. Injects enabled tools and toolsets
3. Adds active skills based on tool availability
4. Includes memory and context files
5. Formats everything for optimal prompt caching

### Tool Registry (`tools/registry.py`)

Central registry for all tools in the system.

**Key Methods:**

```python
class ToolRegistry:
    def register(
        self,
        name: str,
        toolset: str,
        schema: dict,
        handler: callable,
        check_fn: callable = None,
        requires_env: list = None
    ):
        """Register a tool with its schema and handler."""

    def get_tool_schemas(
        self,
        enabled_toolsets: list,
        disabled_toolsets: list = None
    ) -> list:
        """Get schemas for all available tools in enabled toolsets."""

    def dispatch(
        self,
        name: str,
        args: dict,
        **kwargs
    ) -> str:
        """Execute a tool handler by name."""
```

**Usage in Tools:**

```python
# tools/example_tool.py
from tools.registry import registry
import json

def my_tool(param: str, task_id: str = None) -> str:
    """Tool implementation."""
    result = {"status": "success", "data": param}
    return json.dumps(result)

SCHEMA = {
    "type": "function",
    "function": {
        "name": "my_tool",
        "description": "What this tool does",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "Parameter description"}
            },
            "required": ["param"]
        }
    }
}

# Self-register at import time
registry.register(
    name="my_tool",
    toolset="example",
    schema=SCHEMA,
    handler=lambda args, **kw: my_tool(**args, **kw)
)
```

### Session Database (`hermes_state.py`)

SQLite-based session persistence with full-text search.

**Key Methods:**

```python
class SessionDB:
    def save_session(
        self,
        session_id: str,
        messages: list,
        metadata: dict = None
    ):
        """Save conversation to database."""

    def load_session(self, session_id: str) -> list:
        """Load conversation history."""

    def search_sessions(
        self,
        query: str,
        limit: int = 10
    ) -> list:
        """Full-text search across all sessions."""

    def get_session_title(self, session_id: str) -> str:
        """Get AI-generated title for session."""
```

**Usage Example:**

```python
from hermes_state import SessionDB

db = SessionDB()

# Save a session
db.save_session(
    session_id="cli_20260328_1234",
    messages=[
        {"role": "user", "content": "Hello"},
        {"role": "assistant", "content": "Hi!"}
    ],
    metadata={"platform": "cli", "model": "claude-sonnet-4.5"}
)

# Search conversations
results = db.search_sessions("error handling")
for session_id, snippet in results:
    print(f"{session_id}: {snippet}")
```

---

## Agent Execution Flow

### Complete Conversation Loop

```
1. User Input
   ↓
2. Load Session History (if exists)
   ↓
3. Build System Prompt
   ├─ Load identity/personality
   ├─ Inject enabled tools
   ├─ Add active skills
   ├─ Include memory
   └─ Add context files
   ↓
4. Prepare Messages
   ├─ System prompt (ephemeral)
   ├─ Conversation history
   └─ New user message
   ↓
5. Call LLM API
   ├─ model: selected model
   ├─ messages: prepared messages
   ├─ tools: enabled tool schemas
   └─ reasoning: if supported
   ↓
6. Process Response
   ├─ Has tool_calls?
   │  ├─ Yes → Execute each tool
   │  │        Add results to messages
   │  │        Go back to step 5
   │  └─ No → Continue
   ↓
7. Extract Final Response
   ↓
8. Save to Database
   ├─ Update session
   └─ Generate title (if new)
   ↓
9. Return to User
```

### Tool Execution Flow

```python
# Pseudo-code of tool execution
def execute_tool_call(tool_call):
    # 1. Extract tool name and arguments
    name = tool_call.function.name
    args = json.loads(tool_call.function.arguments)

    # 2. Check if tool is available
    if name not in registry.tools:
        return error_response("Tool not found")

    # 3. Validate requirements
    if not registry.check_requirements(name):
        return error_response("Requirements not met")

    # 4. Dispatch to handler
    try:
        result = registry.dispatch(name, args, task_id=task_id)
        return result
    except Exception as e:
        return error_response(str(e))
```

### Context Compression Flow

When approaching token limits:

```
1. Detect token limit approaching
   ↓
2. Trigger compression
   ├─ Keep system prompt
   ├─ Keep recent N messages
   └─ Summarize older messages
   ↓
3. Call auxiliary LLM
   └─ Generate concise summary
   ↓
4. Inject summary
   └─ Replace old messages with summary message
   ↓
5. Continue conversation
```

---

## Tool System

### Tool Architecture

Tools are self-contained modules that:
1. Define their schema (OpenAI function format)
2. Implement a handler function
3. Register themselves at import time
4. Declare requirements and dependencies

### Tool File Structure

```python
# tools/example_tool.py

"""example_tool — Brief description of what this tool does."""

import json
import os
from tools.registry import registry

# 1. Handler implementation
def example_tool(param1: str, param2: int = 10, task_id: str = None) -> str:
    """
    Tool handler function.

    Args:
        param1: First parameter
        param2: Optional second parameter
        task_id: Optional task ID for context

    Returns:
        JSON string with result
    """
    result = {
        "success": True,
        "param1": param1,
        "param2": param2
    }
    return json.dumps(result)

# 2. Schema definition
EXAMPLE_TOOL_SCHEMA = {
    "type": "function",
    "function": {
        "name": "example_tool",
        "description": "What this tool does and when to use it.",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "Description of param1"
                },
                "param2": {
                    "type": "integer",
                    "description": "Description of param2",
                    "default": 10
                }
            },
            "required": ["param1"]
        }
    }
}

# 3. Requirements check
def _check_requirements() -> bool:
    """Check if tool dependencies are available."""
    return bool(os.getenv("EXAMPLE_API_KEY"))

# 4. Self-registration
registry.register(
    name="example_tool",
    toolset="example",  # Toolset group
    schema=EXAMPLE_TOOL_SCHEMA,
    handler=lambda args, **kw: example_tool(**args, **kw),
    check_fn=_check_requirements,
    requires_env=["EXAMPLE_API_KEY"]
)
```

### Toolset System

Tools are organized into toolsets (`toolsets.py`):

```python
# Core tools - always available
_HERMES_CORE_TOOLS = [
    "read_file", "write_file", "search_code",
    "list_files", "terminal"
]

# Toolset definitions
TOOLSETS = {
    "file": ["read_file", "write_file", "search_code", "list_files"],
    "terminal": ["terminal"],
    "web": ["web_search", "web_extract"],
    "browser": ["browser_navigate", "browser_screenshot"],
    "code": ["execute_code"],
    "delegate": ["delegate_task"],
}

# Platform presets
PLATFORM_PRESETS = {
    "cli": ["file", "terminal", "web", "code", "delegate"],
    "telegram": ["file", "web", "code"],
    "discord": ["file", "web", "code"],
}
```

### Adding a New Tool

**Step 1: Create tool file**

```python
# tools/my_new_tool.py
from tools.registry import registry
import json

def my_new_tool(input_text: str) -> str:
    result = {"output": input_text.upper()}
    return json.dumps(result)

SCHEMA = {
    "type": "function",
    "function": {
        "name": "my_new_tool",
        "description": "Converts text to uppercase",
        "parameters": {
            "type": "object",
            "properties": {
                "input_text": {"type": "string", "description": "Text to convert"}
            },
            "required": ["input_text"]
        }
    }
}

registry.register(
    name="my_new_tool",
    toolset="text",
    schema=SCHEMA,
    handler=lambda args, **kw: my_new_tool(**args, **kw)
)
```

**Step 2: Add import to `model_tools.py`**

```python
_modules = [
    # ... existing modules ...
    "tools.my_new_tool",
]
```

**Step 3: Add to toolset in `toolsets.py`**

```python
TOOLSETS = {
    # ... existing toolsets ...
    "text": ["my_new_tool"],
}
```

---

## Skills System

Skills are instruction sets that guide the agent through specific tasks.

### Skill Structure

```
skills/
├── category/
│   └── skill-name/
│       ├── SKILL.md          # Main instructions (required)
│       ├── scripts/          # Helper scripts (optional)
│       │   └── helper.py
│       └── references/       # Reference materials (optional)
│           └── docs.pdf
```

### SKILL.md Format

```markdown
---
name: my-skill
description: Brief description for skill search
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]  # Optional: restrict to specific platforms
required_environment_variables:
  - name: API_KEY
    prompt: Your API key
    help: Get it from https://example.com
    required_for: full functionality
metadata:
  hermes:
    tags: [Category, Keywords]
    related_skills: [other-skill]
    fallback_for_toolsets: [web]    # Show only when web toolset unavailable
    requires_toolsets: [terminal]   # Show only when terminal available
---

# Skill Title

## When to Use
Describe when the agent should load this skill.

## Quick Reference
| Command | Description |
|---------|-------------|
| command1 | What it does |

## Procedure
1. Step one
2. Step two
3. Step three

## Pitfalls
- Known issue 1
- Known issue 2

## Verification
How to confirm the task was successful.
```

### Conditional Skill Activation

Skills can be shown/hidden based on tool availability:

```yaml
metadata:
  hermes:
    # Show ONLY when web toolset is unavailable (fallback)
    fallback_for_toolsets: [web]

    # Show ONLY when terminal toolset is available (dependency)
    requires_toolsets: [terminal]

    # Show ONLY when specific tool is unavailable
    fallback_for_tools: [web_search]

    # Show ONLY when specific tool is available
    requires_tools: [browser_navigate]
```

### Skill Loading Flow

```python
# Simplified skill loading
def load_skills(skills_dir, available_tools, available_toolsets):
    skills = []

    for skill_path in find_skill_files(skills_dir):
        metadata = parse_skill_metadata(skill_path)

        # Check platform compatibility
        if not is_platform_compatible(metadata):
            continue

        # Check tool/toolset conditions
        if not should_show_skill(metadata, available_tools, available_toolsets):
            continue

        # Load skill content
        content = read_skill_content(skill_path)
        skills.append(content)

    return "\n\n".join(skills)
```

---

## CLI Architecture

### CLI Components

```
cli.py (HermesCLI class)
├─ Banner display (Rich panels)
├─ Input handling (prompt_toolkit)
│  ├─ Multiline editing
│  ├─ Autocomplete
│  └─ History
├─ Slash command dispatch
├─ Spinner/progress (KawaiiSpinner)
└─ Response formatting
```

### Key CLI Methods

```python
class HermesCLI:
    def __init__(self, config: dict):
        """Initialize CLI with configuration."""

    def run(self):
        """Main CLI loop."""

    def process_command(self, command: str) -> bool:
        """
        Handle slash commands.
        Returns True if command was processed, False otherwise.
        """

    def handle_user_input(self, user_input: str):
        """Process user message and get agent response."""

    def display_response(self, response: str):
        """Format and display agent response."""
```

### Slash Command System

Commands are centrally defined in `hermes_cli/commands.py`:

```python
from dataclasses import dataclass

@dataclass
class CommandDef:
    name: str              # Canonical name (no slash)
    description: str       # Help text
    category: str          # "Session", "Configuration", etc.
    aliases: tuple = ()    # Alternative names
    args_hint: str = ""    # Argument placeholder
    cli_only: bool = False # Only in CLI
    gateway_only: bool = False  # Only in gateway

# Central registry
COMMAND_REGISTRY = [
    CommandDef("new", "Start a new conversation", "Session",
               aliases=("reset",)),
    CommandDef("model", "Change the model", "Configuration",
               args_hint="[provider:model]"),
    # ... more commands
]
```

### Adding a CLI Command

**Step 1: Add to registry**

```python
# hermes_cli/commands.py
COMMAND_REGISTRY = [
    # ... existing commands ...
    CommandDef("mycommand", "Does something", "Session",
               aliases=("mc",), args_hint="[optional-arg]"),
]
```

**Step 2: Add handler**

```python
# cli.py - in HermesCLI.process_command()
def process_command(self, cmd_original: str) -> bool:
    canonical = resolve_command(cmd_without_slash)

    # ... existing handlers ...

    elif canonical == "mycommand":
        self._handle_mycommand(cmd_original)
        return True

    return False

def _handle_mycommand(self, cmd: str):
    """Handle /mycommand."""
    # Implementation here
    pass
```

### Skin/Theme System

CLI appearance is data-driven via `hermes_cli/skin_engine.py`:

```python
# Skin configuration
skin = {
    "name": "mytheme",
    "colors": {
        "banner_border": "#FFD700",
        "response_border": "#87CEEB"
    },
    "spinner": {
        "thinking_faces": ["(◕‿◕)", "(⌐■_■)"],
        "thinking_verbs": ["pondering", "analyzing"]
    },
    "branding": {
        "agent_name": "My Agent",
        "prompt_symbol": "→ "
    }
}

# Activate skin
set_active_skin("mytheme")
```

---

## Gateway Architecture

### Gateway System

```
gateway/run.py (GatewayRunner)
├─ Platform lifecycle
│  ├─ Telegram
│  ├─ Discord
│  ├─ Slack
│  ├─ WhatsApp
│  └─ Signal
├─ Message routing
├─ Session management
└─ Cron integration
```

### Platform Adapter Pattern

```python
# gateway/platforms/example.py
class ExamplePlatform:
    def __init__(self, config: dict, agent_factory):
        """Initialize platform with config."""

    async def start(self):
        """Start platform listener."""

    async def handle_message(self, message):
        """Process incoming message."""
        # 1. Extract user info and text
        # 2. Load or create session
        # 3. Call agent
        # 4. Send response

    async def send_message(self, user_id: str, text: str):
        """Send message to user."""
```

### Gateway Message Flow

```
1. Platform receives message
   ↓
2. Extract user and content
   ↓
3. Load session for user
   ↓
4. Check for slash command
   ├─ Command? → Handle and return
   └─ No → Continue
   ↓
5. Get or create agent for session
   ↓
6. Send to agent
   ├─ Stream tool progress
   └─ Get final response
   ↓
7. Send response to platform
   ↓
8. Save session
```

---

## Method Usage Guide

### Common Usage Patterns

#### 1. Simple Agent Chat

```python
from run_agent import AIAgent

agent = AIAgent(model="anthropic/claude-sonnet-4.5")
response = agent.chat("Hello, how are you?")
print(response)
```

#### 2. Agent with Specific Toolsets

```python
agent = AIAgent(
    model="anthropic/claude-sonnet-4.5",
    enabled_toolsets=["file", "terminal", "web"],
    disabled_toolsets=["browser"]  # Explicitly disable
)

response = agent.chat("Search the web for Python tutorials")
```

#### 3. Session Management

```python
from hermes_state import SessionDB

db = SessionDB()
session_id = "my_session_123"

# Load existing session
history = db.load_session(session_id)

# Run conversation with history
agent = AIAgent(session_id=session_id)
result = agent.run_conversation(
    user_message="Continue the previous task",
    conversation_history=history
)

# Save updated session
db.save_session(session_id, result['messages'])
```

#### 4. Tool Registration

```python
from tools.registry import registry
import json

def my_tool(input: str) -> str:
    return json.dumps({"result": input.upper()})

registry.register(
    name="uppercase",
    toolset="text",
    schema={
        "type": "function",
        "function": {
            "name": "uppercase",
            "description": "Convert to uppercase",
            "parameters": {
                "type": "object",
                "properties": {
                    "input": {"type": "string"}
                },
                "required": ["input"]
            }
        }
    },
    handler=lambda args, **kw: my_tool(args["input"])
)
```

#### 5. Prompt Caching

```python
# Prompt caching is automatic but can be configured
agent = AIAgent(
    model="anthropic/claude-sonnet-4.5",
    skip_memory=False,  # Include memory (cached)
    skip_context_files=False  # Include context (cached)
)

# System prompt is cached across turns
# Only new messages invalidate cache
```

#### 6. Background Tool Execution

```python
# In tool implementation
def terminal(command: str, background: bool = False, check_interval: int = None):
    if background:
        # Start background process
        process_id = start_background_process(command)

        # Set up monitoring
        if check_interval:
            monitor_process(process_id, check_interval)

        return json.dumps({
            "status": "started",
            "process_id": process_id
        })
    else:
        # Run synchronously
        result = run_command(command)
        return json.dumps(result)
```

#### 7. Skill Management

```python
# Load all skills
from agent.prompt_builder import build_skills_system_prompt

skills_prompt = build_skills_system_prompt(
    skills_dir="~/.hermes/skills",
    available_tools={"terminal", "read_file", "web_search"},
    available_toolsets={"terminal", "file", "web"}
)

# Skills are automatically filtered based on:
# - Platform compatibility
# - Tool/toolset availability
# - Conditional activation rules
```

#### 8. Context Compression

```python
from agent.context_compressor import compress_messages

# When approaching token limit
if estimate_tokens(messages) > MAX_TOKENS * 0.8:
    compressed = compress_messages(
        messages=messages,
        keep_recent=10,  # Keep last 10 messages
        model="gpt-4o-mini"  # Use cheaper model for summary
    )
    messages = compressed
```

---

## Extension Patterns

### Pattern 1: Adding a New Platform

```python
# gateway/platforms/myplatform.py
import asyncio
from typing import Any

class MyPlatformAdapter:
    def __init__(self, config: dict, agent_factory):
        self.config = config
        self.agent_factory = agent_factory
        self.sessions = {}

    async def start(self):
        """Start platform listener."""
        # Initialize platform SDK
        # Set up event handlers
        pass

    async def handle_message(self, event: Any):
        """Process incoming message."""
        user_id = event.user_id
        text = event.text

        # Get or create session
        if user_id not in self.sessions:
            self.sessions[user_id] = {
                "session_id": f"myplatform_{user_id}",
                "history": []
            }

        session = self.sessions[user_id]

        # Get agent
        agent = self.agent_factory(session_id=session["session_id"])

        # Process message
        result = agent.run_conversation(
            user_message=text,
            conversation_history=session["history"]
        )

        # Update history
        session["history"] = result["messages"]

        # Send response
        await self.send_message(user_id, result["final_response"])

    async def send_message(self, user_id: str, text: str):
        """Send message via platform."""
        # Use platform SDK to send
        pass
```

### Pattern 2: Custom Tool with State

```python
# tools/stateful_tool.py
from tools.registry import registry
import json

# Module-level state (session-isolated in practice)
_cache = {}

def stateful_tool(action: str, key: str = None, value: str = None) -> str:
    """Tool that maintains state across calls."""
    if action == "set":
        _cache[key] = value
        return json.dumps({"status": "stored", "key": key})
    elif action == "get":
        value = _cache.get(key)
        return json.dumps({"key": key, "value": value})
    elif action == "list":
        return json.dumps({"keys": list(_cache.keys())})

registry.register(
    name="stateful_tool",
    toolset="state",
    schema={...},
    handler=lambda args, **kw: stateful_tool(**args)
)
```

### Pattern 3: Tool with External API

```python
# tools/api_tool.py
from tools.registry import registry
import json
import requests
import os

def api_tool(query: str) -> str:
    """Call external API."""
    api_key = os.getenv("MY_API_KEY")
    if not api_key:
        return json.dumps({"error": "API key not configured"})

    try:
        response = requests.post(
            "https://api.example.com/query",
            json={"query": query},
            headers={"Authorization": f"Bearer {api_key}"},
            timeout=30
        )
        response.raise_for_status()
        return json.dumps(response.json())
    except Exception as e:
        return json.dumps({"error": str(e)})

def _check_requirements():
    return bool(os.getenv("MY_API_KEY"))

registry.register(
    name="api_tool",
    toolset="api",
    schema={...},
    handler=lambda args, **kw: api_tool(**args),
    check_fn=_check_requirements,
    requires_env=["MY_API_KEY"]
)
```

### Pattern 4: Subagent Delegation

```python
# Using the delegate tool
def complex_task():
    # Main agent delegates subtask to specialized agent
    result = delegate_task(
        task_description="Analyze this code for security issues",
        context={"file": "app.py", "content": file_content},
        enabled_toolsets=["file", "code"],
        max_iterations=20
    )
    return result
```

### Pattern 5: Custom Skill Creation

```python
# Create skill programmatically
import os
from pathlib import Path

def create_custom_skill(name: str, instructions: str):
    """Create a new skill at runtime."""
    skill_dir = Path.home() / ".hermes" / "skills" / "custom" / name
    skill_dir.mkdir(parents=True, exist_ok=True)

    skill_content = f"""---
name: {name}
description: Custom skill created at runtime
version: 1.0.0
metadata:
  hermes:
    tags: [Custom]
---

# {name.title()}

{instructions}
"""

    skill_file = skill_dir / "SKILL.md"
    skill_file.write_text(skill_content)

    return str(skill_file)
```

---

## Best Practices

### 1. Prompt Caching

**DO:**
- Keep system prompt stable across turns
- Load memory and skills once at session start
- Use ephemeral injection for dynamic content

**DON'T:**
- Modify past messages mid-conversation
- Change toolsets mid-session
- Reload skills or memory unnecessarily

### 2. Tool Design

**DO:**
- Return JSON strings from handlers
- Validate inputs before processing
- Handle errors gracefully
- Document when to use the tool

**DON'T:**
- Return Python objects directly
- Raise exceptions without catching
- Make blocking calls without timeouts
- Reference tools from other toolsets in descriptions

### 3. Session Management

**DO:**
- Use unique, stable session IDs
- Save sessions after each turn
- Load history before continuing conversations

**DON'T:**
- Share sessions across users
- Modify session history arbitrarily
- Store sensitive data in session metadata

### 4. Error Handling

**DO:**
- Catch specific exceptions
- Log errors with context
- Return informative error messages
- Fail gracefully

**DON'T:**
- Let exceptions crash the agent loop
- Log sensitive data (API keys, passwords)
- Use bare except clauses

### 5. Cross-Platform Compatibility

**DO:**
- Use `pathlib.Path` for file paths
- Check platform before using Unix-specific APIs
- Handle encoding errors when reading files
- Test on Linux, macOS, and Windows

**DON'T:**
- Use Unix-only modules (`termios`, `fcntl`) without fallbacks
- Assume `/` as path separator
- Use `os.setsid()` on Windows

---

## Troubleshooting

### Common Issues

**Issue: Tool not appearing in agent**
- Check toolset is enabled
- Verify tool registered successfully
- Check requirements function returns True
- Ensure environment variables set

**Issue: Session not persisting**
- Check `~/.hermes/state.db` exists
- Verify write permissions
- Check session_id is stable

**Issue: Prompt caching not working**
- Ensure model supports caching (Anthropic models)
- Check system prompt is stable
- Verify not modifying past messages

**Issue: Skills not loading**
- Check SKILL.md format and frontmatter
- Verify platform compatibility
- Check conditional activation rules
- Ensure skill directory in skills path

---

## Additional Resources

- **[README.md](README.md)** - Project overview and quick start
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - Detailed contribution guide
- **[AGENTS.md](AGENTS.md)** - Development guide for AI assistants
- **[Documentation Website](https://hermes-agent.nousresearch.com/docs/)** - Complete user guide
- **[Discord](https://discord.gg/NousResearch)** - Community support

---

## License

MIT — see [LICENSE](LICENSE).
