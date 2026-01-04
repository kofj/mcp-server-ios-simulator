# Design: 会话管理

## Context
MCP 服务器需要连接和管理多个 iOS 模拟器实例。fb-idb SDK 提供 GrpcClient 通过 gRPC 协议与 idb-companion 通信，但没有内置的会话管理概念。我们需要在 IDBManager 上层构建会话抽象，使得 MCP 工具可以操作特定的模拟器。

### 约束条件
- 必须使用 fb-idb GrpcClient（官方 SDK）
- 所有 I/O 操作使用 async/await
- 支持 streamable HTTP 传输（无状态服务器）
- 单 gRPC 连接复用（零开销）
- 性能目标：延迟 < 5ms

## Goals / Non-Goals

### Goals
- 提供 UDID 到会话的映射管理
- 支持管理多个模拟器（单一活动会话）
- 连接复用（单 gRPC 连接）
- 友好的错误处理和状态反馈
- 集成到 FastMCP Context（状态注入）

### Non-Goals
- 不实现持久化存储（会话仅存在内存中）
- 不实现会话过期和自动清理
- 不实现模拟器启动/停止（仅管理已存在的模拟器）

**注意**: 当模拟器不存在时，会话会自动失效。每次操作会验证 UDID 有效性，不存在时返回错误并建议重新创建会话。

## Decisions

### Decision 1: 使用 FastMCP Context 注入会话
**理由**: FastMCP 提供 `Context[ServerSession, SessionContext]` 依赖注入机制，可以安全地在 HTTP 请求间传递会话状态，无需全局变量。

**实现**:
```python
from typing import Optional
from pydantic import BaseModel
from mcp.server.fastmcp import FastMCP, Context

class SessionContext(BaseModel):
    active_udid: Optional[str] = None

mcp = FastMCP("ios-simulator")
mcp.register_context(ServerSession, SessionContext)

@mcp.tool()
async def create_session(udid: str, ctx: Context[ServerSession, SessionContext]) -> str:
    ctx.session_context.active_udid = udid
    return f"Created session for {udid}"
```

### Decision 2: IDBManager 使用线程安全单例模式管理 gRPC 连接
**理由**: fb-idb gRPC 连接是长连接，创建开销大。使用单例确保整个进程只有一个 GrpcClient 实例，通过 UDID 参数区分不同模拟器。

**实现**:
```python
import asyncio
from typing import ClassVar, Optional
from idb.client.grpc import GrpcClient
from idb.common.types import Target

class IDBManager:
    _instance: ClassVar[Optional["IDBManager"]] = None
    _lock: ClassVar[asyncio.Lock] = asyncio.Lock()

    def __new__(cls) -> "IDBManager":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance._client: Optional[GrpcClient] = None
        return cls._instance

    async def _get_client(self) -> GrpcClient:
        async with self._lock:
            if self._client is None:
                self._client = await GrpcClient().start()
        return self._client

    async def _get_context(self, udid: str):
        client = await self._get_client()
        target = Target(udid=udid)
        return await client.create_context(target=target)
```

### Decision 3: 使用 Pydantic 模型定义会话状态
**理由**: 类型安全和序列化支持，便于 HTTP 传输和验证。

**实现**:
```python
from pydantic import BaseModel

class SimulatorInfo(BaseModel):
    udid: str
    name: str
    state: str  # "booted", "shutdown"

class SessionContext(BaseModel):
    active_udid: Optional[str] = None
```

### Decision 4: 错误处理使用友好消息而非异常抛出
**理由**: MCP 工具通过 JSON-RPC 调用，异常会中断请求链。友好消息让 LLM 能够理解并重试。

**实现**:
```python
import logging
from idb.common.exceptions import TargetNotFoundError

logger = logging.getLogger(__name__)

try:
    simulators = await self._client.list_targets()
except TargetNotFoundError as e:
    return f"Simulator not found: {str(e)}"
except Exception as e:
    logger.error(f"Unexpected error listing simulators: {e}")
    return f"Failed to list simulators: {str(e)}. Ensure idb-companion is running: idb-companion"
```

## Risks / Trade-offs

### Trade-off 1: 内存存储 vs 持久化
**选择**: 内存存储
**风险**: 服务重启后会话丢失
**缓解**: 调用端需在服务重启后重建会话（符合无状态设计）

### Risk 2: gRPC 连接断开
**场景**: idb-companion 进程崩溃或网络问题
**缓解**:
- 实现自动重连逻辑
- 提供重新连接工具
- 错误消息中提示解决方案

## Migration Plan
首次实现，无需迁移。

## Open Questions
无
