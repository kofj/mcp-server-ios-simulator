# Tasks: 会话管理实现

## 1. 项目结构初始化
- [ ] 1.1 创建 `src/` 目录和空的 `__init__.py`
- [ ] 1.2 初始化 `requirements.txt` (mcp, fb-idb, pydantic, pytest, pytest-asyncio, pytest-bdd)
- [ ] 1.3 创建 `features/` 目录用于 BDD 测试

## 2. BDD: 编写 Feature 文件
- [ ] 2.1 创建 `features/session_management.feature`，使用 Gherkin 语法定义业务场景
- [ ] 2.2 验证 Feature 文件符合中文 GIVEN-WHEN-THEN 格式

## 3. TDD-Red: 编写单元测试
- [ ] 3.1 创建 `tests/unit/test_idb_manager.py`，编写 IDBManager 的测试用例
- [ ] 3.2 Mock fb-idb GrpcClient，避免真实依赖
- [ ] 3.3 创建 `tests/unit/test_types.py`，测试 Pydantic 模型
- [ ] 3.4 运行 `pytest tests/unit/` 确认所有测试失败（Red 阶段）

## 4. TDD-Green: 实现 IDBManager
- [ ] 4.1 创建 `src/types.py`，定义 SimulatorInfo 和 SessionContext Pydantic 模型
- [ ] 4.2 创建 `src/idb_manager.py`，实现 IDBManager 单例类
- [ ] 4.3 实现 `list_simulators()` 方法，调用 GrpcClient.list_targets()
- [ ] 4.4 实现 `create_session(udid)` 方法，设置 SessionContext.active_udid
- [ ] 4.5 实现 `disconnect_session()` 方法，清除 SessionContext
- [ ] 4.6 实现 `get_session_info()` 方法，返回当前会话状态
- [ ] 4.7 添加错误处理和友好消息
- [ ] 4.8 运行 `pytest tests/unit/` 确认所有测试通过（Green 阶段）

## 5. Refactor: 代码重构优化
- [ ] 5.1 检查代码重复，提取公共方法
- [ ] 5.2 优化类型注解，确保完整覆盖
- [ ] 5.3 验证异步操作正确性（async/await）
- [ ] 5.4 运行 `ruff check src/` 和 `mypy src/` 检查代码质量
- [ ] 5.5 运行 `pytest tests/unit/` 确保重构后测试依然通过

## 6. 集成测试：MCP 工具注册
- [ ] 6.1 创建 `src/ios_simulator_mcp.py`，初始化 FastMCP 服务器
- [ ] 6.2 注册 `list_simulators` 工具，使用 `@mcp.tool()` 装饰器
- [ ] 6.3 注册 `create_session` 工具，注入 `Context[ServerSession, SessionContext]`
- [ ] 6.4 注册 `disconnect_session` 工具
- [ ] 6.5 注册 `get_session_info` 工具
- [ ] 6.6 创建 `tests/integration/test_server.py`，测试 MCP 服务器启动

## 7. E2E 测试：BDD 场景验证
- [ ] 7.1 实现 `features/session_management.feature` 对应的步骤定义 (`features/steps/session_management_steps.py`)
- [ ] 7.2 运行 `pytest-bdd` 验证所有 BDD 场景通过
- [ ] 7.3（可选）使用真实 iOS 模拟器运行端到端测试

## 8. 文档和工具配置
- [ ] 8.1 更新 README.md，添加 uvx 快速启动说明
- [ ] 8.2 配置 `pytest.ini`（asyncio_mode, testpaths）
- [ ] 8.3 配置 `ruff.toml`（line-length, target-version）
- [ ] 8.4 配置 `mypy.ini`（python_version, strict mode）

## 9. 验收检查
- [ ] 9.1 运行 `pytest --cov=src tests/` 确保测试覆盖率 > 80%
- [ ] 9.2 运行 `ruff check src/` 无 lint 错误
- [ ] 9.3 运行 `mypy src/` 无类型错误
- [ ] 9.4 使用 `uv run mcp dev ios_simulator_mcp.py` 验证工具可被发现和调用
- [ ] 9.5 确认所有 `tasks.md` 条目已完成
