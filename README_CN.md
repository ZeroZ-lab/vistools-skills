English | **中文文档**

# vistools Skills

> **给 AI 编程助手的坐标可信视觉调试协议。**

[vistools](https://github.com/ZeroZ-lab/vistools) 的 Claude Code 插件 — 让 AI 编程助手具备检查、导航、裁剪、采样、衡量大图焦点分布，以及估算白平衡偏色的能力，所有输出都是结构化 JSON，附带指回源图的坐标映射。

本仓库是**插件分发包**。核心 Rust CLI 源码在 [ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools)。

## 问题

当 AI Agent 收到一张 3200×2400 的截图时，它只能一次性看到整张图 — 被压缩、被缩小、细节丢失。它可能说"按钮看起来正确"，却无法验证实际像素值或追溯到源图坐标。

## Before vs After

**没有 vistools：**
```
Claude 读取 3200×2400 截图
  → 压缩后细节丢失
  → 声称"按钮看起来正确"
  → 无法验证
```

**使用 vistools：**
```
1. inspect → 3200×2400，需要 overview
2. overview --max-side 1200 → 缩小预览，scale_factor = 0.375
3. 在 overview 中发现异常位置 (800, 600)
4. 映射回源图：(800 / 0.375, 600 / 0.375) = (2133, 1600)
5. viewport rect → 精确裁剪该区域
6. sample --x 2133 --y 1600 → 颜色是 #e74c3c，不是预期的 #2563eb
7. focus-map bug.png --rows 3 --cols 3 → 最清晰区域不在预期文字附近
8. 报告："源图坐标 (2133, 1600) 处的按钮背景颜色错误"
```

## 安装

```bash
/plugin marketplace add ZeroZ-lab/vistools-skills
/plugin install vistools@vistools-skills
```

macOS（arm64/x64）和 Linux（arm64/x64）二进制文件通过 CI 自动打包。

## 使用

**`/vistools <图片路径> [关注点]`**

`关注点` 是可选的自然语言指令，告诉 Claude 要看什么或做什么。由 AI 解释，不作为 CLI 参数解析。

```
/vistools screenshot.png
/vistools large-image.jpg "聚焦 header 区域"
/vistools design.png "检查登录按钮颜色是否匹配设计稿"
```

自动化工作流：
1. 检查图片 → 建议 overview/tile/viewport
2. 生成带坐标映射的输出
3. 采样颜色验证像素
4. 在需要判断清晰度或对焦时测量局部锐度
5. 在图像质量相关任务中估算焦点或白平衡
6. 报告发现并附源图坐标

### 示例工作流

> 带 `/vistools` 前缀的命令由你在聊天中输入。带 `vistools`（无斜杠）前缀的命令由 Claude 内部执行。

```bash
# 1. 你对一张大截图调用 skill
/vistools screenshot.png

# Claude 看到 3200×2400，决定生成缩小概览
# 2. Overview（Claude 内部执行）
vistools overview screenshot.png overview.png --max-side 1200

# 3. 你在概览中发现约 (800, 600) 处有 bug
# 4. Claude 映射回源图：(800 / 0.375, 600 / 0.375) = (2133, 1600)
# 5. 从源图裁剪该区域
vistools viewport rect screenshot.png bug.png \
  --x 2000 --y 1500 --width 500 --height 400

# 6. 采样像素验证颜色
vistools sample screenshot.png --x 2100 --y 1550
```

## 命令

| 命令 | 用途 |
|------|------|
| `inspect` | 读取元数据 + 获取策略建议（亚毫秒） |
| `overview` | 缩小预览（`--max-side`） |
| `tile` | 网格切图（`--rows`/`--cols`） |
| `viewport` | 裁剪区域：`anchor` / `percent` / `rect` 三种模式 |
| `sample` | 点/区域取色器（只读） |
| `focus-map` | NxM 锐度网格、最清晰 cell 与 focus point |
| `white-balance` | 灰世界 R/G/B 增益，以及 warm/cool 或 green/magenta 偏色方向 |

所有命令返回结构化 JSON，包含 `coordinate_mapping` 用于追溯源图坐标。

`focus-map` 需要 CLI `v0.2.4+`；`white-balance` 需要 CLI `v0.2.5+`。

## 坐标映射

每个生成的图片都返回 `coordinate_mapping`：

```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

把输出坐标映射回源图：
- **裁剪**：`source = (result_x + origin_x, result_y + origin_y)`
- **概览（缩放）**：`source = (result_x / scale + origin_x, result_y / scale + origin_y)`

## 插件结构

```
.claude-plugin/
└── plugin.json              # 插件清单

skills/
└── vistools/
    ├── SKILL.md             # Skill 定义（给 Claude 的指令）
    ├── VERSION              # 插件 + CLI 版本映射
    ├── CURSOR.md            # Cursor 规则
    └── scripts/
        ├── vistools                 # Bash 包装器 — 检测操作系统/架构
        ├── vistools-macos-arm64     # CI 构建的二进制
        ├── vistools-macos-x64
        ├── vistools-linux-arm64
        └── vistools-linux-x64
```

## 版本兼容性

| 插件版本 | 捆绑 CLI 版本 | 构建来源 |
|---------|-------------|---------|
| 0.5.4 | 0.2.5 | [vistools v0.2.5](https://github.com/ZeroZ-lab/vistools) |

验证本地二进制：`skills/vistools/scripts/vistools --version`

## 排错

**"vistools binary not found for platform"**

从源码编译（需要 Rust 1.88+）：
```bash
git clone https://github.com/ZeroZ-lab/vistools && cd vistools && cargo build --release
cp target/release/vistools ../vistools-skills/skills/vistools/scripts/vistools-macos-arm64  # 根据你的平台调整
```

**"Build failed"**

确保已安装 Rust：
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env
```

## 卸载

```bash
/plugin uninstall vistools
```

## 相关

- **[ZeroZ-lab/vistools](https://github.com/ZeroZ-lab/vistools)** — Rust CLI 源码、测试和设计文档
- **[SKILL.md](skills/vistools/SKILL.md)** — 完整 Skill 定义，含决策策略
- **[CURSOR.md](skills/vistools/CURSOR.md)** — Cursor IDE 规则

## 许可证

MIT 或 Apache-2.0
