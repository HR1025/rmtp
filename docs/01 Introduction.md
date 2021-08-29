# 介绍

Adobe的实时消息协议（RTMP）通过可靠的流传输（如TCP[RFC0793]）提供双向消息多路传输服务，旨在在一对通信对等方之间传输视频、音频和数据消息的并行流以及相关的定时信息。实现通常为不同类别的消息分配不同的优先级，当传输容量受到限制时，这会影响消息排队到底层流传输的顺序。

本备忘录描述了实时消息协议的语法和操作。

## 术语
本备忘录中的关键词“MUST”、“MUST NOT”、“REQUIRED”、“SHALL”、“SHALL NOT”、“SHOULD”、“SHOULD NOT”、“RECOMMENDED”、“NOT RECOMMENDED”、“MAY”和“OPTIONAL”应按照[RFC2119]中的说明进行解释。


