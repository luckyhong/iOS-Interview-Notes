# iOS Interview Notes · iOS 面试复习笔记

一份持续更新的 iOS 面试知识库：既保留 Runtime、RunLoop、内存管理、多线程这些经典的"硬核八股"，也补充了 **AI 时代 iOS 工程师会被实际问到的新问题**——怎么和 AI 工具协作、怎么把大模型能力做进 App、以及"AI 会不会取代 iOS 开发"这种面试官越来越爱问的软性问题。

无论你是正在准备面试的候选人，还是在设计面试题的面试官，都可以把这里当一份可以直接查阅的参考资料。所有答案提供的是一种思路参考，不追求"标准答案"式的长篇论证，欢迎指正和提 PR 补充。

## 目录导航

目录按"从语言基础到工程实践，再到 AI 时代新增内容"的顺序编号，可以按顺序系统复习，也可以直接跳到需要的模块查阅。

| 编号 | 模块 | 内容 |
| --- | --- | --- |
| 01 | [语言基础](01-language-fundamentals) | Swift 语言特性高频问题 |
| 02 | [Runtime 与 RunLoop](02-runtime-and-runloop) | 消息发送、方法交换、动态特性、RunLoop 与线程、Mode、消息传递方式对比 |
| 03 | [内存与并发](03-memory-and-concurrency) | ARC、循环引用、内存布局、GCD、线程安全、锁、Swift Concurrency |
| 04 | [性能优化](04-performance-optimization) | 启动优化、卡顿、包体积、Instruments |
| 05 | [架构与设计模式](05-architecture-and-design-patterns) | MVC/MVVM、常见设计模式在 iOS 中的应用 |
| 06 | [算法](06-algorithms) | 面试高频算法题整理 |
| 07 | [AI 时代的 iOS 开发](07-ai-era-ios-development) | AI 编程工具协作、苹果端侧 AI 技术栈、大模型能力集成、职业发展与面试软实力 |
| — | [附录](appendix) | 《招聘一个靠谱的 iOS》经典题集参考答案、配套底层原理讲义（PPT/PDF） |

### 🤖 关于「AI 时代的 iOS 开发」

这是本仓库新增、也是更新最频繁的模块，按面试里被问到的先后顺序拆成四篇：

| 模块 | 内容 | 面试官想看什么 |
| --- | --- | --- |
| [01·AI 编程工具与研发效率](07-ai-era-ios-development/01-ai-coding-tools-and-productivity.md) | Cursor / Xcode 内建补全 / Claude Code / Copilot 怎么用、AI 生成代码的坑 | 你是否理性、高效地使用工具，而不是盲目依赖 |
| [02·苹果端侧 AI 技术栈](07-ai-era-ios-development/02-apple-on-device-ai-stack.md) | Core ML、Apple Intelligence、Foundation Models framework、Neural Engine | 你对苹果自家 AI 生态的了解深度 |
| [03·大模型能力集成到 iOS App](07-ai-era-ios-development/03-llm-integration-in-ios-apps.md) | 流式响应、上下文管理、多模态输入、安全性、Function Calling | 你能不能把 AI 特性真正落地成可用的产品功能 |
| [04·职业发展与面试软实力](07-ai-era-ios-development/04-career-and-interview-soft-skills.md) | "AI 会不会取代 iOS 开发"、简历怎么写、传统八股是否还重要 | 你对行业变化的判断力和表达能力 |

详细说明见 [07-ai-era-ios-development/README.md](07-ai-era-ios-development/README.md)。

## 怎么用这份笔记

- **临阵磨枪**：直接打开对应编号的目录，从上到下过一遍问题列表，先看要点，答不上来的再展开看详细解析。
- **系统复习**：按 01 → 07 的顺序过一遍，最后补一遍附录里的经典题集，查漏补缺。
- **面试官选题**：可以直接从每个模块里挑题目，07 模块的"面试官想看什么"一列，也可以直接拿来判断一道题是想考技术深度还是判断力。

## 项目结构

```
iOS-Interview-Notes/
├── 01-language-fundamentals/          # Swift 语言基础
├── 02-runtime-and-runloop/            # Runtime、RunLoop、消息传递
├── 03-memory-and-concurrency/         # 内存管理、多线程与并发
├── 04-performance-optimization/       # 性能优化
├── 05-architecture-and-design-patterns/  # 架构与设计模式
├── 06-algorithms/                     # 高频算法
├── 07-ai-era-ios-development/         # AI 时代的 iOS 开发（新增）
└── appendix/                          # 经典题集参考答案 + 配套讲义
```

目录用编号 + 英文短横线命名，方便在任意系统/终端/链接里稳定引用；每个目录内的文档标题、问答内容均为中文。

## 贡献指南

欢迎通过 PR 补充新问题、修正现有答案里的错误，或者针对某个答案继续深入解析。提交时请遵循：

1. **格式**：每个知识点用 `### 序号. 问题` 作为标题，先给可以直接记住的要点，再展开说明，涉及代码的补一段可运行示例；每个目录下的正文文件以一级标题 `# 模块名` 开头。
2. **命名**：目录和文件名使用英文短横线命名（如 `03-memory-and-concurrency`），内容本身仍使用中文，保持链接在任意环境下都稳定可用。
3. **新增模块**：优先在已有目录下新建 Markdown 文件；如果是全新主题，新建目录时延续现有的编号规则，并同步更新本 README 的目录导航表。
4. **AI 相关内容**：涉及较新的系统 API 或工具（尤其是 07 模块）时，注明适用的最低系统版本或发布时间，避免读者踩版本/时效性的坑。

## License

见 [LICENSE](LICENSE)。
