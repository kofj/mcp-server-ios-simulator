## ADDED Requirements

### Requirement: 列出所有可用模拟器
系统 SHALL 提供列出所有可用 iOS 模拟器的功能，包括模拟器的 UDID、名称和状态。

#### Scenario: 成功列出模拟器列表
- **GIVEN** 系统已连接到 idb-companion
- **WHEN** LLM 调用 `list_simulators` 工具
- **THEN** 返回所有模拟器的 UDID、名称和状态
- **AND** 至少包含一个已启动（booted）状态的模拟器

#### Scenario: 无可用模拟器
- **GIVEN** 系统已连接到 idb-companion
- **WHEN** 系统中没有任何可用的 iOS 模拟器
- **THEN** 返回友好的提示消息 "No simulators available"
- **AND** 不抛出异常

### Requirement: 创建或选择模拟器会话
系统 SHALL 允许 LLM 创建新的模拟器会话或选择现有会话，通过指定 UDID 来操作特定的模拟器实例。

#### Scenario: 创建新会话成功
- **GIVEN** 系统中存在 UDID 为 "BOOTED-SIMULATOR" 的已启动模拟器
- **WHEN** LLM 调用 `create_session` 工具并传入 UDID "BOOTED-SIMULATOR"
- **THEN** 会话成功创建并关联到该 UDID
- **AND** 返回确认消息 "Session created for simulator: BOOTED-SIMULATOR"

#### Scenario: 模拟器未启动
- **GIVEN** 系统中存在 UDID 为 "SHUTDOWN-SIMULATOR" 的模拟器但状态为 shutdown
- **WHEN** LLM 调用 `create_session` 工具并传入 UDID "SHUTDOWN-SIMULATOR"
- **THEN** 返回错误消息 "Simulator SHUTDOWN-SIMULATOR is not booted"
- **AND** 会话创建失败

#### Scenario: 无效的 UDID
- **GIVEN** 系统中不存在 UDID 为 "INVALID-UDID" 的模拟器
- **WHEN** LLM 调用 `create_session` 工具并传入 UDID "INVALID-UDID"
- **THEN** 返回错误消息 "Simulator INVALID-UDID not found"
- **AND** 会话创建失败

### Requirement: 断开会话连接
系统 SHALL 允许 LLM 断开当前活动的模拟器会话，释放会话资源。

#### Scenario: 成功断开会话
- **GIVEN** 当前已激活 UDID 为 "ACTIVE-SIMULATOR" 的会话
- **WHEN** LLM 调用 `disconnect_session` 工具
- **THEN** 会话被成功断开
- **AND** 返回确认消息 "Session disconnected: ACTIVE-SIMULATOR"
- **AND** 会话状态变更为未激活

#### Scenario: 无活动会话
- **GIVEN** 当前没有激活的任何模拟器会话
- **WHEN** LLM 调用 `disconnect_session` 工具
- **THEN** 返回提示消息 "No active session to disconnect"
- **AND** 不抛出异常

### Requirement: 当前会话状态查询
系统 SHALL 允许 LLM 查询当前活动的会话信息，包括关联的模拟器 UDID 和名称。

#### Scenario: 成功获取活动会话信息
- **GIVEN** 当前已激活 UDID 为 "ACTIVE-SIMULATOR" 的会话
- **WHEN** LLM 调用 `get_session_info` 工具
- **THEN** 返回当前会话的 UDID 和模拟器名称
- **AND** 包含会话激活时间戳

#### Scenario: 无活动会话
- **GIVEN** 当前没有激活的任何模拟器会话
- **WHEN** LLM 调用 `get_session_info` 工具
- **THEN** 返回消息 "No active session"
- **AND** 不提供任何模拟器信息

#### Scenario: 模拟器失效后查询会话
- **GIVEN** 当前已激活 UDID 为 "DELETED-SIMULATOR" 的会话
- **AND** 该模拟器已被删除或不存在
- **WHEN** LLM 调用 `get_session_info` 工具
- **THEN** 返回错误消息 "Simulator DELETED-SIMULATOR no longer exists"
- **AND** 建议用户调用 `create_session` 重新创建会话
