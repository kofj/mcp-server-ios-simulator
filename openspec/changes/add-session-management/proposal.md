# Change: 添加会话管理功能

## Why
会话管理是 iOS 模拟器 MCP 服务器的核心基础。通过 UDID 管理模拟器会话，为后续 UI 交互、App 管理等所有功能提供必要的上下文支持。没有会话管理，其他功能无法识别或操作特定的模拟器实例。

## What Changes
- 添加可用模拟器列表功能
- 添加创建/选择模拟器会话功能
- 添加断开会话连接功能
- 实现会话状态管理（UDID 映射）
- 构建 IDBManager 封装层，抽象 fb-idb gRPC SDK

## Impact
- Affected specs: 新增 `session-management` capability
- Affected code:
  - 新建 `src/idb_manager.py` - IDB 封装层，管理 gRPC 连接和会话
  - 新建 `src/ios_simulator_mcp.py` - MCP 服务器入口，注册工具
- Dependencies: fb-idb gRPC SDK (GrpcClient)
- Breaking changes: 无
