<img width="475" height="467" alt="Alexandria Logo" src="https://github.com/user-attachments/assets/fa2c36d3-a5f3-49ab-9dfe-30933359dfbd" />

# Alexandria 有声书生成器

[English](README.md) | 中文

> **关于本分支：** 这是 [Finrandojin/alexandria-audiobook](https://github.com/Finrandojin/alexandria-audiobook) 的一个独立分支，围绕**逐字忠实**重新构建：原文逐字朗读，引号内的对话分配角色声音，而“said Marcus”这类引述标签由旁白朗读而不是被丢弃。生成的脚本与源文本逐字节一致。上游的全部功能依然可用，本分支改变的是脚本的生成方式。关于声音类型、LoRA 训练和批量生成的通用说明，上游 [Wiki](https://github.com/Finrandojin/alexandria-audiobook/wiki) 仍然适用；但与本分支行为相关的问题请在此仓库提交，而不是上游。

利用 AI 驱动的脚本标注和文本转语音技术，将任何书籍或小说转化为全配音有声书。内置 Qwen3-TTS 引擎，支持批量处理，并提供浏览器端编辑器，可逐行精调后导出。

## 示例音频：[sample.mp3](https://github.com/user-attachments/files/25276110/sample.mp3)

逐字渲染示例：[The Ransom of Red Chief](https://github.com/crystal-coding-time/alexandria-audiobook/releases/download/showcase-red-chief/red_chief_alexandria.mp3) — O. Henry（1907），公有领域。32 分钟，五个声音，脚本与源文本逐字节一致。

## 截图

<img src="https://github.com/user-attachments/assets/874b5e30-56d2-4292-b754-4408fc53f5d6" width="30%"></img> <img src="https://github.com/user-attachments/assets/488cde02-6b93-47fa-874b-97a618ae482c" width="30%"></img> <img src="https://github.com/user-attachments/assets/4c0805a6-bb9d-42c1-a9ff-79bb29d0613c" width="30%"></img> <img src="https://github.com/user-attachments/assets/8e58a5bf-ed8f-4864-8545-1e3d9681b0cf" width="30%"></img> <img src="https://github.com/user-attachments/assets/531830da-8668-4189-a0dc-020e6661bfb6" width="30%"></img>

## 主要功能

### AI 驱动流水线
- **本地与云端 LLM 支持** — 兼容任何 OpenAI 兼容 API（LM Studio、Ollama、OpenAI 等）
- **逐字脚本标注** — 代码将每个分块切分为引号内/引号外的片段（span），LLM 只返回标签（`id`、`speaker`、`role`、`instruct`）。脚本文本按偏移量从源文本重新拼接，因此作者的原文永远不会被改写
- **LLM 脚本审校** — 可选的二次 LLM 校验，每个条目只检查两件事：`speaker` 的归属是否正确，以及 `instruct` 是否是可用的语音指令。条目文本始终从原始条目重新套回，因此审校无法改动书中的任何一个字符
- **角色生成** — LLM 分析脚本为每个角色创建声音描述，通过 VoiceDesign 生成参考音频并自动分配克隆声音 — 一键完成从脚本到全角色配音
- **说话人规范化** — 说话人标签被规范化为统一的大写形式（折叠重音符号、去除括注和外层引号、剥离职级头衔、保留标示性别的头衔）。仅在空格、连字符或撇号上存在差异的拼写（"ABBEMARIGNAN" / "ABBE MARIGNAN"）会被统一；相似但不同的名字（JON/JOHN）永远不会被合并
- **说话人别名** — 将多个说话人名映射到同一声音（例如 "YOUNG ELENA" → "ELENA"），共享同一份声音配置
- **智能分块** — 按说话人连续分组（最多 500 字符），保持自然语流
- **上下文保持** — 在分块间传递一份有上限的角色名单（默认为最近说话的 50 个名字）以及最后 3 条脚本条目的简短片段，确保角色名和风格连贯

#### 逐字忠实

目标是符合行业标准的有声书，而不是广播剧：原文逐字朗读。引号内的对话分配角色声音；其余一切 — 包括“said Marcus”这类引述标签 — 都由旁白朗读。LLM 是一个分析型分类器，永远不是书中文本的生成者。

- **逐字重组** — 每个分块被切分为连续的引号内/引号外片段，它们精确地铺满源文本。LLM 看到的是带编号的片段，只返回标签；代码按 `(start, end)` 偏移量从源字符串重建条目。被截断或格式错误的响应损失的是标签，永远不是原文
- **字节一致是断言，不是期望** — 每个分块的条目必须能逐字节拼回该分块；不一致会抛出异常而不是只给出警告。源文本 → `annotated_script.json` 的词覆盖率精确为 1.0
- **NARRATOR 回退** — LLM 未能标注的任何片段都会保留其文本，并归属给 NARRATOR。枚举式占位标签（"SPEAKER 1"、"VOICE 3"）也会被同样拒绝
- **显式降级** — 当脚本已写出但部分片段回退到 NARRATOR 时，脚本生成以返回码 **3** 退出 — 区别于 0（完全正常）和 1（没有产出）。任务日志将其报告为 "completed with degradations"，运行摘要会列出每一个降级的分块

### 语音生成
- **内置 TTS 引擎** — Qwen3-TTS 本地运行，无需外部服务器
- **外部服务器模式** — 可选连接到远程 Qwen3-TTS Gradio 服务器
- **多语言支持** — 英语、中文、法语、德语、意大利语、日语、韩语、葡萄牙语、俄语、西班牙语，或自动检测
- **预置声音** — 9 种预训练声音，支持基于指令的情感/语调控制
- **声音克隆** — 仅需 5-15 秒参考音频即可克隆任何声音
- **声音设计器** — 通过文字描述创建新声音（例如："温暖、低沉的男性声音，语调沉稳"）
- **LoRA 声音训练** — 在自定义语音数据集上微调 Base 模型，创建可跟随指令的持久声音身份
- **内置 LoRA 预设** — 开箱即用的预训练声音适配器，可直接分配给角色
- **数据集构建器** — 交互式工具，逐条创建 LoRA 训练数据集，支持逐条文本、情感设置和音频预览
- **批量处理** — 同时生成数十个语音块，吞吐量达实时速度的 3-6 倍
- **编解码器编译** — 可选的 `torch.compile` 优化，批量解码速度提升 3-4 倍
- **非语言声音** — 作者写出的拟声词（"Ahh!"、"Mmm..."、"Haha!"）按原样朗读，并配合上下文相关的 instruct 指令 — 没有方括号标记，也没有特殊 token。LLM 绝不会添加书中没有的声音；如果你想要，可在编辑器中自行添加
- **自然停顿** — 可配置不同说话人之间（默认 500 毫秒）和同一说话人片段之间（默认 250 毫秒）的静音

### Web UI 编辑器
- **简洁界面** — 5 步核心流水线（设置、脚本、声音、编辑器、结果）加高级工具（设计器、数据集、训练）
- **分块编辑** — 编辑任意行的说话人、文本和指令
- **选择性重新生成** — 单独重新渲染某一分块，无需全部重做
- **批量处理** — 优化的批量渲染，带子批次划分以高效利用 GPU
- **实时进度** — 所有操作的实时日志和状态跟踪
- **音频预览** — 单独播放或按顺序预览整本有声书
- **脚本库** — 保存和载入标注脚本及其声音配置

### 导出选项
- **合并有声书** — 包含所有声音和自然停顿的单个 MP3 文件
- **单独语音行** — 每行单独导出 MP3，方便在 DAW（Audacity 等）中编辑
- **Audacity 导出** — 一键导出 ZIP，包含按说话人分轨的 WAV 文件、LOF 项目文件和标签，可自动多轨导入 Audacity
- **M4B 有声书** — 带章节标记的 M4B 格式（AAC），支持自动检测章节或逐块章节，适用于 Audiobookshelf、Apple Books、VLC 等播放器

---

## 系统要求

- [Pinokio](https://pinokio.computer/)
- LLM 服务器（以下任选其一）：
  - [LM Studio](https://lmstudio.ai/)（本地）— 推荐使用 Qwen3 或类似模型
  - [Ollama](https://ollama.ai/)（本地）
  - [OpenAI API](https://platform.openai.com/)（云端）
  - 任何 OpenAI 兼容 API
- **GPU：** 最低 8 GB 显存，推荐 16 GB 以上 — 详见下方兼容性表格
  - 每个 TTS 模型占用约 3.4 GB 显存；剩余显存决定批量大小
  - 所有平台均可使用 CPU 模式，但速度明显较慢
- **内存：** 推荐 16 GB（最低 8 GB）
- **磁盘：** 约 20 GB（8 GB venv/PyTorch + 约 7 GB 模型权重 + 音频工作空间）

### GPU 兼容性

| GPU | 操作系统 | 状态 | 驱动要求 | 备注 |
|-----|---------|------|---------|------|
| **NVIDIA** | Windows | 完全支持 | 驱动 550+（CUDA 12.8） | 包含 Flash Attention 加速编码 |
| **NVIDIA** | Linux | 完全支持 | 驱动 550+（CUDA 12.8） | 包含 Flash Attention + Triton |
| **AMD** | Linux | 完全支持 | ROCm 6.3+ | 自动应用 ROCm 优化 |
| **AMD** | Windows | 仅 CPU | 不适用 | 不支持 GPU 加速 — 如需 AMD GPU 加速请使用 Linux |
| **Apple Silicon** | macOS | 仅 CPU | 不适用 | 暂不支持 MPS 加速，可运行但速度较慢 |
| **Intel** | macOS | 仅 CPU | 不适用 | |

> **提示：** 无需外部 TTS 服务器。Alexandria 内置 Qwen3-TTS 引擎，模型权重在首次使用时自动从 Hugging Face 下载（每个模型变体约 3.5 GB）。

> **文档：** 关于声音类型、LoRA 训练、批量生成等的深入说明，请参阅 [Wiki](https://github.com/Finrandojin/alexandria-audiobook/wiki)。

---

## 安装

### 方式 A：Pinokio（推荐）

1. 安装 [Pinokio](https://pinokio.computer/)（如尚未安装）
2. 在 Pinokio 中点击 **Download**，粘贴 `https://github.com/crystal-coding-time/alexandria-audiobook`
   - [Pinokio 目录条目](https://beta.pinokio.co/apps/github-com-finrandojin-alexandria-audiobook) 安装的是原始上游项目，不是本分支。
3. 点击 **Install** 安装依赖
4. 点击 **Start** 启动 Web 界面

### 方式 B：Google Colab（无需安装）

没有 GPU 或系统不兼容？在浏览器中使用免费 T4 GPU 运行 Alexandria：

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/crystal-coding-time/alexandria-audiobook/blob/main/alexandria_colab.ipynb)

需要免费的 [ngrok 账号](https://dashboard.ngrok.com/signup) 用于 Web UI 隧道。详细说明请参阅 notebook。

### 方式 C：Docker（NVIDIA GPU）

适用于集成到自动化流水线或服务器部署：

```bash
git clone https://github.com/crystal-coding-time/alexandria-audiobook.git
cd alexandria-audiobook
docker compose up --build
```

需要 [Docker](https://docs.docker.com/get-docker/) 以及 [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)。Web UI 地址为 `http://localhost:4200`。TTS 模型在首次使用时下载，并缓存在 Docker 卷中。用户数据（上传文件、声音配置、训练好的 LoRA 适配器、音频输出）通过绑定挂载持久化到项目目录。

---

## 首次启动 — 必读

如果你是第一次运行 Alexandria，请在操作前仔细阅读本节。

### 1. 必须先启动 LLM 服务器

Alexandria **不包含** LLM — 它通过 API 连接到外部 LLM。在生成脚本之前，你必须先启动以下任一服务：

| 服务器 | 默认 URL | 安装方式 |
|--------|---------|---------|
| [LM Studio](https://lmstudio.ai/) | `http://localhost:1234/v1` | 下载安装，加载模型，启动服务器 |
| [Ollama](https://ollama.ai/) | `http://localhost:11434/v1` | 安装后运行 `ollama run qwen3` |
| [OpenAI API](https://platform.openai.com/) | `https://api.openai.com/v1` | 获取 API Key |

如果在点击"Generate Script"时 LLM 服务器未运行，生成将会失败。请查看 Pinokio 终端获取错误详情。

### 2. 首次 TTS 生成会下载约 3.5 GB 模型

TTS 模型**不包含在安装中**，首次生成音频时会自动从 Hugging Face 下载：

- **每个模型变体约 3.5 GB**（CustomVoice、Base/克隆、VoiceDesign）
- 只有你使用的变体才会下载（大多数用户从 CustomVoice 开始）
- 下载在后台进行 — **请在 Pinokio 终端中查看进度**
- 此时 Web UI 可能看起来没有响应，这是正常的 — 它在等待下载完成
- 首次下载后，模型将缓存在本地，后续加载只需几秒钟

> **提示：** 如果下载似乎卡住了，请检查网络连接。如果失败，重启应用再试 — 会从断点处继续下载。

> **中国大陆用户：** 如果 Hugging Face 下载缓慢或无法连接，请在启动前设置镜像：将环境变量 `HF_ENDPOINT` 设为 `https://hf-mirror.com`。也可以在 start.js 的 `env` 字段中添加：`env: { HF_ENDPOINT: "https://hf-mirror.com" }`。如果遇到速率限制，可注册免费的 [Hugging Face 账号](https://huggingface.co/join) 并设置 `HF_TOKEN` 为你的访问令牌。

### 3. 首批生成需要额外预热时间

每个会话中的首次批量生成比后续生成更慢：

- **MIOpen 自动调优**（AMD GPU）：GPU 核心优化器每个会话运行一次，增加约 30-60 秒
- **编解码器编译**（如已启用）：一次性约 30-60 秒预热，之后所有批次速度提升 3-4 倍
- **这是正常现象。** 首批之后，生成速度会稳定下来

### 4. 显存决定你能做什么

| 可用显存 | 可行操作 |
|---------|---------|
| 8 GB | 一次只能加载一个模型，小批量（2-5 个语音块），可能需要 CPU 卸载 |
| 16 GB | 大多数用例都能舒适运行，批量 10-20 个语音块 |
| 24 GB+ | 全速运行，批量 40-60 个语音块，搭配编解码器编译 |

- 如果显存不足，请在设置标签页中降低 **Parallel Workers** 或 **Max Chars/Batch**
- 生成前关闭其他 GPU 应用程序（游戏、其他 AI 工具）
- 切换声音类型（Custom → Clone → LoRA）时会卸载并重新加载模型，暂时释放显存

### 5. 出问题时去哪里查看

Web UI 显示的是高层状态，**详细日志在 Pinokio 终端中**：

- 点击 Pinokio 侧栏中的 **Terminal** 查看实时输出
- 模型加载、下载进度、显存估算和错误信息都会显示在这里
- 如果 UI 中生成静默失败，终端会显示原因

常见问题及解决方法请参阅 [Troubleshooting](https://github.com/Finrandojin/alexandria-audiobook/wiki/Troubleshooting)。

---

## 快速入门

界面分为 **5 步核心流水线**（绿色标签页，带编号）和 **高级工具**（蓝色标签页，无编号）。只需核心流水线即可生成有声书。

### 核心流水线

**第 1 步 — 设置**
配置 LLM 连接和 TTS 引擎，至少需要：
- **LLM Base URL**：`http://localhost:1234/v1`（LM Studio）或 `http://localhost:11434/v1`（Ollama）
- **LLM API Key**：你的 API 密钥（本地服务器使用 `local`）
- **LLM Model Name**：要使用的模型（例如 `qwen2.5-14b`）
- **TTS Mode**：`local`（内置引擎，推荐）— 直接加载模型，无需外部服务器
- 完成后点击 **Save Configuration**

**第 2 步 — 脚本**
- 使用文件选择器选取书籍文件（.txt、.md 或 .epub）— 选择后自动上传
- 点击 **Generate Annotated Script** — 将书籍发送给 LLM，分割为带有说话人标签和语音指令的标注块
- *（可选）* 如果生成的脚本有问题，点击 **Review Script** — 运行二次 LLM 校验，修复说话人归属错误和不可用的 instruct 指令。它无法改动文本
- 可以使用下方的保存功能将脚本保存供以后使用

**第 3 步 — 声音**
脚本中检测到的每个角色都会有一张声音卡片。为每个说话人：
- 选择声音类型：Custom Voice（最简单）、Clone Voice、LoRA Voice 或 Voice Design
- 使用 Custom Voice 时，从 9 个预设中选择（Ryan、Serena、Aiden 等），可选设置角色风格（例如 "Heavy Scottish accent"）
- **Generate Personas** — 点击后 LLM 分析脚本，为每个角色创建声音描述、生成参考音频并自动分配克隆声音。切换 "Advanced" 可控制批量大小。这是为所有角色分配独特声音的最快方式
- **说话人别名** — 使用声音卡片上的 "Alias of" 下拉菜单将一个说话人映射到另一个角色的声音（例如将 "YOUNG ELENA" 设为 "ELENA" 的别名）。生成音频时，别名说话人使用目标的声音配置
- 更改自动保存 — 各类型详细说明参见 [Voice Types](https://github.com/Finrandojin/alexandria-audiobook/wiki/Voice-Types)

**第 4 步 — 编辑器**
- 点击 **Render Pending** 批量生成所有语音块的音频
- 点击单个语音块试听，或点击 **Play Sequence** 按顺序预览
- 可以内联编辑任何语音块的文本、说话人或指令，然后单独重新生成
- 满意后点击 **Merge All** 将所有内容合并为最终有声书

**第 5 步 — 结果**
- 在浏览器中试听完成的有声书
- 下载 MP3，导出 **M4B**（带章节标记），或点击 **Export to Audacity** 导出按说话人分轨的 WAV 文件

### 高级工具（可选）

这些标签页面向需要更多声音控制的高级用户：

- **设计器** — 通过文字描述创建新声音（例如 "A warm elderly woman with a gentle raspy voice"）。保存后可在声音标签页中用作克隆参考
- **数据集** — 交互式构建 LoRA 训练数据集，逐条创建并支持音频预览
- **训练** — 在语音数据集上训练 LoRA 适配器，创建可跟随指令的持久声音身份

---

## Web 界面

### 设置标签页
配置与 LLM 和 TTS 引擎的连接。

**TTS 设置：**
- **Mode** — `local`（内置引擎）或 `external`（连接 Gradio 服务器）
- **Device** — `auto`（推荐）、`cuda`、`cpu` 或 `mps`
- **Language** — TTS 合成语言：English（默认）、Chinese、French、German、Italian、Japanese、Korean、Portuguese、Russian、Spanish，或 Auto（由模型自行检测）
- **Parallel Workers** — 快速批量渲染的批量大小（越高占用显存越多）
- **Batch Seed** — 固定种子以获得可复现的批量输出（留空则随机）
- **Compile Codec** — 启用 `torch.compile`，批量解码速度提升 3-4 倍（首次生成增加约 30-60 秒预热）
- **Sub-batching** — 按文本长度拆分批次，减少填充造成的 GPU 算力浪费（默认启用）
- **Min Sub-batch Size** — 允许拆分前每个子批次的最少语音块数（默认：4）
- **Length Ratio** — 强制拆分子批次的最长/最短文本长度比上限（默认：5）
- **Speaker Change Pause** — 合并时不同说话人之间的静音毫秒数（默认：500）
- **Same Speaker Pause** — 合并时同一说话人继续时的静音毫秒数（默认：250）

**提示设置（高级）：**
- **Generation Settings** — 分块大小和 LLM 响应的最大 token 数
- **LLM Sampling Parameters** — Temperature、Top P、Top K、Min P 和 Presence Penalty
- **Banned Tokens** — 以逗号分隔的 token 列表，用于禁止出现在 LLM 输出中（可用于关闭 GLM4、DeepSeek-R1 等模型的思考模式）
- **Prompt Customization** — 用于脚本生成的 system 和 user 提示。默认值从 `default_prompts.txt` 载入，可在 UI 中按会话自定义。点击 "Reset to Defaults" 重新载入文件中的默认值（无需重启应用即可生效）

**高级配置键（直接编辑 `app/config.json`）：**

这些键没有 UI 控件，由生成/审校/TTS 代码直接从 `app/config.json` 读取。注意点击 **Save Configuration** 会用 UI 表单字段重写 `config.json` 并丢弃这些键，因此请在最后一次保存之后再设置（或保存后重新添加）。

| 键 | 默认值 | 作用 |
|-----|---------|--------------|
| `generation.num_ctx` | 未设置 | 每次调用向 LLM 服务器请求的服务端上下文窗口，适用于支持该参数的服务器（Ollama）。当看到提示被截断的警告时设置它 |
| `generation.max_context_roster_names` | 50 | 每个分块的角色名单块最多可列出的角色名数量上限。保留的是最近说话的名字；当列表不完整时，提示中会说明这一点 |
| `generation.review_batch_char_budget` | 12000 | 每个审校批次渲染后的 JSON 字符预算，与 `review_batch_size` 一同生效。避免少数超长条目产生的提示超出较小的服务端上下文窗口 |
| `tts.enable_nemo_normalization` | `false` | 可选的文本规范化（"Dr." → "doctor"），**仅**在 TTS 调用边界应用 — 绝不作用于 `instruct`，也绝不作用于脚本文件。可安装时使用 `nemo_text_processing`，否则使用 `wetext`（它有预编译的 Windows wheel）。默认关闭，因为是否有帮助取决于具体书籍和声音 |
| `pronunciation_dict.json`（仓库根目录） | 不存在 | 可选的按书配置文件，内容是扁平的 `{"from": "to"}` 字面替换，在规范化之后应用以纠正其错误（例如 `{"Marlborough Dr.": "Marlborough Drive"}`）。仅在启用规范化时生效 |

在决定之前，可运行 `python tools/verify_tts_normalization.py` 查看测试句子的原始文本、规范化后文本和应用词典后的文本，或加 `--synthesize` 将该句子两种方式都渲染出来试听。

### 脚本标签页
上传文本文件（.txt、.md 或 .epub）并生成标注脚本。EPUB 文件在上传时按 spine（阅读）顺序自动转换为纯文本；每章的字符数会打印到终端，便于确认没有内容丢失。若某个 spine 条目的文件无法解析，或某个 `idref` 在 manifest 中没有对应条目，提取会中止，而不是静默丢掉一章。非文本的 spine 条目（例如 SVG 封面页）和 EPUB3 导航文档会被跳过并给出提示。

LLM 将你的书籍标注为结构化 JSON 格式，包含：
- 说话人识别（NARRATOR 与角色名）
- 作者的文本，逐字从源文本重新拼接 — LLM 只标注片段，永远不输出书中文本
- 用于 TTS 表现的风格指令

**Review Script** — 生成后点击 "Review Script" 可运行二次 LLM 校验。它对每个条目只检查两个字段：
1. `speaker` — 该条目是否归属给了正确的角色？
2. `instruct` — 它是否是可用的语音指令（不是肢体动作，也不为空）？

条目的 `text` 始终从原始条目重新套回，因此审校过程无法改写、拆分、合并或删除任何内容。如果某个批次返回的条目数量与发送时不一致，该批次会被整体拒绝并保留原始条目。说话人更正会被规范化，并对齐到脚本中已经确立的拼写。

审校提示可在 `review_prompts.txt` 中自定义（格式与 `default_prompts.txt` 相同）。

> **注意：** `default_prompts.txt` 带有 `span-labels-v2` 标记，`review_prompts.txt` 带有 `verbatim-review-v1` 标记。在本流水线出现之前保存到 `config.json` 中的自定义提示，向 LLM 要求的是它已无法使用的东西，因此缺少标记的已保存提示会被拒绝并给出明显警告，改用内置默认值。如果你自定义提示，请保留标记行。

### 声音标签页
脚本生成后，声音会自动从标注脚本中载入。对每个说话人：

**角色生成：**
点击 **Generate Personas** 自动为所有角色分配声音。LLM 分析脚本中的对话，为每个说话人生成声音描述和示例文本。这些会被送入 VoiceDesign 模型生成参考音频，保存后作为克隆声音分配。结果是一整套配好声音的角色阵容，无需手动配置。

- **标准模式** — 一次处理所有说话人
- **高级模式** — 切换 "Advanced" 可为大型角色阵容控制批量大小（默认 40）。在高级模式下，说话人按发现批次处理，LLM 会收到对话样本以推断声音特征

**说话人别名：**
每张声音卡片都有一个 "Alias of" 下拉菜单。将一个说话人设为另一个说话人的别名，意味着它在生成音频时使用目标的声音配置。适用于：
- 年龄变体（例如 "YOUNG ELENA" → "ELENA"）
- 自动规则刻意不合并的昵称或拼写变体（例如 "JON" / "JOHN" — 流水线从不自动合并相似的名字，因为它们往往是两个不同的人）
- 减少需要配置的声音数量

别名可传递解析（A → B → C 使用 C 的配置），并带有循环检测。

职级头衔变体（"DR. SMITH" → "SMITH"、"CAPTAIN PICARD" → "PICARD"）不再需要别名 — 规范化会自动剥离开头的职级头衔。

标示性别的头衔会被**保留**，因为它们是用来区分角色而不是装饰某个角色：`MR SMITH` → `MISTER SMITH`，`MRS SMITH` → `MISSUS SMITH`，二者保持为两张声音卡片，因此夫妻不会被合并成同一个声音。被保留的头衔有 MISTER、MISSUS、MS、MISS、MX、MONSIEUR、MADAME、MADEMOISELLE、SIR、LADY、LORD、DAME 和 FR；法语形式在其自身语言内被规范化（"M." → MONSIEUR，"MME" → MADAME），绝不会折叠到英语形式上。如果某本书确实用同一个头衔指代两个人，请将其中一个设为另一个的别名。

`GET /api/voices/alias_suggestions` 会针对当前名单返回基于模糊相似度的建议（JON/JOHN、ELLA/BELLA）。它纯属建议性 — 不会合并任何内容。声音界面会在摘要栏中以可关闭的提示形式展示这些建议。

**Custom Voice 模式：**
- 从 9 种预训练声音中选择：Aiden、Dylan、Eric、Ono_anna、Ryan、Serena、Sohee、Uncle_fu、Vivian
- 设置角色风格，将持久特征附加到每条 TTS instruct 上（例如 "Heavy Scottish accent"、"Refined aristocratic tone"）
- 可选设置种子以获得可复现的输出

**Clone Voice 模式：**
- 选择一个已设计的声音，或输入自定义参考音频路径
- 提供参考音频的准确转录文本
- 注意：克隆声音会忽略 instruct 指令

**LoRA Voice 模式：**
- 从训练标签页选择已训练好的 LoRA 适配器
- 设置角色风格（与 Custom 相同 — 附加到每条 instruct 上）
- 将训练得到的声音身份与 Base 模型的指令跟随能力结合

**Voice Design 模式：**
- 设置基础声音描述（例如 "Young strong soldier"）
- 每行的 instruct 会作为表现/情感指令附加其后
- 使用 VoiceDesign 模型即时生成声音 — 适合次要角色

### 声音设计器标签页
通过文字描述创建新声音，无需参考音频。

- **用自然语言描述声音**（例如 "A warm elderly woman with a gentle, raspy voice and a slight Southern drawl"）
- **预览** — 保存前用示例文本试听
- **保存到库** — 之后可在声音标签页中作为克隆声音参考使用
- 使用 Qwen3-TTS VoiceDesign 模型从描述合成声音特征

### 训练标签页
在 Base 模型上训练 LoRA 适配器以创建自定义声音身份。内置若干 LoRA 预设开箱即用，会与你训练的适配器一同显示。

**数据集：**
- **Upload ZIP** — WAV 文件（24kHz 单声道）+ 包含 `audio_filepath` 和 `text` 字段的 `metadata.jsonl`
- **Generate Dataset** — 从声音设计器描述出发，使用自定义示例文本自动生成训练样本
- **Dataset Builder** — 独立标签页中的交互式工具（见下文），逐条构建数据集并支持预览

**训练配置：**
- **Adapter Name** — 训练出的模型的标识符
- **Epochs** — 遍历数据集的完整轮数（20 条以上样本推荐 15-30）
- **Learning Rate** — 默认 5e-6（保守值）。更高训练更快但有不稳定风险
- **LoRA Rank** — 适配器容量。较高（64+）能强力锁定声音身份，但可能让表现变平。较低（8-16）更能保留表现力
- **LoRA Alpha** — 缩放因子。有效强度 = alpha / rank。常见起点：alpha = 2 倍 rank
- **Language** — 编解码器前缀 token 的语言。应与训练数据的语言一致（English、Chinese、Korean、Japanese 等）。语言不匹配会导致适配器丢失说话人身份
- **Batch Size / Grad Accum** — 24GB 显卡上典型配置是批量 1 搭配梯度累积 8

**训练提示：**
- 包含不同情感（开心、悲伤、愤怒、平静）的样本可获得富有表现力的声音
- 只有中性语气的训练数据会产生平淡的声音，且难以通过 instruct 提示调动
- UI 中的设置信息面板解释了每个参数对声音质量的影响

### 数据集构建器标签页
交互式构建 LoRA 训练数据集，逐条进行。

- **创建项目** — 带声音描述和可选的全局种子
- **定义样本** — 逐行设置文本和情感/风格
- **预览音频** — 生成并试听单条样本，或一次批量生成全部
- **取消批次** — 停止正在进行的批量生成，已完成的样本不会丢失
- **保存为数据集** — 将项目导出为可直接训练的数据集，会出现在训练标签页中
- 已设计的声音和声音设计器描述通过 Qwen3-TTS VoiceDesign 模型驱动音频生成

### 编辑器标签页
导出前精调你的有声书：
- **查看所有语音块** — 带状态标识的表格视图
- **内联编辑** — 点击即可修改说话人、文本或指令
- **单条生成** — 编辑后只重新生成某一个语音块
- **批量渲染** — 处理所有待生成的语音块（见下方渲染模式）
- **按顺序播放** — 按顺序预览音频
- **全部合并** — 将语音块合并为最终有声书

### 渲染模式

Alexandria 提供两种批量渲染音频的方式：

#### Render Pending（标准）
默认渲染模式。按配置的 worker 数量并行发送单独的 TTS 调用。

- **按说话人种子** — 每个声音使用其配置的种子以获得可复现输出
- **支持声音克隆** — 同时适用于预置声音和克隆声音

#### Batch（快速）
高速渲染，在一次批量调用中将多行一起发送给 TTS 引擎。语音块按文本长度排序，并在优化的子批次中处理，以尽量减少填充浪费。

- **3-6 倍实时吞吐量** — 启用编解码器编译后，20-60 个语音块的批次可达实时速度的 3-6 倍
- **子批次划分** — 自动将长度相近的语音块分到一组，高效利用 GPU
- **单一种子** — 所有声音共用配置中的 `Batch Seed`（留空则随机）
- **支持所有声音类型** — Custom、Clone 和 LoRA 声音会被批量处理；Voice Design 为顺序处理
- **Parallel Workers** 设置控制批量大小（值越高占用显存越多）

### 结果标签页
将完成的有声书下载为 MP3，导出为带章节标记的 **M4B** 以供有声书播放器使用，或点击 **Export to Audacity** 导出按说话人分轨的 WAV 文件。

- **Download MP3** — 标准的合并有声书
- **Export M4B** — 带章节标记的 AAC 有声书。默认情况下，章节从脚本中的标题自动检测（例如 "Chapter 1"、"Prologue"）。切换 **Per-chunk chapters** 可获得细粒度导航，每一行都成为一个章节。
- **Export to Audacity** — 包含按说话人分轨 WAV 文件的 ZIP。解压后在 Audacity 中打开 `project.lof` 载入所有音轨，再通过 File > Import > Labels 导入 `labels.txt` 获得语音块标注。

> **注意：** 某些 Linux 有声书播放器（例如 Cozy）对 M4B 支持有限，可能无法识别文件。M4B 输出已在 VLC、Haruna 和 Audiobookshelf 上测试通过。

---

## 性能

### 批量生成推荐设置

| 设置 | 推荐值 | 备注 |
|---------|-------------|-------|
| TTS Mode | `local` | 内置引擎，无需外部服务器 |
| Compile Codec | `true` | 一次性预热后解码速度提升 3-4 倍 |
| Parallel Workers | 20-60 | 越高吞吐量越大，占用显存也越多 |
| Render Mode | Batch (Fast) | 使用批量 TTS 调用 |

### 基准测试

在 AMD RX 7900 XTX（24 GB 显存，ROCm 6.3/7.2）上测试：

| 配置 | 吞吐量 |
|--------------|------------|
| 标准模式（顺序） | 约 1 倍实时 |
| 批量模式，无编解码器编译 | 约 2 倍实时 |
| 批量模式 + compile_codec | **3-6 倍实时** |

一本 273 个语音块的有声书（约 54 分钟音频），在启用批量模式和编解码器编译后，大约 16 分钟即可生成完成。

### ROCm（AMD GPU）说明

> **仅限 Linux。** AMD GPU 加速需要 Linux 上的 ROCm 6.3+。Windows 上的 AMD GPU 以 CPU 模式运行 — 参见 [GPU 兼容性](#gpu-兼容性)。

在 AMD GPU 上运行时，Alexandria 会自动应用 ROCm 专属优化：
- **MIOpen fast-find 模式** — 避免工作区分配失败导致的 GEMM 慢速回退
- **Triton AMD flash attention** — 为 whisper 编码器启用原生 flash attention
- **triton_key 兼容垫片** — 修复 pytorch-triton-rocm 上的 `torch.compile`

这些都是透明应用的，无需任何配置。

> **ROCm 7.x GPU 降频修复：** ROCm 7.x 存在一个回归问题：GPU 的 DPM 控制器会在自回归生成步骤之间激进地降低着色器引擎频率，导致批量生成慢如蜗牛或看起来像卡住。解决方法是将 GPU 电源配置设为 COMPUTE，以强制一个最低频率下限：
>
> ```bash
> echo 5 | sudo tee /sys/class/drm/card1/device/pp_power_profile_mode
> ```
>
> 每次开机需要运行一次（不会跨重启保留）。你可以把它加入系统启动项，或在启动 Alexandria 前手动运行。要确认已生效，可检查以下命令输出中是否有 `COMPUTE*`：
>
> ```bash
> cat /sys/class/drm/card1/device/pp_power_profile_mode
> ```
>
> ROCm 6.x 用户和 NVIDIA 用户不受影响。

---

## 脚本格式

生成的脚本是一个 JSON 数组，每项包含 `speaker`、`text` 和 `instruct` 字段：

```json
[
  {"speaker": "NARRATOR", "text": "The door creaked open slowly. ", "instruct": "Calm, even narration."},
  {"speaker": "ELENA", "text": "\"Ah! Who's there?\"", "instruct": "Startled and fearful, sharp whispered question, voice cracking with panic."},
  {"speaker": "NARRATOR", "text": " she whispered.\n\n", "instruct": "Neutral, even narration."},
  {"speaker": "MARCUS", "text": "\"Haha... did you miss me?\"", "instruct": "Menacing confidence, low smug drawl with a dark chuckle, savoring the moment."}
]
```

- **`speaker`** — 规范化的大写角色名，或 `NARRATOR`。大小写是载荷性的：流水线会与字面值 `"NARRATOR"` 做比较
- **`text`** — 逐字取自源书籍，包括引号和条目之间的空白字符。按顺序拼接每个条目的 `text` 可以精确重现原书，这也正是为什么引述标签（"she whispered"）会成为独立的 NARRATOR 条目而不是被丢弃
- **`instruct`** — 2-3 句直接发送给引擎的 TTS 语音指令。先定语气，再描述表现方式，最后给出具体参照。例如："Devastated by grief, Sniffing between words and pausing to collect herself, end with a wracking sob."

### 非语言声音
拟声词是 TTS 可直接朗读的真实可发音文本 — 没有方括号标记，也没有特殊 token。由于脚本是逐字的，这些内容来自书籍本身（或你在编辑器中自行的编辑），LLM 会为它们配一条简短的 instruct 指令：
- 惊气："Ah!"、"Oh!"，配合类似 "Fearful, sharp gasp." 的 instruct
- 叹息："Haah..."、"Hff..."
- 笑声："Haha!"、"Ahaha..."
- 哭泣："Hic... sniff..."
- 感叹："Mmm..."、"Hmm..."、"Ugh..."

---

## 输出文件

**最终有声书：**
- `cloned_audiobook.mp3` — 带自然停顿的合并有声书

**单独语音行（用于 DAW 编辑）：**
```
voicelines/
├── voiceline_0001_narrator.mp3
├── voiceline_0002_elena.mp3
├── voiceline_0003_marcus.mp3
└── ...
```

文件按时间线顺序编号并带说话人名，便于：
- 导入 Audacity 或其他 DAW
- 放置到各自的角色音轨上
- 精调时间和效果

**Audacity 导出（按说话人分轨）：**
```
audacity_export.zip
├── project.lof       # Open this in Audacity to import all tracks
├── labels.txt        # Import via File > Import > Labels for chunk annotations
├── narrator.wav      # Full-length track with only NARRATOR audio
├── elena.wav         # Full-length track with only ELENA audio
├── marcus.wav        # Full-length track with only MARCUS audio
└── ...
```

每个 WAV 音轨都被填充到相同的总时长，其他说话人发声的位置以静音填充。同时播放所有音轨，听起来与合并后的 MP3 完全一致。

**M4B 有声书（带章节）：**
- `audiobook.m4b` — 内嵌章节标记的 AAC 有声书
- 章节从脚本标题自动检测，或在切换后按语音块划分
- 兼容 Audiobookshelf、Apple Books、VLC、Haruna 以及大多数有声书播放器

---

## API 参考

Alexandria 提供 REST API 以供程序化访问：

### 配置
```bash
# Get current config (empty prompts fall through to file defaults)
curl http://127.0.0.1:4200/api/config

# Get file-based default prompts (hot-reloads from default_prompts.txt)
curl http://127.0.0.1:4200/api/default_prompts

# Save config
curl -X POST http://127.0.0.1:4200/api/config \
  -H "Content-Type: application/json" \
  -d '{
    "llm": {"base_url": "...", "api_key": "...", "model_name": "..."},
    "tts": {
      "mode": "local",
      "device": "auto",
      "language": "English",
      "parallel_workers": 25,
      "batch_seed": 12345,
      "compile_codec": true,
      "sub_batch_enabled": true,
      "sub_batch_min_size": 4,
      "sub_batch_ratio": 5,
      "pause_between_speakers_ms": 500,
      "pause_same_speaker_ms": 250
    }
  }'
```

### 脚本生成
```bash
# Upload text file (supports .txt, .md, .epub)
curl -X POST http://127.0.0.1:4200/api/upload \
  -F "file=@mybook.epub"

# Generate script (returns task ID)
curl -X POST http://127.0.0.1:4200/api/generate_script

# Check status
curl http://127.0.0.1:4200/api/status/script_generation

# Review script (fix speaker attribution and instruct fields; text is never altered)
curl -X POST http://127.0.0.1:4200/api/review_script

# Check review status
curl http://127.0.0.1:4200/api/status/review
```

### 声音管理
```bash
# Get voices and config
curl http://127.0.0.1:4200/api/voices

# Parse voices from script
curl -X POST http://127.0.0.1:4200/api/parse_voices

# Advisory alias suggestions for similar roster names (never merges anything)
# Kept off the /api/voices hot path — it is O(n^2) in roster size
curl http://127.0.0.1:4200/api/voices/alias_suggestions

# Save voice config
curl -X POST http://127.0.0.1:4200/api/save_voice_config \
  -H "Content-Type: application/json" \
  -d '{"NARRATOR": {"type": "custom", "voice": "Ryan", "character_style": "calm"}}'
```

### 语音块管理
```bash
# Get all chunks
curl http://127.0.0.1:4200/api/chunks

# Update a chunk
curl -X POST http://127.0.0.1:4200/api/chunks/5 \
  -H "Content-Type: application/json" \
  -d '{"text": "Updated dialogue", "instruct": "Excited, bright energy."}'

# Generate audio for single chunk
curl -X POST http://127.0.0.1:4200/api/chunks/5/generate

# Standard batch render (parallel individual calls)
curl -X POST http://127.0.0.1:4200/api/generate_batch \
  -H "Content-Type: application/json" \
  -d '{"indices": [0, 1, 2, 3, 4]}'

# Fast batch render (batched TTS calls, much faster)
curl -X POST http://127.0.0.1:4200/api/generate_batch_fast \
  -H "Content-Type: application/json" \
  -d '{"indices": [0, 1, 2, 3, 4]}'

# Merge all chunks into final audiobook
curl -X POST http://127.0.0.1:4200/api/merge
```

### 已保存脚本
```bash
# List saved scripts
curl http://127.0.0.1:4200/api/scripts

# Save current script
curl -X POST http://127.0.0.1:4200/api/scripts/save \
  -H "Content-Type: application/json" \
  -d '{"name": "my-novel"}'

# Load a saved script
curl -X POST http://127.0.0.1:4200/api/scripts/load \
  -H "Content-Type: application/json" \
  -d '{"name": "my-novel"}'
```

### 角色生成
```bash
# Generate personas (LLM + VoiceDesign, assigns clone voices automatically)
curl -X POST http://127.0.0.1:4200/api/generate_personas

# Generate personas in advanced mode with custom batch size
curl -X POST http://127.0.0.1:4200/api/generate_personas \
  -H "Content-Type: application/json" \
  -d '{"advanced": true, "batch_size": 40}'

# Check persona generation status
curl http://127.0.0.1:4200/api/status/persona

# Cancel persona generation
curl -X POST http://127.0.0.1:4200/api/cancel_persona
```

### 声音设计器
```bash
# Preview a voice from text description
curl -X POST http://127.0.0.1:4200/api/voice_design/preview \
  -H "Content-Type: application/json" \
  -d '{"description": "A warm, deep male voice", "text": "Hello world."}'

# Save a designed voice
curl -X POST http://127.0.0.1:4200/api/voice_design/save \
  -H "Content-Type: application/json" \
  -d '{"name": "warm_narrator", "description": "A warm, deep male voice", "text": "Hello world."}'

# List saved designed voices
curl http://127.0.0.1:4200/api/voice_design/list

# Delete a designed voice
curl -X DELETE http://127.0.0.1:4200/api/voice_design/delete/voice_id_here
```

### LoRA 训练
```bash
# Upload a training dataset (ZIP with WAV + metadata.jsonl)
curl -X POST http://127.0.0.1:4200/api/lora/upload_dataset \
  -F "file=@dataset.zip" -F "name=my_voice"

# Generate a dataset from Voice Designer description
curl -X POST http://127.0.0.1:4200/api/lora/generate_dataset \
  -H "Content-Type: application/json" \
  -d '{"name": "warm_voice", "description": "A warm male voice", "texts": ["Hello.", "Goodbye."]}'

# List uploaded datasets
curl http://127.0.0.1:4200/api/lora/datasets

# Delete a dataset
curl -X DELETE http://127.0.0.1:4200/api/lora/datasets/dataset_id_here

# Start LoRA training
curl -X POST http://127.0.0.1:4200/api/lora/train \
  -H "Content-Type: application/json" \
  -d '{"name": "narrator_warm", "dataset_id": "my_voice", "epochs": 25, "lr": "5e-6", "lora_r": 32, "lora_alpha": 64}'

# Check training status
curl http://127.0.0.1:4200/api/status/lora_training

# List trained adapters
curl http://127.0.0.1:4200/api/lora/models

# Test a trained adapter
curl -X POST http://127.0.0.1:4200/api/lora/test \
  -H "Content-Type: application/json" \
  -d '{"adapter_id": "narrator_warm_1234567890", "text": "Test line.", "instruct": "Calm narration."}'

# Delete an adapter
curl -X DELETE http://127.0.0.1:4200/api/lora/models/adapter_id_here
```

### 数据集构建器
```bash
# List all dataset builder projects
curl http://127.0.0.1:4200/api/dataset_builder/list

# Create a new project
curl -X POST http://127.0.0.1:4200/api/dataset_builder/create \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset"}'

# Update project metadata (description and global seed)
curl -X POST http://127.0.0.1:4200/api/dataset_builder/update_meta \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset", "description": "A warm male narrator", "global_seed": "42"}'

# Update sample rows
curl -X POST http://127.0.0.1:4200/api/dataset_builder/update_rows \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset", "rows": [{"text": "Hello world.", "emotion": "cheerful"}]}'

# Generate a single sample preview
curl -X POST http://127.0.0.1:4200/api/dataset_builder/generate_sample \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset", "description": "A warm male voice", "sample_index": 0, "samples": [{"text": "Hello.", "emotion": "cheerful"}]}'

# Batch generate all samples
curl -X POST http://127.0.0.1:4200/api/dataset_builder/generate_batch \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset", "description": "A warm male voice", "samples": [{"text": "Hello.", "emotion": "cheerful"}]}'

# Check batch generation status
curl http://127.0.0.1:4200/api/dataset_builder/status/my_voice_dataset

# Cancel a running batch generation
curl -X POST http://127.0.0.1:4200/api/dataset_builder/cancel \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset"}'

# Save project as a training dataset
curl -X POST http://127.0.0.1:4200/api/dataset_builder/save \
  -H "Content-Type: application/json" \
  -d '{"name": "my_voice_dataset", "ref_sample_index": 0}'

# Delete a project
curl -X DELETE http://127.0.0.1:4200/api/dataset_builder/my_voice_dataset
```

### 音频下载
```bash
# Download audiobook (after merging in editor)
curl http://127.0.0.1:4200/api/audiobook --output audiobook.mp3

# Export to Audacity (per-speaker tracks + LOF + labels)
curl -X POST http://127.0.0.1:4200/api/export_audacity

# Poll for completion
curl http://127.0.0.1:4200/api/status/audacity_export

# Download the zip
curl http://127.0.0.1:4200/api/export_audacity --output audacity_export.zip
```

---

## 常见问题

### 脚本生成失败
- 确认 LLM 服务器正在运行且可访问
- 验证模型名称与已加载模型一致
- 尝试使用其他模型 — 某些模型在 JSON 输出方面表现不佳
- 思维链模型（DeepSeek-R1、GLM4 等）可能干扰 JSON 输出。如需使用，请在设置中的 **Banned Tokens** 字段添加 `<think>` 以禁用思考模式

### 脚本生成提示 "completed with degradations"（返回码 3）
这不是失败：脚本已写出且没有丢失任何文本，但有部分片段无法被分类，被归属给了 NARRATOR。运行摘要会列出每一个降级的分块。请在编辑器中检查这些条目，或在修复下面的成因后重新运行。

### 说话人大多是 NARRATOR，或 `prompt_tokens` 始终不变
你的 LLM 服务器在静默截断提示。上下文窗口小于提示的服务器不会报错 — 它会丢掉溢出部分并用剩下的内容作答，于是模型被要求分类它根本没见过的片段。终端中的症状：
- 个别分块上出现 `PROMPT LIKELY TRUNCATED BY THE SERVER` 警告
- 运行摘要中出现 `SERVER CONTEXT WINDOW SUSPECT` 一行，报告 `prompt_tokens` 在长度差异极大的提示间被固定为同一个值（Ollama 默认的 `num_ctx=2048` 恰好会产生这种情况）
- 大量重试和降级分块，而整体运行看起来是绿色的

修复方法是提高**服务器**的上下文窗口：
- **Ollama：** 在环境中设置 `OLLAMA_CONTEXT_LENGTH=8192`，把 `num_ctx` 写进模型的 Modelfile，或在 `app/config.json` 中设置 `generation.num_ctx`（会按请求转发给支持该参数的服务器）
- **llama.cpp / vLLM：** 提高 `-c` / `--max-model-len`
- 或者缩减提示：降低 `generation.chunk_size` 或 `generation.max_context_roster_names`

### 模型下载失败或速度很慢
- TTS 模型（每个约 3.5 GB）在首次使用时从 Hugging Face 下载
- **中国大陆用户**：设置环境变量 `HF_ENDPOINT=https://hf-mirror.com` 使用国内镜像
- 如遇速率限制，注册免费 [Hugging Face 账号](https://huggingface.co/join) 并设置 `HF_TOKEN`
- 下载中断后会自动续传 — 重启应用即可

### TTS 生成失败
- 查看 Pinokio 终端中的模型加载错误
- 确保有足够的显存（推荐 16 GB 以上 bfloat16）
- 使用外部模式时，确认 Gradio TTS 服务器正在配置的 URL 上运行
- 检查 voice_config.json 中所有说话人的设置是否有效
- 克隆声音时，确认参考音频存在且转录文本准确

### 生成速度慢
- 在设置中启用 **Compile Codec**（首次预热后速度提升 3-4 倍）
- 如果显存允许，增加 **Parallel Workers**（批量大小）
- 使用 **Batch (Fast)** 渲染模式而非 Standard
- 如果在 AMD 上看到 MIOpen 警告，这些会被自动处理
- 首批生成较慢是正常现象（参见上方"首次启动"部分）

### 显存不足 / OOM 错误
- 在设置中降低 **Max Chars/Batch**（特别是使用克隆/LoRA 声音且参考音频较长时）
- 降低 **Parallel Workers**（批量大小）
- 关闭其他 GPU 密集型应用程序
- 如果仍然不行，尝试 `device: cpu`（速度会慢很多）

### MP3 文件损坏或很小（428 字节）
Conda 自带的 ffmpeg 在 Windows 上通常缺少 MP3 编码器（libmp3lame）。Alexandria 会自动检测并回退到 WAV 格式。如需 MP3 输出：
- 安装带 MP3 支持的 ffmpeg：`conda install -c conda-forge ffmpeg`
- 或移除 conda 的 ffmpeg 以使用系统自带的：`conda remove ffmpeg`
- 验证：`ffmpeg -encoders 2>/dev/null | grep mp3`

### 音质问题
- 克隆时使用 5-15 秒清晰的参考音频
- 避免参考样本中有背景噪音
- 为预置声音尝试不同的种子

### 输出中出现乱码字符
- 系统会自动修复常见的编码问题
- 如果问题仍然存在，请确保输入文本为 UTF-8 编码

### 中文书籍处理提示
- 在设置标签页的 **Language** 下拉菜单中选择"Chinese"或"Auto"
- 默认 LLM 提示是为英文编写的 — 处理中文书籍时，建议在设置标签页的"Prompt Customization"部分修改提示，使其适配中文对话约定（如使用「」引号等）
- 提示文件 `default_prompts.txt` 和 `review_prompts.txt` 可永久修改，更改即时生效无需重启

> **局限：** 片段分词器只识别直双引号和弯双引号（`"` `“` `”`）以及单引号（`'` `‘` `’`）作为对话定界符。书名号式的 guillemets（« »）和 CJK 括号（「」）**不被**识别，因此以这种方式标记的对话目前会被归类为旁白并由旁白朗读。没有文本丢失 — 这是安全的失败方向 — 但在分词器学会这些符号之前，这类书籍无法获得角色声音。自定义提示无法绕过这一点，因为 LLM 只能标注代码交给它的片段。

---

## 推荐 LLM 模型

用于脚本生成，非思维链模型效果最佳：
- **Qwen3-next**（80B-A3B-instruct）— JSON 输出和指令方向优秀
- **Gemma3**（推荐 27B）— JSON 输出和指令方向出色
- **Qwen2.5**（任意大小）— JSON 输出稳定
- **Qwen3**（非思维链变体）
- **Llama 3.1/3.2** — 角色区分能力强
- **Mistral/Mixtral** — 速度快，稳定可靠

**思维链模型**（DeepSeek-R1、GLM4-air 等）可能干扰 JSON 输出。如果必须使用，请在设置中的 **Banned Tokens** 字段添加 `<think>` 以禁用思考模式。

---

## 更多文档

完整文档请参阅：
- [English README](README.md) — 完整英文文档，包含 Python/JavaScript 集成示例、提示自定义、测试和项目结构
- [Wiki](https://github.com/Finrandojin/alexandria-audiobook/wiki) — 详细指南：声音类型、LoRA 训练、批量生成等

## 致谢

- [Ayush Naphade](https://github.com/aayushnaphade) — 角色生成、说话人别名解析和上下文脚本审校功能（[PR #42](https://github.com/Finrandojin/alexandria-audiobook/pull/42)）。欢迎访问他的项目 [Lily](https://lily.rayoneai.in/)！
- [Michii](https://github.com/on22s) — 系统健康仪表板，实时GPU/磁盘监控（[PR #45](https://github.com/Finrandojin/alexandria-audiobook/pull/45)）、跨平台子进程运行器（[PR #46](https://github.com/Finrandojin/alexandria-audiobook/pull/46)）、语音训练数据集准备标签页（[PR #47](https://github.com/Finrandojin/alexandria-audiobook/pull/47)）

## 许可证

MIT

### 第三方许可证

- [qwen_tts](https://github.com/Qwen/Qwen3-TTS) — Apache License 2.0，版权归阿里巴巴通义千问团队所有
