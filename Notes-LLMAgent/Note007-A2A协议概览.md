## A2A 简要介绍

A2A 协议是 Google 在今年 4 月 9 日对外发布的一个协议，全称是 Agent to Agent 协议，这个协议规定了 Agent 与 Agent 之间的沟通规范

<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/c8096462188141d5bfff4a58b3c6cad7.png#pic_center " style="zoom:60%;" />
</div>




## A2A 协议使用场景

假设你住在西雅图，你想在未来三天里面挑一天飞往纽约，不仅如此，你还有一个额外的要求，因为你喜欢好天气，所以呢你希望出发的那天西雅图阳光明媚

如果你生活在一个没有大模型，没有 Agent 的时代，你需要自己来做以下三件事情：

1. 查找西雅图未来三天的天气预报
2. 根据天气预报，从这三天里面选择西雅图天气最好的一天
3. 查询那一天从西雅图到纽约的机票信息


不过好在你生活在一个有着大模型的时代，你发现有个系统正好可以帮你做这几件事情：

<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/dfc2ea2b3ae048318a2604804d94fd25.png#pic_center " style="zoom:60%;" />
</div>

这个系统内部部署了三个 Agent，分别是调度 Agent、天气 Agent 和机票 Agent，这个系统可以一次性帮你完成之前的三个任务

----


那现在有个问题，A2A 协议作用在这个链路中的哪一部分呢？🤔

<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/93449cc000e443c48a735b53a7a0269e.png#pic_center " style="zoom:60%;" />
</div>

没错，就是 Agent 与 Agent 之间的交互部分，简单来说，只要两个 Agent 之间需要沟通，它们就需要使用 A2A 协议，这个就是 A2A 协议的使用场景了

----


天气查询 Agent 内部肯定有一个大模型，但是除了大模型之外，它可能还部署了多个 MCP 工具，有些负责天气预告，有些负责历史气象分析

<div align="center">
    <img src="https://i-blog.csdnimg.cn/direct/bc836166c20349ac8fc88b068c95dd2c.png#pic_center " style="zoom:60%;" />
</div>

所以总结一下，**A2A 协议是作用在 Agent 与 Agent 之间的，MCP 协议是作用在 Agent 内部的，它们的作用域不同**，这个呢就是这两个协议的最大区别了