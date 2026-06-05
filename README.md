# imagegen-fix — OpenAI gpt-image-2 脚本修复版

> **核心修复：默认禁用自动重试，确保一次 API 调用只扣一次费，杜绝由于重试导致的重复扣费问题。**

## 背景问题

原始 OpenAI gpt-image-2 脚本存在**两层重试机制叠加**导致的重复扣费问题：

### 第一层：OpenAI Python SDK 内置重试
```python
# 原始代码
return OpenAI()  # 默认 max_retries=2
```
SDK 默认在收到 HTTP 响应的解析过程中如果连接中断，会自动**重新发送完整的 API 请求**。这意味着即使首次请求已成功生成图片，只要客户端收响应时断连，SDK 就会发第二次请求，生成第二张图，扣除第二次费用。

### 第二层：脚本自建重试
```python
# 原始代码默认最多重试 3 次
getattr(args, "max_attempts", 3)
```
`generate` 和 `edit` 子命令没有独立的 `--max-attempts` 参数，fallback 默认是 3，与 SDK 的 2 次重试叠加后造成大量静默重复扣费。

## 修复内容

| 修改项 | 原始值 | 修复值 | 作用 |
|--------|--------|--------|------|
| `OpenAI()` | `max_retries` 默认 2 | `OpenAI(max_retries=0)` | 禁用 SDK 层自动重试 |
| `AsyncOpenAI()` | `max_retries` 默认 2 | `AsyncOpenAI(max_retries=0)` | 禁用异步 SDK 层自动重试 |
| `generate` 子命令 | 无 `--max-attempts` 参数，fallback=3 | 新增 `--max-attempts`，默认值=1 | 脚本层重试禁用 |
| `edit` 子命令 | 无 `--max-attempts` 参数，fallback=3 | 新增 `--max-attempts`，默认值=1 | 同上 |
| `generate-batch` 子命令 | `--max-attempts` 默认值=3 | `--max-attempts` 默认值=1 | 批量模式也默认单次 |

### 修复后行为

- **默认无任何重试**：一次调用 = 一次 API 请求 = 一次扣费
- **如果上游瞬时故障**（如 502），脚本立即报错退出，无需额外等待，不浪费额度
- **保留手动重试选项**：用户可显式使用 `--max-attempts 2` 开启重试

## 使用方法

### 安装依赖

```bash
pip install -r requirements.txt
# 或
uv pip install openai Pillow
```

### 环境变量

```bash
export OPENAI_API_KEY="sk-..."
export OPENAI_BASE_URL="https://api.openai.com"  # 可选，默认使用 OpenAI 官方
```

### 生成图片

```bash
# 默认无重试，一次调用一次扣费
python image_gen.py generate --prompt "描述你的图片" --size 1024x1024 --out output.png

# 指定重试次数（仅在网络不稳定时使用）
python image_gen.py generate --prompt "..." --size 1024x1024 --max-attempts 2

# 使用提示词文件
python image_gen.py generate --prompt-file prompt.txt --size 1024x1536 --out output.png
```

### 编辑图片

```bash
python image_gen.py edit --image input.png --prompt "修改背景" --size 1024x1024 --out edited.png
```

### 批量生成

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

## 验证方法

运行以下命令后，检查 API 日志确认只扣了一次费用：

```bash
python image_gen.py generate --prompt "test" --size 1024x1024 --out test.png --max-attempts 1
```

如果遇到 502 等上游错误，脚本会**立即报错退出**，不会发起第二次请求。此时可自行判断是否需要重试。

## 注意事项

- **不要同时使用 `--force` 和 `--max-attempts` 高值**：如果网络不稳定，高重试次数可能导致多张图片生成
- **第一次使用建议用 `--dry-run`** 检查参数是否正确
- 如果使用第三方中转 API（如 tokenstation.top），部分上游可能返回 502，属于正常范围，重试即可
- 本脚本与官方 gpt-image-2 模型兼容，也兼容支持该模型的第三方中转接口

## License

MIT
