English | **[中文文档](README_CN.md)**

# vistools Skills

[vistools](https://github.com/ZeroZ-lab/vistools) 的 Claude Code 插件 —— 为 AI 编程助手提供视觉工具。支持检查、导航、裁剪和采样大图，输出结构化 JSON 并附带坐标映射。

## 为什么需要这个工具

Claude Code 直接读取大图（如 3200×2400 的截图）时会丢失细节，甚至超出上下文限制。`vistools` 通过为 Claude 提供一套图像工具来解决这个问题：检查尺寸、生成缩放概览、精确裁剪区域、采样单个像素。每次操作都返回带有**坐标映射**的结构化 JSON，让 Claude 始终能将看到的内容追溯回源图。

## 安装

添加市场并安装插件：

```bash
/plugin marketplace add ZeroZ-lab/vistools-skills
/plugin install vistools@vistools-skills
```

macOS（arm64/x64）和 Linux（arm64/x64）的二进制文件通过 CI 自动打包。

## 使用方法

**`/vistools <图片路径> [关注点]`**

`focus` 参数是可选的自然语言指令，告诉 Claude 要关注什么或做什么。它由 AI 解读，不会作为 CLI 参数解析。

```
/vistools screenshot.png
/vistools large-image.jpg "关注页头部分"
/vistools design.png "提取登录按钮"
```

自动化工作流：
1. 检查图片 → 推荐概览/分块/视口策略
2. 生成输出并附带坐标映射
3. 采样颜色以验证像素
4. 报告结果并标注源图坐标

### 示例工作流

> 以 `/vistools` 开头的命令是你在聊天中输入的。以 `vistools`（无斜杠）开头的命令是 Claude 内部执行的。

```bash
# 1. 你对一张大截图调用 skill
/vistools screenshot.png

# Claude 看到 3200×2400，决定生成缩放概览
# 2. 生成概览（Claude 内部执行）
vistools overview screenshot.png overview.png --max-side 1200

# 3. 你在概览中发现约 (800, 600) 处有个 bug
# 4. Claude 映射回源图坐标：(800 / 0.375, 600 / 0.375) = (2133, 1600)
# 5. 从源图中裁剪该区域
vistools viewport rect screenshot.png bug.png \
  --x 2000 --y 1500 --width 500 --height 400

# 6. 采样一个像素以验证其颜色
vistools sample screenshot.png --x 2100 --y 1550
```

## 坐标映射

每个生成的图片都会返回 `coordinate_mapping`：

```json
{
  "crop_origin_in_source": [960, 720],
  "scale_factor": null,
  "formula": "source_x = result_x + 960, source_y = result_y + 720"
}
```

将输出坐标映射回源图：
- **裁剪**：`source = (result_x + origin_x, result_y + origin_y)`
- **概览（缩放）**：`source = (result_x / scale + origin_x, result_y / scale + origin_y)`

## 插件结构

```
.claude-plugin/
└── plugin.json              # 插件清单

skills/
└── vistools/
    ├── SKILL.md             # Skill 定义（给 Claude 的指令）
    └── scripts/
        ├── vistools                 # Bash 包装脚本 —— 检测操作系统/架构
                                     # 并运行下方对应的二进制文件
        ├── vistools-macos-arm64     # CI 构建的二进制文件
        ├── vistools-macos-x64
        ├── vistools-linux-arm64
        └── vistools-linux-x64
```

## 故障排除

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

## 相关资源

- **[vistools](https://github.com/ZeroZ-lab/vistools)** —— Rust CLI 源码
- **[SKILL.md](skills/vistools/SKILL.md)** —— Skill 定义

## 许可证

MIT OR Apache-2.0
