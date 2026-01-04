# Project Context

## Purpose

实现一个基于 Python 的 MCP (Model Context Protocol) 服务器，让 LLM 能够通过结构化工具调用与 iOS 模拟器交互。通过官方 Python MCP SDK 和 Facebook 的 fb-idb gRPC SDK，提供高性能、类型安全的 iOS 模拟器自动化能力。

**核心目标**：
- 使用官方 FastMCP API 实现简洁、类型安全的 MCP 工具
- 通过 fb-idb gRPC SDK 实现零进程创建开销，性能提升 10-30 倍
- 支持 Streamable HTTP 传输，适应生产环境部署
- 使用 uvx 实现零配置快速启动
- 提供 UI 交互、App 管理、系统控制等核心 iOS 自动化功能

## Requirements

### Functional Requirements

#### Core Features
- [ ] **Session Management**
  - Create/select simulator session (UDID management)

- [ ] **UI Interactions**
  - Tap at screen coordinates
  - Swipe/drag gestures
  - Element interaction (by ID/xPath if needed)

- [ ] **App Management**
  - Install/uninstall iOS apps (.ipa files)
  - Launch/terminate apps by bundle ID

- [ ] **System Interactions**
  - Set device location
  - Control debug HUD
  - Send push notifications

- [ ] **Data Access**
  - Retrieve device contacts/photos
  - Get battery/metadata info

- [ ] **Media & Debug**
  - Capture screenshots
  - Record screen video
  - Fetch crash/console logs

### Non-Functional Requirements
- **延迟**: < 5ms (vs exec 方式 20-50ms)
- **吞吐量**: > 200 req/s
- **部署**: 支持 uvx 零配置启动
- **官方 SDK**: 使用官方 FastMCP API + fb-idb SDK

### Implementation Guidance
- 使用 `@mcp.tool()` 装饰器注册工具（带类型注解和文档字符串）
- LLM 直接调用工具，不需要自然语言解析器
- 返回结构化数据（Pydantic 模型或 dict）
- 使用 `stateless_http=True` 实现无状态服务器
- 所有 I/O 操作使用 async/await

## Tech Stack

### 核心技术栈
- **语言**: Python 3.10+
- **MCP SDK**: `mcp` (官方 Python SDK, v1.25.0+)
  - FastMCP 高级 API
  - Streamable HTTP 传输
  - 结构化工具输出 (Pydantic 验证)
- **iOS 自动化**: `fb-idb` (Facebook 官方 SDK)
  - gRPC 客户端 (GrpcClient)
  - 长连接通信 (零 exec 开销)
  - 结构化返回数据 (protobuf)
- **包管理**: `uv`
   - 快速依赖解析和安装
   - 虚拟环境自动化
   - `uvx` 原生支持

### 辅助库
- **验证**: `pydantic` (类型验证和序列化)
- **异步**: `asyncio` (Python 原生)
- **HTTP 服务器**: 内置 ASGI (FastMCP 集成)

### 开发工具
- **测试**: `pytest` + `pytest-asyncio`
- **代码检查**: `ruff` (linting), `mypy` (类型检查)
- **文档**: `mkdocs` (可选)

## Project Conventions

### Code Style

#### 命名约定
- **模块/文件**: `snake_case` (e.g., `ios_simulator_mcp.py`, `idb_manager.py`)
- **函数**: `snake_case` (e.g., `process_instruction()`)
- **类**: `PascalCase` (e.g., `IDBManager`, `NLParser`)
- **常量**: `UPPER_SNAKE_CASE` (e.g., `DEFAULT_UDID`)
- **类型注解**: 强制使用 (e.g., `async def tap(x: int, y: int) -> str`)

#### 导入顺序
```python
# 1. 标准库
import asyncio
from dataclasses import dataclass

# 2. 第三方库
from mcp.server.fastmcp import FastMCP, Context
from idb.client.grpc import GrpcClient

# 3. 本地模块
from .idb_manager import IDBManager
from .types import SessionContext
```

#### 文档字符串
使用 Google 风格 (兼容 Sphinx/Napoleon):
```python
async def tap(x: int, y: int) -> str:
    """Tap at screen coordinates.

    Args:
        x: X coordinate.
        y: Y coordinate.

    Returns:
        Status message indicating the tap was executed.

    Example:
        >>> await tap(100, 200)
        'Tapped at (100, 200)'
    """
```

#### 异步约定
- 所有 I/O 操作必须使用 `async/await`
- 使用类型注解 `AsyncIterator`, `Optional`, `dict[str, Any]`
- 超时控制使用 `asyncio.wait_for()`

### Architecture Patterns

#### 分层架构
```
┌────────────────────────────────────┐
│  MCP Layer (mcp.server.fastmcp)   │  ← FastMCP, @mcp.tool()
│  - Tool registration               │
│  - Request/response handling       │
│  - Streamable HTTP transport       │
├────────────────────────────────────┤
│  Business Logic Layer              │  ← 工具函数
│  - UI interactions (tap, swipe)    │
│  - App management (install, launch)│
│  - System control (location, logs) │
│  - Session management              │
├────────────────────────────────────┤
│  IDB Abstraction Layer             │  ← IDBManager (包装 fb-idb)
│  - gRPC client lifecycle           │
│  - Session ↔ UDID mapping          │
│  - Error handling                  │
├────────────────────────────────────┤
│  Transport Layer                   │  ← fb-idb (GrpcClient)
│  - gRPC over TCP                   │
│  - Structured responses            │
└────────────────────────────────────┘
```

#### 设计模式
- **装饰器模式**: `@mcp.tool()` 注册工具
- **上下文管理器**: `@asynccontextmanager` 管理 IDB 连接生命周期
- **依赖注入**: `Context[ServerSession, SessionContext]` 注入会话资源
- **适配器模式**: `IDBManager` 适配 fb-idb API 到业务逻辑

#### 关键原则
1. **优先使用官方 SDK**: 避免重新造轮子 (FastMCP, fb-idb)
2. **类型安全**: 所有函数必须有类型注解和返回类型
3. **异步优先**: 所有 I/O 使用 async/await
4. **最小化状态**: 使用 `stateless_http=True` 实现无状态服务器
5. **错误处理**: 使用 try-except 捕获 IDB 错误,返回友好消息

### Development Methodology

#### BDD + TDD Workflow
本项目采用 **BDD (Behavior-Driven Development)** + **TDD (Test-Driven Development)** 结合的开发方法：

**核心原则**：
1. **Behavior-Driven Development (BDD)**：
   - 使用 Gherkin 语法描述业务场景 (Given-When-Then)
   - 从用户行为视角定义需求和验收标准
   - 使用自然语言 (中文) 编写 Feature 文件，便于业务理解
   - Feature 文件作为开发指南和文档

2. **Test-Driven Development (TDD)**：
   - 遵循 RGR (Red-Green-Refactor) 循环
   - 先写测试，再写实现 (测试优先)
   - 每个功能都有对应的单元/集成测试
   - 保持高测试覆盖率 (>80%)

**RGR 工作流**：
```
Red (红)    → 编写失败的测试
Green (绿)  → 编写最小实现让测试通过
Refactor    → 重构代码，保持测试通过
```

**BDD + TDD 结合流程**：
```
1. 编写 Feature (BDD) → 明确业务场景和验收标准
2. 编写测试 (TDD-Red) → 编写失败的单元测试
3. 实现功能 (TDD-Green) → 编写最小实现让测试通过
4. 验证场景 (BDD) → 运行 E2E 测试验证业务场景
5. 重构优化 (Refactor) → 优化代码，保持测试通过
```

**工具支持**：
- **BDD**: `pytest-bdd` (Gherkin 语法支持)
- **TDD**: `pytest` (单元/集成测试)
- **测试报告**: `pytest-cov` (覆盖率), `pytest-html` (HTML 报告)

**示例结构**：
```
features/
  ├── ui_automation.feature
  ├── app_management.feature
  └── system_control.feature
tests/
  ├── unit/
  │   ├── test_idb_manager.py
  │   └── test_tools.py
  └── integration/
      └── test_server.py
```

**Feature 文件示例** (`features/ui_automation.feature`):
```gherkin
Feature: UI 自动化交互
  作为 LLM 助手
  我想要与 iOS 模拟器进行 UI 交互
  以便自动化执行测试和验证功能

  Scenario: 点击屏幕坐标
    Given 模拟器已启动 (UDID: "BOOTED")
    When 在屏幕位置 (100, 200) 点击
    Then 点击操作应执行成功
    And 返回状态消息 "Tapped at (100, 200)"
```

### Testing Strategy

#### 测试层级
1. **单元测试** (`tests/unit/`):
    - 测试 IDBManager 方法 (tap, swipe, install 等)
    - 测试工具参数验证
    - Mock IDB 客户端 (pytest-mock)

2. **集成测试** (`tests/integration/`):
    - 测试 MCP 服务器启动
    - 测试工具调用流程
    - 测试 gRPC 连接管理

3. **端到端测试** (`tests/e2e/`):
    - 需要真实的 iOS 模拟器 (可选)
    - 测试真实 IDB 命令执行

#### 测试库
```bash
pytest --cov=src tests/
```

#### CI/CD
- 每次提交运行单元测试
- PR 自动运行完整测试套件
- 使用 GitHub Actions

### Git Workflow

#### 分支策略
- `main`: 稳定发布版本
- `develop`: 开发集成分支
- `feature/*`: 功能分支 (e.g., `feature/nl-parser`)
- `fix/*`: 修复分支 (e.g., `fix/gRPC-connection-pool`)

#### 提交约定 (Conventional Commits)
```
feat: add tap and swipe tools for UI automation
fix: resolve gRPC connection timeout issue
docs: update README with uvx usage
refactor: simplify IDB manager with context manager
test: add unit tests for screenshot tool
```

#### PR 流程
1. 分支从 `develop` 创建
2. 通过至少 1 个 reviewer 审批
3. 所有 tests 通过
4. 合并回 `develop`

## Domain Context

### MCP (Model Context Protocol)
- 定义: 标准化 LLM 上下文提供协议
- 核心 Primitives: Tools (工具), Resources (资源), Prompts (模板)
- 传输: stdio, SSE, Streamable HTTP (本项目使用)
- 工具调用: JSON-RPC 2.0 格式

### IDB (iOS Debug Bridge)
- 定义: Facebook 开源的 iOS 自动化工具
- 架构: `idb-companion` (macOS 进程) + `fb-idb` (Python 客户端)
- 通信: gRPC over TCP (替代 xcrun/stdio)
- 能力: 模拟器管理、app 安装、UI 交互、调试、日志收集

### iOS Simulator Automation
- IDB 能力 (通过 fb-idb SDK):
  - 启动应用: `client.launch(bundle_id="com.apple.mobilesafari")`
  - 点击屏幕: `client.ui.tap(udid=udid, x=100, y=200)`
  - 截屏: `client.screenshot.save(udid=udid)`
  - 获取日志: `client.log(udid=udid)`
- 会话管理: `udid` (唯一设备标识)
- 关键架构点:
  - 使用 fb-idb GrpcClient 直接调用 SDK 方法 (不执行命令)
  - gRPC 长连接通信 (零进程创建开销)
  - 返回结构化数据 (protobuf 对象，非文本解析)
- MCP 工具映射:
  - `@mcp.tool()` 装饰器直接暴露结构化 API
  - LLM 通过工具参数直接调用 (无需自然语言解析)

## Important Constraints

### 技术约束
1. **平台依赖**: 必须在 macOS 上运行 (需要 Xcode + iOS 模拟器)
2. **Python 版本**: ≥ 3.10 (类型注解支持)
3. **IDB 兼容性**: 需验证 fb-idb v1.1.8+ 与 macOS 15/Xcode 16 兼容性
4. **无状态要求**: 生产环境必须使用 `stateless_http=True`

### 性能约束
- 目标延迟: < 5ms (vs 当前 20-50ms)
- 吞吐量: > 200 req/s (vs 当前 ~20 req/s)
- 内存: < 100MB (单 gRPC 连接)

### 安全约束
1. **认证** (可选): 实现 OAuth 2.1 Resource Server (MCP spec)
2. **沙箱隔离**: IDB 客户端与服务器进程分离
3. **输入验证**: 所有用户输入通过 Pydantic 验证

## External Dependencies

### 必需依赖
| 依赖 | 版本 | 用途 |
|------|------|------|
| `mcp` | ≥1.25.0 | 官方 Python MCP SDK (FastMCP API) |
| `fb-idb` | ≥1.1.8 | Facebook IDB gRPC 客户端 |
| `pydantic` | ≥2.5.0 | 类型验证和序列化 (mcp 自带依赖) |

### 系统依赖
| 依赖 | 说明 |
|------|------|
| macOS | 运行 iOS 模拟器的唯一平台 |
| Xcode | 提供 iOS SDK 和 模拟器 |
| idb-companion | `brew install facebook/fb/idb-companion` (IDB 服务端) |
| fb-idb | `uv pip install fb-idb` (Python gRPC 客户端) |
| Python 3.10+ | 运行时环境 |
| uv | 包管理器 (推荐) |

### 可选依赖
| 依赖 | 用途 |
|------|------|
| `uvicorn` | 独立 ASGI 服务器 (可选,FastMCP 内置) |
| `pytest` | 测试框架 |
| `ruff` | 代码检查 |
| `mypy` | 类型检查 |

### API/服务依赖
- **IDB Companion**: 需要在 macOS 上本地运行 (`/usr/local/bin/idb_companion`)
- **iOS Simulator**: 通过 Xcode 安装,由 idb-companion 管理
- **无外部 API**: 纯本地运行,无需网络服务

## Development Quick Start

```bash
# 1. 安装系统依赖
brew install idb-companion

# 2. 使用 uvx 快速启动 (无需本地安装)
uvx --from mcp,fb-idb python ios_simulator_mcp.py

# 3. 或使用本地开发环境
uv init mcp-server-ios-simulator
cd mcp-server-ios-simulator
uv add mcp fb-idb pydantic
uv run python ios_simulator_mcp.py

# 4. 使用 MCP Inspector 测试
uv run mcp dev ios_simulator_mcp.py
```
