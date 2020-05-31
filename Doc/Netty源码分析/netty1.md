这篇主要关注下事件是如何通过管道一步步传递给用户的处理器的。

根据上一篇的分析，调用流程：
> logHandle-->ServerBootstrapAcceptor-->用户处理器