---
title: UnrealAIBridge 与 UnrealMCPServer 开发测试总结
date: 2026-07-10
tags:
  - Unreal
  - MCP
  - CodeBuddy
  - Lua
  - 工具链
  - 技术沉淀
---

# UnrealAIBridge 与 UnrealMCPServer 开发测试总结

## 1. 背景与目标

本次工作的目标是打通一条从 AI 工具到 Unreal 项目运行时 / 编辑器的控制链路：

```text
CodeBuddy / MCP Client
  -> UnrealMCPServer
  -> UnrealAIBridgeRuntime HTTP JSON-RPC
  -> runtime.* / editor.*
  -> Lua AICommandBridgeExecute / GEditor
```

核心诉求有两个：

1. **运行时可控**：游戏运行时、PIE、独立游戏、packaged runtime 都能够接收外部 AI 指令，并转发到 Lua。
2. **编辑器可控**：在 Unreal Editor 中，能够通过 `editor.*` 操作 GEditor、Actor、资产、PIE 等能力。

最初实现中，HTTP 入口放在 Editor 模块 `FUnrealAIBridgeModule` 中。这个设计在 PIE 下可用，但在真正游戏运行时不可用，因为 `Type = Editor` 的模块不会在独立游戏 / packaged runtime 中加载。

因此本次重构的核心是：**将 RPC 入口下沉到 Runtime 模块，Editor 模块只注册 editor.* 扩展能力。**

---

## 2. 最终架构

### 2.1 Unreal 插件分层

插件分为两个模块：

```text
UnrealAIBridgeRuntime
  Type = Runtime
  负责：
    - 启动本地 HTTP JSON-RPC 服务
    - 解析 /rpc 请求
    - namespace 分发
    - 内建 runtime.* -> Lua
    - 暴露 RegisterNamespaceHandler 给其它模块扩展

UnrealAIBridge
  Type = Editor
  负责：
    - 启动时向 Runtime 注册 editor namespace
    - 处理 editor.* 命令
    - 使用 GEditor / AssetRegistry / UnrealEd API
```

最终链路：

```text
外部 HTTP / MCP
  -> UnrealAIBridgeRuntime /rpc
  -> Dispatch(method)
     -> runtime.* -> UUnrealAIBridgeRuntimeLibrary::ExecuteAICommand
                 -> ULuaScriptSubsystem
                 -> Lua AICommandBridgeExecute(command, paramsJson)

     -> editor.*  -> Editor 模块注册的 handler
                 -> GEditor / AssetRegistry / PIE API
```

### 2.2 MCP 分层

MCP 不在 Unreal 插件内部，而是一个独立进程：

```text
CodeBuddy / MCP Client
  -> UnrealMCPServer（stdio MCP Server）
  -> http://127.0.0.1:18777/rpc
  -> UnrealAIBridgeRuntime
```

也就是说：

- `UnrealAIBridgeRuntime` 是 **Unreal 内部执行器**。
- `UnrealMCPServer` 是 **MCP 协议适配层**。
- CodeBuddy 通过 MCP 调工具，MCP Server 再转发到 Unreal HTTP RPC。

---

## 3. UnrealAIBridge 实现总结

### 3.1 Runtime 模块新增能力

`UnrealAIBridgeRuntime` 现在承担 RPC 主机职责：

- 监听：`http://127.0.0.1:18777/rpc`
- 请求格式：

```json
{
  "method": "runtime.ping",
  "params": {}
}
```

- 返回格式：

```json
{
  "ok": true,
  "result": {}
}
```

或：

```json
{
  "ok": false,
  "error": {
    "code": "...",
    "message": "..."
  }
}
```

Runtime 模块实现了：

1. `StartServer()` / `StopServer()`
2. `HandleRpcRequest()`
3. `ProcessRpcRequestBody()`
4. `Dispatch()`
5. `HandleRuntimeCommand()`
6. `RegisterNamespaceHandler()` / `UnregisterNamespaceHandler()`

### 3.2 Runtime 分发到 Lua

`runtime.*` 会剥掉 `runtime.` 前缀，只把命令名传给 Lua：

```text
runtime.ping -> AICommandBridgeExecute("ping", "{}")
runtime.app.info -> AICommandBridgeExecute("app.info", "{}")
```

C++ 调用点：

```cpp
UUnrealAIBridgeRuntimeLibrary::ExecuteAICommand(World, Command, ParamsJson)
```

Lua 侧全局入口：

```lua
_G.AICommandBridgeExecute = dispatch
```

Lua 内置命令包括：

| 命令 | 作用 |
|---|---|
| `ping` | 检查 Lua bridge 是否连通 |
| `echo` | 回显参数 |
| `list` | 列出已注册 Lua 命令 |
| `app.info` | 获取 App 运行状态 |

外部完整调用时是：

```text
runtime.ping
runtime.echo
runtime.list
runtime.app.info
```

### 3.3 Editor 模块能力

Editor 模块不再启动 HTTP Server，只在启动时注册：

```cpp
RuntimeModule.RegisterNamespaceHandler(
    TEXT("editor"),
    FUnrealAIBridgeNamespaceHandler::CreateRaw(this, &FUnrealAIBridgeModule::HandleEditorCommand));
```

当前支持的 `editor.*` 命令：

| 命令                   | 作用                    |
| -------------------- | --------------------- |
| `editor.ping`        | 检查 Editor bridge 是否连通 |
| `editor.info`        | 获取引擎版本、项目名、PIE 状态     |
| `editor.exec`        | 执行 Editor 控制台命令       |
| `editor.actors.list` | 列举当前编辑器世界 Actor       |
| `editor.assets.find` | 通过 AssetRegistry 查询资产 |
| `editor.level.save`  | 保存当前关卡                |
| `editor.play`        | 启动 PIE                |
| `editor.stop`        | 停止 PIE                |

在独立游戏 / packaged runtime 中，`editor.*` 不可用，返回 `unknown_namespace`，这是符合预期的行为。

---

## 4. UnrealMCPServer 实现总结

### 4.1 位置

独立项目路径：

```text
E:/AkStudio/UnrealMCPServer
```

已上传 GitHub：

```text
https://github.com/californiawz/UnrealMCPServer
```

### 4.2 技术选型

采用 **零依赖 Python stdio MCP Server**。

原因：

1. 不需要 `npm install` 或 `pip install`。
2. Windows 下容易运行。
3. CodeBuddy MCP 支持 `type = stdio`、`command`、`args`、`env`。
4. 该 MCP Server 只是协议适配层，不需要复杂框架。

### 4.3 暴露的 MCP Tools

#### 通用工具

| MCP Tool | 作用 |
|---|---|
| `unreal_rpc_call` | 调任意完整 Bridge method，如 `runtime.ping` / `editor.info` |
| `unreal_runtime_call` | 调 `runtime.*`，只传 command |
| `unreal_editor_call` | 调 `editor.*`，只传 command |

#### Runtime / Lua 工具

| MCP Tool | 内部调用 |
|---|---|
| `unreal_runtime_ping` | `runtime.ping` |
| `unreal_runtime_list_commands` | `runtime.list` |
| `unreal_runtime_app_info` | `runtime.app.info` |

#### Editor 工具

| MCP Tool | 内部调用 |
|---|---|
| `unreal_editor_info` | `editor.info` |
| `unreal_editor_exec` | `editor.exec` |
| `unreal_editor_list_actors` | `editor.actors.list` |
| `unreal_editor_find_assets` | `editor.assets.find` |
| `unreal_editor_save_level` | `editor.level.save` |
| `unreal_editor_play` | `editor.play` |
| `unreal_editor_stop` | `editor.stop` |

### 4.4 CodeBuddy 安装配置

项目级 MCP 配置文件：

```text
E:/AkStudio/.mcp.json
```

内容：

```json
{
  "mcpServers": {
    "unreal": {
      "type": "stdio",
      "command": "python",
      "args": [
        "e:/AkStudio/UnrealMCPServer/server.py"
      ],
      "env": {
        "UNREAL_BRIDGE_URL": "http://127.0.0.1:18777/rpc",
        "UNREAL_BRIDGE_TIMEOUT": "10"
      },
      "description": "Unreal MCP Server backed by UnrealAIBridgeRuntime"
    }
  }
}
```

CodeBuddy 加载后会出现名为 `unreal` 的 MCP Server。

---

## 5. 使用示例

### 5.1 检查 Runtime 是否连通

自然语言：

```text
用 unreal MCP 检查 Unreal 运行时是否在线
```

MCP tool：

```json
{
  "name": "unreal_runtime_ping",
  "arguments": {}
}
```

内部调用：

```text
runtime.ping
```

### 5.2 列出 Lua 命令

```json
{
  "name": "unreal_runtime_list_commands",
  "arguments": {}
}
```

内部调用：

```text
runtime.list
```

### 5.3 调 Lua echo

```json
{
  "name": "unreal_runtime_call",
  "arguments": {
    "command": "echo",
    "params": {
      "hello": "world"
    }
  }
}
```

内部调用：

```text
runtime.echo
```

### 5.4 查询 Editor 信息

```json
{
  "name": "unreal_editor_info",
  "arguments": {}
}
```

内部调用：

```text
editor.info
```

### 5.5 列出当前关卡 Actor

```json
{
  "name": "unreal_editor_list_actors",
  "arguments": {
    "limit": 20
  }
}
```

内部调用：

```text
editor.actors.list
```

### 5.6 查询 Blueprint 资产

```json
{
  "name": "unreal_editor_find_assets",
  "arguments": {
    "path": "/Game",
    "classFilter": "Blueprint",
    "limit": 20
  }
}
```

内部调用：

```text
editor.assets.find
```

### 5.7 启动和停止 PIE

```json
{
  "name": "unreal_editor_play",
  "arguments": {}
}
```

```json
{
  "name": "unreal_editor_stop",
  "arguments": {}
}
```

---

## 6. 测试总结

### 6.1 Unreal 插件编译测试

已执行并通过：

```powershell
& "e:/AkStudio/UnrealEngine/Engine/Build/BatchFiles/Build.bat" TheWildsEditor Win64 Development -Project="e:/AkStudio/TheWildsGame/TheWildsClient/TheWilds/TheWilds.uproject" -WaitMutex -NoHotReload
```

结果：

```text
Result: Succeeded
```

验证点：

- Runtime 模块可编译。
- Editor 模块可编译。
- Editor 模块成功依赖 Runtime 模块。
- `editor.*` 注册机制无编译问题。

### 6.2 非 Editor 游戏目标编译测试

已执行并通过：

```powershell
& "e:/AkStudio/UnrealEngine/Engine/Build/BatchFiles/Build.bat" TheWilds Win64 Development -Project="e:/AkStudio/TheWildsGame/TheWildsClient/TheWilds/TheWilds.uproject" -WaitMutex -NoHotReload
```

结果：

```text
Result: Succeeded
```

验证点：

- Runtime 模块不依赖 `UnrealEd`。
- 非 Editor 游戏目标可以加载 `UnrealAIBridgeRuntime`。
- `UnrealAIBridge` Editor 模块不会进入游戏目标。
- 运行时 RPC Server 架构正确。

### 6.3 MCP Server 协议测试

已测试：

1. `initialize`
2. `tools/list`
3. `tools/call` 在 Unreal 未启动时的错误返回

验证点：

- MCP Server 可被 stdio 拉起。
- 能返回 MCP capabilities。
- 能返回工具列表。
- Unreal 未启动时，工具调用返回 `isError: true`，不会导致 MCP Server 崩溃。

### 6.4 Python 语法测试

已执行：

```powershell
python -m py_compile e:/AkStudio/UnrealMCPServer/server.py
```

结果：通过。

### 6.5 CodeBuddy MCP 配置测试

已执行：

```powershell
python -m json.tool e:/AkStudio/.mcp.json
```

结果：通过。

并验证 `tools/list` 中包含关键工具：

- `unreal_runtime_ping`
- `unreal_runtime_call`
- `unreal_editor_info`
- `unreal_editor_list_actors`
- `unreal_editor_find_assets`

---

## 7. 关键问题与经验

### 7.1 Editor 模块不能作为 Runtime 消息入口

最初误区：

```text
外部 HTTP -> Editor 模块 -> runtime.* -> Lua
```

这个设计只在 Editor / PIE 下成立。

真正游戏运行时不会加载 Editor 模块，因此：

- `FUnrealAIBridgeModule` 不存在。
- `StartServer()` 不会执行。
- `/rpc` 不会监听。
- `runtime.*` 无法分发到 Lua。

最终经验：

> 只要目标包含 packaged game / standalone runtime，消息入口必须放在 Runtime 模块中。

### 7.2 Editor 能力应该作为 Runtime 的扩展 namespace

正确方式：

```text
Runtime 模块拥有 Dispatch
Editor 模块只注册 editor handler
```

优点：

1. Runtime 不依赖 Editor。
2. Editor 能力在 Editor 下自动出现。
3. 游戏运行时不会误链接 `UnrealEd`。
4. 后续可以继续注册其它 namespace。

### 7.3 HTTP 回调线程不能直接操作 Lua / GEditor

HTTPServer 的请求回调不一定在 GameThread。

Lua VM 和 GEditor API 都应在 GameThread 使用。

因此 Runtime RPC Host 中做了线程切换：

```text
HTTP 线程 -> AsyncTask(GameThread) -> Dispatch -> Lua / Editor
```

经验：

> 所有 Unreal 对象、Lua VM、GEditor 操作都应尽量回到 GameThread。

### 7.4 避免 Runtime Public Header 暴露不必要依赖

一开始 Runtime 头文件直接暴露了 HTTPServer 类型，后来改为 PIMPL：

```cpp
struct FUnrealAIBridgeRuntimeServerState;
TUniquePtr<FUnrealAIBridgeRuntimeServerState> ServerState;
```

好处：

- 降低 public header 依赖。
- 避免 Editor 模块被迫知道 HTTPServer 实现细节。
- 编译耦合更小。

### 7.5 命名避免与 UE 全局模板冲突

曾出现 `MakeError` 与 UE 全局 `MakeError` 模板冲突的问题。

现象：

```text
无法从 TValueOrError_ErrorProxy 转换为 FString
```

原因：

- 自己定义的匿名 namespace `MakeError` 名字过通用。
- UE 中已有同名模板函数。
- `TEXT()` 字面量参与重载时选到了错误函数。

解决：

```cpp
MakeError -> MakeBridgeError
MakeOk -> MakeBridgeOk
```

经验：

> 在 UE 项目里，helper 函数不要用过于通用的名字，尤其是 `MakeError`、`Result`、`Check` 这类容易撞名的命名。

### 7.6 MCP Server 不应直接耦合 Unreal C++

MCP Server 只负责协议适配：

```text
MCP tools -> HTTP JSON-RPC -> Unreal Bridge
```

好处：

- MCP Server 可独立发布。
- 不依赖 Unreal 编译环境。
- 可被 CodeBuddy / Claude / Cursor 复用。
- Unreal 插件和 MCP 协议可分别演进。

### 7.7 工具错误应返回 MCP tool error，而不是协议级错误

当 Unreal 未启动时，调用 `unreal_runtime_ping` 会连接失败。

MCP Server 应返回：

```json
{
  "isError": true,
  "content": [
    {
      "type": "text",
      "text": "Tool 'unreal_runtime_ping' failed: ..."
    }
  ]
}
```

而不是让 server 崩溃或返回 JSON-RPC `-32603`。

经验：

> 工具执行失败是业务错误，不一定是 MCP 协议错误。

---

## 8. 当前能力边界

### 8.1 Runtime 可用范围

`runtime.*` 可在以下场景使用：

- Unreal Editor
- PIE
- Standalone Game
- packaged runtime

前提：

- `UnrealAIBridgeRuntime` 已加载。
- HTTP Server 成功监听。
- Lua VM 已启动。
- Lua 侧已安装 `_G.AICommandBridgeExecute`。

### 8.2 Editor 可用范围

`editor.*` 只在 Unreal Editor 下可用。

独立游戏或 packaged runtime 中调用 `editor.*` 会返回：

```text
unknown_namespace
```

这是正确行为。

### 8.3 安全边界

当前 HTTP 服务绑定本机 loopback：

```text
127.0.0.1:18777
```

这是本地开发工具，不应直接暴露到公网。

其中 `editor.exec` 可以执行 Editor 控制台命令，能力较强，应只在可信环境中使用。

---

## 9. 后续可扩展方向

### 9.1 Unreal 插件侧

可继续增加：

- `editor.actor.spawn`
- `editor.actor.destroy`
- `editor.actor.select`
- `editor.actor.set_transform`
- `editor.screenshot`
- `editor.viewport.focus`
- `editor.map.open`
- `editor.asset.load`
- `editor.asset.create`
- `runtime.world.info`
- `runtime.player.info`
- `runtime.console.exec`

### 9.2 MCP Server 侧

可继续增加：

- 更细粒度工具 schema。
- 对常用 Lua 命令自动生成 MCP tools。
- 增加健康检查 resource。
- 增加 prompts，比如「分析当前关卡 Actor」等。
- 支持多 Unreal 实例，通过 env 或参数指定不同端口。

### 9.3 Lua 侧

可继续沉淀：

- AI 可调用的业务命令注册规范。
- 命令参数 schema。
- 命令权限控制。
- 命令执行日志。
- 异步命令回调机制。

---

## 10. 结论

本次开发形成了完整的 AI 控制 Unreal 链路：

```text
CodeBuddy MCP
  -> UnrealMCPServer
  -> UnrealAIBridgeRuntime
  -> Lua Runtime / Unreal Editor
```

最关键的架构收益是：

1. **运行时入口下沉到 Runtime 模块**，解决独立游戏 / packaged runtime 无法接收消息的问题。
2. **Editor 能力通过 namespace 注册扩展**，避免 Runtime 依赖 `UnrealEd`。
3. **MCP Server 独立实现**，成为 CodeBuddy 与 Unreal Bridge 之间的标准协议适配层。
4. **编译和协议测试通过**，验证了 Editor 与非 Editor 两条链路。

一句话总结：

> UnrealAIBridge 负责在 Unreal 内执行命令，UnrealMCPServer 负责把 CodeBuddy 的 MCP 工具调用翻译成 Unreal RPC；二者组合后，AI Agent 就可以同时驱动游戏运行时 Lua 和 Unreal Editor 能力。
