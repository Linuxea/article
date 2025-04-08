# Dify

Dify 是一个开源平台，旨在简化 AI 应用程序的开发，特别是那些利用大型语言模型（LLMs）的应用程序。它是一个无代码/低代码工具，使包括非工程师和初学者在内的用户能够通过直观的拖放界面创建生成式 AI 应用程序，例如聊天机器人、文本摘要工具、内容生成器和工作流自动化工具。

## Dify 的主要功能

### 无代码开发：

Dify 允许用户无需编程知识即可构建 AI 应用程序。其拖放界面使初学者和非技术用户也能轻松上手。

平台支持可视化编程，用户可以通过连接功能模块来创建工作流和流程。

### 支持多种 LLM：

Dify 集成了多种 LLM 提供商，包括 OpenAI、Anthropic（Claude）、Azure OpenAI、Llama2、Hugging Face 和 Replicate。这种灵活性使用户能够选择最适合其需求的模型。

### 检索增强生成（RAG）：

RAG 引擎使 AI 应用程序能够引用外部知识源，例如公司文档、网页或 Notion 数据库，从而生成更准确的响应。

### 可定制的工作流：

用户可以通过将多个操作链接在一起来自动化复杂任务。此功能适用于根据特定触发条件或情况执行顺序流程的工作流创建。

### 与外部 API 和工具集成：

Dify 支持与 Google 搜索、Slack、DALL-E、Stable Diffusion 和 Firecrawl 等工具的集成，以增强功能。

### 开源设计：

作为一个开源平台，Dify 允许用户在自己的基础设施（例如 AWS）上托管，从而确保数据隐私和安全，无需将数据发送到第三方 SaaS 提供商。

## Dify 的应用场景

![dify_app.png](dify_app.png 'dify_app.png')

- **聊天机器人**：构建能够引用内部文档或外部网站的智能聊天机器人，用于客户支持或内部查询。
- **AI 代理**：开发 AI 驱动的代理，自动化重复性任务或根据用户查询执行网页搜索。
- **文本生成**：创建用于摘要长文档、生成文章或将内容翻译成多种语言的工具。
- **工作流自动化**：（chatflow & workflow）设计能够高效处理多步骤流程的工作流，利用 AI 功能提升效率。


今天我们通过动手实践这几种 app 类型来熟悉在 Dify 创建 AI 应用。


## 创建 App

### 聊天机器人

支持多轮对话的 AI：

![chat_llm.png](chat_llm.png 'chat_llm.png')



### AI 代理 

配置工具集，指定工具名称与作用：

![dify_agent_2.png](dify_agent_2.png 'dify_agent_2.png')


Dify 的 Agent 模式允许用户创建能够自主执行任务的 AI 代理。通过结合 LLM 和外部工具，Agent 可以根据用户的指令完成复杂的任务，例如数据检索、信息处理和自动化操作。


### 文本生成

配置`目标语言`以及`源文本` 组件，将用户的输入翻译为目标语言。

![dify_textgenerator.png](dify_textgenerator.png 'dify_textgenerator.png')

### Chatflow

通过`问题分类器`节点，将用户的请求分为售前，售后，闲聊，将请求流转到不同的 LLM 进行后续处理。

![dify_chatflow.png](dify_chatflow.png 'dify_chatflow.png')

### workflow


根据用户的输入，将请求交由 R1，借助 R1 强大的思维链能力，将对用户问题的思维链过程完整地输出到下一个非推理LLM节点，让非推理LLM复现思维过程，给出答案。
最终通过 `Text-2-Speech`节点生成语音文件。

![dify_workflow.png](dify_workflow.png 'dify_workflow.png')

## 总结与展望
通过本文的介绍，我们了解了Dify这一强大的开源AI应用开发平台，它通过无代码/低代码的方式，使各种背景的用户都能轻松构建AI应用。我们详细探讨了Dify的主要功能和应用场景，包括聊天机器人、AI代理、文本生成、Chatflow和Workflow等多种应用类型。

实践价值
Dify的实际应用价值在于:

- 降低开发门槛：即使是非技术人员也能快速构建专业AI应用

- 灵活的模型选择：支持OpenAI、Anthropic、Llama2等多种LLM，满足不同需求

- 数据安全与隐私：开源设计允许用户在自己的基础设施上部署，确保数据安全

- 强大的集成能力：与Google搜索、DALL-E等多种外部工具无缝集成

Dify为AI应用开发带来了民主化，让更多人能够参与到AI创新中。无论是企业内部工具开发、客户服务优化，还是创意内容生成，Dify都提供了一个直观、强大且灵活的平台，帮助用户将AI的潜力转化为实际应用价值。