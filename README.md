# api2img — 用中转 API Key 在 Codex 里生成图片

> 📌 **本仓库基于 [MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill) 优化而来。**
> 原项目解决了「中转 Key 在 Codex 里生图」的问题，本仓库在此基础上修复了可能多重扣费的 bug。
> 如果你只需要基础功能，直接用原版就行；如果你遇到过重试导致重复扣费，可以试试这个修复版。

---

## 这技能是干嘛的？

**一句话：让你在 Codex 里，用你买的中转 API Key 生成图片。**

如果你有 OpenAI 官方 Key，直接走官方路径就行，不需要这个技能。  
但大部分人买的是**中转 Key**（比如从 cc-vibe、tokenstation 等各种平台买的），这些 Key 通常比官方便宜很多，但配置起来麻烦——每次都要设环境变量、怕覆盖原来的配置、Key 还会暴露在命令行历史里。

**api2img 就是来解决这些麻烦的。**

---

## 跟官方有什么区别？

| | 官方 OpenAI Key | 中转 Key（用本技能） |
|---|---|---|
| 价格 | 贵 | 便宜很多 |
| 配置 | 设 `OPENAI_API_KEY` 就行 | 要设 Key + Base URL |
| Key 安全 | 你自己管 | DPAPI 加密存本机 |
| 影响原有配置 | 会覆盖 | ✅ 不碰你原来的环境变量 |
| 适不适合本技能 | ❌ 不需要，用官方路径更香 | ✅ 就是为你准备的 |

---

## 快速开始（小白版）

### 第一步：安装依赖

```bash
pip install openai Pillow
```

### 第二步：配置你的中转 Key

把你买中转 Key 时商家给的 **Key** 和 **地址（Base URL）** 准备好，然后运行：

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -BaseUrl "https://商家给你的地址" -Language zh
```

运行后会**弹出一个窗口**，让你输入 Key（输入时看不到字符是正常的，这是安全设计），输入完关掉窗口就行。

> **一次配置，永久生效。** Key 用 Windows 的 DPAPI 加密存到本机，只有你的账号能解密。以后开电脑直接用，不用再配。

### 第三步：生成第一张图

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\invoke-api2img.ps1 generate --prompt "一只可爱的橘猫，趴在窗台上晒太阳" --size 1024x1024 --out my-first-image.png
```

等一两分钟，`my-first-image.png` 就是你的第一张图了。

---

## 完整使用指南

### 我该用哪种方式？

**情况 A：你只想安安静静用，不想每次折腾环境变量**

→ 用 `api2img\scripts\invoke-api2img.ps1` 调用  
→ 它会自动加载你之前配好的 Key 和 URL，不需要你手动设任何东西

```powershell
# 生成图片
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\invoke-api2img.ps1 generate --prompt "描述" --size 1024x1024 --out 1.png

# 编辑图片
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\invoke-api2img.ps1 edit --image 原图.png --prompt "改个背景" --size 1024x1024 --out 改完.png
```

**情况 B：你习惯自己管环境变量**

→ 自己设好环境变量，直接调引擎脚本

```bash
set OPENAI_API_KEY=你的中转Key
set OPENAI_BASE_URL=https://商家给你的地址
python imagegen\image_gen.py generate --prompt "描述" --size 1024x1024 --out 1.png
```

---

### 常用参数

| 你想干嘛 | 命令 |
|---------|------|
| 生成一张方形图 | `... generate --prompt "描述" --size 1024x1024 --out 1.png` |
| 生成竖版人像 | `... generate --prompt "描述" --size 1024x1536 --out 1.png` |
| 一次出 4 张 | `... generate --prompt "描述" --n 4 --out-dir ./output` |
| 编辑图片 | `... edit --image 1.png --prompt "换背景" --out 2.png` |
| 先看看参数对不对 | `... generate --prompt "测试" --dry-run` |

---

### 管理你的配置

**换 Key 不换地址：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -UpdateKey -Language zh
```

**换地址（顺便也能换 Key）：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -BaseUrl "新地址" -Language zh
```

**清除所有配置（换商家了，不想留旧信息）：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -Clear -Language zh
```

---

## 本仓库修复了什么

> 这部分是**这个仓库相对于原版 [MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill) 唯一的不同**。

### 问题

OpenAI 的 Python SDK 有一个「贴心」设计：如果网络波动导致客户端收响应时断连，它会**自动把请求再发一次**。结果就是：

- 第一次请求：API 收到 → 生成了图片 → 准备返回时断了一下
- SDK 自动重试 → API 又收到一次请求 → **又生成了一张新图**
- **后台扣了两次费，你只拿到一张图**

加上底层引擎脚本本身也有重试机制（默认 3 次），两层叠在一起就更乱了。

### 修复内容

**改了一个文件：`imagegen/image_gen.py`**

| 改了什么 | 原来 | 现在 |
|---------|------|------|
| OpenAI 客户端创建 | `OpenAI()`（默认自动重试 2 次） | `OpenAI(max_retries=0)`（不重试） |
| AsyncOpenAI 客户端创建 | `AsyncOpenAI()`（同上） | `AsyncOpenAI(max_retries=0)`（同上） |
| generate 子命令 | 隐式默认重试 3 次 | 显式 `--max-attempts` 默认 1 |
| edit 子命令 | 同上 | 同上 |
| 兜底逻辑 | 默认重试 3 次 | 默认 1 |

### 修复后

- ✅ **默认一次都不重试**：一次调用 = 一次扣费
- ✅ **遇到网络故障**：立即报错退出，不浪费钱
- ✅ **如果你网络真不稳**：可以手动加 `--max-attempts 2` 允许重试
- ✅ **改动极小**：只动了底层引擎的 6 行代码，api2img 入口层未动

### 如果你不想用这个修复版

直接用原版 [MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill) 即可，功能是一样的，只是网络波动时可能出现多重扣费。

---

## 这个技能安全吗？

是的。

1. **你的 Key 不会出现在聊天里**——配置时是弹窗输入，不是打在命令行里
2. **你的 Key 加密存在本机**——Windows DPAPI 加密，只有你的账号能解开
3. **不影响你电脑上其他 API 配置**——它不会改你原来的 `OPENAI_API_KEY`，只是在调用时临时借用
4. **生成的图片发到哪？**——发到你配置的那个中转商家，不是 OpenAI 官方。如果不熟悉那个商家，别用来处理身份证、人脸等敏感内容

---

## 常见问题

### Q: 生成时报错 502？
A：这是你买的中转商家那边的问题。等几秒重试。如果一直报，联系你的商家客服。

### Q: 为什么要等一两分钟？
A：图片生成本来就慢，中转还要多一层转发，这是正常的。

### Q: 提示 `ExecutionPolicy` 错误？
A：Windows 默认禁止跑 .ps1 脚本。加参数：
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 脚本路径
```

### Q: 换电脑了怎么办？
A：Key 是加密存在当前电脑上的，换电脑需要重新配一次。

### Q: 可能重复扣费吗？
A：这个仓库修复了这个问题。如果你用的是原版 [MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill)，在网络波动时可能出现多重扣费。

### Q: 这个仓库跟原版什么关系？
A：本仓库是原版 [MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill) 的一个优化分支，改动只有 `imagegen/image_gen.py` 中的 6 行代码（禁用自动重试），其余文件保持原样。

---

## 致谢

- 原项目：[MrVoler/api2img-skill](https://github.com/MrVoler/api2img-skill) — 解决了中转 Key 在 Codex 里生成图片的问题
- 本仓库仅在其基础上修复了多重扣费问题


