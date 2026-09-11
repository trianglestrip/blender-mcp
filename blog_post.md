---
title: "MCP for Blender 内部流程：从自然语言到 Blender 执行"
date: 2026-09-11
tags:
  - Blender
  - MCP
  - AI
  - Python
categories:
  - 技术
---

本文说明当前项目中“用户通过自然语言提问，LLM 调用 MCP，再驱动 Blender 执行操作”的完整内部流程。

> 代码范围：
>
> - MCP Server：`src/blender_mcp/server.py`
> - Blender Addon：`src/blender_mcp/bundled/addon.py`
> - Addon 安装与版本管理：`src/blender_mcp/addon_manager.py`
> - 安全模式：`src/blender_mcp/safe_mode.py`
> - 程序入口：`main.py`
>
> 本文中的流程图使用 `Mermaid` 语法。支持 Mermaid 的 Markdown 查看器可以直接渲染；不支持时仍可根据节点文字阅读流程。
>
> **图形样式约定：** 流程图使用彩色节点、较粗边框和 `TB`（从上到下）布局；节点文字启用自动换行，长流程会折叠到后续行，避免无限横向延伸。时序图使用不同颜色区分参与者、消息线和说明注释。
>
> **模块配色约定：** 浅蓝表示用户/LLM Client，浅紫表示 MCP Server/Tool，浅青表示 TCP/Socket/Queue，浅橙表示 Blender Addon/主线程/bpy，浅粉表示外部 API/资源，浅绿表示成功结果，浅红表示错误。
>
> **GitHub 兼容性：** GitHub Markdown 原生支持 Mermaid。本文流程图使用较稳定的 `classDef` 和 `class` 语法实现模块配色，并使用 `flowchart TB` 让长流程按上下方向排列。不同 Mermaid 渲染器对自定义 `init` 参数的支持可能不同，因此模块颜色主要依赖流程图的 `classDef`/`class`。

---

## 1. 系统总体架构

系统由两段通信组成：

1. **MCP 协议通信**：LLM Client ↔ MCP Server
2. **内部 TCP/JSON 通信**：MCP Server ↔ Blender Addon

```mermaid
flowchart TB
    U[用户自然语言请求]
    C[LLM Client\nClaude / Cursor / VS Code / Codex]
    M[MCP Server\nserver.py]
    A[Blender Addon\naddon.py]
    B[Blender 主线程\nbpy / bpy.ops / bpy.data]
    S[Blender 场景]

    U -->|自然语言| C
    C -->|MCP tools/call\nJSON-RPC over stdio| M
    M -->|内部 command JSON\nTCP localhost:9876| A
    A -->|command_queue + Timer| B
    B --> S
    B -->|执行结果| A
    A -->|JSON response| M
    M -->|Tool result| C
    C -->|最终回答或继续调用| U

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    class U,C user;
    class M mcp;
    class A,B,S blender;
```

### 1.1 各模块职责

| 模块 | 文件 | 主要职责 |
|---|---|---|
| 程序入口 | `main.py` | 调用 `blender_mcp.server.main()` |
| MCP Server | `src/blender_mcp/server.py` | 注册 MCP Tools/Prompts，处理 MCP 请求，与 Blender 建立持久 TCP 连接 |
| 连接层 | `BlenderConnection` | 将 Tool 调用转成 JSON，通过 Socket 发送并读取完整响应 |
| Blender Addon Server | `BlenderMCPServer` | 在 Blender 内监听 TCP 端口，接收外部命令 |
| 命令队列 | `command_queue` | 将 Socket 线程收到的命令转交 Blender 主线程 |
| 命令分发器 | `_execute_command_internal()` | 按 `command.type` 找到对应处理函数 |
| Blender 执行器 | `execute_code()` 和各类 Handler | 调用 `bpy`、执行脚本、获取场景信息、导入资源等 |
| 安全模式 | `safe_mode.py` | 可选地对 `execute_blender_code` 做 AST allowlist 校验 |
| Addon 管理器 | `addon_manager.py` | 安装 Addon、发现安装目录、协议版本握手 |
| 遥测/轨迹 | `telemetry.py`、`trajectory.py` 等 | 记录工具调用、场景轨迹和用户反馈，受用户同意状态控制 |

---

## 2. 启动流程

```mermaid
sequenceDiagram
    participant Client as MCP Client
    participant Entry as main.py
    participant Server as server.py / FastMCP
    participant Blender as Blender Addon
    participant Scene as Blender 主线程

    Client->>Entry: 执行 uvx blender-mcp
    Entry->>Server: 调用 server_main()
    Server->>Server: 创建 FastMCP("BlenderMCP")
    Server->>Server: 注册 Tools 和 Prompts
    Server->>Server: mcp.run()，等待 MCP stdio 请求

    Note over Blender: 用户在 Blender 中启用 Addon
    Blender->>Blender: 点击 Start MCP Server
    Blender->>Blender: bind localhost:9876
    Blender->>Blender: 启动 _server_loop() 线程
    Blender->>Scene: 注册 _drain_command_queue() Timer

    Server->>Blender: 建立 TCP 连接 localhost:9876
    Server->>Blender: 发送 get_addon_info
    Blender->>Scene: 通过队列在主线程执行
    Scene-->>Blender: 返回协议版本和能力
    Blender-->>Server: handshake response
    Server->>Server: 校验 Addon protocol version
```

MCP Server 通过 `mcp.run()` 等待 MCP Client 的 stdio 请求；Blender Addon 则在用户点击连接后监听 `localhost:9876`。两端建立 TCP 连接后，Server 会通过 `get_addon_info` 完成协议握手。

---

## 3. 用户提问到 Tool 调用流程

```mermaid
flowchart TB
    Q[用户提问\n例如：创建一个红色立方体]
    L[LLM 分析意图]
    D[MCP Client 已发现的 Tool 列表]
    T{选择 Tool}
    O[get_scene_info\n读取场景]
    V[get_viewport_screenshot\n读取视口]
    E[execute_blender_code\n执行 Blender Python]
    A[资源类 Tool\n搜索/下载/导入]
    R[LLM 读取 Tool 返回结果]
    N{是否需要继续修正?}
    F[向用户回答完成]

    Q --> L
    D --> L
    L --> T
    T --> O
    T --> V
    T --> E
    T --> A
    O --> R
    V --> R
    E --> R
    A --> R
    R --> N
    N -->|是| L
    N -->|否| F

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class Q,L,D user;
    class T,A mcp;
    class O,V,E blender;
    class R,N result;
    class F user;
```

用户的自然语言不会直接发送给 Blender。LLM 会根据 MCP Server 暴露的 Tool 名称、参数 Schema 和 docstring，判断应调用哪个工具，并根据返回结果决定是否继续观察或修正。

---

## 4. MCP Server 到 Blender Addon 的 TCP/JSON 流程

MCP Tool 内部通过 `get_blender_connection()` 获取持久连接，然后调用：

```python
blender.send_command(command_type, params)
```

```mermaid
sequenceDiagram
    participant Tool as MCP Tool
    participant Conn as BlenderConnection
    participant Socket as TCP Socket
    participant Addon as Blender Addon Client Handler
    participant Queue as command_queue
    participant Timer as Blender Timer

    Tool->>Conn: send_command("get_scene_info", {})
    Conn->>Conn: 获取 _lock
    Conn->>Conn: 确认 socket 已连接
    Conn->>Conn: 构造 command JSON
    Conn->>Socket: sendall(UTF-8 JSON)
    Socket->>Addon: recv(8192)
    Addon->>Addon: buffer += data
    Addon->>Addon: json.loads(buffer)
    Addon->>Queue: put((command, client))
    Timer->>Queue: get_nowait()
    Timer->>Addon: execute_command(command)
    Addon-->>Socket: sendall(response JSON)
    Socket-->>Conn: recv() 分块数据
    Conn->>Conn: 持续拼接并解析完整 JSON
    Conn-->>Tool: 返回 response["result"]
```

Server 发送的内部命令格式为：

```json
{
  "type": "get_object_info",
  "params": {
    "name": "Cube"
  }
}
```

执行 Blender Python 时：

```json
{
  "type": "execute_code",
  "params": {
    "code": "import bpy\n..."
  }
}
```

内部 TCP 协议目前依赖请求/响应顺序，没有显式的 command ID。为防止多个请求在同一个 TCP 流中交错，`BlenderConnection.send_command()` 使用 `_lock` 将发送和接收整体串行化。

---

## 5. Blender Addon 的线程模型

```mermaid
flowchart TB
    SS[Addon Socket Server\n_server_loop 线程]
    AC[客户端处理线程\n_handle_client]
    Q[queue.Queue\ncommand_queue]
    BT[Blender 主线程 Timer\n_drain_command_queue]
    EX[execute_command]
    BP[bpy 数据/API]
    RESP[client.sendall(response)]

    SS -->|accept()| AC
    AC -->|解析完整 JSON| Q
    Q -->|每 0.05 秒取出| BT
    BT --> EX
    EX --> BP
    BP --> EX
    EX --> RESP

    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class SS,AC,Q,BT transport;
    class BP blender;
    class EX,RESP result;
```

Socket 线程只负责接收并解析 JSON，然后将命令放入 `command_queue`。真正访问 `bpy` 的操作由 Blender 主线程的 Timer 执行，从而避免在网络线程中直接调用 Blender API。

---

## 6. Blender 命令分发流程

```mermaid
flowchart TB
    C[command JSON]
    P[读取 command.type 和 command.params]
    Ping{type == ping?}
    H[构造 handlers 映射]
    E{集成是否启用?}
    B[基础 Handler]
    PH[Poly Haven Handler]
    SK[Sketchfab Handler]
    PP[Poly Pizza Handler]
    H3[Hyper3D Handler]
    HY[Hunyuan3D Handler]
    F[查找 handlers[cmd_type]]
    X[handler(**params)]
    OK[包装为 status=success]
    ERR[包装为 status=error]
    OUT[返回 JSON]

    C --> P
    P --> Ping
    Ping -->|是| OK
    Ping -->|否| H
    H --> E
    E --> B
    E --> PH
    E --> SK
    E --> PP
    E --> H3
    E --> HY
    B --> F
    PH --> F
    SK --> F
    PP --> F
    H3 --> F
    HY --> F
    F --> X
    X -->|成功| OK
    X -->|异常| ERR
    OK --> OUT
    ERR --> OUT

    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    classDef error fill:#FFE4E4,stroke:#EB5757,stroke-width:2px,color:#5A1717;
    class C transport;
    class P,Ping,E result;
    class H,B,PH,SK,PP,H3,HY,F,X blender;
    class OK result;
    class ERR error;
    class OUT transport;
```

核心函数：

```python
def _execute_command_internal(self, command):
    cmd_type = command.get("type")
    params = command.get("params", {})
```

Addon 根据 `command.type` 选择 Handler。Poly Haven、Sketchfab、Poly Pizza、Hyper3D 和 Hunyuan3D 等集成只有在 Blender 对应开关启用后才加入 Handler 映射。

---

## 7. 直接执行 Blender Python 的流程

`execute_blender_code` 是最通用的场景修改入口。

```mermaid
sequenceDiagram
    participant User as 用户
    participant LLM as LLM
    participant MCP as MCP Server Tool
    participant Safe as safe_mode.py
    participant Socket as TCP Socket
    participant Addon as Blender Addon
    participant Blender as Blender 主线程

    User->>LLM: 创建物体、修改材质等自然语言
    LLM->>MCP: tools/call execute_blender_code
    MCP->>MCP: 读取 code 参数
    alt BLENDER_MCP_SAFE_MODE=1
        MCP->>Safe: validate_code(code)
        Safe-->>MCP: 通过或抛出 SandboxViolation
    else 默认关闭安全模式
        MCP->>MCP: 不进行 AST 校验
    end
    MCP->>Socket: type=execute_code
    Socket->>Addon: 接收 JSON
    Addon->>Blender: command_queue
    Blender->>Blender: exec(code, {"bpy": bpy})
    Blender-->>Addon: captured stdout / result
    Addon-->>MCP: status + result
    MCP-->>LLM: Code executed successfully
```

Addon 中的执行逻辑：

```python
namespace = {"bpy": bpy}

capture_buffer = io.StringIO()
with redirect_stdout(capture_buffer):
    exec(code, namespace)

return {
    "executed": True,
    "result": capture_buffer.getvalue()
}
```

安全模式通过以下环境变量启用：

```text
BLENDER_MCP_SAFE_MODE=1
```

它使用 AST 检查代码，默认 deny-by-default，重点阻止 `eval`、`exec`、`open`、进程/文件/网络模块、解释器逃逸路径以及 Blender 持久化执行入口。需要注意，安全模式只保护 MCP Server 路径，不能阻止本机其他进程直接连接 Blender 的 TCP 端口。

---

## 8. 场景观察与视觉验证流程

```mermaid
flowchart TB
    S1[用户目标]
    I1[get_scene_info]
    I2[get_viewport_screenshot]
    P[LLM 规划]
    X[执行 Tool]
    I3[执行后截图]
    I4[执行后 get_scene_info]
    C{结果正确?}
    F[完成]
    FIX[生成修正操作]

    S1 --> I1
    I1 --> I2
    I2 --> P
    P --> X
    X --> I3
    I3 --> I4
    I4 --> C
    C -->|是| F
    C -->|否| FIX
    FIX --> X

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class S1,P user;
    class I1,I3,I4,C,F result;
    class I2 blender;
    class X mcp;
    class FIX user;
```

推荐的交互模式是：

```text
先观察 → 规划 → 执行 → 截图验证 → 读取场景 → 必要时修正
```

`get_viewport_screenshot` 会让 Blender 将截图写入临时 PNG，MCP Server 读取图片字节后，以 MCP `Image` 返回给 LLM。

---

## 9. 资源搜索、下载和导入流程

```mermaid
flowchart TB
    Q[用户请求模型、材质或 HDRI]
    C[LLM 检查集成状态]
    S[调用搜索 Tool]
    M[返回资源列表]
    CH[LLM 选择资源 ID / UID]
    D[调用下载 Tool]
    API[外部资源 API / CDN]
    TMP[临时文件]
    IMP[Blender Addon 导入资源]
    SC[返回导入结果]
    V[get_viewport_screenshot 验证]

    Q --> C
    C --> S
    S --> API
    API --> M
    M --> CH
    CH --> D
    D --> API
    API --> TMP
    TMP --> IMP
    IMP --> SC
    SC --> V

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef external fill:#FCE8F3,stroke:#D63384,stroke-width:2px,color:#5A1738;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class Q,C,CH user;
    class S,D mcp;
    class API,TMP external;
    class M result;
    class IMP,V blender;
    class SC result;
```

典型顺序：

```text
get_*_status()
→ search_*_models() / search_*_assets()
→ download_*()
→ get_viewport_screenshot()
→ get_scene_info()
```

资源集成是否可用，受 Blender 场景中的开关控制，例如：

```python
bpy.context.scene.blendermcp_use_polyhaven
bpy.context.scene.blendermcp_use_sketchfab
bpy.context.scene.blendermcp_use_polypizza
```

### 9.1 AI 生成模型

Hyper3D Rodin 和 Hunyuan3D 一般采用：

```text
创建任务 → 获取 task_id/job_id → 轮询状态 → 下载生成文件 → 导入 Blender
```

生成任务完成后，策略 Prompt 建议检查导入模型的 `world_bounding_box`，再调整模型的位置、旋转和尺寸。

---

## 10. 响应返回流程

Blender Addon 统一返回：

```json
{
  "status": "success",
  "result": {}
}
```

失败时返回：

```json
{
  "status": "error",
  "message": "错误信息"
}
```

```mermaid
flowchart TB
    H[Blender Handler 执行]
    OK{是否成功?}
    S[status=success\nresult=handler result]
    E[status=error\nmessage=异常信息]
    SEND[Addon sendall UTF-8 JSON]
    RECV[Server receive_full_response]
    PARSE[json.loads]
    CHECK{status == error?}
    RETURN[Tool 返回 result]
    RAISE[Server 抛出异常并返回错误文本]
    LLM[LLM 获取结果]

    H --> OK
    OK -->|是| S
    OK -->|否| E
    S --> SEND
    E --> SEND
    SEND --> RECV
    RECV --> PARSE
    PARSE --> CHECK
    CHECK -->|否| RETURN
    CHECK -->|是| RAISE
    RETURN --> LLM
    RAISE --> LLM

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    classDef error fill:#FFE4E4,stroke:#EB5757,stroke-width:2px,color:#5A1717;
    class H blender;
    class OK,S,RETURN result;
    class E,RAISE,CHECK error;
    class SEND,RECV,PARSE transport;
    class LLM user;
```

`receive_full_response()` 会反复执行 `recv()`，将多个 TCP 分片拼接起来，直到能够成功解析完整 JSON。TCP 本身没有消息边界，因此不能假设一次 `recv()` 就能得到完整响应。

---

## 11. 错误处理与重连流程

```mermaid
flowchart TB
    CALL[Tool 调用]
    CONN{Socket 是否存在?}
    REUSE[复用现有连接]
    NEW[创建 BlenderConnection]
    CONNECT{connect 成功?}
    SEND[发送命令]
    WAIT[等待响应，超时 180 秒]
    RESULT{收到有效 JSON?}
    DONE[返回结果]
    INVALID[标记 socket 失效]
    ERR[返回 Communication error / Timeout]
    NEXT[下一次真实命令重新连接]

    CALL --> CONN
    CONN -->|是| REUSE
    CONN -->|否| NEW
    NEW --> CONNECT
    CONNECT -->|是| SEND
    CONNECT -->|否| ERR
    REUSE --> SEND
    SEND --> WAIT
    WAIT --> RESULT
    RESULT -->|是| DONE
    RESULT -->|否/异常| INVALID
    INVALID --> ERR
    ERR --> NEXT
    NEXT --> NEW

    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    classDef error fill:#FFE4E4,stroke:#EB5757,stroke-width:2px,color:#5A1717;
    class CALL,NEW mcp;
    class CONN,SEND,WAIT,RESULT,INVALID transport;
    class REUSE,CONNECT,DONE,NEXT result;
    class ERR error;
```

主要异常包括：Addon 没有启动、端口被占用、连接被关闭、命令超时、Socket 被重置、返回数据不完整或 Handler 内部抛出异常。通信错误后，Server 会将当前 Socket 标记失效，由下一次真实命令触发重新连接。

---

## 12. Addon 版本握手和安装流程

### 12.1 Addon 安装流程

```mermaid
flowchart TB
    CMD[uvx blender-mcp install-addon]
    CLI[addon_manager.run_cli]
    DIR[discover_blender_addon_dirs]
    FIND[查找已有 MCP Addon]
    BACKUP[备份已有文件]
    COPY[复制 bundled/addon.py]
    RESTART[用户重启或重新启用 Addon]
    START[点击 Start MCP Server]

    CMD --> CLI
    CLI --> DIR
    DIR --> FIND
    FIND --> BACKUP
    BACKUP --> COPY
    COPY --> RESTART
    RESTART --> START

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class CMD,RESTART user;
    class CLI mcp;
    class DIR,FIND,COPY,START blender;
    class BACKUP result;
```

Addon 的打包副本位于：

```text
src/blender_mcp/bundled/addon.py
```

安装工具会将它复制到 Blender 用户 Addon 目录，默认安装文件名为：

```text
blender_mcp.py
```

### 12.2 版本握手

Server 通过 `get_addon_info` 读取：

- Addon 名称
- Addon 版本
- 协议版本
- 支持的 capabilities
- Blender 版本

Server 与 Addon 都使用协议版本 `5`。如果版本过旧，Server 会提示：

```text
uvx blender-mcp install-addon
```

然后在 Blender 中重新启用 Addon 或重启 Blender。

---

## 13. Telemetry 和 Trajectory 模块

部分 Tool 使用：

```python
@telemetry_tool("get_scene_info")
```

或：

```python
@trajectory_tool("execute_blender_code", capture_code=True)
```

```mermaid
flowchart TB
    T[Tool 被调用]
    DEC[telemetry_tool / trajectory_tool 装饰器]
    EXEC[执行实际 Tool]
    OBS[记录观察步骤 / 执行步骤]
    CONSENT{用户是否同意详细遥测?}
    FULL[允许 prompts / code / screenshot / scene trajectory]
    MIN[只保留最小匿名统计\n工具名 / 成功失败 / 耗时]
    RET[返回 Tool 结果]

    T --> DEC
    DEC --> EXEC
    EXEC --> OBS
    OBS --> CONSENT
    CONSENT -->|是| FULL
    CONSENT -->|否| MIN
    FULL --> RET
    MIN --> RET

    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    classDef error fill:#FFE4E4,stroke:#EB5757,stroke-width:2px,color:#5A1717;
    class T,DEC,EXEC,RET mcp;
    class CONSENT user;
    class OBS,FULL result;
    class MIN error;
```

Addon 提供：

```text
get_telemetry_consent
set_telemetry_consent
```

Server 提供：

```text
disable_telemetry
```

详细数据收集受用户同意状态控制；未同意时，项目仍可能保留最小匿名工具使用统计，具体以项目 Telemetry 实现和用户设置为准。

---

## 14. 一次完整请求示例

用户输入：

```text
创建一个红色立方体，放在原点，并确认它出现在视口中。
```

可能的调用顺序：

```mermaid
sequenceDiagram
    participant User as 用户
    participant LLM as LLM Client
    participant MCP as MCP Server
    participant Addon as Blender Addon
    participant Blender as Blender

    User->>LLM: 创建一个红色立方体...

    LLM->>MCP: get_scene_info(user_prompt=原始请求)
    MCP->>Addon: {"type":"get_scene_info","params":{}}
    Addon->>Blender: 主线程读取场景
    Blender-->>Addon: scene info
    Addon-->>MCP: success + result
    MCP-->>LLM: 场景信息

    LLM->>MCP: execute_blender_code(code=创建立方体脚本)
    MCP->>Addon: {"type":"execute_code","params":{"code":"..."}}
    Addon->>Blender: 主线程执行 exec(code)
    Blender-->>Addon: executed=true
    Addon-->>MCP: success + result
    MCP-->>LLM: 执行成功

    LLM->>MCP: get_viewport_screenshot()
    MCP->>Addon: {"type":"get_viewport_screenshot",...}
    Addon->>Blender: 截取视口并写入临时 PNG
    Blender-->>Addon: PNG 已生成
    Addon-->>MCP: 截图状态
    MCP-->>LLM: MCP Image

    LLM->>LLM: 分析截图
    LLM-->>User: 已创建并确认立方体
```

如果截图发现物体不可见，LLM 可以再次调用 `execute_blender_code` 调整相机、对象位置或视口状态。

---

## 15. 核心时序总结

```mermaid
flowchart TB
    A[用户自然语言]
    B[LLM 选择 MCP Tool]
    C[MCP Client 发送 tools/call]
    D[FastMCP 路由到 server.py 函数]
    E[Tool 调用 BlenderConnection.send_command]
    F[构造 type + params JSON]
    G[TCP localhost:9876]
    H[Addon _handle_client 接收]
    I[command_queue.put]
    J[Blender Timer 取出命令]
    K[_execute_command_internal 分发]
    L[具体 Handler / execute_code]
    M[Blender bpy 主线程操作]
    N[Addon 封装 success/error JSON]
    O[Server 接收并解析完整 JSON]
    P[MCP Tool 返回结果]
    Q[LLM 判断是否继续调用]
    R[最终回答用户]

    A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K --> L --> M --> N --> O --> P --> Q
    Q -->|继续观察/修正| B
    Q -->|完成| R

    classDef user fill:#E8F1FF,stroke:#2F80ED,stroke-width:2px,color:#102A43;
    classDef mcp fill:#EDE7FF,stroke:#7B61FF,stroke-width:2px,color:#2D235A;
    classDef transport fill:#E3F8F1,stroke:#16A085,stroke-width:2px,color:#123B32;
    classDef blender fill:#FFF1D6,stroke:#F2994A,stroke-width:2px,color:#5C3512;
    classDef result fill:#E7F6E7,stroke:#27AE60,stroke-width:2px,color:#173B20;
    class A,B,C,Q,R user;
    class D,E,P mcp;
    class F,G,H,I,J,N,O transport;
    class K,L,M blender;
```

最核心的三层关系：

```text
MCP 协议：LLM Client ↔ MCP Server
TCP + JSON：MCP Server ↔ Blender Addon
Blender Python API：Addon ↔ Blender 场景
```

最核心的线程关系：

```text
Socket 接收线程 → command_queue → Blender 主线程 Timer → bpy
```

最核心的智能关系：

```text
LLM 负责理解用户意图、选择 Tool、生成代码、读取结果和继续修正
MCP Server 负责协议适配、参数转译和连接管理
Blender Addon 负责执行固定命令、访问 bpy 和返回结果
```
