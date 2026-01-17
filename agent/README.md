# 从图片标签匹配到 Agent 设计：一次 LLM API 对接的思考之旅

## 起点：一个看似简单的需求

一个看起来很直接的需求：给用户上传的图片打上内容标签。有一组预定义的候选标签列表，需要从中选出最匹配的几个。这个场景在内容审核、图片分类等业务中很常见。

仅从技术角度分析，当前 AI 的能力下，直接的思路是把这个任务交给大语言模型。现在的多模态模型已经能够理解图片内容，让它从候选列表中挑选合适的标签，应该是一个很自然的应用场景。于是开始对接 LLM API，把图片和标签列表一起发送过去，让模型帮忙做选择。

最初的实现确实很简单。使用标准的 chat completion 接口，构造一个 system message 说明任务目标，然后在 user message 中附上图片和候选标签。模型的回复通常能给出合理的选择，这个方案看起来可以工作。但随着需求的演进，事情开始变得有趣起来。

## 需求演进中对 API 设计哲学的理解

当图片的数量从几张增加到几十张甚至上百张时，批量处理的需求自然浮现。这时会注意到，LLM 的 API 接口其实已经经历了好几代的演化。

最早期的接口是 completion，逻辑非常直接：给一段文本，续写下一段。这种接口适合做文本生成，但对于需要多轮交互或者带有明确角色定位的场景就显得力不从心了。

后来出现的 chat completion 接口引入了一个关键概念：角色（role）。可以通过 system 角色给模型设定行为准则，通过 user 角色输入用户的问题，通过 assistant 角色来表示模型的回复。`这个设计背后的哲学很清晰：对话不是简单的文本续写，而是有上下文、有角色定位的交互过程。`

当需要给模型提供系统级的指令时，比如"你是一个专业的图片标注助手，请严格从候选列表中选择标签"，chat 接口就能很好地承载这个语义。在图片标签匹配的场景中，可以在 system message 中说明整体任务规则，然后在 user message 中分批次提供图片和对应的候选标签。模型能够理解这是同一个任务的不同部分，而不是完全无关的多个请求。

## 引入评分机制与工具调用的必要性

随着需求进一步细化，提出了新的要求：不仅要选出匹配的标签，还要对每张图片的匹配程度打分，并且计算所有图片的平均匹配分数。这个需求看起来合理，但实际操作中很快遇到了问题。

大语言模型在理解语义、做出判断方面表现出色，但在数学计算上却容易出现幻觉。当让模型给 20 张图片分别打分，然后计算平均值时，它有时会算错。这不是偶尔的失误，而是一个系统性的问题：LLM 不擅长精确的数值计算。

> 你是否还记得 9.11 与 9.9 的大小比较问题？

这时需要引入工具调用，也就是 function calling。这是现代 LLM API 的一个重要特性，它让模型不再试图自己完成所有任务，而是成为一个协调者。模型可以声明"需要调用某个工具"，然后由代码来真正执行计算、数据库查询、文件操作等具体任务。

可以定义一个 calculate_average 函数，告诉模型："当需要计算平均值时，调用这个函数，把分数列表传给它。" 模型的工作变成了：理解图片内容，给出合理的分数，收集这些分数，然后声明需要调用计算函数。真正的数学运算由代码完成，结果再返回给模型，让它继续后续的处理或生成最终的报告。

下面是一个用 Go 实现的 AI Agent 核心结构，展示了如何支持工具调用和多轮对话：

```go
// AIAgent AI 代理，支持多轮对话和工具调用
type AIAgent struct {
    client          *arkruntime.Client
    modelName       string
    tools           map[string]Tool  // 工具注册表
    maxRounds       int              // 最大对话轮数，防止无限循环
    enableThinking  bool             // 是否启用思考模式
    enableReasoning bool             // 是否启用推理模式
    responsesText   *responses.ResponsesText
}

// RegisterTool 注册工具
func (a *AIAgent) RegisterTool(tool Tool) {
    a.tools[tool.Name()] = tool
    logger.Infof("Tool registered: %s", tool.Name())
}
```

这个设计包含了几个关键要素：工具注册表用于管理可用的工具，maxRounds 参数防止模型陷入无限循环，enableThinking 和 enableReasoning 开关控制模型的推理模式。

Agent 的核心运行逻辑是一个循环，不断检查模型是否需要调用工具：

```go
func (a *AIAgent) Run(ctx context.Context, input *responses.ResponsesInput) ([]*responses.OutputItem, error) {
    // 构建工具列表并添加到请求中
    var responsesTools []*responses.ResponsesTool
    for _, tool := range a.tools {
        respTool, err := BuildResponsesTool(tool)
        if err != nil {
            return nil, fmt.Errorf("failed to build tool %s: %w", tool.Name(), err)
        }
        responsesTools = append(responsesTools, respTool)
    }

    req := &responses.ResponsesRequest{
        Model: a.modelName,
        Input: input,
        Tools: responsesTools,  // 告知模型可用的工具
    }

    round := 0
    for {
        round++
        if round > a.maxRounds {
            return nil, fmt.Errorf("exceeded max rounds (%d)", a.maxRounds)
        }

        // 发送请求
        resp, err := a.client.CreateResponses(ctx, req)
        if err != nil {
            return nil, fmt.Errorf("CreateResponses error: %w", err)
        }

        // 检查是否有工具调用
        functionCalls := a.extractFunctionCalls(resp.Output)
        if len(functionCalls) == 0 {
            // 没有工具调用，返回最终结果
            return resp.Output, nil
        }

        // 执行所有工具调用
        var toolOutputItems []*responses.InputItem
        for _, functionCall := range functionCalls {
            toolOutput, err := a.executeTool(functionCall)
            if err != nil {
                return nil, fmt.Errorf("failed to execute tool: %w", err)
            }
            // 将工具执行结果添加到下一轮的输入中
            toolOutputItems = append(toolOutputItems, /* ... */)
        }

        // 准备下一轮请求，将工具的执行结果传回模型
        req = &responses.ResponsesRequest{
            Model: a.modelName,
            Input: &responses.ResponsesInput{
                ListValue: toolOutputItems,
            },
        }
    }
}
```

这个循环体现了 Agent 的核心工作模式：发送请求 → 检查是否需要工具 → 执行工具 → 将结果反馈给模型 → 继续下一轮。模型和工具形成了一个协作闭环。

这个过程让人深刻理解了 API 设计背后的一个核心哲学：**模型不应该试图做所有事情**。它应该专注于它擅长的部分——理解语义、规划流程、做出判断，而把精确计算、数据存取、外部系统交互等任务交给专门的工具。这种分工不是模型能力的缺陷，而是一种更合理的架构设计选择。计算交给计算器，数据库查询交给 SQL，文件操作交给文件系统 API，模型负责把这些能力编排起来。



## 切换模型的启示：底座的重要性

当基本的技术方案跑通之后，自然会想到做一件事：换几个不同的模型测试一下效果。这个过程带来了一个意外的收获，也是一个重要的认知跃迁。

不同模型在指令遵循能力上的差异是显而易见的。有些模型能够准确理解 system message 中的多步骤指令，严格按照"先分析图片内容，再从候选列表选择，然后打分，最后汇总"的流程执行。而有些模型则容易跑偏，比如它可能会自己创造一些候选列表里没有的标签，或者忽略了打分的要求，直接给出标签选择结果。

> 在实现工具调用的过程中，能够发现思考链和推理模式的价值。当开启模型的 thinking 或 reasoning 选项后，它的表现会变得更加稳定。模型会先明确说明自己的思考过程，然后再给出最终的结果。这种显式的推理步骤减少了模型的随机性错误，让整个流程更加可控。

更关键的是工具调用能力的差异。有些模型在构造函数调用参数时非常规范，严格按照定义的 JSON schema 来传递参数。但有些模型则经常出现格式错误，比如该传数组的地方传了字符串，或者参数名称拼写错误。这些看似小问题，在生产环境中却意味着更多的错误处理代码和更低的系统可靠性。

这个发现揭示了一个关键点：**选择什么样的 LLM 作为底座，直接决定了能构建什么样的 Agent 系统**。可以用一个比喻来理解这个关系：如果把 Agent 比作一栋建筑，那么 LLM 就是这栋建筑的地基。地基的承重能力、稳定性、抗震性能，决定了能在上面建多高的楼、能承受多大的压力。

同样的道理，如果底座模型连基本的指令遵循都做不好，那么设计再精巧的提示词、再完善的工具定义、再复杂的流程编排，都只是在脆弱的基础上建高楼。模型的理解能力是感知层，决策能力是规划层，工具调用是执行层。任何一层出现系统性的弱点，整个 Agent 的可靠性都会受到影响。

这个认知重新定义了对 Agent 本质的理解。**Agent 不是 LLM 的简单应用，而是基于 LLM 能力的工程设计产物**。需要理解 LLM 的能力边界在哪里，哪些任务它能做好，哪些任务需要借助工具。然后通过提示词工程来引导模型的行为，通过工具设计来扩展模型的能力，通过流程编排来保证任务的完成。但所有这些工程手段的前提是：有一个足够可靠的底座模型。

## Agent 的现在：能力突破与清醒认识

2023 年 ChatGPT 面世， 2024 年是大语言模型井喷的一年，那么过去的 2025 年可以说是 Agent 真正诞生的元年。

能力上确实有令人印象深刻的突破。比如发布时提到 Claude Sonnet 4.5 现在能够持续工作超过 30 个小时而不丢失任务目标[^1]，在编码任务的基准测试上也达到了接近 80% 的成绩[^2]。这不仅仅是时长或分数的提升，更关键的是模型在长时间运行中保持状态连贯性和目标导向的能力有了质的飞跃。

但同时也需要保持清醒的认识。即使是最好的模型，仍然存在一定的失败率，这意味着在生产环境中不能完全放手让 Agent 自主运行关键任务。长时间运行带来的成本问题、对工具和环境可靠性的依赖、非确定性的运行环境等挑战，都是需要面对的现实。

因此谈论 Agent 时，需要认识到它确实取得了长足的进步，但还远远谈不上成熟。它更像是一个展现出巨大潜力的新生事物，而不是一个可以全面推广的成熟技术。在这里可以更加期待 Agent 在 2026 年取得新的突破。

## 对未来的展望

站在 2026 年初这个时间点，可以看到几条隐约的演进路径。也许 Agent 会从单打独斗走向团队协作，不同的 Agent 负责不同环节。也许会出现更多针对特定领域深度优化的 Agent，而不是什么都做的通才。也许最现实的形态不是完全自主，而是人机协作模式——Agent 处理繁重工作，在关键点请求人工确认。也许会有类似容器技术那样的标准化出现，让 Agent 的构建和部署变得更加规范。

这些都只是可能性，但可以确定的是，现在正站在一个转折点上。LLM 不再只是回答问题的工具，而是开始成为能够行动的代理。这个转变才刚刚开始。

## 回到起点

回到最初的那个需求：给图片打标签。现在不仅知道如何实现这个功能，更重要的是，理解了从 API 调用到 Agent 设计的整条思考链路。知道了 API 规范为什么会从 completion 演化到 chat completion，为什么需要引入 function calling。知道了不同模型在能力上的差异，以及如何根据底座模型的特点来设计上层的应用。也看到了 Agent 的现状和未来的可能性。

作为工程师，角色也在悄然变化。过去调用 API，期待它给出一个答案。现在设计 Agent，定义它的感知范围、决策逻辑、行动能力，然后像管理一个团队成员那样去监督和优化它的表现。这不单是技术栈的更新，或者更重要的是工作方式的转变。而我们需要做的，是更快地跟上步伐。

---

## 参考来源

[^1]: [Adrian Faryniuk - Autonomous AI that lasts 30+ hours without losing focus](https://www.linkedin.com/posts/adrian-faryniuk_most-ai-models-break-after-4-hours-of-autonomous-activity-7410930075686146048-1xcW) | [Comet API - Claude Sonnet 4.5 技术分析](https://www.cometapi.com/thinking-mode-in-claude-4-5-all-you-need-to-know/)
[^2]: [SWE-bench Verified 基准测试](https://www.lilbigthings.com/post/anthropic-vs-openai) - Claude Sonnet 4.5 达到 77.2%-80.9% 的分数
