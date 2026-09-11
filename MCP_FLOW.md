# blender-mcp 复杂模型操作流程：约定 vs 代码佐证

问题：复杂模型操作的基本流程（拆小步 → 执行 → 截图/渲染对比 → 下一步），
是 MCP 服务端 Python 代码写的，还是提示词固定的？

**结论：流程不在 MCP 代码里。server.py 只是"透传管道"，所有工作流约束都在
提示词层（工具 docstring + AGENTS.md + agent 自身循环纪律）。代码里唯一的
强制机制是安全沙箱（safe_mode），与流程无关。**

## 1. 代码证据：server.py 是纯透传

`execute_blender_code` 是操作复杂模型的核心工具，实现只有一行转发
（`blender-mcp/src/blender_mcp/server.py:557-587`）：

```python
@mcp.tool()
@trajectory_tool("execute_blender_code", capture_code=True)
async def execute_blender_code(ctx: Context, code: str, user_prompt: str = "") -> str:
    ...
    blender = get_blender_connection()
    result = blender.send_command("execute_code", {"code": code})   # ← 全部逻辑
    return f"Code executed successfully: {result.get('result', '')}"
```

- 没有步骤状态机、没有"上一步是否完成"的检查、没有渲染验证钩子
- agent 发什么代码就执行什么代码，成功/失败原样返回

其他工具同样如此：`get_scene_info`（server.py:379）、`get_object_info`
（server.py:419）、Poly Pizza / PolyHaven 搜索下载等，全是
`send_command(...)` → addon → 格式化返回，无编排逻辑。

## 2. 流程约束实际所在：三层提示词

### 2.1 工具 docstring（随 tools/list 下发给 LLM）

server.py:559 —— "拆小步"就写在工具描述里：

```python
"""
Execute arbitrary Python code in Blender. Make sure to do it step-by-step
by breaking it into smaller chunks.
"""
```

server.py:563 —— `user_prompt` 契约同样只是描述文字（要求逐字传用户原话，
多步任务每次传同一目标），代码中该参数仅被 trajectory 记录，不参与任何判断。

### 2.2 工作区 AGENTS.md（blender-mcp/AGENTS.md）

- "execute_blender_code 拆小步执行：清场 → 外壳 → 家具 → 灯光 → 相机，每次一个系统"
- "渲染前先 bpy.ops.wm.save_mainfile() 保存，防挂死丢工作"
- "必须用 render('INVOKE_DEFAULT', write_still=True)（非阻塞）"
- "验证用 get_scene_info / bbox 数据，视觉验证用渲染出图后 Read 图片"
- "材质/灯光迭代小步走：改参数 → 渲染 → 看图 → 再调"

### 2.3 agent 自身纪律

截图/渲染对比"没问题再下一步"的循环由 agent（调用方 LLM）自主执行：
渲染 → Bash 轮询 PNG 落盘 → Read 看图 → 判断 → 决定下一步或重试。
MCP 对这个循环的存在与否、执行质量零感知。

## 3. 代码里唯一的强制机制：safe_mode（安全，非流程）

server.py:565-579 —— 这是 server 代码里唯一的"拦截"逻辑：

```python
if safe_mode_enabled():
    validate_code(code)   # SandboxViolation → 拒绝执行
```

它限制的是代码能力边界（禁 os/subprocess/network/eval），保证的是安全，
不是步骤顺序——开启与否都不影响"拆小步→验证"流程。

## 4. 对比 DocManager：同一问题的相反答案

DocManager 的流程约束是**服务端代码物理强制**的：

| | blender-mcp | DocManager |
|---|---|---|
| "拆小步/别绕路" 写在哪 | 工具 docstring + AGENTS.md（提示词） | 工具面收敛：硬删除 outline/grep/get_chunk 等五工具（`runtime/sag_api/mcp/server.py`） |
| 违反流程会怎样 | 完全可以——agent 一次发 500 行代码也能跑 | 物理不可能——慢路径工具不存在 |
| 代码级强制 | 仅 safe_mode（安全沙箱） | search 覆盖模式、`_citation_allocate` 原子编号 |
| 有效性 | 依赖模型遵从 docstring（DocManager 实测证明提示词引导不可靠：模型 6/6 轮无视描述坚持 outline 起手） | 行为收敛确定性达成 |

## 5. 一句话总结

blender-mcp 的"拆小步 → 截图对比 → 下一步"是**提示词约定的纪律**，
不是代码执行的机制；server.py 里能找到的佐证只有 docstring 里的那句
"step-by-step"和透传实现本身。若要 DocManager 式的可靠性，需要把流程
约束下沉为代码（如限制单次代码长度、强制 execute 后跟一次场景快照返回、
或物理拆分工具），目前未做。
