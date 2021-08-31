# RTMP 控制信息
本节描述了服务器和客户端之间为相互通信而交换的不同类型的消息和命令。

服务器和客户端之间交换的不同类型的消息包括用于发送音频数据的音频消息、用于发送视频数据的视频消息、用于发送任何用户数据的数据消息、共享对象消息和命令消息。共享对象消息提供了一种通用方法来管理多个客户端和服务器之间的分布式数据。命令消息在客户端和服务器之间传输AMF编码的命令。客户机或服务器可以通过使用命令消息与对等方通信的流请求远程过程调用（Remote Procedure Calls,RPC）。

## 消息的类型
服务器和客户端通过网络发送消息以相互通信。消息可以是任何类型，包括音频消息、视频消息、命令消息、共享对象消息、数据消息和用户控制消息。

### 命令消息(Command Message 20 17)
命令消息在客户端和服务器之间传输AMF编码的命令。对于AMF0编码，这些消息的消息类型值为20；对于AMF3编码，这些消息的消息类型值为17。发送这些消息是为了在对等机上执行一些操作，如连接(connect)、创建流(createStream)、发布(publish)、播放(play)和暂停(pause)。命令消息（如onstatus、result等）用于通知发送方所请求命令的状态。命令消息由命令名、事务ID和包含相关参数的命令对象组成。客户机或服务器可以通过使用命令消息与对等方通信的流请求远程过程调用（RPC）。

### 数据消息(Data Message 18 15)
客户端或服务器发送此消息以向对等方发送元数据或任何用户数据。元数据包括有关数据（音频、视频等）的详细信息，如创建时间(creation time)、持续时间(duration)、主题(theme)等。已为这些消息分配了消息类型值18（适用于AMF0）和消息类型值15（适用于AMF3）。

### 共享对象消息(Shared Object Message 19 16)
共享对象是跨多个客户端、实例等同步的Flash 对象（键值对的集合）。AMF0的消息类型19和AMF3的消息类型16是为共享对象事件保留的。每条消息可以包含多个事件。

```
                 +------+------+-------+-----+-----+------+-----+ +-----+------+-----+
                 |Header|Shared|Current|Flags|Event|Event |Event|.|Event|Event |Event|
                 |      |Object|Version|     |Type |data  |data |.|Type |data  |data |
                 |      |Name  |       |     |     |length|     |.|     |length|     |
                 +------+------+-------+-----+-----+------+-----+ +-----+------+-----+
                 |                                                                   |
                        |<- - - - - - - - - - - - - - - - - - - - - - - - - - - - - >|
                        |                 AMF Shared Object Message body             |
                                 The shared object message format
```
支持以下事件类型：
```
                 +---------------+--------------------------------------------------+
                 |         Event |                   Description                    |
                 +---------------+--------------------------------------------------+
                 | Use(=1)       | The client sends this event to inform the server |
                 |               | about the creation of a named shared object.     |
                 +---------------+--------------------------------------------------+
                 | Release(=2)   | The client sends this event to the server when   |
                 |               | the shared object is deleted on the client side. |
                 +---------------+--------------------------------------------------+
                 | Request Change| The client sends this event to request that the  |
                 | (=3)          | change the value associated with a named         |
                 |               | parameter of the shared object.                  |
                 +---------------+--------------------------------------------------+
                 | Change (=4)   | The server sends this event to notify all        |
                 |               | clients, except the client originating the       |
                 |               | request, of a change in the value of a named     |
                 |               | parameter.                                       |
                 +---------------+--------------------------------------------------+
                 | Success (=5)  | The server sends this event to the requesting    |
                 |               | client in response to RequestChange event if the |
                 |               | request is accepted.                             |
                 +---------------+--------------------------------------------------+
                 | SendMessage   | The client sends this event to the server to     |
                 | (=6)          | broadcast a message. On receiving this event,    |
                 |               | the server broadcasts a message to all the       |
                 |               | clients, including the sender.                   |
                 +---------------+--------------------------------------------------+
                 | Status (=7)   | The server sends this event to notify clients    |
                 |               | about error conditions.                          |
                 +---------------+--------------------------------------------------+
                 | Clear (=8)    | The server sends this event to the client to     |
                 |               | clear a shared object. The server also sends     |
                 |               | this event in response to Use event that the     |
                 |               | client sends on connect.                         |
                 +---------------+--------------------------------------------------+
                 | Remove (=9)   | The server sends this event to have the client   |
                 |               | delete a slot.                                   |
                 +---------------+--------------------------------------------------+
                 | Request Remove| The client sends this event to have the client   |
                 | (=10)         | delete a slot.                                   |
                 +---------------+--------------------------------------------------+
                 | Use Success   | The server sends this event to the client on a   |
                 | (=11)         | successful connection.                           |
                 +---------------+--------------------------------------------------+
```

### 音频消息(Audio Message 8)
客户端或服务器发送此消息以向对等方发送音频数据。消息类型值8保留用于音频消息。

### 视频消息(Video Message 9)
客户端或服务器发送此消息以向对等方发送视频数据。消息类型值9保留用于视频消息。

### 聚合消息(Aggregate Message 22)
聚合消息是包含一系列RTMP子消息的单个消息。消息类型22用于聚合消息。
```
                 +---------+-------------------------+
                 | Header  |  Aggregate Message body |
                 +---------+-------------------------+
                     The Aggregate Message format
```
```
                 +--------+-------+---------+--------+-------+---------+ - - - -
                 |Header 0|Message|Back     |Header 1|Message|Back     |
                 |        |Data 0 |Pointer 0|        |Data 1 |Pointer 1|
                 +--------+-------+---------+--------+-------+---------+ - - - -
                                 The Aggregate Message body format
```
聚合消息的消息流(message stream)ID覆盖聚合内子消息的消息流ID。

聚合消息和第一个子消息的时间戳之间的差异是用于将子消息的时间戳重新规范化为流时间尺度的偏移量。偏移量被添加到每个子消息的时间戳中，以达到标准化流时间。第一个子消息的时间戳应与聚合消息的时间戳相同，因此偏移量应为零。

返回指针包含上一条消息的大小，包括消息头。包含它是为了匹配FLV文件的格式，并用于反向搜索。

使用聚合消息有几个性能优势：
- 块流可以近可能地在块内发送一条完整的消息。因此，增加块的大小并且使用聚合消息可以减少发送块的数量。
- 子消息可以连续存储在内存中。在进行系统调用以在网络上发送数据时效率更高。

### 用户控制消息事件(User Control Message Events 4)
客户端或服务器发送此消息以通知对等方用户控制事件。

支持以下用户控件事件类型：
``` 
                 +---------------+--------------------------------------------------+
                 |      Event    |                    Description                   |
                 +---------------+--------------------------------------------------+
                 |Stream Begin   | The server sends this event to notify the client |
                 | (=0)          | that a stream has become functional and can be   |
                 |               | used for communication. By default, this event   |
                 |               | is sent on ID 0 after the application connect    |
                 |               | command is successfully received from the        |
                 |               | client. The event data is 4-byte and represents  |
                 |               | the stream ID of the stream that became          |
                 |               | functional.                                      |
                 +---------------+--------------------------------------------------+
                 | Stream EOF    | The server sends this event to notify the client |
                 | (=1)          | that the playback of data is over as requested   |
                 |               | on this stream. No more data is sent without     |
                 |               | issuing additional commands. The client discards |
                 |               | the messages received for the stream. The        |
                 |               | 4 bytes of event data represent the ID of the    |
                 |               | stream on which playback has ended.              |
                 +---------------+--------------------------------------------------+
                 | StreamDry     | The server sends this event to notify the client |
                 | (=2)          | that there is no more data on the stream. If the |
                 |               | server does not detect any message for a time    |
                 |               | period, it can notify the subscribed clients     |
                 |               | that the stream is dry. The 4 bytes of event     |
                 |               | data represent the stream ID of the dry stream.  |
                 +---------------+--------------------------------------------------+
                 | SetBuffer     | The client sends this event to inform the server |
                 | Length (=3)   | of the buffer size (in milliseconds) that is     |
                 |               | used to buffer any data coming over a stream.    |
                 |               | This event is sent before the server starts      |
                 |               | processing the stream. The first 4 bytes of the  |
                 |               | event data represent the stream ID and the next  |
                 |               | 4 bytes represent the buffer length, in          |
                 |               | milliseconds.                                    |
                 +---------------+--------------------------------------------------+
                 | StreamIs      | The server sends this event to notify the client |
                 | Recorded (=4) | that the stream is a recorded stream. The        |
                 |               | 4 bytes event data represent the stream ID of    | 
                 |               | the recorded stream.                             |
                 +---------------+--------------------------------------------------+
                 | PingRequest   | The server sends this event to test whether the  |
                 | (=6)          | client is reachable. Event data is a 4-byte      |
                 |               | timestamp, representing the local server time    |
                 |               | when the server dispatched the command. The      |
                 |               | client responds with PingResponse on receiving   |
                 |               | MsgPingRequest.                                  |
                 +---------------+--------------------------------------------------+
                 | PingResponse  | The client sends this event to the server in     |
                 | (=7)          | response to the ping request. The event data is  |
                 |               | a 4-byte timestamp, which was received with the  |
                 |               | PingRequest request.                             |
                 +---------------+--------------------------------------------------+
``` 

## 命令类型(Types of Commands)
客户端和服务器交换AMF编码的命令。发送方发送一条命令消息，该消息由命令名(command name)、事务ID(transaction ID)和包含相关参数的命令对象(command object)组成。

例如，connect命令包含“app”参数，该参数告诉客户端连接到的服务器应用程序名称。接收方处理该命令并使用相同的事务ID发回响应。响应字符串可以是_result、_error或方法名称，例如verifyClient或contactExternalServer。

### 网络连接命令(NetConnection Commands)
NetConnection管理客户端应用程序和服务器之间的双向连接。此外，它还支持异步远程方法调用(remote method calls)。

可以在网络连接上发送以下命令：
- connect
- call
- close
- createStream

#### 连接(connect)
客户端向服务器发送connect命令，请求连接到服务器应用程序实例。

从客户端到服务器的命令结构如下：
```
                 +----------------+---------+---------------------------------------+
                 |   Field Name   |   Type  |                Description            |
                 +--------------- +---------+---------------------------------------+
                 |   Command Name |  String | Name of the command. Set to "connect".|
                 +----------------+---------+---------------------------------------+
                 | Transaction ID | Number  | Always set to 1.                      |
                 +----------------+---------+---------------------------------------+
                 | Command Object | Object  | Command information object which has  |
                 |                |         | the name-value pairs.                 |
                 +----------------+---------+---------------------------------------+
                 | Optional User  | Object  | Any optional information              |
                 | Arguments      |         |                                       |
                 +----------------+---------+---------------------------------------+
``` 
以下是connect命令的命令对象中使用的名称-值对的说明:
```
                 +-----------+--------+-----------------------------+----------------+
                 |  Property |   Type |          Description        |  Example Value |
                 +-----------+--------+-----------------------------+----------------+
                 | app       | String | The Server application name | testapp        |
                 |           |        | the client is connected to. |                |
                 +-----------+--------+-----------------------------+----------------+
                 | flashver  | String | Flash Player version. It is | FMSc/1.0       |
                 |           |        | the same string as returned |                |
                 |           |        | by the ApplicationScript    |                |
                 |           |        | getversion () function.     |                |
                 +-----------+--------+-----------------------------+----------------+
                 | swfUrl    | String | URL of the source SWF file  | file://C:/     |
                 |           |        | making the connection.      | FlvPlayer.swf  |
                 +-----------+--------+-----------------------------+----------------+
                 | tcUrl     | String | URL of the Server.          | rtmp://local   |
                 |           |        | It has the following format.| host:1935/test |
                 |           |        | protocol://servername:port/ | app/instance1  |
                 |           |        | appName/appInstance         |                |
                 +-----------+--------+-----------------------------+----------------+
                 | fpad      | Boolean| True if proxy is being used.| true or false  |
                 +-----------+--------+-----------------------------+----------------+
                 |audioCodecs| Number | Indicates what audio codecs | SUPPORT_SND    |
                 |           |        | the client supports.        | _MP3           |
                 +-----------+--------+-----------------------------+----------------+
                 |videoCodecs| Number | Indicates what video codecs | SUPPORT_VID    |
                 |           |        | are supported.              | _SORENSON      |
                 +-----------+--------+-----------------------------+----------------+
                 |videoFunct-| Number | Indicates what special video| SUPPORT_VID    |
                 |ion        |        | functions are supported.    | _CLIENT_SEEK   |
                 +-----------+--------+-----------------------------+----------------+
                 | pageUrl   | String | URL of the web page from    | http://        |
                 |           |        | where the SWF file was      | somehost/      |
                 |           |        | loaded.                     | sample.html    |
                 +-----------+--------+-----------------------------+----------------+
                 | object    | Number | AMF encoding method.        | AMF3           |
                 | Encoding  |        |                             |                |
                 +-----------+--------+-----------------------------+----------------+
```
audioCodecs属性的标志值：
```
                 +----------------------+----------------------------+--------------+
                 |       Codec Flag     |           Usage            |     Value    |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_NONE    | Raw sound, no compression  | 0x0001       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_ADPCM   | ADPCM compression          | 0x0002       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_MP3     | mp3 compression            | 0x0004       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_INTEL   | Not used                   | 0x0008       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_UNUSED  | Not used                   | 0x0010       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_NELLY8  | NellyMoser at 8-kHz        | 0x0020       | 
                 |                      | compression                |              |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_NELLY   | NellyMoser compression     | 0x0040       |
                 |                      | (5, 11, 22, and 44 kHz)    |              |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_G711A   | G711A sound compression    | 0x0080       |
                 |                      | (Flash Media Server only)  |              |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_G711U   | G711U sound compression    | 0x0100       |
                 |                      | (Flash Media Server only)  |              | 
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_NELLY16 | NellyMouser at 16-kHz      | 0x0200       |
                 |                      | compression                |              |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_AAC     | Advanced audio coding      | 0x0400       |
                 |                      | (AAC) codec                |              |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_SPEEX   | Speex Audio                | 0x0800       |
                 +----------------------+----------------------------+--------------+
                 |  SUPPORT_SND_ALL     | All RTMP-supported audio   | 0x0FFF       |
                 |                      | codecs                     |              |
                 +----------------------+----------------------------+--------------+
```
videoCodecs属性的标志值：
``` 
                 +----------------------+----------------------------+--------------+
                 |        Codec Flag    |             Usage          |     Value    |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_UNUSED   | Obsolete value             | 0x0001       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_JPEG     | Obsolete value             | 0x0002       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_SORENSON | Sorenson Flash video       | 0x0004       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_HOMEBREW | V1 screen sharing          | 0x0008       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_VP6 (On2)| On2 video (Flash 8+)       | 0x0010       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_VP6ALPHA | On2 video with alpha       | 0x0020       |
                 | (On2 with alpha      | channel                    |              |
                 | channel)             |                            |              |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_HOMEBREWV| Screen sharing version 2   | 0x0040       |
                 | (screensharing v2)   | (Flash 8+)                 |              |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_H264     | H264 video                 | 0x0080       |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_ALL      | All RTMP-supported video   | 0x00FF       |
                 |                      | codecs                     |              |
                 +----------------------+----------------------------+--------------+
```
videoFunction属性的标志值：
```
                 +----------------------+----------------------------+--------------+
                 |    Function Flag     |             Usage          |     Value    |
                 +----------------------+----------------------------+--------------+
                 | SUPPORT_VID_CLIENT   | Indicates that the client  | 1            |
                 | _SEEK                | can perform frame-accurate |              |
                 |                      | seeks.                     |              |
                 +----------------------+----------------------------+--------------+
```
对象编码属性的值：
```
                 +----------------------+----------------------------+--------------+
                 |     Encoding Type    |            Usage           |     Value    |
                 +----------------------+----------------------------+--------------+
                 | AMF0                 | AMF0 object encoding       | 0            |
                 |                      | supported by Flash 6 and   |              |
                 |                      | later                      |              |
                 +----------------------+----------------------------+--------------+
                 | AMF3                 | AMF3 encoding from         | 3            |
                 |                      | Flash 9 (AS3)              |              |
                 +----------------------+----------------------------+--------------+
```
从服务器到客户端的命令结构如下：
```
                 +--------------+----------+----------------------------------------+
                 | Field Name   | Type     | Description                            |
                 +--------------+----------+----------------------------------------+
                 | Command Name | String   | _result or _error; indicates whether   |
                 |              |          | the response is result or error.       |
                 +--------------+----------+----------------------------------------+
                 | Transaction  | Number   | Transaction ID is 1 for connect        |
                 | ID           |          | responses                              |
                 |              |          |                                        |
                 +--------------+----------+----------------------------------------+
                 | Properties   | Object   | Name-value pairs that describe the     |
                 |              |          | properties(fmsver etc.) of the         |
                 |              |          | connection.                            |
                 +--------------+----------+----------------------------------------+
                 | Information  | Object   | Name-value pairs that describe the     |
                 |              |          | response from|the server. ’code’,      |
                 |              |          | ’level’, ’description’ are names of few|
                 |              |          | among such information.                |
                 +--------------+----------+----------------------------------------+
```


```
                 +--------------+                              +-------------+
                 |    Client    |            |                 |    Server   |
                 +------+-------+            |                 +------+------+
                        |            Handshaking done                 |
                        |                    |                        |
                        |                    |                        |
                        |                    |                        |
                        |                    |                        |
                        |----------- Command Message(connect) ------->|
                        |                                             |
                        |<------- Window Acknowledgement Size --------|
                        |                                             |
                        |<----------- Set Peer Bandwidth -------------|
                        |                                             |
                        |-------- Window Acknowledgement Size ------->|
                        |                                             |
                        |<------ User Control Message(StreamBegin) ---|
                        |                                             |
                        |<------------ Command Message ---------------|
                        |        (_result- connect response)          |
                        |                                             |
                             Message flow in the connect command
```
执行命令期间的消息流为：
1. 客户端向服务器发送connect命令，请求与服务器应用程序实例连接。
2. 在接收到connect命令后，服务器向客户端发送协议消息(protocol message)“窗口确认大小(Window Acknowledgement Size)”。服务器还连接到connect命令中提到的应用程序。
3. 服务器向客户端发送协议消息(protocol message)“设置对等带宽(Set Peer Bandwidth)”。
4. 在处理协议消息“设置对等带宽”后，客户端向服务器发送协议消息(protocol message)“窗口确认大小(Window Acknowledgement Size)”。
5. 服务器将用户控制消息（StreamBegin）类型(属于另一个协议信息)发送到客户端。
6. 服务器发送结果命令消息，通知客户端连接状态(connection status)（成功/失败）。该命令指定事务ID(transaction ID)（connect命令始终等于1）。该消息还指定属性(properties)，例如闪存媒体服务器版本(Flash Media Server version)（字符串, string）。此外，它还指定了其他与连接响应相关的信息，如级别(level)（字符串, string）、代码(code)（字符串, string）、描述(description)（字符串, string）、对象编码(objectencoding)（编号, number）等。


#### 调用(Call)
NetConnection对象的调用(call)方法使接收端运行远程过程调用（remote procedure calls, RPC）。被调用的RPC名称作为参数传递给call命令。

从发送方到接收方的命令结构如下：
```
                 +--------------+----------+----------------------------------------+
                 |  Field Name  | Type     |              Description               |
                 +--------------+----------+----------------------------------------+
                 | Procedure    | String   | Name of the remote procedure that is   |
                 | Name         |          | called.                                |
                 +--------------+----------+----------------------------------------+
                 | Transaction  | Number   | If a response is expected we give a    |
                 |              |          | transaction Id. Else we pass a value of|
                 | ID           |          | 0                                      |
                 +--------------+----------+----------------------------------------+
                 | Command      | Object   | If there exists any command info this  |
                 | Object       |          | is set, else this is set to null type. |
                 +--------------+----------+----------------------------------------+
                 | Optional     | Object   | Any optional arguments to be provided  |
                 | Arguments    |          |                                        |
                 +--------------+----------+----------------------------------------+
```
响应的命令结构如下所示：
```
                 +--------------+----------+----------------------------------------+
                 | Field Name   | Type     | Description                            |
                 +--------------+----------+----------------------------------------+
                 | Command Name | String   | Name of the command.                   |
                 |              |          |                                        |
                 +--------------+----------+----------------------------------------+
                 | Transaction  | Number   | ID of the command, to which the        |
                 | ID           |          | response belongs.                      |
                 +--------------+----------+----------------------------------------+
                 | Command      | Object   | If there exists any command info this  |
                 | Object       |          | is set, else this is set to null type. |
                 +--------------+----------+----------------------------------------+
                 | Response     | Object   | Response from the method that was      |
                 |              |          | called.                                |
                 +------------------------------------------------------------------+
```

#### 创建流(createStream)
客户端将此命令发送到服务器，以创建用于消息通信的逻辑通道。音频、视频和元数据的发布通过使用createStream命令创建的流通道执行。

NetConnection是默认的通信通道，其流ID为0。协议和一些命令消息（包括createStream）使用默认的通信通道。

从客户端到服务器的命令结构如下：
```
                 +--------------+----------+----------------------------------------+
                 | Field Name   | Type     |            Description                 |
                 +--------------+----------+----------------------------------------+
                 | Command Name | String   | Name of the command. Set to            |
                 |              |          | "createStream".                        |
                 +--------------+----------+----------------------------------------+
                 | Transaction  | Number   | Transaction ID of the command.         |
                 | ID           |          |                                        |
                 +--------------+----------+----------------------------------------+
                 | Command      | Object   | If there exists any command info this  |
                 | Object       |          | is set, else this is set to null type. |
                 +--------------+----------+----------------------------------------+
```
从服务器到客户端的命令结构如下：
```
             +--------------+----------+----------------------------------------+
             | Field Name   | Type     |              Description               |
             +--------------+----------+----------------------------------------+
             | Command Name | String   | _result or _error; indicates whether   |
             |              |          | the response is result or error.       |
             +--------------+----------+----------------------------------------+
             | Transaction  | Number   | ID of the command that response belongs|
             | ID           |          | to.                                    |
             +--------------+----------+----------------------------------------+
             | Command      | Object   | If there exists any command info this  |
             | Object       |          | is set, else this is set to null type. |
             +--------------+----------+----------------------------------------+
             | Stream       | Number   | The return value is either a stream ID |
             | ID           |          | or an error information object.        |
             +--------------+----------+----------------------------------------+
```

### 网络流命令(NetStream命令)
NetStream定义了一个通道，通过该通道，流式音频(audio)、视频(video)和数据消息(data messages)可以通过连接客户端和服务器的网络连接进行传输。NetConnection对象可以支持多个NetStream以支持多个数据流。

客户端可以通过NetStream将以下命令发送到服务器：
- play
- play2
- deleteStream
- closeStream
- receiveAudio
- receiveVideo
- publish
- seek
- pause

服务器使用“onStatus”命令向客户端发送NetStream状态更新：
```
             +--------------+----------+----------------------------------------+
             | Field Name   | Type     |              Description               |
             +--------------+----------+----------------------------------------+
             | Command Name | String   | The command name "onStatus".           |
             +--------------+----------+----------------------------------------+
             | Transaction  | Number   | Transaction ID set to 0.               |
             | ID           |          |                                        |
             +--------------+----------+----------------------------------------+
             | Command      | Null     | There is no command object for         |
             | Object       |          | onStatus messages.                     |
             +--------------+----------+----------------------------------------+
             | Info Object  | Object   | An AMF object having at least the      |
             |              |          | following three properties: "level"    |
             |              |          | (String): the level for this message,  |
             |              |          | one of "warning", "status", or "error";|
             |              |          | "code" (String): the message code, for |
             |              |          | example "NetStream.Play.Start"; and    | 
             |              |          | "description" (String): a human-       |
             |              |          | readable description of the message.   |
             |              |          | The Info object MAY contain other      |
             |              |          | properties as appropriate to the code. |
             +--------------+----------+----------------------------------------+
                       Format of NetStream status message commands.
```

#### 播放(play)
客户端将此命令发送到服务器以播放流。也可以使用此命令多次创建播放列表。

如果您想创建一个动态播放列表，在不同的直播或录制流之间切换，请多次调用play，每次都将false传递给reset。相反，如果要立即播放指定的流，清除排队等待播放的任何其他流，请传递true给reset。

从客户端到服务器的命令结构如下：
```
                 +--------------+----------+-----------------------------------------+
                 | Field Name   | Type     |                 Description             |
                 +--------------+----------+-----------------------------------------+
                 | Command Name | String   | Name of the command. Set to "play".     |
                 +--------------+----------+-----------------------------------------+
                 | Transaction  | Number   | Transaction ID set to 0.                |
                 | ID           |          |                                         |
                 +--------------+----------+-----------------------------------------+
                 | Command      | Null     | Command information does not exist.     |
                 | Object       |          | Set to null type.                       |
                 +--------------+----------+-----------------------------------------+
                 | Stream Name  | String   | Name of the stream to play.             |
                 |              |          | To play video (FLV) files, specify the  |
                 |              |          | name of the stream without a file       |
                 |              |          | extension (for example, "sample"). To   |
                 |              |          | play back MP3 or ID3 tags, you must     |
                 |              |          | precede the stream name with mp3:       |
                 |              |          | (for example, "mp3:sample". To play     |
                 |              |          | H.264/AAC files, you must precede the   |
                 |              |          | stream name with mp4: and specify the   |
                 |              |          | file extension. For example, to play the|
                 |              |          | file sample.m4v,specify "mp4:sample.m4v"|
                 |              |          |                                         |
                 +--------------+----------+-----------------------------------------+
                 | Start        | Number   | An optional parameter that specifies    |
                 |              |          | the start time in seconds. The default  |
                 |              |          | value is -2, which means the subscriber |
                 |              |          | first tries to play the live stream     |
                 |              |          | specified in the Stream Name field. If a|
                 |              |          | live stream of that name is not found,it|
                 |              |          | plays the recorded stream of the same   |
                 |              |          | name. If there is no recorded stream    |
                 |              |          | with that name, the subscriber waits for|
                 |              |          | a new live stream with that name and    |
                 |              |          | plays it when available. If you pass -1 |
                 |              |          | in the Start field, only the live stream|
                 |              |          | specified in the Stream Name field is   |
                 |              |          | played. If you pass 0 or a positive     |
                 |              |          | number in the Start field, a recorded   |
                 |              |          | stream specified in the Stream Name     |
                 |              |          | field is played beginning from the time |
                 |              |          | specified in the Start field. If no     |
                 |              |          | recorded stream is found, the next item |
                 |              |          | in the playlist is played.              |
                 +--------------+----------+-----------------------------------------+
                 | Duration     | Number   | An optional parameter that specifies the|
                 |              |          | duration of playback in seconds. The    |
                 |              |          | default value is -1. The -1 value means |
                 |              |          | a live stream is played until it is no  |
                 |              |          | longer available or a recorded stream is|
                 |              |          | played until it ends. If you pass 0, it |
                 |              |          | plays the single frame since the time   |
                 |              |          | specified in the Start field from the   |
                 |              |          | beginning of a recorded stream. It is   |
                 |              |          | assumed that the value specified in     |
                 |              |          | the Start field is equal to or greater  |
                 |              |          | than 0. If you pass a positive number,  |
                 |              |          | it plays a live stream for              |
                 |              |          | the time period specified in the        |
                 |              |          | Duration field. After that it becomes   |
                 |              |          | available or plays a recorded stream    |
                 |              |          | for the time specified in the Duration  |
                 |              |          | field. (If a stream ends before the     |
                 |              |          | time specified in the Duration field,   |
                 |              |          | playback ends when the stream ends.)    |
                 |              |          | If you pass a negative number other     |
                 |              |          | than -1 in the Duration field, it       |
                 |              |          | interprets the value as if it were -1.  |
                 +--------------+----------+-----------------------------------------+
                 | Reset        | Boolean  | An optional Boolean value or number     |
                 |              |          | that specifies whether to flush any     |
                 |              |          | previous playlist.                      |
                 +--------------+----------+-----------------------------------------+
```


```
                       +-------------+                             +----------+
                       | Play Client |             |               | Server   |
                       +------+------+             |               +-----+----+
                              |           Handshaking and Application    |
                              |                connect done              |
                              |                    |                     |
                              |                    |                     |
                              |                    |                     |
                              |                    |                     |
                    ---+----  |------Command Message(createStream) ----->|
                 Create|      |                                          |
                 Stream|      |                                          |
                    ---+----  |<---------- Command Message --------------|
                              |     (_result- createStream response)     |
                              |                                          |
                    ---+----  |------ Command Message (play) ----------->|
                       |      |                                          |
                       |      |<------------ SetChunkSize ---------------|
                       |      |                                          |
                       |      |<---- User Control (StreamIsRecorded) ----|
                  Play |      |                                          |
                       |      |<---- UserControl (StreamBegin) ----------|
                       |      |                                          |
                       |      |<--Command Message(onStatus-play reset) --|
                       |      |                                          |
                       |      |<--Command Message(onStatus-play start) --|
                       |      |                                          |
                       |      |<-------------Audio Message---------------|
                       |      |                                          |
                       |      |<-------------Video Message---------------|
                       |      |                   |                      | 
                                                  |
                         Keep receiving audio and video stream till finishes
                                 Message flow in the play command
```








