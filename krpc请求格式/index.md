# KRPC请求格式


所有的 KRPC 消息都是通过 UDP 传输的、经过 Bencode 编码的字典。
本文主要介绍`find_node`、 `get_peers` 和 `announce_peer` 这三个 RPC 调用的请求和响应消息结构。

**通用结构元素:**

- **`t`**: 事务 ID (Transaction ID)，二进制字符串，由请求方设置，响应方必须在响应中原样返回。
- **`y`**: 消息类型 (Message Type)，单个字符，'q' 代表查询 (Query)，'r' 代表响应 (Response)，'e' 代表错误 (Error)。
- **`v`** (可选): 客户端版本信息 (Version)，通常是字符串。
- **`ip`** (可选，通常在响应中出现): 响应者看到的请求者的公网 IP 地址和端口 (紧凑的 6 字节二进制字符串)。

---

### 1、`find_node` RPC

**目的:** 向一个节点请求，获取它所知道的、其 Node ID 在 XOR 距离上最接近指定 `target` Node ID 的 `k` 个节点的联系信息。这主要用于路由表的填充和更新，以及在其他查找（如 `get_peers`）中进行迭代查询。
`find_node` 请求用于询问某个节点：“请告诉我你认识的、离 `target` 这个 Node ID 最近的那些节点是谁？”。响应则直接返回一个包含这些节点（最多 k 个）联系信息的紧凑字符串。这个 RPC 是 DHT 网络进行节点发现和路由的基础。

**A. `find_node` 请求结构 (`y` = 'q')**



```JSON
{
  "t": "<transaction_id>",  // 请求者的事务 ID (二进制字符串)
  "y": "q",                 // 类型：查询
  "q": "find_node",         // 查询方法名 (字符串)
  "a": {                    // 参数字典
    "id": "<requester_node_id>", // 请求者自身的 Node ID (20 字节二进制字符串)
    "target": "<target_node_id>"   // 查找的目标 Node ID (20 字节二进制字符串)
  }
  // "v": "<requester_version>" // 可选：请求者客户端版本
}
```

**请求参数 (`a` 字典) 详解:**

- **`id`**: 发起请求的节点的 Node ID。接收节点可以用这个 ID 来更新自己的路由表（K-桶）。
- **`target`**: 查找的目标节点的 Node ID。接收节点需要返回其路由表中，节点 ID 与这个 `target` 值在 XOR 距离上最接近的节点列表。

**B. `find_node` 响应结构 (`y` = 'r')**

JSON

```JSON
{
  "t": "<transaction_id>",  // 与请求中相同的事务 ID
  "y": "r",                 // 类型：响应
  "r": {                    // 返回值字典
    "id": "<responder_node_id>", // 响应节点的 Node ID (20 字节二进制字符串)
    "nodes": "<compact_node_info_string>" // 包含 k 个最接近 target 的节点信息的字符串
  }
  // "ip": "<requester_ip_port>" // 可选：响应者看到的请求者的 IP/端口
  // "v": "<responder_version>" // 可选：响应者客户端版本
}
```

**响应参数 (`r` 字典) 详解:**

- **`id`**: 响应这个 `find_node` 请求的节点的 Node ID。
- **`nodes`**: 一个二进制字符串，包含了响应节点所知道的、最接近 `target` Node ID 的 `k` 个节点的**紧凑节点信息 (Compact Node Info)**。
    - 这个字符串是由多个 **26 字节** 的块连接而成的。
    - 每个 26 字节块代表一个节点，其结构为：
        - 前 20 字节: 节点的 Node ID (网络字节序/Big-Endian)
        - 接下来 4 字节: 节点的 IPv4 地址 (网络字节序/Big-Endian)
        - 最后 2 字节: 节点的端口号 (网络字节序/Big-Endian)


### 2、 `get_peers` RPC (获取 Peer)

**目的:** 向一个 DHT 节点询问，有哪些 Peer 正在参与分享具有特定 info-hash 的种子 (Torrent)。

**A. `get_peers` 请求结构 (`y` = 'q')**


```json
{
  "t": "<transaction_id>",  // 请求者的事务 ID (二进制字符串)
  "y": "q",                 // 类型：查询
  "q": "get_peers",         // 查询方法名 (字符串)
  "a": {                    // 参数字典
    "id": "<requester_node_id>", // 请求者自身的 Node ID (20 字节二进制字符串)
    "info_hash": "<target_info_hash>" // 目标种子的 Info-Hash (20 字节二进制字符串) - 这是查找的 KEY
  }
  // "v": "<requester_version>" // 可选：请求者客户端版本
}
```

**B. `get_peers` 响应结构 (`y` = 'r')**


```json
{
  "t": "<transaction_id>",  // 与请求中相同的事务 ID
  "y": "r",                 // 类型：响应
  "r": {                    // 返回值字典
    "id": "<responder_node_id>", // 响应节点的 Node ID (20 字节二进制字符串)
    "token": "<opaque_token>",     // 一个不透明的令牌 (二进制字符串) - **非常关键**，用于后续的 announce_peer

    // --- 注意：下面的 'values' 或 'nodes' 字段通常只会存在一个 ---

    "values": [             // 可选：仅当响应节点存储了该 info_hash 的 Peer 信息时包含此字段
      "<compact_peer_info_1>", // 这是一个列表 (List)，每个元素是 6 字节的紧凑 Peer 信息 (IP+Port)
      "<compact_peer_info_2>", // (4 字节 IP + 2 字节 Port，均为网络字节序)
      ...
    ],

    "nodes": "<compact_node_info_string>" // 可选：仅当响应节点*没有* Peer 信息时包含此字段
                                      // 这是一个二进制字符串，由多个 26 字节的紧凑节点信息块连接而成
                                      // (20 字节 Node ID + 4 字节 IP + 2 字节 Port，均为网络字节序)
                                      // 代表响应者所知的、更接近目标 info_hash 的其他 DHT 节点
  }
  // "ip": "<requester_ip_port>" // 可选：响应者看到的请求者的 IP/端口
  // "v": "<responder_version>" // 可选：响应者客户端版本
}
```


**关键字段详解 (`r` 字典内部):**

1. **`id` (必须):**
    
    - 类型：二进制字符串 (20 字节)
    - 含义：响应这个 `get_peers` 请求的节点的 Node ID。
2. **`token` (必须):**
    
    - 类型：二进制字符串 (通常较短)
    - 含义：这是一个非常重要的**令牌**。当发起 `get_peers` 请求的节点（请求者）稍后想要向**同一个**响应节点宣告自己是该 `info_hash` 的 Peer 时（通过 `announce_peer` RPC），**必须**提供这个 `token`。接收 `announce_peer` 的节点会用它来验证请求的合法性，防止 DHT 污染。
3. **`values` (可选，与 `nodes` 二选一):**
    
    - 类型：Bencode 列表 (List)，列表中的每个元素是二进制字符串。
    - 含义：如果响应节点**存储了**与请求的 `info_hash` 相关联的 Peer 信息，则会包含此字段。
    - 列表中的每个元素是一个 **6 字节** 的字符串，代表一个 Peer 的联系信息：
        - 前 4 字节：Peer 的 IPv4 地址 (网络字节序/Big-Endian)。
        - 后 2 字节：Peer 的端口号 (网络字节序/Big-Endian)。
    - 这直接告诉了请求者可以尝试连接哪些 Peer 来下载该 Torrent。
4. **`nodes` (可选，与 `values` 二选一):**
    
    - 类型：二进制字符串
    - 含义：如果响应节点**没有存储**关于该 `info_hash` 的 Peer 信息，它会遵循 Kademlia 的 `find_node` 行为，返回它所知道的、其 Node ID 在 XOR 距离上**最接近**目标 `info_hash` 的 `k` 个节点的信息。
    - 这个字符串是由多个 **26 字节** 的紧凑节点信息块连接而成的：
        - 前 20 字节：节点的 Node ID (网络字节序)。
        - 接下来 4 字节：节点的 IPv4 地址 (网络字节序)。
        - 最后 2 字节：节点的端口号 (网络字节序)。
    - 收到这个字段意味着请求者需要继续向这些返回的“更近”的节点发送 `get_peers` 请求，以继续查找 Peer。


---

### 3、 `announce_peer` RPC (宣告 Peer)

**目的:** 通知一个 DHT 节点（通常是其 Node ID 接近目标 info-hash 的节点），告知发送方是特定种子的一个活跃 Peer，并请求该节点存储发送方的联系信息，以便其他 Peer 可以通过 `get_peers` 找到它。

**A. `announce_peer` 请求结构 (`y` = 'q')**


```json
{
  "t": "<transaction_id>",    // 请求者的事务 ID
  "y": "q",                   // 类型：查询
  "q": "announce_peer",       // 查询方法名
  "a": {                      // 参数字典
    "id": "<announcer_node_id>",   // 宣告者自身的 Node ID (20 字节二进制字符串)
    "info_hash": "<target_info_hash>", // 目标种子的 Info-Hash (20 字节二进制字符串)
    "port": <listening_port>,     // 宣告者监听 BitTorrent 连接的端口号 (整数)
    "token": "<received_token>",    // **必须提供**：先前从**该**响应节点为**同一** info_hash 调用 `get_peers` 时收到的那个 `token` (二进制字符串) - **非常关键**
    "implied_port": <0_or_1>      // 可选：整数 0 或 1。如果为 1，则接收方应忽略 'port' 参数，使用该 UDP 包的源端口；如果为 0 或省略，则使用 'port' 参数。
  }
  // "v": "<announcer_version>" // 可选：宣告者客户端版本
}
```

**B. `announce_peer` 响应结构 (`y` = 'r')**

这个响应非常简单，主要是确认收到了宣告请求（如果 `token` 有效且没发生其他错误的话）。


```json
{
  "t": "<transaction_id>",  // 与请求中相同的事务 ID
  "y": "r",                 // 类型：响应
  "r": {                    // 返回值字典
    "id": "<responder_node_id>" // 响应节点的 Node ID (20 字节二进制字符串) - 基本上就是确认收到
  }
  // "ip": "<announcer_ip_port>" // 可选：响应者看到的宣告者的 IP/端口
  // "v": "<responder_version>" // 可选：响应者客户端版本
}
```

**需要注意的是:** `announce_peer` 的成功响应 (`y`='r') 并不绝对保证 Peer 信息已被永久存储，只表明请求已被接收且（很可能）通过了 `token` 验证。节点的存储行为还可能受其内部策略（如存储容量限制、速率限制等）的影响。如果发生错误（如 `token` 无效），则会返回错误消息 (`y`='e')。





