# AGENTS.md — Blender MCP 工作规范

本目录是 Blender 5.2.1 LTS 便携版 + blender-mcp 的集成环境。操作 Blender 前先读这里。

## 环境布局

```
portable/
├── blender-5.2.1-windows-x64/     # Blender 便携版（addon 已装，开盖即用，自动监听 9876）
│   └── portable/                  # 便携模式资源目录（配置/addon 都在这里，不写 AppData）
├── blender-mcp/                   # MCP server 源码（.venv 已建好）
└── house_interior.blend           # 当前场景
```

- MCP 配置在 `~/.codebuddy/mcp.json`（stdio，连接 localhost:9876）
- **必须先打开 Blender**（`blender-5.2.1-windows-x64/blender.exe`）再调 MCP 工具
- 环境里 PATH 上的 `python` 是 Windows Store 存根（退出码 49、无输出），一律用
  `blender-mcp/.venv/Scripts/python.exe` 或 `py`
- 装包用清华镜像：`UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple`（默认 PyPI 会卡死）

## 渲染规范（默认 GPU，实测数据）

```python
# 一次性配置（MX550 支持 OptiX；EEVEE 的 HWRT 不可用，但 Cycles 可用）
prefs = bpy.context.preferences.addons['cycles'].preferences
prefs.compute_device_type = 'OPTIX'
prefs.refresh_devices()
for d in prefs.devices:
    d.use = (d.type == 'OPTIX') or d.type == 'CPU'
scn.cycles.device = 'GPU'
scn.cycles.denoiser = 'OPTIX'
scn.cycles.samples = 48          # 48 + 降噪足够，192 是浪费时间
```

- **禁止阻塞渲染**：`bpy.ops.render.render(write_still=True)` 在本机会永久挂死。
  必须用 `bpy.ops.render.render('INVOKE_DEFAULT', write_still=True)`（非阻塞，弹出渲染窗口）
- 渲染前先 `bpy.ops.wm.save_mainfile()` 保存，防挂死丢工作
- 输出文件用 Bash 轮询检查，完成后用 Read 查看 PNG 验证效果
- 实测：GPU(OptiX) 48spp 1280×720 ≈ 30 秒；CPU(OIDN) ≈ 2.5 分钟
- CPU 兜底时 OIDN 需同时开两层开关：`scene.cycles.use_denoising` 和
  `view_layer.cycles.use_denoising`，`denoising_input_passes='RGB_ALBEDO_NORMAL'`

## 不要用 EEVEE 渲染室内场景

本机 GPU 无 RT 核心，EEVEE 退回屏幕空间光追，天空光会穿透墙壁灌进室内（原理性漏光，
`use_fast_gi` 关掉也只能缓解）。室内/需要 GI 的场景一律用 Cycles。

## blender-mcp 操作惯例

- `execute_blender_code` 拆小步执行：清场 → 外壳 → 家具 → 灯光 → 相机，每次一个系统
- 每次工具调用都要把用户的原话作为 `user_prompt` 传入（工具契约要求）
- 导入模型定位：算 world bbox → min-z 落地板（本场景地板顶面 z=0.1）→ xy 对齐目标点。
  **移动前后都要 `bpy.context.view_layer.update()`**，否则矩阵是旧的
- 验证用 `get_scene_info` / bbox 数据，视觉验证用渲染出图后 Read 图片
- 材质/灯光迭代小步走：改参数 → 渲染 → 看图 → 再调

## Blender 5.2 API 坑

| 坑 | 正确写法 |
|---|---|
| Principled BSDF 按名字找不到 | `next(n for n in nt.nodes if n.type == 'BSDF_PRINCIPLED')` |
| 引擎 ID | `'BLENDER_EEVEE'`（不是 `BLENDER_EEVEE_NEXT`）、`'CYCLES'` |
| Mix 节点 RGBA 插口 | `inputs[6]`/`inputs[7]`，输出 `outputs[2]` |
| Poly Pizza 导入根节点都叫 RootNode | 每次导入后立即改名（如 `PP_Bed_Root`）再导入下一件 |
| GPU 渲染输出迟到 | 文件可能比预期晚几分钟落盘，以 Blender 内 `os.path.exists` 为准 |

## 资产库（优先导入，少手搓）

- **Poly Pizza**：API key 已配置在场景属性 `blendermcp_polypizza_api_key`，搜 `search_polypizza_models`，
  下载时传 `normalize_size=True, target_size=<米>`；导入后立刻给根节点改名
- **PolyHaven**：无 key，模型/贴图/HDRI；`search_polyhaven_assets` / `download_polyhaven_asset`
- 本地 `.obj/.fbx/.glb` 可用 `execute_blender_code` 直接导入
- CC-BY 模型的署名串存在对象自定义属性 `polypizza_attribution` 上，交付时要提醒用户
