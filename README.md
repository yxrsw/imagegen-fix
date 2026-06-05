# imagegen-fix — OpenAI gpt-image-2 脚本修复版

> **核心修复：默认禁用自动重试，确保一次 API 调用只扣一次费，杜绝由于重试导致的重复扣费问题。**
> 完整保留了原始 OpenAI gpt-image-2 技能的所有文档、脚本和参考资源。

## 目录结构

```
imagegen-fix/
├── image_gen.py              # (已修复) 核心生成脚本 - 默认 max_retries=0, max_attempts=1
├── README.md                 # 本文件
├── requirements.txt          # Python 依赖
├── assets/                   # 技能图标资源
│   ├── imagegen.png
│   └── imagegen-small.svg
├── docs/                     # 完整技能文档
│   ├── imagegen-skill.md     # 完整技能使用说明
│   ├── api2img-skill.md      # 第三方 API 中转使用说明
│   ├── imagegen-license.txt  # 原始技能许可证
│   ├── cli.md                # CLI 使用详解
│   ├── prompting.md          # 提示词编写指南
│   ├── sample-prompts.md     # 示例提示词
│   ├── image-api.md          # API 参数参考
│   └── codex-network.md      # 网络配置说明
└── scripts/                  # 辅助脚本
    ├── remove_chroma_key.py  # 图片透明背景处理工具
    ├── configure-api2img.ps1 # API Key 配置脚本(Windows)
    ├── invoke-api2img.ps1    # API 中转调用助手(Windows)
    └── load-api2img-env.ps1  # 环境变量加载脚本(Windows)
```

## 修复了什么

原始 OpenAI gpt-image-2 脚本存在**两层重试机制叠加**导致的重复扣费问题：

### 第一层：OpenAI Python SDK 内置重试
```python
# 修复前
return OpenAI()  # 默认 max_retries=2
# 修复后
return OpenAI(max_retries=0)
```
SDK 默认在收 HTTP 响应时如果连接中断，会自动重新发送完整的 API 请求，导致第二次扣费。

### 第二层：脚本自建重试
```python
# 修复前：generate/edit 没有独立参数，fallback 默认 3
getattr(args, "max_attempts", 3)
# 修复后：所有子命令默认 1
```
与 SDK 的 2 次重试叠加后大量静默重复扣费。

### 修复内容一览

| 修改项 | 原始值 | 修复值 | 作用 |
|--------|--------|--------|------|
| `OpenAI()` | `max_retries` 默认 2 | `OpenAI(max_retries=0)` | 禁用 SDK 层自动重试 |
| `AsyncOpenAI()` | `max_retries` 默认 2 | `AsyncOpenAI(max_retries=0)` | 禁用异步 SDK 层自动重试 |
| `generate` 子命令 | 无 `--max-attempts` 参数，fallback=3 | 新增 `--max-attempts`，默认值=1 | 脚本层重试禁用 |
| `edit` 子命令 | 无 `--max-attempts` 参数，fallback=3 | 新增 `--max-attempts`，默认值=1 | 同上 |
| `generate-batch` 子命令 | `--max-attempts` 默认值=3 | `--max-attempts` 默认值=1 | 批量模式也默认单次 |

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
# 或
uv pip install openai Pillow
```

### 2. 配置环境变量

```bash
export OPENAI_API_KEY="你的 API Key"
export OPENAI_BASE_URL="https://your-proxy-url.com"  # 可选，不填则使用 OpenAI 官方
```

### 3. 生成图片

```bash
# 默认无重试，一次调用一次扣费
python image_gen.py generate --prompt "描述你的图片" --size 1024x1024 --out output.png

# 如需重试（网络不稳定时）
python image_gen.py generate --prompt "..." --size 1024x1024 --max-attempts 2

# 使用提示词文件
python image_gen.py generate --prompt-file prompt.txt --size 1024x1536 --out output.png
```

### 4. 编辑图片

```bash
python image_gen.py edit --image input.png --prompt "修改背景" --size 1024x1024 --out edited.png
```

### 5. 批量生成

```bash
python image_gen.py generate-batch --input jobs.jsonl --out-dir ./output
```

## 完整参数

### generate（生成）

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--prompt` | 提示词 | — |
| `--prompt-file` | 提示词文件路径 | — |
| `--model` | 模型 | `gpt-image-2` |
| `--size` | 图片尺寸 | `1024x1024` |
| `--quality` | 画质 (low/medium/high/auto) | `auto` |
| `--n` | 一次生成张数 (1-10) | `1` |
| `--out` | 输出路径 | `output.png` |
| `--output-format` | 输出格式 (png/webp/jpeg) | `png` |
| `--background` | 背景色 (transparent/white/black) | — |
| `--max-attempts` | 最大重试次数 (1-10) | `1` |
| `--force` | 覆盖已存在文件 | 否 |
| `--dry-run` | 只打印参数不请求 API | 否 |

## 详细文档

所有原始技能文档已包含在 `docs/` 目录中，无需额外下载：

- **`docs/imagegen-skill.md`** — 原始技能完整使用指南
- **`docs/api2img-skill.md`** — 第三方 API 中转配置与使用
- **`docs/prompting.md`** — 提示词编写原则与技巧
- **`docs/sample-prompts.md`** — 各类型图片的示例提示词
- **`docs/cli.md`** — CLI 命令详解
- **`docs/image-api.md`** — API 参数参考
- **`docs/codex-network.md`** — 网络配置与故障排除

## 辅助脚本

- **`scripts/remove_chroma_key.py`** — 为生成的图片移除纯色背景并输出透明 PNG/WebP
- **`scripts/configure-api2img.ps1`** — 交互式配置第三方中转 API Key（Windows DPAPI 加密存储）
- **`scripts/invoke-api2img.ps1`** — 通过第三方中转调用图片生成

## 验证方法

运行以下命令，检查 API 日志确认只扣了一次费用：

```bash
python image_gen.py generate --prompt "test" --size 1024x1024 --out test.png
```

如果遇到 502 等上游错误，脚本会**立即报错退出**，不会发起第二次请求。

## 注意事项

- **Key 和 URL 需要使用者自行配置**，本仓库不包含任何个人凭据
- **不要同时使用 `--force` 和 `--max-attempts` 高值**，可能导致多张图片生成
- **第一次使用建议用 `--dry-run`** 检查参数是否正确
- 本脚本与官方 gpt-image-2 模型兼容，也兼容支持该模型的第三方中转接口

## License

MIT
