# RTMP块流

本节指定实时消息传递协议区块流（RTMP区块流）。它为更高级的多媒体流协议提供多路复用和分组服务。

虽然RTMP区块流设计用于与实时消息协议（第6节）配合使用，但它可以处理发送消息流的任何协议。每条消息都包含时间戳和有效负载类型标识。RTMP Chunk Stream和RTMP一起适用于各种音频视频应用，从一对一和一对多直播到视频点播服务再到交互式会议应用。

当与可靠的传输协议（如TCP[RFC0793]）一起使用时，
RTMP区块流可以横跨多个流，并且保证所有消息按照时间戳的顺序有序地在端到端之间传递。RTMP区块流不提供任何优先级或类似形式的控制，但可由更高级别的协议用于提供此类优先级。例如，实时视频服务器可能会选择删除慢速客户端的视频消息，以确保根据发送时间或确认每条消息的时间及时接收音频消息。

RTMP区块流包含其自己的带内协议控制消息，并且还为更高级别的协议嵌入用户控制消息提供了一种机制。

## 消息格式
消息格式是否可以被拆分为块以便支持多路复用取决于更高级的协议。但是，消息格式应包含创建块所需的以下字段。

- Timestamp : 消息的时间戳。此字段可以传输4个字节。
- Length : 消息有效负载的长度。如果无法省略消息头，则应将其包含在长度中。此字段在区块头中占用3个字节。
- Type Id : 为协议控制消息保留了一系列类型ID。这些传播信息的消息由RTMP区块流协议和更高级别的协议处理。所有其他类型ID可供更高级别协议使用，并被RTMP区块流视为不透明值。事实上，RTMP区块流中的任何内容都不要求将这些值用作类型；所有（非协议）消息可以是相同的类型，或者应用程序可以使用此字段来区分同时跟踪而不是类型。此字段在区块头中占1字节。
- Message Stream ID : 消息流ID可以是任意值。复用到同一块流上的不同消息流根据其消息流ID进行解复用。除此之外，就RTMP块流而言，这是一个不透明的值。此字段以小尾端格式在块头中占据4个字节。

## 握手(Handshake)
RTMP连接从握手开始。握手与协议的其他部分不同；它由三个静态大小的块组成，而不是由带有标题的可变大小的块组成。

客户端（启动连接的端点）和服务器分别发送相同的三个数据块。为了方便说明，当客户端发送时，这些块将被指定为C0、C1和C2；而服务器发送时这些块被指定为S0、S1和S2。

### 握手顺序
握手从客户端发送C0和C1块开始。

客户端必须等到收到S1后再发送C2。在发送任何其他数据之前，客户端必须等到收到S2。

服务器在发送S0和S1之前必须等到收到C0，也可以等到C1之后。服务器必须等到收到C1后再发送S2。在发送任何其他数据之前，服务器必须等待收到C2。

### C0 和 S0 的格式
C0和S0数据包是一个八进制数字，被视作为一个8bit的整数字段：
```
             0 1 2 3 4 5 6 7
             +-+-+-+-+-+-+-+-+
             |    version    |
             +-+-+-+-+-+-+-+-+
             C0 and S0 bits

```
以下是C0/S0数据包中的字段：
- Version (8 bits) : 在 C0 包内，这个字段代表客户端请求的 RTMP 版本号。在 S0 包内，这个字段代表服务端选择的 RTMP 版本号。当前使用的版本是 3。版本 0-2 用在早期的产品中，如今已经弃用；版本 4-31 被预留用于后续产品；版本 32-255 （为了区分 RTMP 协议和文本协议，文本协议通常是可以打印字符）不允许使用。如果服务器无法识别客户端的版本号，应该回复版本 3,。客户端可以选择降低到版本 3，或者终止握手过程。

### C1 和 S1 的格式：
C1 和 S1 包长度为 1536 字节，包含以下字段：
```
                 0                 1                     2                   3
                 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         time (4 bytes)                        |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         zero (4 bytes)                        |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         random bytes                          |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         random bytes                          |    
                 |                           (cont)                              |
                 |                           ....                                |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                         C1 and S1 bits

```
- Time（4 bytes）：本字段包含一个时间戳，客户端应该使用此字段来标识所有流块的时刻。时间戳取值可以为零或其他任意值。为了同步多个块流，客户端可能希望发送另一个快流时间戳的当前值。
- zero（4 bytes）：本字段必须为零。
- random （1528 bytes）：此字段可以包含任意值。由于每个端点都必须区分它发起的握手响应和它的对等方发起的握手响应，因此该数据应该发送足够随机的信息。但不需要加密安全的随机性，甚至不需要动态值。

### C2 和 S2 的格式
C2和S2数据包的长度为1536个字节，几乎是S1和C1的拷贝，由以下字段组成：
```
                 0                   1                   2                   3
                 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         time (4 bytes)                        |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         time2 (4 bytes)                       |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         random echo                           |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                         random echo                           |
                 |                           (cont)                              |
                 |                           ....                                |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                          C2 and S2 bits
```
- Time (4 bytes) : 此字段必须包含S1（对于C2）或C1（对于S2）中对等方发送的时间戳。
- Time2 (4 bytes) : 此字段必须包含读取对等方发送的前一个数据包（s1或c1）的时间戳。
- Random echo (1528 bytes) : 此字段必须包含S1（对于C2）或S2（对于C1）中对等方发送的随机数据字段。任何一个对等方都可以将time和time2字段与当前时间戳一起用作连接带宽和/或延迟的快速估计，但这不太可能有用。

### 握手图
``` 
                 +-------------+                            +-------------+
                 |    Client   |       TCP/IP Network       |    Server   |
                 +-------------+             |              +-------------+
                        |                    |                     |
                   Uninitialized             |                Uninitialized
                        |       C0           |                     |
                        |------------------->|         C0          |
                        |                    |-------------------->|
                        |       C1           |                     |
                        |------------------->|         S0          |
                        |                    |<--------------------|
                        |                    |         S1          |
                    Version sent             |<--------------------|
                        |       S0           |                     |
                        |<-------------------|                     |
                        |       S1           |                     |
                        |<-------------------|                Version sent
                        |                    |         C1          |
                        |                    |-------------------->|
                        |       C2           |                     |
                        |------------------->|         S2          |
                        |                    |<--------------------|
                    Ack sent                 |                  Ack Sent
                        |       S2           |                     |
                        |<-------------------|                     |
                        |                    |         C2          |
                        |                    |-------------------->|
                 Handshake Done              |                Handshake Done
                        |                    |                     |
                            Pictorial Representation of Handshake
```
以下描述了握手图中提到的状态：
- Uninitialized : 协议版本在此阶段发送。客户端和服务器都未初始化。客户端发送数据包C0中的协议版本。如果服务器支持该版本，则会发送S0和S1作为响应。如果没有，服务器将通过采取适当的操作进行响应。在RTMP中，此操作是终止连接。
- Version Sent : 在未初始化状态之后，客户端和服务器都处于版本发送状态。客户端正在等待分组S1，服务器正在等待分组C1。在接收到等待的分组时，客户端发送分组C2，服务器发送分组S2。然后，状态变为Ack Sent。
- Ack Sent : 客户端和服务器分别等待S2和C2。
- Handshake Done : 客户端和服务器交换消息。

## 分块
在经历握手后，这个连接会多路复用一个或者多个块流。每个区块流携带来自一个消息流的一种类型的消息。创建的每个区块都有一个与之关联的唯一ID，称为区块流ID(chunk stream ID)。块通过网络传输。传输时，必须在下一个数据块之前完整发送每个数据块。在接收端，根据区块流ID将区块组装成消息。

分块允许将高级协议中的大型消息分解为较小的消息，例如，防止大型低优先级消息（如视频）阻止较小的高优先级消息（如音频或控制）。

分块还允许以较少的开销发送小消息，因为分块头包含信息的压缩形式，否则必须包含在消息本身中。

区块大小是可配置的。它可以使用 Set Chunk Size control message 设置。较大的块大小可以减少CPU使用，但也会导致较大的写操作，从而延迟低带宽连接上的其他内容。较小的数据块不适用于高比特率流。块大小在每个方向上都是独立保持的。

### 块格式
每个块由一个头部以及数据组成。头部本身有三个部分：
``` 
             +--------------+----------------+--------------------+--------------+
             | Basic Header | Message Header | Extended Timestamp | Chunk Data   |
             +--------------+----------------+--------------------+--------------+
             |                                                                   |
             |<------------------- Chunk Header ----------------->|
                                     Chunk Format

```
- Basic Header (1 to 3 bytes) : 此字段对区块流ID(chunk stream ID)和区块类型(chunk type)进行编码。区块类型确定编码消息头的格式(Message Header)。长度完全取决于区块流ID，它是一个可变长度字段。
- Message Header (0, 3, 7, or 11 bytes) : 此字段编码有关正在发送的消息的信息（无论是全部还是部分）。可以使用区块头(chunk header)中指定的区块类型(chunk type)确定长度。
- Extended Timestamp (0 or 4 bytes) : 根据区块消息头中的编码时间戳或时间戳增量字段，此字段在某些情况下存在。
- Chunk Data (variable size) : 此区块的有效负载，最大为配置的最大区块大小。

#### 块基本头部(Chunk Basic Header)
Chunk Basic Header 对区块流ID和区块类型进行编码（由下图中的fmt字段表示）。区块类型确定编码消息头的格式。区块基本标头字段可以是1、2或3字节，具体取决于区块流ID。

一个好的实现将会以最小的表达形式来存储 ID。

该协议最多支持65597个ID为3-65599的流。ID 0、1和2被保留。值0表示2字节形式和64-319范围内的ID（第二个字节+64）。值1表示3字节形式和64-65599范围内的ID（（第三个字节）*256+第二个字节+64）。范围为3-63的值表示完整的流ID。值为2的区块流ID保留用于底层协议控制消息和命令。区块基本标头中的位0-5（least significant）表示区块流ID。块流 ID 2-63 在这个字段中可以以1字节的形式编码。 
```
                  0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+
                 |fmt|    cs id  |
                 +-+-+-+-+-+-+-+-+
                 Chunk basic header 1 
```

区块流ID 64-319可以以头的2字节形式编码。ID计算为（第二个字节+64）。
```
                  0               1
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |fmt|0          | cs id - 64    |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                      Chunk basic header 2
```
块流ID 64-65599可以在此字段的3字节版本中进行编码。ID计算为（（第三个字节）*256+（第二个字节）+64）。
```
                  0               1               2
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |fmt|1          |        cs id - 64             |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                            Chunk basic header 3
```
- cs id (6 bits) : 此字段包含区块流ID，值范围为2-63。值0和1用于指示此字段的2字节或3字节版本。
- fmt (2 bits) : 此字段标识“区块消息头(chunk message header)”使用的四种格式之一。
- cs id - 64 (8 or 16 bits) : 此字段包含区块流ID减去64。例如，ID 365将由cs id中的1和此字段16位形式的301表示。

#### 块消息头部(Chunk Message Header)
块消息头(Chunk Message Header)有四种不同的格式，由块基本头部(Chunk Basic Header)中的“fmt”字段选择。

实现应该为每个块消息头部使用尽可能紧凑的表示。

##### Type 0
类型0块头的长度为11字节。此类型必须在块流的开始以及流时间戳向后时使用（例如，由于向后搜索）。
```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |             timestamp                         |message length |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |    message length (cont)      |message type id| msg stream id |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |          message stream id (cont)             |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                    Chunk Message Header - Type 0
```
- timestamp (3 bytes) : 对于type-0块，消息的绝对时间戳发送到这里。如果时间戳大于或等于16777215（十六进制0xFFFFFF），则此字段必须为16777215，表示存在扩展的时间戳字段以编码完整的32位时间戳。否则，此字段应为整个时间戳。

##### Type 1
类型1块头的长度为7字节。不包括消息流ID(message stream ID)；此块采用与前一块相同的流ID(stream ID)。具有可变大小消息（例如，许多视频格式）的流应在第一个消息之后的每个新消息的第一个块中使用此格式。
``` 
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |         timestamp delta                       |message length |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |     message length (cont)     |message type id|
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                             Chunk Message Header - Type 1
```


##### Type 2
类型2块头的长度为3字节。不包括流ID(stream ID)和消息长度(message length)；此区块与前一区块具有相同的流ID和消息长度。具有恒定大小消息的流（例如，某些音频和数据格式）应在第一个消息之后的每个消息的第一个块中使用此格式。
```
                  0               1               2
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                timestamp delta                |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                           Chunk Message Header - Type 2 
```

##### Type 3
类型3块没有消息头。流ID、消息长度和时间戳增量字段不存在；此类型的块从前面的块中获取相同块流ID的值。当单个消息拆分为块时，除第一个块外的所有消息块都应使用此类型，参考示例2(Example 2)。由大小、流ID和时间间隔完全相同的消息组成的流应在类型2的块之后的所有块中使用此类型，参考示例1（Example 1）。如果第一条消息和第二条消息之间的增量与第一条消息的时间戳相同，则类型3的块可以立即跟随类型0的块，因为不需要类型2的块来注册增量。如果类型3块跟随类型0块，则此类型3块的时间戳增量与类型0块的时间戳相同。

##### 通用头部字段
块消息头(chunk message header)每个字段的描述:
- timestamp delta (3 bytes) ： 对于 Type 1 或 Typde 2 的块，将前一块的时间戳与当前块的时间戳之间的差异发送到此处。如果增量大于或等于16777215（十六进制0xFFFFFF），则此字段必须为16777215，表示存在扩展时间戳字段以编码完整的32位增量。否则，此字段应为实际增量。
- message length (3 bytes) : 对于 Type 0 或 Type 1 块，消息的长度发送到此处。请注意，这通常与块有效负载(chunk payload)的长度不同。块有效负载长度(chunk payload length)是除最后一个块外的所有块的最大区块大小，以及最后一个块的剩余值（对于小消息，可能是整个长度）。
- message type id (1 byte) : 对于 类型 0 或 类型 1 的块，消息类型在这里设置。
- message stream id (4 bytes) : 对于 Type 0 的 块，将存储消息流ID(message stream ID)。消息流ID以小端格式存储。通常，同一块流中的所有消息将来自同一消息流。虽然可以将单独的消息流多路复用到同一块流中，但这会破坏头部压缩的好处。但是，如果一个消息流被关闭，而另一个消息流随后被打开，则可以通过重新发送新的 Type 0 的块来重新复用现有的块流。

#### 拓展时间戳(Extended Timestamp)
扩展时间戳字段用于对大于16777215（0xFFFFFF）的时间戳或时间戳增量进行编码；也就是说，对于不适合0、1或2类块的24位字段的时间戳或时间戳增量。 也就是说，24位的bit可能不适合于编码时间戳(timestamp)或时间戳增量(timestamp deltas)(此字段存在于 Type 0、Type 1、Type 2 的块中)。此字段对完整的32位时间戳或时间戳增量进行编码。通过将 Type 0 的时间戳字段或 Type 1或 Type 2 的时间戳增量字段设置为16777215（0xFFFFFF）来指示此字段的存在。当同一块流ID(chunk stream ID)最近更新的 Type 0、Type 1、Type 2 块指示存在扩展时间戳字段时，此字段存在于类型3块中。

### 示例(Examples)
#### 示例1(Example 1)
此示例显示了一个简单的音频消息流。此示例演示了信息的冗余。
```
                 +---------+-----------------+-----------------+-----------------+
                 |         |Message Stream ID| Message TYpe ID | Time   | Length |
                 +---------+-----------------+-----------------+-------+---------+
                 | Msg # 1 |   12345         |        8        | 1000   |   32   |
                 +---------+-----------------+-----------------+-------+---------+
                 | Msg # 2 |   12345         |        8        | 1020   |   32   |
                 +---------+-----------------+-----------------+-------+---------+
                 | Msg # 3 |   12345         |        8        | 1040   |   32   |
                 +---------+-----------------+-----------------+-------+---------+
                 | Msg # 4 |   12345         |        8        | 1060   |   32   |
                 +---------+-----------------+-----------------+-------+---------+
                         Sample audio messages to be made into chunks 
```
下一个表显示了此流中生成的块。从消息3开始，数据传输得到优化。在此点之后，每条消息只有1字节的开销。
```
                 +--------+---------+-----+------------+------- ---+------------+
                 |        | Chunk   |Chunk|Header Data |No.of Bytes|Total No.of |
                 |        |Stream ID|Type |            |After      |Bytes in the|
                 |        |         |     |            |Header     |Chunk       |
                 +--------+---------+-----+------------+-----------+------------+
                 |Chunk#1 |   3     |  0  | delta: 1000|    32     |     44     |
                 |        |         |     | length: 32,|           |            |
                 |        |         |     | type: 8,   |           |            |
                 |        |         |     | stream ID: |           |            |
                 |        |         |     | 12345 (11  |           |            |
                 |        |         |     | bytes)     |           |            |
                 +--------+---------+-----+------------+-----------+------------+
                 |Chunk#2 |   3     |  2  | 20 (3      |    32     |     36     |
                 |        |         |     | bytes)     |           |            |
                 +--------+---------+-----+----+-------+-----------+------------+
                 |Chunk#3 |   3     |  3  | none (0    |    32     |     33     |
                 |        |         |     | bytes)     |           |            |
                 +--------+---------+-----+------------+-----------+------------+
                 |Chunk#4 |   3     |  3  | none (0    |    32     |     33     |
                 |        |         |     | bytes)     |           |            |
                 +--------+---------+-----+------------+-----------+------------+
                         Format of each of the chunks of audio messages
 
```

#### 示例2(Example 2)
此示例演示了一条消息，该消息太长，无法放入128字节的块中，并且被分成了几个块。
```
                 +-----------+-------------------+-----------------+-----------------+
                 |           | Message Stream ID | Message TYpe ID | Time   | Length |
                 +-----------+-------------------+-----------------+-----------------+
                 | Msg # 1   |       12346       |     9 (video)   |   1000 |   307  |
                 +-----------+-------------------+-----------------+-----------------+
                                Sample Message to be broken to chunks 
```
以下是生成的块：
```
                 +-------+------+-----+-------------+-----------+------------+
                 |       |Chunk |Chunk|Header       |No. of     |Total No. of|
                 |       |Stream| Type|Data         |Bytes after| bytes in   |
                 |       | ID   |     |             | Header    | the chunk  |
                 +-------+------+-----+-------------+-----------+------------+
                 |Chunk#1|   4  |  0  | delta: 1000 |    128    |    140     |
                 |       |      |     | length: 307 |           |            |
                 |       |      |     | type: 9,    |           |            |
                 |       |      |     | stream ID:  |           |            |
                 |       |      |     | 12346 (11   |           |            |
                 |       |      |     | bytes)      |           |            |
                 +-------+------+-----+-------------+-----------+------------+
                 |Chunk#2|   4  |  3  | none (0     |    128    |    129     |
                 |       |      |     | bytes)      |           |            |
                 +-------+------+-----+-------------+-----------+------------+
                 |Chunk#3|   4  |  3  | none (0     |     51    |     52     |
                 |       |      |     | bytes)      |           |            |
                 +-------+------+-----+-------------+-----------+------------+
                                 Format of each of the chunks
```
块1的头部数据(header data)指定整个消息为307字节。

请注意，从这两个示例中，Type 3 的块可以以两种不同的方式使用。第一个是用于指定消息的延续。第二种方法是明确一个新消息的开头，该消息的头部可以从现有状态数据派生。

## 协议控制消息(Protocol Control Messages)
RTMP块流使用消息类型ID(message type IDs)1、2、3、5和6进行协议信息控制。这些消息包含RTMP区块流协议所需的信息。

这些协议控制消息必须具有消息流(message stream)ID 0（称为控制流,control stream），并以区块(chunk stream)流ID 2的形式发送。协议控制消息在收到后立即生效；它们的时间戳被忽略。

### 设置块的大小(Set Chunk Size 1)
协议控制消息1（设置块大小）用于通知对等方新的最大块大小。

最大区块大小默认为128字节，但客户端或服务器可以更改此值，并使用此消息更新其对等方。例如，假设一个客户端想要发送131字节的音频数据，而数据块大小是128。在这种情况下，客户端可以将此消息发送到服务器，通知它数据块大小现在是131字节。然后，客户端可以在单个块中发送音频数据。

最大块大小应至少为128字节，并且必须至少为1字节。每个方向的最大块大小都是独立保持的。
```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |0|                        chunk size (31 bits)                 |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                     Payload for the ‘Set Chunk Size’ protocol message 
```
- 0 : 该位必须为零。
- chunk size (31 bits) : 此字段保存新的最大块大小（以字节为单位），它将用于发送方的所有后续块，直到另行通知。有效尺寸为1至2147483647（0x7FFFFFFF）。但是，所有大于 16777215 (0xFFFFFF)的设置都是等效，因为块的大小一定小于消息的大小，而没有消息的大小能够大于 16777215 个字节。

### 中止消息(Abort Message 2)
协议控制消息2，中止消息，用于通知对等方如果对等方正在等待块完成消息，然后通过块流丢弃部分接收的消息。对等方接收区块流ID作为此协议消息的有效负载。应用程序可在关闭时发送此消息，以指示不需要进一步处理消息。

```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                        chunk stream id (32 bits)              |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                         Payload for the ‘Abort Message’ protocol message
```
- chunk stream ID (32 bits) : 此字段保存块流ID，即当前消息将被丢弃。

### 确认 (Acknowledgement 3)
客户端或服务器必须在收到等于窗口大小的字节后向对等方发送确认。窗口大小是发送方在未收到接收方确认的情况下发送的最大字节数。此消息指定序列号，即到目前为止接收到的字节数。
```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                        sequence number (4 bytes)              |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                        Payload for the ‘Acknowledgement’ protocol message
```
- sequence number (32 bits) : 此字段保存到目前为止接收的字节数。

### 窗口确认大小 (Window Acknowledgement Size 5)
客户端或服务器发送此消息以通知对等方在发送确认之间要使用的窗口大小。在发送方发送窗口大小字节后，发送方希望得到对等方的确认。自上次确认发送后，接收端必须在收到指定字节数后发送确认，如果尚未发送确认，则从会话开始发送确认。
```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |             Acknowledgement Window size (4 bytes)             |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                  Payload for the ‘Window Acknowledgement Size’ protocol message
```

### 设置对等方的带宽 (Set Peer Bandwidth 6)
客户端或服务器发送此消息以限制其对等服务器的输出带宽。接收此消息的对等方通过将已发送但未确认的数据量限制在此消息中指示的窗口大小来限制其输出带宽。如果窗口大小与上次发送给此消息发送方的窗口大小不同，则接收此消息的对等方应使用窗口确认大小消息(Window Acknowledgement Size)进行响应。
```
                  0               1               2               3
                  0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |                   Acknowledgement Window size                 |
                 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                 |   Limit Type  |
                 +-+-+-+-+-+-+-+-+
                        Payload for the ‘Set Peer Bandwidth’ protocol message
```
Limit Type是以下值之一：
- 0 - Hard : 对等方应将其输出带宽限制为指定的窗口大小。
- 1 - Soft : 对等方应将其输出带宽限制在此消息中指示的窗口或已生效的限制，以较小者为准。
- 2 - Dynamic : 如果以前的限制类型为硬(Hard)限制，请将此消息视为标记为硬限制，否则忽略此消息。































