# 02 · 苹果端侧 AI 技术栈

### 1. Core ML 的基本工作原理是什么？一个训练好的模型是怎么跑到 App 里的？

要点：

- Core ML 是苹果的**端侧推理框架**，只负责"跑模型"，不负责训练。训练通常在 PyTorch / TensorFlow 里完成。
- 训练好的模型需要通过 `coremltools`（Python 库）转换成 `.mlmodel` / `.mlpackage` 格式，转换过程会做算子映射、量化等优化。
- `.mlpackage` 拖进 Xcode 工程后，Xcode 会在编译期生成一个强类型的 Swift/Objective-C 接口类（类名和模型文件名一致），调用时直接像调用普通对象一样传入输入、拿到输出，不需要手写张量操作。
- 运行时，Core ML 会根据 `MLModelConfiguration.computeUnits` 的设置，把计算调度到 CPU、GPU 或 Neural Engine（ANE）上执行。

一个典型链路：`PyTorch 模型 → coremltools 转换 → .mlpackage → Xcode 生成接口 → MLModel 加载 → 输入预处理（MLFeatureProvider / MLMultiArray）→ 推理 → 输出解析`。

面试里常被追问的点：**为什么要在端侧跑模型**——低延迟（不用等网络往返）、离线可用、隐私（数据不用上传）、省服务器成本；代价是**模型体积占用包大小**、**设备算力有上限**（复杂模型跑不动或很慢）、**升级模型需要发版**（除非用 Core ML 的模型云端下发机制）。

### 2. Apple Intelligence 和 Foundation Models framework 是什么？和调用云端大模型 API（比如接 GPT/Claude 的接口）有什么区别？

要点：

- **Apple Intelligence** 是苹果从 iOS 18 开始推出的系统级 AI 能力集合（写作工具、图像生成、Siri 增强等），运行在苹果自己的端侧模型 + 私有云计算（Private Cloud Compute）之上。
- **Foundation Models framework**（WWDC 2025 起提供）是开放给第三方开发者的 API，可以直接调用**驱动 Apple Intelligence 的那个约 30 亿参数端侧语言模型**，用几行 Swift 代码就能在 App 里加入文本摘要、内容生成、结构化数据抽取等能力：

```swift
import FoundationModels

let session = LanguageModelSession()
let response = try await session.respond(
    to: "帮我把这段用户反馈总结成一句话：\(feedbackText)"
)
```

- 和直接调用云端大模型 API 相比，核心区别在于：

| 维度 | Foundation Models（端侧） | 云端大模型 API（如 GPT / Claude） |
| --- | --- | --- |
| 网络依赖 | 不需要，离线可用 | 必须联网 |
| 隐私 | 数据不出设备 | 数据发送到服务商 |
| 成本 | 系统自带，无调用费用 | 按 token 计费 |
| 能力上限 | 模型规模小，适合摘要/分类/结构化抽取等轻量任务 | 能力更强，适合复杂推理、长文写作、多轮深度对话 |
| 可用性前提 | 需要用户开启 Apple Intelligence、设备/系统版本达标 | 只要能联网基本都可用 |

- 实际项目里两者常常是互补关系：轻量、高频、对隐私敏感的任务交给 Foundation Models 做端侧处理；复杂推理或需要联网检索最新信息的任务走云端 API。面试时能讲清楚"什么任务该放在端侧、什么该走云端"，比单纯知道这两个名词更有价值。

### 3. Neural Engine 是什么？Core ML 的 Compute Units 应该怎么选？

要点：

- **Neural Engine（ANE）**是苹果芯片里专门为机器学习推理设计的硬件单元，针对矩阵乘法、卷积等运算做了专门优化，能效比远高于用 CPU/GPU 硬跑。
- `MLModelConfiguration.computeUnits` 可选：`.cpuOnly`、`.cpuAndGPU`、`.cpuAndNeuralEngine`、`.all`（默认，让系统自动选择最优组合）。
- 大部分场景直接用默认的 `.all` 即可，系统会根据模型算子支持情况和当前设备负载自动调度；只有在需要**避免和 GPU 渲染任务抢资源**（比如一个正在用 GPU 做复杂界面渲染的 App，同时要跑模型）或者**明确知道某些自定义算子 ANE 不支持、指定 CPU/GPU 反而更快**时，才需要手动指定。
- 排查性能问题时，可以用 Xcode 的 Core ML Performance Report（或 Instruments 的 Core ML 模板）查看模型每一层实际被调度到了哪个计算单元，如果发现关键层被"退化"到 CPU 执行，通常是某个算子不被 ANE 支持，需要在模型转换阶段调整。

### 4. 用 Vision + Core ML 做图像识别，常见的面试考点有哪些？

要点：

- **输入预处理是最容易出错也最常被追问的环节**：模型训练时用的图像归一化方式（像素值范围、均值方差）、resize 策略（是否保持长宽比、用什么插值）必须和推理时完全一致，否则准确率会明显下降，且这种错误编译不会报错，只会"结果不准"，排查成本很高。
- `Vision` 框架提供了 `VNCoreMLRequest`，可以直接接收 `CVPixelBuffer` 甚至相机实时帧，内部帮你处理好坐标系转换、裁剪缩放，比手动拼 `MLMultiArray` 更不容易出错，实际项目里优先使用 Vision 而不是直接手搓输入张量。
- 性能瓶颈通常出在**预处理阶段的 CPU 拷贝/转换**，而不是模型推理本身，尤其是实时摄像头场景（比如扫码、实时滤镜），要注意 `CVPixelBuffer` 的格式转换、避免不必要的内存拷贝，必要时用 `CVPixelBufferPool` 复用缓冲区。
- 实时场景要考虑推理帧率和界面帧率不匹配的问题：模型推理耗时可能超过一帧（16.7ms @ 60fps），常见做法是把推理放在单独的串行队列，做丢帧/节流处理，而不是每一帧都同步等待推理结果。

### 5. MLX 是什么？和 Core ML 是什么关系？

要点：

- **MLX** 是苹果开源的机器学习**数组计算框架**（类似 NumPy/PyTorch 的定位），专为 Apple Silicon 的统一内存架构设计，主要面向**研究、训练、以及在 Mac 上做大模型的本地推理/微调实验**。
- **Core ML** 是面向 App 集成的**推理部署框架**，重点是把训练好的模型高效、稳定地跑在 iPhone/iPad/Apple Watch 上，和 App 的生命周期、系统资源调度深度集成。
- 简单说：MLX 更偏"训练/实验"，目前主要活跃在 Mac 端跑大模型场景（比如本地跑量化过的 Llama/Mistral 类模型做实验）；Core ML 更偏"部署/生产"，是目前唯一官方支持、可以直接上架 App Store 的 iOS 端侧推理方案。两者不是竞争关系，面试时能分清"部署到 iPhone 用 Core ML，本地搞模型实验用 MLX"这个定位差异即可，不需要精通 MLX 的具体 API。

### 6. App Intents 结合 Apple Intelligence（比如让 Siri 调用 App 内某个功能）背后的原理是什么？

要点：

- `App Intents` 框架让开发者把 App 内的具体操作（比如"创建一条待办""发送一条消息"）声明成系统可发现、可调用的"意图"（`AppIntent`），系统级的 Siri、Shortcuts、Spotlight 都可以直接调用这些意图，而不需要打开 App 界面。
- 从 iOS 18 起，Apple Intelligence 增强了 Siri 对自然语言的理解能力，用户可以用更口语化、跨 App 的方式表达需求（比如"把刚才那张照片发给张三"），系统负责把这句话拆解、匹配到具体某个 App 声明的 `AppIntent` 并传入合适的参数，App 本身不需要自己做自然语言理解。
- 对开发者来说，这意味着接入门槛主要在**把功能正确地声明成结构化的 Intent（参数类型、`ParameterSummary`、可发现性配置）**，语言理解这一层完全由系统承担；面试里如果被问"如何让 App 支持 Siri 语音操作"，回答的重点应该落在 `AppIntent` 的声明和参数设计上，而不是自己去做语音识别或 NLP。

### 7. 端侧模型 vs 云端大模型，实际项目里怎么取舍？

要点（可以直接当作决策清单）：

- **延迟敏感、需要离线可用**（比如相机实时处理、无网环境下的基础对话）→ 优先端侧（Core ML / Foundation Models）。
- **涉及用户隐私数据、且合规要求数据不出设备**（比如健康、财务类内容摘要）→ 优先端侧。
- **任务复杂度高、需要强推理或长上下文**（比如复杂的多轮咨询、代码生成）→ 云端大模型更合适，端侧小模型能力不够。
- **需要联网获取实时信息**（比如查天气、查最新资讯再回答）→ 必须走云端 API 或带检索能力的方案。
- **成本敏感、调用量巨大**→ 能用端侧处理的部分尽量放端侧，减少云端 token 消耗。

实际产品里这两者常常组合使用，比如先用端侧模型做一次轻量分类/过滤，只有需要深度处理的请求才转发到云端，这也是目前不少 AI 类 App 在成本和体验之间做平衡的常见架构。面试时能给出类似这样有具体依据的取舍逻辑，比"看情况"这种回答更有说服力。
