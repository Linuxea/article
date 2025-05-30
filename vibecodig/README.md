# 氛围编程入门

## Vibe Coding 含义与特点

Vibe coding （氛围编程，我认为是快乐编程）是由 AI 大神 Andrej Karpathy，在2025年2月的时候提出的。

氛围编程是开发者最大限度依赖 AI，通过自然语言（文本与语音）与大语言模型进行交互，指导特别是在代码领域优化过的大语言模型来生成代码。这种新型编程方式转变了程度员的角色，从手动编写代码到指导 AI 编程，测试输出，重定义结果。

vibe coding 与一般的 AI 辅助工具相比，具有以下几个方面的特点：

- 编程普及性：它显著降低了初学者或非程序员自动化任务或构建自定义工具的门槛。
- 快速原型制作：它极大地加速了将想法转化为功能原型的过程，实现了更快的迭代。
- 学习工具：通过观察和与AI生成的代码互动，它可以帮助程序员学习新的语言、框架或技术。
- 效率：开发者可以专注于应用架构和功能设计等更高层次的方面，而AI则处理大部分实现细节。

## 氛围编程的聊天室实践

今天，我们借助 github copilot 的 agent ，通过一个示例来完整地体验氛围编程。

这是我们的初始提示词

> - 后端使用 golang + websocket 实现聊天室功能，并且支持广播。
> - 前端需要首先输入唯一的用户名。进行聊天界面后，左边是聊天室列表，点击某个聊天室可以进入该聊天室，有一个发言框，可以往该聊天室发送消息，同时有一个广播的按钮，当点亮时，会向所有的聊天室发送消息。

!['1.png'](1.png '1.png')

首先，AI 进行了需求的梳理，然后创建了前后端各自的基础目录：
!['3.png'](3.png '3.png')

其次，AI 创建了后端的 main.go 文件，并完成了 websocket 代码的编写：
!['4.png'](4.png '4.png')

接下来，AI 开始编写前端的 react 基础文件，同时提示我们 AI agent 已经工作一小会，提示我们是否继续它的需求迭代工作：
!['5.png'](5.png '5.png')

继续完成前端业务代码的编写：
!['6.png'](6.png '6.png')

继续完成前端 CSS 样式的编写：
!['7.png'](7.png '7.png')


基础代码已经完成。
但是看到后端代码出现了异常，选中异常信息直接向 AI 提问，得到处理方法，解决这个异常：
!['8.png'](8.png '8.png')

询问 npm install 一直停留在转圈的原因，AI 给出了解释，并且它想要重新运行以查看更加详细的日志，因为后续又重新正常下载，所以原因可能是 AI 提到的网络问题：
!['9.png'](9.png '9.png')


执行 npm start 控制台提示异常：

```bash
Could not find a required file.
Name: index.html
Searched in: D:code\vibecoding-chatroomfrontendpublic
```

直接把异常信息转发的 AI，让它马上修复：
!['10.png'](10.png '10.png')


代码全部完成，启动无异常，效果演示：
!['11.png'](11.png '11.png')
!['12.png'](12.png '12.png')
!['13.png'](13.png '13.png')

甚至还贴心地整理了 README.md
!['README.png'](README.png 'README.png')

最后，让它再帮助我们初始化git仓库，并提交代码到远程仓库：
!['16.png'](16.png '16.png')


从头到尾只需要通过自然语言与 AI 交互，氛围编程的一大特点是对 AI 的几乎完全信任，对于 AI 的输出，我们只需要不停地点击 Accept，出现问题又反馈给 AI，以此循环反复，直到最终成果产出。

## 开发者启示

通过一个完整的示例，对氛围编程有了初步的了解。从一个开发者角度来说，或者就是：

- 技能重心转移： 开发者需要更加擅长需求描述（Prompt Engineering）、系统设计、AI 输出的评估与验证、以及高层次的决策。纯粹的编码技能相对重要性可能下降，但理解代码和系统原理仍然至关重要。

- 学习新工具和范式： 开发者需要持续学习如何有效地与 AI 协作，掌握新的 AI 辅助工具和工作流程。

- 角色的演变： 开发者的角色可能更偏向于“架构师”、“产品经理”或“技术指导”，负责定义问题、引导 AI 并确保最终产品的质量。

## 潜在的挑战与局限性：

同时也要清楚地认识到，目前氛围编程同时会带来一些无法避免的挑战与局限。

- 代码质量和可靠性： AI 生成的代码可能存在隐藏的 Bug、安全漏洞或性能问题。开发者仍需要具备审查、测试和调试能力。

- 调试复杂性： 理解和调试由 AI 生成的、可能风格迥异或使用了不熟悉模式的代码，有时可能比调试自己写的代码更困难。

- 过度依赖与技能退化： 如果完全依赖 AI，开发者可能会逐渐丧失基础的编码能力和解决问题的直觉。

- AI 的“理解”局限： 对于非常复杂、创新或领域特定的问题，AI 可能难以完全理解需求或生成最优方案，容易产生“幻觉”。

- 成本与数据隐私： 高效的 AI 编程工具通常需要付费订阅，且将代码或需求发送给第三方 AI 服务可能涉及数据隐私和安全风险。

## 总结：拥抱人机协作的新篇章

总而言之，氛围编程旨在增强开发者的能力，改变工作的重心，将开发者从繁琐的实现细节中解放出来，更专注于创造性、架构设计和问题解决本身。积极理解、适应并掌握这种人机协作的新范式，有效利用 AI 工具，将是未来开发者保持核心竞争力的关键。它预示着一个更加注重创意、效率和人机协同的软件开发新时代的到来。