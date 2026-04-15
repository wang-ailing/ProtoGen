# ProtoGen DSL (ProtoCC) 规范说明

ProtoGen 使用了一种专门为缓存一致性协议设计的领域特定语言（Domain-Specific Language），称为 **ProtoCC**（通常以 `.pcc` 为文件后缀）。

ProtoCC 的核心设计理念是：**让设计者只需编写协议的顺序规范（Sequential Specification Protocol, SSP），编译器会自动生成处理并发竞争和中间状态的完整协议状态机，并编译为 Murphi 模型代码。**

下面以 `MSI_Proto.pcc` 为例，解析 ProtoCC 的语法结构。

---

## 1. 系统参数与网络配置

协议的开头通常定义全局参数和网络通道类型：

```c
# NrCaches 3

Network { 
    Ordered fwd;    // 有序通道，例如：FwdGetS, FwdGetM, Inv, PutAck
    Unordered resp; // 无序通道，例如：Data, InvAck
    Unordered req;  // 无序通道，例如：GetS, GetM, PutM
};
```
- `# NrCaches`：定义缓存节点的数量。
- `Network`：定义网络通道（Channel）。支持 `Ordered`（保序通道）和 `Unordered`（无序通道）。这会直接影响后续生成 Murphi 代码时的网络行为。

---

## 2. 节点定义 (Machine Definition)

定义系统中的组件机器（如 Cache、Directory）及其内部状态变量：

```c
Cache {
    State I;                           // 初始状态
    Data cl;                           // 缓存行数据
    int[0..NrCaches] acksReceived = 0; // 接收到的 Ack 数量计数器
    int[0..NrCaches] acksExpected = 0; // 期望的 Ack 数量
} set[NrCaches] cache;                 // 声明这是一个包含 NrCaches 个实例的集合

Directory {
    State I;
    Data cl;
    set[NrCaches] ID cache;            // 记录当前缓存了该数据的节点集合（Sharers 列表）
    ID owner;                          // 记录当前拥有独占权限的节点 ID
} directory;
```
- `State`：定义状态机的初始状态（也是保留字，用于表示状态枚举）。
- `set[NrCaches] ID`：一种集合类型，用来存储节点 ID（如 Directory 的 Sharers 列表）。

---

## 3. 消息定义 (Message Types)

定义协议中传输的消息格式：

```c
Message Request{};

Message Ack{};

Message Resp{
    Data cl;
};

Message RespAck{
    Data cl;
    int[0..NrCaches] acksExpected;
};
```
- 可以定义不同类型的消息体，大括号内可以定义消息所携带的负载（Payload），比如数据 `cl` 或者需要等待的确认数量 `acksExpected`。

---

## 4. 架构与状态机 (Architecture Behavior)

这是 ProtoCC 最核心的部分，用于定义各个机器（如 Cache 或 Directory）的事件处理逻辑（Transitions）。

### 4.1 声明稳定状态
```c
Architecture cache {
    Stable{I, S, M}
```
- `Stable`：显式声明该状态机中的**稳定状态**。ProtoGen 算法会基于这些稳定状态自动推导并插入并发过程中的瞬态（Transient States，如 `I_to_S`、`M_evict` 等）。

### 4.2 状态转移 (Process / Rule)
定义在某个特定状态下，接收到特定事件（本地请求或网络消息）时的行为：

```c
// 语法：Process(当前状态, 触发事件, [可选的下一个状态])
Process(I, load, State){
    // 发送请求消息
    msg = Request(GetS, ID, directory.ID);
    req.send(msg);

    // 阻塞等待响应 (SSP 顺序语法的核心)
    await{
        when GetS_Ack:
            cl = GetS_Ack.cl;
            State = S;
            break;
    }
}
```

#### 关键操作说明：
- `Process(CurrentState, Event, [NextState])`：定义状态转移规则。`NextState` 可以直接硬编码（如 `S`），也可以使用变量 `State` 并在内部进行动态赋值。
- **消息发送**：
  ```c
  msg = Request(GetS, ID, directory.ID); // 构造消息：类型为GetS，源为ID，目的地为directory.ID
  req.send(msg);                         // 通过 req 通道发送
  ```
- **同步阻塞（Await / When）**：
  这是 ProtoCC 与标准 Murphi 或其他硬件描述语言最大的不同。在编写 `.pcc` 规范时，**允许使用 `await` 语句模拟“阻塞等待”**。
  例如发送 `GetS` 后，代码可以直接写 `await { when GetS_Ack: ... }`。
  *编译器会自动将这个 `await` 语句拆分并转换为非阻塞的并发状态机，自动生成类似 `I_load` 这样的中间过渡状态。*

### 4.3 集合操作
ProtoCC 内置了对节点集合（Set）的简易操作支持（常用于 Directory）：
```c
Process(S, Upgrade){
    if cache.contains(Upgrade.src){
        cache.del(Upgrade.src);
        msg = RespAck(GetM_Ack_AD, ID, Upgrade.src, cl, cache.count());
        resp.send(msg);
        State = M;
        break;
    }
}
```
- `cache.add(ID)` / `cache.del(ID)`：向集合中添加或移除节点。
- `cache.contains(ID)`：判断集合是否包含某个节点。
- `cache.count()`：获取集合中的节点数量。

---

## 总结

ProtoGen DSL (ProtoCC) 是一种高度抽象的、面向顺序逻辑的协议描述语言。
开发者只需要以**阻塞式的视角（Send -> Await -> Action）**编写协议在无竞争情况下的理想转移流程。ProtoGen 会通过底层的算法，分析 `await` 块和通道依赖，**自动生成处理所有网络并发、消息乱序、竞争条件的完整 Murphi 状态机**。