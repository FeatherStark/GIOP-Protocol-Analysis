# GIOP-Protocol-Analysis

本文 GIOP 协议是基于 Weblogic 中的 IIOP 协议进行分析和总结。

GIOP（General Inter-ORB Protocol）是一种CORBA规范定义的协议，用于在分布式对象之间进行通信和交互，定义了对象请求、响应、异常、命名等基本的通信模式和协议规范。

GIOP消息由两部分组成：GIOP消息头和消息体。

## GIOP消息头（Message Header）

GIOP消息头是GIOP消息中的前12个字节。

### 标准的GIOP消息头

```
GIOP Header(HEX)：47494f500102000300000017
Magic：GIOP【47494f50】
Major Version：GIOP大版本是1【01】
Minor Version：GIOP小版本是2【02】
Message Flags：普通的消息【00】->【0x00】
Message type：LocateRequest【03】
Message size：消息体的总长度是23【17】
```




| GIOP消息头字段 | 中文名     | 长度(字节)    | 说明                                                         |
| -------------- | ---------- | ------------- | ------------------------------------------------------------ |
| Magic          | GIOP标识   | 4             | 表示这是一个GIOP消息，固定内容为`0x47 0x49 0x4F 0x50`        |
| GIOP Version   | GIOP版本   | 2             | 表示 GIOP 协议的版本号，`Major Version`表示GIOP的大版本号，`Minor Version`表示GIOP的小版本号 |
| Message Flags  | 标志位     | 1(Big Endian) | 包含一些标志位。比如：`0x00` 标识普通的请求消息，`0x02` 标识消息的结束，`0x10` 标识一个请求的最后一个片段。 |
| Message Type   | 消息类型   | 1             | 表示消息的类型。比如：`0` 表示请求消息（Request）;`1` 表示一个回复消息（Reply）;`3` 表示一个定位请求消息（Locate Request）等。 |
| Message size   | 消息体长度 | 4             | 表示整个GIOP消息体的长度，以字节为单位。                     |

### Magic（GIOP标识）

Magic表示该消息是一个GIOP协议的消息，长度为4字节，固定值(hex)： `0x47 0x49 0x4F 0x50`。

### GIOP Version（GIOP版本）

GIOP Version表示GIOP协议使用的版本，长度为2字节，包含两个字段 `Major Version`和`Minor Version`，其中，`Major Version`表示GIOP的大版本号，`Minor Version`表示GIOP的小版本号。


### Message Flags（标志位）

Message Flags字段表示一些特殊的消息标识，16进制的长度为1字节，Message Flags字段是一个Big Endian类型的值，共包含32个比特位，每个比特位代表一种属性或特性。

| Bit位置 | 16进制标识 | 含义                                           |
| ------- | ---------- | ---------------------------------------------- |
| 0       | 0x00000001 | 消息包含一个GIOP异常                           |
| 1       | 0x00000001 | 消息是最后一个消息                             |
| 2       | 0x00000004 | 消息是一个一段消息（Fragment）                 |
| 3       | 0x00000008 | 消息是一个片段的最后一个消息                   |
| 4       | 0x00000010 | 消息是一个请求的最后一个片段                   |
| 5       | 0x00000020 | 消息需要响应                                   |
| 6-15    | 0x000007C0 | 预留，保留将来使用                             |
| 16-31   | 0xFFFF0000 | CORBA规定，这些比特位必须为0，用于将来扩展使用 |

注：如果GIOP消息头中的Message Flags字段的16进制值是00，则说明该消息的所有32个比特位都被设置为0，即没有任何标志位被设置。这表示该GIOP消息是一个普通的消息，没有任何特殊属性，例如异常、一段消息、响应等。这种消息通常用于进行普通的请求和响应通信，例如获取对象的引用、调用对象的方法等。

### Message Type（消息类型）

GIOP消息头中的Message Type字段是一个重要的字段，用于标识GIOP消息的类型和目的，长度为1字节。

| 取值 | 消息格式类型    | 消息发起者    | 说明                                                         |
| ---- | --------------- | ------------- | ------------------------------------------------------------ |
| 0    | Request         | 客户端        | 客户端调用服务器端的服务时，需要发送请求消息，该消息携带和操作相关的信息。 请求消息消息头中包含了服务上下文、请求标识、应答模式、对象键值、操作名称等 信息。操作名称用于识别操作属性和目标对象。请求消息体内包含了请求相关的输入 输出参数，以及附加服务上下文。 |
| 1    | Reply           | 服务端        | 在需要应答时，服务器端会发送应答消息给客户端，消息内包含有和该请求相关执行 结果、参数等信息。根据操作结果的不同，后续的应答报文格式和长度也会不同。 |
| 2    | CancelRequest   | 客户端        | 客户端对于远程调用可能使用一个指定的超时时间。当指定的时间到达时，如果客户 端还没有收到应答消息，这时可以使用取消请求消息来取消之前的调用。在发出取消 请求消息后，客户端不再处理被取消的消息应答。 |
| 3    | LocateRequest   | 客户端        | GIOP 对于底层传输的假设是面向连接的。在进行大数据量的交互时，如果不确定服务 对象是否可用，而直接进行服务请求的调用，可能造成大量无效数据在传输层的占用， 影响传输效率。因此 GIOP 提供了一种类似 ping 的功能，通过发送定位请求消息，可 以探测服务是否可用。 |
| 4    | LocateReply     | 服务端        | ORB接收到Locate Request消息后，会查找并返回该对象的位置信息，这个位置信息会封装在一个Locate Reply消息中返回给客户端。 |
| 5    | Fragment        | 客户端/服务端 | Fragment和Fragment Reply消息用于在消息传输过程中进行分段和重组。如果一个消息太大，无法在一个单独的消息中传输，那么可以将这个消息分成多个片段进行传输，每个片段使用Fragment消息表示。 |
| 6    | FragmentReply   | 客户端/服务端 | 接收端会将Fragment片段重新组装成完整的消息，并使用Fragment Reply消息进行响应。 |
| 7    | CloseConnection | 服务端        | 允许关闭连接，比如关闭处于空闲状态的连接。例如一个客户到服务器的连接空闲一 段时间后，服务器端决定关闭连接，但是客户端可能也会发出关闭连接消息。为了防 止上述假象同时发生，服务器在关闭连接前，会先给客户端发一条关闭连接消息。当 客户端收到该消息时，客户端认为服务器对于它之前发出的而未收到的应答的请求不 再响应，使用新的连接来进行之前的请求，避免了连接限于死等的状态 |

### Message size（消息长度）

Message size字段表示整个GIOP消息体的长度，长度为4字节。

在发送GIOP消息时，需要将整个消息的长度设置为Message size字段的值。接收方在解析GIOP消息时，会先读取Message size字段的值，然后根据该值读取整个GIOP消息的内容。

## GIOP消息体（Message Body）

在 IIOP 协议中，消息体（Message Body）是消息的一个重要组成部分，它包含了消息的具体内容，比如请求的方法名、方法参数、响应结果等。消息体的结构和内容由协议规定，不同的消息类型有不同的消息体结构和内容。

在请求消息中，消息体包含了请求的具体信息，包括请求的方法名、方法参数、请求标识符等。在响应消息中，消息体包含了响应的具体结果，包括响应结果的值、异常信息等。


| 消息体字段         | 中文名           | 长度(字节) | 说明                                                         |
| ------------------ | ---------------- | ---------- | ------------------------------------------------------------ |
| Request id         | 消息标识符       | 4          | 标识请求消息和其对应的响应消息。                             |
| Response flags     | 响应标识位       | 1          | 标识RMI服务器对客户端请求的响应信息的一组标志位              |
| Reserved           | 保留字段         | 3          | 协议的扩展或未来版本的兼容性考虑。                           |
| TargetAddress      | 请求目标对象键ID | 2          | 请求的目标对象的对象键（Object Key）。                       |
| Key Address length | Key字节长度      | 无固定长度 | 标识Object Key的长度（以字节为单位）。                       |
| Key Address        | Key地址          | 无固定长度 | 用于定位 CORBA 服务中的对象。                                |
| Operation length   | 操作方法长度     | 无固定长度 | Operation的字节长度。                                        |
| Request operation  | 操作方法         | 无固定长度 | 标识客户端希望在服务器端执行的操作或方法。                   |
| ServiceContextList | 服务上下文列表   | 无固定长度 | ServiceContextList包含整个SequenceLength和ServiceContext的内容。 |
| Sequence Length    | 序列个数         | 4          | 标识ServiceContext的个数，如果是在ServiceContext字段内，则表示Dndianness和Context Data两个字段的总长度。 |
| ServiceContext     | 服务上下文       | 无固定长度 | 用于在 CORBA 消息中携带一些相关的上下文信息。                |
| ContextID          | 上下文类型       | 4字节      | 标识常见的上下文信息类型。                                   |
| Endianess          | 字节序           | 1          | 用于标识发送方和接收方使用的字节序，取值为0或1。             |
| Context Data       | 上下文数据       | 无固定长度 | 用于传递关于消息的元数据。                                   |
| Stub Data          | 存根数据         | 无固定长度 | 用于描述远程对象的代理信息。                                 |

### Request id（消息标识符）

Request ID 是一个标识符，用于标识请求消息和其对应的响应消息。在客户端发起请求时，客户端会分配一个唯一的请求 ID，然后将该 ID 添加到请求消息头部，服务器收到请求消息后，将该 ID 添加到响应消息头部，这样客户端就可以通过 Request ID 来匹配响应消息，从而确认该响应消息是对哪个请求消息的响应。

Request ID 是一个无符号整数，通常是 4 字节（32 位）或 8 字节（64 位），取值范围为 1 到 2^31-1 或 1 到 2^63-1，请求 ID 值必须在同一连接上是唯一的，因为在同一连接上可能同时存在多个请求和响应消息。

### Response flags（响应标识位）

Response flags是用来标识RMI服务器对客户端请求的响应信息的一组标志位，长度为1个字节。当Response flags的值为3（SYNC_WITH_TARGET）时，它表示在向客户端发送响应消息之前，RMI服务器必须将响应消息与目标对象同步。同步意味着在处理响应消息之前，RMI服务器必须确保与目标对象相关的状态已经完全更新，以确保响应消息的准确性。

在RMI-IIOP协议中，SyncScope具有三种不同的值：

1. SYNC_NONE - 表示响应消息不需要与任何对象同步。
2. SYNC_WITH_TARGET - 表示响应消息需要与目标对象同步。
3. SYNC_WITH_TRANSPORT - 表示响应消息需要与传输层同步，以确保消息在网络上正确传输。

### Reserved（保留字段）

Reserved的含义是保留字段，通常用于协议的扩展或未来版本的兼容性考虑。

Reserved字段的长度为3个字节，但是其具体的含义和使用方式是未定义的，即目前的IIOP规范中没有对其进行明确的定义和规定。这意味着，不同的IIOP实现或不同的版本中，Reserved字段的含义和使用方式可能会不同。

### TargetAddress（请求目标对象键ID）

在 GIOP 协议的请求体中，TargetAddress: KeyAddr(0) 表示请求的目标对象的对象键（Object Key）。Object Key 是分布式对象的唯一标识符，用于在分布式系统中唯一地标识对象。IIOP 中使用 Object Reference 来标识对象，其中包括 Object Key 以及其他信息（如对象所在主机的 IP 地址和端口号等）。

在请求体中，TargetAddress 表示对象所在的网络地址，而 KeyAddr(0) 则表示 Object Key 的长度为 0，即该请求没有指定具体的 Object Key，而是需要服务端根据其他信息来确定目标对象。这种情况下，服务端需要进行一些额外的查找操作，以确定请求的目标对象。

### Key Address length（Key字节长度）

在GIOP协议中，`Key length`是指GIOP消息请求体中，标识`Key Address`的长度（以字节为单位）。

### Key Address（Key地址）

在 GIOP（General Inter-ORB Protocol，通用中间件协议）消息中，Key Address 是请求体中的一部分，用于指示请求的对象。它由一个整数长度和一个字节数组组成。其中，字节数组通常包含对象的名称和位置信息。Key Address 用于定位 CORBA 服务中的对象，以便能够执行相关的操作。


### Operation length（操作方法长度）

Operation length标识operation的字节长度。如果请求体中的 Operation 字段值为"getUserInfo"，那么 Operation length 字段的值为 11，因为这个字符串有 11 个字节。在解析请求时，接收方使用 Operation length 字段来确定接下来要解析的方法名的字节数。

### Request operation（操作方法）

Request operation（请求操作）用于标识客户端希望在服务器端执行的操作或方法。例如，如果客户端要求服务器返回某个对象的属性值，Request operation 字段将包含一个对应于“获取属性”操作的标识符。如果客户端请求执行的是一个方法，Request operation 字段通常包含方法名称和参数列表。服务器端在接收到请求后，将使用 Request operation 字段来决定执行哪个操作或方法，并生成相应的响应消息返回给客户端。

### ServiceContextList（服务上下文列表）

ServiceContextList是一个可选的字段，它包含一个或多个ServiceContext对象，用于传递有关消息上下文的元数据信息。它可以包含有关消息的语言、字符编码、安全和事务等方面的信息。

ServiceContextList通常用于传递有关客户端和服务器之间的上下文信息，以便双方能够理解对方的请求和响应。例如，在分布式事务处理中，ServiceContextList可以用于传递有关事务上下文的信息，以确保所有参与者都在同一事务范围内工作。

ServiceContextList的具体格式是由CORBA定义的，可以在IIOP消息中以二进制形式进行传输。它由一个ServiceContextList头部和一个或多个ServiceContext组成。每个ServiceContext包含一个标识符和一个字节数组，用于传输相关数据。

### Sequence Length（序列个数）

Sequence Length用于标识ServiceContext的个数。

如果是在ServiceContext字段内，Sequence Length表示的是这些参数在序列化后的总字节数，则表示Dndianness和Context Data两个字段的总长度。

 ### ServiceContext（服务上下文）

ServiceContext（服务上下文）是 CORBA 中一种重要的元数据机制，用于在 CORBA 消息中携带一些相关的上下文信息。

在 CORBA 中，客户端与服务器端之间的交互都是通过消息来完成的，其中包含了方法调用请求和响应的信息。而服务上下文（ServiceContext）则是一种机制，用于在消息中携带一些额外的信息，这些信息可以帮助客户端和服务器端更好地进行交互和处理请求。

ServiceContext 由一个 context_id 和一个 Octet 序列组成。其中 context_id 用于标识上下文类型，而 Octet 序列则包含了具体的上下文信息。在 CORBA 中，ServiceContext 通常用于传递一些非标准的信息，例如安全信息、日志信息、性能信息等。context_id 可以用于指定特定的服务或操作，以便远程 CORBA 对象知道如何处理请求。通常，context_id 是使用 Object Request Broker (ORB) 的 Naming Service 注册的名称或 ID，以便可以通过名称或 ID 查找特定服务或操作。

在 IIOP 协议中，ServiceContext 通常包含在 GIOP 消息中，用于携带额外的信息。一个 GIOP 消息可以包含多个 ServiceContext，每个 ServiceContext 的 context_id 可以不同，用于区分不同类型的上下文信息。

### ContextID（上下文类型）

ContextID用于标识常见的上下文信息类型，长度为4字节。其CORBA预设了一些ContextID,每个 CORBA 实现可能都支持不同的 Context ID。因此，需要查看具体实现的文档来确定所支持的 Context ID 值。

**RMI-IIOP中定义的ContextID**

| ContextID (RMI-IIOP) | 含义                                                         |
| -------------------- | ------------------------------------------------------------ |
| 0x00000000           | 表示一个标准的上下文（Standard Context），是一个默认的、空的ServiceContext类型。上下文信息可以在请求的上下文中包含其他信息，例如事务、安全认证等，但在此情况下，这个ServiceContext没有任何信息要传递。 |
| 0x00000001           | 表示一个Codebase上下文（Codebase Context），是一种用于指定类的加载器的上下文类型。当客户端需要使用服务端的某个类时，服务端可以在Codebase Context中指定这个类的加载器。这样，客户端在接收到服务端返回的类信息后，可以使用这个加载器将这个类加载到客户端的JVM中。 |
| 0x00000005           | 表示Transaction Service上下文。Transaction Service是Java RMI规范中定义的一种服务，用于支持分布式事务。该上下文包含了与事务相关的信息，如事务ID、超时时间等。 |
| 0x00000006           | 表示一个Codebase上下文（Codebase Context）。Codebase上下文是一种特殊的上下文类型，它用于传递代码库的URL地址信息。在RMI-IIOP协议中，如果客户端需要加载服务端的远程对象，并且服务端的远程对象包含代码库的引用，客户端需要知道代码库的URL地址才能加载相关的代码库。因此，在这种情况下，服务端需要使用Codebase上下文来传递代码库的URL地址信息给客户端。 |
| 0x0000000f           | 用于在RMI-IIOP协议请求体中传输Java对象的序列化版本号信息，以确保对象的正确序列化和反序列化。 |
| 0x42454100           | 表示一个标准上下文（Standard Context）。标准上下文是一种通用的上下文类型，它用于传递各种不同类型的上下文信息。在RMI-IIOP协议中，如果请求中需要传递一些通用的上下文信息，可以使用标准上下文来进行传递。 |
| 0x4245410e           | 表示一个Java RMI上下文管理器（Java RMI Context Manager）     |

Java RMI Context Manager 是一种上下文类型，它用于管理Java RMI调用之间的上下文信息，包括Java RMI上下文、ClassLoader等，并确保正确的上下文信息传递给相应的调用。

### Endianess（字节序）

节序（Endianness）指的是一个多字节整数类型在内存中存储的顺序。在 IIOP 中，使用 ServiceContext 来传递一些上下文信息，其中 Endianess 字段用于指示发送方和接收方使用的字节序。

Endianess 字段是一个布尔类型的字段，它占用了一个字节。它有两个可能的取值，即 0 和 1，分别对应小端字节序和大端字节序。如果发送方使用的是小端字节序，则 Endianess 字段的值为 0；如果发送方使用的是大端字节序，则 Endianess 字段的值为 1。

当发送方和接收方的字节序不一致时，接收方需要进行字节序转换才能正确地解析 IIOP 消息。在 ServiceContext 中使用 Endianess 字段可以帮助接收方确定消息的字节序，从而正确地进行字节序转换。

### Context Data（上下文数据）

在IIOP消息的ServiceContext中，ContextData字段指的是上下文数据，用于传递关于消息的元数据。具体来说，ContextData是一个可变长度的字节数组，包含了与上下文相关的数据，其具体内容根据上下文类型（ContextID）而不同。

举例来说，如果上下文类型是ORB_INITIAL_REFERENCE（0），那么ContextData中就包含了ORB的初始引用；如果上下文类型是CodeSets（1），则ContextData中包含了编码集信息等。

需要注意的是，ContextData的具体编码格式与上下文类型相关。有些上下文类型可能需要特定的编码格式来解析ContextData。因此，在处理IIOP消息时，需要根据上下文类型来正确解析ContextData。

### Stub Data（存根数据）

Stub Data包含了客户端需要调用的远程对象的信息，包括远程对象的唯一标识符、对象的方法列表、以及对象的位置等。在客户端发起远程方法调用时，客户端的Stub（代理对象）会将这些信息封装成请求体中的Stub Data，并通过IIOP协议将请求发送给远程对象的服务端。

服务端接收到请求后，会解析请求体中的Stub Data，找到对应的远程对象，并调用对象的方法。由于Stub Data中包含了对象的位置信息，因此服务端可以根据这些信息找到正确的对象进行调用，实现了远程方法调用的功能。
