# vLLM Semantic Router 学习笔记 07：模型提供商分类体系与苹果 MLX 对接真相

> **源码事实源**：
> - `dashboard/frontend/src/pages/ConfigPageConnectModelsDialogView.tsx` (添加模型弹窗)
> - `dashboard/frontend/src/pages/modelProviderCatalog.ts` (三大预设分类逻辑)
> - `website/static/model-catalog/catalog.json` (官方全量 Providers 字典)
> **锚定版本**：v0.4.0 (Git Commit: `ad2bf68ae`, 2026-10-05)

---

## 一、弹窗三大分类体系与搞笑机翻真相

弹窗将所有模型接入点清晰划分为三大区域（`providerCategories`）：
1. **Start here（从这里开始 / 主流自建引擎）**
2. **Model APIs（商业模型云端 API）**
3. **Private runtimes（私有 / 本地运行时，机翻为“私人运行时间”）**

### 1. 破案：“AMD 原子” 与 “SG 朗” 是什么？
- **“AMD 原子”**：英文原名 **`AMD ATOM`**，是 AMD 为自己的 ROCm 显卡生态开发的推理优化引擎。浏览器整页翻译按字面把 ATOM 翻成了“原子”。
- **“SG 朗”**：英文原名 **`SGLang`**，由加州大学伯克利 LMSYS 团队开源的超快大模型推理框架。浏览器把 SGLang 拆开硬译成了“SG 朗”。
- **为什么把它们放在第一栏？**
  因为全球自建推理 GPU 算力中，**vLLM、SGLang、AMD ATOM、OpenAI Compatible** 是统治级的四大通用框架。

---

## 二、疑惑解答：“OpenAI 兼容”与“商业模型 API”重复吗？

### 答：完全不重复，是“万能白板”与“快捷通道”的精妙分工！

1. **第一栏的“OpenAI 兼容（OpenAI Compatible）” $\to$ 万能白板**：
   - 它是给**任何第三方、自建或开源反向代理**准备的通用协议转接头；
   - 没有任何默认地址，你需要自己填 `Base URL`（如 `http://192.168.0.10:8000/v1`）和私有 Key。

2. **第二栏的“商业模型 API（Model APIs，如 DeepSeek、通义千问、OpenAI）” $\to$ 官方快捷通道**：
   - 系统已经内置了各大厂商的官方根地址（比如 DeepSeek 的 `https://api.deepseek.com/v1`）；
   - 你只要粘贴对应厂商的 Key，系统甚至能直接调用官方 `list_models` 接口，**自动把该厂商最新的模型清单拉下来让你点选**，免去用户手动查文档敲模型名字的麻烦！

---

## 三、私有运行时与苹果 MLX 的接入真相

师弟提出了极其敏锐的问题：
> **“私人运行时里有 Docker、LM Studio、Ollama，那我们最常用的苹果 MLX 呢？”**

### 1. 私有运行时（Private runtimes）各有什么角色？
- **Ollama**：最普及的终端轻量一键运行器，默认监听 `http://localhost:11434/v1`；
- **LM Studio**：带漂亮 GUI 的桌面大模型运行器，默认监听 `http://localhost:1234/v1`；
- **Docker Model Runner**：企业容器化镜像打包运行的模型；
- **Xinference**：国内高并发多卡分布式推理调度器。

### 2. 苹果生态核心：MLX 该怎么接？
苹果官方的深度学习推理框架是 **`mlx-lm`**（由 Apple 机器学习研发团队专门为 Apple Silicon M 系列芯片深度优化）：
- 当我们在 Mac mini（NE1）或 Mac 本机通过命令启动 MLX 服务时：
  ```bash
  python3 -m mlx_lm.server --model mlx-community/DeepSeek-V2-Lite-Chat-4bit --port 8080
  ```
- **MLX 原生直接提供标准的 OpenAI 兼容接口（`/v1/chat/completions`）！**
- **结论**：
  在 vLLM 或任何网关中连接苹果 MLX，**直接选择第一栏的【OpenAI 兼容】**，地址填 `http://localhost:8080/v1`，即可 100% 享受苹果 M 系列芯片统一内存的高速推理！
  此外，桌面软件 **LM Studio** 在 Mac 上的后端也支持调用 MLX 引擎。

---

## 四、对我们小毛驴架构的落地启示

1. **模型录入分层设计极其优秀**：
   - 学习其“快捷厂商预设（填 Key 自动拉列表）”+“通用自定义白板（自填 URL）”的双轨制，既满足小白极速开通云端模型，又满足自建算力灵活对接；
2. **打通苹果 MLX 原生标准**：
   - 我们的 Mac mini M4 算力机未来跑本地开源模型时，统一通过 `mlx-lm.server` 输出 OpenAI 规范接口，接入成本为零。
