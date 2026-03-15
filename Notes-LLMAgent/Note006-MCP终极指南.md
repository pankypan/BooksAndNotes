# MCP终极指南(1)-基础
## MCP简要介绍

MCP 全称是 Model Context Protocol，是 Anthropic 公司在 2024 年 11 月 25 号发布的一个协议

MCP 能做什么
<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/97b6848b05754ae5823c8517ccd99509.png#pic_center " style="zoom:80%;" />
</div>

简单来说，MCP 就是能够让大模型更好地使用各类工具的一个协议。
- 比如借助 MCP 我们可以让模型使用浏览器上网查询信息;
- 可以让模型操作 Unity 编写游戏;
- 也可以让模型查询实时路况;


要想使用 MCP，你还得用到一个东西叫做 **MCP Host**，它本质上就是一个支持 MCP 协议的软件，常见的 MCP Host 包括 Claude Desktop、Cursor、Cline、Cherry Studio 等等



## 安装 MCP Host (Cline) 并使用

Cline 其实是 vscode 的一个插件
1. 我们可以在扩展应用商店搜索安装
2. 紧接着我们需要配置模型，Cline 支持不同的 API 接入方和不同的模型
3. 我们随便问它一个问题，给它打个招呼，看它能不能够正常回复
4. 一切都准备就绪，那我们就试着问它一个真正的问题：明天纽约的天气怎么样？



## 概念解释：MCP Server 和 Tool

实际上 MCP Server 跟我们传统意义上的 server 并没有什么太大的关系，它就是一个程序而已，只不过这个程序的执行是符合 MCP 协议的

大部分的 MCP Server 都是本地通过 Node 或者 Python 启动的，只不过在使用的过程中可能会联网，当然它也可能不联网，纯本地使用也是可以的。不管是联不联网，它都可以叫做 MCP Server


Cline 想要给我们安装的这个 OpenWeather 的 MCP Server 也内置了一些模块，这些模块在 MCP 领域的专业名词叫做 Tool


比如一个处理天气的 MCP Server，它内部可能会包含两个函数，分别是 get_forecast 和 get_alerts，如下图所示
<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/b96049d2f6274a9d961db3f24b54da1a.png#pic_center " style="zoom:80%;" />
</div>




## 配置 MCP Server




## 使用 MCP Server




## MCP 交互流程详解




## 如何使用他人制作的 MCP Server: uvx 部分




## 如何使用他人制作的 MCP Server: npx 部分








