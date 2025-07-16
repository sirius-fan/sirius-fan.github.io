# KRPC


好的，我们来详细介绍一下 KRPC 协议。这是 BitTorrent Mainline DHT 网络中节点之间进行通信所使用的协议。
KRPC 是 BitTorrent DHT 的基石通信协议。它利用 UDP 的高效性和 Bencode 的简洁性，定义了一套清晰的查询/响应机制（`ping`, `find_node`, `get_peers`, `announce_peer`）以及错误处理方式

**1. 基础**

- **传输层协议**: KRPC 完全构建在 **UDP (User Datagram Protocol)** 之上。选择 UDP 是因为它开销低、无连接，适合 P2P 网络中大量节点间的快速、短暂交互，避免了 TCP 连接管理的复杂性。
- **编码**: 所有 KRPC 消息都使用 **Bencode (B-encoding)** 进行编码。这是 BitTorrent 生态系统中广泛使用的一种简单、紧凑的数据序列化格式（也用于 `.torrent` 文件和 Tracker 响应）。

**2. 消息结构**

所有 KRPC 消息都是一个 Bencode 编码的**字典 (Dictionary)**。这个字典包含一些通用字段，以及根据消息类型（查询、响应或错误）特定的字段。

**通用字段:**

- `t`: **Transaction ID** (事务 ID)。这是一个由查询方生成的短二进制字符串（通常 2 字节）。响应方必须在对应的响应或错误消息中原样包含这个 ID。这使得查询方能够将收到的响应与之前发出的查询匹配起来，尤其是在同时发出多个查询时。
- `y`: **Message Type** (消息类型)。单个字符，表示消息的类型：
    - `'q'`: **Query** (查询) - 由发起通信的节点发送。
    - `'r'`: **Response** (响应) - 对成功处理的查询的回复。
    - `'e'`: **Error** (错误) - 当无法处理查询时返回。
- `v` (可选): **Version** (版本号)。表示发送者客户端的软件版本信息，通常是一个短字符串。主要用于统计和调试，协议本身不强制要求或使用它。
- `ip` (可选, 主要在响应中): 发送者看到的请求者的**公网 IP 地址和端口**。这可以帮助位于 NAT 后面的节点发现自己的公网 IP。格式是紧凑的 6 字节（4 字节 IP + 2 字节 Port）。

**3. 消息类型详解**

**a) 查询 (Query) - `y` = `'q'`**

查询消息的字典中还包含以下字段：

- `q`: **Query Type** (查询类型)。表示具体 RPC 方法名称的字符串。例如 `"ping"`, `"find_node"`, `"get_peers"`, `"announce_peer"`。
- `a`: **Arguments** (参数)。一个 Bencode 字典，包含该查询类型所需的所有参数。

**具体的查询类型及其参数 (`a` 字典内容):**

- **`ping`**:
    - `id`: 发送该查询的节点的 Node ID (20 字节)。
    - **目的**: 检查目标节点是否在线，并让目标节点将查询方加入其路由表。

- **`find_node`**:    
    - `id`: 发送查询的节点的 Node ID (20 字节)。
    - `target`: 要查找的目标 Node ID (20 字节)。
    - **目的**: 请求接收方返回其路由表中离 `target` ID 最近的 `k` 个节点的信息。

- **`get_peers`**:    
    - `id`: 发送查询的节点的 Node ID (20 字节)。
    - `info_hash`: 目标 Torrent 的 Info-Hash (20 字节)。
    - **目的**: 请求接收方返回它所知道的正在下载/分享该 `info_hash` 的 Peer 列表。如果它没有 Peer 列表，则其行为应类似于 `find_node`，返回离 `info_hash` 最近的 `k` 个节点。

- **`announce_peer`**:
    - `id`: 发送查询的节点的 Node ID (20 字节)。
    - `info_hash`: 要宣告的 Torrent 的 Info-Hash (20 字节)。
    - `port`: 发送方（宣告者）监听 BitTorrent 连接的端口号 (整数)。
    - `token`: **必需**。这是之前从同一接收节点处执行 `get_peers` 查询时收到的 `token` 值 (二进制字符串)。
    - `implied_port` (可选): 一个整数值 0 或 1。如果设置为 1，表示接收方应忽略消息中明确指定的 `port` 参数，而使用该 UDP 包的**源端口**作为宣告的端口。这对于位于 NAT 后面的节点，如果不确定自己的公网监听端口时很有用。
    - **目的**: 告知接收节点，发送方正在指定的端口上参与 `info_hash` 的 Torrent 下载/分享，请求接收节点存储这些信息以便回复给其他 `get_peers` 查询。

**b) 响应 (Response) - `y` = `'r'`**

响应消息的字典中还包含以下字段：

- `r`: **Return Values** (返回值)。一个 Bencode 字典，包含查询成功执行后的结果。

**具体的响应类型及其返回值 (`r` 字典内容):**

- **对 `ping` 的响应**:
    
    - `id`: 响应节点的 Node ID (20 字节)。
- **对 `find_node` 的响应**:
    
    - `id`: 响应节点的 Node ID (20 字节)。
    - `nodes`: 一个包含**紧凑节点信息 (Compact Node Info)** 的字符串。包含了响应者认为离 `target` ID 最近的 `k` 个节点的信息。
- **对 `get_peers` 的响应**:
    
    - `id`: 响应节点的 Node ID (20 字节)。
    - `token`: 一个短的二进制字符串。查询方在后续向该节点发送 `announce_peer` 时必须提供此 `token`。
    - **可能包含以下之一**:
        - `values`: 一个 Bencode **列表 (List)**，其中每个元素是一个**紧凑 Peer 信息 (Compact Peer Info)** 字符串，代表一个 Peer 的 IP 和端口。
        - `nodes`: 如果响应节点没有关于该 `info_hash` 的 Peer 信息，则返回此字段，其格式和含义与 `find_node` 响应中的 `nodes` 字段相同。
- **对 `announce_peer` 的响应**:
    
    - `id`: 响应节点的 Node ID (20 字节)。
    - **注意**: `announce_peer` 的成功响应通常只包含 `id`，表示确认收到并（可能已）存储了宣告信息。不需要返回其他特定值。

**c) 错误 (Error) - `y` = `'e'`**

错误消息的字典中还包含以下字段：

- `e`: **Error Info** (错误信息)。一个 Bencode **列表 (List)**，包含两个元素：
    - 第一个元素: **Error Code** (错误码)，一个整数。
    - 第二个元素: **Error Message** (错误消息)，一个描述错误的字符串。

**常见的错误码:**

- `201`: Generic Error (通用错误)
- `202`: Server Error (服务器错误，节点内部问题)
- `203`: Protocol Error (协议错误，如格式错误、无效参数等)
- `204`: Method Unknown (方法未知，如请求了不支持的 RPC)

**4. 紧凑节点/Peer 信息格式 (Compact Node/Peer Info)**

为了节省带宽，节点和 Peer 的信息通常使用紧凑的二进制格式表示：

- **紧凑节点信息 (Compact Node Info)**: 每个节点信息是 **26 字节** 的字符串。
    
    - 前 20 字节: Node ID (网络字节序/Big-Endian)。
    - 接下来 4 字节: IPv4 地址 (网络字节序)。
    - 最后 2 字节: 端口号 (网络字节序)。
    - `nodes` 字段的值就是多个这样的 26 字节块连接起来的字符串。
- **紧凑 Peer 信息 (Compact Peer Info)**: 每个 Peer 信息是 **6 字节** 的字符串。
    
    - 前 4 字节: IPv4 地址 (网络字节序)。
    - 后 2 字节: 端口号 (网络字节序)。
    - `values` 字段的值是 Bencode 列表，列表中的每个元素是这样一个 6 字节的字符串。

# 总结

KRPC 是 BitTorrent DHT 的基石通信协议。它利用 UDP 的高效性和 Bencode 的简洁性，定义了一套清晰的查询/响应机制（`ping`, `find_node`, `get_peers`, `announce_peer`）以及错误处理方式。通过事务 ID (`t`) 匹配请求和响应，并使用紧凑的二进制格式 (`nodes`, `values`) 传输节点和 Peer 信息，KRPC 使得数百万 BitTorrent 客户端能够有效地组成一个庞大、去中心化的网络，用于发现彼此并协调文件共享。
