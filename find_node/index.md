# 



### `find_node` RPC

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



