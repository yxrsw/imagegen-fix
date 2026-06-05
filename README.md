# api2img — 用中转 API Key 在 Codex 里生成图片

> **你不是因为没有 OpenAI 官方 Key，而是因为你买的是中转 Key。**
> 这个技能就是专门解决这个问题的：让你买的中转 Key 也能在 Codex 里正常生成图片。

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

如果你不想每次打这么长的命令，也可以直接用引擎脚本（跳过 api2img，自己设环境变量）：

```bash
set OPENAI_API_KEY=你的中转Key
set OPENAI_BASE_URL=https://商家给你的地址
python imagegen\image_gen.py generate --prompt "一只橘猫" --out cat.png
```

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

**情况 B：你习惯自己管环境变量，或者想用脚本跑**

→ 自己设好环境变量，直接调 `imagegen\image_gen.py`

```bash
export OPENAI_API_KEY="你的中转Key"
export OPENAI_BASE_URL="https://商家给你的地址"
python imagegen\image_gen.py generate --prompt "描述" --size 1024x1024 --out 1.png
```

**情况 C：你是用 Codex 聊天界面，让它帮你生成**

→ 直接跟 Codex 说："帮我生成一张 xxx 的图片"  
→ Codex 会自动调用这个技能（只要它检测到你已经配好了）

---

### 常用参数

| 你想干嘛 | 命令 |
|---------|------|
| 生成一张方形图 | `... generate --prompt "描述" --size 1024x1024 --out 1.png` |
| 生成竖版人像 | `... generate --prompt "描述" --size 1024x1536 --out 1.png` |
| 生成 2K 高清 | `... generate --prompt "描述" --size 2048x2048 --out 1.png` |
| 一次出 4 张 | `... generate --prompt "描述" --n 4 --out-dir ./output` |
| 编辑图片 | `... edit --image 1.png --prompt "换背景" --out 2.png` |
| 先看看参数对不对 | `... generate --prompt "测试" --dry-run` |

---

### 管理你的配置

**换一个新的中转 Key：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -UpdateKey -Language zh
```

**换一个中转地址：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -BaseUrl "新地址" -Language zh
```

**清除所有配置（换了商家不想留旧信息）：**
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File api2img\scripts\configure-api2img.ps1 -Clear -Language zh
```

---

## 这个技能安全吗？

**安全。因为是这么设计的：**

1. **你的 Key 不会出现在聊天里**——配置时是弹窗输入，不是打在命令行里
2. **你的 Key 加密存在本机**——Windows DPAPI 加密，只有你的账号能解开
3. **不影响你电脑上其他 API 配置**——它不会改你原来的 `OPENAI_API_KEY` 环境变量，只是在调用的瞬间临时借用一下
4. **生成的图片发到哪？**——发到你配置的那个中转商家，不是发给 OpenAI 官方。如果不熟悉那个商家，别用来处理身份证、人脸等敏感内容

---

## 常见问题

### Q: 生成时报错 502 / 连接失败？
A：这是你买的中转商家那边的问题。等几秒重试一下。如果一直报，联系你的商家客服。

### Q: 为什么要等一两分钟？
A：图片生成本来就慢，中转 API 还要再经过一层转发，比直连 OpenAI 会慢一些，这是正常的。

### Q: 提示 `ExecutionPolicy` 错误？
A：Windows 默认禁止跑 .ps1 脚本。加参数绕过：
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File 脚本路径
```

### Q: 换电脑了怎么办？
A：Key 是加密存在当前电脑上的，换电脑需要重新配一次。

### Q: 怎么知道中转支不支持 gpt-image-2？
A：可以先试 `--dry-run` 看看参数对不对。如果商家明确说支持 DALL·E / gpt-image-2 类接口，一般就能用。

### Q: 为什么不像官方那样直接设 `OPENAI_API_KEY` 就行？
A：因为你买的中转 Key 除了 Key 本身，还需要一个**Base URL**（中转地址）。官方 OpenAI 不需要这个，所以一套标准的环境变量就够了。中转用户需要两个信息，这个技能就是帮你把这套东西管起来。

---

## 文件说明

```
imagegen-fix/
│
├── imagegen/               ← 引擎（真正做图片生成的代码）
│   ├── image_gen.py        ← 核心脚本，已修双重扣费问题
│   ├── references/         ← 提示词技巧、API参数等文档
│   └── remove_chroma_key.py  ← 去背景工具
│
├── api2img/                ← 入口（中转 Key 的配置和管理）
│   ├── scripts/
│   │   ├── configure-api2img.ps1  ← 配 Key 和 URL 用的
│   │   ├── invoke-api2img.ps1     ← 调它来生成图片
│   │   └── load-api2img-env.ps1   ← 自动加载环境的
│   └── SKILL.md / README.md
│
├── scripts/
│   └── remove_chroma_key.py  ← 去背景工具（快捷入口）
│
└── requirements.txt
```

---

## 进阶：关于双重扣费修复

> 如果你只是来用中转 Key 生成图片的，这一段可以跳过。

上一版的脚本有个 bug：网络稍微波动一下，它会自动重试，导致你**付了两张的钱，只拿到一张图**。

这个仓库修复了这个问题：
- **默认不重试了**：一次调用 = 一次扣费
- **还想要重试？**：手动加 `--max-attempts 2`
- **遇到网络故障？**：立即报错，不会偷偷重试浪费钱

修复的是 `imagegen/image_gen.py`，跟 api2img 入口层无关。
