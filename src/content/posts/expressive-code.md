---
title: 使用Ollama在本地跑大模型
published: 2026-04-10
description: 本文详细讲解Ollama的安装、模型下载、对话交互、API调用及私有化部署全过程，包含命令行示例、Modelfile定制、GPU加速配置与常见问题排查，帮助开发者在自己的机器上低成本运行Llama 3、Mistral等开源大模型。
tags: [Ollama]
category: 'Ollama'
draft: false
---
# 从零开始，用Ollama在本地跑大模型

> 本文详细讲解Ollama的安装、模型下载、对话交互、API调用及私有化部署全过程，包含命令行示例、Modelfile定制、GPU加速配置与常见问题排查，帮助开发者在自己的机器上低成本运行Llama 3、Mistral等开源大模型。

## 一、为什么你需要 Ollama？

大语言模型（LLM）正在重塑我们与代码、文档和知识打交道的方式。但云端 API 总是伴随着网络延迟、数据隐私和成本焦虑。如果你只是想在自己的笔记本或者一台带 GPU 的服务器上跑一个能写代码、能翻译、能总结文档的模型，Ollama 是目前最简单、最省心的选择。

它把模型权重、运行环境、推理接口打包成一个单一命令行工具，让你用一条命令就能下载并运行 Llama 3、Mistral、Gemma、Phi 等几十种主流开源模型。更妙的是，它对 macOS、Linux 和 Windows 都提供了原生支持，还能自动检测并利用 Apple Silicon 的 GPU、NVIDIA CUDA 或 AMD ROCm 加速。

本文将从零开始带你完成安装、部署、对话、API 集成与自定义模型，最后还会给出生产环境下的优化建议。

## 二、安装 Ollama

### macOS & Windows

最简单的方式是去 [ollama.com](https://ollama.com) 下载图形化安装包。双击安装后，任务栏会出现一个羊驼图标，说明服务已后台运行。终端里就能直接使用 `ollama` 命令。

Windows 上也支持通过 WSL 2 使用 Linux 版。Ollama 会默认启用 GPU 加速（NVIDIA CUDA），安装包会自动处理驱动依赖。

### Linux

在 Linux 上，使用官方脚本一键安装：

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

该脚本会添加 Ollama 的 APT/YUM 源，安装 `ollama` 二进制与 systemd 服务。安装完成后，服务会自启动并监听 `11434` 端口。你可以用以下命令确认状态：

```bash
systemctl status ollama
```

如果不想使用 systemd，也可以手动启动：

```bash
ollama serve
```

### Docker 部署

如果你更习惯容器化环境，官方提供了 Docker 镜像：

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

需要 GPU 加速时，安装 `nvidia-container-toolkit` 并添加 `--gpus all` 参数。对于 AMD GPU，可以拉取 `ollama/ollama:rocm` 镜像。

## 三、下载并运行你的第一个模型

Ollama 的模型库托管在 ollama.com/library，涵盖各类开源模型。常用的几个模型如下：

| 模型名称 | 参数规模 | 适用场景 | 显存要求（4-bit量化） |
|----------|---------|-----------|----------------------|
| llama3.1 | 8B | 通用对话、写作、代码生成 | ~6 GB |
| llama3.1:70b | 70B | 复杂推理、长文本理解 | ~40 GB |
| mistral | 7B | 多语言、轻量级高效推理 | ~5 GB |
| gemma2 | 9B / 27B | 谷歌出品，知识密集型任务 | ~6 GB / ~18 GB |
| codellama | 7B / 13B / 34B | 代码补全与生成 | ~5 GB / ~9 GB / ~22 GB |
| phi3:mini | 3.8B | 小型设备，快速响应 | ~3 GB |
| qwen2:0.5b | 0.5B | 嵌入式计算、玩具项目 | < 1 GB |

运行模型的命令非常简单，比如下载并启动 Llama 3.1 8B：

```bash
ollama run llama3.1
```

第一次运行时会自动下载模型权重，之后就会进入交互式对话界面。你可以直接输入问题，模型会以流式方式返回结果。

**常用操作：**

- 退出对话：输入 `/bye`
- 查看当前模型：`/show`
- 清空上下文：`/clear`
- 多行输入：使用 `"""` 包围文本

Ollama 默认使用 2048 token 的上下文窗口，如果需要更长的记忆，可以用 `ollama run llama3.1` 时通过 `num_ctx` 参数指定，或者在后续 Modelfile 中设置。

### 指定量化级别与标签

很多模型提供不同量化精度的标签，例如 `llama3.1:8b-instruct-q4_K_M`。如果不指定标签，Ollama 会拉取默认的推荐版本（通常为 Q4_K_M）。对于极致追求速度，可用 `q4_0`；对于追求质量并显存充足，可用 `q8_0` 或 `q6_K`。查看具体标签可以在模型页面点选“Tags”下拉列表。

## 四、用 API 将 Ollama 集成到你的应用

Ollama 启动后，除了命令行对话，还会在本地开启一个兼容 OpenAI 风格的 HTTP API。这使得你可以用任何编程语言来调用模型，甚至替换 OpenAI SDK 的 endpoint。

### 基础接口

**生成对话补全（Chat Completions）**

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.1",
  "messages": [
    {"role": "user", "content": "用Python写一个快速排序"}
  ],
  "stream": false
}'
```

返回 JSON 中包含 `message.content` 字段。

**通用生成（Generate）**

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1",
  "prompt": "你好，请用中文介绍你自己",
  "stream": false
}'
```

如果想实现打字机效果，保留 `"stream": true` 即可逐 token 返回。

### OpenAI 兼容端点

从 0.1.24 版本开始，Ollama 提供了一个 `/v1/` 前缀的 OpenAI 兼容接口。你可以直接使用 OpenAI Python/JS 库，只需将 `base_url` 指向本地：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="ollama"  # 任意非空字符串即可
)

response = client.chat.completions.create(
    model="llama3.1",
    messages=[{"role": "user", "content": "解释一下量子纠缠"}]
)
print(response.choices[0].message.content)
```

通过这种方式，任何支持 OpenAI API 的工具（比如 LangChain、LlamaIndex、 Continue 插件、代码助手）都可以直接连接到本地模型，无需修改大量代码。

### REST API 一览

| 端点 | 功能 |
|------|------|
| POST /api/generate | 纯文本生成 |
| POST /api/chat | 多轮对话生成 |
| POST /api/create | 从 Modelfile 创建新模型 |
| GET /api/tags | 列出本地已有模型 |
| DELETE /api/delete | 删除模型 |
| POST /api/pull | 下载模型 |
| POST /api/embed | 生成文本嵌入向量 |

## 五、管理与自定义模型

### 查看、复制与删除

```bash
ollama list          # 列出本地所有模型
ollama rm llama3.1  # 删除模型
ollama cp llama3.1 my-llama  # 复制模型副本（用于试验不同参数）
```

### 使用 Modelfile 定制模型

Modelfile 相当于 Dockerfile，让你在已有的模型基础上调整系统提示、温度、惩罚参数、上下文长度，甚至引入 GGUF 格式的自定义权重。

创建一个文件 `Modelfile`：

```dockerfile
FROM llama3.1

# 设置系统提示词
SYSTEM "你是一个精通Python和Rust的高级软件工程师。回答要简洁，使用中文。除非要求，不要给代码示例。"

# 调整参数
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 4096

# 禁用某些内置安全过滤（仅限合规用途）
PARAMETER repeat_penalty 1.1
```

然后执行：

```bash
ollama create my-coder -f ./Modelfile
ollama run my-coder
```

现在 `my-coder` 会按你设定的风格回答。你可以随时继续修改 Modelfile 并重新 `create` 来迭代。

### 导入自定义 GGUF 模型

许多社区微调模型都以 GGUF 格式发布。假设你下载了 `qwen2-math-7b.Q4_K_M.gguf`，先写一个不含 `FROM` 指令的 Modelfile（或者 `FROM /path/to/file`）：

```dockerfile
FROM ./qwen2-math-7b.Q4_K_M.gguf
```

然后 `ollama create qwen-math -f Modelfile` 即可。Ollama 会自动推断架构并完成量化适配。如果模型包含多个分片文件（如 `.gguf` 分段），FROM 指令也可以指向目录。

## 六、GPU 加速与性能调优

### 检查 GPU 是否被使用

运行模型时，Ollama 日志会打印 “blas=1” 以及 CUDA 或 Metal 相关信息。在 macOS 上，活动监视器可以看到 GPU 使用率飙升；在 Linux 上可用 `nvidia-smi` 监视显存占用。

若要强制使用某块 GPU，可以设置环境变量，比如 Linux 上：

```bash
export CUDA_VISIBLE_DEVICES=0
ollama run llama3.1
```

### 多 GPU 与分布式

Ollama 原生支持模型分层放置在多张 GPU 上。例如 70B 模型在家用两张 RTX 3090（24GB）上时，Ollama 会自动将层平均分配到两卡。不需要额外配置文件。

### 并发请求与队列

默认情况下，Ollama 会串行处理请求。如果需要并行，可以通过以下环境变量启动服务来启用队列和并行工作器：

```bash
OLLAMA_NUM_PARALLEL=4 OLLAMA_MAX_LOADED_MODELS=2 ollama serve
```

- `OLLAMA_NUM_PARALLEL`：每个模型允许的最大并发请求数
- `OLLAMA_MAX_LOADED_MODELS`：同时驻留在显存里的模型数量（超出会触发 LRU 卸载）

当多个请求同时到达且模型未完全加载到内存时，Ollama 会排队等待。配合适当的值可以提升吞吐量，但要注意显存总量。

### 上下文长度与显存

上下文长度（`num_ctx`）对显存的消耗非常显著。例如 LLaMA 3.1 8B 在 4096 tokens 时占用约 6 GB，拉到 32k tokens 可能膨胀到 14 GB 以上。建议根据任务需求设定，并在 Modelfile 中固化为 `PARAMETER num_ctx 8192`。

### 量化参数对比表

不同量化等级的效果与速度差异大致如下（以 LLaMA 3.1 8B 为例）：

| 量化标签 | 模型大小 | 困惑度上升 | 推理速度 | 推荐场景 |
|----------|---------|------------|---------|----------|
| q4_0 | 4.9 GB | 较高 | 极快 | 资源极度受限 |
| q4_K_M | 5.3 GB | 中等 | 快 | **默认推荐** |
| q5_K_M | 6.2 GB | 较低 | 较快 | 对质量敏感 |
| q8_0 | 8.5 GB | 低 | 中 | 近无损效果 |
| fp16 | 16 GB | 无损 | 慢（尤其非GPU） | 微调基座 |

## 七、嵌入模型与高级功能

Ollama 不仅支持对话模型，还支持生成文本嵌入向量的专用模型，如 `nomic-embed-text`、`mxbai-embed-large`。这对 RAG（检索增强生成）应用非常重要。

下载并生成嵌入：

```bash
ollama pull nomic-embed-text
curl http://localhost:11434/api/embed -d '{
  "model": "nomic-embed-text",
  "input": "你好，世界"
}'
```

返回的 embedding 可直接存入向量数据库（Chroma、Qdrant 等）。与 LangChain 集成时，可使用 `OllamaEmbeddings` 类。

## 八、生产环境部署建议

1. **使用 systemd 或 Docker 保持服务常驻**  
   服务崩溃后需要自动重启，Linux 下 systemd 的 `Restart=on-failure` 是最简单的方案。Docker 使用 `--restart=unless-stopped`。

2. **前置反向代理**  
   用 Nginx 或 Caddy 反代 `11434` 端口并添加 HTTPS，保护内部 API 不被非授权访问。注意做好请求频率限制。

3. **监控与日志**  
   Ollama 会将请求日志输出到 journald。可搭配 Prometheus + Loki 进行指标收集与日志分析。未来版本可能直接暴露 metrics 端点。

4. **模型热加载**  
   通过 `OLLAMA_MAX_LOADED_MODELS` 让常用模型常驻显存，减少卸载-加载的等待时间。

5. **权限控制**  
   若需暴露到公网，务必在前面加一层认证中间件。Ollama 本身没有鉴权，任何能访问 11434 端口的人都能调用模型。

6. **定期更新**  
   模型和运行时都会不断更新。执行 `ollama pull <model>` 可以拉取最新量化版本；更新 Ollama 本身只需重新执行安装脚本或替换二进制文件，模型数据不受影响。

## 九、常见问题排查

**Q：运行模型时提示 “CUDA error: out of memory”**

减少上下文长度 `num_ctx`，或者换用更小量化标签。如果仍超显存，考虑使用 `OLLAMA_NUM_GPU` 限制使用的 GPU 层数。例如只用 20 层 GPU，其余放入 CPU：

```bash
ollama run llama3.1:70b --num-gpu 20
```

**Q：Mac 上运行速度慢**

确认是 Apple Silicon 芯片（M1/M2/M3），并且模型大小不超过内存限制。Ollama 自动使用 Metal 加速。如果内存接近耗尽，macOS 会大量使用 swap，导致速度急剧下降。使用 `ollama ps` 查看模型占用。

**Q：Windows 上无法找到 GPU**

确保已安装最新的 NVIDIA 驱动，Ollama 使用 WSL 2 运行时需要安装 CUDA Toolkit 在 WSL 内（一般通过官网安装脚本即可）。或者使用原生 Windows 版本直接调用 DirectML。

**Q：模型响应被截断或出现乱码**

可能是指定了不支持的 `stop` 参数或特殊 token 设置。先用默认参数测试，再逐步调整。也可以在 Modelfile 中设置 `TEMPLATE` 以适配不同模型的对话格式。

## 十、总结

Ollama 成功地把大模型本地部署的门槛降到了和安装普通软件一样低。无论是个人开发者想在本地跑一个私有的编程助手，还是团队需要自托管一个可嵌入产品的推理服务，Ollama 都提供了命令行、API、Docker 以及丰富的模型生态系统。

下一步你可以：用它搭建一个私有的 ChatGPT 替代、嵌入到 Obsidian/Logseq 做笔记问答、用 Continue 插件在 VS Code 里代码补全，或是结合 LangChain 构建自己的知识库问答系统。当一切都跑在自己机器上，数据主权和零成本调用就不再是口号，而是你手边的日常工具。
