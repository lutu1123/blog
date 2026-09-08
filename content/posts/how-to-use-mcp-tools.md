---
title: "如何使用 MCP(Model Context Protocol)工具:深度技术指南"
date: 2026-09-08
tags: [MCP, AI, LLM, Agent, 工具调用]
author: "路途"
---

给你的 LLM 应用接上「查数据库、订机票、读写文件」的能力,你要写什么?大概率是每个模型供应商一套 function calling 声明,换个宿主就重写一遍对接。这个痛点,MCP(Model Context Protocol)想一次性解决:像 USB-C 统一充电口一样统一 AI 应用与外部系统的连接——官方原话就叫 "a USB-C port for AI applications"。

MCP 由 Anthropic 于 2024 年 11 月 25 日发布并开源,基于 JSON-RPC 2.0;2025 年 12 月被捐赠给 Linux Foundation 旗下的 Agentic AI Foundation(AAIF)。到协议版本 2026-07-28 的今天,它已有 10 种官方语言 SDK、一万多个生产服务器。它不是又一个框架,而是一层「工具接入」的通用协议。本文先讲清楚它是什么,再把可直接抄的配置与代码给你。

**为什么重要**:标准一旦建立,「写一次、处处用」才成为可能——同一个 server 既能进 Claude Desktop,也能进 Cursor、ChatGPT,而不是为每个宿主各写一遍。

## 核心概念与架构

MCP 是三层参与者:Host(AI 应用,如 Claude Desktop、Claude Code、VS Code)为**每个** MCP server 各实例化一个 Client,Client 与 Server 建立一条专用连接。本地 stdio server 通常服务单个 client;远程 Streamable HTTP server 通常服务多个 client。

两种 transport,按部署位置选:

- **stdio**:宿主拉起本地子进程,经 stdin/stdout 传消息,零网络开销——适合文件系统、本地代码这类高可信工具;
- **Streamable HTTP**:HTTP POST 传消息、可选 SSE 流式,支持 bearer token/API key,官方推荐 OAuth——适合把业务 API 暴露给远程 agent。

2026-07-28 版把协议改为**无状态**:协议版本、客户端能力、身份随每个请求放进 `_meta` 字段,server 用 `server/discover` 宣告能力,取代旧版 initialize 握手。典型交互是三步:`server/discover → tools/list → tools/call`。

Server 暴露三类原语,差别在于「谁控制」:

| 原语 | 是什么 | 控制者 |
|---|---|---|
| Tools | LLM 主动调用的可执行函数(写库、调 API、改文件),JSON Schema 校验入参 | 模型 |
| Resources | 只读上下文数据,唯一 URI + MIME type(如 `weather://forecast/{city}/{date}`) | 应用 |
| Prompts | 可复用提示模板(系统提示、few-shot),用户显式调用 | 用户 |

**为什么重要**:三原语对应三种接入形态——动作给模型(tools)、数据给应用(resources)、模板给人(prompts)。动手写 server 前先想清楚:你暴露的到底是哪一种?(顺带一提,旧版的 sampling、roots 自 2026-07-28 起已弃用,新代码别学。)

## 客户端配置:两份可直接抄的示例

以 Claude Desktop 为例,配置在 `claude_desktop_config.json`,声明 `mcpServers`:

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:\\Users\\username\\Desktop"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "<YOUR_TOKEN>" }
    }
  }
}
```

注意两个细节:一是凭据放进 `env` 而不是命令行参数,避免出现在进程列表里;二是 Windows 上 npx 型 server 要包一层 `cmd /c`(`"command": "cmd", "args": ["/c", "npx", ...]`),这是官方 README 明确注明的坑。

换个宿主语法就不同。Hermes 用 YAML 的顶层 `mcp_servers` 键,我本机挂的远程 12306 查票 server 就是一条声明:

```yaml
mcp_servers:
  12306-mcp:
    transport: streamable_http
    url: https://mcp.api-inference.modelscope.net/42b50a8c94a849/mcp
```

挂上后,工具以 `mcp__<server>__<tool>` 的命名出现在我的工具目录:本会话里就有 `mcp__12306_mcp__get_tickets`、`get_train_route_stations` 等 10 个工具。宿主无需自带任何 12306 对接或爬虫代码,就能查余票、车站、车次经停——这就是「远程 MCP server = 把业务 API 标准化成 LLM 可调用工具」的活例子。

除了手改配置,现代宿主普遍给了 CLI:比如 Hermes 的 `hermes mcp` 子命令,除 `add`/`remove`/`list`/`test` 外,还有 `catalog`、`install` 这类目录市场操作——找 server 像装包一样,连 URL 都不用记。

**为什么重要**:配置声明因宿主而异,本质却一致——告诉宿主「怎么拉起或连到 server」。看懂上面两份,换到 Cursor、VS Code Copilot 只是换个文件位置的事。

## 写一个最小 server:15 行 Python

官方 Python SDK v2 只需 `pip install "mcp[cli]"`,一个完整可运行的最小 server:

```python
from mcp.server import MCPServer

mcp = MCPServer("Demo")

@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

@mcp.resource("greeting://{name}")
def greeting(name: str) -> str:
    """Greet someone by name."""
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

函数签名加 docstring,JSON Schema 由 SDK 自动生成,不用手写 inputSchema。调试用 `uv run mcp dev server.py`(会拉起官方 MCP Inspector);要跑成远程服务,一条命令切到 HTTP:`uv run mcp run server.py --transport streamable-http`。TypeScript 端是等价写法:`@modelcontextprotocol/server` 包 + `registerTool` + Zod 定义 schema。

**为什么重要**:你的工具因此天然「可发现、可校验」——模型先 `tools/list` 看到 schema,再按 schema 调用。你只写业务逻辑,协议细节全部交给 SDK。

## 生态与选型

官方参考服务器仓库目前维护 7 个:Filesystem(安全文件操作)、Git、Memory(知识图谱记忆)、Fetch(网页抓取)、Time、Sequential Thinking、Everything(测试用);更多社区服务器可到 registry.modelcontextprotocol.io 目录浏览。参考实现官方明确标注「用于教学,非生产就绪,安全需自评」。

采用史是判断生态成熟度的最好信号:OpenAI 2025 年 3 月官方采纳,Google DeepMind 同年 4 月跟进,2025 年 12 月 MCP 归入 Linux Foundation;到 2026 年中,生产服务器超一万个、SDK 月下载量九千七百万以上。顺带区分:MCP 与 Agent Skills(Anthropic 2025 年 12 月开源的另一个开放标准)互补而非竞争——一个管外部工具连接,一个管模型内部技能。

那 function calling 怎么办?它俩不是替代关系:function calling 是模型供应商 API 内原生的 tool 参数,单体应用直连少量函数时最轻;MCP 的增量价值是标准化发现(tools/list)、进程与网络边界、跨宿主复用,以及和整个工具生态互通。它在设计上复用了 Language Server Protocol(LSP)的消息流思想——先把连接方式定为开放标准,各家宿主再在实现质量上竞争。选型建议一句话:函数少、单宿主、要极简——直接用 function calling;工具多、要隔离、想一处实现处处可用——上 MCP。

**为什么重要**:生态已过「要不要跟」的观望点,进入「怎么接」的执行期;不接 MCP 不丢分,但接了能白嫖整个 registry 的工具库。

## 安全与生产化

与同进程的函数调用不同,MCP 引入了真实边界——stdio 子进程、远程 HTTP。隔离带来安全,也带来攻击面。官方威胁模型点名的包括 Confused Deputy(工具权限混淆)、Token Passthrough、SSRF(比如禁止访问云 metadata 地址 169.254.169.254)、投毒工具外渗数据。落地守住三条:

1. **收敛本地权限**:stdio server 以你的用户权限运行,官方原文警告只给「你放心的目录」——filesystem server 别挂整个磁盘,用白名单路径;
2. **凭据与调用解耦**:密钥只进 `env` 或 secret 存储,不进命令参数;远程 server 只连可信来源,并审视它请求的授权范围;
3. **宿主侧兜底**:工具级批准对话框、预授权、活动日志。参考 Hermes 的做法:stdio 子进程默认只继承白名单环境变量(PATH、HOME 等),API key 必须显式写进 `env` 才传递,错误信息还自动脱敏——双向防泄漏。

**为什么重要**:给模型工具等于把执行权交给不可预测的输入。权限面收敛到最小、凭据与代码解耦,是 MCP 生产化的前提,不是可选项。

## 小结

MCP 的价值一句话:把「给 LLM 接工具」从每个应用的手工活,变成协议级的即插即用。上手路径也很短:先去 registry 挑一个现成 server 挂进你的宿主找手感;再照着上面的 15 行模板,写一个你自己的工具 server;最后用安全三原则过一遍权限面。十分钟,你就拥有了一个跨宿主可用的 AI 工具。
