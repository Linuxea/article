## 1. 模型上下文协议 (MCP)：概述

模型上下文协议 (MCP) 是由 Anthropic 于 2024 年 11 月推出的开放标准。它旨在简化和标准化 AI 应用程序（特别是由大型语言模型 (LLM) 提供支持的应用程序）与外部工具、数据源和系统的连接。MCP 类似于硬件外设的 USB 标准，充当 AI 集成的通用接口。

### 1.1 目的与优势
MCP 解决了将 AI 应用程序与各种工具和系统集成的复杂性。传统上，开发人员面临 M×N 的问题，其中 M 代表 AI 应用程序的数量，N 代表外部系统的数量。每种组合都需要定制集成，导致效率低下。MCP 通过定义标准化的客户端-服务器架构，将其简化为 M+N 的问题。

### 1.2 核心架构
MCP 基于客户端-服务器模型运行，包含三个主要组件：

#### MCP 主机：面向用户的应用程序（例如 Claude Desktop、Cursor 等 IDE），与 MCP 服务器交互。

#### MCP 客户端：嵌入在主机应用程序中，管理与 MCP 服务器的连接。每个客户端与一个服务器保持 1:1 的关系。

#### MCP 服务器：通过标准化 API 提供特定功能（工具、资源、提示）的外部程序。


今天动手实现一个简易的 MCP Server，利用现有的 MCP Host 来连接上我们的 MCP Server.

## 2. Search MCP Server

### 2.1 GPT模型静态知识的局限性总结

众所周知，预训练模型特有的静态知识属性，带来了主要挑战有：

- **更新成本高**：需要通过计算密集且耗时的完整重训练来更新知识
- **"幻觉"现象**：当回答超出训练范围的问题时可能生成看似合理但实际不正确的内容
- **上下文理解受限**：缺乏外部信息整合能力，难以提供特定领域或最新的准确回答


大模型需要通过外部数据库解决幻觉与上下文理解受限的挑战。通过构造一个 Search MCP Server，将搜索功能独立部署，提供给支持 MCP 的 LLM 共同使用。

### 2.2 MCP Server 

#### 2.2.1. 底层搜索工具的实现
```python
# Please install OpenAI SDK first: `pip3 install openai`
from openai import OpenAI
import os

client = OpenAI(
    api_key=此处替换为api key, base_url="https://api.perplexity.ai/"
)

def search(query: str) -> str:
    return nlpProcess(query)

def nlpProcess(query: str, llm="sonar-pro") -> str:
    response = client.chat.completions.create(
        model=llm,
        messages=[
            {"role": "system", "content": "You are a helpful assistant"},
            {"role": "user", "content": query},
        ],
        stream=False,
    )
    resp = response.choices[0].message.content
    print("查询响应结果:" + resp)
    return resp
```

通过初始化 openAI client，使用 perplexity AI 搜索服务获取查询的结果。

#### 2.2.2. 封装为 MCP tool
```python
import helpers
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("search")

@mcp.tool()
def search(query: str) -> str:
    """根据 query 发起互联网搜索最新的结果

    Args:
        query: 一段自然语言描述的查询
    """
    return helpers.search(query)
```

使用 mcp.tool 标识为 MCP Tool，添加相应的注解让 LLM 理解函数的作用以及必要的参数。

#### 2.2.3. MCP Server
```python
from tools import mcp

if __name__ == "__main__":
    # Run the server
    mcp.run(transport="stdio")
```
启动 MCP server。

#### 2.2.4. Claude Desktop APP 测试

在 Claude Desktop APP 配置 MCP server：
!['claude_desktop.png'](claude_desktop.png 'claude_desktop.png')

检测到我们实现的工具：
![claude_desktop_tool.png](claude_desktop_tool.png 'claude_desktop_tool.png')

发起对话
![claude_desktop_search.png](claude_desktop_search.png 'claude_desktop_search.png')

## 总结
通过对模型上下文协议 (MCP) 的深入探讨，我们可以清楚地看到它在简化 AI 应用程序与外部工具和系统集成方面的巨大潜力。MCP 不仅解决了传统集成方式的效率问题，还为开发者提供了一个标准化的平台，推动了 AI 技术在各领域的广泛应用。


## Reference
[代码地址][https://github.com/Linuxea/mcp]

[文章地址][https://github.com/Linuxea/article/tree/main/MCP]
