# imagegen-fix — OpenAI gpt-image-2 图片生成修复版

> **核心修复：默认禁用自动重试，确保一次 API 调用只扣一次费。**
> 完整保留了 OpenAI gpt-image-2 官方技能的 engine（imagegen）和 proxy（api2img）两层架构。
> 使用者下载后，只需配置自己的 API Key 即可使用。

---

## 目录结构

```
imagegen-fix/
├── README.md                       # 本文件 — 项目总说明
├── requirements.txt                # Python 依赖
│
├── imagegen/                       # 🔧 引擎层 — 实际发 API 请求的脚本
│   ├── image_gen.py                #    (已修复) 核心图片生成脚本
│   ├── SKILL.md                    #    原始技能完整文档
│   ├── LICENSE.txt                 #    原始技能许可证
│   ├── remove_chroma_key.py        #    图片透明背景处理工具
│   ├── agents/openai.yaml          #    Codex 代理配置
│   ├── assets/                     #    技能图标
│   └── references/                 #    各类文档
│       ├── prompting.md            #       提示词编写原则
│       ├── sample-prompts.md       #       各类型示例提示词
│       ├── cli.md                  #       CLI 命令详解
│       ├── image-api.md            #       API 参数参考
│       └── codex-network.md        #       网络配置说明
│
├── api2img/                        # 🔑 入口层 — 管理中转 API Key 和 URL
│   ├── SKILL.md                    #    完整使用文档
│   ├── README.md                   #    快速说明
│   ├── scripts/
│   │   ├── configure-api2img.ps1   #       交互式配置中转 API Key（DPAPI 加密存储）
│   │   ├── invoke-api2img.ps1      #       调用助手（自动加载 Key 后调用 image_gen.py）
│   │   └── load-api2img-env.ps1    #       环境变量加载脚本
│   └── agents/openai.yaml          #    Codex 代理配置
│
└── scripts/
    └── remove_chroma_key.py        # 透明背景处理（快捷访问）
```

---

## 架构说明

本项目由**两层**组成，用户可以根据需要选择使用方式：

```
┌─────────────────────────────────────────────────────────┐
│                     api2img 层（入口）                      │
│                                                         │
│  职责：管理第三方中转 API Key 和 URL                          │
│  适用：使用中转代理（如 tokenstation.top）的用户                 │
│  特点：Windows DPAPI 加密存储 Key，不暴露密钥                    │
│                                                         │
│  scripts/invoke-api2img.ps1                              │
│      ↓ 自动加载 Key + URL → 设置环境变量                      │
│      ↓ 调用 image_gen.py                                   │
├─────────────────────────────────────────────────────────┤
│                   imagegen 层（引擎）                       │
│                                                         │
│  职责：实际执行图片生成/编辑的 API 调用                         │
│  适用：直接使用 OpenAI 官方 API，或通过 api2img 中转             │
│  特点：已修复双重扣费问题，默认无重试                              │
│                                                         │
│  image_gen.py generate/edit/generate-batch               │
│      ↓ 发送请求到 OPENAI_BASE_URL                          │
│      ↓ 返回图片并写入磁盘                                    │
└─────────────────────────────────────────────────────────┘
```

**两种使用路径：**

| 路径 | 适合场景 | 配置方式 |
|------|---------|---------|
| **直接使用 image_gen.py** | 自己有 OpenAI API Key 或中转地址 | 设 `OPENAI_API_KEY` + `OPENAI_BASE_URL` |
| **通过 api2img 脚本** | 想用 DPAPI 加密存储 Key，怕泄露 | 运行 `configure-api2img.ps1` 交互式配置 |

---

## 修复说明

### 原始问题

OpenAI Python SDK 默认 `max_retries=2`，加上脚本自己也有 3 次重试，两层叠加导致：

1. 第一次 API 调用成功生成了图片
2. 客户端收响应时连接小波动
3. SDK 自动重新发送请求 → API 又生成一张新图
4. **扣了两次费，只拿到一张图**

### 修复内容

| 修改文件 | 修改点 | 原始值 | 修复值 | 作用 |
|---------|--------|--------|--------|------|
| `imagegen/image_gen.py:402` | `OpenAI()` | `max_retries` 默认 2 | `OpenAI(max_retries=0)` | 禁用 SDK 自动重试 |
| `imagegen/image_gen.py:419` | `AsyncOpenAI()` | `max_retries` 默认 2 | `AsyncOpenAI(max_retries=0)` | 禁用异步 SDK 自动重试 |
| `imagegen/image_gen.py:945` | `generate` 子命令 | 无参数，fallback=3 | 新增 `--max-attempts`，默认=1 | 脚本层重试禁用 |
| `imagegen/image_gen.py:963` | `edit` 子命令 | 无参数，fallback=3 | 新增 `--max-attempts`，默认=1 | 同上 |
| `imagegen/image_gen.py:955` | `generate-batch` 子命令 | 默认=3 | 默认=1 | 批量也默认单次 |
| `imagegen/image_gen.py:972` | 兜底逻辑 | fallback=3 | fallback=1 | 任何未配置的情况 |

### 修复后效果

- ✅ **默认无任何重试**：一次调用 = 一次 API 请求 = 一次扣费
- ✅ **遇到 502 等上游故障**：立即报错退出，不重试、不浪费额度
- ✅ **保留手动选项**：需要时可加 `--max-attempts 2` 开启重试

---

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
# 或
uv pip install openai Pillow
```

### 2. 配置 API Key

**方式 A：直接设环境变量（简单快速）**

```bash
# Linux / macOS
export OPENAI_API_KEY="你的 API Key"
export OPENAI_BASE_URL="https://你的中转地址.com"   # 官方 API 可以不设

# Windows PowerShell
$env:OPENAI_API_KEY="你的 API Key"
$env:OPENAI_BASE_URL="https://你的中转地址.com"       # 官方 API 可以不设
```

**方式 B：通过 api2img 脚本配置（加密存储，推荐中转用户）**

Windows 下运行：
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -BaseUrl "https://你的中转地址.com" -Language zh
```

Key 会被 Windows DPAPI 加密保存到当前用户本机，只有你的账号能解密。

### 3. 生成图片

```bash
# 直接使用引擎
python imagegen\image_gen.py generate --prompt "一只可爱的柴犬" --size 1024x1024 --out my_dog.png

# 通过 api2img 入口（自动加载 Key）
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\invoke-api2img.ps1 generate --prompt "一只可爱的柴犬" --size 1024x1024 --out dog.png
```

---

## 详细用法

### 生成图片（generate）

```bash
# 基础用法
python imagegen\image_gen.py generate --prompt "你的描述" --size 1024x1024 --out output.png

# 使用提示词文件（适合长提示词）
python imagegen\image_gen.py generate --prompt-file prompt.txt --size 1024x1536 --out output.png

# 指定画质
python imagegen\image_gen.py generate --prompt "..." --size 1024x1024 --quality high --out output.png

# 一次性生成多张
python imagegen\image_gen.py generate --prompt "..." --n 4 --out-dir ./output

# 如果在网络不稳定的环境，可以手动开启重试
python imagegen\image_gen.py generate --prompt "..." --size 1024x1024 --max-attempts 3 --out output.png

# 先用 dry-run 检查参数
python imagegen\image_gen.py generate --prompt "test" --size 1024x1024 --dry-run
```

### 编辑图片（edit）

```bash
# 修改已有图片的背景
python imagegen\image_gen.py edit --image input.png --prompt "把背景改成海边日落" --size 1024x1024 --out edited.png
```

### 批量生成（generate-batch）

```bash
# 准备 JSONL 文件（每行一个任务）
echo '{"prompt":"图片1描述"}' > jobs.jsonl
echo '{"prompt":"图片2描述"}' >> jobs.jsonl

# 批量执行
python imagegen\image_gen.py generate-batch --input jobs.jsonl --out-dir ./output
```

---

## 完整参数表

### generate 子命令

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--prompt` | 图片描述提示词 | — |
| `--prompt-file` | 提示词文件路径（与 `--prompt` 二选一） | — |
| `--model` | 模型名称 | `gpt-image-2` |
| `--size` | 图片尺寸，如 `1024x1024`、`1024x1536`、`2048x1152`、`3840x2160` | `1024x1024` |
| `--quality` | 画质：`low` / `medium` / `high` / `auto` | `auto` |
| `--n` | 一次性成张数（1-10） | `1` |
| `--out` | 输出文件路径 | `output.png` |
| `--out-dir` | 输出目录（配合 `--n` 使用） | — |
| `--output-format` | 输出格式：`png` / `webp` / `jpeg` | `png` |
| `--output-compression` | 压缩率（0-100） | — |
| `--background` | 背景色：`transparent` / `white` / `black` | — |
| `--max-attempts` | 最大重试次数（1-10） | **`1`（修复后）** |
| `--force` | 覆盖已存在的输出文件 | 否 |
| `--dry-run` | 只校验参数并打印请求内容，不发真实 API 请求 | 否 |

### edit 子命令

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--prompt` | 修改描述 | — |
| `--image` | 要编辑的图片路径（可多次使用传入多张） | — |
| `--mask` | 蒙版图片路径（指定需修改的区域） | — |
| `--size` | 输出尺寸 | `1024x1024` |
| `--max-attempts` | 最大重试次数 | **`1`（修复后）** |

### generate-batch 子命令

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--input` | JSONL 文件路径（必填） | — |
| `--out-dir` | 输出目录（必填） | — |
| `--concurrency` | 并发数（1-25） | 自动 |
| `--max-attempts` | 每个任务最大重试次数 | **`1`（修复后）** |
| `--fail-fast` | 遇到错误立即终止全部任务 | 否 |

---

## 场景示例

### 场景 1：只想快速生成一张图（最简单）

```bash
export OPENAI_API_KEY="sk-..."
python imagegen\image_gen.py generate --prompt "夕阳下的海滩" --out beach.png
```

### 场景 2：自己有中转服务商

```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://api.你的中转.com"
python imagegen\image_gen.py generate --prompt "古风美女" --size 1024x1536 --out gufeng.png
```

### 场景 3：在中转平台买的是套餐（推荐用 api2img 加密存储）

```powershell
# 首次配置（一次性的）
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -BaseUrl "https://你的中转.com" -Language zh
# 会弹窗让你输入 Key，输入后 DPAPI 加密保存

# 日常使用
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\invoke-api2img.ps1 generate --prompt "赛博朋克城市" --size 1024x1024 --out cyberpunk.png
```

### 场景 4：需要透明背景的图

```bash
# 先生成带纯色背景的图
python imagegen\image_gen.py generate --prompt "一个红色苹果，纯白色背景" --out apple_with_bg.png

# 然后用 remove_chroma_key.py 去掉背景
python imagegen\remove_chroma_key.py --input apple_with_bg.png --output apple.png
```

### 场景 5：给提示词加上详细风格控制

```bash
python imagegen\image_gen.py generate ^
    --prompt "用写实摄影风格生成一张中国美女肖像，自然光、真实皮肤纹理、清冷气质" ^
    --size 1024x1536 ^
    --quality high ^
    --out portrait.png
```

可参考 `imagegen/references/prompting.md` 和 `imagegen/references/sample-prompts.md` 获取更多提示词技巧。

---

## 图片尺寸参考

| 场景 | 尺寸 | 比例 |
|------|------|------|
| 方形头像/缩略图 | `1024x1024` | 1:1 |
| 竖版人像/手机壁纸 | `1024x1536` | 2:3 |
| 横版风景 | `1536x1024` | 3:2 |
| 2K 方形 | `2048x2048` | 1:1 |
| 2K 横版 | `2048x1152` | 16:9 |
| 4K 横版 | `3840x2160` | 16:9 |
| 4K 竖版 | `2160x3840` | 9:16 |

> 注意：尺寸必须符合规范——最长边 ≤ 3840px，两边都是 16 的倍数，长宽比 ≤ 3:1，总像素在 655,360 ~ 8,294,400 之间。

---

## 常见问题

### Q: 生成图片时报错 502？

A: 这是上游服务器（中转或 OpenAI）临时故障。默认不会自动重试，不会浪费额度。
可以等几秒后手动重跑。如果频繁出现，检查你的中转服务状态。

### Q: 怎么确认没有重复扣费？

A：运行后检查你的中转平台/OpenAI 后台的 API 日志。如果 `--max-attempts` 用的是默认值 1，一次调用只会有一次记录。

### Q: `--force` 和 `--max-attempts` 能一起用吗？

A：能用但小心——如果网络不稳定，高重试次数 + `--force` 可能导致多次覆盖写入。正常用默认值 1 即可。

### Q: 在 Windows 上提示 `ExecutionPolicy` 错误？

A：运行 PowerShell 脚本时加参数：
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 脚本路径
```

### Q: 不小心把 Key 写在了命令行历史里？

A：使用 api2img 的 configure 脚本，Key 通过弹窗输入，不会进命令行历史，且用 DPAPI 加密存储。

### Q: 这个项目跟原始 OpenAI 技能有什么区别？

A：唯一的区别就是**修复了双重扣费问题**（SDK `max_retries=0` + 各子命令默认 `max-attempts=1`）。其他功能、参数、行为完全一致。

---

## License

本仓库包含两个许可证：
- 根目录 `requirements.txt` 等文件：MIT
- `imagegen/LICENSE.txt`：原始技能许可证（OpenAI）
- `imagegen/SKILL.md` 等原始文档：保留原始版权声明

详见各子目录中的许可证文件。
