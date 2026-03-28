# Hermes Agent - 架構文檔

## 目錄

1. [概覽](#概覽)
2. [系統架構](#系統架構)
3. [核心組件](#核心組件)
4. [Agent 執行流程](#agent-執行流程)
5. [工具系統](#工具系統)
6. [技能系統](#技能系統)
7. [CLI 架構](#cli-架構)
8. [Gateway 架構](#gateway-架構)
9. [方法使用指南](#方法使用指南)
10. [擴展模式](#擴展模式)

---

## 概覽

Hermes Agent 是一個具有內建學習循環的自我改進 AI 代理系統。主要特點：

- **多平台支持**：CLI、Telegram、Discord、Slack、WhatsApp、Signal
- **工具編排**：40+ 工具組織成工具集
- **技能系統**：代理管理的程序記憶和自主技能創建
- **會話持久化**：基於 SQLite 的對話存儲與全文搜索
- **提供商無關**：適用於任何 OpenAI 兼容的 API

### 核心設計原則

1. **自註冊工具**：工具在導入時自動註冊
2. **工具集分組**：工具組織成可啟用/禁用的邏輯組
3. **會話持久化**：所有對話存儲並支持全文搜索
4. **臨時注入**：系統提示從不持久化以維持提示緩存
5. **提供商抽象**：適用於任何 OpenAI 兼容的 API

---

## 系統架構

```
┌─────────────────────────────────────────────────────────────────┐
│                         入口點                                    │
├─────────────────┬───────────────────────┬──────────────────────┤
│   CLI (cli.py)  │  Gateway (gateway/)   │  Batch (batch_runner.py) │
└────────┬────────┴───────────┬───────────┴──────────┬───────────┘
         │                    │                       │
         └────────────────────┼───────────────────────┘
                              │
         ┌────────────────────▼────────────────────┐
         │      AIAgent (run_agent.py)             │
         │  - 核心對話循環                          │
         │  - 工具調度                              │
         │  - 會話管理                              │
         └────────┬────────────────────┬───────────┘
                  │                    │
      ┌───────────▼──────────┐    ┌───▼─────────────────┐
      │  提示構建器           │    │  工具註冊表          │
      │  (agent/prompt_*)    │    │  (tools/registry.py)│
      └──────────────────────┘    └───┬─────────────────┘
                                      │
                            ┌─────────▼──────────┐
                            │   工具處理器        │
                            │  (tools/*.py)      │
                            └────────────────────┘
```

### 組件層次

| 層次 | 組件 | 職責 |
|------|------|------|
| **界面** | CLI、Gateway、Batch Runner | 用戶交互、消息路由 |
| **核心代理** | AIAgent、Prompt Builder | 對話循環、LLM 編排 |
| **工具系統** | Registry、Handlers、Toolsets | 工具發現、調度、執行 |
| **持久化** | SessionDB、Memory | 會話存儲、全文搜索 |
| **後端** | Terminal environments | Local、Docker、SSH、Modal、Daytona 執行 |

---

## 核心組件

### AIAgent 類 (`run_agent.py`)

所有代理對話的中央編排器。

**主要方法：**

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
        """使用配置初始化代理。"""

    def chat(self, message: str) -> str:
        """簡單同步接口 - 返回最終響應字符串。"""

    def run_conversation(
        self,
        user_message: str,
        system_message: str = None,
        conversation_history: list = None,
        task_id: str = None
    ) -> dict:
        """
        完整對話接口 - 返回包含以下內容的字典：
        - final_response: str
        - messages: list
        - tool_calls: list
        """
```

**使用示例：**

```python
from run_agent import AIAgent

# 初始化代理
agent = AIAgent(
    model="anthropic/claude-sonnet-4.5",
    enabled_toolsets=["terminal", "file", "web"],
    platform="cli"
)

# 簡單聊天
response = agent.chat("列出當前目錄中的文件")
print(response)

# 帶歷史記錄的完整對話
result = agent.run_conversation(
    user_message="分析這個錯誤",
    conversation_history=previous_messages
)
print(result['final_response'])
```

### 提示構建器 (`agent/prompt_builder.py`)

從多個來源組裝系統提示。

**主要函數：**

```python
def build_system_prompt(
    agent,
    include_memory: bool = True,
    include_skills: bool = True,
    include_context_files: bool = True
) -> str:
    """從所有來源構建完整的系統提示。"""

def build_skills_system_prompt(
    skills_dir: str,
    available_tools: set,
    available_toolsets: set
) -> str:
    """構建具有條件激活的技能部分。"""
```

**使用模式：**

提示構建器自動：
1. 加載基本身份/個性
2. 注入啟用的工具和工具集
3. 根據工具可用性添加活動技能
4. 包含記憶和上下文文件
5. 格式化所有內容以實現最佳提示緩存

### 工具註冊表 (`tools/registry.py`)

系統中所有工具的中央註冊表。

**主要方法：**

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
        """使用其架構和處理器註冊工具。"""

    def get_tool_schemas(
        self,
        enabled_toolsets: list,
        disabled_toolsets: list = None
    ) -> list:
        """獲取啟用工具集中所有可用工具的架構。"""

    def dispatch(
        self,
        name: str,
        args: dict,
        **kwargs
    ) -> str:
        """按名稱執行工具處理器。"""
```

**工具中的使用：**

```python
# tools/example_tool.py
from tools.registry import registry
import json

def my_tool(param: str, task_id: str = None) -> str:
    """工具實現。"""
    result = {"status": "success", "data": param}
    return json.dumps(result)

SCHEMA = {
    "type": "function",
    "function": {
        "name": "my_tool",
        "description": "此工具的作用",
        "parameters": {
            "type": "object",
            "properties": {
                "param": {"type": "string", "description": "參數描述"}
            },
            "required": ["param"]
        }
    }
}

# 在導入時自動註冊
registry.register(
    name="my_tool",
    toolset="example",
    schema=SCHEMA,
    handler=lambda args, **kw: my_tool(**args, **kw)
)
```

### 會話數據庫 (`hermes_state.py`)

基於 SQLite 的會話持久化與全文搜索。

**主要方法：**

```python
class SessionDB:
    def save_session(
        self,
        session_id: str,
        messages: list,
        metadata: dict = None
    ):
        """將對話保存到數據庫。"""

    def load_session(self, session_id: str) -> list:
        """加載對話歷史。"""

    def search_sessions(
        self,
        query: str,
        limit: int = 10
    ) -> list:
        """跨所有會話進行全文搜索。"""

    def get_session_title(self, session_id: str) -> str:
        """獲取 AI 生成的會話標題。"""
```

**使用示例：**

```python
from hermes_state import SessionDB

db = SessionDB()

# 保存會話
db.save_session(
    session_id="cli_20260328_1234",
    messages=[
        {"role": "user", "content": "你好"},
        {"role": "assistant", "content": "嗨！"}
    ],
    metadata={"platform": "cli", "model": "claude-sonnet-4.5"}
)

# 搜索對話
results = db.search_sessions("錯誤處理")
for session_id, snippet in results:
    print(f"{session_id}: {snippet}")
```

---

## Agent 執行流程

### 完整對話循環

```
1. 用戶輸入
   ↓
2. 加載會話歷史（如果存在）
   ↓
3. 構建系統提示
   ├─ 加載身份/個性
   ├─ 注入啟用的工具
   ├─ 添加活動技能
   ├─ 包含記憶
   └─ 添加上下文文件
   ↓
4. 準備消息
   ├─ 系統提示（臨時）
   ├─ 對話歷史
   └─ 新用戶消息
   ↓
5. 調用 LLM API
   ├─ model: 選定的模型
   ├─ messages: 準備的消息
   ├─ tools: 啟用的工具架構
   └─ reasoning: 如果支持
   ↓
6. 處理響應
   ├─ 有 tool_calls？
   │  ├─ 是 → 執行每個工具
   │  │        將結果添加到消息
   │  │        返回到步驟 5
   │  └─ 否 → 繼續
   ↓
7. 提取最終響應
   ↓
8. 保存到數據庫
   ├─ 更新會話
   └─ 生成標題（如果是新的）
   ↓
9. 返回給用戶
```

### 工具執行流程

```python
# 工具執行的偽代碼
def execute_tool_call(tool_call):
    # 1. 提取工具名稱和參數
    name = tool_call.function.name
    args = json.loads(tool_call.function.arguments)

    # 2. 檢查工具是否可用
    if name not in registry.tools:
        return error_response("工具未找到")

    # 3. 驗證要求
    if not registry.check_requirements(name):
        return error_response("不符合要求")

    # 4. 調度到處理器
    try:
        result = registry.dispatch(name, args, task_id=task_id)
        return result
    except Exception as e:
        return error_response(str(e))
```

### 上下文壓縮流程

當接近令牌限制時：

```
1. 檢測令牌限制即將到來
   ↓
2. 觸發壓縮
   ├─ 保留系統提示
   ├─ 保留最近 N 條消息
   └─ 總結較舊的消息
   ↓
3. 調用輔助 LLM
   └─ 生成簡潔摘要
   ↓
4. 注入摘要
   └─ 用摘要消息替換舊消息
   ↓
5. 繼續對話
```

---

## 工具系統

### 工具架構

工具是自包含的模塊：
1. 定義其架構（OpenAI 函數格式）
2. 實現處理器函數
3. 在導入時自我註冊
4. 聲明要求和依賴關係

### 工具文件結構

```python
# tools/example_tool.py

"""example_tool — 此工具的簡要描述。"""

import json
import os
from tools.registry import registry

# 1. 處理器實現
def example_tool(param1: str, param2: int = 10, task_id: str = None) -> str:
    """
    工具處理器函數。

    參數：
        param1: 第一個參數
        param2: 可選的第二個參數
        task_id: 可選的任務 ID 用於上下文

    返回：
        帶結果的 JSON 字符串
    """
    result = {
        "success": True,
        "param1": param1,
        "param2": param2
    }
    return json.dumps(result)

# 2. 架構定義
EXAMPLE_TOOL_SCHEMA = {
    "type": "function",
    "function": {
        "name": "example_tool",
        "description": "此工具的作用以及何時使用它。",
        "parameters": {
            "type": "object",
            "properties": {
                "param1": {
                    "type": "string",
                    "description": "param1 的描述"
                },
                "param2": {
                    "type": "integer",
                    "description": "param2 的描述",
                    "default": 10
                }
            },
            "required": ["param1"]
        }
    }
}

# 3. 要求檢查
def _check_requirements() -> bool:
    """檢查工具依賴項是否可用。"""
    return bool(os.getenv("EXAMPLE_API_KEY"))

# 4. 自我註冊
registry.register(
    name="example_tool",
    toolset="example",  # 工具集組
    schema=EXAMPLE_TOOL_SCHEMA,
    handler=lambda args, **kw: example_tool(**args, **kw),
    check_fn=_check_requirements,
    requires_env=["EXAMPLE_API_KEY"]
)
```

### 工具集系統

工具組織成工具集（`toolsets.py`）：

```python
# 核心工具 - 始終可用
_HERMES_CORE_TOOLS = [
    "read_file", "write_file", "search_code",
    "list_files", "terminal"
]

# 工具集定義
TOOLSETS = {
    "file": ["read_file", "write_file", "search_code", "list_files"],
    "terminal": ["terminal"],
    "web": ["web_search", "web_extract"],
    "browser": ["browser_navigate", "browser_screenshot"],
    "code": ["execute_code"],
    "delegate": ["delegate_task"],
}

# 平台預設
PLATFORM_PRESETS = {
    "cli": ["file", "terminal", "web", "code", "delegate"],
    "telegram": ["file", "web", "code"],
    "discord": ["file", "web", "code"],
}
```

### 添加新工具

**步驟 1：創建工具文件**

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
        "description": "將文本轉換為大寫",
        "parameters": {
            "type": "object",
            "properties": {
                "input_text": {"type": "string", "description": "要轉換的文本"}
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

**步驟 2：添加導入到 `model_tools.py`**

```python
_modules = [
    # ... 現有模塊 ...
    "tools.my_new_tool",
]
```

**步驟 3：添加到 `toolsets.py` 中的工具集**

```python
TOOLSETS = {
    # ... 現有工具集 ...
    "text": ["my_new_tool"],
}
```

---

## 技能系統

技能是指導代理完成特定任務的指令集。

### 技能結構

```
skills/
├── category/
│   └── skill-name/
│       ├── SKILL.md          # 主要指令（必需）
│       ├── scripts/          # 輔助腳本（可選）
│       │   └── helper.py
│       └── references/       # 參考材料（可選）
│           └── docs.pdf
```

### SKILL.md 格式

```markdown
---
name: my-skill
description: 技能搜索的簡要描述
version: 1.0.0
author: Your Name
license: MIT
platforms: [macos, linux]  # 可選：限制到特定平台
required_environment_variables:
  - name: API_KEY
    prompt: 您的 API 密鑰
    help: 從 https://example.com 獲取
    required_for: 完整功能
metadata:
  hermes:
    tags: [類別, 關鍵詞]
    related_skills: [other-skill]
    fallback_for_toolsets: [web]    # 僅在 web 工具集不可用時顯示
    requires_toolsets: [terminal]   # 僅在 terminal 可用時顯示
---

# 技能標題

## 何時使用
描述代理何時應加載此技能。

## 快速參考
| 命令 | 描述 |
|------|------|
| command1 | 它的作用 |

## 程序
1. 步驟一
2. 步驟二
3. 步驟三

## 陷阱
- 已知問題 1
- 已知問題 2

## 驗證
如何確認任務成功。
```

### 條件技能激活

技能可以根據工具可用性顯示/隱藏：

```yaml
metadata:
  hermes:
    # 僅在 web 工具集不可用時顯示（後備）
    fallback_for_toolsets: [web]

    # 僅在 terminal 工具集可用時顯示（依賴）
    requires_toolsets: [terminal]

    # 僅在特定工具不可用時顯示
    fallback_for_tools: [web_search]

    # 僅在特定工具可用時顯示
    requires_tools: [browser_navigate]
```

---

## CLI 架構

### CLI 組件

```
cli.py (HermesCLI 類)
├─ Banner 顯示（Rich 面板）
├─ 輸入處理（prompt_toolkit）
│  ├─ 多行編輯
│  ├─ 自動完成
│  └─ 歷史記錄
├─ 斜槓命令調度
├─ 加載器/進度（KawaiiSpinner）
└─ 響應格式化
```

### 主要 CLI 方法

```python
class HermesCLI:
    def __init__(self, config: dict):
        """使用配置初始化 CLI。"""

    def run(self):
        """主 CLI 循環。"""

    def process_command(self, command: str) -> bool:
        """
        處理斜槓命令。
        如果命令已處理返回 True，否則返回 False。
        """

    def handle_user_input(self, user_input: str):
        """處理用戶消息並獲取代理響應。"""

    def display_response(self, response: str):
        """格式化並顯示代理響應。"""
```

### 斜槓命令系統

命令在 `hermes_cli/commands.py` 中集中定義：

```python
from dataclasses import dataclass

@dataclass
class CommandDef:
    name: str              # 規範名稱（無斜槓）
    description: str       # 幫助文本
    category: str          # "Session"、"Configuration" 等
    aliases: tuple = ()    # 替代名稱
    args_hint: str = ""    # 參數佔位符
    cli_only: bool = False # 僅在 CLI 中
    gateway_only: bool = False  # 僅在 gateway 中

# 中央註冊表
COMMAND_REGISTRY = [
    CommandDef("new", "開始新對話", "Session",
               aliases=("reset",)),
    CommandDef("model", "更改模型", "Configuration",
               args_hint="[provider:model]"),
    # ... 更多命令
]
```

### 添加 CLI 命令

**步驟 1：添加到註冊表**

```python
# hermes_cli/commands.py
COMMAND_REGISTRY = [
    # ... 現有命令 ...
    CommandDef("mycommand", "做某事", "Session",
               aliases=("mc",), args_hint="[optional-arg]"),
]
```

**步驟 2：添加處理器**

```python
# cli.py - 在 HermesCLI.process_command() 中
def process_command(self, cmd_original: str) -> bool:
    canonical = resolve_command(cmd_without_slash)

    # ... 現有處理器 ...

    elif canonical == "mycommand":
        self._handle_mycommand(cmd_original)
        return True

    return False

def _handle_mycommand(self, cmd: str):
    """處理 /mycommand。"""
    # 在這裡實現
    pass
```

---

## Gateway 架構

### Gateway 系統

```
gateway/run.py (GatewayRunner)
├─ 平台生命週期
│  ├─ Telegram
│  ├─ Discord
│  ├─ Slack
│  ├─ WhatsApp
│  └─ Signal
├─ 消息路由
├─ 會話管理
└─ Cron 集成
```

### 平台適配器模式

```python
# gateway/platforms/example.py
class ExamplePlatform:
    def __init__(self, config: dict, agent_factory):
        """使用配置初始化平台。"""

    async def start(self):
        """啟動平台監聽器。"""

    async def handle_message(self, message):
        """處理傳入消息。"""
        # 1. 提取用戶信息和文本
        # 2. 加載或創建會話
        # 3. 調用代理
        # 4. 發送響應

    async def send_message(self, user_id: str, text: str):
        """向用戶發送消息。"""
```

---

## 方法使用指南

### 常見使用模式

#### 1. 簡單代理聊天

```python
from run_agent import AIAgent

agent = AIAgent(model="anthropic/claude-sonnet-4.5")
response = agent.chat("你好，你好嗎？")
print(response)
```

#### 2. 帶特定工具集的代理

```python
agent = AIAgent(
    model="anthropic/claude-sonnet-4.5",
    enabled_toolsets=["file", "terminal", "web"],
    disabled_toolsets=["browser"]  # 明確禁用
)

response = agent.chat("在網上搜索 Python 教程")
```

#### 3. 會話管理

```python
from hermes_state import SessionDB

db = SessionDB()
session_id = "my_session_123"

# 加載現有會話
history = db.load_session(session_id)

# 運行帶歷史記錄的對話
agent = AIAgent(session_id=session_id)
result = agent.run_conversation(
    user_message="繼續上一個任務",
    conversation_history=history
)

# 保存更新的會話
db.save_session(session_id, result['messages'])
```

#### 4. 工具註冊

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
            "description": "轉換為大寫",
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

---

## 擴展模式

### 模式 1：添加新平台

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
        """啟動平台監聽器。"""
        # 初始化平台 SDK
        # 設置事件處理器
        pass

    async def handle_message(self, event: Any):
        """處理傳入消息。"""
        user_id = event.user_id
        text = event.text

        # 獲取或創建會話
        if user_id not in self.sessions:
            self.sessions[user_id] = {
                "session_id": f"myplatform_{user_id}",
                "history": []
            }

        session = self.sessions[user_id]

        # 獲取代理
        agent = self.agent_factory(session_id=session["session_id"])

        # 處理消息
        result = agent.run_conversation(
            user_message=text,
            conversation_history=session["history"]
        )

        # 更新歷史
        session["history"] = result["messages"]

        # 發送響應
        await self.send_message(user_id, result["final_response"])

    async def send_message(self, user_id: str, text: str):
        """通過平台發送消息。"""
        # 使用平台 SDK 發送
        pass
```

### 模式 2：帶狀態的自定義工具

```python
# tools/stateful_tool.py
from tools.registry import registry
import json

# 模塊級狀態（實際上是會話隔離的）
_cache = {}

def stateful_tool(action: str, key: str = None, value: str = None) -> str:
    """在調用之間維護狀態的工具。"""
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

### 模式 3：帶外部 API 的工具

```python
# tools/api_tool.py
from tools.registry import registry
import json
import requests
import os

def api_tool(query: str) -> str:
    """調用外部 API。"""
    api_key = os.getenv("MY_API_KEY")
    if not api_key:
        return json.dumps({"error": "API 密鑰未配置"})

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

---

## 最佳實踐

### 1. 提示緩存

**應該做：**
- 在回合之間保持系統提示穩定
- 在會話開始時加載記憶和技能一次
- 對動態內容使用臨時注入

**不應該做：**
- 在對話中修改過去的消息
- 在會話中更改工具集
- 不必要地重新加載技能或記憶

### 2. 工具設計

**應該做：**
- 從處理器返回 JSON 字符串
- 在處理前驗證輸入
- 優雅地處理錯誤
- 記錄何時使用工具

**不應該做：**
- 直接返回 Python 對象
- 不捕獲就拋出異常
- 進行沒有超時的阻塞調用
- 在描述中引用其他工具集的工具

### 3. 會話管理

**應該做：**
- 使用唯一、穩定的會話 ID
- 每回合後保存會話
- 在繼續對話前加載歷史

**不應該做：**
- 跨用戶共享會話
- 任意修改會話歷史
- 在會話元數據中存儲敏感數據

---

## 疑難解答

### 常見問題

**問題：工具未出現在代理中**
- 檢查工具集已啟用
- 驗證工具註冊成功
- 檢查要求函數返回 True
- 確保設置環境變量

**問題：會話未持久化**
- 檢查 `~/.hermes/state.db` 存在
- 驗證寫入權限
- 檢查 session_id 是否穩定

**問題：提示緩存不工作**
- 確保模型支持緩存（Anthropic 模型）
- 檢查系統提示是否穩定
- 驗證未修改過去的消息

**問題：技能未加載**
- 檢查 SKILL.md 格式和前置內容
- 驗證平台兼容性
- 檢查條件激活規則
- 確保技能目錄在技能路徑中

---

## 其他資源

- **[README.md](README.md)** - 項目概覽和快速開始
- **[CONTRIBUTING.md](CONTRIBUTING.md)** - 詳細貢獻指南
- **[AGENTS.md](AGENTS.md)** - AI 助手的開發指南
- **[文檔網站](https://hermes-agent.nousresearch.com/docs/)** - 完整用戶指南
- **[Discord](https://discord.gg/NousResearch)** - 社區支持

---

## 許可證

MIT — 查看 [LICENSE](LICENSE)。
